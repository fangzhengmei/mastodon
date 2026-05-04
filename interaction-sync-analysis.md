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

## 三、关键代码位置

| 功能 | 文件路径 |
|------|----------|
| 收藏模型 | `app/models/favourite.rb` |
| 书签模型 | `app/models/bookmark.rb` |
| 状态模型 | `app/models/status.rb` |
| 状态统计模型 | `app/models/status_stat.rb` |
| 收藏服务 | `app/services/favourite_service.rb` |
| 取消收藏服务 | `app/services/unfavourite_service.rb` |
| 转发服务 | `app/services/reblog_service.rb` |
| 状态序列化器 | `app/serializers/rest/status_serializer.rb` |
| ActivityPub 创建处理 | `app/lib/activitypub/activity/create.rb` |
| 状态更新处理 | `app/services/activitypub/process_status_update_service.rb` |
