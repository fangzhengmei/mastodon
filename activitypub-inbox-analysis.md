# ActivityPub Inbox 处理流程分析

本文档分析 Mastodon 中远端实例投递 Activity 到 inbox 后的完整处理流程，包括 HTTP 签名验证、Activity 解析、去重和本地副作用处理。

## 一、整体流程概览

```
远端实例 POST Activity → InboxesController#create
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  1. 前置检查                                                             │
│     - skip_large_payload (payload 大小限制 1MB)                           │
│     - skip_unknown_actor_activity (未知 actor 的 Delete/Update)            │
└─────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  2. HTTP 签名验证 (require_actor_signature!)                            │
│     - 解析 Signature/Signature-Input 头                                          │
│     - 从 keyId 获取公钥                                                      │
│     - 验证签名正确性                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  3. 同步处理 (create 动作)                                              │
│     - upgrade_account (升级 OStatus 账户)                                    │
│     - process_collection_synchronization (粉丝同步)                           │
└─────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  4. 异步处理 (process_payload)                                            │
│     - ActivityPub::ProcessingWorker.perform_async                         │
│     - 返回 202 Accepted                                                      │
└─────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  5. 异步 Worker 处理                                                       │
│     - ActivityPub::ProcessCollectionService                                │
│     - ActivityPub::Activity.factory (根据 type 分发)                       │
│     - 去重检查                                                              │
│     - 执行本地副作用                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 二、HTTP 签名验证机制

### 2.1 入口点

签名验证通过 `SignatureVerification` 模块 (`app/controllers/concerns/signature_verification.rb`) 在 `before_action :require_actor_signature!` 中执行。

### 2.2 支持的签名格式

Mastodon 支持两种 HTTP 签名格式：

| 格式 | 检测条件 | 实现类 |
|------|----------|--------|
| **HttpSignature** (旧版) | `Signature` 头存在，`signature-input` 头不存在 | `SignedRequest::HttpSignature` |
| **HttpMessageSignature** (新版) | `signature-input` 头存在 | `SignedRequest::HttpMessageSignature` |

### 2.3 签名验证流程

```ruby
# 位于 app/controllers/concerns/signature_verification.rb:51-85

def signed_request_actor
  # 1. 检查请求是否有签名
  raise Mastodon::SignatureVerificationError, 'Request not signed' unless signed_request?
  
  # 2. 从 keyId 获取密钥对
  keypair = keypair_from_key_id
  
  # 3. 检查密钥有效性（未撤销、未过期）
  check_keypair_validity!(keypair)
  
  # 4. 尝试验证签名
  return (@signed_request_actor = keypair.actor) if signed_request.verified?(keypair)
  
  # 5. 验证失败时，尝试刷新密钥后重新验证
  keypair = stoplight_wrapper.run { keypair_refresh_key!(keypair) }
  # ... 重新验证
end
```

### 2.4 密钥获取逻辑 (`keypair_from_key_id`)

```ruby
# 位于 app/controllers/concerns/signature_verification.rb:94-115

def keypair_from_key_id
  key_id = signed_request.key_id
  domain = key_id.start_with?('acct:') ? key_id.split('@').last : key_id
  
  # 检查域名是否被允许
  if domain_not_allowed?(domain)
    @signature_verification_failure_code = 403
    return
  end
  
  if key_id.start_with?('acct:')
    #  Legacy 格式: acct:user@domain
    stoplight_wrapper.run { fetch_key_from_acct(key_id.delete_prefix('acct:')) }
  elsif !ActivityPub::TagManager.instance.local_uri?(key_id)
    # 远程密钥
    keypair = Keypair.from_keyid(key_id)
    return keypair if keypair.present?
    
    # 本地没有，从远程获取
    stoplight_wrapper.run { ActivityPub::FetchRemoteKeyService.new.call(key_id, suppress_errors: false) }
  end
end
```

### 2.5 签名验证核心 (`SignedRequest#verified?`)

