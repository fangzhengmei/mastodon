# Mastodon 关注请求批准流程修正分析报告

## 目录
1. [概述](#概述)
2. [问题一：批准通知发给谁？](#问题一批准通知发给谁)
3. [问题二：快速通道 vs 普通批准](#问题二快速通道-vs-普通批准)
4. [问题三：活跃用户门槛对回填和分发的影响](#问题三活跃用户门槛对回填和分发的影响)
5. [完整修正流程图](#完整修正流程图)
6. [关键代码位置索引](#关键代码位置索引)

---

## 概述

本文档修正前一份报告中的三个关键问题：

1. **批准通知的接收方**：之前的分析部分正确但不完整
2. **快速通道和普通批准的差异**：之前混在一起，需要明确区分
3. **活跃用户门槛的影响**：需要明确 `signed_in_recently?` 对时间线回填和后续分发的不同影响

---

## 问题一：批准通知发给谁？

### 1.1 代码分析

让我重新仔细分析 `FollowRequestsController#authorize`：

```ruby
# app/controllers/api/v1/follow_requests_controller.rb:14-18
def authorize
  # 调用批准服务
  AuthorizeFollowService.new.call(account, current_account)
  
  # 发送通知
  LocalNotificationWorker.perform_async(
    current_account.id,                                    # 参数1：接收者账户 ID
    Follow.find_by(account: account, target_account: current_account).id,  # 参数2：Follow 记录 ID
    'Follow',                                               # 参数3：activity_class_name
    'follow'                                                # 参数4：notification_type
  )
  
  render json: account, serializer: REST::RelationshipSerializer, relationships: relationships
end

private

def account
  @account ||= Account.find(params[:id])  # 通过 URL 参数查找请求发起者
end
```

### 1.2 参数解析

| 参数 | 变量来源 | 实际含义 |
|------|----------|----------|
| `account` | `Account.find(params[:id])` | **请求发起者 A**（关注者） |
| `current_account` | 当前登录用户 | **被请求者 B**（被关注者） |
| 通知接收者 | `current_account.id` | **B（被关注者）** |
| Follow 记录 | `account: account, target_account: current_account` | A 关注 B 的记录 |

### 1.3 通知类型分析

`LocalNotificationWorker` 的参数：

```ruby
LocalNotificationWorker.perform_async(
  receiver_account_id,      # 接收者账户 ID
  activity_id,              # 活动记录 ID
  activity_class_name,      # 活动类名（如 'Follow'）
  type,                     # 通知类型（如 'follow'）
  options                    # 可选参数
)
```

通知类型 `'follow'` 的含义：
- 被关注者收到通知："某某开始关注你了"
- 不是"你的关注已被批准"

### 1.4 另一个通知通道：ActivityPub Accept

在 `AuthorizeFollowService` 中：

```ruby
# app/services/authorize_follow_service.rb:6-15
def call(source_account, target_account, **options)
  if options[:skip_follow_request]
    follow_request = FollowRequest.new(account: source_account, target_account: target_account, uri: options[:follow_request_uri])
  else
    follow_request = FollowRequest.find_by!(account: source_account, target_account: target_account)
    follow_request.authorize!
  end

  # 关键：只有 source_account 是远程时才发送 ActivityPub 活动
  create_notification(follow_request) if !source_account.local? && source_account.activitypub?
end

# app/services/authorize_follow_service.rb:20-22
def create_notification(follow_request)
  ActivityPub::DeliveryWorker.perform_async(
    build_json(follow_request),                    # Accept 活动 JSON
    follow_request.target_account_id,               # 签名账户：本地批准者 B
    follow_request.account.inbox_url                 # 目标：A 的 inbox
  )
end
```

### 1.5 完整通知流向图

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                         批准通知流向完整分析                                            │
└──────────────────────────────────────────────────────────────────────────────────────┘

                          本地用户 B 批准关注请求
                                       │
                                       ▼
              ┌────────────────────────────────────────┐
              │  FollowRequestsController#authorize    │
              └────────────────────────────────────────┘
                                       │
                    ┌──────────────────┴──────────────────┐
                    ▼                                      ▼
      ┌──────────────────────────┐          ┌──────────────────────────────┐
      │ LocalNotificationWorker  │          │ AuthorizeFollowService#call  │
      │                          │          │                              │
      │ 接收者: current_account  │          │ 检查 source_account.local?   │
      │        = B（被关注者）   │          │                              │
      │                          │          │  如果 A 是远程：             │
      │ 通知类型: 'follow'       │          │  发送 Accept 活动到 A 的 inbox│
      │                          │          └──────────────────────────────┘
      │ 内容: "你有新粉丝 A"     │                       │
      └──────────────────────────┘                       ▼
                                                       ┌─────────────────┐
                                                       │ 如果 A 是本地： │
                                                       │ ❌ 不发送任何东西│
                                                       └─────────────────┘
```

### 1.6 不同场景的通知对比

| 场景 | 关注者 A | 被关注者 B | A 收到通知 | B 收到通知 |
|------|----------|------------|-----------|-----------|
| **场景 1** | 本地用户 | 本地用户 | ❌ 无 | ✅ "你有新粉丝 A" |
| **场景 2** | 本地用户 | 远程用户 | ❌ 无 | 远程实例处理 |
| **场景 3** | 远程用户 | 本地用户 | ✅ Accept 活动（远程实例处理） | ✅ "你有新粉丝 A" |

### 1.7 关键结论

**之前的错误**：之前的分析暗示通知发给关注者 A，实际上：

1. **本地批准时**：通知发给 **B（被关注者）**，内容是"你有新粉丝 A"
2. **本地关注者 A**：不会收到"关注已被批准"的本地通知
3. **远程关注者 A**：本地实例发送 ActivityPub Accept 活动到 A 的远程实例，由远程实例处理

**用户感知**：
- 本地关注者 A 只能通过时间线出现被关注者的帖子来间接得知批准已生效
- 没有显式的"关注已批准"通知

---

## 问题二：快速通道 vs 普通批准

### 2.1 快速通道的触发条件

快速通道在 `ActivityPub::Activity::Follow` 中触发：

```ruby
# app/lib/activitypub/activity/follow.rb:23-29
# 快速通道重复 Follow 请求
existing_follow = ::Follow.find_by(account: @account, target_account: target_account)

unless existing_follow.nil?
  # Follow 记录已存在！
  existing_follow.update!(uri: @json['id'])  # 只更新 URI
  
  # 调用批准服务，带 skip_follow_request 参数
  AuthorizeFollowService.new.call(
    @account, 
    target_account, 
    skip_follow_request: true,          # 关键参数
    follow_request_uri: @json['id']
  )
  return
end
```

**场景解释**：

远程用户 A 已经成功关注了本地用户 B，然后因为某种原因重新发送了 Follow 活动：
- 网络问题导致的重复发送
- 服务器重启后重新同步
- 其他分布式同步场景

本地发现 Follow 记录已经存在，所以走快速通道。

### 2.2 AuthorizeFollowService 的两种分支

```ruby
# app/services/authorize_follow_service.rb:6-15
def call(source_account, target_account, **options)
  if options[:skip_follow_request]
    # ============================================
    # 分支 A：快速通道
    # ============================================
    follow_request = FollowRequest.new(
      account: source_account, 
      target_account: target_account, 
      uri: options[:follow_request_uri]
    )
    # 注意：这里没有调用 authorize!
    # 注意：这个 FollowRequest 只是内存对象，没有保存到数据库
  else
    # ============================================
    # 分支 B：正常批准
    # ============================================
    follow_request = FollowRequest.find_by!(account: source_account, target_account: target_account)
    follow_request.authorize!  # 调用核心转换方法
  end

  # 如果 source_account 是远程，发送 Accept 活动（两种分支都可能执行）
  create_notification(follow_request) if !source_account.local? && source_account.activitypub?
end
```

### 2.3 快速通道 vs 正常批准：完整对比

| 维度 | 快速通道（skip_follow_request: true） | 正常批准 |
|------|---------------------------------------|----------|
| **触发场景** | Follow 记录已存在（重复 Follow 活动） | FollowRequest 等待批准 |
| **FollowRequest 操作** | 内存创建，不存数据库 | 数据库查询 + `authorize!` |
| **Follow 记录创建** | ❌ 不创建（已存在） | ✅ 创建 |
| **时间线合并（MergeWorker）** | ❌ 不触发 | ✅ 触发 |
| **ListAccount 更新** | ❌ 不执行 | ✅ 执行 |
| **发送 Accept 活动** | 远程时发送 | 远程时发送 |
| **设计目的** | 分布式同步、重复请求处理 | 正常关注流程 |

### 2.4 为什么快速通道不执行时间线合并？

```
场景：远程用户 A 已经关注了本地用户 B

时间线 T0：A 第一次关注 B
           ├── Follow 记录创建
           ├── MergeWorker 执行（A 的远程实例处理）
           └── A 的时间线包含 B 的历史帖子

时间线 T1：A 的服务器重新发送 Follow 活动（可能因为网络问题）
           ├── 本地发现 Follow 已存在
           ├── 走快速通道
           ├── 只更新 URI
           ├── 发送 Accept 确认
           └── ❌ 不执行时间线合并（因为之前已经合并过）
```

**关键理解**：

快速通道的 `skip_follow_request` 意味着：
1. **跳过**数据库中的 `FollowRequest` 记录处理
2. **不执行** `FollowRequest#authorize!`（这是时间线合并的入口）
3. 只做**最小必要操作**：更新 URI + 发送 Accept 确认

### 2.5 两种流程的详细对比图

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                    快速通道 vs 正常批准 完整流程图                                     │
└──────────────────────────────────────────────────────────────────────────────────────┘

                           收到远程 Follow 活动
                                       │
                                       ▼
                    ┌──────────────────────────────────┐
                    │  Follow 记录已存在？              │
                    │  (existing_follow.present?)     │
                    └──────────────────────────────────┘
                         │                  │
                         │ Yes              │ No
                         ▼                  ▼
              ┌──────────────┐    ┌──────────────────────┐
              │   快速通道    │    │     正常流程         │
              │              │    │                      │
              │ skip_follow_ │    │ FollowRequest.create │
              │ request: true│    │                      │
              └──────────────┘    └──────────────────────┘
                      │                         │
                      ▼                         ▼
         ┌─────────────────────────┐  ┌───────────────────────────┐
         │ AuthorizeFollowService  │  │ AuthorizeFollowService    │
         │                         │  │                            │
         │ 内存创建 FollowRequest  │  │ 查询数据库 FollowRequest   │
         │ (不保存)                 │  │                            │
         │                         │  │ 调用 authorize!            │
         │ ❌ 不执行时间线合并      │  │                            │
         │ ❌ 不更新 ListAccount   │  │ ✅ 创建 Follow 记录       │
         │                         │  │ ✅ 触发 MergeWorker       │
         │ 只更新 URI              │  │ ✅ 更新 ListAccount       │
         │ 发送 Accept 确认        │  │                            │
         └─────────────────────────┘  └───────────────────────────┘
```

### 2.6 另一个快速通道场景

在 `ActivityPub::Activity::Follow` 中还有另一个快速通道：

```ruby
# app/lib/activitypub/activity/follow.rb:15-21
# 快速通道：URI 已存在的 FollowRequest
existing_follow_request = ::FollowRequest.find_by(account: @account, target_account: target_account)

unless existing_follow_request.nil?
  existing_follow_request.update!(uri: @json['id'])
  return  # 直接返回，什么都不做
end
```

场景：远程用户 A 之前已经发送过 Follow 请求，现在重新发送：
- FollowRequest 记录已存在
- 只更新 URI
- 不做其他操作（等待 B 批准）

---

## 问题三：活跃用户门槛对回填和分发的影响

### 3.1 活跃用户的定义

```ruby
# app/models/concerns/user/activity.rb:12-21
ACTIVE_DURATION = ENV.fetch('USER_ACTIVE_DAYS', 7).to_i.days  # 默认 7 天

included do
  # SQL 范围：current_sign_in_at 在最近 7 天内
  scope :signed_in_recently, -> { where(current_sign_in_at: ACTIVE_DURATION.ago..) }
end

def signed_in_recently?
  # 实例方法：检查是否最近登录
  current_sign_in_at.present? && current_sign_in_at >= ACTIVE_DURATION.ago
end
```

**配置说明**：
- `USER_ACTIVE_DAYS` 环境变量控制活跃阈值
- 默认 7 天，可通过环境变量调整

### 3.2 受影响的代码位置

让我系统性地分析 `signed_in_recently?` 的所有使用场景：

| 代码位置 | 影响场景 | 影响类型 |
|----------|----------|----------|
| `FeedManager#merge_into_home` | 关注/批准时的时间线回填 | **时间线回填** |
| `FeedManager#push_to_home` | 新帖发布时的推送 | **实时分发** |
| `followers_for_local_distribution` | 分发时的粉丝筛选 | **实时分发** |
| `lists_for_local_distribution` | 列表时间线分发筛选 | **实时分发** |
| `tag_follows.for_local_distribution` | 标签时间线分发筛选 | **实时分发** |

### 3.3 场景一：时间线回填（MergeWorker）

当用户 A 关注 B 或 B 批准 A 的关注请求时：

```ruby
# app/lib/feed_manager.rb:128-132
def merge_into_home(from_account, into_account)
  # 关键：只有活跃用户才执行合并
  return unless into_account.user&.signed_in_recently?
  
  # ... 合并逻辑（读取 B 的历史帖子，插入到 A 的时间线）
end
```

**影响分析**：

| 用户 A 的状态 | `signed_in_recently?` | 时间线回填行为 |
|---------------|----------------------|----------------|
| 最近 7 天登录过 | ✅ true | ✅ 执行合并，B 的历史帖子插入到 A 的时间线 |
| 超过 7 天未登录 | ❌ false | ❌ 不执行合并，A 的时间线不会出现 B 的历史帖子 |

### 3.4 场景二：实时分发（新帖推送）

当 B 发布新帖时：

```ruby
# app/lib/feed_manager.rb:76-79
def push_to_home(account, status, update: false)
  # 关键：只有活跃用户才推送
  return false unless account.user&.signed_in_recently?
  
  # ... 推送逻辑
end
```

同时，分发时也会筛选：

```ruby
# app/models/concerns/account/interactions.rb:216-220
def followers_for_local_distribution
  followers.local
    .joins(:user)
    .merge(User.signed_in_recently)  # 只选择活跃用户
end
```

**影响分析**：

| 用户 A 的状态 | 在 `followers_for_local_distribution` 中吗？ | B 的新帖推送到 A 的时间线吗？ |
|---------------|----------------------------------------------|------------------------------|
| 最近 7 天登录过 | ✅ 是 | ✅ 实时推送 |
| 超过 7 天未登录 | ❌ 否 | ❌ 不推送 |

### 3.5 不活跃用户的时间线恢复机制

不活跃用户的时间线不是永久丢失的，当他们重新登录时会触发重建：

```ruby
# app/models/user.rb:501-506
def prepare_returning_user!
  return unless confirmed?
  
  ActivityTracker.record('activity:logins', id)
  regenerate_feed! if inactive_since_duration?  # 长时间不活跃则重建
end

# app/models/user.rb:516-522
def regenerate_feed!
  home_feed = HomeFeed.new(account)
  return if home_feed.regenerating?  # 避免重复重建

  home_feed.regeneration_in_progress!  # 标记正在重建
  RegenerationWorker.perform_async(account_id)  # 异步重建
end
```

### 3.6 完整数据流：不活跃用户关注后重新登录

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                    不活跃用户关注 + 重新登录 完整流程                                  │
└──────────────────────────────────────────────────────────────────────────────────────┘

时间线 T0：用户 A（不活跃，超过 7 天未登录）
         │
         ▼
    关注用户 B（或 B 批准 A 的关注请求）
         │
         ▼
    Follow 记录创建 ✓
    ListAccount 更新（如适用）✓
         │
         ▼
    MergeWorker 入队
         │
         ▼
    FeedManager#merge_into_home(from: B, into: A)
         │
         ▼
    检查 A.user&.signed_in_recently?
         │
         ▼
         ❌ false（A 不活跃）
         │
         ▼
    直接 return，不执行合并
         │
         ▼
    结果：A 的时间线 ❌ 没有 B 的历史帖子
         │
         ▼
         ────────────────────────────────────────────► 时间流逝
                                          │
                                          ▼
                               时间线 T1：A 重新登录
                                          │
                                          ▼
                               Devise::SessionsController#create
                                          │
                                          ▼
                               User#prepare_returning_user!
                                          │
                                          ▼
                               检查 inactive_since_duration?
                                          │
                                          ▼
                                          ✅ true（长时间不活跃）
                                          │
                                          ▼
                               User#regenerate_feed!
                                          │
                                          ├───► HomeFeed.new(A).regeneration_in_progress!
                                          │
                                          └───► RegenerationWorker.perform_async(A.id)
                                          │
                                          ▼
                               RegenerationWorker 执行
                                          │
                                          ▼
                               重建 A 的完整时间线
                               （包括所有关注者的历史帖子）
                                          │
                                          ▼
                               结果：A 的时间线 ✅ 现在包含 B 的帖子
```

### 3.7 时间线合并 vs 时间线重建：完整对比

| 维度 | 时间线合并（MergeWorker） | 时间线重建（RegenerationWorker） |
|------|---------------------------|----------------------------------|
| **触发时机** | 关注/批准时 | 不活跃用户重新登录时 |
| **触发条件** | 自动 | `inactive_since_duration?` 为 true |
| **范围** | 仅新关注者的历史帖子（最多 200 条） | 所有关注者的帖子，重建完整时间线 |
| **活跃用户检查** | ✅ 检查，不活跃则跳过 | ❌ 不检查，强制重建 |
| **调用入口** | `FollowService#direct_follow!`, `FollowRequest#authorize!` | `User#prepare_returning_user!` |
| **性能影响** | 轻量，只处理一个关注者 | 较重，处理所有关注者 |
| **去重机制** | 依赖 Redis ZSET 天然去重 | 同上 |

### 3.8 活跃用户门槛的设计意图

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                         活跃用户门槛的设计哲学                                         │
└──────────────────────────────────────────────────────────────────────────────────────┘

                    为什么要区分活跃和不活跃用户？
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
           ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
           │  性能优化   │    │  资源优化   │    │  用户体验   │
           └─────────────┘    └─────────────┘    └─────────────┘
                    │                  │                  │
                    ▼                  ▼                  ▼
    不活跃用户可能很久不登录，         Redis 内存有限，         活跃用户需要实时体验，
    现在就处理时间线是浪费。         800 条上限是硬约束。      不活跃用户下次登录
    推迟到他们真正登录时。           优先保证活跃用户。         时重建也完全可以。
                    │                  │                  │
                    └──────────────────┼──────────────────┘
                                       ▼
                         核心原则：按需计算
```

### 3.9 关键代码位置汇总

| 检查点 | 文件路径 | 行号 | 影响场景 |
|--------|----------|------|----------|
| `signed_in_recently` 定义 | `app/models/concerns/user/activity.rb` | L15 | 所有场景 |
| `merge_into_home` 检查 | `app/lib/feed_manager.rb` | L129 | 时间线回填 |
| `push_to_home` 检查 | `app/lib/feed_manager.rb` | L77 | 实时分发 |
| `followers_for_local_distribution` | `app/models/concerns/account/interactions.rb` | L219 | 实时分发筛选 |
| `lists_for_local_distribution` | `app/models/concerns/account/interactions.rb` | L225 | 列表分发筛选 |
| `regenerate_feed!` | `app/models/user.rb` | L516 | 重建入口 |
| `prepare_returning_user!` | `app/models/user.rb` | L505 | 登录触发 |

---

## 完整修正流程图

### 1. 批准流程完整修正图

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                    关注请求批准流程（修正版）                                          │
└──────────────────────────────────────────────────────────────────────────────────────┘

  用户 A 发送关注请求
         │
         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│  步骤 1：创建 FollowRequest 记录                                                       │
│  ┌────────────────────────────────────────────────────────────────────────────────┐  │
│  │ FollowRequest                                                                   │  │
│  │ ├── account_id: A（关注者）                                                    │  │
│  │ ├── target_account_id: B（被关注者）                                           │  │
│  │ ├── show_reblogs: 继承自请求选项                                                │  │
│  │ ├── notify: 继承自请求选项                                                       │  │
│  │ └── uri: ActivityPub 活动 URI（如适用）                                        │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
         用户 B 批准请求
         │
         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│  步骤 2：FollowRequestsController#authorize                                            │
│  ┌────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 参数解析：                                                                       │  │
│  │ ├── account = Account.find(params[:id]) → 请求发起者 A                         │  │
│  │ └── current_account = 当前登录用户 → 被请求者 B                                │  │
│  │                                                                                  │  │
│  │ 调用：                                                                           │  │
│  │ └── AuthorizeFollowService.new.call(account, current_account)                 │  │
│  │                                                                                  │  │
│  │ 发送通知：                                                                       │  │
│  │ └── LocalNotificationWorker.perform_async(                                     │  │
│  │       current_account.id,  → ❗ 接收者是 B（被关注者）                          │  │
│  │       ...,                                                                        │  │
│  │       'follow'         → 通知类型："你有新粉丝"                                │  │
│  │     )                                                                             │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│  步骤 3：AuthorizeFollowService#call                                                    │
│  ┌────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 检查 skip_follow_request 参数：                                                  │  │
│  │                                                                                  │  │
│  │ ┌────────────────────────────────────────────────────────────────────────────┐ │  │
│  │ │ 快速通道（skip_follow_request: true）                                        │ │  │
│  │ │ ├── 内存创建 FollowRequest（不存数据库）                                      │ │  │
│  │ │ ├── ❌ 不调用 authorize!                                                     │ │  │
│  │ │ ├── ❌ 不触发时间线合并                                                       │ │  │
│  │ │ └── 适用场景：Follow 已存在，重复 Follow 活动                                │ │  │
│  │ └────────────────────────────────────────────────────────────────────────────┘ │  │
│  │                                                                                  │  │
│  │ ┌────────────────────────────────────────────────────────────────────────────┐ │  │
│  │ │ 正常批准（skip_follow_request: false）                                       │ │  │
│  │ │ ├── 查询数据库中的 FollowRequest                                              │ │  │
│  │ │ ├── 调用 follow_request.authorize!                                           │ │  │
│  │ │ ├── ✅ 触发时间线合并                                                         │ │  │
│  │ │ └── 适用场景：正常关注流程                                                    │ │  │
│  │ └────────────────────────────────────────────────────────────────────────────┘ │  │
│  │                                                                                  │  │
│  │ 发送 Accept 活动（如果 A 是远程）：                                              │  │
│  │ └── ActivityPub::DeliveryWorker.perform_async(...)                            │  │
│  │     发送到 A 的远程实例 inbox                                                   │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│  步骤 4：FollowRequest#authorize!（仅正常批准）                                       │
│  ┌────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段 1：创建 Follow 记录                                                         │  │
│  │ follow = account.follow!(                                                        │  │
│  │   target_account,                                                                │  │
│  │   reblogs: show_reblogs,     ◄── 继承 FollowRequest 的设置                      │  │
│  │   notify: notify,               ◄── 继承                                         │  │
│  │   languages: languages,         ◄── 继承                                         │  │
│  │   uri: uri,                      ◄── 继承                                         │  │
│  │   bypass_limit: true            ◄── 绕过数量限制                                  │  │
│  │ )                                                                                  │  │
│  │                                                                                  │  │
│  │ 阶段 2：本地关注者特殊处理（if account.local?）                                   │  │
│  │                                                                                  │  │
│  │   2.1 更新 ListAccount：                                                         │  │
│  │   ListAccount.where(follow_request: self).update_all(                           │  │
│  │     follow_request_id: nil,                                                      │  │
│  │     follow_id: follow.id            ◄── 关联到新创建的 Follow                   │  │
│  │   )                                                                               │  │
│  │                                                                                  │  │
│  │   2.2 触发时间线合并：                                                           │  │
│  │   MergeWorker.perform_async(target_account.id, account.id, 'home')            │  │
│  │   MergeWorker.push_bulk(...) { ... 'list' }                                     │  │
│  │                                                                                  │  │
│  │ 阶段 3：删除 FollowRequest                                                        │  │
│  │ destroy!                                                                         │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│  步骤 5：MergeWorker 执行（时间线回填）                                                │
│  ┌────────────────────────────────────────────────────────────────────────────────┐  │
│  │ MergeWorker#perform                                                              │  │
│  │ └── FeedManager.instance.merge_into_home(from_account, into_account)           │  │
│  │                                                                                  │  │
│  │ FeedManager#merge_into_home                                                      │  │
│  │ ┌────────────────────────────────────────────────────────────────────────────┐ │  │
│  │ │ 关键检查：return unless into_account.user&.signed_in_recently?             │ │  │
│  │ │                                                                              │ │  │
│  │ │ 如果 A 是活跃用户（最近 7 天登录过）：                                        │ │  │
│  │ │ ├── 获取 B 的最近 200 条帖子                                                 │ │  │
│  │ │ ├── 应用过滤规则（拉黑、静音、语言等）                                        │ │  │
│  │ │ ├── 插入到 A 的 Redis 时间线                                                 │ │  │
│  │ │ └── 裁剪到 800 条上限                                                        │ │  │
│  │ │                                                                              │ │  │
│  │ │ 如果 A 不活跃：                                                               │ │  │
│  │ │ └── 直接 return，不执行合并                                                  │ │  │
│  │ │     A 的时间线不会出现 B 的历史帖子                                          │ │  │
│  │ └────────────────────────────────────────────────────────────────────────────┘ │  │
│  │                                                                                  │  │
│  │ ensure 块（无论活跃与否都执行）：                                                │  │
│  │ └── HomeFeed.new(into_account).regeneration_finished!                         │  │
│  │     标记时间线重建完成                                                           │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│  步骤 6：后续分发（B 发布新帖时）                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐  │
│  │ FanOutOnWriteService#deliver_to_all_followers!                                  │  │
│  │ └── @account.followers_for_local_distribution                                   │  │
│  │                                                                                  │  │
│  │ followers_for_local_distribution 定义：                                          │  │
│  │ followers.local                                                                 │  │
│  │   .joins(:user)                                                                 │  │
│  │   .merge(User.signed_in_recently)  ◄── 只选择活跃用户                         │  │
│  │                                                                                  │  │
│  │ 结果：                                                                           │  │
│  │ ├── 活跃用户 A：✅ B 的新帖实时推送到 A 的时间线                               │  │
│  │ └── 不活跃用户 A：❌ B 的新帖不推送                                           │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│  步骤 7：不活跃用户重新登录时的恢复                                                    │
│  ┌────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 用户 A 重新登录                                                                   │  │
│  │ └── Devise::SessionsController#create                                            │  │
│  │     └── User#prepare_returning_user!                                             │  │
│  │                                                                                  │  │
│  │ prepare_returning_user!                                                          │  │
│  │ ├── ActivityTracker.record('activity:logins', id)  ◄── 更新登录时间           │  │
│  │ └── regenerate_feed! if inactive_since_duration?   ◄── 长时间不活跃则重建     │  │
│  │                                                                                  │  │
│  │ regenerate_feed!                                                                 │  │
│  │ ├── home_feed = HomeFeed.new(account)                                           │  │
│  │ ├── return if home_feed.regenerating?          ◄── 避免重复重建               │  │
│  │ ├── home_feed.regeneration_in_progress!         ◄── 标记正在重建               │  │
│  │ └── RegenerationWorker.perform_async(account_id) ◄── 异步重建                 │  │
│  │                                                                                  │  │
│  │ 结果：                                                                           │  │
│  │ └── A 的时间线被完整重建，包含所有关注者（包括 B）的历史帖子                   │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 关键代码位置索引

| 功能 | 文件路径 | 关键方法/行号 |
|------|----------|---------------|
| **批准控制器** | `app/controllers/api/v1/follow_requests_controller.rb` | `#authorize` L14 |
| **批准服务** | `app/services/authorize_follow_service.rb` | `#call` L6, `#create_notification` L20 |
| **核心批准方法** | `app/models/follow_request.rb` | `#authorize!` L34 |
| **快速通道检测** | `app/lib/activitypub/activity/follow.rb` | L15-21, L23-29 |
| **活跃用户定义** | `app/models/concerns/user/activity.rb` | `signed_in_recently` L15, `ACTIVE_DURATION` L12 |
| **时间线回填检查** | `app/lib/feed_manager.rb` | `#merge_into_home` L129 |
| **实时推送检查** | `app/lib/feed_manager.rb` | `#push_to_home` L77 |
| **分发筛选** | `app/models/concerns/account/interactions.rb` | `#followers_for_local_distribution` L216 |
| **时间线重建入口** | `app/models/user.rb` | `#regenerate_feed!` L516, `#prepare_returning_user!` L501 |
| **HomeFeed 状态** | `app/models/home_feed.rb` | `#regenerating?` L13, `#regeneration_in_progress!` L19 |

---

## 总结

### 1. 批准通知的正确流向

| 场景 | 通知接收方 | 通知内容 | 发送方式 |
|------|-----------|----------|----------|
| 本地用户 B 批准本地用户 A | **B（被关注者）** | "你有新粉丝 A" | `LocalNotificationWorker` |
| 本地用户 A 批准后 | **无** | 无 | ❌ 不发送 |
| 本地用户 B 批准远程用户 A | **B（被关注者）** | "你有新粉丝 A" | `LocalNotificationWorker` |
| 远程用户 A 批准后 | **远程实例** | Accept 活动 | `ActivityPub::DeliveryWorker` |

**关键修正**：
- 本地关注者 A **不会**收到"关注已被批准"的显式通知
- 只能通过时间线出现被关注者的帖子来间接得知

### 2. 快速通道 vs 正常批准

| 维度 | 快速通道 | 正常批准 |
|------|----------|----------|
| **触发条件** | Follow 记录已存在 | FollowRequest 等待批准 |
| **`skip_follow_request`** | `true` | `false` 或未设置 |
| **调用 `authorize!`** | ❌ 否 | ✅ 是 |
| **时间线合并** | ❌ 否（之前已合并） | ✅ 是 |
| **ListAccount 更新** | ❌ 否 | ✅ 是 |
| **设计目的** | 分布式同步、重复请求处理 | 正常关注流程 |

### 3. 活跃用户门槛的完整影响

| 场景 | 活跃用户（< 7 天） | 不活跃用户（> 7 天） | 不活跃用户重新登录后 |
|------|---------------------|----------------------|---------------------|
| **时间线回填** | ✅ 执行 | ❌ 跳过 | ✅ 重建时包含 |
| **实时分发** | ✅ 推送 | ❌ 不推送 | ✅ 登录后变为活跃，开始接收 |
| **Redis 资源** | 优先分配 | 延迟分配 | 按需分配 |
| **用户体验** | 实时 | 延迟但不丢失 | 完整恢复 |

**核心设计原则**：
- 活跃用户：实时体验，优先保证
- 不活跃用户：延迟计算，资源优化
- 重新登录：完整重建，无数据丢失

---

*分析日期：2026-05-02*
*基于 Mastodon 代码库版本：当前工作目录版本*
