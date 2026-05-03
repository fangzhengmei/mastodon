# Mastodon 内容审核、屏蔽与可见性控制协作机制分析报告

## 概述

本文档详细分析 Mastodon 平台中内容审核、账号/域名屏蔽和帖子可见性控制三者的协作机制，重点说明策略叠加顺序、联邦场景下本地与远端边界，以及状态同步的一致性保障。

---

## 1. 核心概念与机制

### 1.1 帖子可见性控制

帖子可见性是最基础的访问控制机制，由发帖者在创建时指定。

#### 1.1.1 可见性级别定义

```ruby
# app/models/concerns/status/visibility.rb
enum :visibility,
     { public: 0, unlisted: 1, private: 2, direct: 3, limited: 4 },
     suffix: :visibility,
     validate: true
```

| 级别 | 值 | 描述 | 时间线可见性 |
|-----|---|------|-------------|
| `public` | 0 | 公开 | 公共同步 + 主页 + 列表 + 标签时间线 + 标签流 |
| `unlisted` | 1 | 不公开列出 | 主页 + 列表（**不显示在公共同步、标签时间线、标签流**） |
| `private` | 2 | 仅关注者 | 仅关注者的主页 + 列表（包括远程关注者） |
| `direct` | 3 | 私信 | 仅提及的用户（不投递给任何 followers） |
| `limited` | 4 | 受限 | 仅提及的关注者（不投递给任何 followers） |

#### 1.1.2 可见性作用域

```ruby
# app/models/concerns/status/visibility.rb
scope :distributable_visibility, -> { where(visibility: %i(public unlisted)) }
scope :list_eligible_visibility, -> { where(visibility: %i(public unlisted private)) }
scope :not_direct_visibility, -> { where.not(visibility: :direct) }
```

**分发规则**：
- `distributable_visibility` (`public` + `unlisted`): 可在联邦网络中传播，可出现在公共同步
- `list_eligible_visibility` (`public` + `unlisted` + `private`): 可出现在列表时间线
- `not_direct_visibility`: 非私信可见性

#### 1.1.3 关键方法定义

```ruby
# app/models/concerns/status/visibility.rb:31-33
def distributable?
  public_visibility? || unlisted_visibility?
end

# app/services/fan_out_on_write_service.rb:188-190
def broadcastable?
  @status.public_visibility? && !@status.reblog? && !@account.silenced?
end
```

**重要区别**：
| 方法 | 条件 | 控制内容 |
|-----|------|---------|
| `distributable?` | `public` 或 `unlisted` | 是否可被转发、引用、出现在公共同步查询 |
| `broadcastable?` | **仅** `public` + 非转发 + 账号未被静默 | 是否进入标签时间线、公共流（WebSocket） |

#### 1.1.4 默认可见性

```ruby
# app/models/concerns/status/visibility.rb
def visibility_from_account
  account.locked? ? :private : :public
end
```

- 锁定账号（需审核关注请求）: 默认 `private`
- 普通账号: 默认 `public`

#### 1.1.5 可见性限制

```ruby
validates :visibility, exclusion: { in: %w(direct limited) }, if: :reblog?
```

- 转发（reblog）不能使用 `direct` 或 `limited` 可见性

### 1.2 账号级别屏蔽与静音

#### 1.2.1 屏蔽 (Block)

**模型定义**：
```ruby
# app/models/block.rb
class Block < ApplicationRecord
  belongs_to :account
  belongs_to :target_account, class_name: 'Account'
  
  validates :account_id, uniqueness: { scope: :target_account_id }
  
  after_commit :invalidate_blocking_cache
  after_commit :invalidate_follow_recommendations_cache
end
```

**屏蔽效果**：
1. **双向解除关注**：屏蔽者和被屏蔽者互相取消关注
2. **拒绝关注请求**：被屏蔽者的关注请求被拒绝
3. **时间线过滤**：被屏蔽者的帖子不出现在屏蔽者的时间线
4. **通知过滤**：被屏蔽者的通知被过滤
5. **对话清理**：包含被屏蔽者的对话被移除

**屏蔽服务流程**：
```ruby
# app/services/block_service.rb
def call(account, target_account)
  return if account.id == target_account.id
  
  @account = account
  @target_account = target_account
  
  handle_following_relationships  # 解除双向关注
  handle_collections              # 从收藏集中移除
  NotificationPermission.where(...).destroy_all
  
  block = account.block!(target_account)
  
  BlockWorker.perform_async(account.id, target_account.id)
  create_notification(block) if !target_account.local? && target_account.activitypub?
  
  block
end
```

**屏蔽后处理**：
```ruby
# app/services/after_block_service.rb
def call(account, target_account)
  clear_home_feed!           # 从主页时间线移除
  clear_list_feeds!          # 从列表时间线移除
  clear_notification_requests!  # 清除通知请求
  clear_notifications!       # 清除现有通知
  clear_conversations!       # 清除对话
end
```

#### 1.2.2 静音 (Mute)

**模型定义**：
```ruby
# app/models/mute.rb
class Mute < ApplicationRecord
  include Expireable  # 支持过期时间
  
  belongs_to :account
  belongs_to :target_account, class_name: 'Account'
  
  # 字段:
  # - expires_at: 过期时间（可选）
  # - hide_notifications: 是否隐藏通知（默认 true）
end
```

**静音与屏蔽的区别**：

| 特性 | 屏蔽 (Block) | 静音 (Mute) |
|-----|-------------|-------------|
| 双向解除关注 | 是 | 否 |
| 被对方知道 | ActivityPub 传递 | 仅本地 |
| 可过期 | 否 | 是（可选） |
| 控制通知粒度 | 完全隐藏 | 可选择是否隐藏通知 |

#### 1.2.3 账号域名屏蔽 (AccountDomainBlock)

用户级别的域名屏蔽，与管理员域名屏蔽不同：

```ruby
# app/models/account.rb:400-402
def excluded_from_timeline_domains
  Rails.cache.fetch("exclude_domains_for:#{id}") { domain_blocks.pluck(:domain) }
end
```

**作用**：屏蔽特定域名的所有账号出现在时间线

### 1.3 管理员级别域名屏蔽

#### 1.3.1 屏蔽级别

```ruby
# app/models/domain_block.rb
enum :severity, { silence: 0, suspend: 1, noop: 2 }, validate: true
```

| 级别 | 值 | 描述 |
|-----|---|------|
| `silence` (静默) | 0 | 该域名账号的帖子不显示在公共同步，但可被关注 |
| `suspend` (封禁) | 1 | 完全封禁，删除该域名所有账号数据 |
| `noop` (无操作) | 2 | 仅记录，不执行实际操作 |

#### 1.3.2 附加选项

```ruby
# app/models/domain_block.rb
def policies
  if suspend?
    [:suspend]
  else
    [severity.to_sym, reject_media? ? :reject_media : nil, reject_reports? ? :reject_reports : nil]
      .reject { |policy| policy == :noop }
      .compact
  end
end
```

| 选项 | 描述 |
|-----|------|
| `reject_media` | 拒绝接收该域名的媒体附件 |
| `reject_reports` | 拒绝接收来自该域名的举报 |

#### 1.3.3 域名屏蔽服务

