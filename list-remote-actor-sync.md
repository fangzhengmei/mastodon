# Mastodon 列表与远端账号同步机制

## 1. 数据模型与关系

### 1.1 List 模型

列表是用户创建的用于组织关注账号的分组。

**核心属性**：
- `title`：列表名称
- `replies_policy`：回复显示策略（`list`/`followed`/`none`）
- `exclusive`：是否为排他列表（在 exclusive 列表中的账号不会出现在主时间线）
- `account_id`：列表创建者的账号 ID

**关键关联**：
```ruby
# app/models/list.rb
belongs_to :account
has_many :list_accounts, inverse_of: :list, dependent: :destroy
has_many :accounts, through: :list_accounts
has_many :active_accounts, -> { merge(ListAccount.active) }, through: :list_accounts, source: :account
```

### 1.2 ListAccount 模型

ListAccount 是列表与账号之间的关联表，管理列表中的成员关系。

**核心属性**：
- `list_id`：列表 ID
- `account_id`：账号 ID
- `follow_id`：关注关系 ID（可为空）
- `follow_request_id`：关注请求 ID（可为空）

**关键逻辑**：
```ruby
# app/models/list_account.rb
scope :active, -> { where.not(follow_id: nil) }

before_validation :set_follow, unless: :list_owner_account_is_account?

def set_follow
  self.follow = Follow.find_by(account_id: list.account_id, target_account_id: account.id)
  self.follow_request = FollowRequest.find_by(account_id: list.account_id, target_account_id: account.id) if follow.nil?
end
```

**重要特性**：
1. **必须有关注关系**：除了列表所有者自己，其他账号必须已被关注或有未决的关注请求才能被添加到列表
2. **active 状态**：只有 `follow_id` 不为空的 ListAccount 才被视为活跃，其状态会参与时间线 fan-out
3. **自动关联**：添加账号到列表时，自动查找并关联对应的 Follow 或 FollowRequest

### 1.3 Account 模型

账号是 Mastodon 的核心实体，通过 `domain` 字段区分本地和远端账号。

**区分方式**：
```ruby
# app/models/account.rb
def local?
  domain.nil?
end

def remote?
  !domain.nil?
end
```

**远端账号特征**：
- `domain` 存储远端实例域名（如 `mastodon.social`）
- 有 `inbox_url`、`outbox_url`、`shared_inbox_url`、`uri` 等 ActivityPub 相关字段
- `protocol` 为 `activitypub`（或已废弃的 `ostatus`）

## 2. 列表时间线 Fan-out 机制

### 2.1 触发点：FanOutOnWriteService

当新状态发布或更新时，`FanOutOnWriteService` 负责将状态分发给相关时间线。

**核心流程**：
```ruby
# app/services/fan_out_on_write_service.rb
def call(status, options = {})
  @status    = status
  @account   = status.account
  
  fan_out_to_local_recipients!
  fan_out_to_public_recipients! if broadcastable?
  fan_out_to_public_streams! if broadcastable?
end

def fan_out_to_local_recipients!
  deliver_to_self!
  # ... 通知相关代码
  
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

### 2.2 查找相关列表：lists_for_local_distribution

`deliver_to_lists!` 方法使用 `lists_for_local_distribution` 找到需要更新的列表。

**实现逻辑**：
```ruby
# app/models/concerns/account/interactions.rb
def lists_for_local_distribution
  scope = lists.joins(account: :user)
  scope.where.not(list_accounts: { follow_id: nil }).or(scope.where(account_id: id))
    .merge(User.signed_in_recently)
end
```

**筛选条件**：
1. **有活跃关注关系**：`list_accounts.follow_id` 不为空
2. **或列表所有者就是该账号**：`account_id = id`（自己的列表）
3. **列表所有者最近登录过**：`User.signed_in_recently`（优化性能，只更新活跃用户）

**分发代码**：
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

### 2.3 异步处理：FeedInsertWorker

`FeedInsertWorker` 异步处理时间线的插入和过滤。

**核心逻辑**：
```ruby
# app/workers/feed_insert_worker.rb
def perform(status_id, id, type = 'home', options = {})
  with_primary do
    @type      = type.to_sym
    @status    = Status.find(status_id)
    @options   = options.symbolize_keys
    
    case @type
    when :home, :tags
      @follower = Account.find(id)
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
  when :home
    FeedManager.instance.filter(:home, @status, @follower)
  when :tags
    FeedManager.instance.filter(:tags, @status, @follower)
  when :list
    FeedManager.instance.filter(:list, @status, @list)
  end
