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

## 四、三层去重机制详解

Mastodon 的通知去重机制分布在三个不同层面，形成完整的去重体系：

| 层面 | 去重类型 | 触发时机 | 作用渠道 |
|------|---------|---------|---------|
| **第一层** | 源头重复拦截 | 通知创建前 | 全站通知 |
| **第二层** | Web Push 客户端聚合 | 浏览器显示时 | Web Push |
| **第三层** | 通知列表分组展示 | 用户查看时 | API/前端 |

---

### 4.1 第一层：源头重复拦截

**入口文件**: `app/workers/local_notification_worker.rb`

这是最底层的去重机制，在通知创建之前拦截重复事件。

#### 核心实现

```ruby
def perform(receiver_account_id, activity_id, activity_class_name, type = nil, options = {})
  receiver = Account.find(receiver_account_id)
  activity = activity_class_name.constantize.find(activity_id)

  # 两种处理策略
  if %w(update quoted_update collection_update).include?(type)
    # 策略A：更新类通知 - 删除旧的，创建新的
    Notification.where(account: receiver, activity: activity, type: type).in_batches.delete_all
  elsif Notification.where(account: receiver, activity: activity, type: type).any?
    # 策略B：普通通知 - 已存在则直接返回
    return
  end

  NotifyService.new.call(receiver, type || activity_class_name.underscore, activity, **options.symbolize_keys)
end
```

#### 去重键

去重基于三元组 `(account_id, activity_id, activity_type, type)`：

| 字段 | 说明 |
|------|------|
| `account_id` | 接收者账户 ID |
| `activity_id` | 关联活动 ID (如 Follow、Favourite、Mention 等) |
| `activity_type` | 活动类型 |
| `type` | 通知类型 (可选，用于区分同一活动的不同通知类型) |

#### 两种处理策略

| 策略 | 适用类型 | 行为 | 设计意图 |
|------|---------|------|---------|
| **替换策略** | `update`, `quoted_update`, `collection_update` | 删除旧通知，创建新通知 | 编辑嘟文时，用新通知替换旧通知，用户看到最新状态 |
| **去重策略** | 其他所有类型 | 已存在则直接返回，不重复创建 | 防止同一事件（如点赞、转发）产生多条通知 |

#### 触发场景

`LocalNotificationWorker` 被以下服务调用：

| 服务 | 调用场景 | 活动类型 |
|------|---------|---------|
| `FollowService` | 关注用户 | `Follow` |
| `FavouriteService` | 点赞嘟文 | `Favourite` |
| `ReblogService` | 转发嘟文 | `Status` |
| `FanOutOnWriteService` | 发布嘟文时提及/引用 | `Mention`, `Quote` |
| `PollExpirationNotifyWorker` | 投票结束 | `Poll` |
| `CreateCollectionService` | 创建收藏集 | `CollectionItem` |
| `Admin::BaseAction` | 管理员 moderation | `AccountWarning` |

#### 对各渠道的影响

- **站内通知**：直接避免重复记录
- **Streaming API**：避免重复推送
- **Web Push**：避免重复推送
- **邮件**：避免重复发送

#### 时间窗口

- **无时间窗口限制**：基于数据库查询，只要记录存在就去重
- 依赖 `notifications` 表的存在性检查

#### 数据库层面的辅助

虽然没有数据库唯一约束（`(account_id, activity_id, activity_type, type)`），但 `LocalNotificationWorker` 的存在性检查起到了相同作用：

```ruby
# schema.rb 中 notifications 表的索引
t.index ["activity_id", "activity_type"], name: "index_notifications_on_activity_id_and_activity_type"
```

这个索引加速了去重查询。

---

### 4.2 第二层：Web Push 客户端聚合

**入口文件**: `app/javascript/mastodon/service_worker/web_push_notifications.js`

这是在浏览器 Service Worker 层面的去重，当通知数量过多时进行合并展示。

#### 核心实现

```javascript
const MAX_NOTIFICATIONS = 5;
const GROUP_TAG = 'tag';

const notify = options =>
  self.registration.getNotifications().then(notifications => {
    if (notifications.length >= MAX_NOTIFICATIONS) {
      // 策略1：达到最大数量，创建分组通知
      const group = {
        title: formatMessage('notifications.group', ...),
        body: notifications.map(n => n.title).join('\n'),
        tag: GROUP_TAG,
        data: { count: notifications.length + 1, ... }
      };
      
      notifications.forEach(notification => notification.close());
      return self.registration.showNotification(group.title, group);
      
    } else if (notifications.length === 1 && notifications[0].tag === GROUP_TAG) {
      // 策略2：已存在分组，追加到分组
      const group = cloneNotification(notifications[0]);
      group.title = formatMessage('notifications.group', ..., { count: group.data.count + 1 });
      group.body  = `${options.title}\n${group.body}`;
      group.data  = { ...group.data, count: group.data.count + 1 };
      
      return self.registration.showNotification(group.title, group);
    }
    
    // 策略3：正常显示单个通知
    return self.registration.showNotification(options.title, options);
  });
```

