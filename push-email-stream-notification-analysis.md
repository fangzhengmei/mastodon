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

## 二、Streaming 心跳机制与邮件判定窗口

### 2.1 问题背景

邮件发送的核心判断逻辑依赖 `recipient_online?` 方法：

```ruby
# app/services/notify_service.rb:291-298
def email_needed?
  (!recipient_online? || always_send_emails?) && send_email_for_notification_type?
end

def recipient_online?
  subscribed_to_streaming_api? || subscribed_to_web_push?
end

def subscribed_to_streaming_api?
  redis.exists?("subscribed:timeline:#{@recipient.id}") || 
  redis.exists?("subscribed:timeline:#{@recipient.id}:notifications")
end
```

这里的问题是：`subscribed:timeline:{user_id}` 这些 Redis 键的存在性并不完全等于用户"在线"。它们有自己的生命周期管理机制。

### 2.2 Streaming 心跳机制详解

**入口文件**: `streaming/index.js:274-288`

```javascript
const subscriptionHeartbeat = channels => {
  const interval = 6 * 60;  // 心跳周期：6 分钟 = 360 秒

  const tellSubscribed = () => {
    channels.forEach(channel => 
      redisClient.set(redisNamespaced(`subscribed:${channel}`), '1', 'EX', interval * 3)
    );
  };

  tellSubscribed();  // 立即执行一次

  const heartbeat = setInterval(tellSubscribed, interval * 1000);  // 每 6 分钟执行一次

  return () => {
    clearInterval(heartbeat);
  };
};
```

### 2.3 关键参数量化

| 参数 | 值 | 说明 |
|------|-----|------|
| **心跳周期** (`interval`) | 6 分钟 (360 秒) | 每次更新 Redis 键的间隔 |
| **TTL** (`interval * 3`) | 18 分钟 (1080 秒) | Redis 键的过期时间 |
| **最大"幽灵在线"窗口** | 18 分钟 | 连接断开后，键最长存活时间 |

### 2.4 心跳机制的时序图

```
时间轴（分钟）:
  0          6         12         18         24
  │          │          │          │          │
  ▼          ▼          ▼          ▼          ▼
  │          │          │          │          │
  ├─ 连接建立
  │  tellSubscribed() ──► Redis 键设置 TTL=18min
  │          │
  │          ├─ 心跳 1 ──► 重置 TTL=18min
  │          │
  │          │          ├─ 心跳 2 ──► 重置 TTL=18min
  │          │          │
  │          │          ├─ 连接断开 (用户关闭页面)
  │          │          │
  │          │          │          ├─ Redis 键过期
  │          │          │          │
  │          │          │          │
  ├──────────┴──────────┴──────────┤
  │      实际在线期间               │
  │                                 │
  ├─────────────────────────────────┴───────────────┤
  │              键存在期间（判定为"在线"）          │
  │              幽灵在线窗口：最多 18 分钟          │
  └───────────────────────────────────────────────────┘
```

### 2.5 对邮件判定的实际影响

#### 场景分析

**场景 1：用户正常在线浏览**

```
时间线:
0min: 用户打开页面 → Streaming 连接建立 → Redis 键 TTL=18min
6min: 心跳 1 → 重置 TTL=18min
12min: 心跳 2 → 重置 TTL=18min
15min: 有新通知 → recipient_online? = true → 不发邮件
18min: 心跳 3 → 重置 TTL=18min
...
```

**结果**：用户在线期间，邮件被正确抑制。

**场景 2：用户关闭页面（连接断开）**

```
时间线:
0min: 用户打开页面 → Redis 键 TTL=18min
6min: 心跳 1 → 重置 TTL=18min
10min: 用户关闭页面 → 连接断开 ❌
12min: 心跳不再执行（连接已断）
15min: 有新通知 
       → Redis 键仍存在（还剩 3min 过期）
       → recipient_online? = true
       → 邮件被抑制 ❌（但用户实际已离线）
28min: Redis 键过期 (10min + 18min = 28min)
30min: 有新通知 
       → recipient_online? = false
       → 邮件正常发送 ✓
```