```ruby
# app/services/block_domain_service.rb
def call(domain_block, update: false)
  @domain_block = domain_block
  
  process_domain_block!          # 执行屏蔽
  process_retroactive_updates! if update  # 处理规则变更
  notify_of_severed_relationships!  # 通知受影响用户
end

private

def process_domain_block!
  if domain_block.silence?
    silence_accounts!    # 静默该域名所有账号
  elsif domain_block.suspend?
    suspend_accounts!    # 封禁该域名所有账号
  end
  
  if domain_block.suspend?
    PurgeCustomEmojiWorker.perform_async(blocked_domain)
  elsif domain_block.reject_media?
    DomainClearMediaWorker.perform_async(domain_block.id)
  end
end
```

#### 1.3.4 域名匹配规则

```ruby
# app/models/domain_block.rb
def self.rule_for(domain)
  return if domain.blank?

  uri      = Addressable::URI.new.tap { |u| u.host = domain.strip.delete('/') }
  variants = domain_variants(uri.normalized_host)
  where(domain: variants).by_domain_length.first
rescue Addressable::URI::InvalidURIError, IDN::Idna::IdnaError
  nil
end
```

**特点**：
- 支持子域名匹配（`by_domain_length` 表示选择最精确匹配）
- 支持国际化域名 (IDN)

### 1.4 内容审核机制

#### 1.4.1 管理员审核操作

```ruby
# app/models/admin/moderation_action.rb
TYPES = %w(
  delete
  mark_as_sensitive
).freeze
```

| 操作类型 | 描述 |
|---------|------|
| `delete` | 删除帖子及相关内容 |
| `mark_as_sensitive` | 标记为敏感内容 |

#### 1.4.2 删除操作处理

```ruby
# app/models/admin/moderation_action.rb
def handle_delete!
  ApplicationRecord.transaction do
    delete_statuses!       # 软删除帖子
    delete_collections!    # 删除收藏集
    
    resolve_report!        # 标记举报已解决
    process_strike!(:delete_statuses)  # 记录警告
    
    create_tombstones! unless target_account.local?  # 为远程账号创建墓碑
  end
  
  process_notification!
  
  RemovalWorker.push_bulk(status_ids) { |status_id| [status_id, { 'preserve' => target_account.local?, 'immediate' => !target_account.local? }] }
end
```

**关键处理**：
1. **软删除**：使用 `discard_with_reblogs` 而非真删除
2. **墓碑记录**：远程账号删除时创建 `Tombstone` 记录
3. **异步清理**：`RemovalWorker` 异步清理关联数据

#### 1.4.3 标记敏感内容

```ruby
def handle_mark_as_sensitive!
  mark_statuses_as_sensitive!
  mark_collections_as_sensitive!
  
  resolve_report!
  process_strike!(:mark_statuses_as_sensitive)
  process_notification!
end

def mark_statuses_as_sensitive!
  representative_account = Account.representative
  
  statuses.includes(:media_attachments, ...).find_each do |status|
    next if status.discarded? || !(status.with_media? || status.with_preview_card?)
    
    if target_account.local?
      UpdateStatusService.new.call(status, representative_account.id, sensitive: true)
    else
      status.update(sensitive: true)
    end
  end
end
```

**区别处理**：
- **本地账号**：使用 `UpdateStatusService`，触发完整更新流程（包括联邦传播）
- **远程账号**：直接更新数据库，不传播到原实例

---

## 2. 协作机制与策略叠加顺序

### 2.1 过滤层级概览

Mastodon 的过滤系统采用多层级叠加策略，按以下顺序执行：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           过滤层级（从高到低）                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Level 1: 帖子可见性（发帖者控制）                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ public → unlisted → private → direct → limited                      │  │
│  │ 决定帖子可被哪些时间线接收                                              │  │
│  │ 关键点：                                                                │  │
│  │ - unlisted 不会进入标签时间线和公共流                                  │  │
│  │ - private 会投递给所有 followers（包括远程）                           │  │
│  │ - direct/limited 不会投递给任何 followers                              │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                     ↓                                       │
│  Level 2: 管理员级别控制（实例策略）                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ DomainBlock (suspend/silence)                                        │  │
│  │ - suspend: 完全封禁，账号数据被删除                                    │  │
│  │ - silence: 静默，不显示在公共同步                                      │  │
│  │ - reject_media: 拒绝媒体                                              │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                     ↓                                       │
│  Level 3: 账号级别屏蔽（用户策略）                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ Block (双向)                                                          │  │
│  │ - 解除双向关注                                                         │  │
│  │ - 从所有时间线移除                                                      │  │
│  │ - 通知完全过滤                                                          │  │
│  │                                                                       │  │
│  │ Mute (单向)                                                           │  │
│  │ - 可选择隐藏通知                                                       │  │
│  │ - 可设置过期时间                                                       │  │
│  │ - 不影响关注关系                                                       │  │
│  │                                                                       │  │
│  │ AccountDomainBlock                                                    │  │
│  │ - 用户级域名屏蔽                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                     ↓                                       │
│  Level 4: 内容审核（管理员操作）                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ - 删除帖子 (discard_with_reblogs)                                     │  │
│  │ - 标记敏感 (sensitive: true)                                          │  │
│  │ - 账号封禁/静默                                                        │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                     ↓                                       │
│  Level 5: 通知策略（精细化控制）                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ NotificationPolicy                                                    │  │
│  │ - for_not_following (未关注者)                                        │  │
│  │ - for_not_followers (未关注我的)                                       │  │
│  │ - for_new_accounts (新账号)                                            │  │
│  │ - for_private_mentions (私信)                                          │  │
│  │ - for_limited_accounts (受限账号)                                      │  │
│  │                                                                       │  │
│  │ 每种策略可选: accept / filter / drop                                  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 可见性级别与分发目标的精确映射

#### 2.2.1 FanOutOnWriteService 分发逻辑

```ruby
# app/services/fan_out_on_write_service.rb
def call(status, options = {})
  # ...
  fan_out_to_local_recipients!
  fan_out_to_public_recipients! if broadcastable?  # 注意：只有 broadcastable?
  fan_out_to_public_streams! if broadcastable?      # 注意：只有 broadcastable?
end

def broadcastable?
  @status.public_visibility? && !@status.reblog? && !@account.silenced?
end
```

**关键结论**：只有 `public` 且非转发且账号未被静默的帖子才会：
1. `deliver_to_hashtag_followers!` → 标签时间线
2. `broadcast_to_hashtag_streams!` → 标签流（WebSocket）
3. `broadcast_to_public_streams!` → 公共流

#### 2.2.2 本地接收者分发

```ruby
# app/services/fan_out_on_write_service.rb:50-59
def fan_out_to_local_recipients!
  deliver_to_self!

  unless @options[:skip_notifications]
    notify_quoted_account!      # 通知被引用者（只有 accepted 的 quote）
    notify_mentioned_accounts!
    notify_about_update! if update?
  end

  case @status.visibility.to_sym
  when :public, :unlisted, :private
    deliver_to_all_followers!   # 投递给所有 followers
    deliver_to_lists!           # 投递给列表
  when :limited
    deliver_to_mentioned_followers!  # 仅投递给被提及的关注者
  else  # :direct
    deliver_to_mentioned_followers!
    deliver_to_conversation!
  end
end
```

#### 2.2.3 联邦投递范围

```ruby
# app/lib/status_reach_finder.rb:104-112
def followers_scope
  if @status.in_reply_to_local_account? && distributable?
    # 回复本地账号且可分发：投递给作者和被回复者的 followers
    @status.account.followers.or(@status.thread.account.followers.not_domain_blocked_by_account(@status.account))
  elsif @status.direct_visibility? || @status.limited_visibility?
    # direct 和 limited：不投递给任何 followers
    Account.none
  else
    # public, unlisted, private：投递给所有 followers
    @status.account.followers
  end
end
```

