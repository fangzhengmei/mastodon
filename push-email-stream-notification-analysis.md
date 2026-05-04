# Mastodon 通知分发与去重机制分析

## 一、通知系统架构概览

Mastodon 的通知系统通过 `NotifyService` 作为核心分发中心，将通知事件分发到四个主要渠道：

1. **站内通知** - 存储在数据库 `notifications` 表
2. **Streaming API** - 实时 WebSocket 推送
3. **Web Push** - 浏览器推送通知
4. **邮件通知** - 异步邮件发送

### 核心分发流程

```
NotifyService.call(recipient, type, activity, **options)
         │
         ├──► 1. DropCondition 检查是否完全丢弃通知
         │
         ├──► 2. FilterCondition 检查是否过滤到通知请求
         │
         ├──► 3. 保存通知到数据库 (站内通知)
         │
         └──► 4. 分发到各渠道 (未被过滤的通知)
                  │
                  ├──► push_to_streaming_api! (Streaming)
                  ├──► push_to_web_push_subscriptions! (Web Push)
                  └──► send_email! (邮件)
```

## 二、各渠道分发机制详解

### 2.1 站内通知 (数据库存储)

**入口文件**: `app/models/notification.rb`

通知首先会被保存到 `notifications` 表，这是所有通知的基础存储。关键字段：

| 字段 | 用途 |
|------|------|
| `type` | 通知类型 (mention, reblog, follow, favourite 等) |
| `filtered` | 是否被过滤 (true 时不会推送到其他渠道) |
| `group_key` | 用于分组展示的键 (去重相关) |
| `from_account_id` | 发送者账户 ID |
| `activity_id` | 关联的活动 ID |

**通知类型定义** (`app/models/notification.rb:37-106`):

```ruby
PROPERTIES = {
  mention: { filterable: true, baseline: true },
  status: { filterable: false, baseline: true },
  reblog: { filterable: true, baseline: true },
  follow: { filterable: true, baseline: true },
  # ... 更多类型
}
```

### 2.2 Streaming API (实时 WebSocket)

**入口文件**: `app/services/notify_service.rb:254-260`

```ruby
def push_to_streaming_api!
  redis.publish(
    "timeline:#{@recipient.id}:notifications",
    { event: :notification, payload: InlineRenderer.render(@notification, @recipient, :notification) }.to_json
  )
end
```

**分发条件**:
- 检查用户是否订阅了 Streaming API (`subscribed_to_streaming_api?`)
- 通过 Redis 键 `subscribed:timeline:{user_id}` 或 `subscribed:timeline:{user_id}:notifications` 判断

**技术实现**:
- 使用 Redis Pub/Sub 机制
- Streaming Server 订阅 Redis 频道，实时推送给 WebSocket 客户端

### 2.3 Web Push (浏览器推送)

**入口文件**: `app/services/notify_service.rb:270-276`

```ruby
def push_to_web_push_subscriptions!
  ::Web::PushNotificationWorker.push_bulk(
    web_push_subscriptions.select { |subscription| subscription.pushable?(@notification) }
  ) { |subscription| [subscription.id, @notification.id] }
end
```

**分发条件** (`app/models/web/push_subscription.rb:36-38`):

```ruby
def pushable?(notification)
  policy_allows_notification?(notification) && alert_enabled_for_notification_type?(notification)
end
```

两层检查：

1. **策略检查** (`policy_allows_notification?`):
   - `all`: 允许所有通知
   - `none`: 拒绝所有通知
   - `followed`: 仅允许已关注账户的通知
   - `follower`: 仅允许关注我的账户的通知

2. **类型检查** (`alert_enabled_for_notification_type?`):
   - 检查订阅数据 `data['alerts'][notification.type]` 是否为 true

**异步处理**:
- `Web::PushNotificationWorker` 处理实际的推送
- 支持标准和 legacy 两种加密方式
- 4xx 错误会清理无效订阅

### 2.4 邮件通知

**入口文件**: `app/services/notify_service.rb:282-305`

```ruby
def send_email!
  return unless NotificationMailer.respond_to?(@notification.type)

  NotificationMailer
    .with(recipient: @recipient, notification: @notification)
    .public_send(@notification.type)
    .deliver_later(wait: 2.minutes)
end
```

**分发条件** (`email_needed?` 方法):

```ruby
def email_needed?
  (!recipient_online? || always_send_emails?) && send_email_for_notification_type?
end
```

三个关键条件：

1. **用户在线状态** (`recipient_online?`):
   ```ruby
   def recipient_online?
     subscribed_to_streaming_api? || subscribed_to_web_push?
   end
   ```
   - 如果用户在线（订阅了 Streaming 或 Web Push），默认不发邮件

