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

## 七、异常链路分析

本章详细分析三种异常场景：**验签失败**、**解析失败**、**去重命中**，包括各自的终止条件、返回结果和是否进入后续副作用处理。

### 7.1 异常场景总览

| 异常类型 | 发生阶段 | 终止方式 | HTTP 状态码 | 是否进入 Worker | 是否执行副作用 |
|----------|----------|----------|-------------|-----------------|----------------|
| **验签失败** | 控制器 before_action | 渲染错误响应 | 400/401/403/503 | ❌ 否 | ❌ 否 |
| **解析失败** | Worker 处理阶段 | 静默 return | 已返回 202 | ✅ 已进入 | ❌ 否 |
| **去重命中** | Activity 处理器 | 条件 return/reject_payload! | 已返回 202 | ✅ 已进入 | ⚠️ 部分情况 |

---

### 7.2 验签失败场景

验签发生在 **控制器 before_action** 阶段（`require_actor_signature!`），在 `InboxesController#create` 动作执行之前。

#### 7.2.1 验签失败的各种子场景

| 失败原因 | 错误信息 | HTTP 状态码 | 代码位置 |
|----------|----------|-------------|----------|
| **请求未签名** | `Request not signed` | 401 | `signature_verification.rb:54` |
| **公钥未找到** | `Public key not found for key #{key_id}` | 401 | `signature_verification.rb:58` |
| **密钥已撤销** | `Key #{key_id} is revoked` | 401 | `signature_verification.rb:160` |
| **密钥已过期** | `Key #{key_id} has expired` | 401 | `signature_verification.rb:161` |
| **域名被屏蔽** | 无特殊错误信息 | 403 | `signature_verification.rb:98-100` |
| **签名参数缺失** | `Incompatible request signature. ... are required` | 401 | `signed_request.rb:248` |
| **算法不支持** | `Unsupported signature algorithm ...` | 401 | `signed_request.rb:249` |
| **时间窗口过期** | `Signed request date outside acceptable time window` | 401 | `signed_request.rb:250` |
| **签名强度不足** | 多种信息（Date/Digest/Host 未签名） | 401 | `signed_request.rb:55-58` |
| **摘要不匹配** | `Invalid Digest value. Computed: ... given: ...` | 401 | `signed_request.rb:79,197` |
| **签名验证失败** | `Verification failed for ...` | 401 | `signature_verification.rb:70` |
| **密钥刷新失败** | `Could not refresh public key #{key_id}` | 401 | `signature_verification.rb:65` |
| **网络错误** | `Failed to fetch remote data: ...` | 503 | `signature_verification.rb:77-78` |
| **熔断保护** | `Fetching attempt skipped because of recent connection failure` | 503 | `signature_verification.rb:83-84` |
| **签名头格式错误** | `Error parsing signature parameters` | 400 | `signed_request.rb:91` |
| **重复签名参数** | `Error parsing signature with duplicate keys` | 401 | `signature_parser.rb:30` |

#### 7.2.2 验签失败的终止条件

```ruby
# 位于 app/controllers/concerns/signature_verification.rb:19-21

def require_actor_signature!
  render json: signature_verification_failure_reason, status: signature_verification_failure_code unless signed_request_actor
end
```

**终止条件**：`signed_request_actor` 返回 `nil`

#### 7.2.3 验签失败的完整流程

```
远端实例 POST /inbox
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ before_action :require_actor_signature!                      │
│                                                              │
│ 1. signed_request? 检查是否有 Signature 头                    │
│    └── 无签名头 → 401 Unauthorized                          │
│                                                              │
│ 2. keypair_from_key_id 获取公钥                              │
│    ├── 域名被屏蔽 → 403 Forbidden                           │
│    ├── 本地无缓存 → 远程获取                                 │
│    └── 获取失败 → 401/503                                   │
│                                                              │
│ 3. check_keypair_validity! 检查密钥状态                      │
│    ├── 已撤销 → 401                                         │
│    └── 已过期 → 401                                         │
│                                                              │
│ 4. signed_request.verified?(keypair) 验证签名               │
│    ├── 参数缺失 → 401                                       │
│    ├── 算法不支持 → 401                                     │
│    ├── 时间窗口过期 → 401                                   │
│    ├── 签名强度不足 → 401                                   │
│    ├── 摘要不匹配 → 401                                     │
│    └── 签名验证失败 → 尝试刷新密钥后重试                     │
└─────────────────────────────────────────────────────────────┘
         │
         ├── 验签成功 → 继续执行 create 动作 → process_payload → 进入 Worker
         │
         └── 验签失败 → render json: { error: ... }, status: 4xx/503
                      → ❌ 不进入后续流程
                      → ❌ 不执行 process_payload
                      → ❌ 不进入 Worker
                      → ❌ 不执行任何副作用
```

#### 7.2.4 验签失败的返回结果

```ruby
# 位于 app/controllers/concerns/signature_verification.rb:87-92

def fail_with!(message, **options)
  Rails.logger.debug { "Signature verification failed: #{message}" }
  
  @signature_verification_failure_reason = { error: message }.merge(options)
  @signed_request_actor = nil
end
```

**返回格式**：
```json
{
  "error": "具体错误信息"
}
```

**状态码**：
- `400`：签名头格式错误（`MalformedHeaderError`）
- `401`：签名验证失败（默认）
- `403`：域名被屏蔽
- `503`：网络错误或熔断保护

#### 7.2.5 验签失败的副作用影响

| 流程步骤 | 是否执行 |
|----------|----------|
| `upgrade_account` | ❌ 否 |
| `process_collection_synchronization` | ❌ 否 |
| `process_payload` (入队 Worker) | ❌ 否 |
| `ActivityPub::ProcessingWorker` | ❌ 否 |
| 本地副作用处理 | ❌ 否 |

---

### 7.3 解析失败场景

解析失败发生在 **Worker 处理阶段**，此时控制器已返回 `202 Accepted`。

#### 7.3.1 解析失败的各种子场景

| 失败原因 | 发生位置 | 处理方式 | 是否记录日志 |
|----------|----------|----------|--------------|
| **JSON 解析失败** | `ProcessCollectionService#call` | `rescue JSON::ParserError` → `return nil` | 否 |
| **JSON-LD compact 失败** | `ProcessCollectionService#call` | 移除 signature 后继续 | `Rails.logger.debug` |
| **不支持的 JSON-LD Context** | `ProcessCollectionService#call` | 直接 `return` | 否 |
| **Actor 不匹配且 LD 签名验证失败** | `ProcessCollectionService#call` | 直接 `return` | `Rails.logger.debug` |
| **Actor 已被暂停** | `ProcessCollectionService#call` | 直接 `return` | 否 |
| **Actor 是本地账户** | `ProcessCollectionService#call` | 直接 `return` | 否 |
| **未知的 Activity type** | `Activity.factory` | 返回 `nil`，`activity&.perform` 不执行 | 否 |

