# Mastodon 列表与远端账号同步机制

## 1. 核心数据模型与关系

### 1.1 数据库外键约束（关键设计）

ListAccount 表有两个关键的外键约束，这是理解列表成员生命周期的核心：

```ruby
# db/schema.rb
add_foreign_key "list_accounts", "follows", on_delete: :cascade
add_foreign_key "list_accounts", "follow_requests", on_delete: :cascade
```

**这意味着**：
- 当 `Follow` 被删除时，引用它的 `ListAccount` 会被**级联删除**
- 当 `FollowRequest` 被删除时，引用它的 `ListAccount` 会被**级联删除**

这是数据库级别的约束，不是应用层逻辑。

### 1.2 ListAccount 模型

ListAccount 是列表与账号之间的关联表，其生命周期与关注关系**完全绑定**。

```ruby
# app/models/list_account.rb
class ListAccount < ApplicationRecord
  belongs_to :list
  belongs_to :account
  belongs_to :follow, optional: true
  belongs_to :follow_request, optional: true

  validates :account_id, uniqueness: { scope: :list_id }
  validate :validate_relationship

  scope :active, -> { where.not(follow_id: nil) }

  before_validation :set_follow, unless: :list_owner_account_is_account?
end
```

**关键字段**：
| 字段 | 说明 | 对列表的影响 |
|------|------|-------------|
| `list_id` | 列表 ID | 必填 |
| `account_id` | 成员账号 ID | 必填 |
| `follow_id` | 关注关系 ID | **关键**：有值 = 活跃成员，参与 fan-out |
| `follow_request_id` | 关注请求 ID | 有值 = 待审核成员，不参与 fan-out |

**验证逻辑**：
```ruby
def validate_relationship
  return if list_owner_account_is_account?  # 列表所有者自己不需要关注关系

  errors.add(:account_id, :must_be_following) if follow_id.nil? && follow_request_id.nil?
end
```

**关键结论**：除了列表所有者自己，其他账号必须有 `follow_id` 或 `follow_request_id` 才能被添加到列表。

### 1.3 List 模型

列表通过 `list_accounts` 关联表管理成员。

```ruby
# app/models/list.rb
has_many :list_accounts, inverse_of: :list, dependent: :destroy
has_many :accounts, through: :list_accounts
has_many :active_accounts, -> { merge(ListAccount.active) }, through: :list_accounts, source: :account

scope :with_list_account, ->(account) { joins(:list_accounts).where(list_accounts: { account: }) }
```

**关键**：`has_many :list_accounts, dependent: :destroy` 意味着列表被删除时，所有 ListAccount 也会被删除。

### 1.4 Account 模型

通过 `domain` 字段区分本地和远端账号：

```ruby
# app/models/account.rb
def local?
  domain.nil?
end

def remote?
  !domain.nil?
end
```

## 2. 列表成员的完整生命周期

### 2.1 生命周期状态图

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                    列表成员生命周期                         │
                    └─────────────────────────────────────────────────────────┘

  ┌──────────────────┐                    ┌──────────────────┐
  │   待审核状态      │                    │    活跃状态        │
  │                  │                    │                  │
  │ follow_request_id│    关注请求被接受   │   follow_id      │
  │ follow_id: nil   │ ─────────────────► │  有有效值         │
  │                  │                    │                  │
  │ 不参与 fan-out   │                    │  参与 fan-out     │
  │ 历史状态不合并    │                    │  历史状态被合并    │
  └──────────────────┘                    └──────────────────┘
           │                                       │
           │ 关注请求被拒绝/取消                    │
           │ FollowRequest 被删除                  │ 取消关注
           │ 触发 ON DELETE CASCADE                │ Follow 被删除
           │                                       │ 触发 ON DELETE CASCADE
           ▼                                       ▼
    ┌──────────────────┐                    ┌──────────────────┐
    │    被移除         │◄───────────────────│    被移除         │
    │                  │   显式从列表移除     │                  │
    │ ListAccount 被   │   ListAccount 被    │ ListAccount 被   │
    │ 级联删除         │   直接删除          │ 级联删除         │
    │                  │                    │                  │
    │ 不再出现在列表    │                    │ 不再出现在列表    │
    │ 时间线状态被清理  │                    │ 时间线状态被清理  │
    └──────────────────┘                    └──────────────────┘
```

### 2.2 状态一：待审核（只有 follow_request_id）

**触发条件**：
1. 用户 A 发送关注请求给用户 B（B 的账号是 locked）
2. 用户 A 同时把用户 B 添加到列表

**内部流程**：
```ruby
# app/models/list_account.rb
before_validation :set_follow, unless: :list_owner_account_is_account?

def set_follow
  self.follow = Follow.find_by(account_id: list.account_id, target_account_id: account.id)
  self.follow_request = FollowRequest.find_by(account_id: list.account_id, target_account_id: account.id) if follow.nil?
end
```

**结果**：
- ListAccount 的 `follow_request_id` 指向 FollowRequest
- ListAccount 的 `follow_id` 为 nil
- `ListAccount.active` scope 不会包含这个记录

**对列表时间线的影响**：
- `lists_for_local_distribution` 不会包含这个列表
```ruby
def lists_for_local_distribution
  scope.where.not(list_accounts: { follow_id: nil }).or(scope.where(account_id: id))
  #                                          ^^^^^^^^^^^^^^^^^^^^^
  #                                    follow_id 为 nil 时不满足这个条件
