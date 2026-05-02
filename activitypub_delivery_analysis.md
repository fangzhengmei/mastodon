# Mastodon ActivityPub 状态投递完整链路分析

## 概述

本报告详细分析 Mastodon 中，本地用户发布状态后，如何通过 ActivityPub 协议投递给其他实例关注者的完整技术链路。涵盖从本地发布、序列化、签名、投递队列到远端实例接收处理的全流程。

---

## 第一阶段：本地状态发布

### 1.1 API 入口点

用户通过 REST API 发布状态，入口位于：

**文件**: `app/controllers/api/v1/statuses_controller.rb:80-103`

```ruby
def create
  @status = PostStatusService.new.call(
    current_user.account,
    text: status_params[:status],
    thread: @thread,
    quoted_status: @quoted_status,
    quote_approval_policy: quote_approval_policy,
    media_ids: status_params[:media_ids],
    sensitive: status_params[:sensitive],
    spoiler_text: status_params[:spoiler_text],
    visibility: status_params[:visibility],
    language: status_params[:language],
    scheduled_at: status_params[:scheduled_at],
    application: doorkeeper_token.application,
    poll: status_params[:poll],
    allowed_mentions: status_params[:allowed_mentions],
    idempotency: request.headers['Idempotency-Key'],
    with_rate_limit: true
  )

  render json: @status, serializer: serializer_for_status
end
```

### 1.2 PostStatusService 核心流程

状态发布的核心服务处理完整的创建逻辑：

**文件**: `app/services/post_status_service.rb`

#### 主要执行步骤：

1. **幂等性检查** (第 49-58 行)
   - 支持 `Idempotency-Key` 请求头防止重复发布
   - 使用 Redis 锁和缓存实现

2. **预处理属性** (第 72-82 行)
   - 处理可见性设置
   - 处理敏感内容标记
   - 处理定时发布时间

3. **保存状态** (第 84-100 行)
   ```ruby
   def process_status!
     @status = @account.statuses.new(status_attributes)
     process_mentions_service.call(@status)
     safeguard_mentions!(@status)
     safeguard_private_mention_quote!(@status)
     attach_tagged_objects!(@status)
     attach_quote!(@status)

     antispam = Antispam.new(@status)
     antispam.local_preflight_check!

     ApplicationRecord.transaction do
       @status.save!
     end
   end
   ```

4. **触发后台任务** (第 162-171 行)
   ```ruby
   def postprocess_status!
     process_hashtags_service.call(@status)
     Trends.tags.register(@status)
     LinkCrawlWorker.perform_async(@status.id)
     DistributionWorker.perform_async(@status.id)  # 本地时间线分发
     process_email_subscriptions!
     ActivityPub::DistributionWorker.perform_async(@status.id)  # ActivityPub 联邦分发
     # ... 其他任务
   end
   ```

**关键点**: 第 168 行触发 `ActivityPub::DistributionWorker`，这是 ActivityPub 联邦的起点。

---

## 第二阶段：ActivityPub 分发准备

### 2.1 DistributionWorker 入口

**文件**: `app/workers/activitypub/distribution_worker.rb`

```ruby
class ActivityPub::DistributionWorker < ActivityPub::RawDistributionWorker
  def perform(status_id)
    @status  = Status.find(status_id)
    @account = @status.account

    distribute!
  rescue ActiveRecord::RecordNotFound
    true
  end

  protected

  def inboxes
    @inboxes ||= StatusReachFinder.new(@status).inboxes
  end

  def payload
    @payload ||= serialize_payload(@status, activity_serializer, serializer_options.merge(signer: @account)).to_json
  end

  def activity_serializer
    @status.reblog? ? ActivityPub::AnnounceNoteSerializer : ActivityPub::CreateNoteSerializer
  end
end
```

### 2.2 确定目标收件箱

`StatusReachFinder` 负责确定需要投递的所有目标实例 inbox URL：

**文件**: `app/lib/status_reach_finder.rb:12-117`