#### 7.3.2 解析失败的终止条件分析

**场景 1：JSON 解析失败**

```ruby
# 位于 app/services/activitypub/process_collection_service.rb:41-43

rescue JSON::ParserError
  nil
end
```

- **触发条件**：`JSON.parse(body)` 抛出 `JSON::ParserError`
- **终止方式**：`rescue` 块返回 `nil`
- **影响**：整个 `call` 方法提前结束

**场景 2：JSON-LD Context 不支持**

```ruby
# 位于 app/services/activitypub/process_collection_service.rb:26

return if !supported_context? || (different_actor? && verify_account!.nil?) || suspended_actor? || @account.local?
```

- **触发条件**：`supported_context?` 返回 `false`
- **终止方式**：条件 `return`
- **影响**：不处理 activity

**场景 3：Actor 不匹配且 LD 签名验证失败**

```ruby
# 位于 app/services/activitypub/process_collection_service.rb:72-83

def verify_account!
  return unless @json['signature'].is_a?(Hash)
  return if domain_not_allowed?(@json['signature']['creator'])
  
  @options[:relayed_through_actor] = @account
  @account = ActivityPub::LinkedDataSignature.new(@json).verify_actor!
  @account = nil unless @account.is_a?(Account)
  @account
rescue JSON::LD::JsonLdError, RDF::WriterError => e
  Rails.logger.debug { "Could not verify LD-Signature for #{value_or_id(@json['actor'])}: #{e.message}" }
  nil
end
```

- **触发条件**：`@json['actor'] != @account.uri` 且 `verify_account!` 返回 `nil`
- **终止方式**：条件 `return`
- **影响**：不处理 activity

**场景 4：未知的 Activity type**

```ruby
# 位于 app/lib/activitypub/activity.rb:30-65

def klass_for(json)
  case json['type']
  when 'Create'      then ActivityPub::Activity::Create
  when 'Announce'    then ActivityPub::Activity::Announce
  # ... 其他 type
  # 没有 else 分支，未知 type 返回 nil
  end
end

# 位于 app/services/activitypub/process_collection_service.rb:246-250

def process_item(item)
  activity = ActivityPub::Activity.factory(item, @account, **@options)
  activity&.perform  # &. 安全导航，activity 为 nil 时不执行
end
```

- **触发条件**：`json['type']` 不在支持列表中
- **终止方式**：`klass_for` 返回 `nil`，`activity&.perform` 不执行
- **影响**：`perform` 方法不被调用

#### 7.3.3 解析失败的完整流程

```
控制器已返回 202 Accepted
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ ActivityPub::ProcessingWorker#perform                        │
│                                                              │
│ 1. Account.find_by(id: actor_id)                            │
│    └── actor 不存在 → return                                │
│                                                              │
│ 2. ProcessCollectionService.new.call(body, actor, ...)     │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ ProcessCollectionService#call                                │
│                                                              │
│ 1. JSON.parse(body)                                          │
│    └── 解析失败 → rescue JSON::ParserError → return nil    │
│                                                              │
│ 2. JSON-LD compact (如果有 signature)                        │
│    └── 失败 → 移除 signature 后继续                         │
│                                                              │
│ 3. 前置检查 return unless ...                                │
│    ├── !supported_context? → return                         │
│    ├── different_actor? && verify_account!.nil? → return   │
│    ├── suspended_actor? → return                            │
│    └── @account.local? → return                             │
│                                                              │
│ 4. process_item(item)                                        │
│    └── Activity.factory → 未知 type 返回 nil                │
│         └── activity&.perform 不执行                        │
└─────────────────────────────────────────────────────────────┘
```

#### 7.3.4 解析失败的返回结果

由于解析失败发生在 **异步 Worker** 中，控制器已提前返回 `202 Accepted`，因此：

- **HTTP 响应**：`202 Accepted`（控制器已返回）
- **Worker 行为**：静默 `return`，不抛出异常
- **日志记录**：
  - JSON-LD compact 失败：`Rails.logger.debug`
  - LD-Signature 验证失败：`Rails.logger.debug`
  - 其他解析失败：无日志

#### 7.3.5 解析失败的副作用影响

| 流程步骤 | 是否执行 |
|----------|----------|
| 控制器 `create` 动作 | ✅ 已执行（返回 202） |
| `ProcessingWorker` 入队 | ✅ 已入队 |
| `ProcessCollectionService` 调用 | ✅ 已调用 |
| `Activity.factory` 创建 | ⚠️ 部分情况（JSON 解析成功才执行） |
| `activity.perform` 执行 | ❌ 否 |
| 本地副作用处理 | ❌ 否 |

---

### 7.4 去重命中场景

去重命中发生在 **Activity 处理器的 `perform` 方法** 中，此时已通过验签和基本解析。

#### 7.4.1 去重机制回顾

Mastodon 使用四层去重机制：

| 机制 | 检查方法 | 用途 |
|------|----------|------|
| **URI 查找** | `find_existing_status` | 检查是否已存在相同 URI 的记录 |
| **Tombstone (墓碑)** | `tombstone_exists?` | 检查是否已被删除过 |
| **Delete Upon Arrival** | `delete_arrived_first?` | 处理 Delete 先于 Create 到达的情况 |
| **Redis 分布式锁** | `with_redis_lock` | 防止并发处理相同资源 |

#### 7.4.2 各 Activity 类型的去重命中分析

##### 7.4.2.1 Create (发帖) 的去重命中

```ruby
# 位于 app/lib/activitypub/activity/create.rb:17-35

def create_status
  # 第 1 层：前置检查（去重相关）
  return reject_payload! if unsupported_object_type? || non_matching_uri_hosts?(@account.uri, object_uri) || tombstone_exists? || !related_to_local_activity?
  
  with_redis_lock("create:#{object_uri}") do
    Status.uncached do
      # 第 2 层：Delete Upon Arrival 检查
      return if delete_arrived_first?(object_uri) || poll_vote?
      # 第 3 层：URI 查找
      @status = find_existing_status
    end
    
    if @status.nil?
      process_status  # 创建新状态
    elsif @options[:delivered_to_account_id].present?
      postprocess_audience_and_deliver  # 已存在，更新受众
    end
  end
end
```

**Create 去重命中场景**：

| 去重检查 | 命中条件 | 终止方式 | 是否执行副作用 |
|----------|----------|----------|----------------|
| `tombstone_exists?` | URI 存在于 Tombstone 表 | `reject_payload!` | ❌ 否 |
| `delete_arrived_first?` | Redis 有 `delete_upon_arrival` 标记 | 直接 `return` | ❌ 否 |
| `find_existing_status` | 已存在相同 URI 的 Status | 不 `return` | ⚠️ 部分执行 |