2. **总是发送邮件** (`always_send_emails?`):
   - 用户设置 `settings['always_send_emails']` 可以覆盖在线状态判断

3. **通知类型允许** (`send_email_for_notification_type?`):
   ```ruby
   def send_email_for_notification_type?
     NON_EMAIL_TYPES.exclude?(@notification.type) && @recipient.user.settings["notification_emails.#{@notification.type}"]
   end
   ```

**不发送邮件的类型** (`NON_EMAIL_TYPES`):
- `admin.report`, `admin.sign_up`
- `update`, `quoted_update`, `poll`, `status`
- `moderation_warning`, `severed_relationships`, `annual_report`
- `added_to_collection`, `collection_update`

**邮件通知设置** (`app/models/user_settings.rb:45-57`):

```ruby
namespace :notification_emails do
  setting :follow, default: true
  setting :reblog, default: false
  setting :favourite, default: false
  setting :mention, default: true
  setting :quote, default: true
  setting :follow_request, default: true
  # ... 更多
end
```

## 三、用户偏好与通知过滤机制

### 3.1 三层过滤体系

通知在分发前会经过三层过滤：

```
通知事件
    │
    ▼
┌───────────────┐
│  DropCondition │ ──► 完全丢弃，不创建通知记录
└───────────────┘
    │ 不丢弃
    ▼
┌───────────────┐
│ FilterCondition│ ──► 标记为 filtered，仅存数据库，不推送
└───────────────┘
    │ 不过滤
    ▼
┌───────────────┐
│  各渠道偏好检查  │ ──► 按用户设置分发到各渠道
└───────────────┘
```

### 3.2 DropCondition (完全丢弃)

**入口文件**: `app/services/notify_service.rb:102-163`

丢弃条件：

| 条件 | 说明 |
|------|------|
| `recipient.unavailable?` | 接收者账户不可用 (停用/删除) |
| `from_self?` | 自己发给自己 (除特定类型外) |
| `domain_blocking?` | 域屏蔽且未关注对方 |
| `blocking?` | 屏蔽了发送者 |
| `muting_notifications?` | 静音了发送者的通知 |
| `conversation_muted?` | 会话被静音 |
| `blocked_mention?` | 提及被过滤 (feed filter) |

**策略过滤** (需同时满足):
- `filterable_type?` - 通知类型可过滤
- 非 `override_for_sender?` - 没有针对该发送者的权限
- 非 staff 消息

策略选项 (`NotificationPolicy`):
- `drop_not_following?` - 未关注的人
- `drop_not_followers?` - 非关注者
- `drop_new_accounts?` - 新注册账户 (30天内)
- `drop_private_mentions?` - 非回复的私密提及
- `drop_limited_accounts?` - 受限账户

### 3.3 FilterCondition (过滤到通知请求)

**入口文件**: `app/services/notify_service.rb:165-199`

过滤后通知会：
- 保存到数据库 (`filtered = true`)
- 不推送到 Streaming/Web Push/邮件
- 生成 `NotificationRequest` 等待用户审核

过滤条件与 DropCondition 类似，但使用 `filter_*` 策略而非 `drop_*`。

### 3.4 NotificationPolicy 模型

**入口文件**: `app/models/notification_policy.rb`

每个账户的通知策略：

```ruby
enum :for_not_following, { accept: 0, filter: 1, drop: 2 }, suffix: :not_following
enum :for_not_followers, { accept: 0, filter: 1, drop: 2 }, suffix: :not_followers
enum :for_new_accounts, { accept: 0, filter: 1, drop: 2 }, suffix: :new_accounts
enum :for_private_mentions, { accept: 0, filter: 1, drop: 2 }, suffix: :private_mentions
enum :for_limited_accounts, { accept: 0, filter: 1, drop: 2 }, suffix: :limited_accounts
```

每种情况都有三种处理方式：
- `accept`: 正常接收
- `filter`: 过滤到通知请求
- `drop`: 完全丢弃

## 四、多渠道去重机制

### 4.1 隐式去重：邮件 vs 实时渠道

**核心逻辑** (`app/services/notify_service.rb:291-297`):

```ruby
def email_needed?
  (!recipient_online? || always_send_emails?) && send_email_for_notification_type?
end

def recipient_online?
  subscribed_to_streaming_api? || subscribed_to_web_push?
end
```

**去重策略**：
- 如果用户**在线**（订阅了 Streaming 或 Web Push），默认**不发送邮件**
- 这是一种**隐式去重**，避免用户在实时渠道已看到通知的情况下再收到邮件
- 用户可以通过 `always_send_emails` 设置覆盖此行为