end
```
- 用户 B 的新状态不会分发到这个列表时间线
- 历史状态不会被合并

### 2.3 状态二：活跃（有 follow_id）

**触发条件 A：关注请求被接受**

```ruby
# app/models/follow_request.rb:38
def authorize!
  follow = account.follow!(target_account, ...)

  if account.local?
    # 关键：先显式更新 ListAccount，避免被级联删除
    ListAccount.where(follow_request: self).update_all(follow_request_id: nil, follow_id: follow.id)
    
    MergeWorker.perform_async(target_account.id, account.id, 'home')
    MergeWorker.push_bulk(account.owned_lists.with_list_account(target_account).pluck(:id)) do |list_id|
      [target_account.id, list_id, 'list']
    end
  end

  destroy!  # 现在删除 FollowRequest 不会级联删除 ListAccount 了
end
```

**关键逻辑解释**：
1. 先创建新的 `Follow` 记录
2. **显式更新** ListAccount：`follow_request_id: nil, follow_id: follow.id`
3. 然后才 `destroy!` FollowRequest
4. 如果不先更新，FollowRequest 的 `destroy!` 会触发 `ON DELETE CASCADE` 删除 ListAccount

**触发条件 B：直接关注（不需要审核）**

```ruby
# app/services/follow_service.rb:80
def direct_follow!
  follow = @source_account.follow!(@target_account, ...)

  MergeWorker.perform_async(@target_account.id, @source_account.id, 'home')
  MergeWorker.push_bulk(@source_account.owned_lists.with_list_account(@target_account).pluck(:id)) do |list_id|
    [@target_account.id, list_id, 'list']
  end

  follow
end
```

**后续流程：添加账号到列表时**

```ruby
# app/services/add_accounts_to_list_service.rb:50
def merge_account_ids
  ListAccount.where(list: @list, account: @accounts).where.not(follow_id: nil).pluck(:account_id)
  #                                                         ^^^^^^^^^^^^^^^^^^^^
  #                                                  只选择有活跃关注关系的账号
end

def merge_into_list!
  MergeWorker.push_bulk(merge_account_ids) do |account_id|
    [account_id, @list.id, 'list']
  end
end
```

**活跃状态的特征**：
- ListAccount 的 `follow_id` 有有效值
- `ListAccount.active` scope 包含这个记录
- `lists_for_local_distribution` 包含这个列表
- 新状态会分发到列表时间线
- 历史状态会被合并

### 2.4 状态三：被移除（ListAccount 被删除）

**情况 A：取消关注（最常见）**

```ruby
# app/services/unfollow_service.rb:25
def unfollow!
  follow = Follow.find_by(account: @follower, target_account: @followee)
  return unless follow

  # 关键注释：List members are removed immediately with the follow relationship removal,
  # so we need to fetch the list IDs first
  #
  # 翻译：列表成员会随着关注关系的删除而立即被移除，
  # 所以我们需要先获取列表 ID
  list_ids = @follower.owned_lists.with_list_account(@followee).pluck(:list_id) unless @options[:skip_unmerge]

  follow.destroy!  # 触发 ON DELETE CASCADE，ListAccount 被级联删除

  # ... 发送 ActivityPub 消息

  unless @options[:skip_unmerge]
    UnmergeWorker.perform_async(@followee.id, @follower.id, 'home')
    UnmergeWorker.push_bulk(list_ids) do |list_id|
      [@followee.id, list_id, 'list']
    end
  end
end
```

**关键流程**：
1. **先**获取 `list_ids`（在删除 Follow 之前）
2. `follow.destroy!` 触发数据库 `ON DELETE CASCADE`
3. ListAccount 被**级联删除**（不是 `follow_id` 变为 nil）
4. 触发 `UnmergeWorker` 从 Redis 时间线移除状态

**情况 B：关注请求被拒绝/取消**

```ruby
# app/models/follow_request.rb:48
alias reject! destroy!
```

当 `FollowRequest#destroy!` 被调用时：
- 数据库 `ON DELETE CASCADE` 触发
- 引用该 FollowRequest 的 ListAccount 被级联删除

**情况 C：显式从列表移除**

```ruby
# app/services/remove_accounts_from_list_service.rb:20
def call(list, accounts)
  unmerge_from_list!  # 先触发 UnmergeWorker
  update_list!         # 再删除 ListAccount
end

def update_list!
  ListAccount.where(list: @list, account: @accounts).destroy_all
end

def unmerge_from_list!
  UnmergeWorker.push_bulk(unmerge_account_ids) do |account_id|
    [account_id, @list.id, 'list']
  end
end

def unmerge_account_ids
  ListAccount.where(list: @list, account: @accounts).where.not(follow_id: nil).pluck(:account_id)
end
```

**情况 D：列表被删除**

```ruby
# app/models/list.rb
has_many :list_accounts, inverse_of: :list, dependent: :destroy
```

当 List 被删除时，所有 ListAccount 也会被删除（应用层 `dependent: :destroy`）。

### 2.5 特殊情况：列表所有者自己

列表所有者可以把自己添加到列表，不需要关注关系：

```ruby
# app/models/list_account.rb
def list_owner_account_is_account?
  list.account_id == account_id
end

def validate_relationship
  return if list_owner_account_is_account?  # 跳过验证
  # ...
end

def set_follow
  return if list_owner_account_is_account?  # 跳过设置
  # ...
end
```

**特征**：
- ListAccount 的 `follow_id` 和 `follow_request_id` 都为 nil
- 但 `lists_for_local_distribution` 通过 `or(scope.where(account_id: id))` 包含
- 自己的状态会通过 `deliver_to_self!` 路径分发

