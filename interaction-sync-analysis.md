# 收藏、书签和转发互动处理方式分析

## 一、三种互动方式的异同

### 1. 数据模型对比

| 特性 | 收藏 (Favourite) | 书签 (Bookmark) | 转发 (Reblog) |
|------|------------------|-----------------|---------------|
| 模型类 | `Favourite` | `Bookmark` | `Status` (特殊类型) |
| 数据表 | `favourites` | `bookmarks` | `statuses` (reblog_of_id 字段) |
| 关联关系 | belongs_to :account, :status | belongs_to :account, :status | belongs_to :account, :reblog |
| 唯一性约束 | account_id + status_id | account_id + status_id | account_id + reblog_of_id |

### 2. 核心处理逻辑

#### 收藏 (Favourite)
- **服务类**: `FavouriteService` / `UnfavouriteService`
- **主要流程**:
  1. 权限检查 (`authorize_with account, status, :favourite?`)
  2. 检查是否已收藏，避免重复
  3. 创建 `Favourite` 记录
  4. 触发趋势统计 (`Trends.statuses.register(status)`)
  5. 创建通知：
     - 本地账户：`LocalNotificationWorker`
     - 远程账户：`ActivityPub::DeliveryWorker` (发送 Like 活动)
  6. 增加互动统计 (`ActivityTracker.increment('activity:interactions')`)

#### 书签 (Bookmark)
- **服务类**: 无独立服务类，通过控制器直接操作
- **主要流程**:
  1. 创建/销毁 `Bookmark` 记录
  2. 仅在销毁时触发清理策略失效检查 (`invalidate_cleanup_info`)
  3. **不发送通知**
  4. **不进行 ActivityPub 同步**

#### 转发 (Reblog)
- **服务类**: `ReblogService`
- **主要流程**:
  1. 权限检查 (`authorize_with account, reblogged_status, :reblog?`)
  2. 检查是否已转发，避免重复
  3. 创建新的 `Status` 记录，`reblog_of_id` 指向原状态
  4. 触发趋势统计 (`Trends.register!(reblog)`)
  5. 分发到时间线 (`DistributionWorker`)
  6. ActivityPub 分发 (`ActivityPub::DistributionWorker`)
  7. 创建通知：仅本地原作者 (`LocalNotificationWorker`)
  8. 增加互动统计 (`ActivityTracker.increment('activity:interactions')`)

### 3. 关键异同点总结

#### 相同点
1. **关联结构**: 都是账户与状态之间的关联关系
2. **唯一性约束**: 都保证同一账户对同一状态只能进行一次操作
3. **转发处理**: 创建时都会自动处理转发链，指向原始状态 (`before_validation` 中设置 `self.status = status.reblog if status&.reblog?`)

#### 不同点
1. **公开性**:
   - 收藏和转发是公开互动，会影响状态的公开计数
   - 书签是私人功能，不影响公开计数，也不同步到远程

2. **通知机制**:
   - 收藏：本地和远程作者都会收到通知
   - 转发：仅本地原作者收到通知
   - 书签：不发送任何通知

3. **ActivityPub 同步**:
   - 收藏：通过 `Like`/`Undo Like` 活动同步
   - 转发：通过 `Create` 活动（类型为 `Announce`）同步
   - 书签：不同步

4. **计数器更新**:
   - 收藏和转发会更新 `status_stats` 表中的对应计数
   - 书签不更新任何公开计数

---

## 二、互动计数的本地与远端同步机制

### 1. 数据存储结构

#### 状态统计表 (status_stats)
```ruby
# app/models/status_stat.rb
class StatusStat < ApplicationRecord
  belongs_to :status
  
  # 本地计数（基于本地数据库中的实际关联记录）
  field :favourites_count, :bigint, default: 0
  field :reblogs_count, :bigint, default: 0
  field :replies_count, :bigint, default: 0
  field :quotes_count, :bigint, default: 0
  
  # 远程实例提供的"不可信"计数
  field :untrusted_favourites_count, :bigint
  field :untrusted_reblogs_count, :bigint
end
```

### 2. 本地计数更新机制

#### 收藏计数更新
```ruby
# app/models/favourite.rb
class Favourite < ApplicationRecord
  after_create :increment_cache_counters
  after_destroy :decrement_cache_counters

  private

  def increment_cache_counters
    status&.increment_count!(:favourites_count)
  end

  def decrement_cache_counters
    return if association(:status).loaded? && status.marked_for_destruction?
    status&.decrement_count!(:favourites_count)
  end
end
```

