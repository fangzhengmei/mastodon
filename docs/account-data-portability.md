# 账户数据可移植性

本文档描述了 Mastodon 中用户数据导出、导入以及跨实例联邦同步时的后台任务处理、敏感字段裁剪机制和隐私约束。

---

## 目录

1. [用户导出存档](#用户导出存档)
2. [导入关注/屏蔽列表](#导入关注屏蔽列表)
3. [联邦协议同步边界与隐私约束](#联邦协议同步边界与隐私约束)

---

## 用户导出存档

### 后台任务处理机制

用户导出存档通过异步后台任务处理，确保不会阻塞用户界面操作。

#### 任务流程

1. **用户请求导出**：用户在设置页面请求导出完整存档
2. **创建备份记录**：系统创建 `Backup` 记录，标记为待处理状态
3. **异步任务入队**：`BackupWorker` 被推送到 Sidekiq 队列（'pull' 队列）
4. **后台处理**：`BackupWorker` 调用 `BackupService` 执行实际的归档构建
5. **完成通知**：归档完成后，删除旧备份并通过邮件通知用户

#### 关键组件

**BackupWorker** (`app/workers/backup_worker.rb`)

```ruby
def perform(backup_id)
  backup = Backup.find(backup_id)
  user   = backup.user

  BackupService.new.call(backup)  # 执行实际的归档构建

  user.backups.where.not(id: backup.id).destroy_all  # 清理旧备份
  UserMailer.backup_ready(user, backup).deliver_later  # 发送通知邮件
end
```

**任务特性**：
- 队列：`pull`
- 重试：最多 5 次
- 失败处理：重试耗尽后删除备份记录

**BackupService** (`app/services/backup_service.rb`)

负责构建完整的用户数据归档 ZIP 文件，包含以下内容：

| 文件 | 内容描述 |
|------|----------|
| `actor.json` | 用户的 ActivityPub 表示（账户信息） |
| `outbox.json` | 用户发布的所有状态（嘟文） |
| `likes.json` | 用户的点赞记录 |
| `bookmarks.json` | 用户的书签记录 |
| 媒体文件 | 头像、头部图、状态中的媒体附件 |

#### 归档构建流程

1. **创建临时 ZIP 文件**
2. **按组件分批写入**：
   - 状态数据分批处理（每批后执行 GC）
   - 媒体附件分批下载并添加到归档
3. **更新备份记录**：标记为已处理并保存文件引用
4. **清理临时文件**

### 敏感字段裁剪机制

用户导出存档时，系统会严格裁剪敏感字段，确保只导出必要的、用户有权获取的数据。

#### 裁剪策略

**1. 用户信息 (actor.json)**

使用 `ActivityPub::ActorSerializer` 序列化，只包含公开可访问的字段：

```ruby
attributes :id, :webfinger, :type, :following, :followers,
           :inbox, :outbox, :featured, :featured_tags,
           :preferred_username, :name, :summary,
           :url, :manually_approves_followers,
           :discoverable, :indexable, :published, :memorial,
           :show_featured, :show_media
```

**不包含的敏感信息**：
- 密码哈希
- 邮箱地址
- 双因素认证密钥
- 会话令牌
- 私有设置（如邮件通知偏好）

**2. 状态数据 (outbox.json)**

使用 `ActivityPub::CreateNoteSerializer` 或 `ActivityPub::AnnounceNoteSerializer`，并进行额外裁剪：

```ruby
# 删除 @context 字段（避免重复）
item.delete(:@context)

# 媒体附件 URL 转换为相对路径
unless item[:type] == 'Announce' || item[:object][:attachment].blank?
  item[:object][:attachment].each do |attachment|
    attachment[:url] = Addressable::URI.parse(attachment[:url]).path.delete_prefix('/system/')
  end
end
```

**裁剪内容**：
- 移除重复的 JSON-LD 上下文
- 将绝对 URL 转换为相对路径（去除 `/system/` 前缀）
- 不包含原始文件系统路径
- 不包含内部数据库 ID（仅使用 URI）

**3. 点赞和书签记录**

只包含状态的 URI，不包含额外元数据：

```ruby
# likes.json 和 bookmarks.json 只存储状态 URI
ActivityPub::TagManager.instance.uri_for(status).to_json
```

**4. 媒体附件处理**

媒体附件使用相对路径存储在归档中：

```ruby
# 去除文件系统路径前缀，只保留相对路径
path = path.gsub(%r{\A.*/system/}, '')
path = path.gsub(%r{\A/+}, '')
```

#### 数据最小化原则

导出存档遵循数据最小化原则：

1. **只导出用户拥有的数据**：不导出其他用户的私有信息
2. **只导出必要的元数据**：时间戳、URI 等必要信息，不含内部标识符
3. **相对路径引用**：媒体文件使用相对路径，避免泄露服务器结构
4. **无敏感凭证**：不包含任何可用于认证或访问的凭证

---

## 导入关注/屏蔽列表

### 后台任务处理机制

导入关注/屏蔽列表采用异步处理机制，支持大量数据的批量导入。

#### 导入流程

1. **用户上传 CSV 文件**
2. **表单验证**：`Form::Import` 验证文件格式和内容
3. **创建导入记录**：创建 `BulkImport` 和 `BulkImportRow` 记录
4. **用户确认**：用户确认后开始处理
5. **异步处理**：`Import::RowWorker` 逐行处理导入数据
6. **进度跟踪**：实时更新导入进度和状态

#### 关键组件

**Form::Import** (`app/models/form/import.rb`)

处理 CSV 上传和初步验证：

**限制条件**：
- 文件大小上限：20 MB
- 行数上限：20,000 行
- 关注数量上限：遵循用户的关注限制

**支持的导入类型**：

| 类型 | 描述 | 必需表头 |
|------|------|----------|
| `following` | 关注列表 | Account address, Show boosts, Notify on new posts, Languages |
| `blocking` | 屏蔽列表 | Account address |
| `muting` | 静音列表 | Account address, Hide notifications |
| `domain_blocking` | 域名屏蔽 | #domain |
| `bookmarks` | 书签 | #uri |
| `lists` | 列表 | List name, Account address |

**数据处理**：

```ruby
# 只提取期望的字段，忽略其他所有字段
def parsed_rows
  csv_data.rewind
  expected_headers = EXPECTED_HEADERS_BY_TYPE[type.to_sym]
  
  csv_data.take(ROWS_PROCESSING_LIMIT + 1).map do |row|
    row.to_h.slice(*expected_headers).transform_keys { |key| ATTRIBUTE_BY_HEADER[key] }
  end
end
```

**BulkImportService** (`app/services/bulk_import_service.rb`)

根据导入类型执行不同的处理逻辑：

**处理模式**：
- `merge`：合并到现有数据（默认）
- `overwrite`：覆盖现有数据

**异步处理流程**（以关注导入为例）：

```ruby
def import_follows!
  rows_by_acct = extract_rows_by_acct

  # 覆盖模式：先取消不在导入列表中的关注
  if @import.overwrite?
    @account.following.reorder(nil).find_each do |followee|
      row = rows_by_acct.delete(followee.acct)
      if row.nil?
        UnfollowService.new.call(@account, followee)
      else
        # 更新现有关注关系的设置
        FollowService.new.call(@account, followee, reblogs: row.data['show_reblogs'], ...)
      end
    end
  end

  # 剩余的行使用异步 Worker 处理
  Import::RowWorker.push_bulk(rows_by_acct.values) do |row|
    [row.id]
  end
end
```

**Import::RowWorker** (`app/workers/import/row_worker.rb`)

逐行处理导入数据：

```ruby
def perform(row_id)
  row = BulkImportRow.eager_load(bulk_import: :account).find_by(id: row_id)
  return true if row.nil?

  imported = BulkImportRowService.new.call(row)  # 处理单行
  mark_as_processed!(row, imported)  # 更新进度
end
```

**任务特性**：
- 队列：`pull`
- 重试：最多 6 次
- 失败处理：重试耗尽后标记为处理完成但不计入成功

#### 批量导入优化

1. **批量入队**：使用 `push_bulk` 一次性将所有行入队，减少 Redis 操作
2. **进度跟踪**：每处理完一行更新 `processed_items` 和 `imported_items`
3. **自动完成检测**：当 `processed_items` 等于 `total_items` 时标记导入完成

### 敏感字段裁剪机制

导入过程中严格裁剪和验证输入数据，防止注入恶意或敏感内容。

#### 裁剪策略

**1. 字段白名单机制**

只处理预定义的字段，忽略所有其他字段：

```ruby
# 每种导入类型有明确的期望字段
EXPECTED_HEADERS_BY_TYPE = {
  following: ['Account address', 'Show boosts', 'Notify on new posts', 'Languages'],
  blocking: ['Account address'],
  muting: ['Account address', 'Hide notifications'],
  domain_blocking: ['#domain'],
  bookmarks: ['#uri'],
  lists: ['List name', 'Account address'],
}.freeze

# 表头到内部属性的映射
ATTRIBUTE_BY_HEADER = {
  'Account address' => 'acct',
  'Show boosts' => 'show_reblogs',
  'Notify on new posts' => 'notify',
  'Languages' => 'languages',
  'Hide notifications' => 'hide_notifications',
  '#domain' => 'domain',
  '#uri' => 'uri',
  'List name' => 'list_name',
}.freeze
```

**2. 数据类型转换和验证**

```ruby
csv_converter = lambda do |field, field_info|
  case field_info.header
  when 'Show boosts', 'Notify on new posts', 'Hide notifications'
    # 布尔值转换
    ActiveModel::Type::Boolean.new.cast(field&.downcase)
  when 'Languages'
    # 语言列表解析
    field&.split(',')&.map(&:strip)&.presence
  when 'Account address'
    # 去除 @ 前缀，去除首尾空格
    field.strip.gsub(/\A@/, '')
  when '#domain'
    # 域名标准化：小写、去空格
    field&.strip&.downcase
  when '#uri', 'List name'
    # 简单去空格
    field.strip
  else
    field
  end
end
```

**3. 表头自动检测**

如果 CSV 表头不匹配已知格式，使用默认表头：

```ruby
# 如果第一行不是已知的表头，使用默认表头
@csv_data = CSV.open(data.path, encoding: 'UTF-8', skip_blanks: true, 
                     headers: default_csv_headers, converters: csv_converter) 
  unless KNOWN_FIRST_HEADERS.include?(@csv_data.headers&.first)
```

**4. 严格验证**

```ruby
def validate_data
  return if data.nil?
  # 文件大小验证
  return errors.add(:data, I18n.t('imports.errors.too_large')) if data.size > FILE_SIZE_LIMIT
  # 表头兼容性验证
  return errors.add(:data, I18n.t('imports.errors.incompatible_type')) unless default_csv_headers.all? { |header| csv_data.headers.include?(header) }
  # 行数限制验证
  errors.add(:data, I18n.t('imports.errors.over_rows_processing_limit', count: ROWS_PROCESSING_LIMIT)) if csv_row_count > ROWS_PROCESSING_LIMIT
  # 关注数量限制验证
  if type.to_sym == :following
    base_limit = FollowLimitValidator.limit_for_account(current_account)
    limit = base_limit
    limit -= current_account.following_count unless overwrite
    errors.add(:data, I18n.t('users.follow_limit_reached', limit: base_limit)) if csv_row_count > limit
  end
end
```

#### 导入数据清理机制

**Vacuum::ImportsVacuum** (`app/lib/vacuum/imports_vacuum.rb`)

定期清理过期的导入数据：

```ruby
def perform
  clean_unconfirmed_imports!  # 清理未确认的导入
  clean_old_imports!          # 清理已完成的旧导入
end
```

**清理策略**：
- 未确认的导入：超过一定时间后自动删除
- 已完成的导入：归档完成后删除记录
- 防止敏感数据长期存储在系统中

---

## 联邦协议同步边界与隐私约束

Mastodon 使用 ActivityPub 协议进行跨实例数据同步。同步过程中严格遵守隐私边界和数据最小化原则。

### ActivityPub 序列化器

所有跨实例同步的数据都通过专门的序列化器控制，确保只同步必要的、公开的信息。

#### ActorSerializer（账户信息）

**ActivityPub::ActorSerializer** (`app/serializers/activitypub/actor_serializer.rb`)

控制账户信息如何序列化为 ActivityPub 格式：

```ruby
attributes :id, :webfinger, :type, :following, :followers,
           :inbox, :outbox, :featured, :featured_tags,
           :preferred_username, :name, :summary,
           :url, :manually_approves_followers,
           :discoverable, :indexable, :published, :memorial,
           :show_featured, :show_media
```

**序列化字段分类**：

| 类别 | 字段 | 描述 |
|------|------|------|
| **标识符** | `id`, `webfinger`, `preferred_username` | 账户唯一标识 |
| **端点** | `inbox`, `outbox`, `following`, `followers`, `featured`, `featured_tags` | ActivityPub 集合端点 |
| **展示信息** | `name`, `summary`, `url`, `icon`, `image` | 用户名、简介、头像、头部图 |
| **隐私设置** | `manually_approves_followers`, `discoverable`, `indexable` | 控制数据如何被发现和使用 |
| **状态信息** | `published`, `memorial`, `suspended`, `moved_to`, `also_known_as` | 账户状态 |
| **内容设置** | `show_featured`, `show_media`, `show_replies_in_media` | 内容展示偏好 |
| **安全** | `public_key` | 用于验证签名的公钥 |

**条件性包含的字段**：

```ruby
attribute :moved_to, if: :moved?           # 仅当账户迁移时
attribute :also_known_as, if: :also_known_as?  # 仅当有别名时
attribute :suspended, if: :suspended?       # 仅当账户被暂停时
attribute :interaction_policy, if: -> { Mastodon::Feature.collections_enabled? }  # 功能开关
```

**不可用账户的特殊处理**：

```ruby
def discoverable
  object.unavailable? ? false : (object.discoverable || false)
end

def name
  object.unavailable? ? object.username : (object.display_name.presence || object.username)
end

def summary
  object.unavailable? ? '' : account_bio_format(object)
end
```

当账户不可用时（如被暂停或删除），敏感信息会被替换为默认值。

#### NoteSerializer（状态/嘟文）

**ActivityPub::NoteSerializer** (`app/serializers/activitypub/note_serializer.rb`)

控制状态数据的序列化：

```ruby
attributes :id, :type, :summary,
           :in_reply_to, :published, :url,
           :attributed_to, :to, :cc, :sensitive,
           :atom_uri, :in_reply_to_atom_uri,
           :conversation, :context

attribute :content
attribute :content_map, if: :language?
attribute :updated, if: :edited?
```

**关键字段说明**：

| 字段 | 描述 | 隐私考虑 |
|------|------|----------|
| `id` | 状态的唯一 URI | 使用 URI 而非内部 ID |
| `to` / `cc` | 受众信息 | 控制状态可见性 |
| `sensitive` | 敏感内容标记 | 继承账户设置或显式标记 |
| `attributed_to` | 作者账户 URI | 仅引用，不嵌入完整信息 |
| `in_reply_to` | 回复的状态 URI | 仅引用，不嵌入完整内容 |

**敏感内容标记**：

```ruby
def sensitive
  object.account.sensitized? || object.sensitive
end
```

如果账户被标记为"敏感内容账户"，或状态本身被标记为敏感，则 `sensitive` 字段为 `true`。

**交互策略**：

```ruby
def interaction_policy
  approved_uris = []
  policy = object.quote_interaction_policy.automatic
  
  # 根据策略决定哪些受众可以自动获得引用权限
  approved_uris << ActivityPub::TagManager::COLLECTIONS[:public] if policy.public?
  approved_uris << ActivityPub::TagManager.instance.followers_uri_for(object.account) if policy.followers?
  approved_uris << ActivityPub::TagManager.instance.following_uri_for(object.account) if policy.following?
  approved_uris << ActivityPub::TagManager.instance.uri_for(object.account) if approved_uris.empty?

  {
    canQuote: {
      automaticApproval: approved_uris,
    },
  }
end
```

### 数据同步边界

#### 1. 可见性控制

ActivityPub 使用 `to` 和 `cc` 字段控制状态的可见性：

| 可见性 | `to` | `cc` | 描述 |
|--------|------|------|------|
| 公开 | `https://www.w3.org/ns/activitystreams#Public` | 粉丝集合 | 所有人可见 |
| 不公开 | - | `Public` + 粉丝集合 | 不在公共时间线显示 |
| 仅限粉丝 | 粉丝集合 | - | 仅粉丝可见 |
| 仅限提及 | 提及的账户 | - | 仅提及的人可见 |
| 私信 | 特定账户 | - | 仅指定账户可见 |

**TagManager 中的受众计算**：

```ruby
# to 和 cc 的计算基于状态的可见性设置
def to(status)
  # 根据 visibility 返回不同的受众集合
end

def cc(status)
  # 根据 visibility 返回不同的抄送集合
end
```

#### 2. 只同步必要数据

**数据最小化原则**：

1. **引用而非嵌入**：
   - 回复的状态使用 URI 引用，不嵌入完整内容
   - 提及的账户使用 URI 引用，不嵌入完整信息
   - 媒体附件使用 URL 引用，不内联二进制数据

2. **条件性字段**：
   - `replies`、`likes`、`shares` 集合仅对本地状态包含
   - `updated` 仅在状态被编辑时包含
   - `content_map` 仅在有语言设置时包含

3. **无内部标识符**：
   - 不包含数据库主键（`id`）
   - 所有引用使用 URI
   - 时间戳使用 ISO 8601 格式

#### 3. 实例间的信任边界

**每个实例独立管理**：

1. **账户管理**：
   - 每个实例只管理自己的用户
   - 远程账户是"影子"副本，定期同步
   - 实例可以选择暂停或限制与特定实例的通信

2. **内容审核**：
   - 每个实例独立进行内容审核
   - 实例可以屏蔽来自特定实例的内容
   - 用户可以屏蔽特定实例或账户

3. **数据保留**：
   - 实例根据自己的政策保留数据
   - 远程数据的同步频率由实例配置决定
   - 删除操作通过 Delete 活动传播，但不保证所有实例都立即删除

### 隐私约束

#### 1. 用户可控的隐私设置

**账户级设置**：

| 设置 | ActivityPub 字段 | 影响 |
|------|-------------------|------|
| 锁定账户 | `manually_approves_followers` | 需要手动批准关注请求 |
| 可发现 | `discoverable` | 是否出现在搜索和推荐中 |
| 可索引 | `indexable` | 是否允许搜索引擎索引 |
| 纪念账户 | `memorial` | 标记为纪念账户 |

**状态级设置**：

| 可见性 | 描述 | 联邦同步范围 |
|--------|------|--------------|
| 公开 | 所有人可见 | 同步到所有关注者和公共时间线 |
| 不公开 | 不在公共时间线 | 同步到关注者，不出现在公共时间线 |
| 仅限粉丝 | 仅粉丝可见 | 仅同步到已确认的粉丝 |
| 仅限提及 | 仅提及的人 | 仅同步到提及的账户 |
| 私信 | 仅指定账户 | 仅同步到指定的接收者 |

#### 2. 敏感信息保护

**不跨实例同步的信息**：

1. **认证信息**：
   - 密码哈希
   - 双因素认证密钥
   - 会话令牌
   - API 密钥

2. **私有设置**：
   - 邮箱地址
   - 邮件通知偏好
   - 隐私设置的详细配置
   - 语言和界面偏好

3. **内部数据**：
   - 数据库主键
   - IP 地址
   - 用户代理字符串
   - 内部统计数据

4. **交互元数据**：
   - 谁查看了你的状态
   - 私信的详细递送状态
   - 草稿内容

#### 3. 数据删除和迁移

**删除操作**：

```ruby
# 删除状态时发送 Delete 活动
ActivityPub::DeleteNoteSerializer

# 删除账户时发送 Delete 活动
ActivityPub::DeleteActorSerializer
```

**注意事项**：
- Delete 活动会发送到所有已知的收件人
- 但不保证所有实例都会立即或完全删除数据
- 实例可能有自己的数据保留政策

**账户迁移**：

```ruby
# 账户迁移时发送 Move 活动
ActivityPub::MoveSerializer

# ActorSerializer 中的 moved_to 字段
attribute :moved_to, if: :moved?
```

迁移流程：
1. 用户在新实例上声明旧账户
2. 旧实例验证所有权
3. 发送 Move 活动通知关注者
4. 关注者的实例自动更新关注关系

#### 4. 媒体附件处理

**媒体附件的序列化**：

```ruby
class MediaAttachmentSerializer < ActivityPub::Serializer
  attributes :type, :media_type, :url, :name, :blurhash
  attribute :focal_point, if: :focal_point?
  attribute :width, if: :width?
  attribute :height, if: :height?
end
```

**隐私考虑**：
- 媒体文件通过 URL 访问
- 远程实例可以缓存媒体文件
- 敏感媒体可能需要额外的访问控制
- 实例可以选择不代理或缓存远程媒体

### 交互策略与内容控制

**引用/转嘟权限控制**：

Mastodon 支持精细的内容交互策略：

```ruby
def interaction_policy
  {
    canQuote: {
      automaticApproval: approved_uris,  # 可以自动引用的受众
    },
  }
end
```

**策略选项**：
- `public`：所有人可以引用
- `followers`：仅粉丝可以引用
- `following`：仅关注的人可以引用
- `self`：仅自己可以引用

**引用授权流程**：

1. 作者设置引用策略
2. 当其他用户尝试引用时，检查策略
3. 如果策略不允许自动批准，需要请求授权
4. 作者可以批准或拒绝引用请求

### 总结：数据同步的隐私原则

1. **数据最小化**：只同步完成功能所需的最少数据
2. **用户控制**：用户可以通过隐私设置控制数据的可见性和传播
3. **实例自治**：每个实例独立管理自己的数据和政策
4. **透明性**：用户应该知道哪些数据会被同步到哪里
5. **可删除性**：用户应该能够删除自己的数据（尽最大努力）
6. **安全传输**：所有联邦通信使用 HTTPS 加密

---

## 参考代码位置

| 功能 | 文件路径 |
|------|----------|
| 导出模型 | `app/models/export.rb` |
| 备份服务 | `app/services/backup_service.rb` |
| 备份 Worker | `app/workers/backup_worker.rb` |
| 导入表单 | `app/models/form/import.rb` |
| 批量导入服务 | `app/services/bulk_import_service.rb` |
| 导入行 Worker | `app/workers/import/row_worker.rb` |
| 导入数据清理 | `app/lib/vacuum/imports_vacuum.rb` |
| Actor 序列化器 | `app/serializers/activitypub/actor_serializer.rb` |
| Note 序列化器 | `app/serializers/activitypub/note_serializer.rb` |