**关键发现**：用户关闭页面后，最多 **18 分钟** 内的通知仍会被判定为"在线"，邮件被错误抑制。

#### 邮件延迟的复合效应

邮件本身还有 **2 分钟延迟**：

```ruby
# app/services/notify_service.rb:282-290
def send_email!
  NotificationMailer
    .with(recipient: @recipient, notification: @notification)
    .public_send(@notification.type)
    .deliver_later(wait: 2.minutes)  # 2 分钟延迟
end
```

这意味着：
- 通知发生时 → 检查在线状态 → 决定是否入队
- 2 分钟后 → 实际发送邮件

**时间窗口叠加**：

| 阶段 | 时间 | 说明 |
|------|------|------|
| 通知时刻检查 | T=0 | 基于 Redis 键存在性判断 |
| 邮件入队延迟 | T+2min | 如果判定在线，不入队；否则入队 2min 后发送 |
| 最大误判窗口 | 18min | Redis 键最长存活时间 |

**最坏情况**：用户在第 1 分钟关闭页面，第 17 分 59 秒有新通知
- Redis 键还剩 1 秒过期
- `recipient_online?` = true
- 邮件被抑制
- 用户实际已离线 16 分 59 秒

### 2.6 与 Web Push 的联动

`recipient_online?` 同时检查两个渠道：

```ruby
def recipient_online?
  subscribed_to_streaming_api? || subscribed_to_web_push?
end
```

这意味着：
- 如果用户**订阅了 Web Push**，即使关闭了浏览器页面，`subscribed_to_web_push?` 仍可能返回 true
- 邮件会被持续抑制，直到 Web Push 订阅过期或被撤销

**Web Push 订阅的生命周期**：
- Web Push 订阅是持久化的，存储在 `web_push_subscriptions` 表
- 不会随浏览器关闭而消失
- 只有用户手动撤销或推送失败时才会被清理

**对邮件判定的影响**：
- 如果用户启用了 Web Push，邮件可能被**长期抑制**
- 这是设计意图（Web Push 作为实时通知渠道）
- 但如果 Web Push 实际不可用（如浏览器未打开），用户可能长时间收不到通知

### 2.7 缓解机制：always_send_emails 设置

用户可以通过设置覆盖此行为：

```ruby
# app/models/user_settings.rb:39-41
namespace :notification_emails do
  setting :always_send, default: false
  # ...
end
```

当 `always_send_emails?` 为 true 时：

```ruby
def email_needed?
  (!recipient_online? || always_send_emails?) && send_email_for_notification_type?
  #                    ^^^^^^^^^^^^^^^^^^^^^
  #                    这个条件为 true 时，忽略在线状态
end
```

### 2.8 心跳机制的边界情况

| 场景 | 实际在线 | Redis 键存在 | 邮件判定 | 结果 |
|------|---------|-------------|---------|------|
| 用户活跃浏览 | ✅ 是 | ✅ 是 | 在线 = 不发邮件 | ✓ 正确 |
| 刚关闭页面（<18min） | ❌ 否 | ✅ 是 | 在线 = 不发邮件 | ✗ 误判 |
| 关闭页面很久（>18min） | ❌ 否 | ❌ 否 | 离线 = 发邮件 | ✓ 正确 |
| 订阅 Web Push | 可能否 | ✅ 是（Web Push） | 在线 = 不发邮件 | 设计意图 |
| 多设备同时在线 | ✅ 是 | ✅ 是 | 在线 = 不发邮件 | ✓ 正确 |

---

## 三、LocalNotificationWorker 并发风险分析

### 3.1 核心实现回顾

**入口文件**: `app/workers/local_notification_worker.rb:1-23`