```ruby
# 位于 app/lib/signed_request.rb:246-255

def verified?(keypair)
  # 1. 检查必需参数
  missing_signature_parameters = @signature.missing_signature_parameters
  raise Mastodon::SignatureVerificationError, "..." if missing_signature_parameters
  
  # 2. 检查算法支持 (rsa-sha256, hs2019)
  raise Mastodon::SignatureVerificationError, '...' unless @signature.algorithm_supported?
  
  # 3. 检查时间窗口 (防止重放攻击)
  raise Mastodon::SignatureVerificationError, '...' unless matches_time_window?
  
  # 4. 验证签名强度要求 (必须签名 Date/(created) 和 Digest/(request-target)
  @signature.verify_signature_strength!
  
  # 5. 验证请求体摘要
  @signature.verify_body_digest!
  
  # 6. 最终验证签名
  @signature.verified?(keypair)
end
```

### 2.6 时间窗口检查

```ruby
# 位于 app/lib/signed_request.rb:259-270

def matches_time_window?
  created_time = @signature.created_time
  expires_time = @signature.expires_time
  
  # 默认过期时间：创建时间 + 5分钟
  expires_time ||= created_time + 5.minutes unless created_time.nil?
  # 最大过期时间上限：创建时间 + 12小时
  expires_time = [expires_time, created_time + EXPIRATION_WINDOW_LIMIT].min unless created_time.nil?
  
  # 允许的时钟偏差：1小时
  return false if created_time.present? && created_time > Time.now.utc + CLOCK_SKEW_MARGIN
  return false if expires_time.present? && Time.now.utc > expires_time + CLOCK_SKEW_MARGIN
  
  true
end
```

## 三、Activity 解析流程

### 3.1 异步处理入口

```ruby
# 位于 app/controllers/activitypub/inboxes_controller.rb:75-77

def process_payload
  ActivityPub::ProcessingWorker.perform_async(
    signed_request_actor.id, 
    body, 
    @account&.id, 
    signed_request_actor.class.name
)
end
```

### 3.2 ProcessingWorker

```ruby
# 位于 app/workers/activitypub/processing_worker.rb:3-19

class ActivityPub::ProcessingWorker
  include Sidekiq::Worker
  sidekiq_options queue: 'ingress', backtrace: true, retry: 8
  
  def perform(actor_id, body, delivered_to_account_id = nil, actor_type = 'Account')
    actor = Account.find_by(id: actor_id)
    return if actor.nil?
    
    ActivityPub::ProcessCollectionService.new.call(
      body, 
      actor, 
      override_timestamps: true, 
      delivered_to_account_id: delivered_to_account_id, 
      delivery: true
    )
  end
end
```

### 3.3 ProcessCollectionService

```ruby
# 位于 app/services/activitypub/process_collection_service.rb:3-83

class ActivityPub::ProcessCollectionService < BaseService
  def call(body, actor, **options)
    @account = actor
    @json = original_json = JSON.parse(body)
    @options = options
    
    # 1. JSON-LD 规范化 (如果有签名)
    begin
      @json = compact(@json) if @json['signature'].is_a?(Hash)
    rescue JSON::LD::JsonLdError => e
      @json = original_json.without('signature')
    end
    
    # 2. 前置检查
    return if !supported_context? || (different_actor? && verify_account!.nil?) || suspended_actor? || @account.local?
    return unless @account.is_a?(Account)
    
    # 3. 处理签名转发
    if @json['signature'].present?
      patch_for_forwarding!(original_json, @json)
      @json.delete('signature') unless safe_for_forwarding?(original_json, @json)
    end
    
    # 4. 根据 type 分发处理
    case @json['type']
    when 'Collection', 'CollectionPage'
      process_items @json['items']
    when 'OrderedCollection', 'OrderedCollectionPage'
      process_items @json['orderedItems']
    else
      process_items [@json]
    end
  end
  
  def process_item(item)
    # 工厂模式创建对应的 Activity 处理器
    activity = ActivityPub::Activity.factory(item, @account, **@options)
    activity&.perform
  end
end
```

### 3.4 Activity 工厂模式

```ruby
# 位于 app/lib/activitypub/activity.rb:23-66

class << self
  def factory(json, account, **)
    klass_for(json)&.new(json, account, **)
  end
  
  private
  
  def klass_for(json)
    case json['type']
    when 'Create'      then ActivityPub::Activity::Create
    when 'Announce'    then ActivityPub::Activity::Announce
    when 'Delete'      then ActivityPub::Activity::Delete
    when 'Follow'      then ActivityPub::Activity::Follow
    when 'Like'        then ActivityPub::Activity::Like
    when 'Block'       then ActivityPub::Activity::Block
    when 'Update'      then ActivityPub::Activity::Update
    when 'Undo'        then ActivityPub::Activity::Undo
    when 'Accept'      then ActivityPub::Activity::Accept
    when 'Reject'      then ActivityPub::Activity::Reject
    when 'Flag'        then ActivityPub::Activity::Flag
    when 'Add'         then ActivityPub::Activity::Add
    when 'Remove'      then ActivityPub::Activity::Remove
    when 'Move'        then ActivityPub::Activity::Move
    when 'QuoteRequest' then ActivityPub::Activity::QuoteRequest
    when 'FeatureRequest' then ActivityPub::Activity::FeatureRequest
    end
  end
end
```