```ruby
def inboxes
  (reached_account_inboxes + followers_inboxes + relay_inboxes).uniq
end

private

def reached_account_inboxes
  scope = Account.where(id: reached_account_ids)
  inboxes_without_suspended_for(scope)
end

def reached_account_ids
  if @status.reblog?
    [reblog_of_account_id]
  else
    [
      replied_to_account_id,    # 被回复的账户
      reblog_of_account_id,     # 被转发的账户
      quote_of_account_id,      # 被引用的账户
      mentioned_account_ids,    # 被@的账户
      reblogs_account_ids,      # 转发过此状态的账户
      quotes_account_ids,       # 引用过此状态的账户
      favourites_account_ids,   # 点赞过此状态的账户
      replies_account_ids,      # 回复过此状态的账户
    ].tap do |arr|
      arr.flatten!
      arr.compact!
      arr.uniq!
    end
  end
end

def followers_inboxes
  scope = followers_scope
  inboxes_without_suspended_for(scope)
end

def relay_inboxes
  if @status.public_visibility?
    Relay.enabled.pluck(:inbox_url)  # 公共状态投递给中继服务器
  else
    []
  end
end

def followers_scope
  if @status.in_reply_to_local_account? && distributable?
    @status.account.followers.or(@status.thread.account.followers.not_domain_blocked_by_account(@status.account))
  elsif @status.direct_visibility? || @status.limited_visibility?
    Account.none  # 私信和有限可见状态不通过 ActivityPub 投递
  else
    @status.account.followers
  end
end
```

**关键逻辑**：
- **公共状态** (`public_visibility?`)：投递给所有关注者 + 中继服务器
- **回复本地账户**：投递给回复者和原作者的关注者
- **私信/有限可见**：不通过 ActivityPub 联邦

---

## 第三阶段：序列化为 ActivityPub 活动

### 3.1 Payloadable 模块

**文件**: `app/services/concerns/payloadable.rb`

```ruby
module Payloadable
  def serialize_payload(record, serializer, options = {})
    signer      = options.delete(:signer)
    sign_with   = options.delete(:sign_with)
    always_sign = options.delete(:always_sign)
    payload     = ActiveModelSerializers::SerializableResource.new(record, options.merge(serializer: serializer, adapter: ActivityPub::Adapter)).as_json
    object      = record.respond_to?(:virtual_object) ? record.virtual_object : record

    if object.respond_to?(:sign?) && object.sign? && signer && (always_sign || signing_enabled?)
      ActivityPub::LinkedDataSignature.new(payload).sign!(signer, sign_with: sign_with)
    else
      payload
    end
  end
end
```

### 3.2 CreateNoteSerializer

**文件**: `app/serializers/activitypub/create_note_serializer.rb`

```ruby
class ActivityPub::CreateNoteSerializer < ActivityPub::Serializer
  attributes :id, :type, :actor, :published, :to, :cc

  has_one :object, serializer: ActivityPub::NoteSerializer

  def id
    ActivityPub::TagManager.instance.activity_uri_for(object)
  end

  def type
    'Create'
  end

  def actor
    ActivityPub::TagManager.instance.uri_for(object.account)
  end

  def to
    ActivityPub::TagManager.instance.to(object)
  end

  def cc
    ActivityPub::TagManager.instance.cc(object)
  end

  def published
    object.created_at.iso8601
  end
end
```

### 3.3 NoteSerializer (核心对象序列化)

**文件**: `app/serializers/activitypub/note_serializer.rb`

```ruby
class ActivityPub::NoteSerializer < ActivityPub::Serializer
  context_extensions :atom_uri, :conversation, :sensitive, :voters_count, :quotes, :interaction_policies

  attributes :id, :type, :summary,
             :in_reply_to, :published, :url,
             :attributed_to, :to, :cc, :sensitive,
             :atom_uri, :in_reply_to_atom_uri,
             :conversation, :context

  attribute :content
  attribute :content_map, if: :language?
  attribute :updated, if: :edited?

  has_many :virtual_attachments, key: :attachment
  has_many :virtual_tags, key: :tag
  # ... 更多属性

  def type
    object.preloadable_poll ? 'Question' : 'Note'
  end

  def content
    status_content_format(object)
  end

  def virtual_tags
    object.active_mentions.to_a.sort_by(&:id) + object.tags + object.emojis + object.tagged_objects.map(&:object)
  end
end
```

### 3.4 生成的 ActivityPub JSON 示例

```json
{
  "@context": [
    "https://www.w3.org/ns/activitystreams",
    "https://w3id.org/security/v1"
  ],
  "id": "https://example.com/users/alice/statuses/123/activity",
  "type": "Create",
  "actor": "https://example.com/users/alice",
  "published": "2024-01-15T10:30:00Z",
  "to": [
    "https://www.w3.org/ns/activitystreams#Public"
  ],
  "cc": [
    "https://example.com/users/alice/followers"
  ],
  "object": {
    "id": "https://example.com/users/alice/statuses/123",
    "type": "Note",
    "content": "<p>Hello, Fediverse!</p>",
    "attributedTo": "https://example.com/users/alice",
    "to": [
      "https://www.w3.org/ns/activitystreams#Public"
    ],
    "cc": [
      "https://example.com/users/alice/followers"
    ],
    "tag": [
      {
        "type": "Mention",
        "href": "https://otherinstance.com/users/bob",
        "name": "@bob@otherinstance.com"
      }
    ]
  },
  "signature": {
    "type": "RsaSignature2017",
    "creator": "https://example.com/users/alice#main-key",
    "created": "2024-01-15T10:30:00Z",
    "signatureValue": "..."
  }
}
```