**特殊情况：Status 已存在但有 delivered_to_account_id**

```ruby
# 位于 app/lib/activitypub/activity/create.rb:156-165

def postprocess_audience_and_deliver
  return if @status.mentions.find_by(account_id: @options[:delivered_to_account_id])
  
  # 添加新的提及
  @status.mentions.create(account: delivered_to_account, silent: true)
  @status.update(visibility: :limited) if @status.direct_visibility?
  
  # 如果收件人已关注作者，插入到时间线
  return unless delivered_to_account.following?(@account)
  
  FeedInsertWorker.perform_async(@status.id, delivered_to_account.id, 'home')
end
```

**这种情况下的副作用**：
- ❌ 不创建新的 Status 记录
- ✅ 可能添加新的 Mention 记录
- ✅ 可能更新 Status 的 visibility
- ✅ 可能插入到收件人的时间线

##### 7.4.2.2 Delete (删除) 的去重命中

```ruby
# 位于 app/lib/activitypub/activity/delete.rb:19-43

def delete_object
  return if object_uri.nil?  # 去重：URI 为空直接返回
  
  with_redis_lock("delete_status_in_progress:#{object_uri}", raise_on_failure: false) do
    unless non_matching_uri_hosts?(@account.uri, object_uri)
      # 标记：即使对象不存在，也要创建 Tombstone 和 delete_upon_arrival
      with_redis_lock("create:#{object_uri}") { delete_later!(object_uri) }
      Tombstone.find_or_create_by(uri: object_uri, account: @account)
    end
    
    # 实际删除
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

# 位于 app/lib/activitypub/activity/delete.rb:45-55

def delete_status
  @status = Status.find_by(uri: object_uri, account: @account)
  @status ||= Status.find_by(uri: @object['atomUri'], account: @account) if @object.is_a?(Hash) && @object['atomUri'].present?
  
  return if @status.nil?  # 去重：状态不存在直接返回
  
  forwarder.forward! if forwarder.forwardable?
  RemoveStatusService.new.call(@status, redraft: false)
  
  true
end
```

**Delete 去重命中场景**：

| 去重检查 | 命中条件 | 终止方式 | 是否执行副作用 |
|----------|----------|----------|----------------|
| `object_uri.nil?` | URI 为空 | 直接 `return` | ❌ 否 |
| `non_matching_uri_hosts?` | URI 域名与 actor 不匹配 | 不创建 Tombstone | ⚠️ 部分 |
| `Status.find_by` 返回 nil | 状态不存在 | `delete_status` 内 `return` | ⚠️ 部分 |

**Delete 的特殊设计**：

```ruby
# 即使 Status 不存在，也会执行：
with_redis_lock("create:#{object_uri}") { delete_later!(object_uri) }  # Redis 标记
Tombstone.find_or_create_by(uri: object_uri, account: @account)        # 墓碑记录
```

**这种设计的目的**：防止后续迟到的 Create 操作重建已删除的对象。

**Delete 去重命中后的副作用**：

| 情况 | 副作用 |
|------|--------|
| `object_uri.nil?` | ❌ 无任何副作用 |
| URI 域名不匹配 | ❌ 不创建 Tombstone，不执行删除 |
| Status 存在 | ✅ 转发删除活动 ✅ 调用 RemoveStatusService |
| Status 不存在 | ✅ 创建 Tombstone ✅ 设置 delete_upon_arrival 标记 |

##### 7.4.2.3 Follow (关注) 的去重命中

```ruby
# 位于 app/lib/activitypub/activity/follow.rb:3-39

def perform
  target_account = account_from_uri(object_uri)
  
  # 前置检查
  return if target_account.nil? || !target_account.local? || delete_arrived_first?(@json['id'])
  
  # 第 1 层去重：已存在 FollowRequest
  existing_follow_request = ::FollowRequest.find_by(account: @account, target_account: target_account)
  unless existing_follow_request.nil?
    existing_follow_request.update!(uri: @json['id'])
    return  # 直接返回
  end
  
  # 被屏蔽检查
  if target_account.blocking?(@account) || target_account.domain_blocking?(@account.domain) || target_account.moved? || target_account.instance_actor?
    reject_follow_request!(target_account)
    return
  end
  
  # 第 2 层去重：已存在 Follow 关系
  existing_follow = ::Follow.find_by(account: @account, target_account: target_account)
  unless existing_follow.nil?
    existing_follow.update!(uri: @json['id'])
    AuthorizeFollowService.new.call(@account, target_account, skip_follow_request: true, follow_request_uri: @json['id'])
    return  # 直接返回
  end
  
  # 创建新的关注请求
  follow_request = FollowRequest.create!(account: @account, target_account: target_account, uri: @json['id'])
  # ... 后续处理
end
```

**Follow 去重命中场景**：

| 去重检查 | 命中条件 | 终止方式 | 是否执行副作用 |
|----------|----------|----------|----------------|
| `delete_arrived_first?(@json['id'])` | Activity URI 被标记为删除 | 直接 `return` | ❌ 否 |
| `existing_follow_request` 存在 | 已存在相同的 FollowRequest | 更新 URI 后 `return` | ⚠️ 部分 |
| `existing_follow` 存在 | 已存在 Follow 关系 | 更新 URI + 调用 AuthorizeFollowService | ⚠️ 部分 |

**Follow 去重命中后的副作用**：

| 情况 | 副作用 |
|------|--------|
| `delete_arrived_first?` | ❌ 无任何副作用 |
| FollowRequest 已存在 | ✅ 更新 FollowRequest.uri ❌ 不创建新记录 |
| Follow 已存在 | ✅ 更新 Follow.uri ✅ 调用 AuthorizeFollowService |

##### 7.4.2.4 Announce (转发) 的去重命中

```ruby
# 位于 app/lib/activitypub/activity/announce.rb:3-33

def perform
  # 前置检查
  return reject_payload! if delete_arrived_first?(@json['id']) || !related_to_local_activity?
  return reject_payload! if @object.nil?
  
  with_redis_lock("announce:#{value_or_id(@object)}") do
    original_status = status_from_object
    
    return reject_payload! if original_status.nil? || !announceable?(original_status)
    return if requested_through_relay?  # 去重：通过中继接收的重复转发
    
    # 去重检查：已存在相同的转发
    @status = Status.find_by(account: @account, reblog: original_status)
    
    return @status unless @status.nil?  # 已存在则直接返回
    
    # 创建新的转发
    @status = Status.create!(...)
    # ... 后续处理
  end
end
```

**Announce 去重命中场景**：