## 3. 列表时间线 Fan-out 机制

### 3.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         列表时间线 Fan-out 架构                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐     ┌──────────────────┐     ┌──────────────────────┐   │
│  │  状态发布/更新 │────►│ FanOutOnWrite    │────►│ lists_for_local_     │   │
│  │              │     │ Service          │     │ distribution         │   │
│  └──────────────┘     └──────────────────┘     └──────────────────────┘   │
│                                                                              │
│                                                        │                     │
│                                                        ▼                     │
│                                              ┌──────────────────┐          │
│                                              │ FeedInsertWorker │          │
│                                              │ (异步队列)        │          │
│                                              └──────────────────┘          │
│                                                        │                     │
│                                                        ▼                     │
│                                              ┌──────────────────┐          │
│                                              │ FeedManager      │          │
│                                              │ .push_to_list()  │          │
│                                              └──────────────────┘          │
│                                                        │                     │
│                              ┌─────────────────────────┼─────────────────┐   │
│                              │                         │                 │   │
│                              ▼                         ▼                 ▼   │
│                     ┌──────────────┐          ┌──────────────┐  ┌─────────┐│
│                     │ Redis 有序集合│          │ PushUpdate   │  │ 裁剪时间线 ││
│                     │ ZADD 操作     │          │ Worker       │  │ (800条)  ││
│                     └──────────────┘          └──────────────┘  └─────────┘│
│                                                        │                     │
│                                                        ▼                     │
│                                              ┌──────────────────┐          │
│                                              │ Redis PUBLISH    │          │
│                                              │ timeline:list:id │          │
│                                              └──────────────────┘          │
│                                                        │                     │
│                                                        ▼                     │
│                                              ┌──────────────────┐          │
│                                              │ Streaming API    │          │
│                                              │ (Node.js)        │          │
│                                              └──────────────────┘          │
│                                                        │                     │
│                                                        ▼                     │
│                                              ┌──────────────────┐          │
│                                              │ WebSocket/SSE    │          │
│                                              │ 前端实时更新      │          │
│                                              └──────────────────┘          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 触发点：FanOutOnWriteService

无论本地还是远端账号发布状态，都会经过这个服务。

```ruby
# app/services/fan_out_on_write_service.rb
def call(status, options = {})
  @status    = status
  @account   = status.account
  @options   = options

  fan_out_to_local_recipients!
  fan_out_to_public_recipients! if broadcastable?
  fan_out_to_public_streams! if broadcastable?
end

def fan_out_to_local_recipients!
  deliver_to_self!
  # ...

  case @status.visibility.to_sym
  when :public, :unlisted, :private
    deliver_to_all_followers!
    deliver_to_lists!  # 分发到列表时间线
  when :limited
    deliver_to_mentioned_followers!
  else
    deliver_to_mentioned_followers!
    deliver_to_conversation!
  end
end
```

### 3.3 核心：lists_for_local_distribution

这个方法决定哪些列表会收到状态更新。

```ruby
# app/models/concerns/account/interactions.rb
def lists_for_local_distribution
  scope = lists.joins(account: :user)
  scope.where.not(list_accounts: { follow_id: nil }).or(scope.where(account_id: id))
    .merge(User.signed_in_recently)
end
```

**拆解分析**：
1. `lists.joins(account: :user)` - 关联到列表所有者的 User
2. `scope.where.not(list_accounts: { follow_id: nil })` - **ListAccount 的 follow_id 不为空**
3. `.or(scope.where(account_id: id))` - **或者是列表所有者自己**
4. `.merge(User.signed_in_recently)` - **且列表所有者最近登录过**

**关键结论**：
- 只有**活跃成员**（`follow_id` 不为空）才会触发列表时间线更新
- 列表所有者自己即使没有 `follow_id` 也会触发
- 只更新活跃用户的时间线（性能优化）

### 3.4 分发流程

```ruby
# app/services/fan_out_on_write_service.rb
def deliver_to_lists!
  @account.lists_for_local_distribution.select(:id).reorder(nil).find_in_batches do |lists|
    FeedInsertWorker.push_bulk(lists) do |list|
      [@status.id, list.id, 'list', { 'update' => update? }]
    end
  end
end
```

**异步处理**：
- 使用 `push_bulk` 批量推入 Sidekiq 队列
- 每个列表一个异步任务

### 3.5 FeedInsertWorker

```ruby
# app/workers/feed_insert_worker.rb
def perform(status_id, id, type = 'home', options = {})
  with_primary do
    @type      = type.to_sym
    @status    = Status.find(status_id)
    @options   = options.symbolize_keys

    case @type
    when :list
      @list     = List.find(id)
      @follower = @list.account
    end
  end

  with_read_replica do
    check_and_insert
  end
end

def check_and_insert
  filter_result = feed_filter

  if filter_result
    perform_unpush if update?
  else
    perform_push
  end
end

def feed_filter
  case @type
  when :list
    FeedManager.instance.filter(:list, @status, @list)
  end
end

def perform_push
  case @type
  when :list
    FeedManager.instance.push_to_list(@list, @status, update: update?)
  end
end
```

### 3.6 FeedManager.push_to_list

```ruby
# app/lib/feed_manager.rb
def push_to_list(list, status, update: false)
  return false if filter_from_list?(status, list)
  return false unless list.account.user&.signed_in_recently?
  return false unless add_to_feed(:list, list.id, status, aggregate_reblogs: list.account.user&.aggregates_reblogs?)

  trim(:list, list.id)
  PushUpdateWorker.perform_async(list.account_id, status.id, "timeline:list:#{list.id}", { 'update' => update }) if push_update_required?("timeline:list:#{list.id}")
  true
end
```

