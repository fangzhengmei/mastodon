# Mastodon Fan-Out 机制分析报告

## 目录
1. [概述](#概述)
2. [关注关系建立流程](#关注关系建立流程)
3. [时间线 Fan-Out 机制](#时间线-fan-out-机制)
4. [通知生成机制](#通知生成机制)
5. [本地 vs 远端关注者处理差异](#本地-vs-远端关注者处理差异)
6. [数据流架构图](#数据流架构图)
7. [关键代码位置索引](#关键代码位置索引)

---

## 概述

Mastodon 的时间线推送（Fan-Out）机制是其核心功能之一，负责在用户发布新帖后，将帖子高效地分发给所有关注者。该机制采用了**写时扇出（Fan-out on Write）**策略，结合了 Redis 有序集合实现高性能时间线存储，并通过异步 Worker 处理分布式场景下的本地和远端关注者。

### 核心组件

| 组件 | 职责 |
|------|------|
| `FollowService` | 处理关注关系建立 |
| `MergeWorker` | 关注时合并历史帖子到时间线 |
| `PostStatusService` | 处理新帖子发布 |
| `DistributionWorker` | 本地时间线分发入口 |
| `FanOutOnWriteService` | 本地时间线扇出核心逻辑 |
| `FeedInsertWorker` | 单个关注者时间线插入 |
| `FeedManager` | Redis 时间线管理（单例） |
| `ActivityPub::DistributionWorker` | 远端 ActivityPub 分发 |
| `StatusReachFinder` | 确定远端目标收件箱 |
| `LocalNotificationWorker` | 本地通知处理 |
| `NotifyService` | 通知创建和推送 |

---

## 关注关系建立流程

### 1. 关注请求入口

当用户 A 关注用户 B 时，`FollowService#call` 处理请求：

```ruby
# app/services/follow_service.rb:38-43
if (@target_account.locked? && !@options[:bypass_locked]) || @source_account.silenced? || @target_account.activitypub?
  request_follow!  # 需要请求批准或目标是远程账户
elsif @target_account.local?
  direct_follow!   # 本地账户直接关注
end
```

### 2. 两种关注模式

#### 模式 A：直接关注（本地账户）

```ruby
# app/services/follow_service.rb:80-89
def direct_follow!
  # 1. 创建 Follow 记录
  follow = @source_account.follow!(@target_account, **follow_options)
  
  # 2. 给被关注者发送关注通知
  LocalNotificationWorker.perform_async(@target_account.id, follow.id, follow.class.name, 'follow')
  
  # 3. 将被关注者的历史帖子合并到关注者时间线
  MergeWorker.perform_async(@target_account.id, @source_account.id, 'home')
  
  # 4. 处理列表时间线
  MergeWorker.push_bulk(@source_account.owned_lists.with_list_account(@target_account).pluck(:id)) do |list_id|
    [@target_account.id, list_id, 'list']
  end
end
```

#### 模式 B：请求关注（远程或私密账户）

```ruby
# app/services/follow_service.rb:68-77
def request_follow!
  # 1. 创建 FollowRequest 记录
  follow_request = @source_account.request_follow!(@target_account, **follow_options)
  
  # 2. 根据目标类型选择通知方式
  if @target_account.local?
    # 本地账户：发送本地通知
    LocalNotificationWorker.perform_async(@target_account.id, follow_request.id, follow_request.class.name, 'follow_request')
  elsif @target_account.activitypub?
    # 远程账户：通过 ActivityPub 发送 Follow 活动
    ActivityPub::DeliveryWorker.perform_async(build_json(follow_request), @source_account.id, @target_account.inbox_url)
  end
end
```

### 3. 历史帖子合并

`MergeWorker` 在关注建立后，将被关注者的历史帖子合并到关注者时间线：

```ruby
# app/workers/merge_worker.rb:25-34
def merge_into_home!(into_account_id)
  @into_account = Account.find(into_account_id)
  
  FeedManager.instance.merge_into_home(@from_account, @into_account)
ensure
  # 标记时间线重建完成
  HomeFeed.new(@into_account).regeneration_finished!
end
```

`FeedManager#merge_into_home` 的核心逻辑：

```ruby
# app/lib/feed_manager.rb:128-150
def merge_into_home(from_account, into_account)
  return unless into_account.user&.signed_in_recently?  # 仅处理活跃用户
  
  timeline_key = key(:home, into_account.id)
  aggregate    = into_account.user&.aggregates_reblogs?
  
  # 1. 获取被关注者的最近帖子（最多 MAX_ITEMS/4）
  query = from_account.statuses.list_eligible_visibility.includes(reblog: :account).limit(FeedManager::MAX_ITEMS / 4)
  
  # 2. 优化：如果时间线已满，只获取比最旧条目新的帖子
  if redis.zcard(timeline_key) >= FeedManager::MAX_ITEMS / 4
    oldest_home_score = redis.zrange(timeline_key, 0, 0, with_scores: true).first.last.to_i
    query = query.where('id > ?', oldest_home_score)
  end
  
  # 3. 过滤并插入每个帖子
  statuses = query.to_a
  crutches = build_crutches(into_account.id, statuses)  # 预加载过滤所需数据
  
  statuses.each do |status|
    next if filter_from_home(status, into_account.id, crutches)  # 应用过滤规则
    add_to_feed(:home, into_account.id, status, aggregate_reblogs: aggregate)
  end
  
  trim(:home, into_account.id)  # 裁剪到最大容量
end
```

### 4. 关注关系数据模型

`Follow` 模型的关键属性和回调：

```ruby
# app/models/follow.rb:18-80
class Follow < ApplicationRecord
  belongs_to :account           # 关注者
  belongs_to :target_account, class_name: 'Account'  # 被关注者
  
  has_one :notification, as: :activity, dependent: :destroy
  
  # 关注选项
  # - notify: 是否开启新帖通知
  # - show_reblogs: 是否显示转发
  # - languages: 语言过滤
  
  # 回调
  after_create :increment_cache_counters    # 增加关注/粉丝计数
  after_destroy :decrement_cache_counters   # 减少关注/粉丝计数
  after_commit :invalidate_follow_recommendations_cache
  after_commit :invalidate_hash_cache
end
```

---

## 时间线 Fan-Out 机制

### 1. 新帖子发布入口

当用户发布新帖时，`PostStatusService` 触发分发流程：

```ruby
# app/services/post_status_service.rb:162-171
def postprocess_status!
  # ... 其他处理 ...
  
  # 触发本地时间线分发
  DistributionWorker.perform_async(@status.id)
  
  # 触发 ActivityPub 远端分发
  ActivityPub::DistributionWorker.perform_async(@status.id)
  
  # ... 其他处理 ...
end
```

### 2. 本地时间线分发流程

#### 阶段 1：DistributionWorker 入口

```ruby
# app/workers/distribution_worker.rb:3-14
class DistributionWorker
  include Sidekiq::Worker
  include Redisable
  include Lockable

  def perform(status_id, options = {})
    # 使用分布式锁防止重复分发
    with_redis_lock("distribute:#{status_id}") do
      FanOutOnWriteService.new.call(Status.find(status_id), **options.symbolize_keys)
    end
  end
end
```

#### 阶段 2：FanOutOnWriteService 核心扇出

```ruby
# app/services/fan_out_on_write_service.rb:12-25
def call(status, options = {})
  @status    = status
  @account   = status.account
  @options   = options

  return if @status.proper.account.suspended?

  check_race_condition!
  warm_payload_cache!  # 预热序列化缓存

  fan_out_to_local_recipients!   # 本地时间线和通知
  fan_out_to_public_recipients! if broadcastable?  # 标签关注者
  fan_out_to_public_streams! if broadcastable?      # 公共流
end
```

#### 阶段 3：本地接收者分发策略

根据帖子可见性选择不同的分发策略：

```ruby
# app/services/fan_out_on_write_service.rb:41-60
def fan_out_to_local_recipients!
  deliver_to_self!  # 先推送给自己

  unless @options[:skip_notifications]
    notify_quoted_account!
    notify_mentioned_accounts!
    notify_about_update! if update?
  end

  # 根据可见性选择分发范围
  case @status.visibility.to_sym
  when :public, :unlisted, :private
    deliver_to_all_followers!  # 所有关注者
    deliver_to_lists!           # 列表时间线
  when :limited
    deliver_to_mentioned_followers!  # 仅被提及的关注者
  else
    deliver_to_mentioned_followers!  # 私信：仅提及者
    deliver_to_conversation!          # 添加到对话
  end
end
```

#### 阶段 4：批量推送到关注者

```ruby
# app/services/fan_out_on_write_service.rb:114-119
def deliver_to_all_followers!
  # 使用 followers_for_local_distribution 筛选活跃的本地关注者
  @account.followers_for_local_distribution.select(:id).reorder(nil).find_in_batches do |followers|
    # 批量推送 FeedInsertWorker
    FeedInsertWorker.push_bulk(followers) do |follower|
      [@status.id, follower.id, 'home', { 'update' => update? }]
    end
  end
end
```

#### 阶段 5：单条时间线插入

`FeedInsertWorker` 处理单个关注者的时间线插入：

```ruby
# app/workers/feed_insert_worker.rb:7-41
def perform(status_id, id, type = 'home', options = {})
  @type      = type.to_sym
  @status    = Status.find(status_id)
  @options   = options.symbolize_keys

  # 确定接收者
  case @type
  when :home, :tags
    @follower = Account.find(id)
  when :list
    @list     = List.find(id)
    @follower = @list.account
  end

  check_and_insert
end

def check_and_insert
  filter_result = feed_filter  # 应用时间线过滤

  if filter_result
    perform_unpush if update?  # 更新时移除旧版本
  else
    perform_push               # 插入时间线
  end

  perform_notify if notify?(filter_result)  # 处理关注通知
end
```

#### 阶段 6：FeedManager 实际操作 Redis

```ruby
# app/lib/feed_manager.rb:76-83
def push_to_home(account, status, update: false)
  # 优化：仅推送给最近活跃的用户
  return false unless account.user&.signed_in_recently?
  
  # 插入 Redis 有序集合
  return false unless add_to_feed(:home, account.id, status, aggregate_reblogs: account.user&.aggregates_reblogs?)

  trim(:home, account.id)  # 裁剪容量
  
  # 发送流式 API 更新（如果有客户端连接）
  PushUpdateWorker.perform_async(account.id, status.id, "timeline:#{account.id}", { 'update' => update }) if push_update_required?("timeline:#{account.id}")
  true
end
```

### 3. Redis 时间线数据结构

时间线使用 Redis **Sorted Set** 实现，核心设计：

```ruby
# app/lib/feed_manager.rb:29-33
def key(type, id, subtype = nil)
  return "feed:#{type}:#{id}" unless subtype
  
  "feed:#{type}:#{id}:#{subtype}"
end
```

**时间线键设计**：
- `feed:home:{account_id}` - 主页时间线（Sorted Set）
- `feed:list:{list_id}` - 列表时间线（Sorted Set）
- `feed:home:{account_id}:reblogs` - 转发追踪集合
- `feed:home:{account_id}:reblogs:{reblogged_id}` - 同一条内容的多条转发

**核心常量**：
- `MAX_ITEMS = 800` - 单条时间线最大条目数
- `REBLOG_FALLOFF = 40` - 转发去重窗口（最近 40 条内不重复显示同一条内容的转发）

### 4. 时间线过滤机制

`FeedManager#filter_from_home` 实现复杂的过滤逻辑：

```ruby
# app/lib/feed_manager.rb:448-479
def filter_from_home(status, receiver_id, crutches, timeline_type = :home)
  # 1. 自己的帖子不过滤
  return            if receiver_id == status.account_id
  
  # 2. 无效回复过滤
  return :filter    if status.reply? && (status.in_reply_to_id.nil? || status.in_reply_to_account_id.nil?)
  
  # 3. 排他列表过滤（用户将某人加入排他列表后，其帖子不出现在主页）
  return :skip_home if timeline_type != :list && crutches[:exclusive_list_users][status.account_id].present?
  
  # 4. 语言过滤
  return :filter    if crutches[:languages][status.account_id].present? && status.language.present? && !crutches[:languages][status.account_id].include?(status.language)
  
  # 5. 转发的原帖缺失
  return :filter    if status.reblog? && status.reblog.blank?
  
  # 6. 拉黑/静音过滤
  check_for_blocks = crutches[:active_mentions][status.id] || []
  check_for_blocks.push(status.account_id)
  # ... 检查被提及者、转发者等
  
  return :filter if check_for_blocks.any? { |target_account_id| crutches[:blocking][target_account_id] || crutches[:muting][target_account_id] }
  return :filter if crutches[:blocked_by][status.account_id]
  
  # 7. 回复过滤（只显示回复了我关注的人的回复）
  if status.reply? && !status.in_reply_to_account_id.nil?
    should_filter   = !crutches[:following][status.in_reply_to_account_id]  # 我不关注被回复的人
    should_filter &&= receiver_id != status.in_reply_to_account_id         # 不是回复给我
    should_filter &&= status.account_id != status.in_reply_to_account_id   # 不是自回复
  elsif status.reblog?
    # 8. 转发过滤
    should_filter   = crutches[:hiding_reblogs][status.account_id]  # 我屏蔽了此人的转发
    should_filter ||= crutches[:blocked_by][status.reblog.account_id]
    should_filter ||= crutches[:domain_blocking][status.reblog.account.domain]
  end
  
  should_filter ? :filter : nil
end
```

### 5. 过滤辅助数据（Crutches）

为了优化性能，Mastodon 使用 `build_crutches` 预加载所有过滤所需数据：

```ruby
# app/lib/feed_manager.rb:625-652
def build_crutches(receiver_id, statuses, list: nil)
  crutches = {}
  
  crutches[:active_mentions]      = crutches_active_mentions(statuses)
  crutches[:following]            = crutches_following(receiver_id, statuses, list)
  crutches[:languages]            = Follow.where(...).pluck(...).to_h
  crutches[:hiding_reblogs]       = Follow.where(...).pluck(...).index_with(true)
  crutches[:blocking]             = Block.where(...).pluck(...).index_with(true)
  crutches[:muting]               = Mute.where(...).pluck(...).index_with(true)
  crutches[:domain_blocking]      = AccountDomainBlock.where(...).pluck(...).index_with(true)
  crutches[:blocked_by]           = Block.where(...).pluck(...).index_with(true)
  crutches[:exclusive_list_users] = crutches_exclusive_list_users(receiver_id, statuses) if list.blank?
  
  crutches
end
```

---

## 通知生成机制

### 1. 通知类型

Mastodon 支持多种通知类型，定义在 `Notification` 模型：

```ruby
# app/models/notification.rb:26-106
LEGACY_TYPE_CLASS_MAP = {
  'Mention'       => :mention,       # 被提及
  'Status'        => :reblog,        # 被转发
  'Follow'        => :follow,        # 被关注
  'FollowRequest' => :follow_request,# 关注请求
  'Favourite'     => :favourite,     # 被收藏
  'Poll'          => :poll,          # 投票结束
  'Quote'         => :quote,         # 被引用
}.freeze

PROPERTIES = {
  mention: { filterable: true, baseline: true },
  status: { filterable: false, baseline: true },      # 关注者新帖通知
  reblog: { filterable: true, baseline: true },
  follow: { filterable: true, baseline: true },
  follow_request: { filterable: true, baseline: true },
  favourite: { filterable: true, baseline: true },
  poll: { filterable: false, baseline: true },
  update: { filterable: false, baseline: true },       # 帖子编辑更新
  quote: { filterable: true, baseline: true },
  quoted_update: { filterable: false, baseline: true }, # 被引用的帖子更新
  # ... 其他类型
}.freeze
```

### 2. 通知触发时机

通知在多个环节被触发：

#### 场景 A：关注时的通知

```ruby
# app/services/follow_service.rb:83
LocalNotificationWorker.perform_async(@target_account.id, follow.id, follow.class.name, 'follow')
```

#### 场景 B：帖子被提及/转发/收藏

```ruby
# app/services/fan_out_on_write_service.rb:81-97
def notify_mentioned_accounts!
  @status.active_mentions.joins(:account).merge(Account.local).select(:id, :account_id).reorder(nil).find_in_batches do |mentions|
    LocalNotificationWorker.push_bulk(mentions) do |mention|
      [mention.account_id, mention.id, 'Mention', 'mention', options].compact
    end
  end
end
```

#### 场景 C：关注者开启了新帖通知

```ruby
# app/workers/feed_insert_worker.rb:54-59
def notify?(filter_result)
  # 仅当：
  # 1. 是主页时间线
  # 2. 不是转发
  # 3. 不是回复他人（或是自回复）
  # 4. 不是更新操作
  # 5. 没有被过滤
  # 6. 关注关系中开启了 notify 选项
  return false if @type != :home || @status.reblog? || (@status.reply? && @status.in_reply_to_account_id != @status.account_id) ||
                  update? || filter_result == :filter

  Follow.find_by(account: @follower, target_account: @status.account)&.notify?
end
```

### 3. 通知处理流程

#### 阶段 1：LocalNotificationWorker

```ruby
# app/workers/local_notification_worker.rb:3-22
class LocalNotificationWorker
  include Sidekiq::Worker

  def perform(receiver_account_id, activity_id, activity_class_name, type = nil, options = {})
    receiver = Account.find(receiver_account_id)
    activity = activity_class_name.constantize.find(activity_id)

    # 去重逻辑
    if %w(update quoted_update collection_update).include?(type)
      # 更新类型：删除旧通知，创建新通知
      Notification.where(account: receiver, activity: activity, type: type).in_batches.delete_all
    elsif Notification.where(account: receiver, activity: activity, type: type).any?
      # 其他类型：已存在则跳过
      return
    end

    # 调用 NotifyService 创建通知
    NotifyService.new.call(receiver, type || activity_class_name.underscore, activity, **options.symbolize_keys)
  end
end
```

#### 阶段 2：NotifyService 核心逻辑

```ruby
# app/services/notify_service.rb:201-229
def call(recipient, type, activity, **options)
  return if recipient.user.nil?  # 本地用户才有通知

  @options      = options
  @recipient    = recipient
  @activity     = activity
  @notification = Notification.new(account: @recipient, type: type, activity: @activity)

  # 阶段 1：检查是否完全丢弃通知
  return if drop?

  # 阶段 2：检查是否过滤到通知请求
  @notification.filtered = filter?
  @notification.set_group_key!
  @notification.save!

  # 阶段 3：分发通知
  return if @notification.activity.nil?  # 底层活动可能已被删除

  if @notification.filtered?
    update_notification_request!  # 添加到通知请求（需要用户批准）
  else
    push_notification!             # 立即推送
    push_to_conversation! if direct_message?
    send_email! if email_needed?
  end
end
```

### 4. 通知过滤策略

通知有两层过滤：**完全丢弃（Drop）** 和 **过滤到通知请求（Filter）**。

#### 丢弃条件（DropCondition）

```ruby
# app/services/notify_service.rb:102-124
class DropCondition < BaseCondition
  def drop?
    blocked   = @recipient.unavailable?
    blocked ||= from_self? && %i(poll severed_relationships moderation_warning annual_report).exclude?(@notification.type)

    return blocked if message? && from_staff?  # 管理员消息例外

    blocked ||= domain_blocking?
    blocked ||= @recipient.blocking?(@sender)
    blocked ||= @recipient.muting_notifications?(@sender)
    blocked ||= conversation_muted?
    blocked ||= blocked_mention? if message?

    return true if blocked
    return false unless filterable_type?
    return false if override_for_sender?  # 已授予通知权限

    # 用户自定义过滤策略
    blocked_by_limited_accounts_policy? ||
      blocked_by_not_following_policy? ||
      blocked_by_not_followers_policy? ||
      blocked_by_new_accounts_policy? ||
      blocked_by_private_mentions_policy?
  end
end
```

#### 过滤到通知请求（FilterCondition）

```ruby
# app/services/notify_service.rb:165-199
class FilterCondition < BaseCondition
  def filter?
    return false unless filterable_type?
    return false if override_for_sender?
    return false if message? && from_staff?

    # 用户策略：将某些发送者的通知放入"通知请求"
    filtered_by_limited_accounts_policy? ||
      filtered_by_not_following_policy? ||
      filtered_by_not_followers_policy? ||
      filtered_by_new_accounts_policy? ||
      filtered_by_private_mentions_policy?
  end
end
```

### 5. 通知推送通道

```ruby
# app/services/notify_service.rb:249-252
def push_notification!
  push_to_streaming_api! if subscribed_to_streaming_api?  # WebSocket 实时推送
  push_to_web_push_subscriptions!                        # 浏览器/移动 Web Push
end
```

---

## 本地 vs 远端关注者处理差异

### 1. 核心差异概览

| 维度 | 本地关注者 | 远端关注者 |
|------|-----------|-----------|
| **识别方式** | `followers.local` + 活跃用户 | `followers.remote` + ActivityPub inbox |
| **数据存储** | 本地 Redis 时间线 | 远端实例自行管理 |
| **分发协议** | 直接操作 Redis | ActivityPub HTTP POST |
| **实时性** | 亚毫秒级（本地操作） | 网络延迟 + 远端处理 |
| **失败处理** | 无（本地操作） | 重试队列 + 送达追踪 |
| **可见性控制** | 完整过滤逻辑 | 远端实例自行过滤 |

### 2. 本地关注者筛选

```ruby
# app/models/concerns/account/interactions.rb:216-220
def followers_for_local_distribution
  followers.local                    # 仅本地账户
    .joins(:user)
    .merge(User.signed_in_recently)  # 仅最近活跃用户
end
```

**设计考虑**：
- **本地账户**：`domain.nil?` 表示是本实例用户
- **最近活跃**：`signed_in_recently` 优化资源使用，不活跃用户下次登录时重建时间线

### 3. 远端关注者分发

#### 阶段 1：ActivityPub::DistributionWorker

```ruby
# app/workers/activitypub/distribution_worker.rb:9-40
class ActivityPub::DistributionWorker < ActivityPub::RawDistributionWorker
  MAX_FOLLOWERS_FOR_SYNCHRONIZATION = 25_000

  def perform(status_id)
    @status  = Status.find(status_id)
    @account = @status.account

    distribute!
  end

  protected

  def inboxes
    @inboxes ||= StatusReachFinder.new(@status).inboxes
  end

  def payload
    # 序列化为 ActivityPub 格式
    @payload ||= serialize_payload(@status, activity_serializer, ...).to_json
  end

  def activity_serializer
    @status.reblog? ? ActivityPub::AnnounceNoteSerializer : ActivityPub::CreateNoteSerializer
  end

  def options
    { 'synchronize_followers' => @status.private_visibility? && @account.followers_count < MAX_FOLLOWERS_FOR_SYNCHRONIZATION }
  end
end
```

#### 阶段 2：StatusReachFinder 确定目标

```ruby
# app/lib/status_reach_finder.rb:12-117
class StatusReachFinder
  def inboxes
    # 合并三类收件箱
    (reached_account_inboxes + followers_inboxes + relay_inboxes).uniq
  end

  private

  def reached_account_inboxes
    # 有互动的账户：回复、转发、引用、提及、收藏
    scope = Account.where(id: reached_account_ids)
    inboxes_without_suspended_for(scope)
  end

  def reached_account_ids
    if @status.reblog?
      [reblog_of_account_id]  # 转发：仅通知原作者
    else
      [
        replied_to_account_id,   # 被回复者
        reblog_of_account_id,    # 被转发者
        quote_of_account_id,     # 被引用者
        mentioned_account_ids,   # 被提及者
        reblogs_account_ids,     # 转发过此帖的人
        quotes_account_ids,      # 引用过此帖的人
        favourites_account_ids,  # 收藏过此帖的人
        replies_account_ids,     # 回复过此帖的人
      ].flatten.compact.uniq
    end
  end

  def followers_inboxes
    scope = followers_scope
    inboxes_without_suspended_for(scope)
  end

  def followers_scope
    if @status.in_reply_to_local_account? && distributable?
      # 回复本地账户：同时分发给原作者的粉丝
      @status.account.followers.or(@status.thread.account.followers.not_domain_blocked_by_account(@status.account))
    elsif @status.direct_visibility? || @status.limited_visibility?
      Account.none  # 私信不分发给粉丝
    else
      @status.account.followers  # 所有粉丝
    end
  end

  def relay_inboxes
    if @status.public_visibility?
      Relay.enabled.pluck(:inbox_url)  # 中继服务器
    else
      []
    end
  end
end
```

#### 阶段 3：RawDistributionWorker 批量投递

```ruby
# app/workers/activitypub/raw_distribution_worker.rb:23-47
def distribute!
  return if inboxes.empty?

  # 批量投递，每批 1000 个
  ActivityPub::DeliveryWorker.push_bulk(inboxes, limit: 1_000) do |inbox_url|
    [payload, source_account_id, inbox_url, options]
  end
end

def inboxes
  @inboxes ||= @account.followers.inboxes - @exclude_inboxes
end
```

### 4. 共享收件箱优化

ActivityPub 支持 Shared Inbox 来减少 HTTP 请求：

```ruby
# app/models/account.rb:404-406
def preferred_inbox_url
  shared_inbox_url.presence || inbox_url
end
```

```ruby
# app/models/account.rb:419-422
def inboxes
  urls = reorder(nil).activitypub.group(:preferred_inbox_url).pluck(...)
  DeliveryFailureTracker.without_unavailable(urls)
end
```

**优化效果**：同一个实例的 1000 个关注者，只需发送 1 次 HTTP 请求到共享收件箱，而非 1000 次。

### 5. 完整数据流对比

```
新帖发布
    │
    ├───► DistributionWorker (本地)
    │       │
    │       └───► FanOutOnWriteService
    │               │
    │               ├───► 推送到自己的时间线 (Redis)
    │               │
    │               ├───► deliver_to_all_followers!
    │               │       │
    │               │       └───► followers_for_local_distribution (本地活跃用户)
    │               │               │
    │               │               └───► FeedInsertWorker (批量)
    │               │                       │
    │               │                       └───► FeedManager#push_to_home
    │               │                               │
    │               │                               ├───► 检查用户活跃状态
    │               │                               ├───► 应用时间线过滤
    │               │                               ├───► Redis ZADD 插入时间线
    │               │                               └───► PushUpdateWorker (WebSocket)
    │               │
    │               ├───► deliver_to_lists! (列表时间线)
    │               │
    │               ├───► notify_mentioned_accounts!
    │               │       │
    │               │       └───► LocalNotificationWorker
    │               │               │
    │               │               └───► NotifyService
    │               │                       │
    │               │                       ├───► 检查过滤策略
    │               │                       ├───► 创建 Notification 记录
    │               │                       ├───► WebSocket 推送
    │               │                       └───► Web Push / Email
    │               │
    │               └───► fan_out_to_public_streams! (公共流)
    │
    │
    └───► ActivityPub::DistributionWorker (远端)
            │
            └───► StatusReachFinder
                    │
                    ├───► 收集所有远程 inbox
                    │       ├─── followers_inboxes (远程关注者)
                    │       ├─── reached_account_inboxes (互动账户)
                    │       └───► relay_inboxes (中继)
                    │
                    └───► ActivityPub::DeliveryWorker (批量 HTTP POST)
                            │
                            └───► 签名请求
                            └───► 发送到远端 inbox
                            └───► 送达追踪/重试
```

---

## 数据流架构图

### 1. 关注关系建立流程

```
用户A 关注 用户B
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│                    FollowService#call                         │
└─────────────────────────────────────────────────────────────┘
     │
     ├───► 目标是远程账户？ ──Yes──► request_follow!
     │                                    │
     │                                    ├───► 创建 FollowRequest
     │                                    └───► ActivityPub::DeliveryWorker
     │                                                    │
     │                                                    ▼
     │                                              发送 Follow 活动
     │                                              到 远程实例 inbox
     │
     └───► No ──► 目标是本地且公开？ ──Yes──► direct_follow!
                                                         │
                                                         ├───► 创建 Follow 记录
                                                         ├───► LocalNotificationWorker (通知B)
                                                         └───► MergeWorker
                                                                  │
                                                                  ▼
                                                         FeedManager#merge_into_home
                                                                  │
                            ┌─────────────────────────────────────┼─────────────────────────────────────┐
                            ▼                                     ▼                                     ▼
                    获取B的历史帖子                      构建过滤 crutches                    插入到A的时间线
                            │                                     │                                     │
                            ▼                                     ▼                                     ▼
              statuses.list_eligible_visibility        blocking/muting/languages           Redis ZADD
                      limit(MAX_ITEMS/4)               following/hiding_reblogs               │
                                                                                                 ▼
                                                                                           裁剪到 MAX_ITEMS
```

### 2. 新帖 Fan-Out 流程

```
用户B 发布新帖
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│                  PostStatusService#call                       │
└─────────────────────────────────────────────────────────────┘
     │
     ├───► 创建 Status 记录
     ├───► 处理提及/标签
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│                   postprocess_status!                          │
└─────────────────────────────────────────────────────────────┘
     │
     ├───► DistributionWorker.perform_async (本地)
     │
     └───► ActivityPub::DistributionWorker.perform_async (远端)
```

### 3. 本地时间线分发详细流程

```
DistributionWorker
     │
     ▼ (Redis 锁防止重复)
FanOutOnWriteService#call
     │
     ├───► warm_payload_cache! (预序列化)
     │
     ├───► fan_out_to_local_recipients!
     │       │
     │       ├───► deliver_to_self! ────────► FeedManager#push_to_home (自己的时间线)
     │       │
     │       ├───► notify_mentioned_accounts!
     │       │       │
     │       │       └───► LocalNotificationWorker.push_bulk
     │       │               │
     │       │               └───► NotifyService
     │       │                       │
     │       │                       ├───► DropCondition 检查
     │       │                       ├───► FilterCondition 检查
     │       │                       ├───► 创建 Notification
     │       │                       └───► 推送 (WebSocket / WebPush / Email)
     │       │
     │       └───► deliver_to_all_followers!
     │               │
     │               └───► followers_for_local_distribution
     │                       │ (本地 + 最近活跃)
     │                       │
     │                       └───► FeedInsertWorker.push_bulk
     │                               │
     │                               ▼
     │                       FeedInsertWorker#perform
     │                               │
     │                               ├───► 加载 Status 和 Account
     │                               │
     │                               ├───► feed_filter
     │                               │       │
     │                               │       └───► FeedManager#filter
     │                               │               │
     │                               │               ├───► 检查是否自己的帖子
     │                               │               ├───► 检查回复有效性
     │                               │               ├───► 检查语言过滤
     │                               │               ├───► 检查拉黑/静音
     │                               │               ├───► 检查回复过滤
     │                               │               └───► 检查转发过滤
     │                               │
     │                               ├───► 未被过滤？
     │                               │       │
     │                               │       Yes ──► perform_push
     │                               │       │               │
     │                               │       │               ▼
     │                               │       │       FeedManager#push_to_home
     │                               │       │               │
     │                               │       │               ├───► 检查用户活跃
     │                               │       │               ├───► add_to_feed (Redis ZADD)
     │                               │       │               ├───► trim (裁剪到 800)
     │                               │       │               └───► PushUpdateWorker (WebSocket)
     │                               │       │
     │                               │       └───► notify? ──► LocalNotificationWorker
     │                               │
     │                               └───► 被过滤？
     │                                       │
     │                                       └───► 如果是更新操作，执行 unpush
     │
     └───► fan_out_to_public_recipients!
             │
             └───► deliver_to_hashtag_followers!
                     │
                     └───► TagFollow.for_local_distribution
                             │
                             └───► FeedInsertWorker (标签时间线)
```

### 4. 远端分发详细流程

```
ActivityPub::DistributionWorker
     │
     ▼
StatusReachFinder#inboxes
     │
     ├───► reached_account_inboxes (互动账户)
     │       │
     │       ├───► 被回复者
     │       ├───► 被转发者
     │       ├───► 被引用者
     │       ├───► 被提及者
     │       ├───► 转发过此帖的人
     │       ├───► 引用过此帖的人
     │       ├───► 收藏过此帖的人
     │       └───► 回复过此帖的人
     │
     ├───► followers_inboxes (关注者)
     │       │
     │       └───► 根据可见性筛选
     │               │
     │               ├───► 公开/未列出：所有关注者
     │               ├───► 回复本地账户：作者+原作者的关注者
     │               └───► 私信/限定：无
     │
     └───► relay_inboxes (中继)
             │
             └───► 仅公开帖子
     │
     ▼
ActivityPub::RawDistributionWorker#distribute!
     │
     ├───► 去重 inbox 列表
     │
     └───► ActivityPub::DeliveryWorker.push_bulk (每批 1000)
             │
             ▼
     ActivityPub::DeliveryWorker
             │
             ├───► 构建 HTTP 签名头
             ├───► POST 到远端 inbox
             ├───► 处理响应
             │       │
             │       ├───► 成功：更新送达追踪
             │       ├───► 4xx：标记失败/暂停送达
             │       └───► 5xx：指数退避重试
             │
             └───► 共享收件箱优化：同一实例仅发一次
```

---

## 关键代码位置索引

| 功能 | 文件路径 | 关键方法/行号 |
|------|----------|---------------|
| **关注服务** | `app/services/follow_service.rb` | `#call` L18, `#direct_follow!` L80, `#request_follow!` L68 |
| **时间线合并** | `app/workers/merge_worker.rb` | `#perform` L8, `#merge_into_home!` L25 |
| **帖子发布** | `app/services/post_status_service.rb` | `#call` L42, `#postprocess_status!` L162 |
| **本地分发入口** | `app/workers/distribution_worker.rb` | `#perform` L8 |
| **本地扇出核心** | `app/services/fan_out_on_write_service.rb` | `#call` L12, `#fan_out_to_local_recipients!` L41 |
| **时间线插入** | `app/workers/feed_insert_worker.rb` | `#perform` L7, `#check_and_insert` L31 |
| **时间线管理** | `app/lib/feed_manager.rb` | `#push_to_home` L76, `#merge_into_home` L128, `#add_to_feed` L537, `#filter_from_home` L448 |
| **远端分发** | `app/workers/activitypub/distribution_worker.rb` | `#perform` L11, `#inboxes` L22 |
| **目标发现** | `app/lib/status_reach_finder.rb` | `#inboxes` L12, `#followers_scope` L104 |
| **原始分发** | `app/workers/activitypub/raw_distribution_worker.rb` | `#distribute!` L25 |
| **本地通知** | `app/workers/local_notification_worker.rb` | `#perform` L6 |
| **通知服务** | `app/services/notify_service.rb` | `#call` L201, `DropCondition` L102, `FilterCondition` L165 |
| **通知模型** | `app/models/notification.rb` | `PROPERTIES` L37 |
| **关注模型** | `app/models/follow.rb` | 全文件 |
| **账户交互** | `app/models/concerns/account/interactions.rb` | `#followers_for_local_distribution` L216, `#follow!` L46 |

---

## 总结

### 核心设计理念

1. **写时扇出（Fan-out on Write）**：帖子发布时立即推送到所有关注者时间线，读时直接查询，优化读取性能
2. **Redis 有序集合**：使用 `ZSET` 实现高效的时间线排序、插入和范围查询
3. **异步处理**：所有耗时操作通过 Sidekiq Worker 异步执行，不阻塞请求
4. **活跃用户优化**：仅向最近活跃用户推送时间线，不活跃用户下次登录时重建
5. **多层过滤**：时间线过滤 + 通知过滤，确保用户只看到想看的内容
6. **ActivityPub 联邦**：通过标准协议与远端实例交互，实现分布式社交网络

### 本地 vs 远端关键差异

| 方面 | 本地关注者 | 远端关注者 |
|------|-----------|-----------|
| **数据位置** | 本地 Redis | 远端实例 |
| **推送方式** | 直接内存操作 | HTTP POST |
| **一致性** | 强一致 | 最终一致 |
| **失败处理** | 无（本地操作） | 重试队列 + 退避策略 |
| **过滤责任** | 本实例过滤 | 远端实例自行过滤 |
| **实时性** | 毫秒级 | 秒级（依赖网络） |

### 性能优化策略

1. **批量处理**：`find_in_batches` + `push_bulk` 减少数据库和 Redis 往返
2. **预加载数据**：`build_crutches` 一次性加载所有过滤所需数据，避免 N+1 查询
3. **共享收件箱**：同一实例的多个关注者共享一次 HTTP 请求
4. **容量限制**：`MAX_ITEMS = 800` 防止单条时间线无限增长
5. **转发去重**：`REBLOG_FALLOFF = 40` 避免同一条内容的多次转发刷屏

---

*分析日期：2026-05-02*
*基于 Mastodon 代码库版本：当前工作目录版本*