end

def perform_push
  case @type
  when :home, :tags
    FeedManager.instance.push_to_home(@follower, @status, update: update?)
  when :list
    FeedManager.instance.push_to_list(@list, @status, update: update?)
  end
end
```

### 2.4 时间线管理：FeedManager

`FeedManager` 是单例类，负责管理各种时间线的 Redis 存储和流式更新。

**列表时间线相关方法**：

#### push_to_list - 推送状态到列表时间线
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

**关键步骤**：
1. **过滤检查**：调用 `filter_from_list?` 检查状态是否应该被过滤
2. **活跃用户检查**：只推送给最近登录的用户
3. **添加到 Redis**：调用 `add_to_feed` 将状态添加到 Redis 有序集合
4. **裁剪时间线**：调用 `trim` 保持时间线大小不超过 `MAX_ITEMS`（800）
5. **推送更新**：如果有客户端订阅，调用 `PushUpdateWorker` 发送流式更新

#### filter_from_list? - 列表时间线过滤逻辑
```ruby
# app/lib/feed_manager.rb
def filter_from_list?(status, list)
  if status.reply? && status.in_reply_to_account_id != status.account_id  # 状态是对其他账号的回复
    should_filter = status.in_reply_to_account_id != list.account_id     # 不是回复给列表所有者
    should_filter &&= !list.show_followed?                                # 列表策略不是 show_followed
    should_filter &&= !(list.show_list? && ListAccount.exists?(list_id: list.id, account_id: status.in_reply_to_account_id))  # 或者回复对象不在列表中
    
    return !!should_filter
  end
  
  false
end
```

**过滤规则**（仅适用于回复）：
- 如果 `replies_policy` 是 `list`：只显示对列表中账号的回复
- 如果 `replies_policy` 是 `followed`：显示对所有已关注账号的回复
- 如果 `replies_policy` 是 `none`：不显示任何回复

#### merge_into_list - 合并历史状态到列表时间线
```ruby
# app/lib/feed_manager.rb
def merge_into_list(from_account, list)
  return unless list.account.user&.signed_in_recently?
  
  timeline_key = key(:list, list.id)
  aggregate    = list.account.user&.aggregates_reblogs?
  query        = from_account.statuses.list_eligible_visibility.includes(reblog: :account).limit(FeedManager::MAX_ITEMS / 4)
  
  # 如果时间线已满，优化查询条件
  if redis.zcard(timeline_key) >= FeedManager::MAX_ITEMS / 4
    oldest_home_score = redis.zrange(timeline_key, 0, 0, with_scores: true).first.last.to_i
    # ... 优化逻辑
  end
  
  statuses = query.to_a
  crutches = build_crutches(list.account_id, statuses, list: list)
  
  statuses.each do |status|
    next if filter_from_home(status, list.account_id, crutches, :list)
    
    add_to_feed(:list, list.id, status, aggregate_reblogs: aggregate)
  end
  
  trim(:list, list.id)
end
```

**使用场景**：
- 将账号添加到列表时
- 账号从暂停状态恢复时

#### unmerge_from_list - 从列表时间线移除账号的所有状态
```ruby
# app/lib/feed_manager.rb
def unmerge_from_list(from_account, list)
  timeline_key        = key(:list, list.id)
  timeline_status_ids = redis.zrange(timeline_key, 0, -1)
  
  from_account.statuses.select(:id, :reblog_of_id).where(id: timeline_status_ids).reorder(nil).find_each do |status|
    remove_from_feed(:list, list.id, status, aggregate_reblogs: list.account.user&.aggregates_reblogs?)
  end