**步骤拆解**：
1. `filter_from_list?` - 检查回复策略
2. `signed_in_recently?` - 只更新活跃用户
3. `add_to_feed` - Redis ZADD 操作
4. `trim` - 保持 800 条限制
5. `PushUpdateWorker` - 如果有客户端订阅，发送流式更新

### 3.7 过滤逻辑：filter_from_list?

```ruby
# app/lib/feed_manager.rb
def filter_from_list?(status, list)
  if status.reply? && status.in_reply_to_account_id != status.account_id
    should_filter = status.in_reply_to_account_id != list.account_id
    should_filter &&= !list.show_followed?
    should_filter &&= !(list.show_list? && ListAccount.exists?(list_id: list.id, account_id: status.in_reply_to_account_id))

    return !!should_filter
  end

  false
end
```

**回复策略**：
| replies_policy | 行为 |
|----------------|------|
| `list` | 只显示对列表中账号的回复 |
| `followed` | 显示对所有已关注账号的回复 |
| `none` | 不显示任何回复 |

## 4. 远端账号状态变化的完整链路

### 4.1 远端账号发布新状态

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                远端账号发布状态触发 Fan-out 流程                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  远端实例 (mastodon.social)                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 1. @alice@mastodon.social 发布新状态                                    │  │
│  │ 2. 活动流投递到所有关注者的 shared_inbox 或 inbox                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                          │
│                                    │ ActivityPub POST /inbox                  │
│                                    │ (Create Note)                            │
│                                    ▼                                          │
│  本地实例 (local.instance)                                                   │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 3. ActivityPub::InboxesController#create                               │  │
│  │    - 验证 HTTP 签名                                                      │  │
│  │    - 异步处理：ActivityPub::ProcessingWorker                            │  │
│  │                                    │                                     │  │
│  │                                    ▼                                     │  │
│  │ 4. ActivityPub::Activity::Create#perform                                │  │
│  │    - 创建 Status 记录                                                    │  │
│  │    - 分发通知                                                             │  │
│  │    - 检查：if @options[:override_timestamps] || @status.within_retention_period?
│  │                                    │                                     │  │
│  │                                    ▼                                     │  │
│  │ 5. FanOutOnWriteService.call(@status)                                   │  │
│  │    - 与本地账号发布状态完全相同的流程                                      │  │
│  │                                    │                                     │  │
│  │                                    ▼                                     │  │
│  │ 6. @account.lists_for_local_distribution                                │  │
│  │    - 查找包含 @alice@mastodon.social 的列表                              │  │
│  │    - 筛选：follow_id 不为空 且 列表所有者最近登录                         │  │
│  │                                    │                                     │  │
│  │                                    ▼                                     │  │
│  │ 7. FeedInsertWorker.push_bulk                                            │  │
│  │    - 异步处理每个列表                                                     │  │
│  │                                    │                                     │  │
│  │                                    ▼                                     │  │
│  │ 8. FeedManager.push_to_list                                              │  │
│  │    - Redis ZADD "feed:list:{list_id}"                                   │  │
│  │    - PushUpdateWorker (如果有客户端订阅)                                  │  │
│  │                                    │                                     │  │
│  │                                    ▼                                     │  │
│  │ 9. Redis PUBLISH "timeline:list:{list_id}"                              │  │
│  │                                    │                                     │  │
│  │                                    ▼                                     │  │
│  │ 10. Streaming API 服务                                                    │  │
│  │     - WebSocket/SSE 推送到前端                                           │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**关键代码**：

```ruby
# app/lib/activitypub/activity/create.rb
def perform
  # ... 创建 Status
  
  if @options[:override_timestamps] || @status.within_retention_period?
    unless @status.account.local? || @options[:delivered_to_owner]
      # 只在本地接收时通知自己
      NotifyService.new.call(@status.account, :status, @status) if @options[:notify]
    end

    # 关键：与本地账号完全相同的 fan-out 流程
    FanOutOnWriteService.new.call(@status, update: @options[:update], skip_notification: true)
  end
end
```

**重要结论**：远端账号发布的状态，经过 ActivityPub 接收后，使用**完全相同**的 `FanOutOnWriteService` 进行分发，包括列表时间线。