#### 转发计数更新
```ruby
# app/models/status.rb
class Status < ApplicationRecord
  after_create_commit :increment_counter_caches
  after_destroy_commit :decrement_counter_caches

  private

  def increment_counter_caches
    return if direct_visibility?
    account&.increment_count!(:statuses_count, status_created_at: created_at)
    reblog&.increment_count!(:reblogs_count) if reblog?  # 转发时更新原状态计数
    thread&.increment_count!(:replies_count) if in_reply_to_id.present? && distributable?
  end

  def decrement_counter_caches
    return if direct_visibility? || new_record?
    account&.decrement_count!(:statuses_count)
    reblog&.decrement_count!(:reblogs_count) if reblog?
    thread&.decrement_count!(:replies_count) if in_reply_to_id.present? && distributable?
  end
end
```

### 3. 计数增减的核心实现

```ruby
# app/models/status.rb
def increment_count!(key)
  if key == :favourites_count && !untrusted_favourites_count.nil?
    # 如果存在远程计数，同时更新本地和远程计数
    update_status_stat!(
      favourites_count: favourites_count + 1,
      untrusted_favourites_count: untrusted_favourites_count + 1
    )
  elsif key == :reblogs_count && !untrusted_reblogs_count.nil?
    update_status_stat!(
      reblogs_count: reblogs_count + 1,
      untrusted_reblogs_count: untrusted_reblogs_count + 1
    )
  else
    # 仅更新本地计数
    update_status_stat!(key => public_send(key) + 1)
  end
end

def decrement_count!(key)
  if key == :favourites_count && !untrusted_favourites_count.nil?
    update_status_stat!(
      favourites_count: [favourites_count - 1, 0].max,
      untrusted_favourites_count: [untrusted_favourites_count - 1, 0].max
    )
  elsif key == :reblogs_count && !untrusted_reblogs_count.nil?
    update_status_stat!(
      reblogs_count: [reblogs_count - 1, 0].max,
      untrusted_reblogs_count: [untrusted_reblogs_count - 1, 0].max
    )
  else
    update_status_stat!(key => [public_send(key) - 1, 0].max)
  end
end
```

### 4. 远程计数获取与更新

#### 从 ActivityPub 活动中获取计数
```ruby
# app/lib/activitypub/activity/create.rb
def attach_counts(status)
  likes = @status_parser.favourites_count
  shares = @status_parser.reblogs_count
  return if likes.nil? && shares.nil?

  status.status_stat.tap do |status_stat|
    status_stat.untrusted_reblogs_count = shares unless shares.nil?
    status_stat.untrusted_favourites_count = likes unless likes.nil?
    status_stat.save if status_stat.changed?
  end
end
```

#### 状态更新时同步计数
```ruby
# app/services/activitypub/process_status_update_service.rb
def update_counts!
  likes = @status_parser.favourites_count
  shares = @status_parser.reblogs_count
  return if likes.nil? && shares.nil?

  @status.status_stat.tap do |status_stat|
    status_stat.untrusted_reblogs_count = shares unless shares.nil?
    status_stat.untrusted_favourites_count = likes unless likes.nil?
    status_stat.save if status_stat.changed?
  end
end
```

### 5. API 序列化时的计数优先级

```ruby
# app/serializers/rest/status_serializer.rb
def reblogs_count
  # 优先级：远程计数 > 关系缓存计数 > 本地计数
  object.untrusted_reblogs_count || 
    relationships&.attributes_map&.dig(object.id, :reblogs_count) || 
    object.reblogs_count
end

def favourites_count
  # 相同的优先级策略
  object.untrusted_favourites_count || 
    relationships&.attributes_map&.dig(object.id, :favourites_count) || 
    object.favourites_count
end
```

### 6. 同步机制总结

#### 本地 → 远程同步
1. **收藏**: 当本地用户收藏远程状态时，通过 `ActivityPub::DeliveryWorker` 发送 `Like` 活动到远程实例
2. **取消收藏**: 发送 `Undo Like` 活动
3. **转发**: 通过 `ActivityPub::DistributionWorker` 分发 `Announce` 活动