#### 去重触发条件

| 条件 | 行为 |
|------|------|
| 通知数 >= `MAX_NOTIFICATIONS` (5) | 创建分组通知，关闭所有现有通知 |
| 已有 1 条分组通知 | 追加内容到该分组 |
| 通知数 < 5 且无分组 | 正常显示单个通知 |

#### 单个通知的去重标识

```javascript
// handlePush 函数中
options.tag = notification.id;  // 使用通知 ID 作为 tag
```

浏览器 Notification API 的 `tag` 属性有特殊行为：
- 如果新通知的 `tag` 与已显示通知相同，**新通知会替换旧通知**
- 这是浏览器层面的隐式去重机制

#### 分组通知的结构

```javascript
{
  title: "5 条新通知",  // 本地化字符串
  body: "Alice 喜欢了你的嘟文\nBob 关注了你\n...",
  tag: GROUP_TAG,        // 固定值 'tag'
  data: {
    url: '/notifications',
    count: 5,              // 累计数量
    preferred_locale: 'zh-CN'
  }
}
```

#### 对各渠道的影响

- **Web Push**：客户端层面合并展示，不影响服务端数据
- **其他渠道**：无影响

#### 时间窗口

- **实时**：浏览器收到推送时立即判断
- **持久化**：通知显示在系统通知中心直到用户关闭

#### 设计意图

1. **避免通知轰炸**：同一时间过多通知会打扰用户
2. **信息聚合**：将相似通知合并展示
3. **渐进式去重**：
   - 1-4 条：单独显示
   - 5 条及以上：分组显示

---

### 4.3 第三层：通知列表分组展示

这是最上层的去重机制，在用户查看通知列表时进行分组展示。分为**服务端分组**和**前端聚合**两部分。

#### 第一部分：服务端分组 (group_key)

**入口文件**: `app/models/concerns/notification/groups.rb`

##### 可分组的通知类型

```ruby
GROUPABLE_NOTIFICATION_TYPES = %i(favourite reblog follow admin.sign_up).freeze
MAXIMUM_GROUP_SPAN_HOURS = 12
```

##### 分组键生成逻辑

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

##### 分组规则

| 通知类型 | 分组依据 | 示例 group_key | 分组逻辑 |
|---------|---------|----------------|---------|
| `favourite` | type + target_status_id | `favourite-12345-456789` | 同一嘟文的点赞合并 |
| `reblog` | type + target_status_id | `reblog-12345-456789` | 同一嘟文的转发合并 |
| `follow` | type | `follow-456789` | 所有新关注合并 |
| `admin.sign_up` | type | `admin.sign_up-456789` | 所有新用户注册合并 |

##### 时间窗口机制

```ruby
MAXIMUM_GROUP_SPAN_HOURS = 12  # 12小时
```

- 使用 Redis 键 `notif-group/{account_id}/{type_prefix}` 存储最后一个小时桶
- 如果新通知的时间与最后一个分组的时间差 **< 12 小时**，复用同一分组
- 超过 12 小时创建新分组
- Redis 键过期时间 = 12 小时

##### 服务端分组查询

**入口文件**: `app/models/concerns/notification/groups.rb:38-126`

`paginate_groups` 使用递归 CTE 实现分组分页：

```sql
WITH RECURSIVE grouped_notifications AS (
  -- Base case: 获取第一条通知
  SELECT notifications.*, ARRAY[group_key] AS groups ...
  
  -- Recursive case: 获取不同 group_key 的下一条通知
  UNION ALL
  SELECT ... WHERE group_key NOT IN (visited groups)
)
```

这确保了每页中相同 `group_key` 的通知只出现一次（取最新的）。

#### 第二部分：前端聚合

**入口文件**: `app/javascript/mastodon/reducers/notification_groups.ts`

##### 核心聚合逻辑