#### 2.2.4 分发目标对照表

| 可见性 | broadcastable? | 标签时间线 | 标签流/公共流 | 本地 followers | 远程 followers | 列表 |
|-------|---------------|-----------|--------------|---------------|---------------|------|
| `public` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `unlisted` | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ |
| `private` | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ |
| `direct` | ✗ | ✗ | ✗ | ✗ (仅提及用户) | ✗ (仅提及用户) | ✗ |
| `limited` | ✗ | ✗ | ✗ | ✗ (仅提及的关注者) | ✗ (仅提及的关注者) | ✗ |

### 2.3 时间线过滤机制

#### 2.3.1 FeedManager 核心过滤逻辑

`FeedManager` 是时间线过滤的核心组件，负责在帖子分发时执行过滤。

```ruby
# app/lib/feed_manager.rb
def filter(timeline_type, status, receiver)
  case timeline_type
  when :home
    filter_from_home(status, receiver.id, build_crutches(receiver.id, [status]), :home)
  when :list
    (filter_from_list?(status, receiver) ? :filter : nil) || filter_from_home(status, receiver.account_id, build_crutches(receiver.account_id, [status], list: receiver), :list)
  when :mentions
    filter_from_mentions?(status, receiver.id) ? :filter : nil
  when :tags
    filter_from_tags?(status, receiver.id, build_crutches(receiver.id, [status])) ? :filter : nil
  end
end
```

#### 2.3.2 主页时间线过滤 (filter_from_home)

这是最复杂的过滤逻辑，包含多层检查：

```ruby
# app/lib/feed_manager.rb:448-479
def filter_from_home(status, receiver_id, crutches, timeline_type = :home)
  # 检查 1: 自己的帖子不过滤
  return if receiver_id == status.account_id
  
  # 检查 2: 无效回复过滤
  return :filter if status.reply? && (status.in_reply_to_id.nil? || status.in_reply_to_account_id.nil?)
  
  # 检查 3: 专属列表用户跳过主页
  return :skip_home if timeline_type != :list && crutches[:exclusive_list_users][status.account_id].present?
  
  # 检查 4: 语言过滤
  return :filter if crutches[:languages][status.account_id].present? && status.language.present? && !crutches[:languages][status.account_id].include?(status.language)
  
  # 检查 5: 转发的原帖不存在
  return :filter if status.reblog? && status.reblog.blank?

  # 检查 6: 屏蔽/静音检查
  check_for_blocks = crutches[:active_mentions][status.id] || []
  check_for_blocks.push(status.account_id)
  
  if status.reblog?
    check_for_blocks.push(status.reblog.account_id)
    check_for_blocks.concat(crutches[:active_mentions][status.reblog_of_id] || [])
  end
  
  # 注意：这里不直接检查 quote 的作者
  # 但如果 quote 创建了 Mention（通过 silent mention），则会被检查
  
  return :filter if check_for_blocks.any? { |target_account_id| 
    crutches[:blocking][target_account_id] || crutches[:muting][target_account_id] 
  }
  
  # 检查 7: 被发帖者屏蔽
  return :filter if crutches[:blocked_by][status.account_id]

  # 检查 8: 回复过滤
  if status.reply? && !status.in_reply_to_account_id.nil?
    # 过滤回复，如果：
    # 1. 我没有关注被回复的人
    # 2. 不是回复给我
    # 3. 不是自回复
    should_filter   = !crutches[:following][status.in_reply_to_account_id]
    should_filter &&= receiver_id != status.in_reply_to_account_id
    should_filter &&= status.account_id != status.in_reply_to_account_id
  
  # 检查 9: 转发过滤
  elsif status.reblog?
    # 过滤转发，如果：
    # 1. 转发者的转发被隐藏
    # 2. 被原帖作者屏蔽
    # 3. 原帖作者域名被屏蔽
    should_filter   = crutches[:hiding_reblogs][status.account_id]
    should_filter ||= crutches[:blocked_by][status.reblog.account_id]
    should_filter ||= crutches[:domain_blocking][status.reblog.account.domain]
  else
    should_filter = false
  end

  should_filter ? :filter : nil
end
```

#### 2.3.3 过滤依赖数据 (crutches)

为了优化性能，`FeedManager` 使用 `build_crutches` 预加载所有需要的过滤数据：

```ruby
# app/lib/feed_manager.rb:625-652
def build_crutches(receiver_id, statuses, list: nil)
  crutches = {}
  
  # 活跃提及
  crutches[:active_mentions] = crutches_active_mentions(statuses)
  
  # 关注关系（用于回复过滤）
  crutches[:following] = crutches_following(receiver_id, statuses, list)
  
  # 语言过滤设置
  crutches[:languages] = Follow.where(...).pluck(:target_account_id, :languages).to_h
  
  # 隐藏转发设置
  crutches[:hiding_reblogs] = Follow.where(...).pluck(:target_account_id).index_with(true)
  
  # 我屏蔽的账号
  crutches[:blocking] = Block.where(...).pluck(:target_account_id).index_with(true)
  
  # 我静音的账号
  crutches[:muting] = Mute.where(...).pluck(:target_account_id).index_with(true)
  
  # 我屏蔽的域名
  crutches[:domain_blocking] = AccountDomainBlock.where(...).pluck(:domain).index_with(true)
  
  # 屏蔽我的账号
  crutches[:blocked_by] = Block.where(...).pluck(:account_id).index_with(true)
  
  # 专属列表用户
  crutches[:exclusive_list_users] = crutches_exclusive_list_users(receiver_id, statuses) if list.blank?
  
  crutches
end
```

#### 2.3.4 标签时间线过滤

```ruby
# app/lib/feed_manager.rb:520-526
def filter_from_tags?(status, receiver_id, crutches)
  receiver_id == status.account_id ||                                      # 自己的帖子
    ((crutches[:active_mentions][status.id] || []) + [status.account_id])   # 提及的账号或发帖者
      .any? { |target_account_id| crutches[:blocking][target_account_id] || crutches[:muting][target_account_id] } ||  # 被屏蔽/静音
    crutches[:blocked_by][status.account_id] ||                            # 被发帖者屏蔽
    crutches[:domain_blocking][status.account.domain]                      # 域名被屏蔽
end
```

### 2.4 Quote (引用) 交互策略与过滤链协作

#### 2.4.1 Quote 模型与状态机

```ruby
# app/models/quote.rb
enum :state,
     { pending: 0, accepted: 1, rejected: 2, revoked: 3, deleted: 4 },
     validate: true
```

| 状态 | 值 | 描述 |
|-----|---|------|
| `pending` | 0 | 等待被引用者同意 |
| `accepted` | 1 | 已接受，可正常显示 |
| `rejected` | 2 | 被拒绝 |
| `revoked` | 3 | 被撤销 |
| `deleted` | 4 | 引用的帖子已删除 |

#### 2.4.2 Quote 可见性约束

```ruby
# app/models/quote.rb:98-102
def validate_visibility
  return if account_id == quoted_account_id || quoted_status.nil? || quoted_status.distributable?
  errors.add(:quoted_status_id, :visibility_mismatch)
end
```

**规则**：
- 可以引用自己的任何帖子
- 引用他人时，只能引用 `distributable?` 的帖子（`public` 或 `unlisted`）

#### 2.4.3 Quote 与时间线过滤链

