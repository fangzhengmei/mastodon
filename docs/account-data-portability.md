# 账户数据可移植性

本文档描述了 Mastodon 中用户数据导出、导入以及跨实例联邦同步时的后台任务处理、敏感字段裁剪机制和隐私约束。

---

## 目录

1. [用户导出存档](#用户导出存档)
2. [导入关注/屏蔽列表](#导入关注屏蔽列表)
3. [联邦协议同步边界与隐私约束](#联邦协议同步边界与隐私约束)
4. [关注/屏蔽操作的跨实例同步链路](#关注屏蔽操作的跨实例同步链路)

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

## 关注/屏蔽操作的跨实例同步链路

当用户导入关注/屏蔽列表或直接执行关注/屏蔽操作时，系统会通过 ActivityPub 协议与远程实例进行同步。本节详细描述这个同步链路的技术细节。

### 1. 投递对象选择机制

#### 1.1 关注操作的投递目标选择

**FollowService** (`app/services/follow_service.rb`)

关注操作的投递逻辑区分本地账户和远程账户：

```ruby
def call(source_account, target_account, options = {})
  @source_account = source_account
  @target_account = target_account
  @options        = { bypass_locked: false, bypass_limit: false, with_rate_limit: false }.merge(options)

  # 前置检查：是否允许关注
  raise ActiveRecord::RecordNotFound if following_not_possible?
  raise Mastodon::NotPermittedError  if following_not_allowed?

  # ... 处理已有关注或请求 ...

  # 根据目标账户类型选择不同的处理方式
  if (@target_account.locked? && !@options[:bypass_locked]) || @source_account.silenced? || @target_account.activitypub?
    request_follow!  # 发送关注请求
  elsif @target_account.local?
    direct_follow!   # 直接关注（本地账户）
  end
end
```

**投递条件判断** (`following_not_allowed?`)：

```ruby
def following_not_allowed?
  domain_not_allowed?(@target_account.domain) ||           # 实例级域名限制
    @target_account.blocking?(@source_account) ||          # 目标屏蔽了源
    @source_account.blocking?(@target_account) ||          # 源屏蔽了目标
    @target_account.moved? ||                               # 目标账户已迁移
    (!@target_account.local? && @target_account.ostatus?) || # 远程账户使用旧协议
    @source_account.domain_blocking?(@target_account.domain) # 用户级域名屏蔽
end
```

**域名限制判定范围**：

`following_not_allowed?` 包含 6 个检查条件，分为两类域名限制：

| 限制类型 | 检查条件 | 触发条件 | 影响范围 |
|---------|----------|----------|----------|
| **实例级域名限制** | `domain_not_allowed?(@target_account.domain)` | 管理员设置的 `DomainBlock.suspend?` 或 `!DomainAllow.allowed?` | 所有用户 |
| **用户级域名屏蔽** | `@source_account.domain_blocking?(@target_account.domain)` | 用户主动屏蔽该域名 | 仅该用户 |

其他检查条件：
- `@target_account.blocking?(@source_account)`：目标账户屏蔽了源账户
- `@source_account.blocking?(@target_account)`：源账户屏蔽了目标账户
- `@target_account.moved?`：目标账户已迁移
- `(!@target_account.local? && @target_account.ostatus?)`：远程账户使用已废弃的 OStatus 协议

**远程账户的关注请求投递**：

```ruby
def request_follow!
  follow_request = @source_account.request_follow!(@target_account, **follow_options.merge(rate_limit: @options[:with_rate_limit], bypass_limit: @options[:bypass_limit]))

  if @target_account.local?
    # 本地账户：发送本地通知
    LocalNotificationWorker.perform_async(@target_account.id, follow_request.id, follow_request.class.name, 'follow_request')
  elsif @target_account.activitypub?
    # 远程账户：通过 ActivityPub 投递
    ActivityPub::DeliveryWorker.perform_async(
      build_json(follow_request), 
      @source_account.id, 
      @target_account.inbox_url, 
      { 'bypass_availability' => true }
    )
  end

  follow_request
end
```

#### 1.2 屏蔽操作的投递目标选择

**BlockService** (`app/services/block_service.rb`)

屏蔽操作同样区分本地和远程账户：

```ruby
def call(account, target_account)
  return if account.id == target_account.id

  @account = account
  @target_account = target_account

  # 处理已有的关注关系
  handle_following_relationships
  handle_collections

  # 创建屏蔽关系
  NotificationPermission.where(account: account, from_account: target_account).destroy_all
  block = account.block!(target_account)

  # 异步处理（清理时间线等）
  BlockWorker.perform_async(account.id, target_account.id)
  
  # 远程账户：通过 ActivityPub 投递
  create_notification(block) if !target_account.local? && target_account.activitypub?
  
  block
end

def create_notification(block)
  ActivityPub::DeliveryWorker.perform_async(
    build_json(block), 
    block.account_id, 
    block.target_account.inbox_url
  )
end
```

#### 1.3 投递目标确定规则

| 目标账户类型 | 投递方式 | 目标地址 |
|-------------|----------|----------|
| **本地账户** | 本地通知 | 无网络投递，直接处理 |
| **远程 ActivityPub 账户** | ActivityPub::DeliveryWorker | `target_account.inbox_url` |
| **远程 OStatus 账户** | 不投递（已废弃） | 无 |

**关键决策点**：
1. **域名检查**：`domain_not_allowed?` 检查目标域名是否被屏蔽
2. **协议检查**：`activitypub?` 确认目标账户支持 ActivityPub 协议
3. **可用性检查**：`unavailable?` 确认账户未被暂停或删除

### 2. 字段最小化策略

关注/屏蔽操作的 ActivityPub 消息采用极端的字段最小化策略，只包含完成功能所需的最少信息。

#### 2.1 Follow 活动序列化器

**ActivityPub::FollowSerializer** (`app/serializers/activitypub/follow_serializer.rb`)

```ruby
class ActivityPub::FollowSerializer < ActivityPub::Serializer
  attributes :id, :type, :actor
  attribute :virtual_object, key: :object

  def id
    ActivityPub::TagManager.instance.uri_for(object) || [ActivityPub::TagManager.instance.uri_for(object.account), '#follows/', object.id].join
  end

  def type
    'Follow'
  end

  def actor
    ActivityPub::TagManager.instance.uri_for(object.account)
  end

  def virtual_object
    ActivityPub::TagManager.instance.uri_for(object.target_account)
  end
end
```

**序列化字段**（仅 4 个字段）：

| 字段 | 值 | 说明 |
|------|-----|------|
| `id` | 关注请求的唯一 URI | 用于后续的 Accept/Reject 响应 |
| `type` | `"Follow"` | 活动类型标识 |
| `actor` | 发起者账户 URI | 谁发起的关注 |
| `object` | 目标账户 URI | 关注谁 |

**不包含的信息**：
- 关注设置（是否显示转嘟、是否通知、语言过滤）
- 发起者的完整账户信息
- 目标的完整账户信息
- 任何内部数据库 ID
- 时间戳（由 HTTP 签名或接收方记录）

#### 2.2 Block 活动序列化器

**ActivityPub::BlockSerializer** (`app/serializers/activitypub/block_serializer.rb`)

```ruby
class ActivityPub::BlockSerializer < ActivityPub::Serializer
  attributes :id, :type, :actor
  attribute :virtual_object, key: :object

  def id
    ActivityPub::TagManager.instance.uri_for(object) || [ActivityPub::TagManager.instance.uri_for(object.account), '#blocks/', object.id].join
  end

  def type
    'Block'
  end

  def actor
    ActivityPub::TagManager.instance.uri_for(object.account)
  end

  def virtual_object
    ActivityPub::TagManager.instance.uri_for(object.target_account)
  end
end
```

**序列化字段**（与 Follow 相同，仅 4 个字段）：

| 字段 | 值 | 说明 |
|------|-----|------|
| `id` | 屏蔽的唯一 URI | 用于后续的 Undo 响应 |
| `type` | `"Block"` | 活动类型标识 |
| `actor` | 发起者账户 URI | 谁发起的屏蔽 |
| `object` | 目标账户 URI | 屏蔽谁 |

**不包含的信息**：
- 屏蔽原因
- 发起者的完整账户信息
- 任何内部元数据

#### 2.3 为什么这样设计？

**安全和隐私考虑**：

1. **防止信息泄露**：
   - 不暴露关注设置（如"不显示转嘟"可能暗示对目标的负面看法）
   - 不暴露发起者的私有信息
   - 最小化可用于追踪或分析的数据

2. **协议兼容性**：
   - 遵循 ActivityPub 规范的最小要求
   - 确保与其他实现（如 Pleroma、Misskey）的互操作性

3. **可验证性**：
   - 使用 URI 引用而非嵌入对象，接收方可以独立验证
   - HTTP 签名提供完整性和身份验证

### 3. 远端验签失败处理边界

所有入站的 ActivityPub 请求都必须通过 HTTP 签名验证。验证失败时，系统有明确的处理边界和降级策略。

#### 3.1 签名验证流程

**SignatureVerification** (`app/controllers/concerns/signature_verification.rb`)

```ruby
def signed_request_actor
  return @signed_request_actor if defined?(@signed_request_actor)

  raise Mastodon::SignatureVerificationError, 'Request not signed' unless signed_request?

  # 1. 从 keyId 获取公钥
  keypair = keypair_from_key_id

  raise Mastodon::SignatureVerificationError, "Public key not found for key #{signature_key_id}" if keypair.nil?

  # 2. 检查密钥有效性
  check_keypair_validity!(keypair)
  
  # 3. 尝试验证签名
  return (@signed_request_actor = keypair.actor) if signed_request.verified?(keypair)

  # 4. 验证失败，尝试刷新密钥（可能密钥已轮换）
  keypair = stoplight_wrapper.run { keypair_refresh_key!(keypair) }

  raise Mastodon::SignatureVerificationError, "Could not refresh public key #{signature_key_id}" if keypair.nil?

  # 5. 再次检查密钥有效性
  check_keypair_validity!(keypair)
  
  # 6. 再次尝试验证
  return (@signed_request_actor = keypair.actor) if signed_request.verified?(keypair)

  # 7. 所有尝试都失败
  fail_with! "Verification failed for #{keypair.actor.to_log_human_identifier} #{keypair.actor.uri} #{keypair.uri}"
end
```

#### 3.2 时间约束

**签名过期和时钟偏差容忍**：

```ruby
EXPIRATION_WINDOW_LIMIT = 12.hours  # 签名最大有效期
CLOCK_SKEW_MARGIN       = 1.hour     # 时钟偏差容忍
```

**处理逻辑**：
1. **签名时间戳检查**：请求的 `Date` 头部必须在合理时间范围内
2. **过期时间**：超过 12 小时的签名被视为无效
3. **时钟偏差**：允许最多 1 小时的时钟偏差

#### 3.3 密钥有效性检查

```ruby
def check_keypair_validity!(keypair)
  raise Mastodon::SignatureVerification, "Key #{signature_key_id} is revoked" if keypair.revoked?
  raise Mastodon::SignatureVerification, "Key #{signature_key_id} has expired" if keypair.expired?
end
```

**密钥状态**：
- `revoked?`：密钥已被撤销
- `expired?`：密钥已过期

#### 3.4 域名限制在验签阶段的应用

域名限制检查在验签的早期阶段就会执行：

```ruby
def keypair_from_key_id
  key_id = signed_request.key_id
  domain = key_id.start_with?('acct:') ? key_id.split('@').last : key_id

  # 域名限制检查
  if domain_not_allowed?(domain)
    @signature_verification_failure_code = 403
    return
  end

  # ... 继续获取密钥
end
```

这意味着：
1. **受限联邦模式**：只有 `DomainAllow` 列表中的域名可以通过
2. **正常模式**：`DomainBlock` 列表中的域名会被拒绝
3. **返回 403 Forbidden**：明确拒绝来自受限域名的请求

#### 3.5 验签失败的响应码

| 失败原因 | HTTP 状态码 | 说明 |
|---------|-------------|------|
| 签名缺失 | 401 | 请求未签名 |
| 签名格式错误 | 400 | Signature 头部格式不正确 |
| 域名受限 | 403 | 来自被屏蔽或未授权的域名 |
| 密钥不存在 | 401 | 无法获取公钥 |
| 密钥已撤销 | 401 | 密钥已被撤销 |
| 密钥已过期 | 401 | 密钥已过期 |
| 签名验证失败 | 401 | 签名与内容不匹配 |
| 网络错误 | 503 | 获取密钥时发生网络错误 |
| 断路器触发 | 503 | 近期连接失败过多，跳过请求 |

#### 3.6 断路器机制（Stoplight）

为防止频繁请求不可用的远程服务器，系统使用断路器模式：

```ruby
STOPLIGHT_COOL_OFF_TIME = 5.minutes.seconds  # 冷却时间
STOPLIGHT_THRESHOLD = 1                        # 失败阈值

def stoplight_wrapper
  Stoplight(
    "source:#{request.remote_ip}",
    cool_off_time: STOPLIGHT_COOL_OFF_TIME,
    threshold: STOPLIGHT_THRESHOLD,
    tracked_errors: [HTTP::Error, OpenSSL::SSL::SSLError]
  )
end
```

**断路器状态**：
1. **关闭（Closed）**：正常状态，请求可以通过
2. **打开（Open）**：失败次数超过阈值，在冷却时间内拒绝所有请求
3. **半开（Half-Open）**：冷却时间过后，尝试少量请求

**捕获的错误类型**：
- `HTTP::Error`：HTTP 协议错误
- `OpenSSL::SSL::SSLError`：SSL/TLS 错误

### 4. 域名限制处理边界

Mastodon 提供多层域名限制机制，在联邦同步的各个阶段都有应用。

#### 4.1 域名限制类型

**DomainBlock 模型** (`app/models/domain_block.rb`)

```ruby
enum :severity, { silence: 0, suspend: 1, noop: 2 }, validate: true
```

| 严重程度 | 说明 | 影响 |
|---------|------|------|
| `suspend`（暂停） | 完全阻止该域名 | 无法关注、无法投递、内容不可见 |
| `silence`（静默） | 限制该域名的可见性 | 内容不出现在公共时间线，未关注的用户看不到 |
| `noop`（无操作） | 仅记录，不执行限制 | 用于跟踪或准备 future 限制 |

**附加限制**：
- `reject_media`：拒绝来自该域名的媒体附件
- `reject_reports`：拒绝来自该域名的举报

#### 4.2 域名控制辅助模块

**DomainControlHelper** (`app/helpers/domain_control_helper.rb`)

```ruby
def domain_not_allowed?(uri_or_domain)
  return false if uri_or_domain.blank?

  domain = if uri_or_domain.include?('://')
             Addressable::URI.parse(uri_or_domain).host
           else
             uri_or_domain
           end

  if limited_federation_mode?
    !DomainAllow.allowed?(domain)  # 受限模式：只允许白名单
  else
    DomainBlock.blocked?(domain)   # 正常模式：拒绝黑名单
  end
end

def limited_federation_mode?
  Rails.configuration.x.mastodon.limited_federation_mode
end
```

#### 4.3 两种联邦模式

| 模式 | 配置 | 行为 |
|------|------|------|
| **正常模式** | `limited_federation_mode = false` | 允许所有域名，除非在 `DomainBlock` 中 |
| **受限模式** | `limited_federation_mode = true` | 只允许 `DomainAllow` 中的域名 |

#### 4.4 域名限制的应用阶段

域名限制在联邦同步的多个阶段都有应用：

##### 阶段 1：出站投递前检查

**FollowService** 中的检查：

```ruby
def following_not_allowed?
  domain_not_allowed?(@target_account.domain) ||           # 目标域名受限
    @source_account.domain_blocking?(@target_account.domain) # 源用户屏蔽了目标域名
end
```

**用户级域名屏蔽**（`account.domain_blocking?`）：
- 每个用户可以单独屏蔽特定域名
- 这会覆盖实例级别的设置

##### 阶段 2：入站验签前检查

**SignatureVerification** 中的检查：

```ruby
def keypair_from_key_id
  # ...
  if domain_not_allowed?(domain)
    @signature_verification_failure_code = 403
    return
  end
  # ...
end
```

这确保：
- 来自受限域名的所有入站 ActivityPub 请求都被拒绝
- 即使请求签名有效，也会被拒绝

##### 阶段 3：投递失败后的自动限制

**UnavailableDomain** (`app/models/unavailable_domain.rb`)

当投递失败次数超过阈值时，系统会自动将域名标记为不可用：

```ruby
class UnavailableDomain < ApplicationRecord
  include DomainNormalizable
  validates :domain, presence: true, uniqueness: true
end
```

**DeliveryFailureTracker** (`app/lib/delivery_failure_tracker.rb`)

```ruby
FAILURE_THRESHOLDS = {
  days: 7,      # 7 天失败标记为不可用
  minutes: 5,   # 5 分钟失败（用于短期问题）
}.freeze

def track_failure!
  redis.sadd(exhausted_deliveries_key, failure_time)
  UnavailableDomain.create(domain: @host) if reached_failure_threshold?
end

def track_success!
  redis.del(exhausted_deliveries_key)
  UnavailableDomain.find_by(domain: @host)&.destroy
end
```

**自动限制机制**：

1. **失败跟踪**：每个投递失败会被记录到 Redis
2. **阈值检查**：连续 7 天失败后，创建 `UnavailableDomain` 记录
3. **投递跳过**：`DeliveryWorker` 在投递前检查 `UnavailableDomain`
4. **自动恢复**：成功投递后删除 `UnavailableDomain` 记录

##### 阶段 4：投递时的可用性检查

**ActivityPub::DeliveryWorker** (`app/workers/activitypub/delivery_worker.rb`)

```ruby
def perform(json, source_account_id, inbox_url, options = {})
  @options        = options.with_indifferent_access

  # 检查域名是否可用（除非 bypass_availability）
  return unless @options[:bypass_availability] || DeliveryFailureTracker.available?(inbox_url)

  # ... 继续投递
end
```

**bypass_availability 选项**：
- 关注请求使用 `{ 'bypass_availability' => true }`
- 这确保即使目标实例近期不可用，关注请求也会尝试投递
- 普通投递会尊重 `UnavailableDomain` 标记

#### 4.5 投递失败重试策略

**ActivityPub::DeliveryWorker** 的重试配置：

```ruby
sidekiq_options queue: 'push', retry: 16, dead: false

# 自定义重试延迟（带抖动）
sidekiq_retry_in do |count|
  delay  = (count**4) + 15           # 指数退避：16s, 31s, 96s, 271s...
  jitter = rand(0.5 * (count**4))    # 随机抖动，避免惊群效应
  delay + jitter
end
```

**重试参数**：
- 最大重试次数：16 次
- 重试队列：不进入死信队列（`dead: false`）
- 延迟策略：指数退避 + 随机抖动

**响应处理**：

```ruby
def perform_request
  stoplight_wrapper.run do
    request_pool.with(@host) do |http_client|
      build_request(http_client).perform do |response|
        if response_successful?(response)
          @performed = true                              # 成功
        elsif response_error_unsalvageable?(response) || unsalvageable_authorization_failure?(response)
          @unsalvageable = true                         # 不可恢复错误
        else
          raise Mastodon::UnexpectedResponseError, response  # 可恢复错误（触发重试）
        end
      end
    end
  end
end

def unsalvageable_authorization_failure?(response)
  @source_account.permanently_unavailable? && response.code == 401
end
```

**响应分类**：

| 响应类型 | HTTP 码 | 处理方式 |
|---------|---------|----------|
| **成功** | 2xx | 标记成功，记录成功投递 |
| **不可恢复错误** | 404, 410, 401（源账户永久不可用） | 标记为 `unsalvageable`，不重试 |
| **可恢复错误** | 429, 5xx, 其他 | 触发 Sidekiq 重试机制 |

**断路器（Stoplight）**：

```ruby
STOPLIGHT_COOL_OFF_TIME = 60          # 冷却时间 60 秒
STOPLIGHT_FAILURE_THRESHOLD = 10       # 失败阈值 10 次

def stoplight_wrapper
  Stoplight(
    @inbox_url,
    cool_off_time: STOPLIGHT_COOL_OFF_TIME,
    threshold: STOPLIGHT_FAILURE_THRESHOLD
  )
end
```

这为每个 `inbox_url` 维护独立的断路器状态。

#### 4.6 域名限制层级总结

| 层级 | 检查点 | 触发条件 | 影响 |
|------|--------|----------|------|
| **用户级屏蔽** | `account.domain_blocking?` | 用户主动屏蔽域名 | 该用户无法与该域名交互 |
| **实例级暂停** | `DomainBlock.suspend?` | 管理员暂停域名 | 所有用户无法与该域名交互 |
| **实例级静默** | `DomainBlock.silence?` | 管理员静默域名 | 内容可见性受限 |
| **受限联邦模式** | `!DomainAllow.allowed?` | 白名单模式 | 只允许白名单域名 |
| **自动不可用** | `UnavailableDomain` | 连续 7 天投递失败 | 暂停投递，可自动恢复 |

### 5. 入站活动处理边界

当远程实例的 Follow/Block 活动到达本地实例时，系统有严格的处理边界。

#### 5.1 Follow 活动处理

**ActivityPub::Activity::Follow** (`app/lib/activitypub/activity/follow.rb`)

```ruby
def perform
  target_account = account_from_uri(object_uri)

  # 边界检查 1：目标必须存在且是本地账户
  return if target_account.nil? || !target_account.local? || delete_arrived_first?(@json['id'])

  # 边界检查 2：目标是否屏蔽了发起者
  if target_account.blocking?(@account) || target_account.domain_blocking?(@account.domain) || target_account.moved? || target_account.instance_actor?
    reject_follow_request!(target_account)  # 自动拒绝
    return
  end

  # ... 处理关注请求 ...
end

def reject_follow_request!(target_account)
  # 发送 Reject 活动回发起者
  json = serialize_payload(FollowRequest.new(account: @account, target_account: target_account, uri: @json['id']), ActivityPub::RejectFollowSerializer).to_json
  ActivityPub::DeliveryWorker.perform_async(json, target_account.id, @account.inbox_url)
end
```

**自动拒绝条件**：
- 目标账户屏蔽了发起者 (`blocking?`)
- 目标实例屏蔽了发起者的域名 (`domain_blocking?`)
- 目标账户已迁移 (`moved?`)
- 目标是实例演员（`instance_actor?`）

#### 5.2 Block 活动处理

**ActivityPub::Activity::Block** (`app/lib/activitypub/activity/block.rb`)

```ruby
def perform
  target_account = account_from_uri(object_uri)

  # 边界检查：目标必须是本地账户
  return if target_account.nil? || !target_account.local?

  # 处理已有的关注关系
  unless @account.blocking?(target_account)
    UnfollowService.new.call(@account, target_account) if @account.following?(target_account)
    UnfollowService.new.call(target_account, @account) if target_account.following?(@account)
    RejectFollowService.new.call(target_account, @account) if target_account.requested?(@account)

    # 执行屏蔽
    unless delete_arrived_first?(@json['id'])
      BlockWorker.perform_async(@account.id, target_account.id)
      @account.block!(target_account, uri: @json['id'])
    end
  end
end
```

**关键行为**：
- 当远程账户屏蔽本地账户时，自动解除双方的关注关系
- 这确保屏蔽操作的效果在双方都生效
- 屏蔽关系使用 `uri` 字段记录，便于后续的 Undo 处理

### 6. 同步链路总结

#### 关注操作的完整同步链路

```
用户导入/点击关注
    ↓
FollowService.call()
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 前置检查                                                      │
│ 1. target_account.unavailable?     → 目标账户不可用          │
│ 2. domain_not_allowed?(domain)      → 域名受限               │
│ 3. target_account.blocking?(source) → 目标屏蔽了源           │
│ 4. source_account.blocking?(target) → 源屏蔽了目标           │
│ 5. source_account.domain_blocking?  → 源用户屏蔽了目标域名    │
└─────────────────────────────────────────────────────────────┘
    ↓ 检查通过
┌─────────────────────────────────────────────────────────────┐
│ 目标账户类型判断                                              │
│                                                              │
│ 本地账户                     远程 ActivityPub 账户           │
│     ↓                              ↓                         │
│ direct_follow!()           request_follow!()                │
│     ↓                              ↓                         │
│ LocalNotificationWorker    ActivityPub::DeliveryWorker     │
│     ↓                              ↓                         │
│ 本地处理                    POST target_account.inbox_url    │
│                              携带 HTTP Signature             │
└─────────────────────────────────────────────────────────────┘
```

#### 屏蔽操作的完整同步链路

```
用户导入/点击屏蔽
    ↓
BlockService.call()
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 前置处理                                                      │
│ 1. UnfollowService：解除双方关注关系                          │
│ 2. RejectFollowService：拒绝待处理的关注请求                  │
│ 3. 删除通知权限设置                                            │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 创建屏蔽关系                                                   │
│ account.block!(target_account)                               │
│     ↓                                                         │
│ BlockWorker.perform_async()  → 异步清理时间线等              │
│     ↓                                                         │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 远程账户投递                                                   │
│ if !target_account.local? && target_account.activitypub?    │
│     ↓                                                         │
│ ActivityPub::DeliveryWorker.perform_async(                   │
│   build_json(block),      → ActivityPub::BlockSerializer    │
│   block.account_id,                                            │
│   block.target_account.inbox_url                              │
│ )                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### 入站活动处理链路

```
远程实例 POST /inbox
    ↓
ActivityPub::InboxesController.create()
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 前置过滤器                                                    │
│ 1. skip_large_payload        → 超过 MAX_JSON_SIZE 返回 413  │
│ 2. require_actor_signature!  → 验证 HTTP Signature           │
│    └─ SignatureVerification 模块                              │
│       ├─ 检查域名是否受限                                      │
│       ├─ 获取并验证公钥                                        │
│       ├─ 验证签名                                              │
│       └─ 失败返回 401/403/503                                │
└─────────────────────────────────────────────────────────────┘
    ↓ 验证通过
┌─────────────────────────────────────────────────────────────┐
│ 异步处理                                                      │
│ ActivityPub::ProcessingWorker.perform_async(                 │
│   signed_request_actor.id,                                     │
│   body,                                                        │
│   delivered_to_account_id,                                     │
│   actor_type                                                   │
│ )                                                              │
│     ↓                                                          │
│ ActivityPub::ProcessCollectionService                         │
│     ↓                                                          │
│ 根据 type 字段分派到具体处理器：                               │
│ - Follow → ActivityPub::Activity::Follow                      │
│ - Block  → ActivityPub::Activity::Block                       │
│ - Accept/Reject/Undo 等                                       │
└─────────────────────────────────────────────────────────────┘
```

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
| Follow 序列化器 | `app/serializers/activitypub/follow_serializer.rb` |
| Block 序列化器 | `app/serializers/activitypub/block_serializer.rb` |
| 关注服务 | `app/services/follow_service.rb` |
| 屏蔽服务 | `app/services/block_service.rb` |
| 投递 Worker | `app/workers/activitypub/delivery_worker.rb` |
| 签名验证 | `app/controllers/concerns/signature_verification.rb` |
| HTTP 签名 | `app/lib/http_signature_draft.rb` |
| 域名控制 | `app/helpers/domain_control_helper.rb` |
| 域名屏蔽 | `app/models/domain_block.rb` |
| 不可用域名 | `app/models/unavailable_domain.rb` |
| 投递失败跟踪 | `app/lib/delivery_failure_tracker.rb` |
| 入箱控制器 | `app/controllers/activitypub/inboxes_controller.rb` |
| Follow 活动处理 | `app/lib/activitypub/activity/follow.rb` |
| Block 活动处理 | `app/lib/activitypub/activity/block.rb` |
