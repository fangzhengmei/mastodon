# Mastodon 远端用户账号发现与同步机制分析

本文档详细分析 Mastodon 中远端用户账号的发现、获取和同步机制，重点关注 WebFinger 协议的作用、更新时机以及本地缓存同步策略。

---

## 1. WebFinger 协议在账号查找中的作用

### 1.1 WebFinger 协议概述

WebFinger 是一种用于在互联网上发现资源信息的协议，它使用简单的 HTTP 请求和 JSON 响应格式。在 Mastodon 和其他联邦式社交网络中，WebFinger 主要用于：

- **将用户友好的账号格式**（如 `@username@domain`）**解析为 ActivityPub actor URL**
- **验证账号的真实性**，确保请求的账号确实属于目标域名
- **处理账号重定向**，当用户迁移到新域名时

### 1.2 Mastodon 中的 WebFinger 实现

Mastodon 实现了完整的 WebFinger 协议栈，包括客户端和服务端两部分。

#### 1.2.1 Webfinger 类 - 客户端实现

**文件位置**: `app/lib/webfinger.rb`

这是 WebFinger 客户端的核心实现，负责向远端服务器发送 WebFinger 请求并解析响应。

**关键功能**:

1. **标准 WebFinger 请求**:
   ```ruby
   def standard_url
     if @domain.end_with? '.onion'
       "http://#{@domain}/.well-known/webfinger?resource=#{@uri}"
     else
       "https://#{@domain}/.well-known/webfinger?resource=#{@uri}"
     end
   end
   ```
   - 支持标准 HTTPS 请求
   - 为 .onion 域名（Tor 网络）使用 HTTP

2. **Host-Meta 备用机制**:
   ```ruby
   def body_from_host_meta
     host_meta_request.perform do |res|
       raise Webfinger::Error, "Request for #{@uri} returned HTTP #{res.code}" unless res.code == 200

       body_from_webfinger(url_from_template(res.body_with_limit), use_fallback: false)
     end
   end
   ```
   - 当标准 WebFinger 端点返回 404 时，尝试使用 host-meta 作为备用
   - 从 host-meta 响应中解析 WebFinger 模板 URL

3. **响应解析与验证**:
   ```ruby
   def validate_response!
     raise Webfinger::Error, "Missing subject in response for #{@uri}" if subject.blank?
     raise Webfinger::Error, "Missing self link in response for #{@uri}" if self_link.blank?
   end
   ```
   - 验证响应中必须包含 `subject` 字段
   - 验证必须包含 ActivityPub 类型的 `self` 链接

#### 1.2.2 WebfingerController - 服务端实现

**文件位置**: `app/controllers/well_known/webfinger_controller.rb`

处理来自其他服务器的 WebFinger 请求，返回本地账号的信息。

**关键流程**:
1. 解析请求中的 `resource` 参数
2. 使用 `WebfingerResource` 类查找对应的本地账号
3. 检查账号是否被暂停
4. 使用 `WebfingerSerializer` 返回 JRD (JSON Resource Descriptor) 格式响应

**缓存策略**:
```ruby
def show
  expires_in 3.days, public: true
  render json: @account, serializer: WebfingerSerializer, content_type: 'application/jrd+json'
end
```
- 响应缓存 3 天，减少重复请求

#### 1.2.3 WebfingerResource - 资源解析器

**文件位置**: `app/lib/webfinger_resource.rb`

负责解析不同格式的 WebFinger 资源标识，查找对应的账号。

**支持的资源格式**:
1. **Instance Actor URL**: 用于服务器级别的活动
2. **标准 URL**: 如 `https://domain/users/username`
3. **Acct 格式**: 如 `acct:username@domain` 或 `username@domain`

**核心方法**:
```ruby
def account
  case resource
  when %r{\A(https?://)?#{instance_actor_regexp}/?\Z}
    Account.representative
  when /\Ahttps?/i
    account_from_url
  when /@/
    account_from_acct
  else
    raise InvalidRequest
  end
end
```

### 1.3 WebFinger 在账号查找流程中的角色

WebFinger 在 Mastodon 的账号查找流程中扮演着至关重要的角色，它是连接用户友好的账号格式与机器可读的 ActivityPub URL 的桥梁。

**典型查找流程**:
1. **用户输入**: `@username@remote.domain`
2. **WebFinger 请求**: 向 `remote.domain` 发送请求
3. **响应解析**: 从响应中提取 ActivityPub actor URL
4. **验证循环**: 确保 URL 确实指向目标账号
5. **数据获取**: 使用 ActivityPub 协议获取完整账号信息

**关键作用**:
- **发现**: 找到远端账号的 ActivityPub 端点
- **验证**: 确保账号身份的真实性
- **重定向**: 处理账号迁移的情况

---

## 2. 远端账号资料的获取与更新时机

### 2.1 完整的获取流程

Mastodon 使用多层服务架构来获取和处理远端账号信息，确保数据的一致性和可靠性。

#### 2.1.1 ResolveAccountService - 主入口服务

**文件位置**: `app/services/resolve_account_service.rb`

这是解析远端账号的主要入口点，协调整个查找和更新流程。

**核心流程**:
```ruby
def call(uri, options = {})
  return if uri.blank?

  process_options!(uri, options)

  # 1. 检查本地是否已有记录
  return if domain_not_allowed?(@domain)
  @account ||= Account.find_remote(@username, @domain)
  return @account if @account&.local? || @domain.nil? || !webfinger_update_due?

  # 2. 执行 WebFinger 查询
  process_webfinger!(@uri)
  @domain = nil if TagManager.instance.local_domain?(@domain)

  # 3. 再次检查（可能因重定向而改变）
  return if domain_not_allowed?(@domain)
  @account ||= Account.find_remote(@username, @domain)

  # 4. 处理账号删除的情况
  if gone_from_origin? && not_yet_deleted?
    queue_deletion!
    return
  end

  return @account if @account&.local? || gone_from_origin? || !webfinger_update_due?

  # 5. 获取或更新完整账号数据
  fetch_account!
rescue Webfinger::Error => e
  Rails.logger.debug { "Webfinger query for #{@uri} failed: #{e}" }
  raise unless @options[:suppress_errors]
end
```