| 去重检查 | 命中条件 | 终止方式 | 是否执行副作用 |
|----------|----------|----------|----------------|
| `delete_arrived_first?(@json['id'])` | Activity URI 被标记 | `reject_payload!` | ❌ 否 |
| `requested_through_relay?` | 通过中继接收 | 直接 `return` | ❌ 否 |
| `Status.find_by(account: @account, reblog: original_status)` | 已存在相同转发 | 直接 `return @status` | ❌ 否 |

##### 7.4.2.5 Undo (撤销) 的去重命中

```ruby
# 位于 app/lib/activitypub/activity/undo.rb

def perform
  # Undo 的 object 是另一个 Activity
  @object = @json['object']
  
  case @object['type']
  when 'Follow'
    undo_follow
  when 'Announce'
    undo_announce
  when 'Like'
    undo_like
  when 'Block'
    undo_block
  # ...
  end
end

def undo_follow
  target_account = account_from_uri(object_uri)
  return if target_account.nil?
  
  # 去重：查找并删除 FollowRequest 或 Follow
  follow_request = FollowRequest.find_by(account: @account, target_account: target_account)
  if follow_request
    follow_request.destroy
    return
  end
  
  follow = Follow.find_by(account: @account, target_account: target_account)
  if follow
    UnfollowService.new.call(@account, target_account, skip_unfollow: true)
  end
end
```

**Undo 去重命中场景**：

| 去重检查 | 命中条件 | 终止方式 | 是否执行副作用 |
|----------|----------|----------|----------------|
| `FollowRequest.find_by` 存在 | 待撤销的关注请求存在 | 销毁后 `return` | ✅ 执行销毁 |
| `Follow.find_by` 存在 | 待撤销的关注关系存在 | 调用 UnfollowService | ✅ 执行撤销 |
| 两者都不存在 | 没有可撤销的记录 | 静默结束 | ❌ 无副作用 |

#### 7.4.3 去重命中的返回结果

由于去重命中发生在 **异步 Worker** 中：

- **HTTP 响应**：`202 Accepted`（控制器已返回）
- **Worker 行为**：
  - 大多数情况：静默 `return`
  - 部分情况：调用 `reject_payload!`（记录日志后返回 `nil`）
- **日志记录**：
  - `reject_payload!` 会记录 info 日志：`Rails.logger.info("Rejected #{@json['type']} activity ...")`

```ruby
# 位于 app/lib/activitypub/activity.rb:202-205

def reject_payload!
  Rails.logger.info("Rejected #{@json['type']} activity #{@json['id']} from #{@account.uri}#{@options[:relayed_through_actor] && "via #{@options[:relayed_through_actor].uri}"}")
  nil
end
```

#### 7.4.4 去重命中的副作用影响总结

| Activity 类型 | 去重场景 | 副作用执行情况 |
|---------------|----------|----------------|
| **Create** | Tombstone 存在 | ❌ 无 |
| **Create** | Delete Upon Arrival 标记 | ❌ 无 |
| **Create** | Status 已存在（无 delivered_to） | ❌ 无 |
| **Create** | Status 已存在（有 delivered_to） | ✅ 更新受众 ✅ 可能插入时间线 |
| **Delete** | object_uri 为空 | ❌ 无 |
| **Delete** | Status 不存在 | ✅ 创建 Tombstone ✅ Redis 标记 |
| **Delete** | Status 存在 | ✅ 完整删除流程 |
| **Follow** | Delete Upon Arrival 标记 | ❌ 无 |
| **Follow** | FollowRequest 已存在 | ✅ 仅更新 URI |
| **Follow** | Follow 已存在 | ✅ 更新 URI ✅ 调用 AuthorizeFollowService |
| **Announce** | Delete Upon Arrival 标记 | ❌ 无 |
| **Announce** | 已存在相同转发 | ❌ 无 |
| **Undo** | 目标记录存在 | ✅ 执行撤销 |
| **Undo** | 目标记录不存在 | ❌ 无 |

---

### 7.5 异常链路决策树

```
远端实例 POST Activity
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 阶段 1: 控制器 before_action (验签)                           │
└─────────────────────────────────────────────────────────────┘
         │
         ├── 验签失败 ───────────────────────────────────────┐
         │                                                    │
         │    HTTP 响应: 400/401/403/503                   │
         │    响应体: { "error": "具体错误信息" }            │
         │    后续流程: ❌ 全部终止                           │
         │    副作用: ❌ 无                                   │
         │                                                    │
         └────────────────────────────────────────────────────┘
         │
         ▼ 验签成功
┌─────────────────────────────────────────────────────────────┐
│ 阶段 2: 控制器 create 动作                                    │
│  - upgrade_account                                           │
│  - process_collection_synchronization                        │
│  - process_payload (入队 Worker)                             │
└─────────────────────────────────────────────────────────────┘
         │
         ▼ 返回 202 Accepted
┌─────────────────────────────────────────────────────────────┐
│ 阶段 3: 异步 Worker 处理                                      │
│ ActivityPub::ProcessingWorker                                │
└─────────────────────────────────────────────────────────────┘
         │
         ├── Actor 不存在 ───────────────────────────────────┐
         │                                                    │
         │    Worker: 静默 return                             │
         │    副作用: ❌ 无                                   │
         │                                                    │
         └────────────────────────────────────────────────────┘
         │
         ▼ Actor 存在
┌─────────────────────────────────────────────────────────────┐
│ 阶段 4: ProcessCollectionService                             │
│  - JSON.parse                                                │
│  - JSON-LD compact                                           │
│  - 前置检查 (supported_context?, different_actor?, etc.)     │
└─────────────────────────────────────────────────────────────┘
         │
         ├── 解析失败 ───────────────────────────────────────┐
         │  (JSON 错误 / 不支持的 Context / 未知 type 等)     │
         │                                                    │
         │    Worker: 静默 return                             │
         │    日志: 部分情况有 debug 日志                      │
         │    副作用: ❌ 无                                   │
         │                                                    │
         └────────────────────────────────────────────────────┘
         │
         ▼ 解析成功
┌─────────────────────────────────────────────────────────────┐
│ 阶段 5: Activity 处理器 (根据 type 分发)                       │
│  - 去重检查 (URI 查找 / Tombstone / Delete Upon Arrival)     │
│  - 本地副作用处理                                             │
└─────────────────────────────────────────────────────────────┘
         │
         ├── 去重命中 ───────────────────────────────────────┐
         │                                                    │
         │    ┌──────────────────────────────────────────┐   │
         │    │ 去重类型不同，副作用执行情况不同:          │   │
         │    │                                          │   │
         │    │ ❌ 完全跳过:                             │   │
         │    │    - Tombstone 存在                      │   │
         │    │    - Delete Upon Arrival 标记            │   │
         │    │    - 目标记录完全不存在                   │   │
         │    │                                          │   │
         │    │ ⚠️ 部分执行:                             │   │
         │    │    - Create: 已存在但更新受众           │   │
         │    │    - Delete: 状态不存在但创建 Tombstone  │   │
         │    │    - Follow: 已存在但更新 URI            │   │
         │    │    - Undo: 目标存在则执行撤销            │   │
         │    └──────────────────────────────────────────┘   │
         │                                                    │
         └────────────────────────────────────────────────────┘
         │
         ▼ 去重未命中
┌─────────────────────────────────────────────────────────────┐
│ 阶段 6: 执行完整本地副作用                                     │
│  - Create: 创建 Status + 时间线分发                          │
│  - Delete: 执行删除 + 创建 Tombstone                          │
│  - Follow: 创建 FollowRequest / Follow                        │
│  - ... 其他 activity                                          │
└─────────────────────────────────────────────────────────────┘
```