**关键发现**：`FeedManager#filter_from_home` **不直接检查 Quote 的作者**。

检查的对象：
1. `status.account_id` - 发帖者
2. `status.reblog.account_id` - 转发的原帖作者
3. `crutches[:active_mentions][status.id]` - 被提及的账号

**不直接检查**：
- `status.quote.quoted_account_id` - 被引用的账号

**间接机制**：
- 如果 Quote 创建了 `Mention`（通过 `silent mention` 机制），则被引用者会被包含在 `active_mentions` 中
- 此时屏蔽检查会生效

#### 2.4.4 Quote 与通知过滤

```ruby
# app/services/fan_out_on_write_service.rb:75-79
def notify_quoted_account!
  return unless @status.quote&.quoted_account&.local? && @status.quote&.accepted?
  LocalNotificationWorker.perform_async(@status.quote.quoted_account_id, @status.quote.id, 'Quote', 'quote')
end
```

**规则**：
- 只有 `accepted?` 的 Quote 才会通知被引用者
- 通知类型是 `'Quote'`，会经过 `NotifyService` 的完整过滤链

#### 2.4.5 Quote 与联邦投递

```ruby
# app/lib/status_reach_finder.rb:30-44
def reached_account_ids
  # ...
  quote_of_account_id,      # 被引用的账号（用于投递）
  # ...
  quotes_account_ids,       # 引用我的账号（用于投递更新等）
  # ...
end

# app/lib/status_reach_finder.rb:51-53
def quote_of_account_id
  @status.quote&.quoted_account_id
end

# app/lib/status_reach_finder.rb:63-66
# Beware: Quotes can be created without the author having had access to the status
def quotes_account_ids
  @status.quotes.pluck(:account_id) if distributable? || unsafe?
end
```

**注意**：
- 代码注释："Quotes can be created without the author having had access to the status"
- 这是一个安全提示，说明 Quote 可能在作者没有访问权限的情况下创建
- `quotes_account_ids` 只在 `distributable?` 时返回

#### 2.4.6 Quote 与缓存清理

```ruby
# app/models/media_attachment.rb:220-227
scope :without_local_interaction, lambda {
  # ...
  .where.not(Quote.joins(:status).merge(Status.local).where(Quote.arel_table[:quoted_status_id].eq(MediaAttachment.arel_table[:status_id]).select(1).arel.exists)
  .where.not(Quote.joins(:quoted_status).merge(Status.local).where(Quote.arel_table[:status_id].eq(MediaAttachment.arel_table[:status_id]).select(1).arel.exists)
}
```

**本地交互检查包含**：
1. 本地账号收藏
2. 本地账号书签
3. 本地账号回复
4. 本地账号转发
5. **本地账号引用**
6. **被本地账号引用**

### 2.5 公共时间线过滤

公共时间线使用数据库查询层面的过滤：

```ruby
# app/models/public_feed.rb
def account_filters_scope
  Status.not_excluded_by_account(account).tap do |scope|
    scope.merge!(Status.not_domain_blocked_by_account(account)) unless local_only?
  end
end
```

**作用域定义**：
```ruby
# app/models/status.rb:143-144
scope :not_excluded_by_account, ->(account) { 
  where.not(account_id: account.excluded_from_timeline_account_ids) 
}

scope :not_domain_blocked_by_account, ->(account) { 
  account.excluded_from_timeline_domains.blank? ? 
    left_outer_joins(:account) : 
    left_outer_joins(:account).merge(Account.not_domain_blocked_by_account(account)) 
}
```

**排除列表来源**：
```ruby
# app/models/account.rb:396-402
def excluded_from_timeline_account_ids
  Rails.cache.fetch("exclude_account_ids_for:#{id}") { 
    block_relationships.pluck(:target_account_id) + 
    blocked_by_relationships.pluck(:account_id) + 
    mute_relationships.pluck(:target_account_id) 
  }
end

def excluded_from_timeline_domains
  Rails.cache.fetch("exclude_domains_for:#{id}") { 
    domain_blocks.pluck(:domain) 
  }
end
```

### 2.6 通知过滤机制

通知过滤是最复杂的过滤层级，包含 `DropCondition` 和 `FilterCondition` 两个阶段。

#### 2.6.1 通知策略模型

```ruby
# app/models/notification_policy.rb
enum :for_not_following, { accept: 0, filter: 1, drop: 2 }, suffix: :not_following
enum :for_not_followers, { accept: 0, filter: 1, drop: 2 }, suffix: :not_followers
enum :for_new_accounts, { accept: 0, filter: 1, drop: 2 }, suffix: :new_accounts
enum :for_private_mentions, { accept: 0, filter: 1, drop: 2 }, suffix: :private_mentions
enum :for_limited_accounts, { accept: 0, filter: 1, drop: 2 }, suffix: :limited_accounts
```

| 策略类型 | 触发条件 | 说明 |
|---------|---------|------|
| `for_not_following` | 发送者不是接收者的关注者 | 控制未关注者的通知 |
| `for_not_followers` | 接收者不是发送者的关注者 | 控制未关注我的通知 |
| `for_new_accounts` | 发送者是新账号（< 30天） | 控制新账号通知 |
| `for_private_mentions` | 私信且不是对话回复 | 控制陌生人私信 |
| `for_limited_accounts` | 发送者被静默 | 控制受限账号通知 |

**策略选项**：
- `accept`: 正常接收通知
- `filter`: 放入通知请求箱，需用户确认
- `drop`: 完全丢弃，不产生任何记录

#### 2.6.2 通知服务主流程

```ruby
# app/services/notify_service.rb
def call(recipient, type, activity, **options)
  return if recipient.user.nil?

  @notification = Notification.new(account: @recipient, type: type, activity: @activity)

  # 第一阶段: Drop - 完全丢弃
  return if drop?

  # 第二阶段: Filter - 标记为过滤
  @notification.filtered = filter?
  @notification.set_group_key!
  @notification.save!

  # 处理结果
  if @notification.filtered?
    update_notification_request!  # 创建通知请求
  else
    push_notification!            # 推送通知
    push_to_conversation! if direct_message?
    send_email! if email_needed?
  end
end
```

#### 2.6.3 DropCondition (完全丢弃)

```ruby
# app/services/notify_service.rb:102-163
class DropCondition < BaseCondition
  def drop?
    # 基础检查
    blocked   = @recipient.unavailable?
    blocked ||= from_self? && %i(poll severed_relationships moderation_warning annual_report).exclude?(@notification.type)

    return blocked if message? && from_staff?  # 工作人员消息例外

    # 屏蔽检查
    blocked ||= domain_blocking?
    blocked ||= @recipient.blocking?(@sender)
    blocked ||= @recipient.muting_notifications?(@sender)
    blocked ||= conversation_muted?
    blocked ||= blocked_mention? if message?

    return true if blocked

    # 策略检查
    return false unless filterable_type?
    return false if override_for_sender?

    blocked_by_limited_accounts_policy? ||
      blocked_by_not_following_policy? ||
      blocked_by_not_followers_policy? ||
      blocked_by_new_accounts_policy? ||
      blocked_by_private_mentions_policy?
  end

  private

  def blocked_mention?
    FeedManager.instance.filter?(:mentions, @notification.target_status, @recipient)
  end

  def domain_blocking?
    @recipient.domain_blocking?(@sender.domain) && not_following?
  end
end
```

**丢弃检查顺序**：
1. 接收者账号不可用
2. 自己给自己的通知（特定类型除外）
3. 域名被屏蔽且未关注
4. 账号被屏蔽
5. 通知被静音
6. 对话被静音
7. 提及被时间线过滤
8. 通知策略规则