**关键设计决策**:
1. **双重检查**: 在 WebFinger 查询前后都检查本地缓存
2. **重定向处理**: 支持 WebFinger 响应中的账号重定向
3. **删除检测**: 当远端返回 410 Gone 时，标记本地账号为删除
4. **分布式锁**: 使用 Redis 锁防止并发更新

#### 2.1.2 ActivityPub::FetchRemoteActorService - ActivityPub 数据获取

**文件位置**: `app/services/activitypub/fetch_remote_actor_service.rb`

负责获取和解析远端 ActivityPub actor 数据。

**核心流程**:
```ruby
def call(uri, prefetched_body: nil, break_on_redirect: false, only_key: false, suppress_errors: true, request_id: nil)
  return if domain_not_allowed?(uri)
  return ActivityPub::TagManager.instance.uri_to_actor(uri) if ActivityPub::TagManager.instance.local_uri?(uri)

  # 1. 获取或解析 JSON-LD 数据
  @json = begin
    if prefetched_body.nil?
      fetch_resource(uri, true)
    else
      body_to_json(prefetched_body, compare_id: uri)
    end
  rescue JSON::ParserError
    raise Error, "Error parsing JSON-LD document #{uri}"
  end

  # 2. 验证数据格式
  raise Error, "Error fetching actor JSON at #{uri}" if @json.nil?
  raise Error, "Unsupported JSON-LD context for document #{uri}" unless supported_context?
  raise Error, "Unexpected object type for actor #{uri} (expected any of: #{SUPPORTED_TYPES})" unless expected_type?
  
  # 3. 解析用户名和域名
  @uri = @json['id']
  @username, @domain = split_acct(@json['webfinger']) if @json['webfinger'].present? && @json['webfinger'].is_a?(String)
  
  if @username.blank? || @domain.blank?
    @username = @json['preferredUsername']
    @domain = Addressable::URI.parse(@uri).normalized_host
  end

  # 4. WebFinger 验证（除非只需要密钥）
  check_webfinger! unless only_key

  # 5. 处理账号数据
  ActivityPub::ProcessAccountService.new.call(@username, @domain, @json, only_key: only_key, verified_webfinger: !only_key, request_id: request_id)
end
```

**关键特性**:
1. **FEP-2c59 支持**: 优先使用 `webfinger` 属性（如果存在），减少额外请求
2. **类型验证**: 只处理支持的 actor 类型（Person, Group, Organization, Application, Service）
3. **WebFinger 验证循环**: 确保获取的 actor 确实与 WebFinger 响应匹配

#### 2.1.3 ActivityPub::ProcessAccountService - 数据处理与存储

**文件位置**: `app/services/activitypub/process_account_service.rb`

负责将获取到的 ActivityPub 数据处理并存储到本地数据库。

**核心方法**:
```ruby
def call(username, domain, json, options = {})
  return if json['inbox'].blank? || unsupported_uri_scheme?(json['id']) || domain_not_allowed?(domain)

  # ... 初始化变量 ...

  with_redis_lock("process_account:#{@uri}") do
    if @options[:only_key]
      @account = Account.remote.find_by(uri: @uri)
      return if @account.nil?
    else
      @account = Account.find_remote(@username, @domain)
    end

    # 保存旧状态用于后续比较
    @old_public_keys = @account.present? ? (@account.keypairs.pluck(:public_key) + [@account.public_key.presence].compact) : []
    @old_protocol = @account&.protocol
    @suspension_changed = false

    # 创建或更新账号
    if @account.nil?
      create_account
    end

    update_account
    process_tags

    process_duplicate_accounts! if @options[:verified_webfinger]
  end

  # 处理状态变更后的回调
  after_protocol_change! if protocol_changed?
  after_key_change! if all_public_keys_changed? && !@options[:signed_with_known_key]
  clear_tombstones! if all_public_keys_changed?
  after_suspension_change! if suspension_changed?

  # 处理额外的集合数据
  unless @options[:only_key] || @account.suspended?
    check_featured_collection! if @json['featured'].present?
    check_featured_tags_collection! if @json['featuredTags'].present?
    check_featured_collections_collection! if @json['featuredCollections'].present? && Mastodon::Feature.collections_enabled?
    check_links! if @account.fields.any?(&:requires_verification?)
  end

  @account
end
```

**账号创建**:
```ruby
def create_account
  @account = Account.new
  @account.protocol          = :activitypub
  @account.username          = @username
  @account.domain            = @domain
  @account.private_key       = nil
  @account.suspended_at      = domain_block.created_at if auto_suspend?
  @account.suspension_origin = :local if auto_suspend?
  @account.silenced_at       = domain_block.created_at if auto_silence?

  set_immediate_protocol_attributes!

  @account.save!
end
```

**账号更新**:
```ruby
def update_account
  @account.last_webfingered_at = Time.now.utc unless @options[:only_key]
  @account.protocol            = :activitypub

  set_suspension!
  set_immediate_protocol_attributes!
  set_fetchable_key! unless @account.suspended? && @account.suspension_origin_local?
  set_immediate_attributes! unless @account.suspended?
  set_fetchable_attributes! unless @options[:only_key] || @account.suspended?

  @account.save_with_optional_media!
end
```

### 2.2 更新时机

Mastodon 采用多种策略来确保远端账号信息的及时性和准确性。

#### 2.2.1 关键时间阈值

**文件位置**: `app/models/account.rb`

```ruby
BACKGROUND_REFRESH_INTERVAL = 1.week.freeze
REFRESH_DEADLINE = 6.hours
STALE_THRESHOLD = 1.day
```