### 4.2 远端账号取消关注

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  远端账号取消关注触发列表成员移除流程                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  远端实例                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 1. @alice@mastodon.social 取消关注 @bob@local.instance                 │  │
│  │ 2. 发送 Undo Follow 活动到 @bob 的 inbox                                 │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                          │
│                                    │ ActivityPub POST /inbox                  │
│                                    │ (Undo Follow)                             │
│                                    ▼                                          │
│  本地实例                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 3. ActivityPub::InboxesController#create                               │  │
│  │    - 异步处理：ActivityPub::ProcessingWorker                            │  │
│  │                                    │                                     │  │
│  │                                    ▼                                     │  │
│  │ 4. ActivityPub::Activity::Undo#perform                                  │  │
│  │    case @object['type']                                                 │  │
│  │    when 'Follow'                                                         │  │
│  │      undo_follow                                                         │  │
│  │                                    │                                     │  │
│  │                                    ▼                                     │  │
│  │ 5. ActivityPub::Activity::Undo#undo_follow                              │  │
│  │    if @account.following?(target_account)                               │  │
│  │      @account.unfollow!(target_account)  # 关键！                        │  │
│  │    elsif @account.requested?(target_account)                            │  │
│  │      FollowRequest.find_by(...).destroy                                 │  │
│  │                                    │                                     │  │
│  │                                    ▼                                     │  │
│  │ 6. UnfollowService.call(@alice, @bob)                                   │  │
│  │    ├──► 先获取 list_ids (在删除 Follow 之前)                              │  │
│  │    ├──► follow.destroy!  (触发 ON DELETE CASCADE)                        │  │
│  │    │         │                                                            │  │
│  │    │         ▼                                                            │  │
│  │    │    7. ListAccount 被级联删除！                                       │  │
│  │    │       (数据库级别，不是应用层)                                         │  │
│  │    │                                                                      │  │
│  │    └──► UnmergeWorker.push_bulk(list_ids)                                │  │
│  │              │                                                            │  │
│  │              ▼                                                            │  │
│  │    8. FeedManager.unmerge_from_list                                       │  │
│  │       - 从 Redis 时间线移除该账号的所有状态                                 │  │
│  │              │                                                            │  │
│  │              ▼                                                            │  │
│  │    9. UI 同步：用户刷新列表时，发现该成员已不在列表中                        │  │
│  │       - 时间线中的旧状态已被移除                                           │  │
│  │       - 新状态不会再分发                                                   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**关键代码**：

```ruby
# app/lib/activitypub/activity/undo.rb
def undo_follow
  target_account = account_from_uri(target_uri)

  return if target_account.nil? || !target_account.local?

  if @account.following?(target_account)
    @account.unfollow!(target_account)  # 调用 UnfollowService
  elsif @account.requested?(target_account)
    FollowRequest.find_by(account: @account, target_account: target_account)&.destroy
  else
    delete_later!(object_uri)
  end
end
```

### 4.3 远端账号被暂停

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    远端账号被暂停时的列表时间线处理                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  场景 A：远端实例发送 Delete 活动                                              │
│  ──────────────────────────────────────                                      │
│                                                                              │
│  远端实例                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 1. @alice@mastodon.social 被暂停/删除                                   │  │
│  │ 2. 发送 Delete 活动到所有关注者的 inbox                                   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                          │
│                                    ▼                                          │
│  本地实例                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 3. ActivityPub::Activity::Delete#perform                               │  │
│  │    - 处理 Delete Actor                                                   │  │
│  │    - account.suspended!                                                 │  │
│  │                                    │                                     │  │
│  │                                    ▼                                     │  │
│  │ 4. SuspendAccountService.call(account)                                  │  │
│  │    ├──► reject_remote_follows!                                          │  │
│  │    │         │                                                           │  │
│  │    │         └──► 强制该账号取消关注所有本地账号                           │  │
│  │    │              - 发送 RejectFollow 到远端实例                         │  │
│  │    │              - follows.each(&:destroy)  (触发 ON DELETE CASCADE)   │  │
│  │    │                                                                      │  │
│  │    ├──► unmerge_from_home_timelines!                                    │  │
│  │    │                                                                      │  │
│  │    └──► unmerge_from_list_timelines!  (关键！)                           │  │
│  │              │                                                            │  │
│  │              ▼                                                            │  │
│  │    5. @account.lists_for_local_distribution                              │  │
│  │       - 查找包含该账号的所有列表                                            │  │
│  │              │                                                            │  │
│  │              ▼                                                            │  │
│  │    6. FeedManager.instance.unmerge_from_list(@account, list)             │  │
│  │       - 从 Redis 时间线移除该账号的所有状态                                 │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ════════════════════════════════════════════════════════════════════════  │
│                                                                              │
│  场景 B：本地管理员手动暂停远端账号                                             │
│  ────────────────────────────────────────────                                │
│                                                                              │
│  1. Admin::Action 创建 suspend 记录                                           │
│  2. Account.suspended!                                                       │
│  3. SuspendAccountService.call(account)  (与场景 A 相同的后续流程)            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**关键代码**：

```ruby
# app/services/suspend_account_service.rb
def call(account)
  return unless account.suspended?

  @account = account

  reject_remote_follows!      # 强制取消关注
  distribute_update_actor!     # 通知其他实例
  unmerge_from_home_timelines! # 从主页时间线移除
  unmerge_from_list_timelines! # 从列表时间线移除
  privatize_media_attachments!
  remove_from_trends!
end

def unmerge_from_list_timelines!
  @account.lists_for_local_distribution.reorder(nil).find_each do |list|
    FeedManager.instance.unmerge_from_list(@account, list)
  end
end

def reject_remote_follows!
  return if @account.local? || !@account.activitypub? || @account.suspension_origin_remote?

  # 当暂停远端账号时，该账号在其源实例上并没有真正被暂停
  # 为了防止它继续接收状态，必须强制它取消关注

  Follow.where(account: @account).find_in_batches do |follows|
    ActivityPub::DeliveryWorker.push_bulk(follows) do |follow|
      [serialize_payload(follow, ActivityPub::RejectFollowSerializer).to_json, follow.target_account_id, @account.inbox_url]
    end

    follows.each(&:destroy)  # 触发 ON DELETE CASCADE，ListAccount 被级联删除
  end
end
```

### 4.4 远端账号恢复（Unsuspend）

```ruby
# app/services/unsuspend_account_service.rb
def call(account)
  @account = account

  refresh_remote_account!  # 从远端刷新账号信息

  return if @account.nil? || @account.suspended?

  merge_into_home_timelines!
  merge_into_list_timelines!  # 合并到列表时间线
  publish_media_attachments!
  distribute_update_actor!
end

def merge_into_list_timelines!
  @account.lists_for_local_distribution.reorder(nil).find_each do |list|
    FeedManager.instance.merge_into_list(@account, list)
  end
end
```