end
```

**使用场景**：
- 从列表移除账号时
- 账号被暂停时

### 2.5 流式更新：PushUpdateWorker

当状态被添加到时间线且有客户端订阅时，`PushUpdateWorker` 负责发送流式更新。

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

**频道命名**：
- 列表时间线：`timeline:list:{list_id}`
- 主页时间线：`timeline:{account_id}`
- 通知时间线：`timeline:{account_id}:notifications`

## 3. 远端账号状态变化的检测与处理

### 3.1 ActivityPub 消息处理

Mastodon 通过 ActivityPub 协议接收远端实例的消息。

#### 入口：InboxesController
```ruby
# app/controllers/activitypub/inboxes_controller.rb
def create
  # ... 签名验证
  
  @json = Oj.load(body, mode: :strict)
  
  if @json['signature'].present?
    ActivityPub::ProcessingWorker.perform_async(@account&.id, body.dup)
  else
    # ... 处理逻辑
  end
end
```

#### 消息处理：ActivityPub::ProcessingWorker
```ruby
# app/workers/activitypub/processing_worker.rb
def perform(actor_id, body)
  # ... 处理逻辑
  ActivityPub::Activity.factory(json, account, delivery_attempt_options).perform
end
```

### 3.2 关注关系变化处理

#### 接收关注请求：ActivityPub::Activity::Follow
```ruby
# app/lib/activitypub/activity/follow.rb
def perform
  target_account = account_from_uri(object_uri)
  
  return if target_account.nil? || !target_account.local?
  
  # 检查是否已有关注请求
  existing_follow_request = ::FollowRequest.find_by(account: @account, target_account: target_account)
  unless existing_follow_request.nil?
    existing_follow_request.update!(uri: @json['id'])
    return
  end
  
  # 检查是否已有关注关系
  existing_follow = ::Follow.find_by(account: @account, target_account: target_account)
  unless existing_follow.nil?
    existing_follow.update!(uri: @json['id'])
    AuthorizeFollowService.new.call(@account, target_account, skip_follow_request: true, follow_request_uri: @json['id'])
    return
  end
  
  # 创建新的关注请求
  follow_request = FollowRequest.create!(account: @account, target_account: target_account, uri: @json['id'])
  
  if target_account.locked? || @account.silenced?
    LocalNotificationWorker.perform_async(target_account.id, follow_request.id, 'FollowRequest', 'follow_request')
  else
    AuthorizeFollowService.new.call(@account, target_account)
    LocalNotificationWorker.perform_async(target_account.id, ::Follow.find_by(account: @account, target_account: target_account).id, 'Follow', 'follow')
  end
end
```

#### 接收取消关注：ActivityPub::Activity::Undo
```ruby
# app/lib/activitypub/activity/undo.rb
def perform
  case @object['type']
  when 'Announce'
    undo_announce
  when 'Accept'
    undo_accept
  when 'Follow'
    undo_follow  # 处理取消关注
  when 'Like'
    undo_like
  when 'Block'
    undo_block
  # ...
  end
end

def undo_follow
  target_account = account_from_uri(target_uri)
  
  return if target_account.nil? || !target_account.local?
  
  if @account.following?(target_account)
    @account.unfollow!(target_account)  # 取消关注
  elsif @account.requested?(target_account)
    FollowRequest.find_by(account: @account, target_account: target_account)&.destroy
  else
    delete_later!(object_uri)
  end
end
```

### 3.3 账号状态变化服务

#### 暂停账号：SuspendAccountService
```ruby
# app/services/suspend_account_service.rb
def call(account)
  return unless account.suspended?
  
  @account = account
  
  reject_remote_follows!
  distribute_update_actor!
  unmerge_from_home_timelines!
  unmerge_from_list_timelines!  # 从列表时间线移除
  privatize_media_attachments!
  remove_from_trends!
end

def unmerge_from_list_timelines!
  @account.lists_for_local_distribution.reorder(nil).find_each do |list|
    FeedManager.instance.unmerge_from_list(@account, list)
  end
end
```

**远端账号暂停的特殊处理**：
```ruby
def reject_remote_follows!
  return if @account.local? || !@account.activitypub? || @account.suspension_origin_remote?
  
  # 当暂停一个远端账号时，该账号在其源实例上并没有真正被暂停
  # 为了防止它继续接收因为关注本地账号而获得的状态，我们必须强制它取消关注
  
  Follow.where(account: @account).find_in_batches do |follows|
    ActivityPub::DeliveryWorker.push_bulk(follows) do |follow|
      [serialize_payload(follow, ActivityPub::RejectFollowSerializer).to_json, follow.target_account_id, @account.inbox_url]
    end
    
    follows.each(&:destroy)
  end