---

### 7.6 异常场景对比表

| 维度 | 验签失败 | 解析失败 | 去重命中 |
|------|----------|----------|----------|
| **发生阶段** | 控制器 before_action | Worker ProcessCollectionService | Worker Activity 处理器 |
| **HTTP 状态码** | 400/401/403/503 | 202 Accepted | 202 Accepted |
| **响应体** | `{ "error": "..." }` | 无（异步） | 无（异步） |
| **是否进入 Worker** | ❌ 否 | ✅ 是 | ✅ 是 |
| **是否执行副作用** | ❌ 否 | ❌ 否 | ⚠️ 视情况而定 |
| **是否记录日志** | ✅ Rails.logger.debug | ⚠️ 部分情况 | ⚠️ reject_payload! 时 |
| **重试机制** | 无（客户端重试） | Sidekiq retry: 8 | Sidekiq retry: 8 |
| **数据一致性** | 无影响 | 无影响 | 保证不重复处理 |

---

### 7.7 关键设计理念

1. **验签失败快速失败**：在控制器层面直接返回错误，避免无效请求进入队列
2. **异步解耦**：控制器返回 202 后，实际处理在 Worker 中进行，提高吞吐量
3. **多层去重**：URI 查找 + Tombstone + Delete Upon Arrival + 分布式锁，确保幂等性
4. **宽容处理**：解析失败和部分去重场景静默处理，不抛出异常，避免 Worker 无限重试
5. **部分更新**：某些去重场景（如 Create 更新受众、Follow 更新 URI）允许部分副作用，保证数据最终一致性

---

## 八、重试行为分析

本章详细分析异常链路中的重试行为：哪些分支会触发 Sidekiq 自动重试、哪些只会静默结束，以及各自依赖的异常触发条件。

### 8.1 Sidekiq 重试机制概述

#### 8.1.1 基础配置

```ruby
# 位于 app/workers/activitypub/processing_worker.rb:6

sidekiq_options queue: 'ingress', backtrace: true, retry: 8
```

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `queue` | `'ingress'` | 使用 ingress 队列，优先级 4 |
| `backtrace` | `true` | 记录异常回溯日志 |
| `retry` | `8` | 最多重试 8 次 |

#### 8.1.2 重试触发的核心规则

**Sidekiq 只有在以下情况才会触发自动重试：**

1. **异常未被捕获**：异常向上冒泡到 Sidekiq 框架
2. **异常被重新抛出**：使用 `raise` 或 `raise e` 重新抛出捕获的异常

**不会触发重试的情况：**

1. **异常被捕获且不重新抛出**：`rescue` 块中没有 `raise`
2. **条件 `return`**：正常控制流，不是异常
3. **`retry: 0` 或 `retry: false`**：显式禁用重试

#### 8.1.3 指数退避策略

Sidekiq 使用指数退避算法计算重试延迟：

```
重试次数 n 的延迟 ≈ 15^n 秒
```

| 重试次数 | 延迟（约） |
|----------|------------|
| 第 1 次 | 15 秒 |
| 第 2 次 | 4 分钟 |
| 第 3 次 | 56 分钟 |
| 第 4 次 | 14 小时 |
| ... | ... |
| 第 8 次 | 约 3 天 |

重试 8 次后，如果仍失败，任务会被移到 **Dead Job**（死信）队列。

---

### 8.2 会触发 Sidekiq 自动重试的分支

#### 8.2.1 ProcessingWorker 中未被捕获的异常

```ruby
# 位于 app/workers/activitypub/processing_worker.rb:8-19

def perform(actor_id, body, delivered_to_account_id = nil, actor_type = 'Account')
  case actor_type
  when 'Account'
    actor = Account.find_by(id: actor_id)
  end

  return if actor.nil?

  ActivityPub::ProcessCollectionService.new.call(body, actor, override_timestamps: true, delivered_to_account_id: delivered_to_account_id, delivery: true)
rescue ActiveRecord::RecordInvalid => e  # ⚠️ 只捕获 RecordInvalid
  Rails.logger.debug { "Error processing incoming ActivityPub object: #{e}" }
end
```

**关键点**：只捕获了 `ActiveRecord::RecordInvalid`，其他所有异常都会向上冒泡，触发 Sidekiq 重试。

#### 8.2.2 触发重试的异常类型

| 异常类型 | 触发条件 | 代码位置 |
|----------|----------|----------|
| **数据库连接异常** | 数据库暂时不可用 | `ActiveRecord::StatementInvalid` 等 |
| **乐观锁冲突** | 并发更新同一记录 | `ActiveRecord::StaleObjectError`（部分场景） |
| **Redis 异常** | Redis 连接/命令错误 | `Redis::BaseError` 子类 |
| **网络异常（部分场景）** | 远程请求失败且未被捕获 | `HTTP::Error` 等 |
| **其他未捕获的 StandardError** | 任何未被 rescue 的异常 | 多处 |

#### 8.2.3 Activity::Move 的特殊处理（捕获后重新抛出）

```ruby
# 位于 app/lib/activitypub/activity/move.rb:6-25

def perform
  return if origin_account.uri != object_uri
  return unless mark_as_processing!

  target_account = ActivityPub::FetchRemoteAccountService.new.call(target_uri)

  if target_account.nil? || target_account.unavailable? || !target_account.also_known_as.include?(origin_account.uri)
    unmark_as_processing!
    return
  end

  origin_account.update(moved_to_account: target_account)
  MoveWorker.perform_async(origin_account.id, target_account.id)
rescue                    # ⚠️ 捕获所有异常
  unmark_as_processing!   # 先清理 Redis 状态标记
  raise                   # ⚠️ 重新抛出，触发 Sidekiq 重试！
end
```

**设计意图**：
1. **状态清理**：确保 Redis 中的 `move_in_progress` 标记被清除
2. **允许重试**：重新抛出异常，让 Sidekiq 可以重试
3. **幂等性保障**：`mark_as_processing!` 使用 `nx: true`，确保并发安全