**注意**：恢复时，如果 ListAccount 还存在（即 `follow_id` 还有效），会合并历史状态到列表时间线。但如果之前因为 `reject_remote_follows!` 导致 ListAccount 被级联删除了，就需要重新添加到列表。

## 5. 列表 UI 与数据变化同步

### 5.1 整体同步架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         列表 UI 同步架构                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        数据变化触发点                                    │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │                                                                       │  │
│  │  触发方式 1：状态分发（实时）                                             │  │
│  │  ┌──────────┐    ┌──────────────┐    ┌──────────────┐              │  │
│  │  │ FanOut   │───►│ PushUpdate   │───►│ Redis        │              │  │
│  │  │ Service  │    │ Worker       │    │ PUBLISH      │              │  │
│  │  └──────────┘    └──────────────┘    └──────────────┘              │  │
│  │                                                                       │  │
│  │  触发方式 2：列表成员变化（非实时）                                        │  │
│  │  ┌──────────┐    ┌──────────────┐    ┌──────────────┐              │  │
│  │  │ List-    │───►│ 数据库记录    │───►│ 下次 API 请求 │              │  │
│  │  │ Account  │    │ 变化          │    │ 时刷新       │              │  │
│  │  └──────────┘    └──────────────┘    └──────────────┘              │  │
│  │                                                                       │  │
│  │  触发方式 3：时间线清理（非实时）                                          │  │
│  │  ┌──────────┐    ┌──────────────┐    ┌──────────────┐              │  │
│  │  │ Unmerge  │───►│ Redis ZREM   │───►│ 下次刷新时    │              │  │
│  │  │ Worker   │    │ 操作          │    │ 不显示旧状态  │              │  │
│  │  └──────────┘    └──────────────┘    └──────────────┘              │  │
│  │                                                                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                          │
│                                    ▼                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        Streaming API 服务 (Node.js)                      │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │                                                                       │  │
│  │  ┌────────────────────────────────────────────────────────────────┐  │  │
│  │  │ 1. 客户端 WebSocket 连接                                          │  │  │
│  │  │    - 发送：{ "type": "subscribe", "stream": "list", "list": "123" }
│  │  │                                                                   │  │  │
│  │  │ 2. 权限验证                                                        │  │  │
│  │  │    const result = await pgPool.query(                            │  │  │
│  │  │      'SELECT id FROM lists WHERE id = $1 AND account_id = $2',  │  │  │
│  │  │      [listId, accountId]                                          │  │  │
│  │  │    );                                                              │  │  │
│  │  │                                                                   │  │  │
│  │  │ 3. Redis 订阅                                                      │  │  │
│  │  │    redisSubscribeClient.subscribe(`timeline:list:${listId}`)    │  │  │
│  │  │                                                                   │  │  │
│  │  │ 4. 心跳机制                                                        │  │  │
│  │  │    - 每 6 分钟设置 `subscribed:timeline:list:123`               │  │  │
│  │  │    - 有效期 18 分钟                                                │  │  │
│  │  │    - 服务端检查这个键判断是否有客户端订阅                            │  │  │
│  │  └────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                          │
│                                    ▼                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                           前端 UI 层                                     │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │                                                                       │  │
│  │  实时更新（WebSocket）：                                                 │  │
│  │  - 接收到 update 事件 → 插入新状态到列表顶部                             │  │
│  │  - 接收到 status.update 事件 → 更新现有状态                               │  │
│  │  - 接收到 delete 事件 → 从列表移除状态                                   │  │
│  │                                                                       │  │
│  │  非实时更新（需要用户操作）：                                              │  │
│  │  - 列表成员被添加/移除 → 下次请求 GET /api/v1/lists/:id/accounts 时刷新 │
│  │  - 账号被暂停/恢复 → 时间线状态已被 Unmerge/Merge Worker 处理           │  │
│  │                                                                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 流式更新：PushUpdateWorker

```ruby
# app/workers/push_update_worker.rb
def perform(account_id, status_id, timeline_id = nil, options = {})
  @status      = Status.find(status_id)
  @account_id  = account_id
  @timeline_id = timeline_id || "timeline:#{account_id}"
  @options     = options.symbolize_keys

  render_payload!
  publish!
end

def message
  JSON.generate({
    event: update? ? :'status.update' : :update,
    payload: @payload,
  }.as_json)
end

def publish!
  redis.publish(@timeline_id, message)
end
```

**频道命名规范**：
| 时间线类型 | 频道名称 |
|-----------|----------|
| 列表时间线 | `timeline:list:{list_id}` |
| 主页时间线 | `timeline:{account_id}` |
| 通知时间线 | `timeline:{account_id}:notifications` |
| 公共时间线 | `timeline:public` |

### 5.3 推送更新检查

```ruby
# app/lib/feed_manager.rb
def push_update_required?(timeline_key)
  redis.exists?("subscribed:#{timeline_key}")
end

# 调用点
PushUpdateWorker.perform_async(...) if push_update_required?("timeline:list:#{list.id}")
```

**只有当有客户端订阅时才发送流式更新**，这是一个重要的性能优化。

### 5.4 Streaming API 订阅流程