end
```

#### 恢复账号：UnsuspendAccountService
```ruby
# app/services/unsuspend_account_service.rb
def call(account)
  @account = account
  
  refresh_remote_account!  # 刷新远端账号信息
  
  return if @account.nil? || @account.suspended?
  
  merge_into_home_timelines!
  merge_into_list_timelines!  # 合并回列表时间线
  publish_media_attachments!
  distribute_update_actor!
end

def refresh_remote_account!
  return if @account.local?
  
  # 当我们暂停远端账号时，它可能在其源实例上也被暂停了
  # 所以需要立即刷新以检查这种情况
  
  @account.update!(last_webfingered_at: nil)
  @account = ResolveAccountService.new.call(@account)
  
  # 需要注意的是，远端账号可能不仅被暂停，还被永久删除
  # 这种情况下 @account 会是 nil
end

def merge_into_list_timelines!
  @account.lists_for_local_distribution.reorder(nil).find_each do |list|
    FeedManager.instance.merge_into_list(@account, list)
  end
end
```

### 3.4 关注关系变化对列表的影响

当关注关系发生变化时，ListAccount 的 `follow_id` 字段会自动更新或失效。

**添加账号到列表时**：
```ruby
# app/models/list_account.rb
before_validation :set_follow, unless: :list_owner_account_is_account?

def set_follow
  self.follow = Follow.find_by(account_id: list.account_id, target_account_id: account.id)
  self.follow_request = FollowRequest.find_by(account_id: list.account_id, target_account_id: account.id) if follow.nil?
end
```

**关注请求被接受时**：
- Follow 被创建，ListAccount 的 `follow_id` 会在下次验证时被设置

**关注被取消时**：
- Follow 被销毁，ListAccount 的 `follow_id` 变为 nil
- 该账号不再出现在 `lists_for_local_distribution` 中
- 新状态不再会被分发到包含该账号的列表

## 4. 列表 UI 与数据变化同步

### 4.1 Streaming API 服务

Mastodon 使用独立的 Node.js 服务处理实时流式更新。

**入口文件**：`streaming/index.js`

**核心功能**：
- 支持 WebSocket 和 Server-Sent Events 两种连接方式
- 订阅 Redis 频道
- 处理客户端的订阅/取消订阅请求
- 过滤和转发消息

### 4.2 列表流订阅

#### 订阅流程
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

#### 权限验证
```javascript
const authorizeListAccess = async (listId, req) => {
  const { accountId } = req;
  
  const result = await pgPool.query('SELECT id, account_id FROM lists WHERE id = $1 AND account_id = $2 LIMIT 1', [listId, accountId]);
  
  if (result.rows.length === 0) {
    throw new AuthenticationError('List not found');
  }
};
```

**验证规则**：
- 用户只能订阅自己创建的列表
- 通过查询 `lists` 表确认 `account_id` 匹配

### 4.3 消息分发机制

#### Redis 订阅
```javascript
const subscribe = (channel, callback) => {
  subs[channel] = subs[channel] || [];
  
  if (subs[channel].length === 0) {
    redisSubscribeClient.subscribe(redisNamespaced(channel), (err, count) => {
      // ...
    });
  }
  
  subs[channel].push(callback);
};
```

#### 消息处理
```javascript
const onRedisMessage = (channel, message) => {
  const key = redisUnnamespaced(channel);
  const callbacks = subs[key];
  if (!callbacks) {
    return;
  }
  
  const json = parseJSON(message, null);
  if (!json) return;
  
  callbacks.forEach(callback => callback(json));
};
```

#### 发送到客户端
```javascript
const transmit = (event, payload) => {
  const encodedPayload = typeof payload === 'object' ? JSON.stringify(payload) : payload;
  output(event, encodedPayload);
};
```

### 4.4 心跳与连接管理

#### 订阅心跳
```javascript
const subscriptionHeartbeat = channels => {
  const interval = 6 * 60;  // 6 分钟
  
  const tellSubscribed = () => {
    channels.forEach(channel => redisClient.set(redisNamespaced(`subscribed:${channel}`), '1', 'EX', interval * 3));
  };
  
  tellSubscribed();
  
  const heartbeat = setInterval(tellSubscribed, interval * 1000);
  
  return () => {
    clearInterval(heartbeat);
  };
};
```

**作用**：
- 在 Redis 中设置 `subscribed:{channel}` 键，有效期 18 分钟
- 每 6 分钟刷新一次
- 服务端通过检查该键判断是否有客户端订阅

#### 推送更新检查
```ruby
# app/lib/feed_manager.rb
def push_update_required?(timeline_key)
  redis.exists?("subscribed:#{timeline_key}")