#### 2.6.4 FilterCondition (过滤到请求箱)

```ruby
# app/services/notify_service.rb:165-199
class FilterCondition < BaseCondition
  def filter?
    return false unless filterable_type?
    return false if override_for_sender?
    return false if message? && from_staff?

    filtered_by_limited_accounts_policy? ||
      filtered_by_not_following_policy? ||
      filtered_by_not_followers_policy? ||
      filtered_by_new_accounts_policy? ||
      filtered_by_private_mentions_policy?
  end
end
```

**关键例外**：
- 非可过滤类型不检查
- 已有 `NotificationPermission` 的发送者例外
- 工作人员的消息例外

#### 2.6.5 策略覆盖机制

```ruby
def override_for_sender?
  NotificationPermission.exists?(account: @recipient, from_account: @sender)
end
```

用户可以为特定发送者创建 `NotificationPermission`，覆盖全局策略。

---

## 3. 联邦场景下本地与远端边界

### 3.1 联邦网络架构

Mastodon 采用 ActivityPub 协议实现联邦，每个实例独立管理自己的用户和内容。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        联邦网络中的实例边界                                     │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                          实例 A (mastodon.social)                              │
│                                                                               │
│  可见性规则（本地决策）：                                                       │
│  - public: 进入所有时间线，包括标签时间线                                      │
│  - unlisted: 只进入主页和列表，不进入标签时间线                                │
│  - private: 投递给所有 followers（包括远程）                                   │
│  - direct/limited: 不投递给任何 followers                                      │
│                                                                               │
│  本地决策边界：                                                                 │
│  - 所有屏蔽/静音仅影响本实例用户的时间线                                        │
│  - 域名屏蔽仅影响本实例接收来自该域名的内容                                     │
│  - 管理员操作仅对本实例数据库生效                                              │
└──────────────────────────────────────────────────────────────────────────────┘
              ▲                                    ▲                                    ▲
              │                                    │                                    │
    ActivityPub │                          ActivityPub │                          ActivityPub │
     (Create,   │                           (Block,   │                           (Announce, │
      Update)   │                            Follow)   │                             Like)   │
              │                                    │                                    │
┌─────────────┴──────────────┐      ┌─────────────┴──────────────┐      ┌─────────────┴──────────────┐
│      实例 B (inst-b.com)    │      │      实例 C (inst-c.com)    │      │      实例 D (inst-d.com)    │
│                             │      │                             │      │                             │
│  入站可见性转换：             │      │                             │      │                             │
│  - 根据 ActivityPub 的       │      │                             │      │                             │
│    to/cc 解析可见性          │      │                             │      │                             │
│  - direct 可能转换为 limited │      │                             │      │                             │
│    （当有 silent mention 时） │      │                             │      │                             │
│                             │      │                             │      │                             │
│  本地决策：                   │      │  本地决策：                   │      │  本地决策：                   │
│  - 独立于 A 的决策           │      │  - 不知道被 A 屏蔽          │      │  - 完全独立                  │
│  - 可正常接收 A 的帖子       │      │  - 但被 A 的域名封禁拒绝接收  │      │                              │
└─────────────────────────────┘      └─────────────────────────────┘      └─────────────────────────────┘
```

### 3.2 可见性在联邦入站时的解析与转换

#### 3.2.1 入站时的初始解析

```ruby
# app/lib/activitypub/parser/status_parser.rb:101-111
def visibility
  if audience_to.any? { |to| ActivityPub::TagManager.instance.public_collection?(to) }
    :public
  elsif audience_cc.any? { |cc| ActivityPub::TagManager.instance.public_collection?(cc) }
    :unlisted
  elsif audience_to.include?(@options[:followers_collection])
    :private
  else
    :direct  # 默认解析为 direct
  end
end
```

**解析规则**：

| ActivityPub 属性 | 解析为可见性 |
|-----------------|-------------|
| `to` 包含 `Public` 集合 | `public` |
| `cc` 包含 `Public` 集合 | `unlisted` |
| `to` 包含 `followers` 集合 | `private` |
| 其他情况 | `direct` |

#### 3.2.2 direct → limited 的转换条件

```ruby
# app/lib/activitypub/activity/create.rb:124-154
def process_audience
  accounts_in_audience = (audience_to + audience_cc).uniq.filter_map do |audience|
    account_from_uri(audience) unless ActivityPub::TagManager.instance.public_collection?(audience)
  end
  
  # 如果 payload 是投递到特定 inbox 的，添加该账号
  if @options[:delivered_to_account_id]
    accounts_in_audience << delivered_to_account
    accounts_in_audience.uniq!
  end
  
  accounts_in_audience.each do |account|
    # 如果不是显式提及，则创建 silent mention
    next if @mentions.any? { |mention| mention.account_id == account.id }
    @mentions << Mention.new(account: account, silent: true)
    
    # 关键：当有 silent mention 时，direct 转换为 limited
    @params[:visibility] = :limited if @params[:visibility] == :direct
  end
end

# app/lib/activitypub/activity/create.rb:156-165
def postprocess_audience_and_deliver
  return if @status.mentions.find_by(account_id: @options[:delivered_to_account_id])
  
  @status.mentions.create(account: delivered_to_account, silent: true)
  @status.update(visibility: :limited) if @status.direct_visibility?  # 再次确认转换
  # ...
end
```

#### 3.2.3 转换流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    direct → limited 转换流程                                   │
└─────────────────────────────────────────────────────────────────────────────┘

入站 ActivityPub 帖子
     │
     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 阶段 1: 初始解析 (StatusParser#visibility)                                     │
│                                                                               │
│ 检查 to/cc 属性：                                                             │
│ ├─ to 包含 Public → public                                                    │
│ ├─ cc 包含 Public → unlisted                                                  │
│ ├─ to 包含 followers_collection → private                                     │
│ └─ 其他 → direct (默认)                                                        │
└──────────────────────────────────────────────────────────────────────────────┘
     │
     ▼ (如果解析为 direct)
┌──────────────────────────────────────────────────────────────────────────────┐
│ 阶段 2: 处理 audience (process_audience)                                       │
│                                                                               │
│ 收集 audience 中的账号：                                                        │
│ - 从 to 和 cc 中解析出非 Public 的 URI                                          │
│ - 查找本地已知账号 (account_from_uri)                                           │
│ - 如果有 delivered_to_account_id，添加该账号                                    │
│                                                                               │
│ 对每个账号检查：                                                                │
│ ┌─────────────────────────────────────────────────────────────────────────┐  │
│ │ 账号是否已在显式提及中？(Mention tag)                                      │  │
│ │                                                                           │  │
│ │ 否 → 创建 Mention.new(silent: true)                                        │  │
│ │      └─► @params[:visibility] = :limited (如果当前是 direct)              │  │
│ │                                                                           │  │
│ │ 是 → 跳过（显式提及优先级更高）                                             │  │
│ └─────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
     │
     ▼ (帖子创建后)
┌──────────────────────────────────────────────────────────────────────────────┐
│ 阶段 3: 后处理 (postprocess_audience_and_deliver)                              │
│                                                                               │
│ 仅当 delivered_to_account_id 存在时：                                          │
│ - 检查该账号是否已在 mentions 中                                               │
│ - 如果不在，创建 silent mention                                                 │
│ - 如果当前是 direct_visibility?，更新为 limited                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

#### 3.2.4 转换条件总结

**direct → limited 转换触发条件**（任一满足）：

1. **Audience 中包含本地已知账号**：
   - `to` 或 `cc` 中包含本地账号的 URI
   - 且该账号不是通过显式 Mention tag 提及的

2. **直接投递到特定 inbox**：
   - payload 带有 `delivered_to_account_id`
   - 表示该帖子是直接投递到该账号的 inbox
   - 该账号不是显式提及

**核心逻辑**：
- `direct` 表示"仅显式提及的用户"
- `limited` 表示"所有提及的用户（包括 silent mention）"
- 当发现有非显式提及的受众时，`direct` 不够准确，转换为 `limited`

### 3.3 实例间的边界规则

#### 3.3.1 账号屏蔽的传播

```ruby
# app/services/block_service.rb
def create_notification(block)
  ActivityPub::DeliveryWorker.perform_async(
    build_json(block), 
    block.account_id, 
    block.target_account.inbox_url
  )