```javascript
// streaming/index.js
case 'list':
  if (!params.list) {
    reject(new RequestError('Missing list name parameter'));
    return;
  }

  authorizeListAccess(params.list, req).then(() => {
    resolve({
      channelIds: [`timeline:list:${params.list}`],
      options: { needsFiltering: false },
    });
  }).catch(() => {
    reject(new AuthenticationError('Not authorized to stream this list'));
  });

  break;
```

**权限验证**：
```javascript
const authorizeListAccess = async (listId, req) => {
  const { accountId } = req;

  const result = await pgPool.query(
    'SELECT id, account_id FROM lists WHERE id = $1 AND account_id = $2 LIMIT 1',
    [listId, accountId]
  );

  if (result.rows.length === 0) {
    throw new AuthenticationError('List not found');
  }
};
```

**规则**：用户只能订阅自己创建的列表。

### 5.5 列表 API 端点

| 端点 | 方法 | 功能 |
|------|------|------|
| `/api/v1/lists/:id` | GET | 获取列表信息 |
| `/api/v1/lists/:id/accounts` | GET | 获取列表成员 |
| `/api/v1/lists/:id/accounts` | POST | 添加成员到列表 |
| `/api/v1/lists/:id/accounts` | DELETE | 从列表移除成员 |
| `/api/v1/timelines/list/:id` | GET | 获取列表时间线 |

### 5.6 同步机制总结

**实时同步（WebSocket）**：
- ✅ 新状态发布 → 立即推送到订阅的客户端
- ✅ 状态更新 → 立即推送到订阅的客户端
- ✅ 状态删除 → 立即推送到订阅的客户端

**非实时同步（需要刷新）**：
- ❌ 列表成员被添加 → 下次获取列表成员 API 时刷新
- ❌ 列表成员被移除 → 下次获取列表成员 API 时刷新
- ❌ 账号被暂停 → 时间线状态已被清理，用户刷新时看不到旧状态
- ❌ 账号被恢复 → 时间线状态已被合并，用户刷新时看到历史状态

**关键设计**：列表成员的增删不会发送实时通知到前端，只有时间线状态变化会发送实时通知。

## 6. 列表成员生命周期与关注关系绑定总结

### 6.1 核心绑定关系

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    列表成员与关注关系的绑定关系                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  数据库约束（核心）：                                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │   list_accounts.follow_id ──────► follows.id                         │  │
│  │         │                                                              │  │
│  │         └── ON DELETE CASCADE                                          │  │
│  │                                                                       │  │
│  │   list_accounts.follow_request_id ──► follow_requests.id             │  │
│  │         │                                                              │  │
│  │         └── ON DELETE CASCADE                                          │  │
│  │                                                                       │  │
│  │   这意味着：                                                            │  │
│  │   - 当 Follow 被删除时，ListAccount 也被删除                           │  │
│  │   - 当 FollowRequest 被删除时，ListAccount 也被删除                    │  │
│  │                                                                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  生命周期状态转换：                                                           │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  初始状态（无 ListAccount）                                              │  │
│  │       │                                                                 │  │
│  │       ├──► 用户发送关注请求 + 添加到列表                                  │  │
│  │       │         │                                                       │  │
│  │       │         ▼                                                       │  │
│  │       │    待审核状态                                                    │  │
│  │       │    - follow_request_id: 有值                                    │  │
│  │       │    - follow_id: nil                                             │  │
│  │       │    - 不参与 fan-out                                             │  │
│  │       │         │                                                       │  │
│  │       │         ├──► 关注请求被接受                                      │  │
│  │       │         │         │                                             │  │
│  │       │         │         ▼                                             │  │
│  │       │         │    FollowRequest#authorize!                           │  │
│  │       │         │    - 创建 Follow                                      │  │
│  │       │         │    - 显式更新 ListAccount:                            │  │
│  │       │         │      follow_request_id: nil, follow_id: new_id       │  │
│  │       │         │    - 删除 FollowRequest（不会级联删除 ListAccount）    │  │
│  │       │         │         │                                             │  │
│  │       │         │         ▼                                             │  │
│  │       │         │    活跃状态                                            │  │
│  │       │         │    - follow_id: 有值                                   │  │
│  │       │         │    - 参与 fan-out                                      │  │
│  │       │         │         │                                             │  │
│  │       │         │         ├──► 取消关注                                  │  │
│  │       │         │         │         │                                   │  │
│  │       │         │         │         ▼                                   │  │
│  │       │         │         │    Follow.destroy!                           │  │
│  │       │         │         │    - 触发 ON DELETE CASCADE                  │  │
│  │       │         │         │         │                                   │  │
│  │       │         │         │         ▼                                   │  │
│  │       │         │         │    被移除状态                                │  │
│  │       │         │         │    - ListAccount 已删除                      │  │
│  │       │         │         │                                              │  │
│  │       │         │         └──► 显式从列表移除                             │  │
│  │       │         │                   │                                    │  │
│  │       │         │                   ▼                                    │  │
│  │       │         │              ListAccount.destroy_all                    │  │
│  │       │         │                   │                                    │  │
│  │       │         │                   ▼                                    │  │
│  │       │         │              被移除状态                                 │  │
│  │       │         │                                                         │  │
│  │       │         └──► 关注请求被拒绝/取消                                   │  │
│  │       │                   │                                               │  │
│  │       │                   ▼                                               │  │
│  │       │              FollowRequest.destroy!                               │  │
│  │       │              - 触发 ON DELETE CASCADE                             │  │
│  │       │                   │                                               │  │
│  │       │                   ▼                                               │  │
│  │       │              被移除状态                                            │  │
│  │       │                                                                   │  │
│  │       └──► 直接关注（不需要审核）+ 添加到列表                               │  │
│                │                                                            │  │
│                ▼                                                            │  │
│           活跃状态（直接进入）                                                │  │
│                                                                             │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  特殊情况：列表所有者自己                                                      │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  列表所有者可以把自己添加到列表：                                          │  │
│  │  - 不需要 follow_id 或 follow_request_id                                │  │
│  │  - list_owner_account_is_account? 返回 true                             │  │
│  │  - 跳过 validate_relationship 验证                                       │  │
│  │  - 跳过 set_follow 回调                                                  │  │
│  │                                                                       │  │
│  │  但 lists_for_local_distribution 通过 or 条件包含：                        │  │
│  │  scope.where.not(list_accounts: { follow_id: nil })                    │  │
│  │    .or(scope.where(account_id: id))  ←── 列表所有者自己                  │  │
│  │                                                                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 关键代码对照表