## 四、去重机制

Mastodon 使用多层去重机制防止重复处理相同的 Activity。

### 4.1 去重机制总览

| 机制 | 位置 | 用途 |
|------|------|------|
| **URI 查找** | `find_existing_status` | 检查是否已存在相同 URI 的记录 |
| **Tombstone (墓碑)** | `tombstone_exists?` | 检查是否已被删除过 |
| **Delete Upon Arrival** | `delete_arrived_first?` | 处理 Delete 先于 Create 到达的情况 |
| **Redis 分布式锁** | `with_redis_lock` | 防止并发处理相同资源 |

### 4.2 URI 查找 (`find_existing_status`)

```ruby
# 位于 app/lib/activitypub/activity/create.rb:85-89

def find_existing_status
  status = status_from_uri(object_uri)
  status ||= Status.find_by(uri: @object['atomUri']) if @object['atomUri'].present?
  status if status&.account_id == @account.id
end
```

### 4.3 Tombstone 机制

```ruby
# 位于 app/lib/activitypub/activity/create.rb:460-462

def tombstone_exists?
  Tombstone.exists?(uri: object_uri)
end

# 位于 app/lib/activitypub/activity/delete.rb:31
Tombstone.find_or_create_by(uri: object_uri, account: @account)
```

**Tombstone 模型用于记录已删除的对象 URI，防止已删除的对象被重新创建。

### 4.4 Delete Upon Arrival 机制

处理网络延迟导致的 **Delete 先于 Create 到达**的情况。

```ruby
# 位于 app/lib/activitypub/activity.rb:98-104

def delete_arrived_first?(uri)
  redis.exists?("delete_upon_arrival:#{@account.id}:#{uri}")
end

def delete_later!(uri)
  redis.setex("delete_upon_arrival:#{@account.id}:#{uri}", 6.hours.seconds, true)
end
```

**使用场景**：
```ruby
# 位于 app/lib/activitypub/activity/delete.rb:22-29

def delete_object
  # ...
  with_redis_lock("create:#{object_uri}") { delete_later!(object_uri) }
  Tombstone.find_or_create_by(uri: object_uri, account: @account)
  # ...
end

# 位于 app/lib/activitypub/activity/create.rb:20-32

def create_status
  # ...
  with_redis_lock("create:#{object_uri}") do
    Status.uncached do
      # 检查是否已标记为待删除
      return if delete_arrived_first?(object_uri) || poll_vote?
      @status = find_existing_status
    end
    
    if @status.nil?
      process_status
    elsif @options[:delivered_to_account_id].present?
      postprocess_audience_and_deliver
    end
  end
end
```

### 4.5 Redis 分布式锁

使用 `with_redis_lock` 防止并发处理相同资源。

**锁的类型：
- `create:#{object_uri}` - 创建状态时的锁
- `delete_in_progress:#{account_id}` - 删除账户时的锁
- `delete_status_in_progress:#{object_uri}` - 删除状态时的锁
- `vote:#{poll_id}:#{account_id}` - 投票时的锁

```ruby
# 位于 app/lib/activitypub/activity/create.rb:20-32

with_redis_lock("create:#{object_uri}") do
  # 临界区：防止并发创建相同状态
end

# 位于 app/lib/activitypub/activity/delete.rb:14-16

with_redis_lock("delete_in_progress:#{@account.id}", autorelease: 2.hours, raise_on_failure: false) do
  DeleteAccountService.new.call(@account, reserve_username: false, skip_activitypub: true)
end
```

## 五、本地副作用处理

### 5.1 Create (发帖)

```ruby
# 位于 app/lib/activitypub/activity/create.rb:7-13

def perform
  @account.schedule_refresh_if_stale!
  dereference_object!  # 如果 object 是 URI，解引用获取完整内容
  create_status
end
```