end
```

**关键点**：
- 屏蔽操作通过 ActivityPub 传递给被屏蔽者
- 被屏蔽者知道自己被屏蔽
- 但屏蔽的过滤效果仅在屏蔽者实例生效

#### 3.3.2 域名屏蔽的边界

域名屏蔽是**纯本地决策**，不传递给其他实例：

```ruby
# app/services/block_domain_service.rb
# 没有 ActivityPub 传递逻辑
```

**域名屏蔽的作用边界**：

| 操作 | 本实例 | 被屏蔽域名实例 | 其他实例 |
|-----|-------|---------------|---------|
| 拒绝接收来自该域名的帖子 | ✓ 生效 | ✗ 不知道 | ✗ 不影响 |
| 删除该域名账号的本地数据 | ✓ 生效 | ✗ 原数据保留 | ✗ 不影响 |
| 公共同步不显示该域名 | ✓ 生效 | ✗ 不影响 | ✗ 不影响 |

### 3.4 公共时间线的实例边界

```ruby
# app/models/public_feed.rb
def public_scope
  Status.public_visibility.joins(:account).merge(Account.without_suspended.without_silenced)
end
```

**公共时间线过滤规则**：

| 条件 | 说明 |
|-----|------|
| `public_visibility` | 仅公开可见性的帖子 |
| `without_suspended` | 排除被封禁的账号 |
| `without_silenced` | 排除被静默的账号 |

**静默 (silence) 的联邦特性**：
- 本地实例：静默账号的帖子不出现在公共同步
- 远程实例：不知情，可正常显示在其公共同步
- 关注者：静默账号的帖子仍可出现在关注者时间线

### 3.5 远程内容的审核处理

#### 3.5.1 远程帖子删除

```ruby
# app/models/admin/moderation_action.rb
def create_tombstones! unless target_account.local?
  (statuses + collections).each { |record| 
    Tombstone.find_or_create_by(
      uri: record.uri, 
      account: target_account, 
      by_moderator: true
    ) 
  }
end
```

**Tombstone (墓碑) 的作用**：
- 记录已被本地管理员删除的远程内容 URI
- 防止相同内容通过联邦再次同步过来
- `by_moderator: true` 标记是管理员操作

#### 3.5.2 远程内容更新

```ruby
# app/models/admin/moderation_action.rb
def mark_statuses_as_sensitive!
  if target_account.local?
    UpdateStatusService.new.call(status, representative_account.id, sensitive: true)
  else
    status.update(sensitive: true)  # 仅更新本地数据库
  end
end
```

**关键区别**：
- **本地账号**：使用 `UpdateStatusService`，触发 ActivityPub `Update` 活动
- **远程账号**：直接更新数据库，不传播到原实例

**影响**：
- 本地实例：显示为敏感内容
- 原实例：不知道被标记，仍显示正常
- 其他实例：不受影响

---

## 4. 状态同步的一致性保障

### 4.1 缓存失效机制

#### 4.1.1 屏蔽关系缓存

```ruby
# app/models/block.rb
after_commit :invalidate_blocking_cache

def invalidate_blocking_cache
  Rails.cache.delete("exclude_account_ids_for:#{account_id}")
  Rails.cache.delete("exclude_account_ids_for:#{target_account_id}")
end
```

**缓存键**：
- `exclude_account_ids_for:{account_id}`: 该账号需排除的账号 ID
- `exclude_domains_for:{account_id}`: 该账号需排除的域名

#### 4.1.2 排除列表计算

```ruby
# app/models/account.rb:396-402
def excluded_from_timeline_account_ids
  Rails.cache.fetch("exclude_account_ids_for:#{id}") { 
    block_relationships.pluck(:target_account_id) +  # 我屏蔽的
    blocked_by_relationships.pluck(:account_id) +    # 屏蔽我的
    mute_relationships.pluck(:target_account_id)      # 我静音的
  }
end
```

**缓存包含**：
1. 我屏蔽的账号
2. 屏蔽我的账号
3. 我静音的账号

### 4.2 数据库事务保障

#### 4.2.1 屏蔽操作事务

```ruby
# app/models/admin/moderation_action.rb
def handle_delete!
  ApplicationRecord.transaction do
    delete_statuses!
    delete_collections!
    
    resolve_report!
    process_strike!(:delete_statuses)
    
    create_tombstones! unless target_account.local?
  end
  
  RemovalWorker.push_bulk(status_ids) { ... }
end
```

**事务边界**：
- 数据库操作在事务内执行
- 异步任务 (`RemovalWorker`) 在事务外，确保数据持久化后再执行

#### 4.2.2 域名屏蔽的批处理

```ruby
# app/services/block_domain_service.rb
def silence_accounts!
  blocked_domain_accounts.without_silenced.in_batches.update_all(silenced_at: @domain_block.created_at)
end

def suspend_accounts!
  blocked_domain_accounts.without_suspended.in_batches.update_all(...)
  
  blocked_domain_accounts.where(...).reorder(nil).find_each do |account|
    DeleteAccountService.new.call(account, ...)
  end
end
```

**批处理策略**：
- `in_batches.update_all`: 高效批量更新数据库
- `find_each`: 逐条处理需要复杂操作的记录

### 4.3 异步任务队列

#### 4.3.1 屏蔽后清理

```ruby
# app/workers/block_worker.rb
class BlockWorker
  include Sidekiq::Worker
  
  def perform(account_id, target_account_id)
    AfterBlockService.new.call(
      Account.find(account_id),
      Account.find(target_account_id)
    )
  end
end
```

**AfterBlockService 执行**：
```ruby
# app/services/after_block_service.rb
def call(account, target_account)
  clear_home_feed!           # 从主页时间线移除
  clear_list_feeds!          # 从列表时间线移除
  clear_notification_requests!
  clear_notifications!
  clear_conversations!
end
```

#### 4.3.2 时间线清理

```ruby
# app/lib/feed_manager.rb:231-245
def clear_from_home(account, target_account)
  timeline_key        = key(:home, account.id)
  timeline_status_ids = redis.zrange(timeline_key, 0, -1)
  statuses            = Status.where(id: timeline_status_ids).select(:id, :reblog_of_id, :account_id).to_a
  reblogged_ids       = Status.where(...).pluck(:id)
  with_mentions_ids   = Mention.active.where(...).pluck(:status_id)

  target_statuses = statuses.select do |status|
    # 检查：发帖者是目标、转发目标的帖子、提及目标
    status.account_id == target_account.id || 
    reblogged_ids.include?(status.reblog_of_id) || 
    with_mentions_ids.include?(status.id) || 
    with_mentions_ids.include?(status.reblog_of_id)
  end

  target_statuses.each do |status|
    unpush_from_home(account, status)
  end
