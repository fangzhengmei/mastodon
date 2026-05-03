# Mastodon 关注/取关后时间线写入与通知生成分析

## 目录

1. [关注流程总览](#关注流程总览)
2. [时间线写入路径](#时间线写入路径)
3. [通知生成路径](#通知生成路径)
4. [本地账号 vs 远端联邦账号处理差异](#本地账号-vs-远端联邦账号处理差异)
5. [远端账号"陈旧"判定与刷新触发机制](#远端账号陈旧判定与刷新触发机制)
6. [取关链路与通知分析](#取关链路与通知分析)
7. [静音和屏蔽的影响层次](#静音和屏蔽的影响层次)
8. [远端账号数据延迟/不一致的处理手段](#远端账号数据延迟不一致的处理手段)

---

## 关注流程总览

### FollowService 核心逻辑

文件位置：`app/services/follow_service.rb`

关注流程有两条主要路径：

```ruby
# app/services/follow_service.rb:39-43
if (@target_account.locked? && !@options[:bypass_locked]) || @source_account.silenced? || @target_account.activitypub?
  request_follow!
elsif @target_account.local?
  direct_follow!
end
```

#### 1. direct_follow! (本地直接关注路径)

当目标账号是**本地且未锁定**时走此路径：

```ruby
# app/services/follow_service.rb:80-90
def direct_follow!
  follow = @source_account.follow!(@target_account, **follow_options.merge(rate_limit: @options[:with_rate_limit], bypass_limit: @options[:bypass_limit]))

  LocalNotificationWorker.perform_async(@target_account.id, follow.id, follow.class.name, 'follow')
  MergeWorker.perform_async(@target_account.id, @source_account.id, 'home')
  MergeWorker.push_bulk(@source_account.owned_lists.with_list_account(@target_account).pluck(:id)) do |list_id|
    [@target_account.id, list_id, 'list']
  end

  follow
end
```

**步骤分解：**
1. 创建 Follow 关系记录
2. 发送关注通知给被关注者 (`LocalNotificationWorker`)
3. 合并历史时间线 (`MergeWorker`) - 将被关注者的历史状态添加到关注者的时间线
4. 处理列表时间线的合并

#### 2. request_follow! (关注请求路径)

当目标账号是**远端 ActivityPub 账号**、**锁定的账号**或**源账号被沉默**时走此路径：

```ruby
# app/services/follow_service.rb:68-78
def request_follow!
  follow_request = @source_account.request_follow!(@target_account, **follow_options.merge(rate_limit: @options[:with_rate_limit], bypass_limit: @options[:bypass_limit]))

  if @target_account.local?
    LocalNotificationWorker.perform_async(@target_account.id, follow_request.id, follow_request.class.name, 'follow_request')
  elsif @target_account.activitypub?
    ActivityPub::DeliveryWorker.perform_async(build_json(follow_request), @source_account.id, @target_account.inbox_url, { 'bypass_availability' => true })
  end

  follow_request
end
```

**步骤分解：**
1. 创建 FollowRequest 记录
2. 如果是本地锁定账号：发送 `follow_request` 通知
3. 如果是远端 ActivityPub 账号：通过 `ActivityPub::DeliveryWorker` 投递 Follow 活动到对方 inbox

---

## 时间线写入路径

时间线写入有两种主要场景：
1. **关注时**：合并历史状态
2. **日常发布**：新状态的扇出 (fan-out)

### 关注时的时间线合并 (MergeWorker)

文件位置：`app/workers/merge_worker.rb`

```ruby
# app/workers/merge_worker.rb:8-21
def perform(from_account_id, into_id, type = 'home')
  with_primary do
    @from_account = Account.find(from_account_id)
  end

  case type
  when 'home'
    merge_into_home!(into_id)
  when 'list'
    merge_into_list!(into_id)
  end
rescue ActiveRecord::RecordNotFound
  true
end
```

实际合并逻辑在 `FeedManager` 中：

```ruby
# app/lib/feed_manager.rb:128-150
def merge_into_home(from_account, into_account)
  return unless into_account.user&.signed_in_recently?

  timeline_key = key(:home, into_account.id)
  aggregate    = into_account.user&.aggregates_reblogs?
  query        = from_account.statuses.list_eligible_visibility.includes(reblog: :account).limit(FeedManager::MAX_ITEMS / 4)

  if redis.zcard(timeline_key) >= FeedManager::MAX_ITEMS / 4
    oldest_home_score = redis.zrange(timeline_key, 0, 0, with_scores: true).first.last.to_i
    query = query.where('id > ?', oldest_home_score)
  end

  statuses = query.to_a
  crutches = build_crutches(into_account.id, statuses)

  statuses.each do |status|
    next if filter_from_home(status, into_account.id, crutches)

    add_to_feed(:home, into_account.id, status, aggregate_reblogs: aggregate)
  end

  trim(:home, into_account.id)
end
```

**关键点：**
- 只对**近期登录**的用户执行合并 (性能优化)
- 获取被关注者最多 `MAX_ITEMS/4` (200) 条符合可见性的状态
- 应用过滤规则 (`filter_from_home`)
- 时间线使用 Redis ZSet 存储，以状态 ID 为 score
- 合并后修剪时间线到 `MAX_ITEMS` (800) 条

### 日常状态发布的扇出 (FanOutOnWriteService)

文件位置：`app/services/fan_out_on_write_service.rb`

这是**最核心**的时间线写入路径，当用户发布新状态时触发：

```ruby
# app/services/fan_out_on_write_service.rb:12-25
def call(status, options = {})
  @status    = status
  @account   = status.account
  @options   = options

  return if @status.proper.account.suspended?

  check_race_condition!
  warm_payload_cache!

  fan_out_to_local_recipients!
  fan_out_to_public_recipients! if broadcastable?
  fan_out_to_public_streams! if broadcastable?
end
```

#### 分发到本地关注者

```ruby
# app/services/fan_out_on_write_service.rb:114-120
def deliver_to_all_followers!
  @account.followers_for_local_distribution.select(:id).reorder(nil).find_in_batches do |followers|
    FeedInsertWorker.push_bulk(followers) do |follower|
      [@status.id, follower.id, 'home', { 'update' => update? }]
    end
  end
end
```

**关键点：**
- `followers_for_local_distribution` 只返回**本地且近期登录**的关注者
- 批量处理，使用 `push_bulk` 优化 Sidekiq 队列

#### FeedInsertWorker 的实际插入

文件位置：`app/workers/feed_insert_worker.rb`

```ruby
# app/workers/feed_insert_worker.rb:29-41
def check_and_insert
  filter_result = feed_filter

  if filter_result
    perform_unpush if update?
  else
    perform_push
  end

  perform_notify if notify?(filter_result)
end
```

```ruby
# app/workers/feed_insert_worker.rb:61-68
def perform_push
  case @type
  when :home, :tags
    FeedManager.instance.push_to_home(@follower, @status, update: update?)
  when :list
    FeedManager.instance.push_to_list(@list, @status, update: update?)
  end
end
```

---

## 通知生成路径

### 关注时的通知

从 `FollowService#direct_follow!` 可以看到：

```ruby
LocalNotificationWorker.perform_async(@target_account.id, follow.id, follow.class.name, 'follow')
```

#### LocalNotificationWorker

文件位置：`app/workers/local_notification_worker.rb`

```ruby
# app/workers/local_notification_worker.rb:6-22
def perform(receiver_account_id, activity_id, activity_class_name, type = nil, options = {})
  receiver = Account.find(receiver_account_id)
  activity = activity_class_name.constantize.find(activity_id)

  if %w(update quoted_update collection_update).include?(type)
    Notification.where(account: receiver, activity: activity, type: type).in_batches.delete_all
  elsif Notification.where(account: receiver, activity: activity, type: type).any?
    return
  end

  NotifyService.new.call(receiver, type || activity_class_name.underscore, activity, **options.symbolize_keys)
rescue ActiveRecord::RecordNotFound
  true
end
```

**关键点：**
- 防止重复通知
- 更新类通知先删除旧的
- 实际通知创建委托给 `NotifyService`

#### NotifyService

文件位置：`app/services/notify_service.rb`

这是通知系统的核心，包含复杂的过滤逻辑：

```ruby
# app/services/notify_service.rb:201-229
def call(recipient, type, activity, **options)
  return if recipient.user.nil?

  @options      = options
  @recipient    = recipient
  @activity     = activity
  @notification = Notification.new(account: @recipient, type: type, activity: @activity)

  return if drop?

  @notification.filtered = filter?
  @notification.set_group_key!
  @notification.save!

  return if @notification.activity.nil?

  if @notification.filtered?
    update_notification_request!
  else
    push_notification!
    push_to_conversation! if direct_message?
    send_email! if email_needed?
  end
rescue ActiveRecord::RecordInvalid
  nil
end
```

**通知丢弃条件 (DropCondition)：**

```ruby
# app/services/notify_service.rb:103-124
def drop?
  blocked   = @recipient.unavailable?
  blocked ||= from_self? && %i(poll severed_relationships moderation_warning annual_report).exclude?(@notification.type)

  return blocked if message? && from_staff?

  blocked ||= domain_blocking?
  blocked ||= @recipient.blocking?(@sender)
  blocked ||= @recipient.muting_notifications?(@sender)
  blocked ||= conversation_muted?
  blocked ||= blocked_mention? if message?

  return true if blocked
  return false unless filterable_type?
  return false if override_for_sender?

  blocked_by_limited_accounts_policy? ||
    blocked_by_not_following_policy? ||
    blocked_by_not_followers_policy? ||
    blocked_by_new_accounts_policy? ||
    blocked_by_private_mentions_policy?
end
```

### Notification 支持的类型

从 `app/models/notification.rb:37-106` 可以看到，Mastodon 支持以下通知类型：

| 类型 | 说明 | filterable |
|------|------|-----------|
| `mention` | 提及 | ✅ |
| `status` | 新状态 (关注时设置了 notify) | ❌ |
| `reblog` | 转发 | ✅ |
| `follow` | 关注 | ✅ |
| `follow_request` | 关注请求 | ✅ |
| `favourite` | 收藏 | ✅ |
| `poll` | 投票结束 | ❌ |
| `update` | 状态编辑 | ❌ |
| `severed_relationships` | 关系切断 (域名封禁等) | ❌ |
| `moderation_warning` |  moderation 警告 | ❌ |
| `annual_report` | 年度报告 | ❌ |
| `admin.sign_up` | 管理员新用户注册 | ❌ |
| `admin.report` | 管理员举报 | ❌ |
| `quote` | 引用 | ✅ |
| `quoted_update` | 引用更新 | ❌ |
| `added_to_collection` | 添加到合集 | ✅ |
| `collection_update` | 合集更新 | ❌ |

**重要发现：没有 `unfollow` 类型！**

---

## 本地账号 vs 远端联邦账号处理差异

### 关键差异总结表

| 维度 | 本地账号 | 远端联邦账号 |
|------|---------|-------------|
| **关注入口** | `FollowService#direct_follow!` | `FollowService#request_follow!` |
| **时间线合并** | 关注时立即通过 `MergeWorker` 合并 | 通过 ActivityPub 活动同步 |
| **通知投递** | `LocalNotificationWorker` 直接写入数据库 | 通过 `ActivityPub::DeliveryWorker` 投递到对方 inbox |
| **状态扇出** | `FanOutOnWriteService` → `FeedInsertWorker` | `ActivityPub::DistributionWorker` 投递到各 followers 的 inbox |

### 关注流程差异

#### 本地关注本地

```
FollowService#direct_follow!
    ↓
创建 Follow 记录
    ↓
LocalNotificationWorker (通知被关注者)
    ↓
MergeWorker (合并时间线)
    ↓
后续状态通过 FanOutOnWriteService 扇出
```

#### 本地关注远端

```
FollowService#request_follow!
    ↓
创建 FollowRequest 记录
    ↓
ActivityPub::DeliveryWorker 投递 Follow 活动到远端 inbox
    ↓
等待远端返回 Accept/Reject 活动
    ↓
收到 Accept 后:
  - AuthorizeFollowService 批准关注
  - 创建 Follow 记录
  - MergeWorker 合并时间线
```

#### 远端关注本地

```
ActivityPub::Activity::Follow#perform (处理来自远端的 Follow 活动)
    ↓
检查 blocking/domain_blocking 等
    ↓
如果本地账号锁定或远端被沉默:
  - 创建 FollowRequest
  - LocalNotificationWorker 发送关注请求通知
否则:
  - AuthorizeFollowService 直接批准
  - 创建 Follow 记录
  - LocalNotificationWorker 发送关注通知
  - ActivityPub::DeliveryWorker 投递 Accept 活动
```

### 状态发布差异

#### 本地用户发布状态

```
PostStatusService
    ↓
FanOutOnWriteService
    ├──→ 本地关注者: FeedInsertWorker 写入时间线
    └──→ 远端关注者: ActivityPub::DistributionWorker 投递 Create 活动
```

#### 远端用户发布状态

```
ActivityPub::Activity::Create#perform (处理来自远端的 Create 活动)
    ↓
@account.schedule_refresh_if_stale! (检查并刷新远端账号)
    ↓
创建/更新 Status 记录
    ↓
DistributionWorker (如果在实时窗口内)
    ↓
FanOutOnWriteService 分发给本地关注者
```

**关键代码：**

```ruby
# app/lib/activitypub/activity/create.rb:77-83
def distribute
  LinkCrawlWorker.perform_in(rand(DISTRIBUTE_DELAY), @status.id)

  ::DistributionWorker.perform_async(@status.id, { 'silenced_account_ids' => @silenced_account_ids }) if @options[:override_timestamps] || @status.within_realtime_window?
end
```

注意：只有当状态在"实时窗口"内时才会触发分发。

### followers_for_local_distribution 方法

这是区分本地和远端分发的关键方法：

```ruby
# app/models/concerns/account/interactions.rb:216-220
def followers_for_local_distribution
  followers.local
    .joins(:user)
    .merge(User.signed_in_recently)
end
```

**关键点：**
- `.local` 只选择本地账号
- `.joins(:user)` 确保有用户记录
- `.merge(User.signed_in_recently)` 只选择近期登录的用户 (性能优化)

---

## 远端账号"陈旧"判定与刷新触发机制

### 核心阈值定义

文件位置：`app/models/account.rb:76-78`

```ruby
BACKGROUND_REFRESH_INTERVAL = 1.week.freeze  # 7天 - 后台刷新间隔
REFRESH_DEADLINE = 6.hours                     # 刷新延迟 0-6 小时
STALE_THRESHOLD = 1.day                        # 1天 - 判定为"陈旧"的阈值
```

### 两个关键方法的区别

#### 1. `possibly_stale?` - "可能陈旧"判定

文件位置：`app/models/account.rb:264-266`

```ruby
def possibly_stale?
  last_webfingered_at.nil? || last_webfingered_at <= STALE_THRESHOLD.ago
end
```

**阈值：1天** (`STALE_THRESHOLD`)

**判定条件：**
- `last_webfingered_at` 为空（从未进行过 webfinger 查询）
- 或 `last_webfingered_at` 超过 1 天前

**用途：** 用于决定是否需要**完整刷新**账号信息

---

#### 2. `schedule_refresh_if_stale!` - 安排刷新任务

文件位置：`app/models/account.rb:268-272`

```ruby
def schedule_refresh_if_stale!
  return unless last_webfingered_at.present? && last_webfingered_at <= BACKGROUND_REFRESH_INTERVAL.ago

  AccountRefreshWorker.perform_in(rand(REFRESH_DEADLINE), id)
end
```

**阈值：1周** (`BACKGROUND_REFRESH_INTERVAL`)

**触发条件：**
- `last_webfingered_at` 存在
- 且 `last_webfingered_at` 超过 1 周前

**动作：** 安排 `AccountRefreshWorker` 在 0-6 小时随机延迟后执行

---

### 不同路径的触发场景

#### 路径 1: 处理 ActivityPub 活动时

**触发位置：**
- `app/lib/activitypub/activity/create.rb:8`
- `app/lib/activitypub/activity/update.rb:8`

```ruby
# app/lib/activitypub/activity/create.rb:7-13
def perform
  @account.schedule_refresh_if_stale!  # ← 检查并安排刷新

  dereference_object!

  create_status
end
```

**触发条件：**
- `last_webfingered_at <= 1.week.ago`

**执行动作：**
- 安排 `AccountRefreshWorker` 在 0-6 小时随机延迟后执行
- **不会立即刷新**

---

#### 路径 2: 签名验证失败时

**触发位置：** `app/controllers/concerns/signature_verification.rb:135-157`

```ruby
def keypair_refresh_key!(keypair)
  return if keypair.actor.local? || !keypair.actor.activitypub?

  actor = if keypair.actor.possibly_stale?  # ← 检查是否超过 1 天
            # Doing a full profile refresh
            keypair.actor.refresh!           # ← 完整刷新
          else
            # Only refreshing keys, skipping potentially more expensive requests
            ActivityPub::FetchRemoteActorService.new.call(keypair.actor.uri, only_key: true, suppress_errors: false)
          end
  # ...
end
```

**触发条件：**
- HTTP 签名验证失败
- 需要重新获取远端账号的公钥

**分支逻辑：**
| 条件 | 动作 |
|------|------|
| `possibly_stale?` 为 true (超过 1 天) | 调用 `refresh!` 进行**完整刷新** |
| `possibly_stale?` 为 false (1 天内) | 只调用 `FetchRemoteActorService` 刷新**密钥** |

---

#### 路径 3: 定期后台刷新 (AccountRefreshWorker)

**触发位置：** `app/workers/account_refresh_worker.rb`

```ruby
class AccountRefreshWorker
  include Sidekiq::Worker

  sidekiq_options queue: 'pull', retry: 3, dead: false, lock: :until_executed, lock_ttl: 1.day.to_i

  def perform(account_id)
    account = Account.find_by(id: account_id)
    return if account.nil? || account.last_webfingered_at > Account::BACKGROUND_REFRESH_INTERVAL.ago

    ResolveAccountService.new.call(account)  # ← 完整刷新
  end
end
```

**触发条件：**
- 被 `schedule_refresh_if_stale!` 安排
- 或被其他定时任务触发
- 且 `last_webfingered_at > 1.week.ago` 时会跳过

**执行动作：**
- 调用 `ResolveAccountService` 进行完整刷新

---

### 刷新机制总结表

| 触发路径 | 检查方法 | 阈值条件 | 刷新动作 | 立即/延迟 |
|---------|---------|---------|---------|----------|
| 处理 Create/Update 活动 | `schedule_refresh_if_stale!` | > 1 周 | 安排 `AccountRefreshWorker` | 延迟 0-6 小时 |
| 签名验证失败 | `possibly_stale?` | > 1 天 | `refresh!` 完整刷新 | 立即执行 |
| 签名验证失败 | `possibly_stale?` | ≤ 1 天 | 只刷新密钥 | 立即执行 |
| AccountRefreshWorker | 直接检查 `last_webfingered_at` | > 1 周 | `ResolveAccountService` | 立即执行 |

### 刷新触发流程图

```
                    远端账号活动到达
                           ↓
              ┌────────────────────────┐
              │ 处理 Create/Update 等   │
              │ ActivityPub 活动        │
              └────────────────────────┘
                           ↓
              ┌────────────────────────┐
              │ schedule_refresh_if_   │
              │ stale!                  │
              └────────────────────────┘
                           ↓
              last_webfingered_at > 1 周?
                    /          \
                  否            是
                  /              \
            (不操作)      安排 AccountRefreshWorker
                                    ↓
                            延迟 0-6 小时后执行
                                    ↓
                          ResolveAccountService
                           (完整刷新账号)


                    另一场景: 签名验证失败
                           ↓
              ┌────────────────────────┐
              │ keypair_refresh_key!   │
              └────────────────────────┘
                           ↓
              possibly_stale? (> 1 天?)
                    /          \
                  是            否
                  /              \
            refresh!         只刷新密钥
         (完整刷新)      (FetchRemoteActorService)
```

---

## 取关链路与通知分析

### 核心发现：普通取关不产生通知

从 `app/models/notification.rb` 的 `PROPERTIES` 定义可以看到，**没有 `unfollow` 通知类型**。

### 取关流程 (UnfollowService)

文件位置：`app/services/unfollow_service.rb`

```ruby
# app/services/unfollow_service.rb:8-21
def call(follower, followee, options = {})
  @follower = follower
  @followee = followee
  @options  = options

  with_redis_lock("relationship:#{[follower.id, followee.id].sort.join(':')}") do
    unfollow! || undo_follow_request!
  end
end
```

```ruby
# app/services/unfollow_service.rb:25-49
def unfollow!
  follow = Follow.find_by(account: @follower, target_account: @followee)
  return unless follow

  list_ids = @follower.owned_lists.with_list_account(@followee).pluck(:list_id) unless @options[:skip_unmerge]

  follow.destroy!

  if @followee.local? && @follower.remote? && @follower.activitypub?
    send_reject_follow(follow)
  elsif @followee.remote? && @followee.activitypub?
    send_undo_follow(follow)
  end

  unless @options[:skip_unmerge]
    UnmergeWorker.perform_async(@followee.id, @follower.id, 'home')
    UnmergeWorker.push_bulk(list_ids) do |list_id|
      [@followee.id, list_id, 'list']
    end
  end

  follow
end
```

### 取关链路的四个分支

#### 分支 A: 本地账号取关本地账号

**条件：**
- `@follower.local?` (关注者是本地)
- `@followee.local?` (被关注者是本地)

**执行流程：**
1. 查找并销毁 `Follow` 记录
2. 无 ActivityPub 活动发送（都是本地）
3. 安排 `UnmergeWorker` 从时间线移除
4. **不产生通知**

**代码分支：**
```ruby
# 两个条件都不满足，走 else (无)
if @followee.local? && @follower.remote? && @follower.activitypub?
  send_reject_follow(follow)   # 不满足
elsif @followee.remote? && @followee.activitypub?
  send_undo_follow(follow)      # 不满足
end
# 结果：不发送任何 ActivityPub 活动
```

---

#### 分支 B: 本地账号取关远端账号

**条件：**
- `@follower.local?` (关注者是本地)
- `@followee.remote? && @followee.activitypub?` (被关注者是远端 ActivityPub 账号)

**执行流程：**
1. 查找并销毁 `Follow` 记录
2. 发送 `Undo Follow` 活动到远端 inbox：
   ```ruby
   def send_undo_follow(follow)
     ActivityPub::DeliveryWorker.perform_async(build_json(follow), follow.account_id, follow.target_account.inbox_url)
   end
   ```
3. 安排 `UnmergeWorker` 从时间线移除
4. **不产生本地通知**

**日志/追踪：**
- `ActivityPub::DeliveryWorker` 的执行被 Sidekiq 记录
- 如果投递失败，会有重试机制
- `DeliveryFailureTracker` 追踪失败的投递

---

#### 分支 C: 远端账号取关本地账号（通过 ActivityPub Undo Follow）

**触发：** 本地实例收到远端发来的 `Undo Follow` 活动

**处理位置：** `app/lib/activitypub/activity/undo.rb:89-101`

```ruby
def undo_follow
  target_account = account_from_uri(target_uri)

  return if target_account.nil? || !target_account.local?

  if @account.following?(target_account)
    @account.unfollow!(target_account)  # 调用 Account#unfollow!
  elsif @account.requested?(target_account)
    FollowRequest.find_by(account: @account, target_account: target_account)&.destroy
  else
    delete_later!(object_uri)
  end
end
```

**执行流程：**
1. 解析目标账号 URI，确认是本地账号
2. 如果远端账号正在关注本地：调用 `@account.unfollow!(target_account)`
3. 如果是关注请求：销毁 `FollowRequest`
4. **不产生通知**
5. **不发送额外 ActivityPub 活动**（作为接收方）

**Account#unfollow! 实现：**
```ruby
# app/models/concerns/account/interactions.rb:97-100
def unfollow!(other_account)
  follow = active_relationships.find_by(target_account: other_account)
  follow&.destroy
end
```

这只是销毁 `Follow` 记录，通过 `Follow` 模型的 `before_destroy`/`after_destroy` 回调触发后续清理。

---

#### 分支 D: 远端账号取关本地账号（本地视角 - 发送 Reject）

**条件：**
- `@followee.local?` (被关注者是本地)
- `@follower.remote? && @follower.activitypub?` (关注者是远端 ActivityPub 账号)

**代码位置：** `app/services/unfollow_service.rb:35-36`

```ruby
if @followee.local? && @follower.remote? && @follower.activitypub?
  send_reject_follow(follow)
```

```ruby
def send_reject_follow(follow)
  ActivityPub::DeliveryWorker.perform_async(build_reject_json(follow), follow.target_account_id, follow.account.inbox_url)
end
```

**场景说明：** 这是当本地系统处理"远端账号取关本地账号"时，**主动向远端发送 `Reject Follow` 活动**的情况。

**执行流程：**
1. 销毁 `Follow` 记录
2. 安排 `UnmergeWorker` 从时间线移除
3. 发送 `Reject Follow` 活动到远端 inbox
4. **不产生本地通知**

---

### 远端处理 Undo Follow 的视角

当远端实例收到本地发来的 `Undo Follow` 时，会执行类似的 `undo_follow` 逻辑。

**关键点：** 远端实例也**不会产生通知**，因为 Notification 类型中没有 `unfollow`。

---

### 唯一会产生"关系切断"通知的场景

只有 `severed_relationships` 类型通知，触发于**域名封禁**场景。

#### 场景 A: 管理员域名封禁 (domain_block)

**触发位置：** `app/services/block_domain_service.rb:52-64`

```ruby
def notify_of_severed_relationships!
  return if @domain_block_event.nil?

  @domain_block_event.affected_local_accounts.reorder(nil).find_in_batches do |accounts|
    notification_jobs_args = accounts.map do |account|
      event = AccountRelationshipSeveranceEvent.create!(account:, relationship_severance_event: @domain_block_event)
      [account.id, event.id, 'AccountRelationshipSeveranceEvent', 'severed_relationships']
    end

    LocalNotificationWorker.perform_bulk(notification_jobs_args)
  end
end
```

**触发条件：**
- 管理员执行域名封禁 (`DomainBlock`)
- 封禁类型为 `suspend` 或涉及大量关注关系切断

**通知内容：**
- 类型：`severed_relationships`
- 关联：`AccountRelationshipSeveranceEvent`
- 包含被切断的关系数量统计

---

#### 场景 B: 用户自行域名封禁 (user_domain_block)

**触发位置：** `app/services/after_block_domain_from_account_service.rb:60-65`

```ruby
def notify_of_severed_relationships!
  return if @domain_block_event.nil?

  event = AccountRelationshipSeveranceEvent.create!(account: @account, relationship_severance_event: @domain_block_event)
  LocalNotificationWorker.perform_async(@account.id, event.id, 'AccountRelationshipSeveranceEvent', 'severed_relationships')
end
```

**触发条件：**
- 用户自行封禁某个域名
- 导致该域名下的所有关注/被关注关系被切断

**通知接收者：** 只通知执行封禁操作的用户

---

### 取关链路完整总结表

| 场景 | Follow 记录 | 时间线清理 | ActivityPub 活动 | 本地通知 | 日志/追踪 |
|------|------------|-----------|-----------------|---------|-----------|
| **本地→本地 取关** | ✅ 销毁 | ✅ UnmergeWorker | ❌ 无 | ❌ 无 | Rails destroy 回调 |
| **本地→远端 取关** | ✅ 销毁 | ✅ UnmergeWorker | ✅ Undo Follow | ❌ 无 | Sidekiq + DeliveryFailureTracker |
| **远端→本地 取关 (接收 Undo)** | ✅ 销毁 | ✅ (通过 unfollow! 回调) | ❌ 无（接收方） | ❌ 无 | ActivityPub 入站日志 |
| **远端→本地 取关 (发送 Reject)** | ✅ 销毁 | ✅ UnmergeWorker | ✅ Reject Follow | ❌ 无 | Sidekiq + DeliveryFailureTracker |
| **管理员域名封禁** | ✅ 批量销毁 | ✅ 批量清理 | ❌ 无 | ✅ severed_relationships | 管理日志 + 通知 |
| **用户域名封禁** | ✅ 批量销毁 | ✅ 批量清理 | ❌ 无 | ✅ severed_relationships | 通知 |

### 取关流程图

```
                    用户触发取关操作
                           ↓
              ┌────────────────────────┐
              │    UnfollowService     │
              │  with_redis_lock 保护   │
              └────────────────────────┘
                           ↓
                    查找 Follow 记录
                           ↓
              ┌────────────────────────┐
              │     follow.destroy!     │
              │  触发 after_destroy 回调 │
              │  - 缓存计数器递减       │
              │  - 缓存失效             │
              └────────────────────────┘
                           ↓
              判断关注者/被关注者类型
                           ↓
         ┌─────────────────┼─────────────────┐
         ↓                 ↓                 ↓
   本地→本地         本地→远端         远端→本地
         ↓                 ↓                 ↓
   无 ActivityPub   发送 Undo Follow    发送 Reject Follow
   活动            到远端 inbox         到远端 inbox
         ↓                 ↓                 ↓
   仅销毁 Follow    同时销毁 Follow      同时销毁 Follow
   记录             记录                  记录
         ↓                 ↓                 ↓
   ┌─────────────────────────────────────────────┐
   │      所有分支都执行:                          │
   │      UnmergeWorker.perform_async            │
   │      - 从 home timeline 移除状态             │
   │      - 从 list timelines 移除状态            │
   │                                               │
   │      ❌ 所有分支都不产生通知                  │
   │         (Notification 无 unfollow 类型)      │
   └─────────────────────────────────────────────┘


           唯一产生通知的例外场景: 域名封禁
                           ↓
              ┌────────────────────────┐
              │   管理员/用户封禁域名   │
              └────────────────────────┘
                           ↓
              ┌────────────────────────┐
              │  RelationshipSeverance │
              │      Event 创建        │
              └────────────────────────┘
                           ↓
              ┌────────────────────────┐
              │ LocalNotificationWorker │
              │   'severed_relationships'│
              └────────────────────────┘
                           ↓
                    ✅ 产生通知
```

---

## 静音和屏蔽的影响层次

### 静音 (Mute)

#### MuteService

文件位置：`app/services/mute_service.rb`

```ruby
# app/services/mute_service.rb:3-18
def call(account, target_account, notifications: nil, duration: 0)
  return if account.id == target_account.id

  mute = account.mute!(target_account, notifications: notifications, duration: duration)

  if mute.hide_notifications?
    BlockWorker.perform_async(account.id, target_account.id)
  else
    MuteWorker.perform_async(account.id, target_account.id)
  end

  DeleteMuteWorker.perform_at(duration.seconds, mute.id) if duration != 0

  mute
end
```

#### MuteWorker

文件位置：`app/workers/mute_worker.rb`

```ruby
# app/workers/mute_worker.rb:7-19
def perform(account_id, target_account_id)
  with_primary do
    @account = Account.find(account_id)
    @target_account = Account.find(target_account_id)
  end

  with_read_replica do
    FeedManager.instance.clear_from_home(@account, @target_account)
    FeedManager.instance.clear_from_lists(@account, @target_account)
  end
rescue ActiveRecord::RecordNotFound
  true
end
```

### 屏蔽 (Block)

#### BlockService

文件位置：`app/services/block_service.rb`

```ruby
# app/services/block_service.rb:6-22
def call(account, target_account)
  return if account.id == target_account.id

  @account = account
  @target_account = target_account

  handle_following_relationships
  handle_collections

  NotificationPermission.where(account: account, from_account: target_account).destroy_all

  block = account.block!(target_account)

  BlockWorker.perform_async(account.id, target_account.id)
  create_notification(block) if !target_account.local? && target_account.activitypub?
  block
end
```

```ruby
# app/services/block_service.rb:26-30
def handle_following_relationships
  UnfollowService.new.call(@account, @target_account) if @account.following?(@target_account)
  UnfollowService.new.call(@target_account, @account) if @target_account.following?(@account)
  RejectFollowService.new.call(@target_account, @account) if @target_account.requested?(@account)
end
```

#### AfterBlockService

文件位置：`app/services/after_block_service.rb`

```ruby
# app/services/after_block_service.rb:3-36
class AfterBlockService < BaseService
  def call(account, target_account)
    @account        = account
    @target_account = target_account

    clear_home_feed!
    clear_list_feeds!
    clear_notification_requests!
    clear_notifications!
    clear_conversations!
  end

  private

  def clear_home_feed!
    FeedManager.instance.clear_from_home(@account, @target_account)
  end

  def clear_list_feeds!
    FeedManager.instance.clear_from_lists(@account, @target_account)
  end

  def clear_conversations!
    AccountConversation.where(account: @account).where('? = ANY(participant_account_ids)', @target_account.id).in_batches.destroy_all
  end

  def clear_notifications!
    Notification.where(account: @account).where(from_account: @target_account).in_batches.delete_all
  end

  def clear_notification_requests!
    NotificationRequest.where(account: @account, from_account: @target_account).destroy_all
  end
end
```

### 过滤层次

静音和屏蔽在多个层次影响时间线和通知：

#### 层次 1: 时间线过滤 (FeedManager)

```ruby
# app/lib/feed_manager.rb:434-441
def blocks_or_mutes?(receiver_id, account_ids, context)
  Block.where(account_id: receiver_id, target_account_id: account_ids).any? ||
    (context == :home ? Mute.where(account_id: receiver_id, target_account_id: account_ids).any? : Mute.where(account_id: receiver_id, target_account_id: account_ids, hide_notifications: true).any?)
end
```

```ruby
# app/lib/feed_manager.rb:448-479
def filter_from_home(status, receiver_id, crutches, timeline_type = :home)
  # ... 其他检查 ...
  
  check_for_blocks = crutches[:active_mentions][status.id] || []
  check_for_blocks.push(status.account_id)

  if status.reblog?
    check_for_blocks.push(status.reblog.account_id)
    check_for_blocks.concat(crutches[:active_mentions][status.reblog_of_id] || [])
  end

  return :filter if check_for_blocks.any? { |target_account_id| crutches[:blocking][target_account_id] || crutches[:muting][target_account_id] }
  return :filter if crutches[:blocked_by][status.account_id]
  
  # ... 其他检查 ...
end
```

**关键点：**
- 检查状态作者、被转发者、所有被提及者
- `blocking` 和 `muting` 都会过滤 home 时间线
- `muting` 在 home 上下文检查所有静音，在其他上下文只检查 `hide_notifications: true` 的

#### 层次 2: 通知过滤 (NotifyService)

```ruby
# app/services/notify_service.rb:103-115
def drop?
  blocked   = @recipient.unavailable?
  blocked ||= from_self? && %i(poll severed_relationships moderation_warning annual_report).exclude?(@notification.type)

  return blocked if message? && from_staff?

  blocked ||= domain_blocking?
  blocked ||= @recipient.blocking?(@sender)
  blocked ||= @recipient.muting_notifications?(@sender)
  blocked ||= conversation_muted?
  blocked ||= blocked_mention? if message?

  return true if blocked
  # ...
end
```

**关键点：**
- `blocking?(@sender)` 直接丢弃
- `muting_notifications?(@sender)` 直接丢弃 (这是静音时 `notifications: true` 的情况)

#### 层次 3: 关注前置检查

```ruby
# app/services/follow_service.rb:56-58
def following_not_allowed?
  domain_not_allowed?(@target_account.domain) || @target_account.blocking?(@source_account) || @source_account.blocking?(@target_account) || @target_account.moved? || (!@target_account.local? && @target_account.ostatus?) || @source_account.domain_blocking?(@target_account.domain)
end
```

**关键点：**
- 任何一方 blocking 对方都会阻止关注
- domain_blocking 也会阻止关注

### 静音 vs 屏蔽 影响对比

| 影响项 | 静音 (notifications: false) | 静音 (notifications: true) | 屏蔽 |
|-------|----------------------------|---------------------------|------|
| 时间线可见性 | ❌ 移除 | ❌ 移除 | ❌ 移除 |
| 通知可见性 | ✅ 保留 | ❌ 移除 | ❌ 移除 |
| 对话可见性 | - | - | ❌ 移除 |
| 通知请求 | - | - | ❌ 移除 |
| 阻止关注 | ❌ 不阻止 | ❌ 不阻止 | ✅ 阻止 |
| 解除双方关注 | ❌ 不解除 | ❌ 不解除 | ✅ 解除 |

---

## 远端账号数据延迟/不一致的处理手段

### 1. 账号刷新机制

详见前文 **[远端账号"陈旧"判定与刷新触发机制](#远端账号陈旧判定与刷新触发机制)** 章节。

### 2. 关注者同步机制

#### ActivityPub::SynchronizeFollowersService

文件位置：`app/services/activitypub/synchronize_followers_service.rb`

```ruby
# app/services/activitypub/synchronize_followers_service.rb:9-18
def call(account, partial_collection_url, expected_digest = nil)
  @account = account
  @expected_followers_ids = []
  @digest = [expected_digest].pack('H*') if expected_digest.present?

  return unless process_collection!(partial_collection_url)

  remove_unexpected_local_followers! if expected_digest.blank? || @digest == "\x00" * 32
end
```

```ruby
# app/services/activitypub/synchronize_followers_service.rb:40-44
def remove_unexpected_local_followers!
  @account.followers.local.where.not(id: @expected_followers_ids).reorder(nil).find_each do |unexpected_follower|
    UnfollowService.new.call(unexpected_follower, @account)
  end
end
```

```ruby
# app/services/activitypub/synchronize_followers_service.rb:46-62
def handle_unexpected_outgoing_follows!(expected_followers)
  expected_followers.each do |expected_follower|
    next if expected_follower.following?(@account)

    if expected_follower.requested?(@account)
      expected_follower.follow_requests.find_by(target_account: @account)&.authorize!
    else
      follow = Follow.new(account: expected_follower, target_account: @account)
      ActivityPub::DeliveryWorker.perform_async(build_undo_follow_json(follow), follow.account_id, follow.target_account.inbox_url)
    end
  end
end
```

#### 触发时机

从 `Follow` 模型的回调：

```ruby
# app/models/follow.rb:72-76
def invalidate_hash_cache
  return if account.local? && target_account.local?

  Rails.cache.delete("followers_hash:#{target_account_id}:#{account.synchronization_uri_prefix}")
end
```

以及 `Account` 模型中的方法：

```ruby
# app/models/concerns/account/interactions.rb:228-239
def remote_followers_hash(url)
  url_prefix = url[Account::URL_PREFIX_RE]
  return if url_prefix.blank?

  Rails.cache.fetch("followers_hash:#{id}:#{url_prefix}/") do
    digest = "\x00" * 32
    followers.matches_uri_prefix(url_prefix).pluck_each(:uri) do |uri|
      Xorcist.xor!(digest, Digest::SHA256.digest(uri))
    end
    digest.unpack1('H*')
  end
end
```

### 3. 密钥变更处理

从 `ActivityPub::ProcessAccountService`：

```ruby
# app/services/activitypub/process_account_service.rb:204-206
def after_key_change!
  RefollowWorker.perform_async(@account.id)
end
```

当远端账号的所有密钥都变更时，会触发 `RefollowWorker` 重新建立关注关系。

### 4. 重复检测和幂等性

#### Tombstone 机制

```ruby
# app/lib/activitypub/activity/create.rb:460-462
def tombstone_exists?
  Tombstone.exists?(uri: object_uri)
end
```

#### Delete 活动优先检测

```ruby
# app/lib/activitypub/activity/create.rb:18
return reject_payload! if unsupported_object_type? || non_matching_uri_hosts?(@account.uri, object_uri) || tombstone_exists? || !related_to_local_activity?
```

```ruby
# app/lib/activitypub/activity/create.rb:22-25
Status.uncached do
  return if delete_arrived_first?(object_uri) || poll_vote?
  @status = find_existing_status
end
```

#### 已存在状态检测

```ruby
# app/lib/activitypub/activity/create.rb:85-89
def find_existing_status
  status   = status_from_uri(object_uri)
  status ||= Status.find_by(uri: @object['atomUri']) if @object['atomUri'].present?
  status if status&.account_id == @account.id
end
```

### 5. 延迟重试机制

#### 媒体下载重试

```ruby
# app/lib/activitypub/activity/create.rb:320-325
rescue Mastodon::UnexpectedResponseError, *Mastodon::HTTP_CONNECTION_ERRORS
  RedownloadMediaWorker.perform_in(rand(PROCESSING_DELAY), media_attachment.id)
rescue Seahorse::Client::NetworkingError => e
  Rails.logger.warn "Error storing media attachment: #{e}"
  RedownloadMediaWorker.perform_async(media_attachment.id)
```

#### 未解析提及重试

```ruby
# app/lib/activitypub/activity/create.rb:375-379
def resolve_unresolved_mentions(status)
  @unresolved_mentions.uniq.each do |uri|
    MentionResolveWorker.perform_in(rand(PROCESSING_DELAY), status.id, uri, { 'request_id' => @options[:request_id] })
  end
end
```

#### 引用验证重试

```ruby
# app/lib/activitypub/activity/create.rb:394-401
def fetch_and_verify_quote
  # ...
rescue Mastodon::RecursionLimitExceededError, Mastodon::UnexpectedResponseError, *Mastodon::HTTP_CONNECTION_ERRORS
  ActivityPub::RefetchAndVerifyQuoteWorker.perform_in(rand(PROCESSING_DELAY), @quote.id, @quote_uri, { 'request_id' => @options[:request_id], 'approval_uri' => @quote_approval_uri })
end
```

### 6. 并发控制

#### Redis 锁机制

```ruby
# app/services/resolve_account_service.rb:107-113
def fetch_account!
  with_redis_lock("resolve:#{@username}@#{domain}") do
    @account = ActivityPub::FetchRemoteAccountService.new.call(actor_url, suppress_errors: @options[:suppress_errors])
  end

  @account
end
```

```ruby
# app/lib/activitypub/activity/create.rb:20-32
with_redis_lock("create:#{object_uri}") do
  Status.uncached do
    return if delete_arrived_first?(object_uri) || poll_vote?

    @status = find_existing_status
  end

  if @status.nil?
    process_status
  elsif @options[:delivered_to_account_id].present?
    postprocess_audience_and_deliver
  end
end
```

### 7. 账号迁移处理

```ruby
# app/services/activitypub/process_account_service.rb:185
@account.moved_to_account  = @json['movedTo'].present? ? moved_account : nil
```

以及 `MoveService` 处理账号迁移时的关注关系迁移。

### 数据一致性策略总结

| 策略 | 实现方式 | 用途 |
|------|---------|------|
| **定期刷新** | `AccountRefreshWorker` + 1 周阈值 | 远端账号信息定期更新 |
| **主动刷新** | `schedule_refresh_if_stale!` + 1 周阈值 | 处理活动时检查并安排刷新 |
| **签名验证失败时刷新** | `possibly_stale?` + 1 天阈值 | 超过 1 天则完整刷新，否则只刷新密钥 |
| **关注者同步** | `SynchronizeFollowersService` + 哈希比对 | 检测并修复不一致的关注关系 |
| **幂等处理** | `find_existing_status` + URI 检查 | 防止重复创建状态 |
| **Tombstone** | `Tombstone` 模型 | 标记已删除资源 |
| **延迟重试** | 各种 `*Worker.perform_in(rand(PROCESSING_DELAY), ...)` | 网络错误后的指数退避重试 |
| **并发锁** | `with_redis_lock` | 防止并发处理同一资源 |
| **密钥变更检测** | `all_public_keys_changed?` → `RefollowWorker` | 密钥变更后重新建立关注 |

---

## 附录：关键类和文件索引

### 关注/取关相关
- `app/services/follow_service.rb` - 关注服务
- `app/services/unfollow_service.rb` - 取关服务
- `app/services/authorize_follow_service.rb` - 批准关注请求
- `app/services/reject_follow_service.rb` - 拒绝关注请求
- `app/models/follow.rb` - 关注关系模型
- `app/models/follow_request.rb` - 关注请求模型

### 时间线相关
- `app/lib/feed_manager.rb` - 时间线管理核心
- `app/services/fan_out_on_write_service.rb` - 状态发布扇出
- `app/workers/feed_insert_worker.rb` - 时间线插入 Worker
- `app/workers/merge_worker.rb` - 关注时合并时间线
- `app/workers/unmerge_worker.rb` - 取关时移除时间线

### 通知相关
- `app/services/notify_service.rb` - 通知服务核心
- `app/workers/local_notification_worker.rb` - 本地通知 Worker
- `app/models/notification.rb` - 通知模型
- `app/models/relationship_severance_event.rb` - 关系切断事件
- `app/services/block_domain_service.rb` - 域名封禁服务 (产生 severed_relationships 通知)
- `app/services/after_block_domain_from_account_service.rb` - 用户域名封禁后处理

### 账号刷新相关
- `app/models/account.rb:76-78` - 阈值常量定义
- `app/models/account.rb:264-266` - `possibly_stale?` 方法
- `app/models/account.rb:268-272` - `schedule_refresh_if_stale!` 方法
- `app/workers/account_refresh_worker.rb` - 账号刷新 Worker
- `app/services/resolve_account_service.rb` - 解析账号服务
- `app/controllers/concerns/signature_verification.rb` - 签名验证 (密钥刷新触发)

### 静音/屏蔽相关
- `app/services/mute_service.rb` - 静音服务
- `app/services/block_service.rb` - 屏蔽服务
- `app/services/after_block_service.rb` - 屏蔽后清理
- `app/workers/mute_worker.rb` - 静音后清理时间线
- `app/workers/block_worker.rb` - 屏蔽后处理
- `app/models/mute.rb` - 静音模型
- `app/models/block.rb` - 屏蔽模型

### ActivityPub 联邦相关
- `app/lib/activitypub/activity/follow.rb` - 处理 Follow 活动
- `app/lib/activitypub/activity/undo.rb` - 处理 Undo 活动 (包含 Undo Follow)
- `app/lib/activitypub/activity/create.rb` - 处理 Create 活动
- `app/lib/activitypub/activity/update.rb` - 处理 Update 活动
- `app/workers/activitypub/delivery_worker.rb` - ActivityPub 投递
- `app/workers/activitypub/distribution_worker.rb` - 状态分发
- `app/services/activitypub/process_account_service.rb` - 处理远端账号信息
- `app/services/activitypub/synchronize_followers_service.rb` - 关注者同步