#### 远程 → 本地同步
1. **接收活动**: 当远程实例的活动到达时，通过 `ActivityPub::Activity::Create` 处理
2. **解析计数**: 从 ActivityPub JSON 中解析 `likes` (收藏数) 和 `shares` (转发数)
3. **存储计数**: 将远程计数存入 `untrusted_favourites_count` 和 `untrusted_reblogs_count` 字段
4. **状态更新**: 当远程状态更新时，通过 `ProcessStatusUpdateService` 更新计数

#### 计数优先级策略
1. **最高优先级**: 远程实例提供的 `untrusted_*` 计数
   - 原因：远程实例拥有完整的全局数据，本地只知道部分互动
2. **次优先级**: 关系缓存中的计数
3. **最低优先级**: 本地数据库的实际计数

#### 设计考量
- **"Untrusted" 命名**: 表示这些计数来自外部源，本地无法验证其准确性
- **双计数系统**: 本地维护真实计数（基于本地记录），同时存储远程计数用于展示
- **本地互动时的同步**: 当本地用户进行互动时，同时更新本地计数和远程计数（如果存在），保持一致性

---

## 三、撤销操作的处理链路

### 1. 取消收藏 (Unfavourite)

#### 处理流程

```ruby
# app/controllers/api/v1/statuses/favourites_controller.rb
def destroy
  fav = current_account.favourites.find_by(status_id: params[:status_id])

  if fav
    @status = fav.status
    # 1. 乐观更新：预计算计数用于前端展示
    count = [@status.favourites_count - 1, 0].max
    # 2. 异步执行实际取消操作
    UnfavouriteWorker.perform_async(current_account.id, @status.id)
  else
    @status = Status.find(params[:status_id])
    count = @status.favourites_count
    authorize @status, :show?
  end

  # 3. 使用关系缓存返回更新后的状态
  relationships = StatusRelationshipsPresenter.new([@status], current_account.id, 
    favourites_map: { @status.id => false }, 
    attributes_map: { @status.id => { favourites_count: count } })
  render json: @status, serializer: REST::StatusSerializer, relationships: relationships
end
```

#### 异步处理详情

```ruby
# app/workers/unfavourite_worker.rb
class UnfavouriteWorker
  def perform(account_id, status_id)
    UnfavouriteService.new.call(Account.find(account_id), Status.find(status_id))
  end
end

# app/services/unfavourite_service.rb
class UnfavouriteService < BaseService
  def call(account, status)
    # 1. 查找并销毁收藏记录
    favourite = Favourite.find_by!(account: account, status: status)
    favourite.destroy!  # 触发 after_destroy 回调
    
    # 2. 远端同步：仅当原作者是远程账户且支持 ActivityPub 时
    create_notification(favourite) if !status.account.local? && status.account.activitypub?
    
    favourite
  end

  private

  def create_notification(favourite)
    status = favourite.status
    # 发送 Undo Like 活动到远程实例
    ActivityPub::DeliveryWorker.perform_async(build_json(favourite), favourite.account_id, status.account.inbox_url)
  end

  def build_json(favourite)
    # 使用 UndoLikeSerializer 序列化
    serialize_payload(favourite, ActivityPub::UndoLikeSerializer).to_json
  end
end
```

#### 本地计数回退机制

```ruby
# app/models/favourite.rb
class Favourite < ApplicationRecord
  after_destroy :decrement_cache_counters

  private

  def decrement_cache_counters
    # 避免在状态被标记为销毁时重复递减
    return if association(:status).loaded? && status.marked_for_destruction?
    
    # 调用 Status#decrement_count!
    status&.decrement_count!(:favourites_count)
  end
end
```

#### 远端同步动作

| 条件 | 动作 |
|------|------|
| 原作者是本地账户 | 不同步（本地销毁记录即可） |
| 原作者是远程账户 + 支持 ActivityPub | 发送 `Undo Like` 活动 |

`Undo Like` 活动结构：
```json
{
  "id": "https://本地实例/users/用户#likes/收藏ID/undo",
  "type": "Undo",
  "actor": "https://本地实例/users/用户",
  "object": {
    "id": "https://本地实例/users/用户#likes/收藏ID",
    "type": "Like",
    "actor": "https://本地实例/users/用户",
    "object": "https://远程实例/users/原作者/statuses/状态ID"
  }
}
```

---

### 2. 取消转发 (Unreblog)