| 场景 | 触发操作 | ListAccount 变化 | 时间线变化 |
|------|----------|-----------------|-----------|
| 添加已有 Follow 的账号到列表 | `AddAccountsToListService` | 创建记录，`follow_id` 被设置 | `MergeWorker` 合并历史状态 |
| 添加有 FollowRequest 的账号到列表 | `AddAccountsToListService` | 创建记录，`follow_request_id` 被设置 | 无变化（不参与 fan-out） |
| 关注请求被接受 | `FollowRequest#authorize!` | 显式更新：`follow_request_id: nil, follow_id: new_id` | `MergeWorker` 合并历史状态 |
| 取消关注 | `UnfollowService` | `Follow.destroy!` 触发 `ON DELETE CASCADE`，ListAccount 被级联删除 | `UnmergeWorker` 移除时间线状态 |
| 关注请求被拒绝 | `FollowRequest#destroy!` | 触发 `ON DELETE CASCADE`，ListAccount 被级联删除 | 无变化（本来就不参与） |
| 显式从列表移除 | `RemoveAccountsFromListService` | `ListAccount.destroy_all` 直接删除 | `UnmergeWorker` 移除时间线状态 |
| 列表被删除 | `List#destroy` | `dependent: :destroy` 级联删除 | `before_destroy :clean_feed_manager` 回调调用 `FeedManager.clean_feeds!` 清理 Redis 时间线 |

### 6.3 常见问题解答

**Q1: 取消关注后，ListAccount 是被删除还是只是 follow_id 变为 nil？**

**A: 被删除。** 数据库有 `ON DELETE CASCADE` 约束，当 Follow 被删除时，引用它的 ListAccount 会被级联删除。这不是应用层逻辑，而是数据库级别的约束。

**Q2: 为什么 UnfollowService 要先获取 list_ids 再删除 Follow？**

**A:** 因为 `follow.destroy!` 会触发级联删除，ListAccount 会被立即删除。如果不先获取 `list_ids`，之后就无法知道哪些列表需要清理时间线了。

**Q3: 关注请求被接受时，为什么要显式更新 ListAccount？**

**A:** 如果不先更新，`FollowRequest.destroy!` 会触发 `ON DELETE CASCADE` 删除 ListAccount。显式更新 `follow_request_id: nil, follow_id: new_id` 后，ListAccount 不再引用被删除的 FollowRequest，所以不会被级联删除。

**Q4: 远端账号发布状态时，列表时间线如何更新？**

**A:** 与本地账号完全相同。远端状态通过 ActivityPub 接收后，同样使用 `FanOutOnWriteService` 分发，包括 `lists_for_local_distribution` 筛选和 `FeedInsertWorker` 异步处理。

**Q5: 远端账号取消关注时，列表成员会被移除吗？**

**A:** 会。远端通过 ActivityPub 发送 `Undo Follow`，本地通过 `UnfollowService` 处理，`Follow.destroy!` 触发 `ON DELETE CASCADE`，ListAccount 被级联删除。

**Q6: 列表成员变化时，前端会收到实时通知吗？**

**A:** 不会。只有时间线状态变化（新状态、状态更新、状态删除）会通过 WebSocket 实时推送。列表成员的增删需要用户刷新列表成员 API 才能看到。

## 7. 相关文件索引

| 功能 | 文件路径 | 关键代码行 |
|------|----------|-----------|
| ListAccount 模型 | `app/models/list_account.rb` | 整个文件 |
| FollowRequest 授权 | `app/models/follow_request.rb` | 34-46 行 |
| 数据库外键约束 | `db/schema.rb` | 1528-1531 行 |
| UnfollowService | `app/services/unfollow_service.rb` | 25-49 行 |
| AddAccountsToListService | `app/services/add_accounts_to_list_service.rb` | 整个文件 |
| RemoveAccountsFromListService | `app/services/remove_accounts_from_list_service.rb` | 整个文件 |
| lists_for_local_distribution | `app/models/concerns/account/interactions.rb` | 114-119 行 |
| FanOutOnWriteService | `app/services/fan_out_on_write_service.rb` | 整个文件 |
| FeedManager.push_to_list | `app/lib/feed_manager.rb` | 206-214 行 |
| ActivityPub Create 处理 | `app/lib/activitypub/activity/create.rb` | 整个文件 |
| ActivityPub Undo 处理 | `app/lib/activitypub/activity/undo.rb` | 整个文件 |
| SuspendAccountService | `app/services/suspend_account_service.rb` | 整个文件 |
| Streaming API 服务 | `streaming/index.js` | 整个文件 |