- **STALE_THRESHOLD (1天)**: 超过此时间未更新的账号被认为是"可能过期"的
- **BACKGROUND_REFRESH_INTERVAL (1周)**: 后台刷新任务的执行间隔
- **REFRESH_DEADLINE (6小时)**: 后台刷新任务的随机延迟范围

#### 2.2.2 触发更新的场景

1. **首次查找**:
   - 当用户搜索或提及一个不存在于本地缓存的远端账号时
   - 触发 `ResolveAccountService` 执行完整的查找和获取流程

2. **主动刷新**:
   ```ruby
   def refresh!
     ResolveAccountService.new.call(acct) unless local?
   end
   ```
   - 可以通过 `refresh!` 方法手动触发刷新

3. **访问时检查**:
   ```ruby
   def schedule_refresh_if_stale!
     return unless last_webfingered_at.present? && last_webfingered_at <= BACKGROUND_REFRESH_INTERVAL.ago

     AccountRefreshWorker.perform_in(rand(REFRESH_DEADLINE), id)
   end
   ```
   - 访问远端账号时检查是否需要刷新
   - 如果超过 1 周未更新，安排后台刷新任务

4. **后台任务**:
   ```ruby
   class AccountRefreshWorker
     include Sidekiq::Worker

     sidekiq_options queue: 'pull', retry: 3, dead: false, lock: :until_executed, lock_ttl: 1.day.to_i

     def perform(account_id)
       account = Account.find_by(id: account_id)
       return if account.nil? || account.last_webfingered_at > Account::BACKGROUND_REFRESH_INTERVAL.ago

       ResolveAccountService.new.call(account)
     end
   end
   ```
   - `AccountRefreshWorker` 定期执行刷新任务
   - 使用分布式锁防止重复执行
   - 重试 3 次后放弃

5. **过期检测**:
   ```ruby
   def possibly_stale?
     last_webfingered_at.nil? || last_webfingered_at <= STALE_THRESHOLD.ago
   end
   ```
   - 用于判断是否应该触发 WebFinger 更新

### 2.3 更新决策逻辑

在 `ResolveAccountService` 中，有一个关键的决策方法 `webfinger_update_due?`:

```ruby
def webfinger_update_due?
  return false if @options[:check_delivery_availability] && !DeliveryFailureTracker.available?(@domain)
  return false if @options[:skip_webfinger]

  @options[:skip_cache] || @account.nil? || @account.possibly_stale?
end
```

**更新条件**:
1. **强制跳过缓存**: `skip_cache` 选项为 true
2. **本地无记录**: `@account.nil?`（首次查找）
3. **数据过期**: `@account.possibly_stale?`（超过 1 天未更新）

**跳过条件**:
1. **域传递失败**: 该域名最近有传递失败记录
2. **显式跳过**: `skip_webfinger` 选项为 true

---

## 3. 本地缓存的远端账号数据同步策略

### 3.1 数据存储模型

远端账号信息存储在 `accounts` 表中，使用与本地账号相同的模型。

**关键字段**:

| 字段 | 类型 | 用途 |
|------|------|------|
| `domain` | string | 远端域名，本地账号为 NULL |
| `last_webfingered_at` | datetime | 最后一次通过 WebFinger 验证的时间 |
| `uri` | string | ActivityPub actor URL |
| `inbox_url` | string | 收件箱 URL |
| `outbox_url` | string | 发件箱 URL |
| `shared_inbox_url` | string | 共享收件箱 URL |
| `followers_url` | string | 关注者列表 URL |
| `following_url` | string | 关注列表 URL |
| `protocol` | integer | 使用的协议 (ostatus/activitypub) |
| `public_key` | text | 公钥（旧格式） |
| `actor_type` | string | Actor 类型 (Person/Group/Organization 等) |

**模型关联**:
- `keypairs` - 多个密钥对（支持密钥轮换）
- `fields` - 个人资料字段（JSONB 格式）
- `tags` - 关联的标签
- `statuses` - 发布的嘟文
- `follows` - 关注关系
- 等等...

### 3.2 同步机制

Mastodon 采用多层次的同步机制，确保数据的一致性和新鲜度。

#### 3.2.1 完整同步 vs 增量同步

**完整同步**:
- 触发时机: 首次查找、后台刷新、手动刷新
- 执行流程:
  1. WebFinger 查询获取最新的 actor URL
  2. 获取完整的 ActivityPub actor 数据
  3. 验证数据一致性
  4. 更新所有字段（包括头像、头图等媒体资源）
  5. 更新 `last_webfingered_at` 时间戳

**增量同步**:
- 触发时机: 接收 ActivityPub `Update` 活动
- 执行流程:
  1. 解析收到的 `Update` 活动
  2. 更新相关字段
  3. 可能跳过某些验证（因为消息已签名）

#### 3.2.2 字段更新策略

在 `ActivityPub::ProcessAccountService` 中，字段更新被分为几个层次：

**立即协议属性** (`set_immediate_protocol_attributes!`):
```ruby
def set_immediate_protocol_attributes!
  @account.inbox_url               = valid_collection_uri(@json['inbox'])
  @account.outbox_url              = valid_collection_uri(@json['outbox'])
  @account.shared_inbox_url        = valid_collection_uri(@json['endpoints'].is_a?(Hash) ? @json['endpoints']['sharedInbox'] : @json['sharedInbox'])
  @account.followers_url           = valid_collection_uri(@json['followers'])
  @account.following_url           = valid_collection_uri(@json['following'])
  @account.url                     = url || @uri
  @account.uri                     = @uri
  @account.actor_type              = actor_type
  @account.created_at              = @json['published'] if @json['published'].present?
  @account.feature_approval_policy = feature_approval_policy if Mastodon::Feature.collections_enabled?
end
```
- 这些是协议层面的关键属性
- 每次同步都会更新