end
```

**清理范围**：
1. 目标账号直接发布的帖子
2. 目标账号被转发的帖子
3. 提及目标账号的帖子

### 4.4 最终一致性保障

#### 4.4.1 多组件协同

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    屏蔽操作的一致性流程                                         │
└─────────────────────────────────────────────────────────────────────────────┘

用户点击 "屏蔽"
     │
     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ BlockService#call (同步)                                                       │
│                                                                               │
│ 1. handle_following_relationships                                             │
│    - 解除双向关注关系                                                          │
│    - 拒绝待处理的关注请求                                                      │
│                                                                               │
│ 2. handle_collections                                                         │
│    - 从收藏集中移除                                                            │
│                                                                               │
│ 3. NotificationPermission.where(...).destroy_all                              │
│    - 清除通知权限                                                              │
│                                                                               │
│ 4. account.block!(target_account)                                             │
│    - 创建 Block 记录                                                          │
│    - 触发 after_commit 回调                                                   │
│                                                                               │
│ 5. BlockWorker.perform_async(account.id, target_account.id)                  │
│    - 异步清理时间线和通知                                                      │
│                                                                               │
│ 6. create_notification(block) if remote?                                      │
│    - 传递 ActivityPub Block 活动                                              │
└──────────────────────────────────────────────────────────────────────────────┘
     │
     ├──────────────────────┐
     │                      │
     ▼                      ▼
┌─────────────────┐  ┌─────────────────────────────────────────────────────┐
│ after_commit    │  │ BlockWorker (异步)                                    │
│ 缓存失效        │  │                                                       │
└─────────────────┘  │ 1. AfterBlockService#call                            │
                     │    - clear_home_feed! (Redis ZSet 移除)              │
                     │    - clear_list_feeds!                                 │
                     │    - clear_notification_requests!                      │
                     │    - clear_notifications! (数据库删除)                 │
                     │    - clear_conversations!                              │
                     └─────────────────────────────────────────────────────┘
```

#### 4.4.2 缓存与数据库的一致性

```ruby
# 读取时使用缓存
def excluded_from_timeline_account_ids
  Rails.cache.fetch("exclude_account_ids_for:#{id}") { 
    # 缓存未命中时从数据库计算
    block_relationships.pluck(:target_account_id) +
    blocked_by_relationships.pluck(:account_id) +
    mute_relationships.pluck(:target_account_id)
  }
end

# 写入时失效缓存
# app/models/block.rb
after_commit :invalidate_blocking_cache

def invalidate_blocking_cache
  Rails.cache.delete("exclude_account_ids_for:#{account_id}")
  Rails.cache.delete("exclude_account_ids_for:#{target_account_id}")
end
```

**一致性策略**：
- **Cache-Aside 模式**：读取时先查缓存，未命中则查数据库并回填
- **Write-Invalidate 模式**：写入时使缓存失效，下次读取时重建

#### 4.4.3 时间线的最终一致性

时间线（Redis ZSet）与数据库（Status 表）的同步：

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ 正常流程                                                                       │
│                                                                               │
│ 1. 帖子创建                                                                   │
│    ├─ 数据库写入 Status 记录                                                 │
│    └─ FanOutOnWriteService 分发到 Redis 时间线                               │
│                                                                               │
│ 2. 屏蔽操作                                                                   │
│    ├─ 数据库写入 Block 记录                                                  │
│    ├─ 缓存失效                                                               │
│    └─ BlockWorker 异步清理 Redis 时间线                                       │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│ 故障恢复                                                                       │
│                                                                               │
│ 如果 BlockWorker 失败：                                                       │
│ - 数据库的 Block 记录已持久化                                                 │
│ - 缓存已失效                                                                 │
│ - 新的过滤查询会使用新的排除列表                                              │
│ - 但 Redis 时间线中可能还有旧数据                                             │
│                                                                               │
│ 解决方案：                                                                     │
│ 1. Sidekiq 重试机制（默认重试）                                               │
│ 2. 下次访问时的查询层过滤                                                      │
│ 3. 手动触发时间线重建                                                          │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 4.5 查询层的兜底过滤

即使时间线缓存有旧数据，查询时仍会执行过滤：

```ruby
# 时间线读取时的双重保障

# 1. 写入时过滤 (FeedManager.filter?)
# - 帖子分发时决定是否加入时间线

# 2. 查询时过滤 (数据库作用域)
# app/models/public_feed.rb
def get(limit, max_id = nil, since_id = nil, min_id = nil)
  scope = public_scope
  scope.merge!(account_filters_scope) if account?
  # ...
end

# 3. API 层的额外检查
# 即使 Redis 有数据，渲染时也可能被过滤
```

---

## 5. 关键设计决策总结

### 5.1 可见性与分发的关键决策

| 决策 | 设计意图 | 代码位置 |
|-----|---------|---------|
| `unlisted` 不进入标签时间线 | 标签时间线是"发现"机制，unlisted 更适合"低调分享" | `FanOutOnWriteService#broadcastable?` |
| `private` 投递给远程 followers | 锁定账号的关注者也应该能看到帖子 | `StatusReachFinder#followers_scope` |
| `direct/limited` 不投递给任何 followers | 私信和受限帖子应该严格控制受众 | `StatusReachFinder#followers_scope` |
| `broadcastable?` 比 `distributable?` 更严格 | 公共流和标签时间线需要更高的可见性门槛 | `FanOutOnWriteService` |

### 5.2 Quote 交互的设计决策

| 决策 | 设计意图 | 代码位置 |
|-----|---------|---------|
| 只能引用 `distributable?` 的帖子 | 保护私有内容不被公开引用 | `Quote#validate_visibility` |
| 只有 `accepted?` 才通知被引用者 | 引用需要被引用者同意才能产生交互 | `FanOutOnWriteService#notify_quoted_account!` |
| 时间线过滤不直接检查 quote 作者 | 引用是发帖者的行为，不是被引用者的行为 | `FeedManager#filter_from_home` |
| `quotes_account_ids` 只在 `distributable?` 时返回 | 非公开帖子的引用交互不应该被广泛传播 | `StatusReachFinder#quotes_account_ids` |

### 5.3 联邦入站可见性转换的设计决策

| 决策 | 设计意图 | 代码位置 |
|-----|---------|---------|
| 根据 `to/cc` 解析可见性 | 遵循 ActivityPub 规范 | `StatusParser#visibility` |
| `direct` 转换为 `limited` 当有 silent mention | 准确反映实际受众范围 | `Activity::Create#process_audience` |
| `silent: true` 的 Mention | 区分显式提及和隐式受众 | `Activity::Create#process_audience` |
| `delivered_to_account_id` 特殊处理 | 处理直接投递的场景 | `Activity::Create#postprocess_audience_and_deliver` |

---

## 6. 相关文件索引

### 6.1 可见性控制

| 文件路径 | 功能描述 |
|---------|---------|
| `app/models/concerns/status/visibility.rb` | 可见性级别定义、作用域、`distributable?` 方法 |
| `app/services/fan_out_on_write_service.rb` | 帖子分发逻辑、`broadcastable?` 方法、quote 通知 |
| `app/lib/status_reach_finder.rb` | 联邦投递范围计算、`followers_scope` 方法 |

### 6.2 Quote 交互