#### 处理流程

```ruby
# app/controllers/api/v1/statuses/reblogs_controller.rb
def destroy
  # 1. 查找用户的转发记录
  @status = current_account.statuses.find_by(reblog_of_id: params[:status_id])

  if @status
    authorize @status, :unreblog?
    @reblog = @status.reblog
    # 2. 乐观更新：预计算计数
    count = [@reblog.reblogs_count - 1, 0].max
    # 3. 软删除转发状态
    @status.discard
    # 4. 异步执行完整清理
    RemovalWorker.perform_async(@status.id)
  else
    @reblog = Status.find(params[:status_id])
    count = @reblog.reblogs_count
    authorize @reblog, :show?
  end

  # 5. 使用关系缓存返回更新后的状态
  relationships = StatusRelationshipsPresenter.new([@status], current_account.id,
    reblogs_map: { @reblog.id => false },
    attributes_map: { @reblog.id => { reblogs_count: count } })
  render json: @reblog, serializer: REST::StatusSerializer, relationships: relationships
end
```

#### 异步处理详情

```ruby
# app/workers/removal_worker.rb
class RemovalWorker
  def perform(status_id, options = {})
    RemoveStatusService.new.call(Status.with_discarded.find(status_id), **options.symbolize_keys)
  end
end

# app/services/remove_status_service.rb
class RemoveStatusService < BaseService
  def call(status, **options)
    @payload  = { event: :delete, payload: status.id.to_s }.to_json
    @status   = status
    @account  = status.account
    @options  = options

    with_redis_lock("distribute:#{@status.id}") do
      # 1. 软删除（带关联转发）
      @status.discard_with_reblogs

      # 2. 清理相关记录
      StatusPin.find_by(status: @status)&.destroy

      # 3. 从本地时间线移除
      remove_from_self if @account.local?
      remove_from_followers
      remove_from_lists

      # 4. 远端同步：发送 Undo Announce 活动
      # 注意：如果是原状态被删除导致的转发删除，不同步（因为原状态的 Delete 活动已包含）
      remove_from_remote_reach if @account.local? && !@options[:original_removed]

      # ... 其他清理逻辑
      
      @status.destroy! if permanently?
    end
  end

  private

  def remove_from_remote_reach
    # 找到所有需要通知的收件箱（关注者、中继、被提及者等）
    status_reach_finder = StatusReachFinder.new(@status, unsafe: true)

    # 批量发送活动
    ActivityPub::DeliveryWorker.push_bulk(status_reach_finder.inboxes, limit: 1_000) do |inbox_url|
      [signed_activity_json, @account.id, inbox_url]
    end
  end

  def signed_activity_json
    # 根据状态类型选择序列化器
    @signed_activity_json ||= serialize_payload(
      @status, 
      @status.reblog? ? ActivityPub::UndoAnnounceSerializer : ActivityPub::DeleteNoteSerializer,
      signer: @account, 
      always_sign: true
    ).to_json
  end
end
```

#### 本地计数回退机制

```ruby
# app/models/status.rb
class Status < ApplicationRecord
  after_destroy_commit :decrement_counter_caches

  private

  def decrement_counter_caches
    return if direct_visibility? || new_record?
    
    # 1. 递减用户的状态计数
    account&.decrement_count!(:statuses_count)
    
    # 2. 如果是转发，递减原状态的转发计数
    reblog&.decrement_count!(:reblogs_count) if reblog?
    
    # 3. 如果是回复，递减原状态的回复计数
    thread&.decrement_count!(:replies_count) if in_reply_to_id.present? && distributable?
  end
end
```

#### 远端同步动作

| 条件 | 动作 |
|------|------|
| 转发者是本地账户 + 不是因为原状态删除导致 | 发送 `Undo Announce` 活动 |
| 原状态被删除导致的转发删除 | 不同步（原状态的 `Delete` 活动已涵盖） |

`Undo Announce` 活动结构：
```json
{
  "id": "https://本地实例/users/用户#announces/转发ID/undo",
  "type": "Undo",
  "actor": "https://本地实例/users/用户",
  "to": ["https://www.w3.org/ns/activitystreams#Public"],
  "object": {
    "id": "https://本地实例/users/用户/statuses/转发ID",
    "type": "Announce",
    "actor": "https://本地实例/users/用户",
    "published": "2026-05-04T10:00:00Z",
    "to": ["https://www.w3.org/ns/activitystreams#Public"],
    "object": "https://远程实例/users/原作者/statuses/状态ID"
  }
}
```