**立即属性** (`set_immediate_attributes!`):
```ruby
def set_immediate_attributes!
  @account.featured_collection_url = valid_collection_uri(@json['featured'])
  @account.collections_url         = valid_collection_uri(@json['featuredCollections'])
  @account.display_name            = (@json['name'] || '')[0...(Account::DISPLAY_NAME_LENGTH_HARD_LIMIT)]
  @account.note                    = (@json['summary'] || '')[0...(Account::NOTE_LENGTH_HARD_LIMIT)]
  @account.locked                  = @json['manuallyApprovesFollowers'] || false
  @account.fields                  = property_values || {}
  @account.also_known_as           = as_array(@json['alsoKnownAs'] || []).take(Account::ALSO_KNOWN_AS_HARD_LIMIT).map { |item| value_or_id(item) }
  @account.discoverable            = @json['discoverable'] || false
  @account.indexable               = @json['indexable'] || false
  @account.memorial                = @json['memorial'] || false
  @account.show_featured           = @json['showFeatured'] if @json.key?('showFeatured')
  @account.show_media              = @json['showMedia'] if @json.key?('showMedia')
  @account.show_media_replies      = @json['showRepliesInMedia'] if @json.key?('showRepliesInMedia')
  @account.attribution_domains     = as_array(@json['attributionDomains'] || []).take(Account::ATTRIBUTION_DOMAINS_HARD_LIMIT).map { |item| value_or_id(item) }
end
```
- 用户可见的属性
- 有长度限制保护

**可获取属性** (`set_fetchable_attributes!`):
```ruby
def set_fetchable_attributes!
  begin
    avatar_url, avatar_description = image_url_and_description('icon')
    @account.avatar_remote_url = avatar_url || '' unless skip_download?
    @account.avatar = nil if @account.avatar_remote_url.blank?
    @account.avatar_description = avatar_description || ''
  rescue Mastodon::UnexpectedResponseError, *Mastodon::HTTP_CONNECTION_ERRORS
    RedownloadAvatarWorker.perform_in(rand(PROCESSING_DELAY), @account.id)
  end
  begin
    header_url, header_description = image_url_and_description('image')
    @account.header_remote_url = header_url || '' unless skip_download?
    @account.header = nil if @account.header_remote_url.blank?
    @account.header_description = header_description || ''
  rescue Mastodon::UnexpectedResponseError, *Mastodon::HTTP_CONNECTION_ERRORS
    RedownloadHeaderWorker.perform_in(rand(PROCESSING_DELAY), @account.id)
  end
  @account.statuses_count    = outbox_total_items    if outbox_total_items.present?
  @account.following_count   = following_total_items if following_total_items.present?
  @account.followers_count   = followers_total_items if followers_total_items.present?
  @account.hide_collections  = following_private? || followers_private?
  @account.moved_to_account  = @json['movedTo'].present? ? moved_account : nil
end
```
- 包含媒体资源（头像、头图）
- 包含统计数据（嘟文数、关注数等）
- 失败时安排后台重试

### 3.3 同步触发点详解

Mastodon 有多个同步触发点，确保在不同场景下数据都能保持同步。

#### 3.3.1 用户操作触发

**搜索账号**:
- 当用户在搜索框中输入 `@username@domain` 时
- 触发 `ResolveAccountService` 执行完整查找流程

**提及远端账号**:
- 当用户在嘟文中提及远端账号时
- 如果本地缓存过期，会触发刷新

**访问远端账号主页**:
- 当用户访问 `/users/username@domain` 页面时
- 可能触发 `schedule_refresh_if_stale!` 检查

#### 3.3.2 后台任务触发

**定期刷新**:
- `AccountRefreshWorker` 按计划执行
- 只刷新超过 1 周未更新的账号
- 使用随机延迟分散负载

**媒体重试**:
- `RedownloadAvatarWorker` - 头像下载失败时重试
- `RedownloadHeaderWorker` - 头图下载失败时重试

**集合同步**:
- `ActivityPub::SynchronizeFeaturedCollectionWorker` - 同步精选嘟文
- `ActivityPub::SynchronizeFeaturedTagsCollectionWorker` - 同步精选标签

#### 3.3.3 ActivityPub 消息触发

**Update 活动**:
- 当远端账号更新资料时，会发送 `Update` 活动到所有关注者的收件箱
- 收件箱控制器处理并更新本地缓存

**Delete 活动**:
- 当远端账号被删除时，会发送 `Delete` 活动
- 本地账号会被标记为删除

**Move 活动**:
- 当远端账号迁移到新域名时，会发送 `Move` 活动
- 处理账号迁移逻辑

### 3.4 一致性保证机制

Mastodon 采用多种机制来保证数据一致性。

#### 3.4.1 WebFinger 验证循环

在 `ActivityPub::FetchRemoteActorService` 中实现了严格的验证：

```ruby
def check_webfinger!
  webfinger = Webfinger.new("acct:#{@username}@#{@domain}").perform
  confirmed_username, confirmed_domain = split_acct(webfinger.subject)

  if @username.casecmp(confirmed_username).zero? && @domain.casecmp(confirmed_domain).zero?
    raise Error, "Webfinger response for #{@username}@#{@domain} does not loop back to #{@uri}" if webfinger.self_link_href != @uri

    return
  end

  webfinger = Webfinger.new("acct:#{confirmed_username}@#{confirmed_domain}").perform
  @username, @domain = split_acct(webfinger.subject)

  raise Webfinger::RedirectError, "Too many webfinger redirects for URI #{@uri} (stopped at #{@username}@#{@domain})" unless confirmed_username.casecmp(@username).zero? && confirmed_domain.casecmp(@domain).zero?
  raise Error, "Webfinger response for #{@username}@#{@domain} does not loop back to #{@uri}" if webfinger.self_link_href != @uri
end
```