**设计意图**：
- 实时渠道 (Streaming/Web Push) 是**即时**的
- 邮件是**延迟**的 (2分钟)
- 如果用户在线，优先通过实时渠道送达
- 邮件作为**离线备份**机制

### 4.2 显式去重：通知分组 (Grouping)

**入口文件**: `app/models/concerns/notification/groups.rb`

#### 可分组的通知类型

```ruby
GROUPABLE_NOTIFICATION_TYPES = %i(favourite reblog follow admin.sign_up).freeze
MAXIMUM_GROUP_SPAN_HOURS = 12
```

#### 分组键生成逻辑

```ruby
def set_group_key!
  return if filtered? || GROUPABLE_NOTIFICATION_TYPES.exclude?(type)

  type_prefix = case type
                when :favourite, :reblog
                  [type, target_status&.id].join('-')  # 按状态分组
                when :follow, :'admin.sign_up'
                  type                                   # 按类型分组
                else
                  raise NotImplementedError
                end
  redis_key   = "notif-group/#{account.id}/#{type_prefix}"
  hour_bucket = activity.created_at.utc.to_i / 1.hour.to_i

  # 复用前一个分组（如果时间跨度不超过12小时）
  previous_bucket = redis.get(redis_key).to_i
  hour_bucket = previous_bucket if hour_bucket < previous_bucket + MAXIMUM_GROUP_SPAN_HOURS

  redis.set(redis_key, hour_bucket, ex: MAXIMUM_GROUP_SPAN_HOURS.hours.to_i)

  self.group_key = "#{type_prefix}-#{hour_bucket}"
end
```

#### 分组规则

| 通知类型 | 分组依据 | 示例 group_key |
|---------|---------|----------------|
| `favourite` | type + target_status_id | `favourite-12345-456789` |
| `reblog` | type + target_status_id | `reblog-12345-456789` |
| `follow` | type | `follow-456789` |
| `admin.sign_up` | type | `admin.sign_up-456789` |

#### 时间窗口合并

- 相同分组键的通知在 **12小时** 内会合并到同一组
- 使用 Redis 缓存最后一个小时桶
- 超过12小时创建新分组

### 4.3 API 层面去重展示

#### API V2 通知分组

**入口文件**: `app/controllers/api/v2/notifications_controller.rb`

API V2 使用分组展示机制，将相同 `group_key` 的通知合并显示。

#### 分组查询

**入口文件**: `app/models/concerns/notification/groups.rb:38-126`

`paginate_groups` 方法使用递归 CTE 实现分组分页：

```sql
WITH RECURSIVE grouped_notifications AS (
  -- Base case: 获取第一条通知
  SELECT notifications.*, ARRAY[group_key] AS groups ...
  
  -- Recursive case: 获取不同 group_key 的下一条通知
  UNION ALL
  SELECT ... WHERE group_key NOT IN (visited groups)
)
```

#### NotificationGroup 模型

**入口文件**: `app/models/notification_group.rb`

将数据库中的多条通知记录聚合为一个 `NotificationGroup`:

| 属性 | 说明 |
|------|------|
| `group_key` | 分组键 |
| `sample_accounts` | 最多8个示例账户 |
| `notifications_count` | 组内通知总数 |
| `notification` | 最新的通知对象 |
| `most_recent_notification_id` | 最新通知 ID |

#### 去重序列化器

**入口文件**: `app/serializers/rest/dedup_notification_group_serializer.rb`

```ruby
class REST::DedupNotificationGroupSerializer < ActiveModel::Serializer
  has_many :accounts, serializer: REST::AccountSerializer
  has_many :partial_accounts, serializer: PartialAccountSerializer
  has_many :statuses, serializer: REST::StatusSerializer
  has_many :notification_groups, serializer: REST::NotificationGroupSerializer
end
```

**展示效果**：
- 多条 "A 喜欢了你的嘟文" 合并为 "A、B、C 等 5 人喜欢了你的嘟文"
- 减少通知列表视觉噪音

### 4.4 各渠道去重总结

| 去重类型 | 实现位置 | 作用范围 | 机制说明 |
|---------|---------|---------|---------|
| **隐式渠道去重** | `NotifyService#email_needed?` | 邮件 vs 实时渠道 | 在线用户不发邮件，避免重复打扰 |
| **通知分组** | `Notification#set_group_key!` | 站内通知数据库 | 相同类型/目标的通知共享 group_key |
| **API 展示去重** | `NotificationGroup` + `DedupNotificationGroupSerializer` | API V2 响应 | 前端展示时合并相同 group_key 的通知 |
| **过滤机制** | `DropCondition` / `FilterCondition` | 全渠道 | 根据用户策略丢弃或过滤通知 |