---

### 3. 取消书签 (Unbookmark)

#### 处理流程

```ruby
# app/controllers/api/v1/statuses/bookmarks_controller.rb
def destroy
  # 1. 查找书签记录
  bookmark = current_account.bookmarks.find_by(status_id: params[:status_id])

  if bookmark
    @status = bookmark.status
  else
    @status = Status.find(params[:status_id])
    authorize @status, :show?
  end

  # 2. 直接销毁书签记录
  bookmark&.destroy!

  # 3. 返回更新后的状态
  render json: @status, serializer: REST::StatusSerializer, 
    relationships: StatusRelationshipsPresenter.new([@status], current_account.id, bookmarks_map: { @status.id => false })
end
```

#### 本地计数回退机制

**无公开计数回退**：书签是纯私人功能，不影响 `status_stats` 中的任何公开计数。

唯一的回调是清理策略失效检查：
```ruby
# app/models/bookmark.rb
class Bookmark < ApplicationRecord
  after_destroy :invalidate_cleanup_info

  def invalidate_cleanup_info
    # 仅当是自己的状态且是本地账户时，通知清理策略
    return unless status&.account_id == account_id && account.local?
    
    account.statuses_cleanup_policy&.invalidate_last_inspected(status, :unbookmark)
  end
end
```

#### 远端同步动作

**无远端同步**：书签是纯本地功能，不进行任何 ActivityPub 同步。

---

### 4. 撤销操作对比总结

| 维度 | 取消收藏 | 取消转发 | 取消书签 |
|------|----------|----------|----------|
| **入口控制器** | `FavouritesController#destroy` | `ReblogsController#destroy` | `BookmarksController#destroy` |
| **异步处理** | `UnfavouriteWorker` → `UnfavouriteService` | `RemovalWorker` → `RemoveStatusService` | 无（同步执行） |
| **本地计数回退** | `favourites_count` -1 | `reblogs_count` -1 + `statuses_count` -1 | 无 |
| **回调触发** | `Favourite#after_destroy` | `Status#after_destroy_commit` | `Bookmark#after_destroy` |
| **乐观更新** | 是（预计算计数用于前端） | 是（预计算计数用于前端） | 否 |
| **远端同步** | 条件性发送 `Undo Like` | 条件性发送 `Undo Announce` | 否 |
| **同步条件** | 原作者是远程账户 | 转发者是本地账户且非原状态删除 | - |
| **序列化器** | `UndoLikeSerializer` | `UndoAnnounceSerializer` | - |

---

## 四、完整时序说明：本地 → 远端 → 回本地

### 场景：本地用户收藏远程状态，然后取消收藏

