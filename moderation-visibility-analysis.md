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
| `public` | 0 | 公开 | 公共同步 + 主页 + 列表 + 标签 |
| `unlisted` | 1 | 不公开列出 | 主页 + 列表 + 标签（但不显示在公共同步） |
| `private` | 2 | 仅关注者 | 仅关注者的主页 + 列表 |
| `direct` | 3 | 私信 | 仅提及的用户 |
| `limited` | 4 | 受限 | 仅提及的关注者 |

#### 1.1.2 可见性作用域

```ruby
# app/models/concerns/status/visibility.rb
scope :distributable_visibility, -> { where(visibility: %i(public unlisted)) }
scope :list_eligible_visibility, -> { where(visibility: %i(public unlisted private)) }
scope :not_direct_visibility, -> { where.not(visibility: :direct) }
```

**分发规则**：
- `distributable_visibility`: 可在联邦网络中传播
- `list_eligible_visibility`: 可出现在列表时间线
- `not_direct_visibility`: 非私信可见性

#### 1.1.3 默认可见性

```ruby
# app/models/concerns/status/visibility.rb
def visibility_from_account
  account.locked? ? :private : :public
end
```

- 锁定账号（需审核关注请求）: 默认 `private`
- 普通账号: 默认 `public`

#### 1.1.4 可见性限制

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

### 2.2 时间线过滤机制

#### 2.2.1 FeedManager 核心过滤逻辑

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

#### 2.2.2 主页时间线过滤 (filter_from_home)

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

#### 2.2.3 过滤依赖数据 (crutches)

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

#### 2.2.4 标签时间线过滤

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

### 2.3 公共时间线过滤

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

### 2.4 通知过滤机制

通知过滤是最复杂的过滤层级，包含 `DropCondition` 和 `FilterCondition` 两个阶段。

#### 2.4.1 通知策略模型

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

#### 2.4.2 通知服务主流程

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

#### 2.4.3 DropCondition (完全丢弃)

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

#### 2.4.4 FilterCondition (过滤到请求箱)

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

#### 2.4.5 策略覆盖机制

```ruby
def override_for_sender?
  NotificationPermission.exists?(account: @recipient, from_account: @sender)
end
```

用户可以为特定发送者创建 `NotificationPermission`，覆盖全局策略。

### 2.5 策略叠加决策流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    帖子分发与过滤完整决策流程                                  │
└─────────────────────────────────────────────────────────────────────────────┘

帖子创建
     │
     ▼
┌─────────────────────┐
│ 检查可见性级别       │
│ public/unlisted/    │
│ private/direct/     │
│ limited             │
└──────────┬──────────┘
           │
           ├──────────────────────────────────────────────────────────────┐
           │                                                              │
           ▼                                                              ▼
┌─────────────────────┐                                    ┌─────────────────────────────┐
│ 可见性确定分发目标    │                                    │ FanOutOnWriteService        │
│                     │                                    │                             │
│ public:             │                                    │ - 投递到关注者               │
│   - 所有关注者       │                                    │ - 投递到列表                 │
│   - 公共同步         │◄───────────────────────────────────│ - 广播到 hashtag 流         │
│   - hashtag 流       │                                    │ - 广播到公共流               │
│                     │                                    │                             │
│ unlisted:           │                                    │ 每个目标执行过滤检查          │
│   - 所有关注者       │                                    │                             │
│   - hashtag 流       │                                    │                             │
│                     │                                    │                             │
│ private:            │                                    │                             │
│   - 仅关注者         │                                    │                             │
│   - 列表             │                                    │                             │
│                     │                                    │                             │
│ direct:             │                                    │                             │
│   - 仅提及用户       │                                    │                             │
│                     │                                    │                             │
│ limited:            │                                    │                             │
│   - 提及的关注者      │                                    │                             │
└─────────────────────┘                                    └─────────────────────────────┘
                                                                    │
                                                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           FeedManager.filter()                                    │
