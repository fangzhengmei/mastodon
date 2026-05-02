# Mastodon 关注请求批准流程分析报告

## 目录
1. [概述](#概述)
2. [关注请求的发起](#关注请求的发起)
3. [批准入口：API 与 ActivityPub](#批准入口api-与-activitypub)
4. [状态转换核心：FollowRequest#authorize!](#状态转换核心followrequestauthorize)
5. [列表关联更新机制](#列表关联更新机制)
6. [时间线合并接入](#时间线合并接入)
7. [通知发送流程](#通知发送流程)
8. [与正式分发机制的接入](#与正式分发机制的接入)
9. [完整数据流架构图](#完整数据流架构图)
10. [关键代码位置索引](#关键代码位置索引)

---

## 概述

当用户 A 向用户 B 发送关注请求后，在请求被批准前：
- **数据状态**：存在 `FollowRequest` 记录，不存在 `Follow` 记录
- **时间线**：A 的主页时间线不包含 B 的帖子
- **分发**：B 的新帖不会分发给 A
- **列表**：如果 A 将 B 加入列表，`ListAccount` 记录关联的是 `follow_request_id` 而非 `follow_id`

请求批准后，系统需要完成以下关键转换：
1. **状态转换**：`FollowRequest` → `Follow`
2. **关联修复**：`ListAccount` 的 `follow_request_id` → `follow_id`
3. **历史回填**：B 的历史帖子合并到 A 的时间线
4. **通知发送**：通知 A 关注已生效
5. **分发接入**：A 正式进入 B 的分发列表

---

## 关注请求的发起

### 1. 场景分类

| 发起方 | 目标方 | 是否需要请求 | 触发条件 |
|--------|--------|-------------|----------|
| 本地 | 本地 | 是 | 目标账户 locked=true 或发起方被 silenced |
| 本地 | 本地 | 否 | 目标账户公开且发起方正常 |
| 本地 | 远程 | 总是 | 远程账户走 ActivityPub 协议 |
| 远程 | 本地 | 是 | 远程 Follow 活动到达 |

### 2. 本地用户发起关注请求

```ruby
# app/services/follow_service.rb:38-43
def call(source_account, target_account, options = {})
  # ... 前置检查 ...
  
  if (@target_account.locked? && !@options[:bypass_locked]) || @source_account.silenced? || @target_account.activitypub?
    request_follow!  # 需要请求批准
  elsif @target_account.local?
    direct_follow!   # 直接关注
  end
end
```

**request_follow! 方法**：

```ruby
# app/services/follow_service.rb:68-77
def request_follow!
  # 1. 创建 FollowRequest 记录
  follow_request = @source_account.request_follow!(@target_account, **follow_options)

  # 2. 根据目标类型发送通知
  if @target_account.local?
    # 本地目标：发送本地通知
    LocalNotificationWorker.perform_async(@target_account.id, follow_request.id, follow_request.class.name, 'follow_request')
  elsif @target_account.activitypub?
    # 远程目标：发送 ActivityPub Follow 活动
    ActivityPub::DeliveryWorker.perform_async(build_json(follow_request), @source_account.id, @target_account.inbox_url)
  end

  follow_request
end
```

### 3. 远程用户发起关注请求

当远程实例发送 Follow 活动到本地 inbox 时：

```ruby
# app/lib/activitypub/activity/follow.rb:6-39
class ActivityPub::Activity::Follow < ActivityPub::Activity
  def perform
    target_account = account_from_uri(object_uri)

    return if target_account.nil? || !target_account.local? || delete_arrived_first?(@json['id'])

    # 更新已存在的关注请求的 URI
    existing_follow_request = ::FollowRequest.find_by(account: @account, target_account: target_account)
    unless existing_follow_request.nil?
      existing_follow_request.update!(uri: @json['id'])
      return
    end

    # 检查是否被拉黑或域名被屏蔽
    if target_account.blocking?(@account) || target_account.domain_blocking?(@account.domain) || target_account.moved? || target_account.instance_actor?
      reject_follow_request!(target_account)
      return
    end

    # 快速通道：已存在 Follow 记录（可能之前批准过）
    existing_follow = ::Follow.find_by(account: @account, target_account: target_account)
    unless existing_follow.nil?
      existing_follow.update!(uri: @json['id'])
      AuthorizeFollowService.new.call(@account, target_account, skip_follow_request: true, follow_request_uri: @json['id'])
      return
    end

    # 创建 FollowRequest 记录
    follow_request = FollowRequest.create!(account: @account, target_account: target_account, uri: @json['id'])

    # 根据目标账户状态决定是否自动批准
    if target_account.locked? || @account.silenced?
      # 需要手动批准：发送关注请求通知
      LocalNotificationWorker.perform_async(target_account.id, follow_request.id, 'FollowRequest', 'follow_request')
    else
      # 自动批准
      AuthorizeFollowService.new.call(@account, target_account)
      LocalNotificationWorker.perform_async(target_account.id, ::Follow.find_by(account: @account, target_account: target_account).id, 'Follow', 'follow')
    end
  end
end
```

### 4. FollowRequest 数据模型

```ruby
# app/models/follow_request.rb:18-66
class FollowRequest < ApplicationRecord
  belongs_to :account           # 请求发起者
  belongs_to :target_account, class_name: 'Account'  # 被请求者

  has_one :notification, as: :activity, dependent: :destroy

  # 关注选项（与 Follow 模型相同）
  # - show_reblogs: 是否显示转发
  # - notify: 是否开启新帖通知
  # - languages: 语言过滤
  # - uri: ActivityPub 活动 URI

  def authorize!
    # 核心批准逻辑，见后续章节
  end

  alias reject! destroy!  # 拒绝即删除
end
```

---

## 批准入口：API 与 ActivityPub

### 1. 本地 API 入口

当本地用户 B 批准关注请求时，通过 API 控制器：

```ruby
# app/controllers/api/v1/follow_requests_controller.rb:14-18
def authorize
  # 调用批准服务
  AuthorizeFollowService.new.call(account, current_account)
  
  # 发送关注成功通知给批准者（current_account 是被关注者）
  LocalNotificationWorker.perform_async(
    current_account.id, 
    Follow.find_by(account: account, target_account: current_account).id, 
    'Follow', 
    'follow'
  )
  
  render json: account, serializer: REST::RelationshipSerializer, relationships: relationships
end
```

**关键点**：
- `account` 是请求发起者（A）
- `current_account` 是被请求者（B，当前登录用户）
- 通知发送给 **B**（被关注者），告知其有了新粉丝

### 2. ActivityPub 入口

当远程实例发送 Accept Follow 活动时：

```ruby
# app/lib/activitypub/activity/accept.rb:3-86
class ActivityPub::Activity::Accept < ActivityPub::Activity
  def perform
    return accept_follow_for_relay if relay_follow?
    
    # 场景 1：根据 object URI 查找 FollowRequest
    return accept_follow!(follow_request_from_object) unless follow_request_from_object.nil?
    return accept_quote!(quote_request_from_object) unless quote_request_from_object.nil?
    return accept_feature_request! if Mastodon::Feature.collections_enabled? && feature_request_from_object.present?

    # 场景 2：处理内嵌的 Follow 对象
    case @object['type']
    when 'Follow'
      accept_embedded_follow
    when 'QuoteRequest'
      accept_embedded_quote_request
    end
  end

  private

  # 处理内嵌 Follow 对象的场景
  def accept_embedded_follow
    target_account = account_from_uri(target_uri)

    return if target_account.nil? || !target_account.local?

    # 查找：@account 是远程批准者，target_account 是本地请求者
    follow_request = FollowRequest.find_by(account: target_account, target_account: @account)
    accept_follow!(follow_request)
  end

  # 核心批准方法
  def accept_follow!(request)
    return if request.nil?

    # 检查是否是首次被关注（用于触发远程账户刷新）
    is_first_follow = !request.target_account.followers.local.exists?
    request.authorize!  # 调用 FollowRequest#authorize!

    # 如果是首次被本地用户关注，刷新远程账户信息
    RemoteAccountRefreshWorker.perform_async(request.target_account_id) if is_first_follow
  end
end
```

**两种 Accept 活动格式**：

| 格式 | 场景 | 处理方式 |
|------|------|----------|
| 引用 URI | `object: "https://remote/users/a/follows/123"` | 通过 `follow_request_from_object` 按 URI 查找 |
| 内嵌对象 | `object: { type: "Follow", actor: "...", object: "..." }` | 通过 `accept_embedded_follow` 处理 |

### 3. AuthorizeFollowService

批准服务作为统一入口：

```ruby
# app/services/authorize_follow_service.rb:3-26
class AuthorizeFollowService < BaseService
  include Payloadable

  def call(source_account, target_account, **options)
    if options[:skip_follow_request]
      # 跳过 FollowRequest 查找的场景（用于快速通道）
      follow_request = FollowRequest.new(
        account: source_account, 
        target_account: target_account, 
        uri: options[:follow_request_uri]
      )
    else
      # 正常流程：查找并标记已授权
      follow_request = FollowRequest.find_by!(account: source_account, target_account: target_account)
      follow_request.authorize!  # 调用核心转换逻辑
    end

    # 如果请求发起者是远程账户，发送 Accept 活动回执
    create_notification(follow_request) if !source_account.local? && source_account.activitypub?
    
    follow_request
  end

  private

  def create_notification(follow_request)
    # 发送 Accept Follow 活动到远程发起者的 inbox
    ActivityPub::DeliveryWorker.perform_async(
      build_json(follow_request), 
      follow_request.target_account_id, 
      follow_request.account.inbox_url
    )
  end

  def build_json(follow_request)
    serialize_payload(follow_request, ActivityPub::AcceptFollowSerializer).to_json
  end
end
```

---

## 状态转换核心：FollowRequest#authorize!

这是整个批准流程的**核心方法**，完成从 `FollowRequest` 到 `Follow` 的完整转换。

### 完整代码分析

```ruby
# app/models/follow_request.rb:34-46
def authorize!
  # ============================================
  # 阶段 1：创建正式 Follow 记录
  # ============================================
  follow = account.follow!(
    target_account, 
    reblogs: show_reblogs,      # 继承关注请求中的设置
    notify: notify,              # 继承通知设置
    languages: languages,        # 继承语言过滤
    uri: uri,                    # 继承 ActivityPub URI
    bypass_limit: true           # 绕过关注数量限制
  )

  # ============================================
  # 阶段 2：处理本地关注者的特殊逻辑
  # ============================================
  if account.local?
    # 2.1 更新 ListAccount 关联：follow_request_id → follow_id
    ListAccount.where(follow_request: self).update_all(
      follow_request_id: nil, 
      follow_id: follow.id
    )
    
    # 2.2 触发主页时间线合并
    MergeWorker.perform_async(target_account.id, account.id, 'home')
    
    # 2.3 触发表时间线合并
    MergeWorker.push_bulk(
      account.owned_lists.with_list_account(target_account).pluck(:id)
    ) do |list_id|
      [target_account.id, list_id, 'list']
    end
  end

  # ============================================
  # 阶段 3：删除 FollowRequest 记录
  # ============================================
  destroy!
end
```

### 阶段详解

#### 阶段 1：创建 Follow 记录

`account.follow!` 方法定义在 `Account::Interactions`：

```ruby
# app/models/concerns/account/interactions.rb:46-57
def follow!(other_account, reblogs: nil, notify: nil, languages: nil, uri: nil, rate_limit: false, bypass_limit: false)
  # 查找或创建 Follow 记录
  rel = active_relationships.create_with(
    show_reblogs: reblogs.nil? || reblogs,
    notify: notify.nil? ? false : notify,
    languages: languages,
    uri: uri,
    rate_limit: rate_limit,
    bypass_follow_limit: bypass_limit
  ).find_or_create_by!(target_account: other_account)

  # 更新选项（如果已存在）
  rel.show_reblogs = reblogs   unless reblogs.nil?
  rel.notify       = notify    unless notify.nil?
  rel.languages    = languages unless languages.nil?

  rel.save! if rel.changed?
  rel
end
```

**关键继承机制**：
- `show_reblogs`：是否显示转发（默认 true）
- `notify`：是否开启新帖通知（默认 false）
- `languages`：语言过滤数组
- `uri`：ActivityPub 活动 URI（用于远程同步）

#### 阶段 2：本地关注者特殊处理

**为什么只有本地关注者需要这些处理？**

| 处理项 | 本地关注者 | 远程关注者 | 原因 |
|--------|-----------|-----------|------|
| ListAccount 更新 | 需要 | 不需要 | 列表是本地概念，远程实例自行管理 |
| 时间线合并 | 需要 | 不需要 | 远程时间线由远程实例管理 |
| 列表时间线合并 | 需要 | 不需要 | 同上 |

**如何判断是否为本地关注者？**

```ruby
# app/models/account.rb:208-210
def local?
  domain.nil?  # domain 字段为空表示是本实例账户
end
```

#### 阶段 3：删除 FollowRequest

```ruby
destroy!  # 软删除？不，FollowRequest 没有 discard 机制
```

注意：`FollowRequest` 没有使用 `Discard` 模块，`destroy!` 是物理删除。

---

## 列表关联更新机制

### 1. ListAccount 数据模型

```ruby
# app/models/list_account.rb:14-44
class ListAccount < ApplicationRecord
  belongs_to :list
  belongs_to :account
  
  # 关键：可以关联 Follow 或 FollowRequest
  belongs_to :follow, optional: true
  belongs_to :follow_request, optional: true

  # 有效列表成员的条件：有 follow_id
  scope :active, -> { where.not(follow_id: nil) }

  # 创建时自动设置关联
  before_validation :set_follow, unless: :list_owner_account_is_account?

  private

  def set_follow
    # 优先查找 Follow 记录
    self.follow = Follow.find_by(account_id: list.account_id, target_account_id: account.id)
    
    # 如果没有 Follow，查找 FollowRequest
    self.follow_request = FollowRequest.find_by(account_id: list.account_id, target_account_id: account.id) if follow.nil?
  end

  def validate_relationship
    return if list_owner_account_is_account?
    
    # 必须有关联关系
    errors.add(:account_id, :must_be_following) if follow_id.nil? && follow_request_id.nil?
    # ...
  end
end
```

### 2. 关联状态转换图

```
用户 A 将 B 加入列表时：

          ┌─────────────────────────────────────────┐
          │           ListAccount 记录               │
          │  list_id: A的列表                         │
          │  account_id: B                            │
          │                                         │
          │  follow_id: nil ◄───────────────────────┤
          │  follow_request_id: 123 (若有请求)       │
          └─────────────────────────────────────────┘

B 批准请求后：

          ┌─────────────────────────────────────────┐
          │           ListAccount 记录               │
          │  list_id: A的列表                         │
          │  account_id: B                            │
          │                                         │
          │  follow_id: 456 (新创建的 Follow.id) ◄───┤
          │  follow_request_id: nil                  │
          └─────────────────────────────────────────┘
```

### 3. 更新 SQL 分析

```ruby
ListAccount.where(follow_request: self).update_all(
  follow_request_id: nil, 
  follow_id: follow.id
)
```

**等效 SQL**：
```sql
UPDATE list_accounts 
SET follow_request_id = NULL, 
    follow_id = <follow.id>
WHERE follow_request_id = <self.id>;
```

### 4. 列表时间线的激活条件

```ruby
# app/models/list.rb:28
has_many :active_accounts, -> { merge(ListAccount.active) }, through: :list_accounts, source: :account
```

`ListAccount.active` 定义为 `where.not(follow_id: nil)`，这意味着：

| 状态 | follow_id | follow_request_id | 是否在 active_accounts 中 |
|------|-----------|-------------------|---------------------------|
| 请求中 | nil | 有值 | ❌ 否 |
| 已批准 | 有值 | nil | ✅ 是 |

**这就是为什么需要更新 ListAccount 的原因**：只有将 `follow_request_id` 转换为 `follow_id`，列表时间线的分发机制才能正确工作。

---

## 时间线合并接入

批准后，需要将被关注者的**历史帖子**合并到关注者的时间线中，使用的是与**直接关注**完全相同的机制。

### 1. 触发 MergeWorker

```ruby
# app/models/follow_request.rb:39-42
if account.local?
  # 主页时间线合并
  MergeWorker.perform_async(target_account.id, account.id, 'home')
  
  # 列表时间线合并
  MergeWorker.push_bulk(account.owned_lists.with_list_account(target_account).pluck(:id)) do |list_id|
    [target_account.id, list_id, 'list']
  end
end
```

**参数含义**：
- `target_account.id`：被关注者 B（帖子来源）
- `account.id`：关注者 A（时间线所有者）或 `list_id`（列表 ID）
- `'home'` / `'list'`：时间线类型

### 2. MergeWorker 执行

```ruby
# app/workers/merge_worker.rb:3-45
class MergeWorker
  include Sidekiq::Worker
  include Redisable
  include DatabaseHelper

  def perform(from_account_id, into_id, type = 'home')
    @from_account = Account.find(from_account_id)  # B

    case type
    when 'home'
      merge_into_home!(into_id)  # into_id 是 A 的账户 ID
    when 'list'
      merge_into_list!(into_id)  # into_id 是列表 ID
    end
  end

  private

  def merge_into_home!(into_account_id)
    @into_account = Account.find(into_account_id)  # A

    FeedManager.instance.merge_into_home(@from_account, @into_account)
  ensure
    # 标记时间线重建完成
    HomeFeed.new(@into_account).regeneration_finished!
  end

  def merge_into_list!(into_list_id)
    @into_list = List.find(into_list_id)
    FeedManager.instance.merge_into_list(@from_account, @into_list)
  end
end
```

### 3. FeedManager 合并逻辑

```ruby
# app/lib/feed_manager.rb:128-150
def merge_into_home(from_account, into_account)
  # 优化：仅合并给最近活跃的用户
  return unless into_account.user&.signed_in_recently?

  timeline_key = key(:home, into_account.id)
  aggregate    = into_account.user&.aggregates_reblogs?
  
  # 获取被关注者的最近帖子（最多 200 条）
  query = from_account.statuses
    .list_eligible_visibility
    .includes(reblog: :account)
    .limit(FeedManager::MAX_ITEMS / 4)  # MAX_ITEMS = 800，即 200 条

  # 优化：如果时间线已满，只获取比最旧条目新的帖子
  if redis.zcard(timeline_key) >= FeedManager::MAX_ITEMS / 4
    oldest_home_score = redis.zrange(timeline_key, 0, 0, with_scores: true).first.last.to_i
    query = query.where('id > ?', oldest_home_score)
  end

  statuses = query.to_a
  crutches = build_crutches(into_account.id, statuses)  # 预加载过滤数据

  # 逐条过滤并插入
  statuses.each do |status|
    next if filter_from_home(status, into_account.id, crutches)  # 应用过滤规则
    add_to_feed(:home, into_account.id, status, aggregate_reblogs: aggregate)
  end

  trim(:home, into_account.id)  # 裁剪到最大容量
end
```

### 4. 与直接关注的对比

| 维度 | 直接关注 (`FollowService#direct_follow!`) | 批准关注请求 (`FollowRequest#authorize!`) |
|------|--------------------------------------------|--------------------------------------------|
| 创建 Follow | 是 | 是 |
| 触发 MergeWorker | 是 | 是 |
| 触发列表 MergeWorker | 是 | 是 |
| 发送通知给被关注者 | 是 | 是（在 Controller 中） |
| 发送 Accept 到远程 | - | 是（如果发起者是远程） |

**关键结论**：批准后的时间线合并逻辑与直接关注**完全相同**，确保了两种关注路径的一致性。

---

## 通知发送流程

### 1. 通知类型与时机

批准流程涉及两种通知：

| 通知类型 | 接收者 | 触发时机 | 代码位置 |
|----------|--------|----------|----------|
| `follow` | 被关注者 B | API 批准后 | `FollowRequestsController#authorize` |
| `Accept Follow` 活动 | 远程关注者 A | 批准服务中 | `AuthorizeFollowService#create_notification` |

### 2. 本地通知：被关注者收到新粉丝通知

```ruby
# app/controllers/api/v1/follow_requests_controller.rb:16
LocalNotificationWorker.perform_async(
  current_account.id,                                    # 接收者：被关注者 B
  Follow.find_by(account: account, target_account: current_account).id,  # Follow 记录 ID
  'Follow',                                               # activity_class_name
  'follow'                                                # notification_type
)
```

**通知流向**：
```
A 发送关注请求 → B 收到 follow_request 通知
                    ↓
              B 批准请求
                    ↓
              B 收到 follow 通知（"A 开始关注你了"）
```

### 3. 远程通知：Accept Follow 活动

```ruby
# app/services/authorize_follow_service.rb:20-22
def create_notification(follow_request)
  ActivityPub::DeliveryWorker.perform_async(
    build_json(follow_request),                    # Accept 活动 JSON
    follow_request.target_account_id,               # 签名账户：本地批准者 B
    follow_request.account.inbox_url                 # 目标 inbox：远程发起者 A
  )
end
```

**ActivityPub 通知流程**：
```
本地实例 B                          远程实例 A
     │                                   │
     │  POST /users/a/inbox              │
     │  {                                │
     │    "type": "Accept",              │
     │    "actor": "https://B/users/b", │
     │    "object": {                    │
     │      "type": "Follow",            │
     │      "actor": "https://A/users/a",│
     │      "object": "https://B/users/b"│
     │    }                              │
     │  }                               ──►│
     │                                   │  处理 Accept 活动
     │                                   │  触发本地批准逻辑
     │                                   │  合并时间线
     │◄──────────────────────────────────│
```

### 4. 为什么没有通知给关注者 A？

分析代码后发现，**批准流程中没有给关注者 A 发送"关注已批准"的通知**。

让我确认一下：

```ruby
# FollowRequestsController#authorize
def authorize
  AuthorizeFollowService.new.call(account, current_account)
  
  # 这个通知发送给 current_account (B)，不是 account (A)
  LocalNotificationWorker.perform_async(
    current_account.id,  # B
    Follow.find_by(account: account, target_account: current_account).id,
    'Follow',
    'follow'
  )
end
```

**设计意图推测**：
- 被关注者 B 需要知道"谁关注了我"
- 关注者 A 在关注时已经知道请求已发送
- A 可以通过时间线出现 B 的帖子间接得知批准已生效
- 或者 A 可以通过刷新关注列表看到状态变化

---

## 与正式分发机制的接入

### 1. 分发入口回顾

新帖发布时的分发流程：

```ruby
# app/services/post_status_service.rb:162-171
def postprocess_status!
  # ...
  
  # 本地分发
  DistributionWorker.perform_async(@status.id)
  
  # 远端分发
  ActivityPub::DistributionWorker.perform_async(@status.id)
end
```

### 2. 本地分发：followers_for_local_distribution

```ruby
# app/services/fan_out_on_write_service.rb:114-119
def deliver_to_all_followers!
  @account.followers_for_local_distribution.select(:id).reorder(nil).find_in_batches do |followers|
    FeedInsertWorker.push_bulk(followers) do |follower|
      [@status.id, follower.id, 'home', { 'update' => update? }]
    end
  end
end
```

**关键方法**：

```ruby
# app/models/concerns/account/interactions.rb:216-220
def followers_for_local_distribution
  followers.local                    # 仅本地账户
    .joins(:user)
    .merge(User.signed_in_recently)  # 仅最近活跃用户
end
```

**关联关系链**：
```
Account (B)
    │
    ├───> passive_relationships (Follow 表，target_account_id = B.id)
    │         │
    │         └───> account (Follow 表的 account_id 关联)
    │                   │
    │                   └───> followers 关联方法
    │
    └───> followers_for_local_distribution
              │
              ├───> .local: domain IS NULL
              └───> .signed_in_recently: 最近活跃
```

### 3. Follow vs FollowRequest 的关联差异

```ruby
# app/models/concerns/account/interactions.rb:10-16
# Follow 关系
has_many :active_relationships,  foreign_key: 'account_id', class_name: 'Follow'
has_many :passive_relationships, foreign_key: 'target_account_id', class_name: 'Follow'
has_many :following, through: :active_relationships,  source: :target_account
has_many :followers, through: :passive_relationships, source: :account

# FollowRequest 关系（独立）
has_many :follow_requests, dependent: :destroy
```

**关键差异**：
- `followers` 方法通过 `Follow` 表关联
- `FollowRequest` 是独立的关联，不影响 `followers`

**批准前后的对比**：

| 状态 | B.followers 包含 A 吗？ | A 在分发列表中吗？ |
|------|-------------------------|-------------------|
| 请求中 | ❌ 否（FollowRequest 不计入） | ❌ 否 |
| 已批准 | ✅ 是（Follow 已创建） | ✅ 是 |

### 4. 列表分发：lists_for_local_distribution

```ruby
# app/services/fan_out_on_write_service.rb:130-135
def deliver_to_lists!
  @account.lists_for_local_distribution.select(:id).reorder(nil).find_in_batches do |lists|
    FeedInsertWorker.push_bulk(lists) do |list|
      [@status.id, list.id, 'list', { 'update' => update? }]
    end
  end
end
```

**lists_for_local_distribution 方法**：

```ruby
# app/models/concerns/account/interactions.rb:222-226
def lists_for_local_distribution
  scope = lists.joins(account: :user)
  scope.where.not(list_accounts: { follow_id: nil }).or(scope.where(account_id: id))
    .merge(User.signed_in_recently)
end
```

**ListAccount 过滤条件**：
- `where.not(list_accounts: { follow_id: nil })`：必须有正式 Follow 关联

这就是为什么 `FollowRequest#authorize!` 中需要更新 `ListAccount` 的原因：

```
批准前：
  ListAccount: follow_id=nil, follow_request_id=123
  → 不在 lists_for_local_distribution 结果中
  → 新帖不会分发到列表时间线

批准后：
  ListAccount: follow_id=456, follow_request_id=nil
  → 在 lists_for_local_distribution 结果中
  → 新帖正常分发到列表时间线
```

### 5. 远程分发：StatusReachFinder

```ruby
# app/workers/activitypub/distribution_worker.rb:20-24
def inboxes
  @inboxes ||= StatusReachFinder.new(@status).inboxes
end
```

```ruby
# app/lib/status_reach_finder.rb:83-112
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
```

**远程分发的关键点**：
- 使用 `@status.account.followers`（与本地分发相同的关联）
- `followers` 关联只包括 `Follow` 记录，不包括 `FollowRequest`
- 批准后，远程关注者自动进入分发列表

### 6. 接入点总结

```
┌─────────────────────────────────────────────────────────────────────┐
│                    分发机制接入点总结                                  │
└─────────────────────────────────────────────────────────────────────┘

                              新帖发布
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │   PostStatusService     │
                    └─────────────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    ▼                           ▼
            ┌───────────────┐          ┌───────────────────────┐
            │ Distribution  │          │ ActivityPub::         │
            │    Worker     │          │   DistributionWorker  │
            └───────────────┘          └───────────────────────┘
                    │                           │
                    ▼                           ▼
            ┌───────────────┐          ┌───────────────────────┐
            │FanOutOnWrite  │          │    StatusReachFinder  │
            │   Service     │          │                       │
            └───────────────┘          │  @account.followers   │
                    │                   │  .inboxes              │
                    ▼                   └───────────────────────┘
    ┌───────────────┴───────────────┐
    ▼                               ▼
┌───────────────────┐    ┌───────────────────────────┐
│followers_for_local│    │ lists_for_local_distribution│
│   _distribution   │    │                           │
│                   │    │ 必须满足：                 │
│ .local            │    │ ListAccount.follow_id     │
│ .signed_in_recently│   │ IS NOT NULL               │
└───────────────────┘    └───────────────────────────┘
           │                            │
           │                            │
           └────────────────────────────┘
                        │
                        ▼
              只有 Follow 记录会被分发
              FollowRequest 不计入分发
```

---

## 完整数据流架构图

### 1. 全景流程图

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                         关注请求批准完整数据流                                         │
└──────────────────────────────────────────────────────────────────────────────────────┘

  远程用户 A                    本地实例                    本地用户 B
      │                            │                            │
      │  POST /inbox               │                            │
      │  Follow 活动               │                            │
      ├───────────────────────────►│                            │
      │                            ▼                            │
      │              ┌─────────────────────────┐                │
      │              │ ActivityPub::Activity:: │                │
      │              │        Follow           │                │
      │              └─────────────────────────┘                │
      │                            │                            │
      │                            ▼                            │
      │              ┌─────────────────────────┐                │
      │              │   FollowRequest.create  │                │
      │              │   (domain: "remote-a") │                │
      │              └─────────────────────────┘                │
      │                            │                            │
      │                            ▼                            │
      │              ┌─────────────────────────┐                │
      │              │ LocalNotificationWorker │                │
      │              └─────────────────────────┘                │
      │                            │                            │
      │                            └───────────────────────────►│
      │                            │  follow_request 通知       │
      │                            │                            │
      │                            │◄───────────────────────────┤
      │                            │       批准操作              │
      │                            │                            │
      │                            ▼                            │
      │              ┌─────────────────────────┐                │
      │              │ FollowRequestsController│                │
      │              │     #authorize          │                │
      │              └─────────────────────────┘                │
      │                            │                            │
      │                            ▼                            │
      │              ┌─────────────────────────┐                │
      │              │  AuthorizeFollowService │                │
      │              └─────────────────────────┘                │
      │                            │                            │
      │                            ▼                            │
      │              ┌─────────────────────────┐                │
      │              │ FollowRequest#authorize!│                │
      │              └─────────────────────────┘                │
      │                            │                            │
      │          ┌─────────────────┼─────────────────┐          │
      │          ▼                 ▼                 ▼          │
      │  ┌─────────────┐ ┌─────────────┐ ┌─────────────────┐ │
      │  │  Follow.create│ │ListAccount  │ │ MergeWorker     │ │
      │  │             │ │  .update_all │ │ (home + lists)  │ │
      │  └─────────────┘ └─────────────┘ └─────────────────┘ │
      │          │                 │                 │          │
      │          ▼                 ▼                 ▼          │
      │  ┌─────────────────────────────────────────────────┐   │
      │  │                                                   │   │
      │  │   接入正式分发机制：                              │   │
      │  │   • B.followers 现在包含 A                      │   │
      │  │   • ListAccount.follow_id 已设置               │   │
      │  │   • A 的时间线包含 B 的历史帖子                 │   │
      │  │                                                   │   │
      │  └─────────────────────────────────────────────────┘   │
      │                            │                            │
      │                            ▼                            │
      │              ┌─────────────────────────┐                │
      │              │ ActivityPub::Delivery   │                │
      │◄─────────────┤     Worker              │                │
      │  Accept 活动 │                         │                │
      │              └─────────────────────────┘                │
      │                                                            │
```

### 2. 状态转换详细图

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                         批准前后的状态对比                                             │
└──────────────────────────────────────────────────────────────────────────────────────┘

╔══════════════════════════════════════════════════════════════════════════════════════╗
║                           批准前：FollowRequest 状态                                   ║
╠══════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                          ║
║  数据库记录：                                                                             ║
║  ┌──────────────────────────────────────────────────────────────────────────────────┐  ║
║  │ FollowRequest                                                                      │  ║
║  │ ├── id: 123                                                                        │  ║
║  │ ├── account_id: A (关注者)                                                         │  ║
║  │ ├── target_account_id: B (被关注者)                                                │  ║
║  │ ├── show_reblogs: true                                                             │  ║
║  │ ├── notify: false                                                                  │  ║
║  │ ├── languages: nil                                                                 │  ║
║  │ └── uri: "https://remote-a/activity/follow/456"                                   │  ║
║  └──────────────────────────────────────────────────────────────────────────────────┘  ║
║                                                                                          ║
║  ┌──────────────────────────────────────────────────────────────────────────────────┐  ║
║  │ ListAccount (A 将 B 加入了列表)                                                    │  ║
║  │ ├── id: 789                                                                        │  ║
║  │ ├── list_id: A's list                                                              │  ║
║  │ ├── account_id: B                                                                  │  ║
║  │ ├── follow_id: NULL                    ◄─── 注意这里                               │  ║
║  │ └── follow_request_id: 123              ◄─── 关联的是请求                          │  ║
║  └──────────────────────────────────────────────────────────────────────────────────┘  ║
║                                                                                          ║
║  分发状态：                                                                               ║
║  • B.followers.include?(A) → ❌  false (FollowRequest 不计入)                        ║
║  • A 在 B.followers_for_local_distribution 中吗？ → ❌  否                            ║
║  • ListAccount.active 包含此记录吗？ → ❌  否 (follow_id 为 nil)                     ║
║  • A 的时间线有 B 的帖子吗？ → ❌  否                                                  ║
║                                                                                          ║
╚══════════════════════════════════════════════════════════════════════════════════════╝

╔══════════════════════════════════════════════════════════════════════════════════════╗
║                           批准后：Follow 状态                                           ║
╠══════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                          ║
║  数据库记录：                                                                             ║
║  ┌──────────────────────────────────────────────────────────────────────────────────┐  ║
║  │ Follow (新创建)                                                                     │  ║
║  │ ├── id: 456                                                                        │  ║
║  │ ├── account_id: A                                                                   │  ║
║  │ ├── target_account_id: B                                                            │  ║
║  │ ├── show_reblogs: true              ◄─── 继承自 FollowRequest                      │  ║
║  │ ├── notify: false                   ◄─── 继承自 FollowRequest                      │  ║
║  │ ├── languages: nil                  ◄─── 继承自 FollowRequest                      │  ║
║  │ └── uri: "https://remote-a/activity/follow/456"   ◄─── 继承                      │  ║
║  └──────────────────────────────────────────────────────────────────────────────────┘  ║
║                                                                                          ║
║  ┌──────────────────────────────────────────────────────────────────────────────────┐  ║
║  │ FollowRequest                                                                      │  ║
║  │ └── 已删除 (destroy!)                                                              │  ║
║  └──────────────────────────────────────────────────────────────────────────────────┘  ║
║                                                                                          ║
║  ┌──────────────────────────────────────────────────────────────────────────────────┐  ║
║  │ ListAccount (已更新)                                                                │  ║
║  │ ├── id: 789                                                                        │  ║
║  │ ├── list_id: A's list                                                              │  ║
║  │ ├── account_id: B                                                                  │  ║
║  │ ├── follow_id: 456                    ◄─── 已更新为 Follow.id                      │  ║
║  │ └── follow_request_id: NULL             ◄─── 已清除                                │  ║
║  └──────────────────────────────────────────────────────────────────────────────────┘  ║
║                                                                                          ║
║  分发状态：                                                                               ║
║  • B.followers.include?(A) → ✅  true                                                  ║
║  • A 在 B.followers_for_local_distribution 中吗？ → ✅  是 (如果 A 本地且活跃)        ║
║  • ListAccount.active 包含此记录吗？ → ✅  是                                          ║
║  • A 的时间线有 B 的帖子吗？ → ✅  是 (MergeWorker 已执行)                             ║
║                                                                                          ║
╚══════════════════════════════════════════════════════════════════════════════════════╝
```

### 3. 异步任务时序图

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                         批准流程中的异步任务时序                                        │
└──────────────────────────────────────────────────────────────────────────────────────┘

时间轴 →

  T0          T1          T2          T3          T4          T5
  │           │           │           │           │           │
  ▼           ▼           ▼           ▼           ▼           ▼

┌───────┐
│API 请求│──────────────────────────────────────────────────────────►
│(同步)  │
└───────┘
     │
     │ FollowRequest.authorize! (同步执行)
     │ ├── Follow.create
     │ ├── ListAccount.update_all
     │ └── destroy!
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    异步任务入队（Sidekiq）                           │
└─────────────────────────────────────────────────────────────────────┘
     │
     ├───► MergeWorker (home) ──────► FeedManager.merge_into_home
     │           │                           │
     │           │                           ├───► 读取 B 的最近帖子
     │           │                           ├───► 应用过滤规则
     │           │                           ├───► Redis ZADD 到 A 的时间线
     │           │                           └───► 裁剪到 800 条
     │           │
     ├───► MergeWorker (list) ──────► FeedManager.merge_into_list
     │           │                           │
     │           │                           └───► 类似流程，插入列表时间线
     │           │
     ├───► LocalNotificationWorker (发送给 B)
     │           │
     │           └───► NotifyService
     │                   │
     │                   ├───► 创建 Notification 记录
     │                   ├───► WebSocket 推送
     │                   └───► Web Push / Email (如需)
     │
     └───► ActivityPub::DeliveryWorker (如果 A 是远程)
                 │
                 └───► POST Accept 活动到 A 的 inbox
```

---

## 关键代码位置索引

| 功能 | 文件路径 | 关键方法/行号 |
|------|----------|---------------|
| **关注请求批准 API** | `app/controllers/api/v1/follow_requests_controller.rb` | `#authorize` L14 |
| **批准服务** | `app/services/authorize_follow_service.rb` | `#call` L6, `#create_notification` L20 |
| **核心状态转换** | `app/models/follow_request.rb` | `#authorize!` L34 |
| **快速通道自动批准** | `app/lib/activitypub/activity/follow.rb` | `#perform` L24-29 |
| **远程 Accept 处理** | `app/lib/activitypub/activity/accept.rb` | `#perform` L4, `#accept_follow!` L29 |
| **列表关联模型** | `app/models/list_account.rb` | `set_follow` L29, `active` scope L23 |
| **列表查询 scope** | `app/models/list.rb` | `with_list_account` L36 |
| **时间线合并 Worker** | `app/workers/merge_worker.rb` | `#perform` L8, `#merge_into_home!` L25 |
| **时间线合并逻辑** | `app/lib/feed_manager.rb` | `#merge_into_home` L128 |
| **本地分发筛选** | `app/models/concerns/account/interactions.rb` | `#followers_for_local_distribution` L216 |
| **列表分发筛选** | `app/models/concerns/account/interactions.rb` | `#lists_for_local_distribution` L222 |
| **远程分发目标** | `app/lib/status_reach_finder.rb` | `#followers_scope` L104 |
| **拒绝关注服务** | `app/services/reject_follow_service.rb` | `#call` L6 |

---

## 总结

### 1. 批准流程的五个核心阶段

| 阶段 | 操作 | 代码位置 | 关键影响 |
|------|------|----------|----------|
| **1. 入口** | API 或 ActivityPub | `FollowRequestsController` / `ActivityPub::Activity::Accept` | 决定批准触发方式 |
| **2. 状态转换** | `FollowRequest` → `Follow` | `FollowRequest#authorize!` L35 | 正式建立关注关系 |
| **3. 关联修复** | `ListAccount` 更新 | `FollowRequest#authorize!` L38 | 列表时间线激活 |
| **4. 历史回填** | `MergeWorker` 执行 | `FollowRequest#authorize!` L39-42 | 时间线出现历史帖子 |
| **5. 通知回执** | 本地/远程通知 | `Controller` / `AuthorizeFollowService` | 用户感知批准结果 |

### 2. 与直接关注的一致性

批准后的处理逻辑与 `FollowService#direct_follow!` **完全一致**：

```ruby
# FollowService#direct_follow! (直接关注)
def direct_follow!
  follow = @source_account.follow!(@target_account, ...)
  LocalNotificationWorker.perform_async(...)
  MergeWorker.perform_async(@target_account.id, @source_account.id, 'home')
  MergeWorker.push_bulk(...) { ... 'list' }
end

# FollowRequest#authorize! (批准请求)
def authorize!
  follow = account.follow!(target_account, ...)
  if account.local?
    ListAccount.where(follow_request: self).update_all(...)
    MergeWorker.perform_async(target_account.id, account.id, 'home')
    MergeWorker.push_bulk(...) { ... 'list' }
  end
  destroy!
end
```

**差异点**：
- 批准流程多了 `ListAccount.update_all` 步骤（因为列表可能在请求期间创建）
- 通知发送位置不同（直接关注在 Service 中，批准在 Controller 中）
- 批准流程需要处理远程发起者的 Accept 回执

### 3. 分发接入的关键设计

批准后能够自动接入分发机制，依赖于以下设计：

1. **统一关联入口**：`Account#followers` 只通过 `Follow` 表关联，不包括 `FollowRequest`
2. **列表激活条件**：`ListAccount.active` 依赖 `follow_id IS NOT NULL`
3. **异步解耦**：时间线合并通过 `MergeWorker` 异步执行，不阻塞批准请求
4. **设置继承**：`show_reblogs`、`notify`、`languages` 等设置完整继承

### 4. 边缘场景处理

| 场景 | 处理方式 | 代码位置 |
|------|----------|----------|
| 远程发起者 | 发送 Accept Follow 活动回执 | `AuthorizeFollowService#create_notification` |
| 快速通道（已存在 Follow） | 跳过 FollowRequest，直接批准 | `ActivityPub::Activity::Follow#perform` L24-29 |
| 首次被本地用户关注 | 触发远程账户刷新 | `ActivityPub::Activity::Accept#accept_follow!` L35 |
| 拒绝请求 | 直接删除 FollowRequest，发送 Reject 活动 | `RejectFollowService` |

---

*分析日期：2026-05-02*
*基于 Mastodon 代码库版本：当前工作目录版本*