```ruby
class LocalNotificationWorker
  include Sidekiq::Worker

  def perform(receiver_account_id, activity_id, activity_class_name, type = nil, options = {})
    receiver = Account.find(receiver_account_id)
    activity = activity_class_name.constantize.find(activity_id)

    # 注释说明：大多数通知类型应该只存在一条记录
    if %w(update quoted_update collection_update).include?(type)
      # 策略A：更新类通知 - 删除旧的，创建新的
      Notification.where(account: receiver, activity: activity, type: type).in_batches.delete_all
    elsif Notification.where(account: receiver, activity: activity, type: type).any?
      # 策略B：普通通知 - 已存在则直接返回
      return
    end

    NotifyService.new.call(receiver, type || activity_class_name.underscore, activity, **options.symbolize_keys)
  rescue ActiveRecord::RecordNotFound
    true
  end
end
```

### 3.2 竞态条件分析

这段代码存在经典的 **"检查后操作" (Check-Then-Act)** 竞态条件：

```
时间线:
Worker A                          Worker B
─────────────────────────────────────────────────────
1. 查询：是否存在？
   → 结果：不存在
                                  2. 查询：是否存在？
                                     → 结果：不存在
3. 调用 NotifyService
   → 创建通知记录 #1
   → 推送 Streaming
   → 推送 Web Push
   → 发送邮件
                                  4. 调用 NotifyService
                                     → 创建通知记录 #2 ⚠️ 重复！
                                     → 推送 Streaming ⚠️ 重复！
                                     → 推送 Web Push ⚠️ 重复！
                                     → 发送邮件 ⚠️ 重复！
```

### 3.3 为什么会发生并发？

`LocalNotificationWorker` 被大量服务异步调用：

| 调用场景 | 可能的并发源 |
|---------|-------------|
| `FollowService` | 分布式环境中多次处理同一 Follow 事件 |
| `FavouriteService` | 同一点赞的重复处理 |
| `ReblogService` | 同一转发的重复处理 |
| `FanOutOnWriteService` | 提及通知的多次投递 |
| ActivityPub 处理 | 联邦环境中重复的 inbox 投递 |
| Sidekiq 重试 | 任务失败后重试，原任务可能仍在执行 |

**典型场景**：
1. ActivityPub 投递延迟，发起重试
2. 两次 `Announce` 活动（转发）几乎同时到达
3. Sidekiq 任务超时，自动重试

### 3.4 数据库层面的问题：缺少唯一约束

**对比两张表的索引设计**：

**notifications 表** (`db/schema.rb:843-858`):

```ruby
create_table "notifications", force: :cascade do |t|
  t.bigint "account_id", null: false
  t.bigint "activity_id", null: false
  t.string "activity_type", null: false
  # ... 其他字段
  
  # ⚠️ 注意：没有唯一约束！
  t.index ["activity_id", "activity_type"], name: "index_notifications_on_activity_id_and_activity_type"
  t.index ["from_account_id"], name: "index_notifications_on_from_account_id"
  # ... 其他索引
end
```

**notification_requests 表** (`db/schema.rb:831-841`):

```ruby
create_table "notification_requests", id: :bigint, ... do |t|
  t.bigint "account_id", null: false
  t.bigint "from_account_id", null: false
  # ... 其他字段
  
  # ✅ 有唯一约束！
  t.index ["account_id", "from_account_id"], 
    name: "index_notification_requests_on_account_id_and_from_account_id", 
    unique: true  # 唯一约束
end
```

### 3.5 两张表的设计对比

| 维度 | notifications | notification_requests |
|------|---------------|----------------------|
| **唯一约束** | ❌ 无 | ✅ `(account_id, from_account_id)` |
| **去重键** | `(account_id, activity_id, activity_type, type)` | `(account_id, from_account_id)` |
| **去重方式** | 应用层查询检查 | 数据库唯一约束 + `find_or_initialize_by` |
| **并发安全** | ❌ 存在竞态条件 | ✅ 数据库层面保证 |
| **去重粒度** | 细粒度（按活动） | 粗粒度（按发送者） |

### 3.6 notification_requests 的正确实现

**入口文件**: `app/services/notify_service.rb:241-247`