│                                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ 过滤检查顺序（任一条件满足则过滤）                                          │   │
│  │                                                                           │   │
│  │ 1. 基础检查                                                               │   │
│  │    ├─ 自己的帖子？→ 不过滤                                               │   │
│  │    ├─ 无效回复？→ 过滤                                                   │   │
│  │    ├─ 专属列表用户且非列表时间线？→ 跳过主页                              │   │
│  │    ├─ 语言不匹配？→ 过滤                                                 │   │
│  │    └─ 转发的原帖不存在？→ 过滤                                           │   │
│  │                                                                           │   │
│  │ 2. 屏蔽/静音检查（检查发帖者、被提及者、转发原帖作者）                      │   │
│  │    ├─ 我屏蔽了 TA？→ 过滤                                                │   │
│  │    ├─ 我静音了 TA？→ 过滤                                                │   │
│  │    └─ TA 屏蔽了我？→ 过滤                                                │   │
│  │                                                                           │   │
│  │ 3. 域名屏蔽检查                                                           │   │
│  │    └─ 转发时：原帖作者域名被我屏蔽？→ 过滤                                │   │
│  │                                                                           │   │
│  │ 4. 回复过滤                                                               │   │
│  │    过滤条件（同时满足）：                                                  │   │
│  │    ├─ 我没有关注被回复的人                                                 │   │
│  │    ├─ 不是回复给我                                                        │   │
│  │    └─ 不是自回复                                                          │   │
│  │                                                                           │   │
│  │ 5. 转发过滤                                                               │   │
│  │    过滤条件（任一满足）：                                                  │   │
│  │    ├─ 我隐藏了转发者的转发                                                │   │
│  │    ├─ 原帖作者屏蔽了我                                                    │   │
│  │    └─ 原帖作者域名被我屏蔽                                                │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
                                                                    │
                              ┌─────────────────────────────────────┼─────────────────────────────────────┐
                              │                                     │                                     │
                              ▼                                     ▼                                     ▼
                    ┌─────────────────┐                   ┌─────────────────┐                   ┌─────────────────┐
                    │   过滤通过      │                   │   标记过滤      │                   │   过滤被拒绝    │
                    │                 │                   │   (:skip_home)   │                   │   (:filter)      │
                    ▼                 │                   ▼                 │                   ▼                 │
           ┌─────────────────┐      │            ┌─────────────────┐      │            帖子不加入时间线          │
           │  加入时间线      │      │            │ 不加入主页时间线  │      │                                   │
           │  (Redis ZSet)   │      │            │ 但可加入列表时间线│      │                                   │
           └─────────────────┘      │            └─────────────────┘      │                                   │
                                    │                                     │                                     │
                                    └─────────────────────────────────────┴─────────────────────────────────────┘
                                                                 │
                                                                 ▼
                                                    ┌─────────────────────────┐
                                                    │  通知产生时执行额外过滤    │
                                                    │  (NotifyService)         │
                                                    └─────────────┬───────────┘
                                                                  │
                                        ┌─────────────────────────┼─────────────────────────┐
                                        │                         │                         │
                                        ▼                         ▼                         ▼
                              ┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
                              │   Drop 阶段     │       │  Filter 阶段    │       │    Accept       │
                              │   完全丢弃       │       │  通知请求箱      │       │   正常通知       │
                              └─────────────────┘       └─────────────────┘       └─────────────────┘
```

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
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                  │
│  │   User A1    │    │   User A2    │    │   Admin A    │                  │
│  │              │    │              │    │              │                  │
│  │ Blocks:      │    │ Mutes:       │    │ DomainBlocks:│                  │
│  │ - @B1@inst-b │    │ - @A1         │    │ - inst-c     │                  │
│  │              │    │              │    │  (suspend)    │                  │
│  └──────────────┘    └──────────────┘    └──────────────┘                  │
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
│  ┌──────────────┐           │      │  ┌──────────────┐           │      │  ┌──────────────┐           │
│  │   User B1    │           │      │  │   User C1    │           │      │  │   User D1    │           │
│  │              │           │      │  │              │           │      │  │              │           │
│  │ 不知道被 A1  │           │      │  │  完全被 A 隔离│           │      │  │  与 A 正常通信│           │
│  │ 屏蔽         │           │      │  │              │           │      │              │           │
│  └──────────────┘           │      │  └──────────────┘           │      │  └──────────────┘           │
│                             │      │                             │      │                             │
│  本地决策：                   │      │  本地决策：                   │      │  本地决策：                   │
│  - 独立于 A 的决策           │      │  - 不知道被 A 屏蔽          │      │  - 完全独立                  │
│  - 可正常接收 A 的帖子       │      │  - 但被 A 的域名封禁拒绝接收  │      │                              │
└─────────────────────────────┘      └─────────────────────────────┘      └─────────────────────────────┘
```