#### 8.2.4 重试决策树

```
异常发生
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 异常是否被捕获？                                               │
└─────────────────────────────────────────────────────────────┘
    │
    ├── 否 ─────────────────────────────────────────────────┐
    │                                                        │
    │    ▼ 触发 Sidekiq 重试                                │
    │                                                        │
    └────────────────────────────────────────────────────────┘
    │
    └── 是 ─────────────────────────────────────────────────┐
         │                                                   │
         ▼                                                   │
    ┌─────────────────────────────────────────────────┐     │
    │ rescue 块中是否有 raise？                        │     │
    └─────────────────────────────────────────────────┘     │
         │                                                   │
         ├── 是 ──────────────────────────────────────────┐ │
         │                                                │ │
         │    ▼ 触发 Sidekiq 重试（Activity::Move 模式）│ │
         │                                                │ │
         └────────────────────────────────────────────────┘ │
         │                                                   │
         └── 否 ──────────────────────────────────────────┐ │
              │                                            │ │
              ▼                                            │ │
         ┌─────────────────────────────────────────────┐   │ │
         │ 静默结束，不触发重试                          │   │ │
         │ - 记录日志（部分情况）                        │   │ │
         │ - 可能执行部分副作用（如加入延迟队列）         │   │ │
         └─────────────────────────────────────────────┘   │ │
              │                                            │ │
              └────────────────────────────────────────────┘ │
                   │                                          │
                   └──────────────────────────────────────────┘
```

---

### 8.3 只会静默结束的分支（不触发重试）

#### 8.3.1 被捕获且不重新抛出的异常

##### 8.3.1.1 ProcessingWorker 中的 RecordInvalid

```ruby
# 位于 app/workers/activitypub/processing_worker.rb:17-19

rescue ActiveRecord::RecordInvalid => e
  Rails.logger.debug { "Error processing incoming ActivityPub object: #{e}" }
  # ⚠️ 没有 raise，静默结束
end
```

**设计意图**：
- `RecordInvalid` 通常是数据验证错误（如必填字段缺失、格式错误）
- 这类错误重试也无法解决，所以直接静默结束

##### 8.3.1.2 ProcessCollectionService 中的解析错误

```ruby
# 位于 app/services/activitypub/process_collection_service.rb:41-43

rescue JSON::ParserError
  nil  # ⚠️ 静默返回 nil
end

# 位于 app/services/activitypub/process_collection_service.rb:14-19

begin
  @json = compact(@json) if @json['signature'].is_a?(Hash)
rescue JSON::LD::JsonLdError => e
  Rails.logger.debug { "Error when compacting JSON-LD document for #{value_or_id(@json['actor'])}: #{e.message}" }
  @json = original_json.without('signature')  # ⚠️ 移除 signature 后继续，不重试
end
```

##### 8.3.1.3 Activity::Create 中的局部异常捕获

| 方法 | 捕获的异常 | 处理方式 | 代码位置 |
|------|------------|----------|----------|
| `process_hashtag` | `ActiveRecord::RecordInvalid` | 静默 return | `create.rb:247-248` |
| `process_mention` | `Mastodon::UnexpectedResponseError`, `HTTP_CONNECTION_ERRORS` | 加入 `@unresolved_mentions`，后续由 `MentionResolveWorker` 处理 | `create.rb:260-261` |
| `process_emoji` | `Seahorse::Client::NetworkingError` | 记录 warn 日志 | `create.rb:279-280` |
| `process_attachments` | `Mastodon::UnexpectedResponseError`, `HTTP_CONNECTION_ERRORS` | 入队 `RedownloadMediaWorker` 延迟处理 | `create.rb:320-321` |
| `process_attachments` | `Seahorse::Client::NetworkingError` | 记录 warn 日志，入队 `RedownloadMediaWorker` | `create.rb:322-324` |
| `process_attachments` | `Addressable::URI::InvalidURIError` | 记录 debug 日志 | `create.rb:329-331` |
| `fetch_replies` | 所有异常 | 记录 warn 日志 | `create.rb:390-391` |
| `fetch_and_verify_quote` | `Mastodon::RecursionLimitExceededError`, `UnexpectedResponseError`, `HTTP_CONNECTION_ERRORS` | 入队 `RefetchAndVerifyQuoteWorker` | `create.rb:399-400` |
| `conversation_from_uri` | `ActiveRecord::RecordInvalid`, `RecordNotUnique` | `retry`（局部重试，非 Sidekiq 重试） | `create.rb:410-412` |
| `increment_voters_count!` | `ActiveRecord::StaleObjectError` | `reload` + `retry`（局部重试） | `create.rb:477-479` |

**关键设计模式**：

1. **延迟重试而非立即重试**：
   ```ruby
   # 不是触发 Sidekiq 重试，而是使用专用的延迟 Worker
   RedownloadMediaWorker.perform_in(rand(PROCESSING_DELAY), media_attachment.id)
   MentionResolveWorker.perform_in(rand(PROCESSING_DELAY), status.id, uri, ...)
   RefetchAndVerifyQuoteWorker.perform_in(rand(PROCESSING_DELAY), quote.id, ...)
   ```

2. **局部重试（非 Sidekiq 重试）**：
   ```ruby
   # 使用 begin-rescue-retry 模式在同一方法内重试
   begin
     Conversation.find_or_create_by!(uri: uri)
   rescue ActiveRecord::RecordInvalid, ActiveRecord::RecordNotUnique
     retry
   end
   ```

##### 8.3.1.4 其他 Activity 处理器中的异常捕获

| Activity | 捕获的异常 | 处理方式 |
|----------|------------|----------|
| **Update** | `Date::Error` | 静默 return `false` |
| **LinkedDataSignature** | `OpenSSL::PKey::RSAError` | 静默 return `nil` |
| **ProcessAccountService** | `JSON::ParserError` | 静默 return `nil` |

#### 8.3.2 条件 return（正常控制流，非异常）

这些情况使用 `return` 而非异常，永远不会触发 Sidekiq 重试。

##### 8.3.2.1 去重命中场景

```ruby
# Create - Tombstone 存在
return reject_payload! if ... tombstone_exists? ...

# Create - Delete Upon Arrival 标记
return if delete_arrived_first?(object_uri)

# Create - 已存在相同 Status
@status = find_existing_status
if @status.nil?
  process_status
elsif @options[:delivered_to_account_id].present?
  postprocess_audience_and_deliver  # 部分副作用，非异常
end  # ⚠️ 没有 return，但也没有创建新记录

# Follow - 已存在 FollowRequest
existing_follow_request = ::FollowRequest.find_by(...)
unless existing_follow_request.nil?
  existing_follow_request.update!(uri: @json['id'])
  return  # ⚠️ 直接 return
end

# Follow - 已存在 Follow
existing_follow = ::Follow.find_by(...)
unless existing_follow.nil?
  existing_follow.update!(uri: @json['id'])
  AuthorizeFollowService.new.call(...)
  return  # ⚠️ 直接 return
end
```