```ruby
def update_notification_request!
  return unless %i(mention quote).include?(@notification.type)

  # 使用 find_or_initialize_by + 数据库唯一约束
  notification_request = NotificationRequest.find_or_initialize_by(
    account_id: @recipient.id, 
    from_account_id: @notification.from_account_id
  )
  notification_request.last_status_id = @notification.target_status.id
  notification_request.save
end
```

**即使并发，数据库层面也能保证**：

```
Worker A                          Worker B
─────────────────────────────────────────────────────
1. find_or_initialize_by
   → 不存在，initialize 新对象
                                  2. find_or_initialize_by
                                     → 不存在，initialize 新对象
3. save
   → INSERT 语句
   → 成功 ✓
                                  4. save
                                     → INSERT 语句
                                     → 违反唯一约束 ❌
                                     → 抛出 RecordNotUnique
                                     → Rails 重试查询（内部机制）
                                     → 找到 Worker A 创建的记录
                                     → 更新 instead of 插入 ✓
```

### 3.7 notifications 表缺少唯一约束的影响

#### 可能的后果

| 后果 | 影响范围 | 严重程度 |
|------|---------|---------|
| 站内通知重复显示 | 用户体验 | 中 |
| Streaming 重复推送 | 用户体验 | 中 |
| Web Push 重复推送 | 用户体验（打扰） | 高 |
| 邮件重复发送 | 用户体验（打扰） | 高 |
| 数据库垃圾数据 | 存储/性能 | 低 |

#### 为什么这是一个问题？

1. **用户体验**：重复通知会打扰用户
2. **信任问题**：用户可能认为系统不可靠
3. **资源浪费**：重复的 Web Push 和邮件消耗系统资源

#### 当前的"缓解"措施

代码注释暗示了这个问题的已知性：

```ruby
# For most notification types, only one notification should exist, and the older one is
# preferred. For updates, such as when a status is edited, the new notification
# should replace the previous ones.
```

但实际实现依赖应用层检查，无法保证并发安全。

### 3.8 可能的修复方案

#### 方案 A：添加数据库唯一约束（推荐）

```ruby
# 迁移文件
add_index :notifications, 
  [:account_id, :activity_id, :activity_type, :type],
  unique: true,
  name: 'index_notifications_on_unique_activity'
```

**优点**：
- 数据库层面保证，最可靠
- 应用层无需改变（或只需添加异常处理）

**缺点**：
- 需要数据迁移
- 现有数据可能有重复，需要先清理

#### 方案 B：使用数据库锁

```ruby
def perform(...)
  # 使用 advisory lock 或行级锁
  Notification.transaction do
    # 加锁查询
    existing = Notification.lock("FOR UPDATE")
      .find_by(account: receiver, activity: activity, type: type)
    
    if existing
      return unless update_type
      existing.destroy
    end
    
    NotifyService.new.call(...)
  end
end
```

**优点**：
- 无需迁移
- 相对安全

**缺点**：
- 可能引入死锁
- 性能开销

#### 方案 C：使用 Redis 分布式锁

```ruby
def perform(...)
  lock_key = "notify:#{receiver.id}:#{activity.id}:#{type}"
  
  # 使用 Redis SETNX 获取锁
  redis.set(lock_key, '1', nx: true, ex: 30) do |locked|
    if locked
      begin
        # 原有逻辑
      ensure
        redis.del(lock_key)
      end
    end
  end
end
```

**优点**：
- 跨进程/跨机器有效
- 性能较好

**缺点**：
- Redis 单点故障风险
- 锁过期时间需要谨慎设置

### 3.9 实际发生频率评估

虽然代码存在竞态条件，但在实际生产环境中：

1. **Sidekiq 默认配置**：
   - 大多数任务不会频繁重试
   - 任务执行相对较快

2. **联邦场景**：
   - 重复的 ActivityPub 投递确实可能发生
   - 但通常有一定时间间隔，竞态窗口较小

3. **实际影响**：
   - 这是一个"罕见但可能发生"的问题
   - 一旦发生，用户体验较差

---

## 四、notification_requests 对照分析

### 4.1 什么是 NotificationRequest？