end
```

### 4.5 列表时间线 API

#### 控制器
```ruby
# app/controllers/api/v1/timelines/list_controller.rb
class Api::V1::Timelines::ListController < Api::V1::Timelines::BaseController
  def show
    render json: @statuses,
           each_serializer: REST::StatusSerializer,
           relationships: StatusRelationshipsPresenter.new(@statuses, current_user.account_id)
  end
  
  private
  
  def set_list
    @list = List.where(account: current_account).find(params[:id])
  end
  
  def list_statuses
    list_feed.get(
      limit_param(DEFAULT_STATUSES_LIMIT),
      params[:max_id],
      params[:since_id],
      params[:min_id]
    )
  end
  
  def list_feed
    ListFeed.new(@list)
  end
end
```

#### ListFeed
```ruby
# app/models/list_feed.rb
class ListFeed < Feed
  def initialize(list)
    super(:list, list.id)
  end
end
```

#### Feed 基类
Feed 类封装了对 Redis 有序集合的操作，提供分页获取时间线数据的方法。

## 5. 列表账号管理操作

### 5.1 添加账号到列表

#### 控制器
```ruby
# app/controllers/api/v1/lists/accounts_controller.rb
def create
  AddAccountsToListService.new.call(@list, Account.find(account_ids))
  render_empty
end
```

#### 服务
```ruby
# app/services/add_accounts_to_list_service.rb
class AddAccountsToListService < BaseService
  def call(list, accounts)
    @list = list
    @accounts = accounts
    
    return if @accounts.empty?
    
    update_list!
    merge_into_list!
  end
  
  private
  
  def update_list!
    ApplicationRecord.transaction do
      @accounts.each do |account|
        @list.accounts << account
      end
    end
  end
  
  def merge_into_list!
    MergeWorker.push_bulk(merge_account_ids) do |account_id|
      [account_id, @list.id, 'list']
    end
  end
  
  def merge_account_ids
    ListAccount.where(list: @list, account: @accounts).where.not(follow_id: nil).pluck(:account_id)
  end
end
```

**关键点**：
- `merge_account_ids` 只选择有活跃关注关系的账号
- 没有 `follow_id` 的账号（如只有 `follow_request_id` 或列表所有者自己）不会触发历史状态合并

### 5.2 从列表移除账号

#### 控制器
```ruby
# app/controllers/api/v1/lists/accounts_controller.rb
def destroy
  RemoveAccountsFromListService.new.call(@list, Account.where(id: account_ids))
  render_empty
end
```

#### 服务
```ruby
# app/services/remove_accounts_from_list_service.rb
class RemoveAccountsFromListService < BaseService
  def call(list, accounts)
    @list = list
    @accounts = accounts
    
    return if @accounts.empty?
    
    unmerge_from_list!
    update_list!
  end
  
  private
  
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
end
```

### 5.3 MergeWorker
```ruby
# app/workers/merge_worker.rb
def merge_into_list!(into_list_id)
  with_primary do
    @into_list = List.find(into_list_id)
  end
  
  with_read_replica do
    FeedManager.instance.merge_into_list(@from_account, @into_list)
  end
end
```

### 5.4 UnmergeWorker
```ruby
# app/workers/unmerge_worker.rb
def unmerge_from_list!(into_list_id)
  with_primary do
    @into_list = List.find(into_list_id)
  end
  
  with_read_replica do
    FeedManager.instance.unmerge_from_list(@from_account, @into_list)
  end