**验证流程**:
1. 从 ActivityPub actor 中提取用户名和域名
2. 执行 WebFinger 查询获取确认信息
3. **关键检查**: 确保 WebFinger 响应中的 `self` 链接指向获取的 actor URL
4. 处理重定向情况（最多一层重定向）
5. **最终验证**: 确保重定向后的信息一致

**为什么需要这个验证**:
- 防止域名劫持攻击
- 确保账号确实属于声明的域名
- 正确处理账号迁移

#### 3.4.2 分布式锁

多个地方使用 Redis 分布式锁防止并发更新：

**ResolveAccountService**:
```ruby
def fetch_account!
  with_redis_lock("resolve:#{@username}@#{@domain}") do
    @account = ActivityPub::FetchRemoteAccountService.new.call(actor_url, suppress_errors: @options[:suppress_errors])
  end

  @account
end
```

**ActivityPub::ProcessAccountService**:
```ruby
with_redis_lock("process_account:#{@uri}") do
  # ... 创建或更新账号 ...
end
```

**AccountRefreshWorker**:
```ruby
sidekiq_options queue: 'pull', retry: 3, dead: false, lock: :until_executed, lock_ttl: 1.day.to_i
```

**锁的作用**:
- 防止同一账号被并发更新
- 避免数据竞争和不一致状态
- 减少重复的网络请求

#### 3.4.3 重复账号处理

当发现重复账号时，会触发合并逻辑：

```ruby
def process_duplicate_accounts!
  return unless Account.where(uri: @account.uri).where.not(id: @account.id).exists?

  AccountMergingWorker.perform_async(@account.id)
end
```

**触发条件**:
- 多个账号记录具有相同的 `uri`（ActivityPub actor URL）
- 可能由于 WebFinger 重定向或域名变更导致

**处理方式**:
- 异步执行 `AccountMergingWorker`
- 将旧账号的关联数据迁移到新账号
- 可能包括：嘟文、关注关系、收藏、媒体附件等

### 3.5 特殊情况处理

#### 3.5.1 账号迁移 (movedTo)

当远端账号迁移到新域名时，Mastodon 会智能处理：

**检测**:
```ruby
@account.moved_to_account = @json['movedTo'].present? ? moved_account : nil
```

**处理**:
```ruby
def moved_account
  account = ActivityPub::TagManager.instance.uri_to_resource(@json['movedTo'], Account)
  account ||= ActivityPub::FetchRemoteAccountService.new.call(@json['movedTo'], break_on_redirect: true, request_id: @options[:request_id])
  account
end
```

**用户体验**:
- 显示迁移提示
- 可能自动重定向到新账号
- 保持关注关系的连续性

#### 3.5.2 账号删除 (GoneError)

当 WebFinger 查询返回 410 Gone 时：

```ruby
def process_webfinger!(uri)
  # ... WebFinger 查询 ...
rescue Webfinger::GoneError
  @gone = true
end
```

```ruby
if gone_from_origin? && not_yet_deleted?
  queue_deletion!
  return
end
```

```ruby
def queue_deletion!
  @account.suspend!(origin: :remote)
  AccountDeletionWorker.perform_async(@account.id, { 'reserve_username' => false, 'skip_activitypub' => true })
end
```

**处理流程**:
1. 标记账号为远程暂停
2. 安排异步删除任务
3. 清理相关数据但保留必要的墓碑记录

#### 3.5.3 暂停/恢复状态同步

```ruby
def set_suspension!
  return if @account.suspended? && @account.suspension_origin_local?

  if @account.suspended? && !@json['suspended']
    @account.unsuspend!
    @suspension_changed = true
  elsif !@account.suspended? && @json['suspended']
    @account.suspend!(origin: :remote)
    @suspension_changed = true
  end
end
```

```ruby
def after_suspension_change!
  if @account.suspended?
    Admin::SuspensionWorker.perform_async(@account.id)
  else
    Admin::UnsuspensionWorker.perform_async(@account.id)
  end
end
```

**注意**:
- 本地管理员的暂停决定优先于远程信号
- 远程暂停/恢复会触发相应的后台任务

---

## 4. 完整流程图

### 4.1 典型账号查找与同步流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Mastodon 远端账号查找与同步流程                              │
└─────────────────────────────────────────────────────────────────────────────┘

用户输入: @username@remote.domain
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. ResolveAccountService.call()                                              │
│    ┌─────────────────────────────────────────────────────────────────────┐  │
│    │ 1.1 解析输入，提取 username 和 domain                                │  │
│    │ 1.2 检查域名是否允许（不在黑名单中）                                   │  │
│    │ 1.3 在本地数据库查找 Account.find_remote(username, domain)         │  │
│    │ 1.4 检查是否需要 WebFinger 更新:                                      │  │
│    │     - 本地无记录？→ 需要                                              │  │
│    │     - 数据过期（>1天）？→ 需要                                        │  │
│    │     - 强制刷新？→ 需要                                                 │  │
│    └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼ 需要 WebFinger 更新？
         │
         ├── 否 ──────────────────────────────────────────────────────────────┐
         │                                                                       │
         │  直接返回本地缓存的账号                                                │
         │  (可能触发后台刷新检查)                                                │
         │                                                                       │
         └───────────────────────────────────────────────────────────────────────┘
         │
         ▼ 是
         │
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. 执行 WebFinger 查询                                                         │
│    ┌─────────────────────────────────────────────────────────────────────┐  │
│    │ 2.1 Webfinger.new("acct:username@domain").perform()               │  │
│    │ 2.2 构建标准 URL: https://domain/.well-known/webfinger?resource=   │  │
│    │ 2.3 发送 GET 请求，Accept: application/jrd+json, application/json  │  │
│    │ 2.4 处理响应:                                                         │  │
│    │     - 404 → 尝试 host-meta 备用机制                                  │  │
│    │     - 410 → 标记为已删除 (GoneError)                                │  │
│    │     - 200 → 解析 JSON 响应                                           │  │
│    │ 2.5 验证响应:                                                         │  │
│    │     - 必须有 subject 字段                                             │  │
│    │     - 必须有 ActivityPub 类型的 self 链接                            │  │
│    └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼ 响应包含什么？
         │
         ┌──────────────────────────────────────────────────────────────────────┐
         │ WebFinger 响应示例 (JRD 格式):                                         │
         │ {                                                                       │
         │   "subject": "acct:username@remote.domain",                           │
         │   "aliases": [                                                         │
         │     "https://remote.domain/users/username"                            │
         │   ],                                                                    │
         │   "links": [                                                           │
         │     {                                                                   │
         │       "rel": "self",                                                   │
         │       "type": "application/activity+json",                            │
         │       "href": "https://remote.domain/users/username"                 │
         │     }                                                                   │
         │   ]                                                                     │
         │ }                                                                       │
         └──────────────────────────────────────────────────────────────────────┘
         │
         ▼ 提取 self 链接 (ActivityPub actor URL)
         │
