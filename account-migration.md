# Mastodon 账号迁移机制完整分析

## 目录

1. [概述](#概述)
2. [前端引导流程](#前端引导流程)
3. [旧实例发出 Move 信号机制](#旧实例发出-move-信号机制)
4. [新实例验证并接收 Follower 转移](#新实例验证并接收-follower-转移)
5. [后台任务批量重写关注关系](#后台任务批量重写关注关系)
6. [跨实例协议校验配合](#跨实例协议校验配合)
7. [关键数据结构](#关键数据结构)
8. [流程图总结](#流程图总结)

---

## 概述

Mastodon 的账号迁移是基于 ActivityPub 协议的 `Move` 活动实现的。整个流程涉及三个核心角色：

- **源账号（旧实例）**：发起迁移，发出 Move 信号
- **目标账号（新实例）**：验证迁移合法性，接收 follower
- **关注者实例**：接收 Move 活动，重写关注关系

迁移的核心安全机制是 **双向引用验证**：
- 目标账号的 `also_known_as` 必须包含源账号的 URI
- 源账号的 `moved_to_account` 指向目标账号

---

## 前端引导流程

### 1. 前置条件：新账号添加别名

在发起迁移之前，用户需要在**新账号**的设置中添加旧账号作为别名：

**页面路径**：`/settings/aliases`

**控制器**：`app/controllers/settings/aliases_controller.rb`

**流程**：
1. 用户输入旧账号地址（如 `olduser@oldinstance.com`）
2. `AccountAlias` 模型验证：
   - 通过 WebFinger 解析目标账号
   - 验证不是自己指向自己
   - `app/models/account_alias.rb:50-56`
3. 保存后触发回调：
   ```ruby
   # app/models/account_alias.rb:42-44
   after_create :add_to_account
   
   def add_to_account
     account.update(also_known_as: account.also_known_as + [uri])
   end
   ```
4. 分发 Update 活动让全网知晓：
   ```ruby
   # app/controllers/settings/aliases_controller.rb:17-18
   ActivityPub::UpdateDistributionWorker.perform_async(current_account.id)
   ```

**关键**：这一步将旧账号的 URI 添加到新账号的 `also_known_as` 数组中，为后续的迁移验证做准备。

### 2. 发起迁移

在**旧账号**的设置中发起迁移：

**页面路径**：`/settings/migration`

**控制器**：`app/controllers/settings/migrations_controller.rb`

**前端视图**：`app/views/settings/migrations/show.html.haml`

**用户输入**：
- 目标账号地址（新账号）
- 当前密码验证（或用户名验证，取决于登录方式）

**表单字段**（`app/views/settings/migrations/show.html.haml:45-53`）：
```haml
.fields-row
  .fields-row__column.fields-group.fields-row__column-6
    = f.input :acct, wrapper: :with_block_label, ...
  .fields-row__column.fields-group.fields-row__column-6
    - if current_user.encrypted_password.present?
      = f.input :current_password, ...
    - else
      = f.input :current_username, ...
```

### 3. 仅设置重定向（不转移 Follower）

还有一个轻量级选项：仅设置重定向，不触发 follower 转移。

**页面路径**：`/settings/migration/redirects/new`

**控制器**：`app/controllers/settings/migration/redirects_controller.rb`

**适用场景**：账号被盗、紧急转移等情况

**区别**：
- 只设置 `moved_to_account`
- 只分发 Update 活动
- **不**触发 Move 活动和 follower 转移

---

## 旧实例发出 Move 信号机制

### 1. 迁移验证（AccountMigration 模型）

**文件**：`app/models/account_migration.rb`

当用户提交迁移表单时，`AccountMigration` 模型执行以下验证：

#### 1.1 目标账号解析
```ruby
# app/models/account_migration.rb:69-73
def set_target_account
  self.target_account = ResolveAccountService.new.call(acct, skip_cache: true)
rescue Webfinger::Error, *Mastodon::HTTP_CONNECTION_ERRORS, ...
  # Validation will take care of it
end
```

#### 1.2 核心验证逻辑
```ruby
# app/models/account_migration.rb:79-91
def validate_target_account
  if target_account.nil?
    errors.add(:acct, I18n.t('migrations.errors.not_found'))
  else
    # 关键验证：目标账号的 also_known_as 必须包含源账号 URI
    errors.add(:acct, I18n.t('migrations.errors.missing_also_known_as')) 
      unless target_account.also_known_as.include?(ActivityPub::TagManager.instance.uri_for(account))
    
    # 不能重复迁移到同一个账号
    errors.add(:acct, I18n.t('migrations.errors.already_moved')) 
      if account.moved? && account.moved_to_account_id == target_account.id
    
    # 不能迁移到自己
    errors.add(:acct, I18n.t('migrations.errors.move_to_self')) 
      if account.id == target_account.id
  end
end

def validate_migration_cooldown
  # 30天冷却期
  errors.add(:base, I18n.t('migrations.errors.on_cooldown')) 
    if account.migrations.within_cooldown.exists?
end
```

#### 1.3 密码/用户名验证
```ruby
# app/models/account_migration.rb:45-57
def save_with_challenge(current_user)
  if current_user.encrypted_password.present?
    errors.add(:current_password, :invalid) unless current_user.valid_password?(current_password)
  else
    errors.add(:current_username, :invalid) unless account.username == current_username
  end
  # ...
  with_redis_lock("account_migration:#{account.id}") do
    save
  end
end
```

### 2. MoveService 执行迁移

**文件**：`app/services/move_service.rb`

验证通过后，`MigrationsController` 调用 `MoveService`：

```ruby
# app/controllers/settings/migrations_controller.rb:14-22
def create
  @migration = current_account.migrations.build(resource_params)
  if @migration.save_with_challenge(current_user)
    MoveService.new.call(@migration)
    # ...
  end
end
```

`MoveService` 执行四个关键步骤：

```ruby
# app/services/move_service.rb:3-32
class MoveService < BaseService
  def call(migration)
    @migration      = migration
    @source_account = migration.account
    @target_account = migration.target_account

    update_redirect!           # 步骤1：设置重定向
    process_local_relationships!  # 步骤2：处理本地关注关系
    distribute_update!         # 步骤3：分发 Update 活动
    distribute_move!           # 步骤4：分发 Move 活动
  end

  private

  def update_redirect!
    @source_account.update!(moved_to_account: @target_account)
  end

  def process_local_relationships!
    MoveWorker.perform_async(@source_account.id, @target_account.id)
  end

  def distribute_update!
    ActivityPub::UpdateDistributionWorker.perform_async(@source_account.id)
  end

  def distribute_move!
    ActivityPub::MoveDistributionWorker.perform_async(@migration.id)
  end
end
```

### 3. MoveDistributionWorker 分发 Move 活动

**文件**：`app/workers/activitypub/move_distribution_worker.rb`

#### 3.1 序列化 Move 活动

使用 `ActivityPub::MoveSerializer` 序列化：

**文件**：`app/serializers/activitypub/move_serializer.rb`

```ruby
class ActivityPub::MoveSerializer < ActivityPub::Serializer
  attributes :id, :type, :target, :actor
  attribute :virtual_object, key: :object

  def id
    [ActivityPub::TagManager.instance.uri_for(object.account), '#moves/', object.id].join
  end

  def type
    'Move'
  end

  def target
    ActivityPub::TagManager.instance.uri_for(object.target_account)
  end

  def virtual_object
    ActivityPub::TagManager.instance.uri_for(object.account)
  end

  def actor
    ActivityPub::TagManager.instance.uri_for(object.account)
  end
end
```

生成的 ActivityPub 活动格式：
```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "id": "https://oldinstance.com/users/olduser#moves/123",
  "type": "Move",
  "actor": "https://oldinstance.com/users/olduser",
  "object": "https://oldinstance.com/users/olduser",
  "target": "https://newinstance.com/users/newuser"
}
```

#### 3.2 分发到目标 Inbox

```ruby
# app/workers/activitypub/move_distribution_worker.rb:9-22
def perform(migration_id)
  @migration = AccountMigration.find(migration_id)
  @account   = @migration.account

  # 分发给所有关注者的 inbox
  ActivityPub::DeliveryWorker.push_bulk(inboxes, limit: 1_000) do |inbox_url|
    [signed_payload, @account.id, inbox_url]
  end

  # 分发给中继服务器
  ActivityPub::DeliveryWorker.push_bulk(Relay.enabled.pluck(:inbox_url)) do |inbox_url|
    [signed_payload, @account.id, inbox_url]
  end
end

private

def inboxes
  @inboxes ||= (@migration.account.followers.inboxes + @migration.account.blocked_by.inboxes).uniq
end
```

**注意**：
- 分发给**所有关注者**的 inbox
- 分发给**被屏蔽者**的 inbox（让他们也知道迁移）
- 分发给**中继服务器**（Relay）

---

## 新实例验证并接收 Follower 转移

当关注者所在的实例接收到 Move 活动时，由 `ActivityPub::Activity::Move` 处理。

### 1. ActivityPub::Activity::Move 处理器

**文件**：`app/lib/activitypub/activity/move.rb`

```ruby
class ActivityPub::Activity::Move < ActivityPub::Activity
  PROCESSING_COOLDOWN = 7.days.seconds  # 7天冷却期

  def perform
    # 验证1：object 必须是 actor 自己（防止假冒迁移）
    return if origin_account.uri != object_uri
    
    # 验证2：7天内不能重复处理
    return unless mark_as_processing!

    # 解析目标账号
    target_account = ActivityPub::FetchRemoteAccountService.new.call(target_uri)

    # 验证3：核心安全验证
    # - 目标账号存在且可用
    # - 目标账号的 also_known_as 包含源账号 URI（双向验证）
    if target_account.nil? || target_account.unavailable? || 
       !target_account.also_known_as.include?(origin_account.uri)
      unmark_as_processing!
      return
    end

    # 设置源账号的 moved_to_account（本地缓存）
    origin_account.update(moved_to_account: target_account)

    # 触发关注关系转移
    MoveWorker.perform_async(origin_account.id, target_account.id)
  rescue
    unmark_as_processing!
    raise
  end

  private

  def mark_as_processing!
    redis.set("move_in_progress:#{@account.id}", true, nx: true, ex: PROCESSING_COOLDOWN)
  end
end
```

### 2. 验证流程详解

| 验证项 | 代码位置 | 目的 |
|--------|----------|------|
| `origin_account.uri == object_uri` | 第7行 | 确保 Move 活动的 object 是 actor 自己，防止恶意迁移他人账号 |
| 7天冷却期 | 第8行、37-42行 | 防止频繁迁移或重复处理 |
| `target_account.also_known_as.include?(origin_account.uri)` | 第12行 | **核心安全验证**：目标账号必须"认领"源账号 |

**关键验证逻辑**：
```ruby
# 目标账号的 also_known_as 必须包含源账号 URI
!target_account.also_known_as.include?(origin_account.uri)
```

这确保了：
1. 新账号的持有者必须主动添加旧账号到别名（`also_known_as`）
2. 证明新账号持有者拥有旧账号的控制权（或至少知道迁移意图）
3. 防止恶意账号"偷走"他人的 follower

---

## 后台任务批量重写关注关系

### 1. MoveWorker 主入口

**文件**：`app/workers/move_worker.rb`

`MoveWorker` 处理两种场景：

```ruby
def perform(source_account_id, target_account_id)
  @source_account = Account.find(source_account_id)
  @target_account = Account.find(target_account_id)

  if @target_account.local? && @source_account.local?
    # 场景1：本地账号之间迁移 → 直接批量更新数据库
    num_moved = rewrite_follows!
    @source_account.update_count!(:followers_count, -num_moved)
    @target_account.update_count!(:followers_count, num_moved)
  else
    # 场景2：跨实例迁移 → 异步队列处理
    queue_follow_unfollows!
  end

  # 其他数据迁移
  copy_account_notes!    # 复制账号备注
  carry_blocks_over!     # 迁移屏蔽关系
  carry_mutes_over!      # 迁移静音关系
end
```

### 2. 场景1：本地账号迁移（rewrite_follows!）

**文件**：`app/workers/move_worker.rb:31-78`

本地迁移采用**批量数据库更新**，效率最高。

#### 2.1 三阶段处理逻辑

```ruby
def rewrite_follows!
  num_moved = 0

  # 阶段1：处理待处理的关注请求
  # 先批准新账号的待处理请求，确保列表成员关系正确处理
  FollowRequest.where(account: @source_account.followers, target_account_id: @target_account.id).find_each do |follow_request|
    # 处理列表成员关系
    ListAccount.where(follow_id: follow_request.id).includes(:list).find_each do |list_account|
      list_account.list.accounts << @target_account
    rescue ActiveRecord::RecordInvalid
      nil
    end
    follow_request.authorize!
  end

  # 阶段2：处理同时关注新旧账号的情况
  source_local_followers
    .where(account: @target_account.followers.local)
    .in_batches do |follows|
      # 只需要处理列表成员关系（已经关注了新账号）
      ListAccount.where(follow: follows).includes(:list).find_each do |list_account|
        list_account.list.accounts << @target_account
      rescue ActiveRecord::RecordInvalid
        nil
      end
    end

  # 阶段3：处理只关注旧账号的情况（最常见）
  source_local_followers
    .where.not(account: @target_account.followers.local)
    .where.not(account_id: @target_account.id)
    .in_batches do |follows|
      # 批量更新列表成员关系
      ListAccount.where(follow: follows).in_batches.update_all(account_id: @target_account.id)
      
      # 批量更新关注关系：将 target_account_id 从旧账号改为新账号
      num_moved += follows.update_all(target_account_id: @target_account.id)

      # 清除关系缓存（update_all 不触发回调）
      Rails.cache.delete_multi(follows.flat_map do |follow|
        [
          ['relationships', follow.account_id, follow.target_account_id],
          ['relationships', follow.target_account_id, follow.account_id],
          ['relationships', follow.account_id, @target_account.id],
          ['relationships', @target_account.id, follow.account_id],
        ]
      end)
    end

  num_moved
end
```

#### 2.2 本地迁移优化点

1. **批量操作**：使用 `in_batches` 和 `update_all`，避免逐行操作
2. **分阶段处理**：
   - 待处理请求 → 同时关注 → 仅关注旧账号
3. **缓存清理**：`update_all` 不触发 ActiveRecord 回调，需手动清理缓存
4. **列表迁移**：`ListAccount` 记录的 `account_id` 同步更新

### 3. 场景2：跨实例迁移（queue_follow_unfollows!）

**文件**：`app/workers/move_worker.rb:86-94`

跨实例迁移需要通过 ActivityPub 协议与远程实例交互，因此采用异步队列处理。

```ruby
def queue_follow_unfollows!
  bypass_locked = @target_account.local?

  # 批量推送 UnfollowFollowWorker 任务
  @source_account.followers.local.select(:id).reorder(nil).find_in_batches do |accounts|
    UnfollowFollowWorker.push_bulk(accounts.map(&:id)) { |follower_id| 
      [follower_id, @source_account.id, @target_account.id, bypass_locked] 
    }
  rescue => e
    @deferred_error = e
  end
end
```

**参数说明**：
- `follower_id`：关注者账号 ID
- `@source_account.id`：旧账号 ID
- `@target_account.id`：新账号 ID
- `bypass_locked`：是否绕过新账号的锁（仅当新账号是本地账号时为 true）

### 4. UnfollowFollowWorker 单条处理

**文件**：`app/workers/unfollow_follow_worker.rb`

```ruby
class UnfollowFollowWorker
  include Sidekiq::Worker

  def perform(follower_account_id, old_target_account_id, new_target_account_id, bypass_locked = false)
    follower_account   = Account.find(follower_account_id)
    old_target_account = Account.find(old_target_account_id)
    new_target_account = Account.find(new_target_account_id)

    FollowMigrationService.new.call(
      follower_account, 
      new_target_account, 
      old_target_account, 
      bypass_locked: bypass_locked
    )
  rescue ActiveRecord::RecordNotFound, Mastodon::NotPermittedError
    true
  end
end
```

### 5. FollowMigrationService 核心逻辑

**文件**：`app/services/follow_migration_service.rb`

继承自 `FollowService`，保留原有关注设置。

```ruby
class FollowMigrationService < FollowService
  # @param [Account] source_account 关注者账号
  # @param [Account] target_account 新目标账号
  # @param [Account] old_target_account 旧目标账号
  # @option [Boolean] bypass_locked 是否绕过锁定账号限制
  def call(source_account, target_account, old_target_account, bypass_locked: false)
    @old_target_account = old_target_account

    # 读取原有关注设置
    @original_follow = source_account.active_relationships.find_by(target_account: old_target_account)
    reblogs          = @original_follow&.show_reblogs?
    notify           = @original_follow&.notify?
    languages        = @original_follow&.languages

    # 调用父类 FollowService，保留原有设置
    super(source_account, target_account, 
          reblogs: reblogs, notify: notify, languages: languages, 
          bypass_locked: bypass_locked, bypass_limit: true)
  end
```

#### 5.1 三种关注场景处理

根据新账号的状态，有三种处理方式：

```ruby
private

# 场景A：新账号需要审核（locked）→ 创建关注请求
def request_follow!
  follow_request = @source_account.request_follow!(@target_account, **follow_options)
  migrate_list_accounts!  # 迁移列表成员

  if @target_account.local?
    # 本地账号：发送通知，立即取消关注旧账号
    LocalNotificationWorker.perform_async(@target_account.id, follow_request.id, ...)
    UnfollowService.new.call(@source_account, @old_target_account, skip_unmerge: true)
  elsif @target_account.activitypub?
    # 远程账号：发送 ActivityPub Follow 活动，携带旧账号信息
    ActivityPub::MigratedFollowDeliveryWorker.perform_async(
      build_json(follow_request), 
      @source_account.id, 
      @target_account.inbox_url, 
      @old_target_account.id
    )
  end

  follow_request
end

# 场景B：已有关注请求，更新选项
def change_follow_request_options!
  migrate_list_accounts!
  super
end

# 场景C：直接关注（新账号未锁定或 bypass_locked）
def direct_follow!
  follow = super
  migrate_list_accounts!
  # 立即取消关注旧账号
  UnfollowService.new.call(@source_account, @old_target_account, skip_unmerge: true)
  follow
end

# 迁移列表成员关系
def migrate_list_accounts!
  ListAccount.where(follow_id: @original_follow.id).includes(:list).find_each do |list_account|
    list_account.list.accounts << @target_account
  rescue ActiveRecord::RecordInvalid
    nil
  end
end
```

### 6. 其他数据迁移

`MoveWorker` 还处理以下数据迁移：

#### 6.1 复制账号备注
```ruby
def copy_account_notes!
  @source_account.targeted_account_notes.find_each do |note|
    text = I18n.with_locale(note.account.user_locale.presence || I18n.default_locale) do
      I18n.t('move_handler.copy_account_note_text', acct: @source_account.acct)
    end

    new_note = @target_account.targeted_account_notes.find_by(account: note.account)
    if new_note.nil?
      # 新建备注，前缀迁移提示
      @target_account.targeted_account_notes.create!(
        account: note.account, 
        comment: [text, note.comment].join("\n")
      )
    else
      # 合并备注
      new_note.update!(comment: [text, note.comment, "\n", new_note.comment].join("\n"))
    end
  end
end
```

#### 6.2 迁移屏蔽关系
```ruby
def carry_blocks_over!
  @source_account.blocked_by_relationships.where(account: Account.local).find_each do |block|
    unless skip_block_move?(block)
      BlockService.new.call(block.account, @target_account)
      add_account_note_if_needed!(block.account, 'move_handler.carry_blocks_over_text')
    end
  end
end

def skip_block_move?(block)
  # 如果已经屏蔽了新账号，或者正在关注新账号，则跳过
  block.account.blocking?(@target_account) || block.account.following?(@target_account)
end
```

#### 6.3 迁移静音关系
```ruby
def carry_mutes_over!
  @source_account.muted_by_relationships.where(account: Account.local).find_each do |mute|
    unless skip_mute_move?(mute)
      MuteService.new.call(mute.account, @target_account, notifications: mute.hide_notifications)
      add_account_note_if_needed!(mute.account, 'move_handler.carry_mutes_over_text')
    end
  end
end
```

---

## 跨实例协议校验配合

### 1. 双向引用验证机制

整个迁移流程的核心安全机制是**双向引用**：

| 方向 | 字段 | 设置时机 | 验证位置 |
|------|------|----------|----------|
| 新 → 旧 | `also_known_as` | 新账号添加别名时 | `AccountMigration.validate_target_account` + `ActivityPub::Activity::Move.perform` |
| 旧 → 新 | `moved_to_account_id` | 旧账号发起迁移时 | `ActivityPub::ProcessAccountService` 解析 actor 时 |

### 2. 验证时机

#### 2.1 旧实例发起时验证
```ruby
# app/models/account_migration.rb:83
errors.add(:acct, I18n.t('migrations.errors.missing_also_known_as')) 
  unless target_account.also_known_as.include?(ActivityPub::TagManager.instance.uri_for(account))
```

**目的**：在用户发起迁移时就提示错误，避免无效的 Move 活动。

#### 2.2 接收方验证（最重要）
```ruby
# app/lib/activitypub/activity/move.rb:12
if target_account.nil? || target_account.unavailable? || 
   !target_account.also_known_as.include?(origin_account.uri)
  unmark_as_processing!
  return
end
```

**目的**：防止恶意实例伪造 Move 活动。即使旧实例被攻破，没有新账号的 `also_known_as` 配合，迁移也无法完成。

### 3. ActivityPub 协议层面

#### 3.1 Move 活动结构
```json
{
  "type": "Move",
  "actor": "https://old.example.com/users/olduser",   // 迁移发起人
  "object": "https://old.example.com/users/olduser",  // 被迁移的账号（必须等于 actor）
  "target": "https://new.example.com/users/newuser"   // 目标账号
}
```

#### 3.2 alsoKnownAs 在 Actor 中的表示
当新账号添加了旧账号作为别名后，其 ActivityPub Actor 文档中会包含：
```json
{
  "type": "Person",
  "id": "https://new.example.com/users/newuser",
  "alsoKnownAs": [
    "https://old.example.com/users/olduser"
  ],
  // ... 其他字段
}
```

#### 3.3 movedTo 在 Actor 中的表示
当旧账号设置了迁移后，其 ActivityPub Actor 文档中会包含：
```json
{
  "type": "Person",
  "id": "https://old.example.com/users/olduser",
  "movedTo": "https://new.example.com/users/newuser",
  // ... 其他字段
}
```

### 4. 流程中的协议交互

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   新账号实例     │     │   旧账号实例     │     │  关注者实例      │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                         │                         │
         │  1. 用户添加别名         │                         │
         │  (POST /settings/aliases)│                         │
         │◄─────────────────────────│                         │
         │                         │                         │
         │  2. 分发 Update 活动     │                         │
         │  (Actor 包含 alsoKnownAs) │                         │
         │─────────────────────────►│────────────────────────►│
         │                         │                         │
         │                         │  3. 用户发起迁移          │
         │                         │  (POST /settings/migration)│
         │◄─────────────────────────│                         │
         │  WebFinger 解析新账号     │                         │
         │                         │                         │
         │  4. 验证 alsoKnownAs     │                         │
         │  (目标账号是否认领源账号)  │                         │
         │─────────────────────────►│                         │
         │                         │                         │
         │                         │  5. 分发 Update 活动      │
         │                         │  (Actor 包含 movedTo)    │
         │◄─────────────────────────│────────────────────────►│
         │                         │                         │
         │                         │  6. 分发 Move 活动        │
         │◄─────────────────────────│────────────────────────►│
         │                         │                         │
         │                         │                         │  7. 接收方验证
         │                         │                         │  - object == actor?
         │                         │                         │  - 7天冷却期?
         │                         │                         │  - alsoKnownAs 验证?
         │                         │                         │
         │                         │                         │  8. 处理关注关系转移
         │                         │                         │  - MoveWorker
         │                         │                         │  - UnfollowFollowWorker
         │                         │                         │  - FollowMigrationService
```

---

## 关键数据结构

### 1. Account 模型关键字段

| 字段 | 类型 | 用途 |
|------|------|------|
| `also_known_as` | string[] | 存储别名账号 URI，用于迁移验证 |
| `moved_to_account_id` | bigint | 指向迁移目标账号 |
| `uri` | string | ActivityPub Actor ID，用于协议验证 |

**代码位置**：`app/models/account.rb:9`

```ruby
# Schema 片段
#  also_known_as                 :string           is an Array
#  moved_to_account_id           :bigint(8)
#  uri                           :string           default(""), not null
```

### 2. AccountMigration 模型

**文件**：`app/models/account_migration.rb`

| 字段 | 类型 | 用途 |
|------|------|------|
| `account_id` | bigint | 源账号 ID |
| `target_account_id` | bigint | 目标账号 ID |
| `acct` | string | 目标账号地址（用于表单输入） |
| `followers_count` | bigint | 迁移时的粉丝数（记录用） |

**常量**：
- `COOLDOWN_PERIOD = 30.days.freeze`：迁移冷却期

### 3. AccountAlias 模型

**文件**：`app/models/account_alias.rb`

| 字段 | 类型 | 用途 |
|------|------|------|
| `account_id` | bigint | 所属账号 ID |
| `acct` | string | 别名账号地址 |
| `uri` | string | 别名账号的 URI |

**回调**：
- `after_create :add_to_account`：添加到 `also_known_as`
- `after_destroy :remove_from_account`：从 `also_known_as` 移除

### 4. Redis 键

| 键模式 | 用途 | 过期时间 |
|--------|------|----------|
| `account_migration:{account_id}` | 迁移操作锁 | - |
| `move_in_progress:{account_id}` | Move 处理冷却标记 | 7天 |

---

## 流程图总结

### 完整迁移流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           账号迁移完整流程                                      │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐
│  用户操作阶段  │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│ 1. 新账号添加别名 (新实例)                                      │
│    ┌─────────────────┐                                         │
│    │ 页面: /settings/aliases                                   │
│    │ 控制器: Settings::AliasesController                       │
│    │ 模型: AccountAlias                                         │
│    │                                                             │
│    │ 流程:                                                       │
│    │   输入旧账号地址 → WebFinger 解析 → 验证 →                  │
│    │   更新 also_known_as → 分发 Update 活动                   │
└──────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│ 2. 旧账号发起迁移 (旧实例)                                      │
│    ┌─────────────────┐                                         │
│    │ 页面: /settings/migration                                 │
│    │ 控制器: Settings::MigrationsController                    │
│    │ 模型: AccountMigration                                     │
│    │                                                             │
│    │ 验证:                                                       │
│    │   - 密码/用户名验证                                         │
│    │   - 目标账号 also_known_as 包含源账号 URI (核心验证)       │
│    │   - 30天冷却期                                             │
│    │   - 不是迁移到自己                                          │
└──────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────┐
│  协议分发阶段  │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│ 3. MoveService 执行 (旧实例)                                   │
│    ┌─────────────────┐                                         │
│    │ 文件: app/services/move_service.rb                       │
│    │                                                             │
│    │ 步骤:                                                       │
│    │   1. update_redirect!                                      │
│    │      → 设置 source_account.moved_to_account               │
│    │                                                             │
│    │   2. process_local_relationships!                          │
│    │      → MoveWorker.perform_async (处理本地关注)             │
│    │                                                             │
│    │   3. distribute_update!                                    │
│    │      → ActivityPub::UpdateDistributionWorker              │
│    │      (通知全网账号已迁移)                                    │
│    │                                                             │
│    │   4. distribute_move!                                      │
│    │      → ActivityPub::MoveDistributionWorker                │
│    │      (发送 Move 活动给所有关注者)                            │
└──────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│ 4. MoveDistributionWorker 分发 (旧实例)                        │
│    ┌─────────────────┐                                         │
│    │ 文件: app/workers/activitypub/move_distribution_worker.rb│
│    │                                                             │
│    │ 序列化: ActivityPub::MoveSerializer                       │
│    │                                                             │
│    │ 目标:                                                       │
│    │   - 所有关注者的 inbox                                      │
│    │   - 所有被屏蔽者的 inbox                                    │
│    │   - 中继服务器 (Relay)                                      │
│    │                                                             │
│    │ Activity 格式:                                              │
│    │   {                                                         │
│    │     "type": "Move",                                         │
│    │     "actor": "https://old/users/old",                      │
│    │     "object": "https://old/users/old",  ← 必须等于 actor  │
│    │     "target": "https://new/users/new"                      │
│    │   }                                                         │
└──────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────┐
│  接收处理阶段  │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│ 5. ActivityPub::Activity::Move 处理 (关注者实例)              │
│    ┌─────────────────┐                                         │
│    │ 文件: app/lib/activitypub/activity/move.rb              │
│    │                                                             │
│    │ 验证 (任意失败则中止):                                       │
│    │   1. origin_account.uri == object_uri                     │
│    │      → 防止假冒迁移                                         │
│    │                                                             │
│    │   2. mark_as_processing! (7天冷却期)                      │
│    │      → Redis: move_in_progress:{account_id}               │
│    │                                                             │
│    │   3. 目标账号验证 (核心):                                   │
│    │      - target_account 存在且可用                            │
│    │      - target_account.also_known_as.include?(源账号 URI)  │
│    │      → 双向引用验证，确保新账号"认领"了旧账号               │
│    │                                                             │
│    │ 通过后执行:                                                 │
│    │   - 更新本地缓存: origin_account.moved_to_account          │
│    │   - MoveWorker.perform_async (处理关注关系转移)             │
└──────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────┐
│  关系转移阶段  │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│ 6. MoveWorker 处理 (关注者实例)                                │
│    ┌─────────────────┐                                         │
│    │ 文件: app/workers/move_worker.rb                         │
│    │                                                             │
│    │ 分支:                                                       │
│    │   ┌─────────────────────────────────────────────────┐    │
│    │   │ 场景A: 源账号和目标账号都是本地账号                 │    │
│    │   │ → rewrite_follows! (批量数据库更新)               │    │
│    │   │                                                    │    │
│    │   │ 三阶段处理:                                         │    │
│    │   │   1. 处理待处理的关注请求                           │    │
│    │   │   2. 处理同时关注新旧账号的情况                      │    │
│    │   │   3. 处理只关注旧账号的情况 (批量 update_all)       │    │
│    │   │                                                    │    │
│    │   │ 同时处理:                                           │    │
│    │   │   - ListAccount 列表成员迁移                        │    │
│    │   │   - 缓存清理 (update_all 不触发回调)                │    │
│    │   └─────────────────────────────────────────────────┘    │
│    │                                                             │
│    │   ┌─────────────────────────────────────────────────┐    │
│    │   │ 场景B: 跨实例迁移                                 │    │
│    │   │ → queue_follow_unfollows!                        │    │
│    │   │                                                    │    │
│    │   │ 批量推送:                                          │    │
│    │   │   UnfollowFollowWorker.push_bulk                  │    │
│    │   │                                                    │    │
│    │   │ 参数: [follower_id, old_id, new_id, bypass_locked]│
│    │   └─────────────────────────────────────────────────┘    │
│    │                                                             │
│    │ 其他数据迁移:                                               │
│    │   - copy_account_notes!  (复制账号备注)                   │
│    │   - carry_blocks_over!    (迁移屏蔽关系)                  │
│    │   - carry_mutes_over!     (迁移静音关系)                  │
└──────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│ 7. UnfollowFollowWorker + FollowMigrationService              │
│    ┌─────────────────┐                                         │
│    │ 文件: app/workers/unfollow_follow_worker.rb             │
│    │       app/services/follow_migration_service.rb          │
│    │                                                             │
│    │ 流程:                                                       │
│    │   1. 读取原有关注设置 (reblogs, notify, languages)        │
│    │   2. 根据新账号状态选择处理方式:                            │
│    │                                                             │
│    │      ┌──────────────┬────────────────────────────┐       │
│    │      │ 场景         │ 处理方式                     │       │
│    │      ├──────────────┼────────────────────────────┤       │
│    │      │ 新账号锁定   │ request_follow!            │       │
│    │      │              │ → 创建关注请求              │       │
│    │      │              │ → 发送通知/ActivityPub 活动 │       │
│    │      ├──────────────┼────────────────────────────┤       │
│    │      │ 已有请求     │ change_follow_request_     │       │
│    │      │              │ options!                   │       │
│    │      │              │ → 更新请求选项              │       │
│    │      ├──────────────┼────────────────────────────┤       │
│    │      │ 可直接关注   │ direct_follow!             │       │
│    │      │              │ → 立即创建关注关系          │       │
│    │      │              │ → 立即取消关注旧账号        │       │
│    │      └──────────────┴────────────────────────────┘       │
│    │                                                             │
│    │   3. 迁移列表成员关系 (migrate_list_accounts!)            │
│    │   4. 取消关注旧账号 (UnfollowService)                      │
└──────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────┐
│   完成状态    │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│ 最终状态:                                                       │
│   ✓ 旧账号: moved_to_account 指向新账号                        │
│   ✓ 新账号: also_known_as 包含旧账号 URI                       │
│   ✓ 关注者: 关注关系从旧账号转移到新账号                        │
│   ✓ 列表: 列表成员关系同步更新                                  │
│   ✓ 备注/屏蔽/静音: 同步迁移                                    │
└──────────────────────────────────────────────────────────────┘
```

---

## 关键安全机制总结

### 1. 双向引用验证（核心）

```
新账号 ──alsoKnownAs──► 旧账号 URI (认领)
旧账号 ──movedTo───────► 新账号 URI (指向)
```

**验证位置**：
1. 旧实例发起时：`AccountMigration.validate_target_account`
2. 接收方处理时：`ActivityPub::Activity::Move.perform`

### 2. 冷却期机制

| 冷却期 | 时长 | 用途 | 实现 |
|--------|------|------|------|
| 迁移冷却期 | 30天 | 防止频繁发起迁移 | `AccountMigration.within_cooldown` |
| 处理冷却期 | 7天 | 防止重复处理 Move 活动 | Redis `move_in_progress:{id}` |

### 3. 操作锁

```ruby
# app/models/account_migration.rb:54-56
with_redis_lock("account_migration:#{account.id}") do
  save
end
```

防止并发迁移操作。

### 4. 身份验证

- 密码验证（有密码的账号）
- 用户名验证（无密码的账号，如 OAuth 登录）

---

## 错误处理

### 1. 验证错误

| 错误类型 | 错误信息 Key | 触发条件 |
|----------|-------------|----------|
| 目标账号不存在 | `migrations.errors.not_found` | WebFinger 解析失败 |
| 缺少 also_known_as | `migrations.errors.missing_also_known_as` | 目标账号未认领源账号 |
| 已迁移过 | `migrations.errors.already_moved` | 重复迁移到同一账号 |
| 迁移到自己 | `migrations.errors.move_to_self` | 目标是自己 |
| 冷却期 | `migrations.errors.on_cooldown` | 30天内已迁移过 |

### 2. 运行时错误

- `ActiveRecord::RecordNotFound`：账号已删除 → 静默失败
- `Mastodon::NotPermittedError`：无权限 → 静默失败
- 其他异常 → 记录错误，重试机制由 Sidekiq 处理

---

## 性能优化

### 1. 批量操作

- `find_in_batches` + `push_bulk`：批量推送 Sidekiq 任务
- `in_batches` + `update_all`：批量更新数据库（不实例化对象）
- 限制每批 1000 条（`ActivityPub::DeliveryWorker.push_bulk(..., limit: 1_000)`）

### 2. 异步处理

所有耗时操作都通过 Sidekiq 异步处理：
- `MoveWorker`：主迁移任务
- `UnfollowFollowWorker`：单条关注关系处理
- `ActivityPub::DeliveryWorker`：协议分发

### 3. 缓存策略

- `update_all` 后手动清理缓存（`Rails.cache.delete_multi`）
- `ResolveAccountService` 支持 `skip_cache: true`（迁移时强制刷新）

---

## 相关文件索引

| 功能 | 文件路径 |
|------|----------|
| 迁移模型 | `app/models/account_migration.rb` |
| 别名模型 | `app/models/account_alias.rb` |
| 迁移服务 | `app/services/move_service.rb` |
| 关注迁移服务 | `app/services/follow_migration_service.rb` |
| 主迁移 Worker | `app/workers/move_worker.rb` |
| 单条关注迁移 Worker | `app/workers/unfollow_follow_worker.rb` |
| Move 活动分发 | `app/workers/activitypub/move_distribution_worker.rb` |
| Move 活动处理 | `app/lib/activitypub/activity/move.rb` |
| Move 序列化器 | `app/serializers/activitypub/move_serializer.rb` |
| 迁移控制器 | `app/controllers/settings/migrations_controller.rb` |
| 别名控制器 | `app/controllers/settings/aliases_controller.rb` |
| 重定向控制器 | `app/controllers/settings/migration/redirects_controller.rb` |
| 迁移页面视图 | `app/views/settings/migrations/show.html.haml` |
| 别名页面视图 | `app/views/settings/aliases/index.html.haml` |
| 重定向页面视图 | `app/views/settings/migration/redirects/new.html.haml` |

---

## 测试文件索引

| 测试对象 | 文件路径 |
|----------|----------|
| MoveService | `spec/services/move_service_spec.rb` |
| Move 活动处理 | `spec/lib/activitypub/activity/move_spec.rb` |
| 迁移系统测试 | `spec/system/settings/migrations_spec.rb` |
| 重定向系统测试 | `spec/system/settings/migration/redirects_spec.rb` |
| 重定向请求测试 | `spec/requests/settings/migration/redirects_spec.rb` |
| Move 序列化器 | `spec/serializers/activitypub/move_serializer_spec.rb` |
| UnfollowFollowWorker | `spec/workers/unfollow_follow_worker_spec.rb` |

---

*文档生成日期: 2026-05-05*