当通知被标记为 `filtered = true` 时（根据 `FilterCondition`），它不会直接推送给用户，而是生成一个 `NotificationRequest`，等待用户审核。

**触发条件** (`app/services/notify_service.rb:219-226`):

```ruby
# It's possible the underlying activity has been deleted
# between the save call and now
return if @notification.activity.nil?

if @notification.filtered?
  update_notification_request!    # 生成通知请求
else
  push_notification!               # 正常推送
  push_to_conversation! if direct_message?
  send_email! if email_needed?
end
```

### 4.2 两张表的功能对比

| 维度 | notifications | notification_requests |
|------|---------------|----------------------|
| **用途** | 正常通知记录 | 过滤后的通知请求 |
| **显示位置** | 通知列表 | 通知请求列表（单独入口） |
| **推送渠道** | Streaming / Web Push / 邮件 | ❌ 不推送任何渠道 |
| **用户操作** | 可直接查看/交互 | 需审核（接受/拒绝） |
| **生命周期** | 可被读取/清除 | 可被接受/拒绝/忽略 |

### 4.3 数据模型对比

**NotificationRequest 模型** (`app/models/notification_request.rb`):

```ruby
# == Schema Information
#
# Table name: notification_requests
#
#  id                  :bigint(8)        not null, primary key
#  notifications_count :bigint(8)        default(0), not null
#  created_at          :datetime         not null
#  updated_at          :datetime         not null
#  account_id          :bigint(8)        not null
#  from_account_id     :bigint(8)        not null
#  last_status_id      :bigint(8)
#

class NotificationRequest < ApplicationRecord
  MAX_MEANINGFUL_COUNT = 100

  belongs_to :account
  belongs_to :from_account, class_name: 'Account'
  belongs_to :last_status, class_name: 'Status'

  before_save :prepare_notifications_count

  def prepare_notifications_count
    self.notifications_count = Notification
      .where(
        account: account, 
        from_account: from_account, 
        type: [:mention, :quote], 
        filtered: true
      )
      .limit(MAX_MEANINGFUL_COUNT)
      .count
  end
end
```

### 4.4 去重设计对比

#### notifications 表的去重设计

```ruby
# app/workers/local_notification_worker.rb
if %w(update quoted_update collection_update).include?(type)
  Notification.where(account: receiver, activity: activity, type: type).in_batches.delete_all
elsif Notification.where(account: receiver, activity: activity, type: type).any?
  return
end
```

**去重键**：`(account_id, activity_id, activity_type, type)`

**粒度**：按**活动**去重（同一个 Follow、同一个 Favourite 等）

#### notification_requests 表的去重设计

```ruby
# app/services/notify_service.rb:241-247
def update_notification_request!
  return unless %i(mention quote).include?(@notification.type)

  notification_request = NotificationRequest.find_or_initialize_by(
    account_id: @recipient.id, 
    from_account_id: @notification.from_account_id  # 按发送者
  )
  notification_request.last_status_id = @notification.target_status.id
  notification_request.save
end
```

**去重键**：`(account_id, from_account_id)`

**粒度**：按**发送者**去重（同一人的所有提及/引用聚合）

### 4.5 聚合逻辑对比

#### notifications 的聚合：group_key 机制

- 按**类型 + 目标**聚合
- `favourite/reblog`: 同一嘟文的互动聚合
- `follow`: 所有新关注聚合
- 时间窗口：12 小时

#### notification_requests 的聚合：发送者聚合

- 按**发送者账户**聚合
- 同一人的多条 mention/quote 合并为一个请求
- `notifications_count` 记录数量
- `last_status_id` 指向最新的嘟文

**聚合计算** (`app/models/notification_request.rb:51-53`):

```ruby
def prepare_notifications_count
  self.notifications_count = Notification
    .where(
      account: account, 
      from_account: from_account, 
      type: [:mention, :quote],   # 仅这两种类型
      filtered: true               # 仅被过滤的
    )
    .limit(MAX_MEANINGFUL_COUNT)  # 最多 100
    .count
end
```

### 4.6 唯一约束的设计哲学对比