| 文件路径 | 功能描述 |
|---------|---------|
| `app/models/quote.rb` | Quote 模型、状态机、可见性验证 |
| `app/lib/activitypub/parser/status_parser.rb` | Quote URI 解析、quote_policy 解析 |
| `app/lib/activitypub/activity/create.rb` | Quote 创建处理 |
| `app/services/activitypub/verify_quote_service.rb` | Quote 验证服务 |

### 6.3 联邦入站处理

| 文件路径 | 功能描述 |
|---------|---------|
| `app/lib/activitypub/parser/status_parser.rb` | 可见性解析逻辑 |
| `app/lib/activitypub/activity/create.rb` | `process_audience` 方法、direct→limited 转换 |
| `app/lib/activitypub/tag_manager.rb` | ActivityPub URI 管理 |

### 6.4 屏蔽与过滤

| 文件路径 | 功能描述 |
|---------|---------|
| `app/models/block.rb` | 账号屏蔽模型 |
| `app/models/mute.rb` | 账号静音模型 |
| `app/models/domain_block.rb` | 域名屏蔽模型 |
| `app/lib/feed_manager.rb` | 时间线过滤核心逻辑 |
| `app/services/notify_service.rb` | 通知过滤逻辑 |

### 6.5 内容审核

| 文件路径 | 功能描述 |
|---------|---------|
| `app/models/admin/moderation_action.rb` | 管理员审核操作 |
| `app/models/tombstone.rb` | 墓碑记录 |

---

## 7. 配置要点

### 7.1 可见性相关方法对比

| 方法 | 条件 | 控制内容 |
|-----|------|---------|
| `distributable?` | `public` 或 `unlisted` | 是否可被转发、引用、出现在公共同步查询 |
| `broadcastable?` | **仅** `public` + 非转发 + 账号未被静默 | 是否进入标签时间线、公共流（WebSocket） |
| `list_eligible_visibility` | `public` + `unlisted` + `private` | 是否可出现在列表时间线 |
| `distributable_visibility` | `public` + `unlisted` | 是否可在联邦网络中广泛传播 |

### 7.2 可见性级别分发对照表

| 可见性 | 本地 followers | 远程 followers | 主页时间线 | 列表 | 标签时间线 | 公共流 |
|-------|---------------|---------------|-----------|------|-----------|--------|
| `public` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `unlisted` | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ |
| `private` | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ |
| `direct` | ✗ (仅提及用户) | ✗ (仅提及用户) | ✗ (仅对话) | ✗ | ✗ | ✗ |
| `limited` | ✗ (仅提及的关注者) | ✗ (仅提及的关注者) | ✗ (仅提及的关注者) | ✗ | ✗ | ✗ |

### 7.3 Quote 状态转换

| 状态 | 可被引用者看到 | 通知被引用者 | 可被交互 |
|-----|---------------|-------------|---------|
| `pending` | 否 | 否 | 否 |
| `accepted` | 是 | 是 | 是 |
| `rejected` | 否 | 否 | 否 |
| `revoked` | 否 | 否 | 否 |
| `deleted` | 否 | 否 | 否 |

---

## 总结

本文档详细分析了 Mastodon 中内容审核、账号/域名屏蔽和帖子可见性控制三者的协作机制，特别纠正和补充了以下四个关键点：

### 1. unlisted 与标签时间线

**关键纠正**：`unlisted` **不会**进入标签时间线。

- `broadcastable?` 方法只检查 `public_visibility?`
- 只有 `public` 且非转发且账号未被静默的帖子才会：
  - `deliver_to_hashtag_followers!` → 标签时间线
  - `broadcast_to_hashtag_streams!` → 标签流（WebSocket）
  - `broadcast_to_public_streams!` → 公共流

### 2. private 在联邦投递中的实际边界

**关键纠正**：`private` 会投递给**所有 followers**（包括远程），但 `direct` 和 `limited` 不会。

```ruby
# app/lib/status_reach_finder.rb:104-112
def followers_scope
  # ...
  elsif @status.direct_visibility? || @status.limited_visibility?
    Account.none  # 不投递给任何 followers
  else
    @status.account.followers  # public, unlisted, private 都投递给所有 followers
  end
end
```

### 3. direct 与 limited 在联邦入站时的转换条件

**关键补充**：`direct` 转换为 `limited` 的触发条件。

**入站解析规则**：
- `to` 包含 Public → `public`
- `cc` 包含 Public → `unlisted`
- `to` 包含 followers_collection → `private`
- 其他 → `direct`（默认）

**转换条件**（任一满足）：
1. **Audience 中包含本地已知账号**且不是显式提及
2. **直接投递到特定 inbox**（`delivered_to_account_id` 存在）

**核心逻辑**：
- 创建 `Mention.new(silent: true)` 表示隐式受众
- `direct` 表示"仅显式提及的用户"
- `limited` 表示"所有提及的用户（包括 silent mention）"
- 当发现有非显式提及的受众时，`direct` 不够准确，转换为 `limited`

### 4. quote 交互策略与过滤链协作

**关键补充**：Quote 如何与现有过滤链协作。

**Quote 状态机**：
- `pending` → `accepted` / `rejected` → `revoked` / `deleted`

**可见性约束**：
- 可以引用自己的任何帖子
- 引用他人时，只能引用 `distributable?` 的帖子（`public` 或 `unlisted`）

**与时间线过滤链协作**：
- `FeedManager#filter_from_home` **不直接检查** Quote 的作者
- 检查的对象：发帖者、转发原帖作者、被提及的账号
- **间接机制**：如果 Quote 创建了 `Mention`（通过 `silent mention` 机制），则被引用者会被包含在 `active_mentions` 中，此时屏蔽检查会生效

**与通知过滤协作**：
- 只有 `accepted?` 的 Quote 才会通知被引用者
- 通知类型是 `'Quote'`，会经过 `NotifyService` 的完整过滤链

**与联邦投递协作**：
- `quote_of_account_id` 包含在 `reached_account_ids` 中
- `quotes_account_ids` 只在 `distributable?` 时返回
- 代码注释提示安全风险："Quotes can be created without the author having had access to the status"

---

## 最终确认

### 四个关键点的完整总结

| 关键点 | 核心结论 | 关键代码位置 |
|--------|---------|-------------|
| **unlisted 与标签时间线** | `unlisted` **不会**进入标签时间线。只有 `broadcastable?`（仅 `public` + 非转发 + 账号未被静默）的帖子才会进入标签时间线、标签流和公共流。 | `FanOutOnWriteService#broadcastable?` |
| **private 在联邦投递中的实际边界** | `private` 会投递给**所有 followers**（包括远程实例的关注者）。但 `direct` 和 `limited` 不会投递给任何 followers（返回 `Account.none`）。 | `StatusReachFinder#followers_scope` |
| **direct 与 limited 在联邦入站时的转换条件** | 入站时根据 `to/cc` 解析为 `direct` 后，会在以下情况转换为 `limited`：<br>1. Audience 中包含本地已知账号且不是显式提及（创建 `silent: true` 的 Mention）<br>2. 直接投递到特定 inbox（`delivered_to_account_id` 存在） | `ActivityPub::Activity::Create#process_audience` |
| **quote 交互策略与过滤链协作** | Quote 有独立的状态机和可见性约束。时间线过滤不直接检查 quote 作者，但通过 `Mention` 间接检查；只有 `accepted?` 的 quote 才通知被引用者。 | `Quote#validate_visibility`, `FanOutOnWriteService#notify_quoted_account!` |