┌─────────────────────────────────────────────────────────────────────────────┐
│ 3. 处理重定向 (可选)                                                           │
│    ┌─────────────────────────────────────────────────────────────────────┐  │
│    │ 如果 WebFinger 响应中的 subject 与请求的账号不一致:                   │  │
│    │ 3.1 这表示账号可能已迁移到新的 username 或 domain                    │  │
│    │ 3.2 使用新的账号信息再次执行 WebFinger 查询                           │  │
│    │ 3.3 验证两次查询结果一致（防止无限重定向）                             │  │
│    │ 3.4 记录重定向信息，更新本地查找                                       │  │
│    └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼ 检查账号是否已从远端删除
         │
         ├── 是 (GoneError) ──────────────────────────────────────────────────┐
         │                                                                       │
         │  queue_deletion!()                                                    │
         │  ├── @account.suspend!(origin: :remote)                             │
         │  └── AccountDeletionWorker.perform_async()                          │
         │                                                                       │
         │  结束，返回 nil                                                        │
         │                                                                       │
         └───────────────────────────────────────────────────────────────────────┘
         │
         ▼ 否
         │
┌─────────────────────────────────────────────────────────────────────────────┐
│ 4. ActivityPub::FetchRemoteAccountService.call(actor_url)                   │
│    ┌─────────────────────────────────────────────────────────────────────┐  │
│    │ 4.1 检查是否为本地 URI（是的话直接返回本地账号）                       │  │
│    │ 4.2 获取 ActivityPub actor 数据:                                      │  │
│    │     - 发送 GET 请求到 actor_url                                        │  │
│    │     - Accept: application/activity+json, application/ld+json        │  │
│    │     - 解析 JSON-LD 响应                                                │  │
│    │ 4.3 验证数据:                                                          │  │
│    │     - 检查 JSON-LD context 是否有效                                    │  │
│    │     - 检查 actor 类型是否支持                                          │  │
│    │       (Person, Group, Organization, Application, Service)            │  │
│    │     - 检查是否有 preferredUsername 或 webfinger 属性                  │  │
│    └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
         │
┌─────────────────────────────────────────────────────────────────────────────┐
│ 5. WebFinger 验证循环 (关键安全检查)                                          │
│    ┌─────────────────────────────────────────────────────────────────────┐  │
│    │ 从 ActivityPub actor 数据中提取:                                       │  │
│    │ - @username = json['preferredUsername'] 或从 webfinger 属性解析      │  │
│    │ - @domain = 从 actor URL 解析的主机名                                 │  │
│    │                                                                       │
│    │ 5.1 执行 Webfinger.new("acct:@username@@domain").perform()         │  │
│    │ 5.2 比较响应中的 subject:                                             │  │
│    │     - 如果一致 → 继续检查 self 链接                                   │  │
│    │     - 如果不一致 → 可能是重定向，再次查询验证                          │  │
│    │ 5.3 关键验证:                                                          │  │
│    │     webfinger.self_link_href == @uri (actor URL)                     │  │
│    │     这确保了 WebFinger 响应确实指向我们获取的 actor                    │  │
│    │     防止域名劫持和欺骗攻击                                             │  │
│    └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼ 验证通过
         │
┌─────────────────────────────────────────────────────────────────────────────┐
│ 6. ActivityPub::ProcessAccountService.call(username, domain, json)         │
│    ┌─────────────────────────────────────────────────────────────────────┐  │
│    │ 获取分布式锁: with_redis_lock("process_account:#{@uri}")           │  │
│    │                                                                       │
│    │ 6.1 查找或创建账号:                                                    │  │
│    │     - @account = Account.find_remote(@username, @domain)           │  │
│    │     - 如果不存在 → create_account()                                  │  │
│    │                                                                       │
│    │ 6.2 更新账号数据:                                                      │  │
│    │     ├── update_account()                                              │  │
│    │     │   ├── @account.last_webfingered_at = Time.now.utc  ◄──── 关键 │  │
│    │     │   ├── set_suspension!()           # 处理暂停状态              │  │
│    │     │   ├── set_immediate_protocol_attributes!()                    │  │
│    │     │   │   ├── inbox_url, outbox_url, shared_inbox_url           │  │
│    │     │   │   ├── followers_url, following_url                        │  │
│    │     │   │   ├── uri, url, actor_type                                │  │
│    │     │   │   └── created_at                                           │  │
│    │     │   ├── set_fetchable_key!()         # 处理公钥                 │  │
│    │     │   ├── set_immediate_attributes!()                              │  │
│    │     │   │   ├── display_name, note, locked                         │  │
│    │     │   │   ├── fields, also_known_as, discoverable                │  │
│    │     │   │   └── memorial, show_featured, etc.                       │  │
│    │     │   └── set_fetchable_attributes!()                              │  │
│    │     │       ├── avatar_remote_url, header_remote_url               │  │
│    │     │       ├── statuses_count, following_count, followers_count   │  │
│    │     │       └── moved_to_account (账号迁移)                          │  │
│    │     └── @account.save_with_optional_media!()                        │  │
│    │                                                                       │
│    │ 6.3 处理额外数据:                                                      │  │
│    │     ├── process_tags()                    # 处理自定义 emoji         │  │
│    │     └── process_duplicate_accounts!()    # 处理重复账号             │  │
│    └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
         │