```
┌──────────────┐                    ┌──────────────┐                    ┌──────────────┐
│  本地实例 A  │                    │  远程实例 B  │                    │  其他实例 C  │
│ (用户: alice)│                    │(用户: bob)   │                    │              │
└──────┬───────┘                    └──────┬───────┘                    └──────┬───────┘
       │                                     │                                     │
       │  1. alice 点击"收藏" bob 的状态     │                                     │
       │                                     │                                     │
       ▼                                     │                                     │
┌─────────────────────────────────────────┐ │                                     │
│  收藏操作 (FavouriteService)             │ │                                     │
│  ├── 创建 Favourite 记录                 │ │                                     │
│  ├── 递增 favourites_count (+1)          │ │                                     │
│  └── 发送 Like 活动到 B 的 inbox         │ │                                     │
└───────────────────┬─────────────────────┘ │                                     │
                    │                        │                                     │
                    │ POST /users/bob/inbox │                                     │
                    │ (ActivityPub Like)    │                                     │
                    │───────────────────────>│                                     │
                    │                        │                                     │
                    │                        ▼                                     │
                    │              ┌───────────────────────┐                      │
                    │              │  远程实例 B 处理      │                      │
                    │              │  ├── 验证签名          │                      │
                    │              │  ├── 创建 Favourite   │                      │
                    │              │  ├── 递增计数          │                      │
                    │              │  └── 通知 bob         │                      │
                    │              └───────────┬───────────┘                      │
                    │                          │                                  │
                    │                          │  2. alice 点击"取消收藏"          │
                    │                          │                                  │
                    │                          ▼                                  │
                    │              ┌─────────────────────────────────────────┐   │
                    │              │  取消收藏操作 (UnfavouriteService)      │   │
                    │              │  ├── 销毁 Favourite 记录                │   │
                    │              │  ├── 递减 favourites_count (-1)         │   │
                    │              │  └── 发送 Undo Like 活动到 B 的 inbox   │   │
                    │              └───────────────────┬─────────────────────┘   │
                    │                                  │                          │
                    │  POST /users/bob/inbox         │                          │
                    │  (ActivityPub Undo Like)       │                          │
                    │<───────────────────────────────│                          │
                    │                                  │                          │
                    ▼                                  │                          │
┌─────────────────────────────────────────┐          │                          │
│  本地实例 A 处理 Undo Like               │          │                          │
│  (如果有其他实例转发了此活动)             │          │                          │
│  ├── 验证签名                            │          │                          │
│  ├── 查找 Favourite 记录                 │          │                          │
│  └── 销毁记录并递减计数                   │          │                          │
└─────────────────────────────────────────┘          │                          │
                                                       │                          │
                                                       │  (可选) B 将活动转发给   │
                                                       │  关注者/其他实例          │
                                                       │──────────────────────────>│
                                                       │                          │
                                                       │                          ▼
                                                       │              ┌───────────────────────┐
                                                       │              │  实例 C 处理          │
                                                       │              │  ├── 验证签名          │
                                                       │              │  ├── 查找 Favourite   │
                                                       │              │  └── 销毁并递减计数   │
                                                       │              └───────────────────────┘
                                                       │
```

### 详细时序步骤

#### 第一阶段：收藏操作（本地 → 远端）

| 步骤 | 执行方 | 操作 | 关键代码 |
|------|--------|------|----------|
| 1 | 用户 alice (A) | 点击收藏按钮 | - |
| 2 | 本地实例 A | `FavouriteService#call` | `app/services/favourite_service.rb` |
| 3 | 本地实例 A | 创建 `Favourite` 记录 | `Favourite.create!` |
| 4 | 本地实例 A | 触发 `after_create` 回调，递增计数 | `Favourite#increment_cache_counters` |
| 5 | 本地实例 A | 构建 `Like` 活动并发送 | `ActivityPub::DeliveryWorker` |
| 6 | 远程实例 B | 接收 `Like` 活动 | `ActivityPub::Activity::Like` |
| 7 | 远程实例 B | 创建本地 `Favourite` 记录 | - |
| 8 | 远程实例 B | 递增 `favourites_count` | - |
| 9 | 远程实例 B | 通知 bob 有新收藏 | `LocalNotificationWorker` |

#### 第二阶段：取消收藏操作（本地 → 远端）

| 步骤 | 执行方 | 操作 | 关键代码 |
|------|--------|------|----------|
| 1 | 用户 alice (A) | 点击取消收藏按钮 | - |
| 2 | 本地实例 A | `FavouritesController#destroy` | `app/controllers/api/v1/statuses/favourites_controller.rb` |
| 3 | 本地实例 A | 乐观更新：预计算计数 | `count = [@status.favourites_count - 1, 0].max` |
| 4 | 本地实例 A | 异步执行 `UnfavouriteWorker` | `UnfavouriteWorker.perform_async` |
| 5 | 本地实例 A | 立即返回带缓存的响应 | `StatusRelationshipsPresenter` |
| 6 | 本地实例 A (异步) | `UnfavouriteService#call` | `app/services/unfavourite_service.rb` |
| 7 | 本地实例 A | 销毁 `Favourite` 记录 | `favourite.destroy!` |
| 8 | 本地实例 A | 触发 `after_destroy` 回调，递减计数 | `Favourite#decrement_cache_counters` |
| 9 | 本地实例 A | 构建 `Undo Like` 活动 | `ActivityPub::UndoLikeSerializer` |
| 10 | 本地实例 A | 发送到远程实例 B | `ActivityPub::DeliveryWorker` |
| 11 | 远程实例 B | 接收 `Undo Like` 活动 | `ActivityPub::Activity::Undo#undo_like` |
| 12 | 远程实例 B | 验证目标状态是本地的 | `!status.account.local?` 检查 |
| 13 | 远程实例 B | 查找并销毁 `Favourite` 记录 | `status.favourites.where(account: @account).first&.destroy` |
| 14 | 远程实例 B | 触发 `after_destroy` 回调，递减计数 | 自动 |