### 3.2 实例间的边界规则

#### 3.2.1 账号屏蔽的传播

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

#### 3.2.2 域名屏蔽的边界

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

#### 3.2.3 可见性的联邦传播

帖子可见性在创建时确定，通过 ActivityPub 传播时携带 `visibility` 属性：

```ruby
# 帖子可见性影响联邦传播范围

# public / unlisted:
# - 可传播到所有关注者实例
# - 可被转发进一步传播
# - 可出现在公共同步

# private:
# - 仅传播到已确认的关注者实例
# - 转发受限

# direct:
# - 仅传播到提及的用户实例

# limited:
# - 仅传播到提及的关注者实例
```

### 3.3 公共时间线的实例边界

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

### 3.4 远程内容的审核处理

#### 3.4.1 远程帖子删除

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

#### 3.4.2 远程内容更新

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

### 3.5 跨实例决策的独立性

每个实例对内容的审核决策完全独立：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    实例决策独立性示例                                          │
└─────────────────────────────────────────────────────────────────────────────┘

场景：User A @ instance-a.com 发了一个帖子

┌──────────────────────────────────────────────────────────────────────────────┐
│ Instance A (本地)                                                             │
│                                                                               │
│ 帖子状态:                                                                      │
│ - 可见性: public                                                              │
│ - sensitive: false                                                            │
│ - 显示: 正常                                                                  │
│                                                                               │
│ 管理员决策: 无                                                                 │
└──────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ActivityPub (Create)
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ Instance B                                                                   │
│                                                                               │
│ 本地决策:                                                                      │
│ - Admin B: 标记为 sensitive                                                   │
│ - 但不传播到 Instance A                                                        │
│                                                                               │
│ 本地显示:                                                                      │
│ - 显示为敏感内容（需点击展开）                                                 │
│                                                                               │
│ Instance A 不知道被 Instance B 标记                                            │
└──────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ActivityPub (Create)
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ Instance C                                                                   │
│                                                                               │
│ 本地决策:                                                                      │
│ - Admin C: 删除帖子 + 创建 Tombstone                                          │
│ - 阻止该帖子再次出现                                                           │
│                                                                               │
│ 对其他实例的影响:                                                              │
│ - 无，Instance A 和 B 不受影响                                                │
└──────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ActivityPub (Create)
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ Instance D                                                                   │
│                                                                               │
│ 本地决策:                                                                      │
│ - 无，正常显示                                                                 │
│                                                                               │
│ 但:                                                                           │
│ - User D1 屏蔽了 User A                                                       │
│ - 帖子不出现在 D1 的时间线，但仍在公共同步                                     │
└──────────────────────────────────────────────────────────────────────────────┘
```

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

## 5. 关键设计决策分析

### 5.1 为什么采用多层过滤？

| 层级 | 位置 | 目的 | 性能考虑 |
|-----|------|------|---------|
| 可见性 | 分发前 | 基于内容属性的基础路由 | O(1) 检查 |
| 域名屏蔽 | 分发前/查询时 | 实例级别的粗粒度控制 | O(1) 哈希查找 |
| 账号屏蔽 | 分发时 | 用户级别的细粒度控制 | 预加载 crutches |
| 内容审核 | 任意时间 | 管理员的事后干预 | 异步处理 |
| 通知策略 | 通知产生时 | 精细化的通知控制 | 按需计算 |

**设计考虑**：
1. **早过滤**：尽可能在分发链早期过滤，减少后续处理
2. **多层叠加**：不同层级处理不同维度的控制
3. **性能优化**：使用缓存、批量预加载、Redis 操作

### 5.2 为什么屏蔽是单向决策？

```
实例 A ──────屏蔽──────► 实例 B 的账号