┌─────────────────────────────────────────────────────────────────────────────┐
│ 7. 状态变更后的回调处理                                                        │
│    ┌─────────────────────────────────────────────────────────────────────┐  │
│    │ 7.1 协议变更:                                                          │  │
│    │     after_protocol_change! → ActivityPub::PostUpgradeWorker         │  │
│    │                                                                       │
│    │ 7.2 密钥变更:                                                          │  │
│    │     after_key_change! → RefollowWorker (重新验证关注关系)           │  │
│    │                                                                       │
│    │ 7.3 暂停状态变更:                                                      │  │
│    │     after_suspension_change! → Admin::SuspensionWorker /            │  │
│    │                                    Admin::UnsuspensionWorker          │  │
│    │                                                                       │
│    │ 7.4 额外集合同步:                                                      │  │
│    │     ├── check_featured_collection! → SynchronizeFeaturedCollection  │  │
│    │     ├── check_featured_tags_collection! → SynchronizeFeaturedTags   │  │
│    │     └── check_links! → VerifyAccountLinksWorker (验证资料链接)      │  │
│    └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
         │
┌─────────────────────────────────────────────────────────────────────────────┐
│ 8. 返回更新后的账号对象                                                        │
│                                                                              │
│    现在可以:                                                                   │
│    - 显示在搜索结果中                                                          │
│    - 用于创建嘟文中的提及                                                      │
│    - 用于建立关注关系                                                          │
│    - 等等...                                                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 后台刷新流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        后台账号刷新流程                                        │
└─────────────────────────────────────────────────────────────────────────────┘

触发点 1: 访问账号时
=====================
当访问远端账号的记录时:
┌─────────────────────────────────────────────────────────────────────────────┐
│ account.schedule_refresh_if_stale!()                                         │
│                                                                              │
│ 检查条件:                                                                     │
│ last_webfingered_at <= BACKGROUND_REFRESH_INTERVAL.ago (1周)              │
│                                                                              │
│ 如果为真:                                                                     │
│ AccountRefreshWorker.perform_in(rand(REFRESH_DEADLINE), account.id)       │
│ (随机延迟 0-6 小时，分散负载)                                                 │
└─────────────────────────────────────────────────────────────────────────────┘

触发点 2: 定时任务 (sidekiq-cron 或类似机制)
===============================================
定期扫描需要刷新的账号:
┌─────────────────────────────────────────────────────────────────────────────┐
│ 查找条件:                                                                     │
│ Account.remote                                                               │
│        .where(last_webfingered_at: ...BACKGROUND_REFRESH_INTERVAL.ago)    │
│        .or(Account.remote.where(last_webfingered_at: nil))                 │
│                                                                              │
│ 为每个符合条件的账号:                                                          │
│ AccountRefreshWorker.perform_async(account.id)                             │
└─────────────────────────────────────────────────────────────────────────────┘

AccountRefreshWorker 执行流程:
===============================
┌─────────────────────────────────────────────────────────────────────────────┐
│ def perform(account_id)                                                       │
│   account = Account.find_by(id: account_id)                                  │
│   return if account.nil?                                                      │
│                                                                              │
│   # 双重检查: 确保确实需要刷新                                                 │
│   return if account.last_webfingered_at >                                    │
│              Account::BACKGROUND_REFRESH_INTERVAL.ago                        │
│                                                                              │
│   # 执行完整的解析流程                                                         │
│   ResolveAccountService.new.call(account)                                    │
│ end                                                                          │
│                                                                              │
│ Sidekiq 选项:                                                                 │
│ - queue: 'pull'                                                              │
│ - retry: 3 (失败重试 3 次)                                                   │
│ - dead: false (失败后不进入死信队列)                                          │
│ - lock: :until_executed (执行期间加锁)                                       │
│ - lock_ttl: 1.day (锁过期时间)                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 数据新鲜度决策流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    数据新鲜度与更新决策                                        │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ 关键时间阈值 (account.rb):                                                    │
│                                                                              │
│ STALE_THRESHOLD = 1.day                                                      │
│   └── 超过此时间未更新 → 被认为"可能过期" (possibly_stale?)                │
│                                                                              │
│ BACKGROUND_REFRESH_INTERVAL = 1.week                                         │
│   └── 超过此时间未更新 → 触发后台刷新 (schedule_refresh_if_stale!)         │
│                                                                              │
│ REFRESH_DEADLINE = 6.hours                                                   │
│   └── 后台刷新任务的随机延迟范围                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ webfinger_update_due?() 决策逻辑 (ResolveAccountService):                   │
│                                                                              │
│ 是否需要执行 WebFinger 更新？                                                 │
│                                                                              │
│ 返回 false 的情况（跳过更新）:                                                │
│ ┌─────────────────────────────────────────────────────────────────────────┐│
│ │ 1. @options[:check_delivery_availability] &&                            ││
│ │    !DeliveryFailureTracker.available?(@domain)                          ││
│ │    └── 该域名最近有传递失败，暂时跳过                                      ││
│ │                                                                          ││
│ │ 2. @options[:skip_webfinger]                                             ││
│ │    └── 显式要求跳过 WebFinger 查询                                        ││
│ └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│ 返回 true 的情况（需要更新）:                                                 │
│ ┌─────────────────────────────────────────────────────────────────────────┐│
│ │ 1. @options[:skip_cache]                                                 ││
│ │    └── 强制跳过缓存，获取最新数据                                          ││
│ │                                                                          ││
│ │ 2. @account.nil?                                                         ││
│ │    └── 本地无记录，首次查找                                                ││
│ │                                                                          ││
│ │ 3. @account.possibly_stale?                                             ││
│ │    └── 数据过期（last_webfingered_at > 1 天 或为 nil）                  ││
│ └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ 场景示例:                                                                      │
│                                                                              │
│ 场景 1: 用户首次搜索 @alice@example.com                                      │
│ ─────────────────────────────────────                                        │
│ @account = nil → webfinger_update_due? = true                               │
│ → 执行完整的 WebFinger + ActivityPub 获取流程                                │
│ → 创建本地账号记录                                                            │
│ → last_webfingered_at = 当前时间                                             │
│                                                                              │
│ 场景 2: 12 小时后再次访问 @alice@example.com                                 │
│ ────────────────────────────────────────                                     │
│ last_webfingered_at = 12.hours.ago                                           │
│ possibly_stale? = false (12h < 1day)                                         │
│ webfinger_update_due? = false                                                │
│ → 直接使用本地缓存，不更新                                                    │
│ → schedule_refresh_if_stale!: 12h < 1week → 不安排后台刷新                 │
│                                                                              │
│ 场景 3: 3 天后再次访问 @alice@example.com                                    │
│ ──────────────────────────────────────                                       │
│ last_webfingered_at = 3.days.ago                                             │
│ possibly_stale? = true (3d > 1day)                                           │
│ webfinger_update_due? = true                                                 │
│ → 执行完整的更新流程                                                          │
│ → 更新 last_webfingered_at                                                   │
│                                                                              │
│ 场景 4: 10 天后再次访问 @alice@example.com                                   │
│ ───────────────────────────────────────                                      │
│ last_webfingered_at = 10.days.ago                                            │
│ schedule_refresh_if_stale!: 10d > 1week → 安排后台刷新                     │
│ → AccountRefreshWorker.perform_in(rand(6h), account.id)                    │
│ → 用户当前请求使用缓存数据，但后台会在 0-6 小时内刷新                        │
│ → 如果用户等待足够长时间，下次访问将看到更新后的数据                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 关键代码位置汇总