---

## 第四阶段：签名机制

Mastodon 实现了两种签名机制：
1. **HTTP Signatures** (请求级别签名)
2. **Linked Data Signatures** (文档级别签名)

### 4.1 Linked Data Signatures (LDS)

**文件**: `app/lib/activitypub/linked_data_signature.rb`

```ruby
class ActivityPub::LinkedDataSignature
  CONTEXT = 'https://w3id.org/identity/v1'
  SIGNATURE_CONTEXT = 'https://w3id.org/security/v1'

  def sign!(creator, sign_with: nil)
    options = {
      'type' => 'RsaSignature2017',
      'creator' => ActivityPub::TagManager.instance.key_uri_for(creator),
      'created' => Time.now.utc.iso8601,
    }

    # 1. 规范化并哈希签名选项
    options_hash  = hash(options.without('type', 'id', 'signatureValue').merge('@context' => CONTEXT))
    # 2. 规范化并哈希文档内容
    document_hash = hash(@json.without('signature'))
    # 3. 拼接后签名
    to_be_signed  = options_hash + document_hash
    keypair       = sign_with.present? ? OpenSSL::PKey::RSA.new(sign_with) : creator.keypair

    signature = Base64.strict_encode64(keypair.sign(OpenSSL::Digest.new('SHA256'), to_be_signed))

    # 4. 添加安全上下文
    context_with_security = Array(@json['@context'])
    context_with_security << 'https://w3id.org/security/v1'
    context_with_security.uniq!

    @json.merge('signature' => options.merge('signatureValue' => signature), '@context' => context_with_security)
  end

  private

  def hash(obj)
    Digest::SHA256.hexdigest(canonicalize(obj))
  end
end
```

**LDS 签名流程**：
1. 创建签名选项对象（包含 type、creator、created）
2. 使用 JSON-LD 规范化算法规范化选项和文档
3. 分别 SHA-256 哈希
4. 拼接两个哈希值
5. 使用 RSA-SHA256 私钥签名
6. Base64 编码签名结果
7. 将签名和安全上下文添加到 JSON 中

### 4.2 HTTP Signatures

**文件**: `app/lib/request.rb:100-108, 131, 191-193`

```ruby
def on_behalf_of(actor, sign_with: nil)
  raise ArgumentError, 'actor must not be nil' if actor.nil?

  key_id = ActivityPub::TagManager.instance.key_uri_for(actor)
  keypair = sign_with.present? ? OpenSSL::PKey::RSA.new(sign_with) : actor.keypair
  @signing = HttpSignatureDraft.new(keypair, key_id)

  self
end

def headers
  (@signing ? @headers.merge('Signature' => signature) : @headers)
end

def signature
  @signing.sign(@headers.without('User-Agent', 'Accept-Encoding'), @verb, @url)
end
```

**HTTP Signatures 签名的头信息**：
- `(request-target)`: 请求方法和路径
- `Host`: 目标主机
- `Date`: 当前时间
- `Digest`: 请求体的 SHA-256 哈希（格式：`SHA-256=<base64>`）

**签名头示例**：
```
Signature: keyId="https://example.com/users/alice#main-key",
           algorithm="rsa-sha256",
           headers="(request-target) host date digest",
           signature="..."
```

---

## 第五阶段：投递队列与后台任务

### 5.1 RawDistributionWorker 基类

**文件**: `app/workers/activitypub/raw_distribution_worker.rb`

```ruby
class ActivityPub::RawDistributionWorker
  include Sidekiq::Worker
  include Payloadable

  sidekiq_options queue: 'push'

  def perform(json, source_account_id, exclude_inboxes = [])
    @account         = Account.find(source_account_id)
    @json            = json
    @exclude_inboxes = exclude_inboxes

    distribute!
  rescue ActiveRecord::RecordNotFound
    true
  end

  protected

  def distribute!
    return if inboxes.empty?

    # 批量推送到 DeliveryWorker，每批 1000 个
    ActivityPub::DeliveryWorker.push_bulk(inboxes, limit: 1_000) do |inbox_url|
      [payload, source_account_id, inbox_url, options]
    end
  end
end
```

### 5.2 DeliveryWorker 实际投递