## 五、完整分发流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           NotifyService.call()                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. DropCondition 检查                                                         │
│    - 接收者不可用?                                                             │
│    - 自己发的? (除特定类型)                                                    │
│    - 被屏蔽/静音?                                                              │
│    - 策略要求丢弃? (drop_not_following 等)                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    ▼                                   ▼
              [丢弃通知]                          [继续处理]
              return nil
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. FilterCondition 检查                                                       │
│    - 策略要求过滤? (filter_not_following 等)                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    ▼                                   ▼
              [标记为 filtered]                    [不过滤]
              notification.filtered = true        notification.filtered = false
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 3. 保存通知到数据库 (站内通知)                                                 │
│    - 生成 group_key (如适用)                                                  │
│    - notification.save!                                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
                    ┌─────────────────┴─────────────────┐
                    ▼                                   ▼
           [filtered = true]                    [filtered = false]
                    │                                   │
                    ▼                                   ▼
┌───────────────────────┐           ┌─────────────────────────────────────────┐
│ update_notification_  │           │ 4. push_notification!                    │
│ request!              │           │    - push_to_streaming_api! (如订阅)    │
│                       │           │    - push_to_web_push_subscriptions!    │
│ 仅创建/更新           │           │                                           │
│ NotificationRequest   │           │ 5. push_to_conversation! (如私信)       │
│ 不推送其他渠道         │           │                                           │
└───────────────────────┘           │ 6. send_email! (如需要)                  │
                                    │    - !recipient_online? || always_send?  │
                                    │    - 非 NON_EMAIL_TYPES                   │
                                    │    - 用户开启该类型邮件通知                │
                                    └─────────────────────────────────────────┘
```

## 六、关键代码位置索引

| 功能 | 文件路径 | 关键方法/行号 |
|------|---------|--------------|
| 通知分发核心 | `app/services/notify_service.rb` | `#call`, `#push_notification!`, `#send_email!` |
| 丢弃条件检查 | `app/services/notify_service.rb:102-163` | `DropCondition#drop?` |
| 过滤条件检查 | `app/services/notify_service.rb:165-199` | `FilterCondition#filter?` |
| 邮件发送条件 | `app/services/notify_service.rb:291-305` | `#email_needed?`, `#recipient_online?` |
| Streaming 推送 | `app/services/notify_service.rb:254-260` | `#push_to_streaming_api!` |
| Web Push 推送 | `app/services/notify_service.rb:270-276` | `#push_to_web_push_subscriptions!` |
| Web Push 偏好 | `app/models/web/push_subscription.rb:36-64` | `#pushable?`, `#policy_allows_notification?` |
| 通知策略 | `app/models/notification_policy.rb` | 各类 `drop_*` / `filter_*` 方法 |
| 通知分组 | `app/models/concerns/notification/groups.rb` | `#set_group_key!`, `#paginate_groups` |
| 通知模型 | `app/models/notification.rb` | `PROPERTIES`, `TARGET_STATUS_INCLUDES_BY_TYPE` |
| 用户设置 | `app/models/user_settings.rb` | `notification_emails` namespace |
| API V2 分组 | `app/models/notification_group.rb` | `::from_notifications` |
| 去重序列化 | `app/serializers/rest/dedup_notification_group_serializer.rb` | 完整文件 |

## 七、设计要点总结

### 7.1 渠道差异化设计

| 渠道 | 实时性 | 可靠性 | 适用场景 |
|------|--------|--------|---------|
| Streaming API | 实时 | 会话级 | 用户在线时即时通知 |
| Web Push | 近实时 | 设备级 | 浏览器后台时推送 |
| 邮件 | 延迟 (2min) | 持久化 | 离线备份、重要通知 |
| 站内通知 | 持久化 | 最高 | 历史记录、统一入口 |

### 7.2 去重策略哲学

1. **分层去重**：
   - 业务层：用户偏好策略 (drop/filter)
   - 渠道层：在线状态判断 (邮件 vs 实时)
   - 展示层：通知分组 (UI 优化)

2. **用户控制权优先**：
   - 每种通知类型可独立配置
   - 可覆盖默认行为 (`always_send_emails`)
   - Web Push 有独立的策略控制

3. **性能考虑**：
   - 邮件延迟 2 分钟，给实时渠道预留送达时间
   - 通知分组使用 Redis 缓存，避免复杂查询
   - 异步处理 (Sidekiq workers) 避免阻塞主流程