**创建状态的完整流程：

```ruby
# 位于 app/lib/activitypub/activity/create.rb:17-75

def create_status
  # 前置检查
  return reject_payload! if unsupported_object_type? || non_matching_uri_hosts?(@account.uri, object_uri) || tombstone_exists? || !related_to_local_activity?
  
  with_redis_lock("create:#{object_uri}") do
    Status.uncached do
      return if delete_arrived_first?(object_uri) || poll_vote?
      @status = find_existing_status  # 去重检查
    end
    
    if @status.nil?
      process_status  # 创建新状态
    elsif @options[:delivered_to_account_id].present?
      postprocess_audience_and_deliver  # 已存在，更新受众
    end
  end
end

def process_status
  # 初始化变量
  @tags = []
  @mentions = []
  @tagged_objects = []
  # ...
  
  # 处理流程：
  process_status_params   # 解析状态参数
  process_tags         # 处理标签 (Hashtag, Mention, Emoji)
  process_quote      # 处理引用
  process_audience    # 处理受众
  
  # 事务创建
  ApplicationRecord.transaction do
    @status = Status.create!(@params.merge(quote: @quote))
    attach_tags(@status)
    attach_tagged_objects(@status)
    attach_mentions(@status)
    attach_counts(@status)
  end
  
  # 后置处理
  resolve_thread(@status)           # 解析回复线程
  resolve_unresolved_mentions(@status)  # 解析未解决的提及
  fetch_replies(@status)          # 获取回复
  fetch_and_verify_quote           # 获取并验证引用
  distribute                       # 分发到时间线
  forward_for_reply                # 转发回复
end
```

### 5.2 Delete (删除)

```ruby
# 位于 app/lib/activitypub/activity/delete.rb:3-9

def perform
  return delete_person if @account.uri == object_uri  # 删除账户
  return delete_feature_authorization! unless !Mastodon::Feature.collections_enabled? || feature_authorization_from_object.nil?
  delete_object  # 删除对象
end
```

**删除账户**：
```ruby
# 位于 app/lib/activitypub/activity/delete.rb:13-17

def delete_person
  with_redis_lock("delete_in_progress:#{@account.id}", autorelease: 2.hours, raise_on_failure: false) do
    DeleteAccountService.new.call(@account, reserve_username: false, skip_activitypub: true)
  end
end
```

**删除对象**：
```ruby
# 位于 app/lib/activitypub/activity/delete.rb:19-43

def delete_object
  return if object_uri.nil?
  
  with_redis_lock("delete_status_in_progress:#{object_uri}", raise_on_failure: false) do
    unless non_matching_uri_hosts?(@account.uri, object_uri)
      # 关键：先获取 create 锁，确保与 Create 操作互斥
      with_redis_lock("create:#{object_uri}") { delete_later!(object_uri) }
      # 创建墓碑记录
      Tombstone.find_or_create_by(uri: object_uri, account: @account)
    end
    
    case @object['type']
    when 'QuoteAuthorization'
      revoke_quote
    when 'Note', 'Question'
      delete_status
    else
      delete_status || revoke_quote
    end
  end
end

def delete_status
  @status = Status.find_by(uri: object_uri, account: @account)
  # ...
  forwarder.forward! if forwarder.forwardable?  # 转发删除活动
  RemoveStatusService.new.call(@status, redraft: false)  # 执行删除
end
```

### 5.3 Follow (关注)

```ruby
# 位于 app/lib/activitypub/activity/follow.rb:3-39

def perform
  target_account = account_from_uri(object_uri)
  
  # 前置检查
  return if target_account.nil? || !target_account.local? || delete_arrived_first?(@json['id'])
  
  # 1. 更新已存在的关注请求 URI
  existing_follow_request = ::FollowRequest.find_by(account: @account, target_account: target_account)
  unless existing_follow_request.nil?
    existing_follow_request.update!(uri: @json['id'])
    return
  end
  
  # 2. 检查是否被屏蔽
  if target_account.blocking?(@account) || target_account.domain_blocking?(@account.domain) || target_account.moved? || target_account.instance_actor?
    reject_follow_request!(target_account)
    return
  end
  
  # 3. 快速转发重复的关注请求
  existing_follow = ::Follow.find_by(account: @account, target_account: target_account)
  unless existing_follow.nil?
    existing_follow.update!(uri: @json['id'])
    AuthorizeFollowService.new.call(@account, target_account, skip_follow_request: true, follow_request_uri: @json['id'])
    return
  end
  
  # 4. 创建新的关注请求
  follow_request = FollowRequest.create!(account: @account, target_account: target_account, uri: @json['id'])
  
  # 5. 根据目标账户设置处理
  if target_account.locked? || @account.silenced?
    # 需要审批：发送关注请求通知
    LocalNotificationWorker.perform_async(target_account.id, follow_request.id, 'FollowRequest', 'follow_request')
  else
    # 自动批准
    AuthorizeFollowService.new.call(@account, target_account)
    LocalNotificationWorker.perform_async(target_account.id, ::Follow.find_by(account: @account, target_account: target_account).id, 'Follow', 'follow')
  end
end
```