**文件**: `app/workers/activitypub/delivery_worker.rb`

```ruby
class ActivityPub::DeliveryWorker
  include Sidekiq::Worker
  include RoutingHelper
  include JsonLdHelper

  STOPLIGHT_COOL_OFF_TIME = 60
  STOPLIGHT_FAILURE_THRESHOLD = 10

  sidekiq_options queue: 'push', retry: 16, dead: false

  # 指数退避重试策略
  sidekiq_retry_in do |count|
    delay  = (count**4) + 15      # 基础延迟：count^4 + 15
    jitter = rand(0.5 * (count**4)) # 随机抖动
    delay + jitter
  end

  HEADERS = { 'Content-Type' => 'application/activity+json' }.freeze

  def perform(json, source_account_id, inbox_url, options = {})
    @options        = options.with_indifferent_access

    # 检查目标实例是否可用（基于失败追踪）
    return unless @options[:bypass_availability] || DeliveryFailureTracker.available?(inbox_url)

    @json           = json
    @source_account = Account.find(source_account_id)
    @inbox_url      = inbox_url
    @host           = Addressable::URI.parse(inbox_url).normalized_site
    @performed      = false

    perform_request
  ensure
    # 追踪投递结果
    if @inbox_url.present?
      if @performed
        failure_tracker.track_success!
      elsif !@unsalvageable
        failure_tracker.track_failure!
      end
    end
  end

  private

  def build_request(http_client)
    Request.new(:post, @inbox_url, body: @json, http_client: http_client).tap do |request|
      request.on_behalf_of(@source_account, sign_with: @options[:sign_with])
      request.add_headers(HEADERS)
      # 关注者同步头（用于私有状态）
      request.add_headers({ 'Collection-Synchronization' => synchronization_header }) if ENV['DISABLE_FOLLOWERS_SYNCHRONIZATION'] != 'true' && @options[:synchronize_followers]
    end
  end

  def perform_request
    # 使用 Stoplight 熔断器模式
    stoplight_wrapper.run do
      request_pool.with(@host) do |http_client|
        build_request(http_client).perform do |response|
          if response_successful?(response)
            @performed = true
          elsif response_error_unsalvageable?(response) || unsalvageable_authorization_failure?(response)
            @unsalvageable = true
          else
            raise Mastodon::UnexpectedResponseError, response
          end
        end
      end
    end
  end

  def stoplight_wrapper
    Stoplight(
      @inbox_url,
      cool_off_time: STOPLIGHT_COOL_OFF_TIME,
      threshold: STOPLIGHT_FAILURE_THRESHOLD
    )
  end
end
```

### 5.3 投递机制关键特性

| 特性 | 实现细节 |
|------|----------|
| **队列优先级** | `queue: 'push'` |
| **最大重试** | 16 次 (`retry: 16`) |
| **死信队列** | 不保存死信 (`dead: false`) |
| **重试延迟** | 指数退避：`count^4 + 15 + jitter` 秒 |
| **熔断器** | Stoplight：10 次失败后冷却 60 秒 |
| **失败追踪** | `DeliveryFailureTracker` 记录实例可用性 |
| **连接池** | `RequestPool` 复用 HTTP 连接 |

---

## 第六阶段：远端实例接收处理

### 6.1 InboxesController 入口

**文件**: `app/controllers/activitypub/inboxes_controller.rb`

```ruby
class ActivityPub::InboxesController < ActivityPub::BaseController
  include JsonLdHelper

  before_action :skip_large_payload
  before_action :skip_unknown_actor_activity
  before_action :require_actor_signature!  # 强制签名验证
  skip_before_action :authenticate_user!

  def create
    upgrade_account
    process_collection_synchronization
    process_payload
    head 202  # 返回 202 Accepted，异步处理
  end

  private

  def skip_large_payload
    head 413 if request.content_length > ActivityPub::Activity::MAX_JSON_SIZE  # 1MB 限制
  end

  def process_payload
    ActivityPub::ProcessingWorker.perform_async(signed_request_actor.id, body, @account&.id, signed_request_actor.class.name)
  end
end
```

### 6.2 签名验证

**文件**: `app/controllers/concerns/signature_verification.rb`