#### 为什么 notification_requests 有唯一约束？

```ruby
# db/schema.rb:838
t.index ["account_id", "from_account_id"], 
  name: "index_notification_requests_on_account_id_and_from_account_id", 
  unique: true
```

**原因分析**：

1. **聚合需求更强**：
   - `NotificationRequest` 的核心就是**按发送者聚合**
   - 同一发送者只应该有一个待审核请求
   - 数据模型本身就是聚合体

2. **更新频率更高**：
   - 同一人可能发送多条提及
   - `update_notification_request!` 可能被频繁调用
   - 竞态风险更高

3. **数据一致性要求**：
   - `notifications_count` 是聚合计算的
   - 如果存在重复记录，计数会混乱

#### 为什么 notifications 没有唯一约束？

**可能的设计考虑**：

1. **历史原因**：
   - 早期版本可能没有考虑到并发问题
   - 后来添加的"检查后操作"逻辑是补丁方案

2. **多态复杂性**：
   - `activity_type` 是多态字段
   - 唯一约束需要包含 `type`（区分 `update` 和普通通知）
   - 某些通知类型可能不需要去重

3. **策略差异**：
   - `update` 类型需要删除旧的、创建新的
   - 其他类型需要跳过已存在的
   - 唯一约束的行为可能与这些策略不完全匹配

### 4.7 处理流程对比

#### 正常通知流程

```
LocalNotificationWorker
    │
    ▼
检查是否存在 (account, activity, type)
    │
    ├──► 存在且是 update 类型 → 删除旧的，创建新的
    │
    ├──► 存在且非 update → 直接返回 (去重)
    │
    └──► 不存在 → 继续
              │
              ▼
         NotifyService.call()
              │
              ▼
         ┌────┴────┐
         ▼         ▼
    filtered=true  filtered=false
         │              │
         ▼              ▼
    update_notification  push_to_streaming
    _request!            push_to_web_push
                          send_email (如需要)
```

#### 通知请求流程

```
filtered = true 的通知
         │
         ▼
update_notification_request!
         │
         ▼
find_or_initialize_by (account_id, from_account_id)
         │
         ▼
    ┌────┴────┐
    ▼         ▼
  存在记录   新记录
    │         │
    ▼         ▼
更新字段    新记录
last_status_id
notifications_count
    │         │
    └────┬────┘
         ▼
      save()
         │
         ▼
    数据库唯一约束
         │
         ▼
    并发安全保证
```

### 4.8 两张表的关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           通知事件流                                        │
└─────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
                    ┌─────────────────┴─────────────────┐
                    ▼                                   ▼
            filtered = false                   filtered = true
                    │                                   │
                    ▼                                   ▼
┌───────────────────────────┐         ┌───────────────────────────────────┐
│    notifications 表        │         │      notification_requests 表      │
│                           │         │                                   │
│ - 所有通知记录             │         │ - 仅 mention/quote 类型            │
│ - 按 activity 去重         │         │ - 按 from_account 聚合             │
│ - 无唯一约束               │         │ - 有唯一约束 (account+from_account) │
│ - 推送到所有渠道           │         │ - 不推送，等待审核                 │
│ - 可被分组展示 (group_key) │         │ - notifications_count 聚合计数      │
└───────────────────────────┘         └───────────────────────────────────┘
                    │                                   │
                    └─────────────────┬─────────────────┘
                                      │
                                      ▼
                    notification_requests 通过外键关联
                    到 notifications (间接，通过 count 查询)