## 六、关键流程图总结

### 6.1 完整数据流

```
远端实例
    │
    │ POST /users/:username/inbox 或 /inbox
    │  ┌─────────────────────────────────────────────────────────────────┐
    │  │ InboxesController#create                                            │
    │  │                                                                      │
    │  │ 1. skip_large_payload (>1MB 返回 413)                                │
    │  │ 2. skip_unknown_actor_activity (未知 actor 的 Delete/Update 返回 202)│
    │  │ 3. require_actor_signature! (HTTP 签名验证)                           │
    │  │    ├── 解析 Signature/Signature-Input 头                             │
    │  │    ├── 从 keyId 获取公钥 (本地缓存或远程获取)                         │
    │  │    ├── 验证时间窗口 (防重放)                                          │
    │  │    ├── 验证签名强度 (Date, Digest)                                     │
    │  │    └── 验证签名正确性                                                 │
    │  │                                                                      │
    │  │ 4. upgrade_account (OStatus → ActivityPub)                            │
    │  │ 5. process_collection_synchronization (粉丝同步)                     │
    │  │ 6. process_payload → ProcessingWorker.perform_async                       │
    │  │ 7. 返回 202 Accepted                                                 │
    └──┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Sidekiq Queue: ingress                                                      │
│ ActivityPub::ProcessingWorker                                                │
│                                                                             │
│ 1. 查找 actor (Account.find_by(id: actor_id))                                  │
│ 2. 调用 ProcessCollectionService                                          │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ ActivityPub::ProcessCollectionService                                      │
│                                                                             │
│ 1. JSON.parse(body)                                                        │
│ 2. JSON-LD compact (如果有 signature)                                           │
│ 3. 前置检查:                                                               │
│    - supported_context?                                                    │
│    - different_actor? → verify_account! (LD-Signature 验证)                │
│    - suspended_actor?                                                       │
│    - @account.local?                                                       │
│ 4. 根据 type 分发:                                                          │
│    - Collection/OrderedCollection → 处理 items                           │
│    - 单个 Activity → process_item                                               │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ ActivityPub::Activity.factory (根据 type 创建对应处理器)                      │
│                                                                             │
│ 去重检查:                                                                   │
│ - find_existing_status (URI 查找)                                          │
│ - tombstone_exists? (墓碑记录)                                             │
│ - delete_arrived_first? (Redis 标记)                                        │
│ - with_redis_lock (分布式锁)                                               │
│                                                                             │
│ 本地副作用:                                                                 │
│ - Create: 创建 Status, 处理标签/提及/附件, 分发到时间线                        │
│ - Delete: 创建 Tombstone, 调用 RemoveStatusService                        │
│ - Follow: 创建 FollowRequest 或 Follow, 发送通知                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 关键文件位置

| 功能 | 文件路径 |
|------|----------|
| Inbox 控制器 | `app/controllers/activitypub/inboxes_controller.rb` |
| 签名验证模块 | `app/controllers/concerns/signature_verification.rb` |
| 签名验证实现 | `app/lib/signed_request.rb` |
| 签名解析 | `app/lib/signature_parser.rb` |
| 处理 Worker | `app/workers/activitypub/processing_worker.rb` |
| 集合处理服务 | `app/services/activitypub/process_collection_service.rb` |
| Activity 基类 | `app/lib/activitypub/activity.rb` |
| Create 处理器 | `app/lib/activitypub/activity/create.rb` |
| Delete 处理器 | `app/lib/activitypub/activity/delete.rb` |
| Follow 处理器 | `app/lib/activitypub/activity/follow.rb` |