```ruby
module SignatureVerification
  EXPIRATION_WINDOW_LIMIT = 12.hours
  CLOCK_SKEW_MARGIN       = 1.hour

  def require_actor_signature!
    render json: signature_verification_failure_reason, status: signature_verification_failure_code unless signed_request_actor
  end

  def signed_request_actor
    return @signed_request_actor if defined?(@signed_request_actor)

    raise Mastodon::SignatureVerificationError, 'Request not signed' unless signed_request?

    keypair = keypair_from_key_id

    raise Mastodon::SignatureVerificationError, "Public key not found for key #{signature_key_id}" if keypair.nil?

    check_keypair_validity!(keypair)
    return (@signed_request_actor = keypair.actor) if signed_request.verified?(keypair)

    # 验证失败时尝试刷新公钥
    keypair = stoplight_wrapper.run { keypair_refresh_key!(keypair) }

    raise Mastodon::SignatureVerificationError, "Could not refresh public key #{signature_key_id}" if keypair.nil?

    check_keypair_validity!(keypair)
    return (@signed_request_actor = keypair.actor) if signed_request.verified?(keypair)

    fail_with! "Verification failed for #{keypair.actor.to_log_human_identifier}"
  end

  def keypair_from_key_id
    key_id = signed_request.key_id
    # ...
    if !ActivityPub::TagManager.instance.local_uri?(key_id)
      keypair = Keypair.from_keyid(key_id)
      return keypair if keypair.present?

      # 从远程获取公钥
      stoplight_wrapper.run { ActivityPub::FetchRemoteKeyService.new.call(key_id, suppress_errors: false) }
    end
  end
end
```

### 6.3 ProcessingWorker 异步处理

**文件**: `app/workers/activitypub/processing_worker.rb`

```ruby
class ActivityPub::ProcessingWorker
  include Sidekiq::Worker

  sidekiq_options queue: 'ingress', backtrace: true, retry: 8

  def perform(actor_id, body, delivered_to_account_id = nil, actor_type = 'Account')
    case actor_type
    when 'Account'
      actor = Account.find_by(id: actor_id)
    end

    return if actor.nil?

    ActivityPub::ProcessCollectionService.new.call(body, actor, override_timestamps: true, delivered_to_account_id: delivered_to_account_id, delivery: true)
  rescue ActiveRecord::RecordInvalid => e
    Rails.logger.debug { "Error processing incoming ActivityPub object: #{e}" }
  end
end
```

### 6.4 ProcessCollectionService

**文件**: `app/services/activitypub/process_collection_service.rb`

```ruby
class ActivityPub::ProcessCollectionService < BaseService
  include JsonLdHelper
  include DomainControlHelper

  def call(body, actor, **options)
    @account = actor
    @json    = original_json = JSON.parse(body)
    @options = options

    # 验证 Linked Data Signatures
    if @json['signature'].present?
      # 规范化 JSON-LD 用于签名验证
      @json = compact(@json) if @json['signature'].is_a?(Hash)
    end

    # 验证 actor 一致性和签名
    return if !supported_context? || (different_actor? && verify_account!.nil?) || suspended_actor? || @account.local?
    return unless @account.is_a?(Account)

    # 处理不同类型的活动
    case @json['type']
    when 'Collection', 'CollectionPage'
      process_items @json['items']
    when 'OrderedCollection', 'OrderedCollectionPage'
      process_items @json['orderedItems']
    else
      process_items [@json]
    end
  end

  private

  def process_item(item)
    activity = ActivityPub::Activity.factory(item, @account, **@options)
    activity&.perform
  end

  # 验证 LD-Signature
  def verify_account!
    return unless @json['signature'].is_a?(Hash)
    return if domain_not_allowed?(@json['signature']['creator'])

    @options[:relayed_through_actor] = @account
    @account = ActivityPub::LinkedDataSignature.new(@json).verify_actor!
    @account = nil unless @account.is_a?(Account)
    @account
  end
end
```

### 6.5 Activity 工厂模式

**文件**: `app/lib/activitypub/activity.rb`

```ruby
class ActivityPub::Activity
  MAX_JSON_SIZE = 1.megabyte
  SUPPORTED_TYPES = %w(Note Question).freeze
  CONVERTED_TYPES = %w(Image Audio Video Article Page Event).freeze

  class << self
    def factory(json, account, **)
      klass_for(json)&.new(json, account, **)
    end

    private

    def klass_for(json)
      case json['type']
      when 'Create'   then ActivityPub::Activity::Create
      when 'Announce' then ActivityPub::Activity::Announce
      when 'Delete'   then ActivityPub::Activity::Delete
      when 'Follow'   then ActivityPub::Activity::Follow
      when 'Like'     then ActivityPub::Activity::Like
      when 'Block'    then ActivityPub::Activity::Block
      when 'Update'   then ActivityPub::Activity::Update
      when 'Undo'     then ActivityPub::Activity::Undo
      when 'Accept'   then ActivityPub::Activity::Accept
      when 'Reject'   then ActivityPub::Activity::Reject
      when 'Flag'     then ActivityPub::Activity::Flag
      when 'Add'      then ActivityPub::Activity::Add
      when 'Remove'   then ActivityPub::Activity::Remove
      when 'Move'     then ActivityPub::Activity::Move
      # ... 更多类型
      end
    end
  end
end
```