```

---

## 五、各渠道分发机制详解

### 5.1 站内通知 (数据库存储)

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

### 5.2 Streaming API (实时 WebSocket)

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

### 5.3 Web Push (浏览器推送)

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

### 5.4 邮件通知

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

**不发送邮件的类型** (`NON_EMAIL_TYPES`):
- `admin.report`, `admin.sign_up`
- `update`, `quoted_update`, `poll`, `status`
- `moderation_warning`, `severed_relationships`, `annual_report`
- `added_to_collection`, `collection_update`

---

## 六、用户偏好与通知过滤机制

### 6.1 三层过滤体系

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

### 6.2 DropCondition (完全丢弃)

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

### 6.3 FilterCondition (过滤到通知请求)

**入口文件**: `app/services/notify_service.rb:165-199`

过滤后通知会：
- 保存到数据库 (`filtered = true`)
- 不推送到 Streaming/Web Push/邮件
- 生成 `NotificationRequest` 等待用户审核

### 6.4 NotificationPolicy 模型

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

---

## 七、三层去重机制详解

Mastodon 的通知去重机制分布在三个不同层面，形成完整的去重体系：

| 层面 | 去重类型 | 触发时机 | 作用渠道 |
|------|---------|---------|---------|
| **第一层** | 源头重复拦截 | 通知创建前 | 全站通知 |
| **第二层** | Web Push 客户端聚合 | 浏览器显示时 | Web Push |
| **第三层** | 通知列表分组展示 | 用户查看时 | API/前端 |

### 7.1 第一层：源头重复拦截

详见本章前文关于 `LocalNotificationWorker` 的分析。

### 7.2 第二层：Web Push 客户端聚合

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

### 7.3 第三层：通知列表分组展示

这是最上层的去重机制，在用户查看通知列表时进行分组展示。分为**服务端分组**和**前端聚合**两部分。

#### 服务端分组 (group_key)

**入口文件**: `app/models/concerns/notification/groups.rb`

```ruby
GROUPABLE_NOTIFICATION_TYPES = %i(favourite reblog follow admin.sign_up).freeze
MAXIMUM_GROUP_SPAN_HOURS = 12  # 12小时时间窗口
```

##### 分组规则

| 通知类型 | 分组依据 | 示例 group_key |
|---------|---------|----------------|
| `favourite` | type + target_status_id | `favourite-12345-456789` |
| `reblog` | type + target_status_id | `reblog-12345-456789` |
| `follow` | type | `follow-456789` |
| `admin.sign_up` | type | `admin.sign_up-456789` |

#### 前端聚合

**入口文件**: `app/javascript/mastodon/reducers/notification_groups.ts`

```typescript
function processNewNotification(
  groups: NotificationGroupsState['groups'],
  notification: ApiNotificationJSON,
  groupedTypes: NotificationType[],
) {
  // 查找现有分组
  const existingGroupIndex = groups.findIndex(
    (group) =>
      group.type !== 'gap' && group.group_key === notification.group_key,
  );

  if (existingGroupIndex > -1) {
    // 追加到现有分组
    existingGroup.sampleAccountIds.unshift(notification.account.id);
    if (existingGroup.sampleAccountIds.length > NOTIFICATIONS_GROUP_MAX_AVATARS)
      existingGroup.sampleAccountIds.pop();  // 最多显示8个头像
    
    existingGroup.most_recent_notification_id = notification.id;
    existingGroup.notifications_count += 1;
  } else {
    // 创建新分组
    groups.unshift(createNotificationGroupFromNotificationJSON(notification));
  }
}
```

### 7.4 三层去重对比总结

| 维度 | 第一层：源头重复拦截 | 第二层：Web Push 客户端聚合 | 第三层：通知列表分组展示 |
|------|---------------------|---------------------------|-------------------------|
| **实现位置** | `LocalNotificationWorker` | Service Worker (JS) | 服务端 `group_key` + 前端 reducer |
| **去重键** | `(account, activity, type)` | `tag` (通知 ID 或 `GROUP_TAG`) | `group_key` |
| **触发时机** | 通知创建前 | 浏览器接收推送时 | 通知保存时 + 列表展示时 |
| **时间窗口** | 无限制 (数据库存在性) | 实时 (通知中心生命周期) | 12 小时 (服务端) + 会话级 (前端) |
| **作用渠道** | 全站所有渠道 | Web Push 仅 | API/前端展示仅 |
| **并发安全** | ❌ 存在竞态条件 | ✅ 单线程 (浏览器) | ✅ 无并发问题 |

---

## 八、完整分发流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      LocalNotificationWorker.perform()                        │
│                         【第一层：源头重复拦截】                                │
│                         ⚠️ 存在竞态条件风险                                    │
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
│ 【NotificationRequest】│           │                                           │
│ 聚合：按发送者          │           │ 5. push_to_conversation! (如私信)       │
│ 有唯一约束             │           │                                           │
└───────────────────────┘           │ 6. send_email! (如需要)                  │
                                    │    - !recipient_online? || always_send?  │
                                    │    - ⚠️ 心跳 TTL=18分钟，可能误判         │
                                    │    - 非 NON_EMAIL_TYPES                   │
                                    │    - 用户开启该类型邮件通知                │
                                    │                                           │
                                    │ 【第二层：Web Push 客户端聚合】            │
                                    │  - 浏览器收到推送后，Service Worker 判断   │
                                    │  - 通知数 >= 5 时合并显示                  │
                                    └─────────────────────────────────────────┘
```