##### 8.3.2.2 解析失败场景

```ruby
# ProcessCollectionService - 不支持的 Context
return if !supported_context? || ...

# ProcessCollectionService - Actor 已暂停
return if ... suspended_actor? ...

# ProcessCollectionService - Actor 是本地账户
return if ... @account.local?

# Activity.factory - 未知 type
def klass_for(json)
  case json['type']
  when 'Create' then ...
  # ... 其他 type
  # ⚠️ 没有 else 分支，未知 type 返回 nil
  end
end

# process_item - 使用安全导航
activity = ActivityPub::Activity.factory(item, ...)
activity&.perform  # ⚠️ &. 操作符，activity 为 nil 时不执行
```

##### 8.3.2.3 前置检查失败

```ruby
# ProcessingWorker - Actor 不存在
actor = Account.find_by(id: actor_id)
return if actor.nil?  # ⚠️ find_by 返回 nil，不是异常

# InboxesController - 前置检查（控制器阶段，不入队）
skip_large_payload      # > 1MB 返回 413
skip_unknown_actor_activity  # 未知 actor 返回 202
```

---

### 8.4 静默结束 vs 触发重试的对比表

| 场景 | 处理方式 | 触发条件 | 代码位置 |
|------|----------|----------|----------|
| **验签失败** | 控制器返回错误，不入队 | 签名验证失败 | `signature_verification.rb` |
| **JSON::ParserError** | 静默 return nil | JSON 格式错误 | `process_collection_service.rb:41` |
| **JSON::LD::JsonLdError** | 移除 signature 后继续 | JSON-LD 规范化失败 | `process_collection_service.rb:16-18` |
| **不支持的 Context** | 静默 return | 非标准 JSON-LD Context | `process_collection_service.rb:21` |
| **Actor 不匹配** | 静默 return | actor 与签名不一致 | `process_collection_service.rb:21` |
| **Actor 已暂停** | 静默 return | 账户被暂停 | `process_collection_service.rb:21` |
| **Actor 是本地** | 静默 return | 本地账户的 activity | `process_collection_service.rb:21` |
| **未知 Activity type** | `activity&.perform` 不执行 | type 不在支持列表 | `activity.rb:30-65` |
| **ActiveRecord::RecordInvalid** | 记录 debug 日志 | 数据验证失败 | `processing_worker.rb:17-19` |
| **Tombstone 存在** | `reject_payload!` | 对象已被删除过 | `create.rb:18` |
| **Delete Upon Arrival** | 静默 return | Delete 先于 Create 到达 | `create.rb:22` |
| **Status 已存在** | 部分副作用或直接 return | 已处理过相同 URI | `create.rb:24-31` |
| **FollowRequest 已存在** | 更新 URI 后 return | 重复的关注请求 | `follow.rb:12-16` |
| **Follow 已存在** | 更新 URI + 调用服务后 return | 重复的关注 | `follow.rb:24-29` |
| **网络异常（process_mention）** | 加入 unresolved_mentions | 远程账户获取失败 | `create.rb:260-261` |
| **网络异常（process_attachments）** | 入队 RedownloadMediaWorker | 媒体下载失败 | `create.rb:320-324` |
| **数据库连接异常** | 触发 Sidekiq 重试 | 数据库暂时不可用 | 未被捕获 |
| **Redis 异常** | 触发 Sidekiq 重试 | Redis 故障 | 未被捕获 |
| **Activity::Move 中异常** | 清理后重新抛出 → 触发重试 | 账户迁移失败 | `move.rb:22-24` |
| **乐观锁冲突（部分场景）** | 触发 Sidekiq 重试 | 并发更新 | 未被捕获 |

---

### 8.5 关键设计理念

#### 8.5.1 可重试 vs 不可重试的区分原则

| 维度 | 可重试（触发 Sidekiq） | 不可重试（静默结束） |
|------|------------------------|----------------------|
| **错误性质** | 临时性故障 | 永久性错误 |
| **典型场景** | 数据库连接中断、Redis 故障、网络抖动 | JSON 格式错误、数据验证失败、重复请求 |
| **重试价值** | 重试可能成功 | 重试也无法解决 |
| **处理策略** | 指数退避重试 | 记录日志、部分副作用、延迟 Worker |

#### 8.5.2 延迟重试模式

**不是使用 Sidekiq 重试，而是使用专用的延迟 Worker**：

```ruby
# 示例 1: 媒体下载失败
RedownloadMediaWorker.perform_in(rand(PROCESSING_DELAY), media_attachment.id)
# PROCESSING_DELAY = (30.seconds)..(10.minutes)

# 示例 2: 提及解析失败
MentionResolveWorker.perform_in(rand(PROCESSING_DELAY), status.id, uri, ...)

# 示例 3: 引用验证失败
RefetchAndVerifyQuoteWorker.perform_in(rand(PROCESSING_DELAY), quote.id, ...)
```

**优势**：
1. **精细控制**：可以为不同类型的失败设置不同的延迟策略
2. **不阻塞主流程**：主 Worker 可以继续处理其他任务
3. **避免级联失败**：一个组件的失败不会影响整个 Activity 的处理

#### 8.5.3 幂等性保障

**多层去重机制确保重试是安全的**：

1. **URI 查找**：`find_existing_status` 检查是否已存在
2. **Tombstone**：已删除的对象不会被重建
3. **Delete Upon Arrival**：Redis 标记防止 Delete 先于 Create 到达
4. **Redis 分布式锁**：`with_redis_lock` 防止并发处理

**设计目标**：
> 即使同一个 Activity 被处理多次，也应该产生相同的结果（或不产生副作用）

#### 8.5.4 Move 操作的特殊设计

```ruby
# 位于 app/lib/activitypub/activity/move.rb:22-24

rescue
  unmark_as_processing!  # 清理 Redis 锁
  raise                   # 重新抛出，允许重试
end
```

**为什么这样设计**：

1. **状态一致性**：`move_in_progress` 标记必须在失败时清除
2. **并发安全**：`mark_as_processing!` 使用 `nx: true`（set if not exists）
3. **允许恢复**：临时故障（如网络中断）应该可以重试

**Redis 锁的完整逻辑**：

```ruby
# 位于 app/lib/activitypub/activity/move.rb:37-39

def mark_as_processing!
  # nx: true = 只有 key 不存在时才设置
  # ex: PROCESSING_COOLDOWN = 7 天自动过期
  redis.set("move_in_progress:#{@account.id}", true, nx: true, ex: PROCESSING_COOLDOWN)
end

# 位于 app/lib/activitypub/activity/move.rb:41-43

def unmark_as_processing!
  redis.del("move_in_progress:#{@account.id}")
end
```