### 6.6 Create 活动处理

**文件**: `app/lib/activitypub/activity/create.rb`

```ruby
class ActivityPub::Activity::Create < ActivityPub::Activity
  DISTRIBUTE_DELAY = 1.minute
  PROCESSING_DELAY = (30.seconds)..(10.minutes)

  def perform
    @account.schedule_refresh_if_stale!

    dereference_object!  # 如果 object 是 URI，解引用获取完整对象

    create_status
  end

  private

  def create_status
    # 验证检查
    return reject_payload! if unsupported_object_type? || non_matching_uri_hosts?(@account.uri, object_uri) || tombstone_exists? || !related_to_local_activity?

    with_redis_lock("create:#{object_uri}") do
      Status.uncached do
        return if delete_arrived_first?(object_uri) || poll_vote?

        @status = find_existing_status  # 检查是否已处理
      end

      if @status.nil?
        process_status  # 创建新状态
      elsif @options[:delivered_to_account_id].present?
        postprocess_audience_and_deliver  # 更新受众信息
      end
    end

    @status
  end

  def process_status
    @tags                 = []
    @mentions             = []
    @tagged_objects       = []
    @unresolved_mentions  = []
    @silenced_account_ids = []
    @params               = {}

    process_status_params  # 解析状态参数
    process_tags           # 处理标签、提及、表情
    process_quote          # 处理引用
    process_audience       # 处理受众（to/cc）

    ApplicationRecord.transaction do
      @status = Status.create!(@params.merge(quote: @quote))
      attach_tags(@status)
      attach_tagged_objects(@status)
      attach_mentions(@status)
      attach_counts(@status)
    end

    resolve_thread(@status)           # 解析回复线程
    resolve_unresolved_mentions(@status)  # 解析未解决的提及
    fetch_replies(@status)            # 获取回复
    fetch_and_verify_quote            # 获取并验证引用
    distribute                         # 分发给本地用户
    forward_for_reply                  # 转发回复活动
  end

  def distribute
    # 链接爬行延迟执行，避免 DDoS
    LinkCrawlWorker.perform_in(rand(DISTRIBUTE_DELAY), @status.id)

    # 分发到本地时间线并通知提及的账户
    ::DistributionWorker.perform_async(@status.id, { 'silenced_account_ids' => @silenced_account_ids }) if @options[:override_timestamps] || @status.within_realtime_window?
  end
end
```

---