| 功能 | 文件路径 | 关键方法/类 |
|------|----------|-------------|
| WebFinger 客户端 | `app/lib/webfinger.rb` | `Webfinger` 类, `perform()` 方法 |
| WebFinger 服务端控制器 | `app/controllers/well_known/webfinger_controller.rb` | `WebfingerController` |
| WebFinger 资源解析 | `app/lib/webfinger_resource.rb` | `WebfingerResource` 类 |
| 账号解析主服务 | `app/services/resolve_account_service.rb` | `ResolveAccountService` 类 |
| ActivityPub Actor 获取 | `app/services/activitypub/fetch_remote_actor_service.rb` | `FetchRemoteActorService` 类 |
| ActivityPub 账号获取 | `app/services/activitypub/fetch_remote_account_service.rb` | `FetchRemoteAccountService` 类 |
| 账号数据处理与存储 | `app/services/activitypub/process_account_service.rb` | `ProcessAccountService` 类 |
| 账号模型 | `app/models/account.rb` | `Account` 类, `possibly_stale?`, `refresh!` 等 |
| 后台刷新任务 | `app/workers/account_refresh_worker.rb` | `AccountRefreshWorker` 类 |

---

## 6. 总结

Mastodon 的远端账号发现与同步机制是一个设计精巧的多层架构，它成功地解决了联邦式社交网络中的几个关键问题：

### 6.1 WebFinger 的核心作用

WebFinger 协议在 Mastodon 中扮演着不可或缺的角色：

1. **用户友好的标识符**: 让用户可以使用 `@username@domain` 这种自然的格式来引用其他实例上的用户
2. **发现层**: 作为连接用户标识符和机器可读的 ActivityPub URL 的桥梁
3. **安全验证**: 通过验证循环确保账号身份的真实性，防止域名劫持
4. **迁移支持**: 优雅地处理用户在不同实例间迁移账号的情况

### 6.2 更新策略的平衡艺术

Mastodon 在数据新鲜度和系统负载之间取得了良好的平衡：

1. **分层过期机制**:
   - 1 天阈值用于决定是否在用户请求时同步更新
   - 1 周阈值用于触发后台刷新任务

2. **多触发点**:
   - 首次查找时强制获取
   - 访问时检查并按需安排后台刷新
   - 接收 ActivityPub Update 活动时实时更新

3. **性能优化**:
   - 分布式锁防止并发更新
   - 随机延迟分散后台任务负载
   - 失败重试机制
   - 缓存策略减少重复请求

### 6.3 一致性与可靠性保证

Mastodon 采用多种机制确保数据一致性：

1. **WebFinger 验证循环**: 这是最关键的安全检查，确保获取的数据确实属于目标域名
2. **分布式锁**: 在多个关键点使用 Redis 锁防止数据竞争
3. **重复账号处理**: 能够检测并合并由于重定向或域名变更导致的重复记录
4. **状态同步**: 正确处理账号的暂停、恢复、删除和迁移等状态变更

### 6.4 容错与弹性设计

系统设计考虑了多种故障场景：

1. **WebFinger 故障转移**: 当标准端点失败时，尝试 host-meta 作为备用
2. **媒体下载重试**: 头像和头图下载失败时安排后台重试
3. **传递失败检测**: 当某个域名持续传递失败时，暂时跳过该域名的 WebFinger 查询
4. **优雅降级**: 当远端服务不可用时，继续使用本地缓存数据

这种设计使得 Mastodon 能够在不稳定的联邦网络环境中保持良好的用户体验，同时确保数据的安全性和一致性。

通过这些机制的组合，Mastodon 成功实现了一个既用户友好又技术可靠的分布式社交网络，允许用户跨不同实例自由交互，同时保持对自己数据的控制。