影响范围：
- ✓ A 的用户时间线不显示 B 的帖子
- ✓ A 的公共同步不显示 B 的帖子
- ✓ A 不接收 B 的通知
- ✗ B 不知道被 A 屏蔽（除非 ActivityPub 传递）
- ✗ B 的实例不受影响
- ✗ 其他实例不受影响
```

**设计原因**：
1. **联邦自治**：每个实例有权决定自己的用户看到什么
2. **隐私保护**：屏蔽者的决策不需要被屏蔽者同意
3. **减少冲突**：避免实例间因审核标准不同产生矛盾

### 5.3 为什么本地和远程操作不同？

| 操作 | 本地账号 | 远程账号 |
|-----|---------|---------|
| 删除帖子 | 软删除 + ActivityPub Delete | 软删除 + Tombstone |
| 标记敏感 | UpdateStatusService + ActivityPub Update | 仅数据库更新 |
| 账号封禁 | DeleteAccountService + 联邦传播 | 仅本地数据删除 |

**设计原因**：
1. **主权原则**：远程实例拥有其数据的最终控制权
2. **实际限制**：无法强制远程实例接受修改
3. **墓碑机制**：防止已删除内容再次同步

### 5.4 通知的三态设计 (Accept/Filter/Drop)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ 通知策略三态的权衡                                                             │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│ Accept:                                                                       │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ 优点: 实时性好，不遗漏重要通知                                           │ │
│ │ 缺点: 可能造成信息过载                                                   │ │
│ │ 适用: 关注者、熟人、重要对话                                            │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│ Filter:                                                                       │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ 优点: 用户可控，防骚扰同时不遗漏                                         │ │
│ │ 缺点: 需要用户手动确认，增加操作成本                                     │ │
│ │ 适用: 陌生人消息、新账号通知                                            │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│ Drop:                                                                         │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ 优点: 完全消除干扰，用户体验最佳                                         │ │
│ │ 缺点: 可能错过重要信息，无法恢复                                         │ │
│ │ 适用: 已屏蔽账号、确定的垃圾信息                                        │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 5.5 最终一致性 vs 强一致性

Mastodon 选择最终一致性模型的原因：

| 特性 | 强一致性 | 最终一致性 | Mastodon 选择 |
|-----|---------|-----------|--------------|
| 屏蔽后时间线更新 | 同步阻塞 | 异步清理 | 最终一致 |
| 通知删除 | 同步 | 异步 | 最终一致 |
| 缓存更新 | 同步失效 | 下次读取重建 | 最终一致 |

**设计考虑**：
1. **用户体验**：屏蔽操作需要快速响应，不能被慢操作阻塞
2. **可扩展性**：异步处理支持更大规模的用户基数
3. **故障隔离**：单个组件故障不影响核心功能
4. **兜底机制**：查询层过滤确保即使缓存/时间线有旧数据也不会显示

---

## 6. 相关文件索引

### 6.1 可见性控制

| 文件路径 | 功能描述 |
|---------|---------|
| `app/models/concerns/status/visibility.rb` | 可见性级别定义、作用域、默认值 |
| `app/models/status.rb` | Status 模型的过滤作用域 |

### 6.2 账号屏蔽与静音

| 文件路径 | 功能描述 |
|---------|---------|
| `app/models/block.rb` | 账号屏蔽模型 |
| `app/models/mute.rb` | 账号静音模型 |
| `app/services/block_service.rb` | 屏蔽服务主流程 |
| `app/services/after_block_service.rb` | 屏蔽后清理服务 |
| `app/workers/block_worker.rb` | 屏蔽后异步清理 Worker |
| `app/models/account_domain_block.rb` | 用户级域名屏蔽 |

### 6.3 域名屏蔽

| 文件路径 | 功能描述 |
|---------|---------|
| `app/models/domain_block.rb` | 域名屏蔽模型 |
| `app/services/block_domain_service.rb` | 域名屏蔽服务 |
| `app/workers/domain_clear_media_worker.rb` | 域名媒体清理 Worker |

### 6.4 内容审核

| 文件路径 | 功能描述 |
|---------|---------|
| `app/models/admin/moderation_action.rb` | 管理员审核操作 |
| `app/models/tombstone.rb` | 墓碑记录（防止已删除内容重新同步） |
| `app/services/update_status_service.rb` | 状态更新服务 |
| `app/workers/removal_worker.rb` | 内容移除 Worker |

### 6.5 时间线过滤

| 文件路径 | 功能描述 |
|---------|---------|
| `app/lib/feed_manager.rb` | 时间线管理核心，包含过滤逻辑 |
| `app/models/public_feed.rb` | 公共时间线查询逻辑 |
| `app/models/home_feed.rb` | 主页时间线 |
| `app/services/fan_out_on_write_service.rb` | 写入时扇出服务 |
| `app/workers/feed_insert_worker.rb` | 时间线插入 Worker |

### 6.6 通知过滤

| 文件路径 | 功能描述 |
|---------|---------|
| `app/services/notify_service.rb` | 通知服务，包含 Drop/Filter 逻辑 |
| `app/models/notification_policy.rb` | 通知策略模型 |
| `app/models/notification_request.rb` | 通知请求模型 |
| `app/models/notification_permission.rb` | 通知权限覆盖 |

### 6.7 联邦相关

| 文件路径 | 功能描述 |
|---------|---------|
| `app/lib/activitypub/activity/block.rb` | ActivityPub Block 活动处理 |
| `app/services/activitypub/process_status_update_service.rb` | 远程状态更新处理 |

---

## 7. 配置要点

### 7.1 通知策略阈值

| 配置项 | 默认值 | 说明 |
|-------|-------|------|
| `NEW_ACCOUNT_THRESHOLD` | 30 天 | 新账号判断阈值 |
| `NEW_FOLLOWER_THRESHOLD` | 3 天 | 新关注者判断阈值 |

### 7.2 域名屏蔽级别

| 级别 | 影响 |
|-----|------|
| `suspend` | 完全封禁，删除账号数据 |
| `silence` | 静默，不显示在公共同步 |
| `noop` | 仅记录，无实际操作 |

### 7.3 缓存键

| 键模式 | 内容 | 失效时机 |
|-------|------|---------|
| `exclude_account_ids_for:{id}` | 需排除的账号 ID 列表 | Block/Mute 创建/删除 |
| `exclude_domains_for:{id}` | 需排除的域名列表 | AccountDomainBlock 创建/删除 |
| `follow_recommendations/{id}` | 关注推荐 | Block 创建/删除 |

---

## 8. 故障排查与运维建议

### 8.1 常见一致性问题

| 问题 | 可能原因 | 解决方案 |
|-----|---------|---------|
| 屏蔽后仍看到帖子 | BlockWorker 失败/时间线未清理 | 手动执行 `FeedManager.instance.clear_from_home` |
| 通知策略不生效 | 缓存未更新 | 检查 `NotificationPolicy` 记录，确认 `filtered` 标志 |
| 远程内容重新出现 | Tombstone 缺失 | 确认 `Tombstone` 记录存在 |
| 公共同步显示静默账号 | `without_silenced` 作用域未应用 | 检查查询逻辑 |

### 8.2 性能优化建议

1. **缓存预热**：
   - `excluded_from_timeline_account_ids` 是热点缓存
   - 屏蔽/静音操作后缓存失效，下次读取会重建

2. **批量操作**：
   - 域名屏蔽使用 `in_batches.update_all`
   - 时间线清理使用 `find_each` 避免内存溢出

3. **异步处理**：
   - 屏蔽后的清理操作全部异步
   - 不阻塞用户的屏蔽操作响应

### 8.3 监控要点

1. **Sidekiq 队列**：
   - `BlockWorker` 执行情况
   - `FeedInsertWorker` 延迟
   - `RemovalWorker` 失败率

2. **缓存命中率**：
   - `exclude_account_ids_for:*` 键
   - 可通过 Rails.cache.stats 监控

3. **数据库查询**：
   - `Status.not_excluded_by_account` 的使用
   - 避免 N+1 查询

---

## 总结

Mastodon 的内容审核、屏蔽与可见性控制系统设计体现了以下核心原则：

### 1. 分层过滤，各司其职

- **可见性层**：基于内容属性的基础路由
- **实例策略层**：域名级别的粗粒度控制
- **用户策略层**：账号级别的细粒度控制
- **审核操作层**：事后干预能力
- **通知策略层**：精细化的通知控制

### 2. 联邦自治，决策独立

- 每个实例对自己的用户负责
- 屏蔽/审核决策不传递给其他实例
- 远程内容的修改仅影响本地

### 3. 最终一致，兜底保障

- 异步清理保证响应速度
- 缓存失效确保数据新鲜
- 查询层过滤防止旧数据显示
- Tombstone 机制防止内容回退

### 4. 用户体验优先

- 屏蔽操作快速响应
- 通知三态设计平衡干扰和遗漏
- 工作人员消息例外机制
- 通知权限覆盖能力

这种设计既满足了联邦社交网络的去中心化特性，又提供了强大的内容控制和用户保护机制，是 Mastodon 能够大规模运行的关键架构决策之一。