## 第七阶段：完整链路流程图

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              发送端 (Instance A: example.com)                              │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                             │
│  1. 用户发布状态                                                                            │
│     ┌─────────────────┐                                                                     │
│     │ API 请求        │                                                                     │
│     │ POST /api/v1/statuses │                                                              │
│     └────────┬────────┘                                                                     │
│              ▼                                                                               │
│  ┌─────────────────────────────────┐                                                        │
│  │ PostStatusService#call          │  app/services/post_status_service.rb                 │
│  │  - 验证参数                      │                                                        │
│  │  - 处理 @ 提及                   │                                                        │
│  │  - 保存 Status 到数据库          │                                                        │
│  │  - 触发后台任务                   │                                                        │
│  └────────────────┬────────────────┘                                                        │
│                   │                                                                          │
│                   ▼                                                                          │
│  ┌────────────────────────────────────────┐                                                 │
│  │ ActivityPub::DistributionWorker        │  app/workers/activitypub/distribution_worker.rb│
│  │  - StatusReachFinder 确定目标收件箱     │                                                 │
│  │  - 序列化为 ActivityPub JSON            │                                                 │
│  │  - 批量推送到 DeliveryWorker             │                                                 │
│  └────────────────┬───────────────────────┘                                                 │
│                   │                                                                          │
│                   ▼                                                                          │
│  ┌──────────────────────────────────────────────────────────┐                              │
│  │ ActivityPub::DeliveryWorker (每个目标实例一个)            │  app/workers/activitypub/delivery_worker.rb│
│  │  - 构建 HTTP POST 请求                                    │                              │
│  │  - 添加 HTTP Signatures 签名                              │                              │
│  │  - 发送到目标 inbox URL                                   │                              │
│  │  - 处理重试、熔断、失败追踪                                │                              │
│  └────────────────┬─────────────────────────────────────────┘                              │
│                   │                                                                          │
│                   │  HTTP POST                                                               │
│                   │  Content-Type: application/activity+json                                │
│                   │  Signature: keyId="...", algorithm="rsa-sha256", ...                  │
│                   │  Digest: SHA-256=...                                                    │
│                   ▼                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      │  互联网
                                      │
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              接收端 (Instance B: otherinstance.com)                        │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                             │
│  ┌──────────────────────────────────────────┐                                              │
│  │ ActivityPub::InboxesController#create    │  app/controllers/activitypub/inboxes_controller.rb│
│  │  - 验证 HTTP Signatures                   │                                              │
│  │  - 检查 payload 大小 (<1MB)               │                                              │
│  │  - 触发 ProcessingWorker 异步处理          │                                              │
│  │  - 返回 202 Accepted                      │                                              │
│  └────────────────┬─────────────────────────┘                                              │
│                   │                                                                          │
│                   ▼                                                                          │
│  ┌─────────────────────────────────────────┐                                               │
│  │ ActivityPub::ProcessingWorker            │  app/workers/activitypub/processing_worker.rb│
│  │  - 调用 ProcessCollectionService          │                                               │
│  └────────────────┬────────────────────────┘                                               │
│                   │                                                                          │
│                   ▼                                                                          │
│  ┌─────────────────────────────────────────────────┐                                       │
│  │ ActivityPub::ProcessCollectionService           │  app/services/activitypub/process_collection_service.rb│
│  │  - 解析 JSON-LD                                  │                                       │
│  │  - 验证 Linked Data Signatures (如果有)         │                                       │
│  │  - 根据 activity type 选择处理器                 │                                       │
│  └────────────────┬────────────────────────────────┘                                       │
│                   │                                                                          │
│                   ▼ (type: "Create")                                                        │
│  ┌─────────────────────────────────────────┐                                               │
│  │ ActivityPub::Activity::Create            │  app/lib/activitypub/activity/create.rb     │
│  │  - 解析 object (Note/Question)            │                                               │
│  │  - 处理 @ 提及、标签、媒体附件             │                                               │
│  │  - 创建本地 Status 记录                    │                                               │
│  │  - 分发给本地关注者时间线                   │                                               │
│  └───────────────────────────────────────────┘                                               │
│                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 第八阶段：关键数据结构与文件索引

### 8.1 核心文件索引

| 模块 | 文件路径 | 职责 |
|------|----------|------|
| **API 入口** | `app/controllers/api/v1/statuses_controller.rb` | 状态创建 API |
| **发布服务** | `app/services/post_status_service.rb` | 状态创建核心逻辑 |
| **分发准备** | `app/workers/activitypub/distribution_worker.rb` | 确定目标、序列化 |
| **目标查找** | `app/lib/status_reach_finder.rb` | 计算需要投递的收件箱 |
| **序列化器** | `app/serializers/activitypub/create_note_serializer.rb` | Create 活动序列化 |
| **序列化器** | `app/serializers/activitypub/note_serializer.rb` | Note 对象序列化 |
| **签名模块** | `app/lib/activitypub/linked_data_signature.rb` | LD-Signatures |
| **签名模块** | `app/lib/request.rb` | HTTP Signatures |
| **实际投递** | `app/workers/activitypub/delivery_worker.rb` | HTTP POST 投递 |
| **接收入口** | `app/controllers/activitypub/inboxes_controller.rb` | inbox 端点 |
| **签名验证** | `app/controllers/concerns/signature_verification.rb` | HTTP 签名验证 |
| **异步处理** | `app/workers/activitypub/processing_worker.rb` | 异步处理接收的活动 |
| **活动解析** | `app/services/activitypub/process_collection_service.rb` | 活动类型分发 |
| **活动工厂** | `app/lib/activitypub/activity.rb` | 活动类型工厂 |
| **Create 处理** | `app/lib/activitypub/activity/create.rb` | Create 活动处理器 |

### 8.2 关键数据库模型

| 模型 | 表名 | 主要用途 |
|------|------|----------|
| `Status` | `statuses` | 存储嘟文/状态 |
| `Account` | `accounts` | 用户账户（本地和远程） |
| `Follow` | `follows` | 关注关系 |
| `Mention` | `mentions` | @ 提及 |
| `Keypair` | `keypairs` | 加密密钥对 |
| `DeliveryFailureTracker` | 无 | 追踪投递失败（Redis） |

### 8.3 Sidekiq 队列

| 队列名 | 用途 | 重试次数 |
|--------|------|----------|
| `push` | ActivityPub 投递（出站） | 16 |
| `ingress` | ActivityPub 处理（入站） | 8 |
| `default` | 常规后台任务 | - |