---

### 8.6 重试行为决策树（完整版）

```
远端实例 POST Activity
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 阶段 1: 控制器 before_action (验签)                           │
└─────────────────────────────────────────────────────────────┘
         │
         ├── 验签失败 ───────────────────────────────────────┐
         │                                                    │
         │    HTTP 响应: 400/401/403/503                   │
         │    ❌ 不入队，不重试                               │
         │                                                    │
         └────────────────────────────────────────────────────┘
         │
         ▼ 验签成功
┌─────────────────────────────────────────────────────────────┐
│ 阶段 2: 控制器 create 动作                                    │
│  - 入队 ProcessingWorker                                      │
│  - 返回 202 Accepted                                          │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 阶段 3: ProcessingWorker#perform                             │
│  sidekiq_options retry: 8                                    │
└─────────────────────────────────────────────────────────────┘
         │
         ├── Actor 不存在 ──────────────────────────────────┐
         │  (Account.find_by 返回 nil)                        │
         │                                                    │
         │    静默 return，非异常                             │
         │    ❌ 不触发重试                                   │
         │                                                    │
         └────────────────────────────────────────────────────┘
         │
         ▼ Actor 存在
┌─────────────────────────────────────────────────────────────┐
│ 阶段 4: ProcessCollectionService#call                        │
└─────────────────────────────────────────────────────────────┘
         │
         ├── JSON::ParserError ─────────────────────────────┐
         │                                                    │
         │    rescue 捕获，return nil                         │
         │    ❌ 不触发重试                                   │
         │                                                    │
         └────────────────────────────────────────────────────┘
         │
         ├── JSON::LD::JsonLdError ─────────────────────────┐
         │                                                    │
         │    rescue 捕获，移除 signature 后继续              │
         │    ❌ 不触发重试                                   │
         │                                                    │
         └────────────────────────────────────────────────────┘
         │
         ├── 前置检查失败 ───────────────────────────────────┐
         │  (!supported_context? || suspended_actor? || ...) │
         │                                                    │
         │    条件 return，非异常                             │
         │    ❌ 不触发重试                                   │
         │                                                    │
         └────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 阶段 5: Activity 处理器 (根据 type 分发)                       │
└─────────────────────────────────────────────────────────────┘
         │
         ├── 去重命中 ───────────────────────────────────────┐
         │  (Tombstone / Delete Upon Arrival / URI 已存在)    │
         │                                                    │
         │    条件 return 或 reject_payload!                  │
         │    ❌ 不触发重试                                   │
         │                                                    │
         └────────────────────────────────────────────────────┘
         │
         ├── 局部异常（被捕获）──────────────────────────────┐
         │                                                    │
         │    ┌──────────────────────────────────────────┐   │
         │    │ 异常类型不同，处理方式不同:               │   │
         │    │                                          │   │
         │    │ ❌ 静默结束:                             │   │
         │    │    - ActiveRecord::RecordInvalid         │   │
         │    │    - Addressable::URI::InvalidURIError  │   │
         │    │                                          │   │
         │    │ ⏰ 延迟重试（专用 Worker）:               │   │
         │    │    - 网络异常 → RedownloadMediaWorker    │   │
         │    │    - 提及解析失败 → MentionResolveWorker │   │
         │    │    - 引用验证失败 → RefetchQuoteWorker    │   │
         │    │                                          │   │
         │    │ 🔄 局部重试（begin-rescue-retry）:        │   │
         │    │    - RecordNotUnique (并发创建)           │   │
         │    │    - StaleObjectError (乐观锁)            │   │
         │    └──────────────────────────────────────────┘   │
         │                                                    │
         │    ❌ 不触发 Sidekiq 重试                         │
         │                                                    │
         └────────────────────────────────────────────────────┘
         │
         ├── Activity::Move 异常 ──────────────────────────┐
         │                                                    │
         │    rescue 捕获所有异常                             │
         │    ├── unmark_as_processing! (清理 Redis 锁)      │
         │    └── raise (重新抛出)                            │
         │                                                    │
         │    ✅ 触发 Sidekiq 重试（指数退避）                │
         │                                                    │
         └────────────────────────────────────────────────────┘
         │
         ├── 未被捕获的异常 ─────────────────────────────────┐
         │                                                    │
         │    异常向上冒泡到 Sidekiq 框架                      │
         │                                                    │
         │    典型异常:                                        │
         │    - ActiveRecord::StatementInvalid (数据库)       │
         │    - Redis::BaseError (Redis)                      │
         │    - HTTP::Error (网络，部分场景)                   │
         │    - 其他 StandardError 子类                       │
         │                                                    │
         │    ✅ 触发 Sidekiq 重试（指数退避）                │
         │                                                    │
         └────────────────────────────────────────────────────┘
         │
         ▼ 重试 8 次后仍失败
┌─────────────────────────────────────────────────────────────┐
│ 任务被移到 Dead Job（死信）队列                              │
│ 需要手动干预或使用 sidekiq 重试命令                          │
└─────────────────────────────────────────────────────────────┘
```

---

### 8.7 重试相关配置汇总

| 配置项 | 值 | 说明 |
|--------|-----|------|
| ProcessingWorker `retry` | `8` | 最多重试 8 次 |
| ProcessingWorker `queue` | `'ingress'` | 优先级 4 |
| PROCESSING_DELAY | `30s ~ 10min` | 延迟 Worker 的随机延迟范围 |
| DISTRIBUTE_DELAY | `1min` | 链接爬取的延迟 |
| Sidekiq 指数退避 | `15^n 秒` | 第 n 次重试的延迟 |
| Move 操作冷却期 | `7 天` | `move_in_progress` 锁的 TTL |

---

### 8.8 关键代码位置汇总

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| ProcessingWorker 重试配置 | `app/workers/activitypub/processing_worker.rb` | 6 |
| ProcessingWorker 异常捕获 | `app/workers/activitypub/processing_worker.rb` | 17-19 |
| Move 操作重试设计 | `app/lib/activitypub/activity/move.rb` | 22-24 |
| ProcessCollectionService 解析错误 | `app/services/activitypub/process_collection_service.rb` | 14-19, 41-43 |
| Create 媒体下载延迟重试 | `app/lib/activitypub/activity/create.rb` | 320-324 |
| Create 提及解析延迟重试 | `app/lib/activitypub/activity/create.rb` | 260-261 |
| Create 引用验证延迟重试 | `app/lib/activitypub/activity/create.rb` | 399-400 |
| Sidekiq 队列配置 | `config/sidekiq.yml` | 6 |