```typescript
function processNewNotification(
  groups: NotificationGroupsState['groups'],
  notification: ApiNotificationJSON,
  groupedTypes: NotificationType[],
) {
  if (!groupedTypes.includes(notification.type)) {
    notification = {
      ...notification,
      group_key: `ungrouped-${notification.id}`,
    };
  }

  // 查找现有分组
  const existingGroupIndex = groups.findIndex(
    (group) =>
      group.type !== 'gap' && group.group_key === notification.group_key,
  );

  if (existingGroupIndex > -1) {
    const existingGroup = groups[existingGroupIndex];
    
    if (!existingGroup.sampleAccountIds.includes(notification.account.id)) {
      // 追加到现有分组
      existingGroup.sampleAccountIds.unshift(notification.account.id);
      if (existingGroup.sampleAccountIds.length > NOTIFICATIONS_GROUP_MAX_AVATARS)
        existingGroup.sampleAccountIds.pop();  // 最多显示8个头像
      
      existingGroup.most_recent_notification_id = notification.id;
      existingGroup.notifications_count += 1;
      
      // 移动到列表顶部
      groups.splice(existingGroupIndex, 1);
      groups.unshift(existingGroup);
    }
  } else {
    // 创建新分组
    groups.unshift(createNotificationGroupFromNotificationJSON(notification));
  }
}
```

##### 分组聚合规则

| 条件 | 行为 |
|------|------|
| 找到相同 `group_key` 的分组 | 追加账户到 `sampleAccountIds`，增加 `notifications_count` |
| `sampleAccountIds` > 8 | 只保留最新的 8 个账户头像 |
| 未找到相同分组 | 创建新的 `NotificationGroup` |

##### 分组数据结构

```typescript
interface NotificationGroup {
  group_key: string;                    // 分组键
  type: NotificationType;               // 通知类型
  sampleAccountIds: string[];           // 示例账户 ID（最多8个）
  notifications_count: number;          // 组内通知总数
  most_recent_notification_id: string;  // 最新通知 ID
  statusId?: string;                     // 关联嘟文 ID（如适用）
  // ... 其他属性
}
```

#### 第三部分：API V2 去重响应

**入口文件**: `app/controllers/api/v2/notifications_controller.rb`

```ruby
def index
  @notifications = load_notifications  # 已按 group_key 分组
  @grouped_notifications = load_grouped_notifications
  @presenter = GroupedNotificationsPresenter.new(@grouped_notifications, ...)
  
  render json: @presenter, serializer: REST::DedupNotificationGroupSerializer, ...
end
```

**去重序列化器** (`app/serializers/rest/dedup_notification_group_serializer.rb`):

```ruby
class REST::DedupNotificationGroupSerializer < ActiveModel::Serializer
  has_many :accounts, serializer: REST::AccountSerializer
  has_many :partial_accounts, serializer: PartialAccountSerializer
  has_many :statuses, serializer: REST::StatusSerializer
  has_many :notification_groups, serializer: REST::NotificationGroupSerializer
end
```

#### 对各渠道的影响

- **站内通知列表**：分组展示，减少视觉噪音
- **API V2**：返回聚合后的数据结构
- **其他渠道**：无影响

#### 时间窗口

| 阶段 | 时间窗口 | 说明 |
|------|---------|------|
| 服务端 `group_key` 生成 | 12 小时 | 超过 12 小时创建新分组 |
| 前端聚合 | 会话级 | 页面加载期间持续聚合 |
| API 查询 | 实时 | 每次请求重新计算分组 |

#### 设计意图

1. **用户体验优化**：
   - 100 人点赞同一嘟文 → 显示 "A、B、C 等 100 人喜欢了你的嘟文"
   - 而不是 100 条独立通知

2. **信息密度**：在有限空间展示更多信息

3. **渐进式展示**：
   - 前 8 个用户显示完整头像
   - 超过 8 个显示 "等 N 人"

---

### 4.4 三层去重对比总结

| 维度 | 第一层：源头重复拦截 | 第二层：Web Push 客户端聚合 | 第三层：通知列表分组展示 |
|------|---------------------|---------------------------|-------------------------|
| **实现位置** | `LocalNotificationWorker` | Service Worker (JS) | 服务端 `group_key` + 前端 reducer |
| **去重键** | `(account, activity, type)` | `tag` (通知 ID 或 `GROUP_TAG`) | `group_key` |
| **触发时机** | 通知创建前 | 浏览器接收推送时 | 通知保存时 + 列表展示时 |
| **时间窗口** | 无限制 (数据库存在性) | 实时 (通知中心生命周期) | 12 小时 (服务端) + 会话级 (前端) |
| **作用渠道** | 全站所有渠道 | Web Push 仅 | API/前端展示仅 |
| **去重强度** | 强 (完全阻止重复) | 弱 (仅展示合并) | 中 (数据分组，不删除) |
| **可恢复性** | 不可恢复 (拦截即丢弃) | 可恢复 (用户点击后跳转完整列表) | 可恢复 (分组可展开查看详情) |