end
```

## 6. 完整流程图

### 6.1 新状态发布到列表时间线

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        新状态发布/更新流程                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Status 被创建或更新                                                      │
│         │                                                                    │
│         ▼                                                                    │
│  2. FanOutOnWriteService.call(status)                                       │
│         │                                                                    │
│         ▼                                                                    │
│  3. fan_out_to_local_recipients!                                            │
│         │                                                                    │
│         ├──► deliver_to_self!           (推送给自己)                          │
│         │                                                                    │
│         ├──► deliver_to_all_followers!  (推送给关注者)                        │
│         │                                                                    │
│         └──► deliver_to_lists!          (推送给列表)                          │
│               │                                                              │
│               ▼                                                              │
│  4. @account.lists_for_local_distribution                                   │
│     - 查找包含该账号的所有列表                                                 │
│     - 筛选条件：follow_id 不为空 或 列表所有者自己                             │
│     - 且列表所有者最近登录过                                                   │
│               │                                                              │
│               ▼                                                              │
│  5. FeedInsertWorker.push_bulk(lists)                                       │
│     - 为每个列表创建异步任务                                                   │
│               │                                                              │
│               ▼                                                              │
│  6. FeedInsertWorker.perform                                                 │
│     - @list = List.find(id)                                                  │
│     - @follower = @list.account                                              │
│               │                                                              │
│               ▼                                                              │
│  7. check_and_insert                                                         │
│     ├──► feed_filter = FeedManager.instance.filter(:list, status, list)    │
│     │         │                                                              │
│     │         └──► filter_from_list?  (检查回复策略)                          │
│     │         └──► filter_from_home    (检查屏蔽、静音、语言等)               │
│     │                                                                        │
│     └──► 如果未被过滤                                                        │
│              │                                                               │
│              ▼                                                               │
│  8. perform_push                                                             │
│     FeedManager.instance.push_to_list(list, status)                         │
│              │                                                               │
│              ├──► 1. filter_from_list?  (再次检查)                           │
│              ├──► 2. list.account.user&.signed_in_recently?                │
│              ├──► 3. add_to_feed(:list, list.id, status)                   │
│              │       - Redis ZADD "feed:list:{list_id}"                     │
│              ├──► 4. trim(:list, list.id)  (保持 800 条限制)                │
│              └──► 5. PushUpdateWorker.perform_async                          │
│                       (如果有客户端订阅)                                       │
│              │                                                               │
│              ▼                                                               │
│  9. PushUpdateWorker.perform                                                 │
│     - @timeline_id = "timeline:list:{list_id}"                              │
│     - render_payload!  (使用 StatusCacheHydrator)                            │
│     - publish!                                                                │
│              │                                                               │
│              ▼                                                               │
│  10. redis.publish("timeline:list:{list_id}", message)                      │
│              │                                                               │
│              ▼                                                               │
│  11. Streaming API 服务                                                       │
│      - Redis 订阅收到消息                                                      │
│      - 查找订阅该频道的所有客户端                                               │
│      - 通过 WebSocket/SSE 发送给前端                                          │
│              │                                                               │
│              ▼                                                               │
│  12. 前端 UI 更新                                                              │
│      - 实时显示新状态                                                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 远端账号被暂停时的处理

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      远端账号暂停处理流程                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  方式 A：通过 ActivityPub 接收 Delete/Suspend 消息                           │
│  ─────────────────────────────────────────────                               │
│                                                                              │
│  1. ActivityPub::InboxesController 接收消息                                   │
│         │                                                                    │
│         ▼                                                                    │
│  2. ActivityPub::ProcessingWorker 处理                                       │
│         │                                                                    │
│         ▼                                                                    │
│  3. ActivityPub::Activity::Delete 或 Update                                  │
│         │                                                                    │
│         ▼                                                                    │
│  4. 更新 Account.suspended_at 字段                                           │
│         │                                                                    │
│         ▼                                                                    │
│  5. SuspendAccountService.call(account)                                      │
│                                                                              │
│  ════════════════════════════════════════════════════                        │
│                                                                              │
│  方式 B：本地管理员手动暂停账号                                                │
│  ──────────────────────────────────────                                      │
│                                                                              │
│  1. Admin::Action 创建 suspend 记录                                           │
│         │                                                                    │
│         ▼                                                                    │
│  2. Account.suspended!                                                       │
│         │                                                                    │
│         ▼                                                                    │
│  3. SuspendAccountService.call(account)                                      │
│                                                                              │
│  ════════════════════════════════════════════════                            │
│                                                                              │
│  SuspendAccountService 内部流程：                                             │
│  ───────────────────────────────                                             │
│                                                                              │
│  ├──► reject_remote_follows!  (仅远端账号)                                    │
│  │         │                                                                 │
│  │         └──► 对该账号的每个 Follow：                                        │
│  │              ├──► 发送 RejectFollow 到远端实例                             │
│  │              └──► 销毁 Follow 记录                                         │
│  │                                                                           │
│  ├──► distribute_update_actor!  (仅本地账号)                                  │
│  │         │                                                                 │
│  │         └──► 发送 Update Actor 到所有关注者的实例                           │
│  │                                                                           │
│  ├──► unmerge_from_home_timelines!                                           │
│  │         │                                                                 │
│  │         └──► 从所有关注者的主页时间线移除该账号的状态                        │
│  │                                                                           │
│  └──► unmerge_from_list_timelines!  (关键步骤)                               │
│            │                                                                │
│            ▼                                                                │
│     6. @account.lists_for_local_distribution                                │
│        - 查找包含该账号的所有列表                                              │
│            │                                                                │
│            ▼                                                                │
│     7. 对每个 list：                                                         │
│        FeedManager.instance.unmerge_from_list(@account, list)             │
│            │                                                                │
│            ├──► timeline_key = "feed:list:{list_id}"                       │
│            ├──► 从 Redis 获取时间线中的所有状态 ID                            │
│            └──► 对该账号的每个状态：                                           │
│                 remove_from_feed(:list, list.id, status)                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.3 添加账号到列表的流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        添加账号到列表流程                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. 用户调用 API：POST /api/v1/lists/:list_id/accounts                      │
│     参数：account_ids: [...]                                                 │
│         │                                                                    │
│         ▼                                                                    │
│  2. Api::V1::Lists::AccountsController#create                                │
│         │                                                                    │
│         ▼                                                                    │
│  3. AddAccountsToListService.call(list, accounts)                            │
│         │                                                                    │
│         ├──► update_list!                                                    │
│         │         │                                                          │
│         │         ▼                                                          │
│         │    4. 事务中创建 ListAccount 记录                                   │
│         │         │                                                          │
│         │         └──► before_validation :set_follow                          │
│         │              │                                                      │
│         │              └──► 自动查找 Follow 或 FollowRequest                  │
│         │                   - 如果列表所有者关注了该账号：设置 follow_id        │
│         │                   - 如果只有关注请求：设置 follow_request_id         │
│         │                   - 如果是列表所有者自己：两个都不设置                │
│         │                                                                     │
│         └──► merge_into_list!                                                │
│                   │                                                           │
│                   ▼                                                           │
│            5. merge_account_ids                                               │
│               ListAccount.where(list: @list, account: @accounts)            │
│                 .where.not(follow_id: nil)                                   │
│                 .pluck(:account_id)                                           │
│               │                                                               │
│               └──► 只选择有活跃关注关系的账号                                   │
│                    (follow_id 不为空)                                          │
│                   │                                                           │
│                   ▼                                                           │
│            6. MergeWorker.push_bulk(merge_account_ids)                       │
│               [account_id, list.id, 'list']                                  │
│                   │                                                           │
│                   ▼                                                           │
│            7. MergeWorker.perform                                             │
│               ├──► @from_account = Account.find(from_account_id)              │
│               ├──► @into_list = List.find(into_list_id)                       │
│               └──► FeedManager.instance.merge_into_list(@from_account, @into_list)
│                         │                                                      │
│                         ▼                                                      │
│                    8. 合并历史状态                                              │
│                       ├──► 查询该账号最近的状态                                  │
│                       ├──► 对每个状态：                                          │
│                       │    ├──► 过滤检查                                        │
│                       │    └──► add_to_feed(:list, list.id, status)           │
│                       └──► trim(:list, list.id)                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 7. 关键设计要点总结

### 7.1 列表成员的活跃状态判定

ListAccount 有两个关键字段：
- `follow_id`：关联的 Follow 记录
- `follow_request_id`：关联的 FollowRequest 记录

**只有 `follow_id` 不为空时，该账号才被视为列表的活跃成员：**
- 新状态会被分发到列表时间线
- 添加到列表时会合并历史状态
- 从列表移除时会清理时间线

**这意味着**：
- 只有关注请求的账号：不会出现在列表时间线中
- 列表所有者自己：始终可以添加到列表，但不会触发 fan-out（因为自己的状态会通过其他路径分发）
- 关注被取消后：`follow_id` 变为空，不再参与列表时间线

### 7.2 性能优化策略

1. **lists_for_local_distribution 只选择活跃用户**：
   ```ruby
   .merge(User.signed_in_recently)
   ```
   只为最近登录的用户更新时间线，避免不必要的 Redis 操作。

2. **FeedInsertWorker 异步处理**：
   - 使用 Sidekiq 异步队列
   - 批量处理：`push_bulk`

3. **时间线裁剪**：
   - 限制 800 条（`MAX_ITEMS`）
   - 新状态添加后调用 `trim`

4. **推送更新检查**：
   ```ruby
   PushUpdateWorker.perform_async(...) if push_update_required?("timeline:list:#{list.id}")
   ```
   只有当有客户端订阅时才发送流式更新。

5. **Redis 数据结构**：
   - 使用有序集合（Sorted Set）存储时间线
   - score 使用状态 ID（Snowflake ID，包含时间戳）
   - 高效的范围查询和分页

### 7.3 一致性保证

1. **数据库事务**：
   - 添加/移除列表账号时使用事务
   - ListAccount 有唯一性约束：`validates :account_id, uniqueness: { scope: :list_id }`

2. **Redis 操作**：
   - `add_to_feed` 和 `remove_from_feed` 是原子操作
   - 使用 Redis 管道（pipeline）优化批量操作

3. **状态变化处理**：
   - 账号暂停/恢复时，显式调用 `unmerge_from_list_timelines!` 或 `merge_into_list_timelines!`
   - 确保时间线与账号状态一致

### 7.4 跨实例数据同步

Mastodon 使用 ActivityPub 协议实现跨实例同步：

1. **消息接收**：
   - InboxesController 接收 POST 请求
   - 验证 HTTP 签名
   - 异步处理消息

2. **消息类型处理**：
   - `Follow`：创建关注关系
   - `Undo Follow`：取消关注
   - `Create`：创建状态
   - `Delete`：删除状态或账号
   - `Update`：更新账号信息

3. **状态分发**：
   - 本地状态：直接通过 `FanOutOnWriteService` 分发
   - 远端状态：通过 ActivityPub 接收后，同样通过 `FanOutOnWriteService` 分发

## 8. 相关文件索引

| 功能 | 文件路径 |
|------|----------|
| List 模型 | `app/models/list.rb` |
| ListAccount 模型 | `app/models/list_account.rb` |
| Account 模型 | `app/models/account.rb` |
| Account 交互关系 | `app/models/concerns/account/interactions.rb` |
| 列表时间线 API | `app/controllers/api/v1/timelines/list_controller.rb` |
| 列表管理 API | `app/controllers/api/v1/lists_controller.rb` |
| 列表账号 API | `app/controllers/api/v1/lists/accounts_controller.rb` |
| 添加账号到列表服务 | `app/services/add_accounts_to_list_service.rb` |
| 从列表移除账号服务 | `app/services/remove_accounts_from_list_service.rb` |
| 时间线管理 | `app/lib/feed_manager.rb` |
| 状态分发服务 | `app/services/fan_out_on_write_service.rb` |
| 时间线插入 Worker | `app/workers/feed_insert_worker.rb` |
| 推送更新 Worker | `app/workers/push_update_worker.rb` |
| 合并时间线 Worker | `app/workers/merge_worker.rb` |
| 移除时间线 Worker | `app/workers/unmerge_worker.rb` |
| 暂停账号服务 | `app/services/suspend_account_service.rb` |
| 恢复账号服务 | `app/services/unsuspend_account_service.rb` |
| ActivityPub 入口 | `app/controllers/activitypub/inboxes_controller.rb` |
| ActivityPub 处理 Worker | `app/workers/activitypub/processing_worker.rb` |
| ActivityPub Follow 处理 | `app/lib/activitypub/activity/follow.rb` |
| ActivityPub Undo 处理 | `app/lib/activitypub/activity/undo.rb` |
| Streaming API 服务 | `streaming/index.js` |