---

## 第九阶段：安全与可靠性机制

### 9.1 双重签名保障

Mastodon 实现了两层签名验证：

1. **HTTP Signatures** (传输层)
   - 验证请求确实由声称的发送者发出
   - 签名包含：`(request-target)`, `Host`, `Date`, `Digest`
   - 在 `InboxesController` 的 `before_action` 中强制验证

2. **Linked Data Signatures** (内容层)
   - 签名嵌入在 JSON 文档中
   - 允许活动被转发/中继时保持可验证性
   - 使用 `RsaSignature2017` 算法

### 9.2 失败处理与重试

| 机制 | 实现 | 效果 |
|------|------|------|
| **指数退避** | `delay = (count^4) + 15 + jitter` | 避免同时重试造成 DDoS |
| **熔断器** | Stoplight (10 次失败 → 冷却 60 秒) | 防止对不可用实例的持续尝试 |
| **失败追踪** | `DeliveryFailureTracker` | 记录实例可用性，跳过不可用实例 |
| **最大重试** | 16 次后放弃 | 防止无限重试 |

### 9.3 安全防护

| 防护措施 | 实现位置 | 目的 |
|----------|----------|------|
| **Payload 大小限制** | 1MB (`Activity::MAX_JSON_SIZE`) | 防止内存耗尽 |
| **私有网络阻止** | `Request::Socket.check_private_address` | 防止 SSRF 攻击 |
| **域名白名单** | `domain_not_allowed?` 检查 | 阻止被封禁的实例 |
| **时钟偏差容忍** | ±1 小时 (`CLOCK_SKEW_MARGIN`) | 处理不同步的服务器时钟 |
| **Redis 锁** | `with_redis_lock("create:#{object_uri}")` | 防止重复处理 |

### 9.4 可见性与联邦范围

| 可见性 | to 字段 | cc 字段 | 是否联邦 |
|--------|---------|---------|----------|
| **公开 (public)** | `as:Public` | 关注者集合 | ✓ 是 |
| **不公开 (unlisted)** | 关注者集合 | `as:Public` | ✓ 是 |
| **仅关注者 (private)** | 关注者集合 | 空 | ✓ 是 (带同步头) |
| **私信 (direct)** | 提及的账户 | 空 | ✗ 否 |
| **有限可见 (limited)** | 指定账户 | 空 | ✗ 否 |

---

## 第十阶段：性能优化策略

### 10.1 批量投递

```ruby
# app/workers/activitypub/raw_distribution_worker.rb
ActivityPub::DeliveryWorker.push_bulk(inboxes, limit: 1_000) do |inbox_url|
  [payload, source_account_id, inbox_url, options]
end
```

- 使用 Sidekiq 的 `push_bulk` 批量入队
- 每批 1000 个任务
- 避免为每个收件箱单独创建 Redis 操作

### 10.2 连接池

```ruby
# app/workers/activitypub/delivery_worker.rb
request_pool.with(@host) do |http_client|
  build_request(http_client).perform do |response|
    # ...
  end
end
```

- 同一主机的连接复用
- 减少 TCP 握手开销

### 10.3 异步解耦

| 阶段 | 同步/异步 | 原因 |
|------|-----------|------|
| 状态创建 | 同步 | 立即返回给用户 |
| 序列化和入队 | 异步 | 避免阻塞 API |
| 实际投递 | 异步 | 网络 IO 不确定 |
| 接收解析 | 异步 | 避免阻塞 inbox 端点 |

### 10.4 延迟分散

```ruby
# 链接爬取延迟分散，避免目标服务器压力
LinkCrawlWorker.perform_in(rand(DISTRIBUTE_DELAY), @status.id)

# 未解析的提及和引用，随机延迟重试
RedownloadMediaWorker.perform_in(rand(PROCESSING_DELAY), media_attachment.id)
```

---

## 总结

Mastodon 的 ActivityPub 投递链路是一个设计精良的异步分布式系统，具备以下特点：

1. **清晰的分层架构**：API 层 → 服务层 → Worker 层 → 网络层
2. **双重签名保障**：HTTP Signatures + LD Signatures 确保安全性和可转发性
3. **健壮的错误处理**：指数退避、熔断器、失败追踪
4. **精细的联邦控制**：基于可见性决定投递范围和目标
5. **性能优化**：批量操作、连接池、完全异步

这条链路体现了联邦社交网络的核心挑战和解决方案：如何在不可靠的网络环境中，安全、可靠、高效地在自治实例间传递消息。