#### 第三阶段：远端 → 其他实例（可选）

如果远程实例 B 将 `Undo Like` 活动转发给其关注者：

| 步骤 | 执行方 | 操作 |
|------|--------|------|
| 1 | 远程实例 B | 将 `Undo Like` 活动转发给关注者 |
| 2 | 其他实例 C | 接收活动并验证签名 |
| 3 | 其他实例 C | 查找本地 `Favourite` 记录（如果存在） |
| 4 | 其他实例 C | 销毁记录并递减计数 |

### 转发操作的完整时序（简化版）

```
alice (A) 转发 bob (B) 的状态，然后取消转发：

1. 转发操作：
   A 创建 Status (reblog_of_id = bob_status_id)
   A 递增 reblogs_count
   A 发送 Announce 活动到 B 和关注者

2. 取消转发操作：
   A 调用 ReblogsController#destroy
   A 软删除转发状态 (discard)
   A 乐观更新计数
   A 异步执行 RemovalWorker
   A 调用 RemoveStatusService
   A 从时间线移除
   A 发送 Undo Announce 活动
   A 触发 after_destroy_commit 递减计数
   B 接收 Undo Announce，递减 reblogs_count
```

### 关键设计要点

1. **乐观更新 (Optimistic Update)**
   - 控制器先预计算新计数，通过 `StatusRelationshipsPresenter` 立即返回给前端
   - 实际的数据库操作和远端同步异步执行
   - 目的：提升用户体验，减少等待时间

2. **异步处理**
   - 取消收藏：`UnfavouriteWorker`
   - 取消转发：`RemovalWorker` → `RemoveStatusService`
   - 原因：
     - 远端同步可能涉及网络请求，耗时不确定
     - 避免阻塞 HTTP 请求
     - 失败时可重试（Sidekiq 内置重试机制）

3. **软删除 (Soft Delete)**
   - 转发使用 `discard` 而非直接 `destroy`
   - `deleted_at` 字段标记删除状态
   - `RemoveStatusService` 再执行完整清理
   - 原因：给异步处理留时间，避免状态不一致

4. **双计数同步**
   - 本地操作时同时更新 `favourites_count` 和 `untrusted_favourites_count`（如果存在）
   - 确保本地互动后，前端展示的计数立即反映变化
   - 代码：`Status#increment_count!` / `decrement_count!`

5. **条件性同步**
   - 取消收藏：仅当原作者是远程账户时才发送 `Undo Like`
   - 取消转发：仅当不是因原状态删除导致时才发送 `Undo Announce`
   - 原因：避免不必要的网络请求和重复操作

---

## 五、更新后的关键代码位置

| 功能 | 文件路径 |
|------|----------|
| 收藏模型 | `app/models/favourite.rb` |
| 书签模型 | `app/models/bookmark.rb` |
| 状态模型 | `app/models/status.rb` |
| 状态统计模型 | `app/models/status_stat.rb` |
| 收藏服务 | `app/services/favourite_service.rb` |
| 取消收藏服务 | `app/services/unfavourite_service.rb` |
| 转发服务 | `app/services/reblog_service.rb` |
| 移除状态服务 | `app/services/remove_status_service.rb` |
| 收藏控制器 | `app/controllers/api/v1/statuses/favourites_controller.rb` |
| 转发控制器 | `app/controllers/api/v1/statuses/reblogs_controller.rb` |
| 书签控制器 | `app/controllers/api/v1/statuses/bookmarks_controller.rb` |
| 取消收藏 Worker | `app/workers/unfavourite_worker.rb` |
| 移除状态 Worker | `app/workers/removal_worker.rb` |
| ActivityPub Undo 处理 | `app/lib/activitypub/activity/undo.rb` |
| Undo Like 序列化器 | `app/serializers/activitypub/undo_like_serializer.rb` |
| Undo Announce 序列化器 | `app/serializers/activitypub/undo_announce_serializer.rb` |
| 状态序列化器 | `app/serializers/rest/status_serializer.rb` |
| ActivityPub 创建处理 | `app/lib/activitypub/activity/create.rb` |
| 状态更新处理 | `app/services/activitypub/process_status_update_service.rb` |