#### 各层之间的关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        通知事件流                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  第一层：源头重复拦截 (LocalNotificationWorker)                          │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 检查是否存在 (account, activity, type) 相同的通知                    │ │
│  │  - 存在且类型为 update/quoted_update/collection_update → 删除旧的   │ │
│  │  - 存在且为其他类型 → 直接返回，不创建                                │ │
│  │  - 不存在 → 继续创建通知                                             │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ (通知已创建)
┌─────────────────────────────────────────────────────────────────────────┐
│  第三层：通知列表分组展示 (并行发生)                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 服务端：生成 group_key (如适用)                                       │ │
│  │  - favourite/reblog: type + target_status_id + hour_bucket          │ │
│  │  - follow/admin.sign_up: type + hour_bucket                          │ │
│  │  - 时间窗口：12小时内复用同一分组                                      │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 前端：processNewNotification                                          │ │
│  │  - 查找相同 group_key 的现有分组                                      │ │
│  │  - 存在：追加账户，增加计数                                            │ │
│  │  - 不存在：创建新分组                                                  │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ (通知分发)
┌─────────────────────────────────────────────────────────────────────────┐
│  各渠道分发                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│  │ Streaming    │  │ Web Push     │  │ 邮件         │                  │
│  │ API          │  │              │  │              │                  │
│  └──────────────┘  └──────┬───────┘  └──────────────┘                  │
│                           │                                                │
│                           ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 第二层：Web Push 客户端聚合 (Service Worker)                         │ │
│  │  - 通知数 < 5：单独显示，tag = notification.id                       │ │
│  │  - 通知数 >= 5：创建分组通知，tag = GROUP_TAG                         │ │
│  │  - 已存在分组：追加内容                                                │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 对邮件和实时渠道的影响

| 去重层级 | 对 Streaming 的影响 | 对 Web Push 的影响 | 对邮件的影响 |
|---------|---------------------|-------------------|-------------|
| **源头重复拦截** | ✅ 阻止重复推送 | ✅ 阻止重复推送 | ✅ 阻止重复发送 |
| **Web Push 客户端聚合** | ❌ 无影响 | ✅ 展示合并 | ❌ 无影响 |
| **通知列表分组展示** | ⚠️ 推送原始数据，前端分组 | ⚠️ 推送原始通知，无分组 | ❌ 无影响 |

**关键观察**：
1. **源头重复拦截**是唯一真正影响所有渠道的去重机制
2. **Web Push 客户端聚合**是 Web Push 独有的展示优化
3. **通知列表分组展示**主要影响 API 响应和前端展示，Streaming 推送的是原始通知，由前端自己聚合

## 五、多渠道去重机制（补充）

### 5.1 隐式去重：邮件 vs 实时渠道

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

### 5.2 各渠道去重总结

| 去重类型 | 实现位置 | 作用范围 | 机制说明 |
|---------|---------|---------|---------|
| **源头重复拦截** | `LocalNotificationWorker` | 全站所有渠道 | 基于 (account, activity, type) 阻止重复创建 |
| **隐式渠道去重** | `NotifyService#email_needed?` | 邮件 vs 实时渠道 | 在线用户不发邮件，避免重复打扰 |
| **Web Push 客户端聚合** | `web_push_notifications.js` | Web Push 展示 | 通知过多时合并显示 |
| **通知分组** | `Notification#set_group_key!` | 站内通知数据库 | 相同类型/目标的通知共享 group_key |
| **API 展示去重** | `NotificationGroup` + 前端 reducer | API V2 响应/前端 | 展示时合并相同 group_key 的通知 |
| **过滤机制** | `DropCondition` / `FilterCondition` | 全渠道 | 根据用户策略丢弃或过滤通知 |

## 六、完整分发流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      LocalNotificationWorker.perform()                        │
│                         【第一层：源头重复拦截】                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    ▼                                   ▼
         存在相同 (account, activity, type)        不存在
                    │                                   │
                    ├──────────────┐                    │
                    ▼              ▼                    ▼
         类型是 update/...    其他类型         继续处理
                    │              │                    │
                    ▼              ▼                    │
         删除旧通知         直接返回                   │
         创建新通知         (去重)                     │
                    │                                   │
                    └─────────────────┬─────────────────┘
                                      ▼
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
│    - 【第三层】生成 group_key (如适用)                                        │
│    - 时间窗口：12小时内复用同一分组                                            │
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
                                    │                                           │
                                    │ 【第二层：Web Push 客户端聚合】            │
                                    │  - 浏览器收到推送后，Service Worker 判断   │
                                    │  - 通知数 >= 5 时合并显示                  │
                                    └─────────────────────────────────────────┘