---

## 九、关键代码位置索引

| 功能 | 文件路径 | 关键方法/行号 |
|------|---------|--------------|
| **Streaming 心跳机制** | `streaming/index.js:274-288` | `subscriptionHeartbeat` 函数 |
| **在线状态判断** | `app/services/notify_service.rb:291-298` | `#email_needed?`, `#recipient_online?` |
| **源头重复拦截** | `app/workers/local_notification_worker.rb` | `#perform` 完整方法 |
| **NotificationRequest 创建** | `app/services/notify_service.rb:241-247` | `#update_notification_request!` |
| **NotificationRequest 模型** | `app/models/notification_request.rb` | `#prepare_notifications_count` |
| **通知分发核心** | `app/services/notify_service.rb` | `#call`, `#push_notification!` |
| **通知分组 (group_key)** | `app/models/concerns/notification/groups.rb` | `#set_group_key!` |
| **前端通知聚合** | `app/javascript/mastodon/reducers/notification_groups.ts` | `processNewNotification` 函数 |

---

## 十、设计要点总结

### 10.1 渠道差异化设计

| 渠道 | 实时性 | 可靠性 | 适用场景 | 去重机制 |
|------|--------|--------|---------|---------|
| Streaming API | 实时 | 会话级 | 用户在线时即时通知 | 前端分组展示 |
| Web Push | 近实时 | 设备级 | 浏览器后台时推送 | 客户端聚合 + 前端分组 |
| 邮件 | 延迟 (2min) | 持久化 | 离线备份、重要通知 | 在线状态判断 (⚠️ 18min 窗口) |
| 站内通知 | 持久化 | 最高 | 历史记录、统一入口 | group_key 分组 + 前端聚合 |
| 通知请求 | 持久化 | 最高 | 过滤后等待审核 | 数据库唯一约束 |

### 10.2 去重设计的问题与改进建议

#### 当前问题

1. **源头去重的竞态条件**：
   - `LocalNotificationWorker` 使用"检查后操作"模式
   - 缺少数据库唯一约束
   - 并发时可能产生重复通知

2. **邮件判定的幽灵在线窗口**：
   - Streaming 心跳 TTL = 18 分钟
   - 用户离线后，邮件可能被错误抑制
   - 结合 Web Push 订阅，可能被长期抑制

3. **两张表的设计不一致**：
   - `notification_requests` 有唯一约束
   - `notifications` 没有唯一约束

#### 改进建议

1. **添加唯一约束**（推荐）：
   - 为 `notifications` 表添加 `(account_id, activity_id, activity_type, type)` 唯一约束
   - 这是最可靠的解决方案

2. **优化邮件判定逻辑**：
   - 考虑引入更精确的在线状态检测
   - 或缩短心跳 TTL 时间

3. **文档化已知问题**：
   - `LocalNotificationWorker` 的竞态条件应在代码注释中明确
   - 邮件判定的 18 分钟窗口应被文档化

---

*文档版本: 3.0*
*更新内容: 新增 Streaming 心跳机制量化分析、LocalNotificationWorker 并发风险分析、notification_requests 对照分析*