```

## 七、关键代码位置索引

| 功能 | 文件路径 | 关键方法/行号 |
|------|---------|--------------|
| 源头重复拦截 | `app/workers/local_notification_worker.rb` | `#perform` 完整方法 |
| 通知分发核心 | `app/services/notify_service.rb` | `#call`, `#push_notification!`, `#send_email!` |
| 丢弃条件检查 | `app/services/notify_service.rb:102-163` | `DropCondition#drop?` |
| 过滤条件检查 | `app/services/notify_service.rb:165-199` | `FilterCondition#filter?` |
| 邮件发送条件 | `app/services/notify_service.rb:291-305` | `#email_needed?`, `#recipient_online?` |
| Streaming 推送 | `app/services/notify_service.rb:254-260` | `#push_to_streaming_api!` |
| Web Push 推送 | `app/services/notify_service.rb:270-276` | `#push_to_web_push_subscriptions!` |
| Web Push 偏好 | `app/models/web/push_subscription.rb:36-64` | `#pushable?`, `#policy_allows_notification?` |
| Web Push 客户端聚合 | `app/javascript/mastodon/service_worker/web_push_notifications.js` | `notify` 函数 |
| 通知策略 | `app/models/notification_policy.rb` | 各类 `drop_*` / `filter_*` 方法 |
| 通知分组 (group_key) | `app/models/concerns/notification/groups.rb` | `#set_group_key!`, `#paginate_groups` |
| 前端通知聚合 | `app/javascript/mastodon/reducers/notification_groups.ts` | `processNewNotification` 函数 |
| 通知模型 | `app/models/notification.rb` | `PROPERTIES`, `TARGET_STATUS_INCLUDES_BY_TYPE` |
| 用户设置 | `app/models/user_settings.rb` | `notification_emails` namespace |
| API V2 分组 | `app/models/notification_group.rb` | `::from_notifications` |
| 去重序列化 | `app/serializers/rest/dedup_notification_group_serializer.rb` | 完整文件 |

## 八、设计要点总结

### 8.1 渠道差异化设计

| 渠道 | 实时性 | 可靠性 | 适用场景 | 去重机制 |
|------|--------|--------|---------|---------|
| Streaming API | 实时 | 会话级 | 用户在线时即时通知 | 前端分组展示 |
| Web Push | 近实时 | 设备级 | 浏览器后台时推送 | 客户端聚合 + 前端分组 |
| 邮件 | 延迟 (2min) | 持久化 | 离线备份、重要通知 | 在线状态判断 |
| 站内通知 | 持久化 | 最高 | 历史记录、统一入口 | group_key 分组 + 前端聚合 |

### 8.2 三层去重设计哲学

1. **分层防御**：
   - **第一层** (源头)：阻止重复通知的产生，最严格的去重
   - **第二层** (客户端)：优化展示体验，不影响数据
   - **第三层** (展示)：信息聚合，提升用户体验

2. **用户控制权优先**：
   - 每种通知类型可独立配置
   - 可覆盖默认行为 (`always_send_emails`)
   - Web Push 有独立的策略控制

3. **时间维度考量**：
   - 源头去重：无时间限制 (数据完整性)
   - 服务端分组：12 小时 (平衡新鲜度和聚合度)
   - 客户端聚合：实时 (响应用户状态)

4. **性能考虑**：
   - 邮件延迟 2 分钟，给实时渠道预留送达时间
   - 通知分组使用 Redis 缓存，避免复杂查询
   - 异步处理 (Sidekiq workers) 避免阻塞主流程
   - 前端聚合在客户端完成，减轻服务端压力

### 8.3 去重策略的权衡

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| **源头拦截** | 完全避免重复，节省资源 | 可能误拦截 (竞态条件) | 确保数据一致性 |
| **展示聚合** | 不影响数据，灵活可控 | 需要额外处理逻辑 | UI/UX 优化 |
| **渠道互斥** | 避免打扰用户 | 可能遗漏通知 (多设备场景) | 在线状态判断 |

---

*文档版本: 2.0*
*更新内容: 新增三层去重机制的详细分析*
