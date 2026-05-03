# 账号发现机制与远端资料同步分析报告

## 目录
1. [WebFinger 查询机制](#1-webfinger-查询机制)
2. [数据缓存策略](#2-数据缓存策略)
3. [更新时机与一致性保障](#3-更新时机与一致性保障)
4. [关键数据流](#4-关键数据流)
5. [异常处理与边界情况](#5-异常处理与边界情况)

---

## 1. WebFinger 查询机制

### 1.1 核心实现

WebFinger 是 Mastodon 用于发现远程用户的核心协议，实现在 `app/lib/webfinger.rb` 中。

**主要类结构：**

- `Webfinger::Response` - 封装 WebFinger 响应的解析和验证
- `Webfinger` - 执行实际的 WebFinger 查询

### 1.2 查询流程

WebFinger 查询采用标准协议 + host-meta 回退的双层机制：

```
┌─────────────────────────────────────────────────────────────────┐
│                    WebFinger 查询流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. 标准 URL 请求                                                 │
│     URL: https://{domain}/.well-known/webfinger?resource={uri} │
│     Accept: application/jrd+json, application/json               │
│                                                                   │
│         ↓ HTTP 200?                                              │
│         ├─ Yes → 解析 JRD 响应                                   │
│         │                                                         │
│         └─ No (404) → 2. host-meta 回退                         │
│                        ↓                                          │
│                        URL: https://{domain}/.well-known/host-meta│
│                        Accept: application/xrd+xml, application/xml│
│                        ↓                                          │
│                        从 XML 中提取 lrdd 链接模板                │
│                        ↓                                          │
│                        使用模板构造 WebFinger URL                 │
│                        重新请求                                   │
│                                                                   │
│  注意: .onion 域名使用 HTTP 而非 HTTPS                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

**关键代码位置：**
- `app/lib/webfinger.rb:68-126` - 标准 URL 构造和请求逻辑
- `app/lib/webfinger.rb:85-101` - host-meta 回退机制

### 1.3 响应验证

WebFinger 响应必须满足以下条件：

1. **Subject 存在** - `subject` 字段不能为空
2. **Self Link 存在** - 必须包含 `rel="self"` 的链接，且类型为 ActivityPub 兼容类型

**支持的 Self Link 类型：**
- `application/activity+json`
- `application/ld+json; profile="https://www.w3.org/ns/activitystreams"`

**代码位置：** `app/lib/webfinger.rb:44-47`

### 1.4 重定向处理

在 `ResolveAccountService` 中实现了账号重定向检测：

```ruby
# app/services/resolve_account_service.rb:82-99
def process_webfinger!(uri)
  @webfinger = Webfinger.new("acct:#{uri}").perform
  confirmed_username, confirmed_domain = split_acct(@webfinger.subject)

  # 如果返回的账号与请求的不同，说明发生了重定向
  if confirmed_username.casecmp(@username).zero? && confirmed_domain.casecmp(@domain).zero?
    @username = confirmed_username
    @domain   = confirmed_domain
    return
  end

  # 对重定向后的账号再次执行 WebFinger 查询
  @webfinger = Webfinger.new("acct:#{confirmed_username}@#{confirmed_domain}").perform
  @username, @domain = split_acct(@webfinger.subject)

  # 防止无限重定向
  raise Webfinger::RedirectError, "Too many webfinger redirects..." unless confirmed_username.casecmp(@username).zero?
end
```

### 1.5 特殊状态码处理

- **410 Gone** - 表示账号已在源服务器删除，触发本地删除流程
- **404 Not Found** - 触发 host-meta 回退机制

---

## 2. 数据缓存策略

### 2.1 多级缓存架构

Mastodon 采用多级缓存机制来减少对远端服务器的请求：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        多级缓存架构                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  层级 1: 内存缓存 (EntityCache)                                       │
│  ├── 缓存类型: Rails.cache                                            │
│  ├── 过期时间: MAX_EXPIRATION = 7 天                                 │
│  └── 缓存内容:                                                        │
│      ├── Mention 账号引用 (username + domain)                        │
│      ├── Status 状态 (url)                                            │
│      └── CustomEmoji 自定义表情 (shortcode + domain)                │
│                                                                       │
│  层级 2: 数据库字段 (last_webfingered_at)                            │
│  ├── 字段类型: datetime                                                │
│  ├── 作用: 标记上次 WebFinger 查询时间                                │
│  └── 用于: 决定是否需要刷新远端资料                                   │
│                                                                       │
│  层级 3: JSON-LD Context 缓存                                         │
│  ├── 缓存类型: Rails.cache                                            │
│  └── 过期时间: 30 天                                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 EntityCache 实现

`EntityCache` 是一个单例类，使用 Rails.cache 进行缓存：

```ruby
# app/lib/entity_cache.rb:14-16
def mention(username, domain)
  Rails.cache.fetch(to_key(:mention, username, domain), expires_in: MAX_EXPIRATION) do
    Account.select(:id, :username, :domain, :url).find_remote(username, domain)
  end
end
```

**缓存键构造：**
- 格式: `{type}:{ids.join(':')}`
- 示例: `mention:alice:example.com`

**代码位置：** `app/lib/entity_cache.rb:39-41`

### 2.3 账号引用缓存场景

`EntityCache.mention` 主要在以下场景被调用：

1. **文本格式解析** - `app/lib/text_formatter.rb` 中的提及解析
2. **账号搜索** - `app/services/account_search_service.rb` 中的精确匹配

**代码位置：** `app/models/account.rb:435-444`

```ruby
def self.from_text(text)
  # ...
  text.scan(MENTION_RE).map do |match|
    # ...
    EntityCache.instance.mention(username, domain)
  end
end
```

### 2.4 JSON-LD Context 缓存

在 `JsonLdHelper` 中，JSON-LD 上下文被缓存 30 天：

```ruby
# app/helpers/json_ld_helper.rb:337-350
def load_jsonld_context(url, _options = {}, &block)
  json = Rails.cache.fetch("jsonld:context:#{url}", expires_in: 30.days, raw: true) do
    # 实际请求获取 context
  end
  # ...
end
```

---

## 3. 更新时机与一致性保障

### 3.1 刷新阈值配置

账号模型定义了三个关键时间阈值：

| 常量名 | 值 | 用途 | 代码位置 |
|--------|-----|------|----------|
| `BACKGROUND_REFRESH_INTERVAL` | 1 周 | 后台刷新间隔 | `app/models/account.rb:76` |
| `REFRESH_DEADLINE` | 6 小时 | 刷新任务延迟上限 | `app/models/account.rb:77` |
| `STALE_THRESHOLD` | 1 天 | 数据陈旧阈值 | `app/models/account.rb:78` |

### 3.2 刷新决策逻辑

```ruby
# app/models/account.rb:264-272

# 判断数据是否陈旧（需要立即更新）
def possibly_stale?
  last_webfingered_at.nil? || last_webfingered_at <= STALE_THRESHOLD.ago
end

# 调度后台刷新（如果超过一周未更新）
def schedule_refresh_if_stale!
  return unless last_webfingered_at.present? && last_webfingered_at <= BACKGROUND_REFRESH_INTERVAL.ago
  
  AccountRefreshWorker.perform_in(rand(REFRESH_DEADLINE), id)
end
```

**刷新决策流程图：**

```
┌──────────────────────────────────────────────────────────────────┐
│                    刷新决策逻辑                                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  调用 ResolveAccountService                                        │
│         ↓                                                          │
│  webfinger_update_due?                                             │
│         ↓                                                          │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ 条件判断:                                                       │ │
│  │ - @options[:skip_cache] == true? → 需要更新                   │ │
│  │ - @account.nil? (新账号)? → 需要更新                          │ │
│  │ - @account.possibly_stale? (> 1 天未更新)? → 需要更新         │ │
│  └──────────────────────────────────────────────────────────────┘ │
│         ↓                                                          │
│         ├─ Yes → 执行 WebFinger 查询 + ActivityPub 资料获取      │
│         │         ↓                                                │
│         │         更新 last_webfingered_at = Time.now.utc         │
│         │                                                          │
│         └─ No → 直接返回缓存数据                                   │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

**代码位置：** `app/services/resolve_account_service.rb:115-120`

### 3.3 分布式锁机制

为防止并发更新同一账号，Mastodon 使用 Redis 分布式锁：

**锁实现：** `app/models/concerns/lockable.rb`

```ruby
def with_redis_lock(lock_name, autorelease: 15.minutes, raise_on_failure: true)
  with_redis do |redis|
    RedisLock.acquire(redis: redis, key: "lock:#{lock_name}", autorelease: autorelease.seconds) do |lock|
      if lock.acquired?
        yield
      elsif raise_on_failure
        raise Mastodon::RaceConditionError, "Could not acquire lock for #{lock_name}"
      end
    end
  end
end
```

**锁的使用场景：**

| 操作 | 锁键格式 | 自动释放时间 | 代码位置 |
|------|----------|--------------|----------|
| 解析账号 | `resolve:{username}@{domain}` | 默认 15 分钟 | `app/services/resolve_account_service.rb:108` |
| 处理账号 | `process_account:{uri}` | 默认 15 分钟 | `app/services/activitypub/process_account_service.rb:34` |
| 创建状态 | `create:{object_uri}` | 默认 15 分钟 | `app/lib/activitypub/activity/create.rb:20` |

### 3.4 重复账号检测与合并

**检测时机：** 当 WebFinger 验证通过后（`verified_webfinger: true`）

**触发条件：** 存在其他账号具有相同的 `uri`

**代码位置：** `app/services/activitypub/process_account_service.rb:232-236`

```ruby
def process_duplicate_accounts!
  return unless Account.where(uri: @account.uri).where.not(id: @account.id).exists?
  
  AccountMergingWorker.perform_async(@account.id)
end
```

**合并流程：** `app/workers/account_merging_worker.rb` + `app/models/concerns/account/merging.rb`

```
┌─────────────────────────────────────────────────────────────────────┐
│                      账号合并流程                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. 发现重复账号                                                       │
│     Account.where(uri: account.uri).where.not(id: account.id)      │
│                                                                       │
│         ↓                                                             │
│  2. 执行合并                                                           │
│     account.merge_with!(duplicate)                                   │
│                                                                       │
│         ↓                                                             │
│  3. 迁移关联数据                                                       │
│     ├── 作为拥有者的记录: Status, MediaAttachment, Poll, etc.       │
│     ├── 作为发送者的记录: Notification, NotificationRequest          │
│     ├── 作为目标的记录: Follow, Block, Mute, etc.                   │
│     ├── 其他关联: CanonicalEmailBlock, Appeal, SeveredRelationship │
│     └── 注意: 使用 rescue RecordNotUnique 跳过重复记录              │
│                                                                       │
│         ↓                                                             │
│  4. 清理缓存                                                           │
│     Rails.cache.delete_matched("followers_hash:#{id}:*")            │
│     Rails.cache.delete_matched("relationships:#{id}:*")             │
│                                                                       │
│         ↓                                                             │
│  5. 删除重复账号                                                       │
│     duplicate.destroy                                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.5 异步刷新任务

Mastodon 使用 Sidekiq 异步任务处理账号刷新：

**1. AccountRefreshWorker** - 后台定期刷新

```ruby
# app/workers/account_refresh_worker.rb:8-13
def perform(account_id)
  account = Account.find_by(id: account_id)
  # 双重检查：防止任务执行时数据已被刷新
  return if account.nil? || account.last_webfingered_at > Account::BACKGROUND_REFRESH_INTERVAL.ago
  
  ResolveAccountService.new.call(account)
end
```

**2. RemoteAccountRefreshWorker** - 从 ActivityPub URI 直接刷新

```ruby
# app/workers/remote_account_refresh_worker.rb:10-19
def perform(id)
  account = Account.find_by(id: id)
  return if account.nil? || account.local?
  
  ActivityPub::FetchRemoteAccountService.new.call(account.uri)
rescue Mastodon::UnexpectedResponseError => e
  # 处理不可恢复的错误（如 404, 410 等）
  response = e.response
  raise(e) unless response_error_unsalvageable?(response)
end
```

**3. ResolveAccountWorker** - 延迟解析账号

用于在处理提及或其他延迟场景时解析账号。

---

## 4. 关键数据流

### 4.1 账号发现完整流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    完整账号发现与同步流程                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  入口: 用户搜索 @username@domain 或处理提及                               │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Step 1: ResolveAccountService                                    │   │
│  │                                                                  │   │
│  │  1.1 解析 username 和 domain                                     │   │
│  │  1.2 检查本地是否已存在账号 (Account.find_remote)                │   │
│  │  1.3 判断是否需要 WebFinger 更新 (webfinger_update_due?)        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Step 2: Webfinger 查询 (process_webfinger!)                     │   │
│  │                                                                  │   │
│  │  2.1 构造 acct: 格式 URI                                        │   │
│  │  2.2 请求 /.well-known/webfinger                                │   │
│  │  2.3 验证响应 (subject, self link)                              │   │
│  │  2.4 处理账号重定向 (最多 1 次跳转)                              │   │
│  │  2.5 检测 410 Gone → 触发账号删除                               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Step 3: ActivityPub 资料获取                                     │   │
│  │         (with_redis_lock 保护)                                   │   │
│  │                                                                  │   │
│  │  3.1 FetchRemoteAccountService                                   │   │
│  │      - 从 Webfinger self link 获取 actor URL                    │   │
│  │      - 请求 ActivityPub JSON                                     │   │
│  │      - 验证 context 和 type                                      │   │
│  │      - 可选: 二次 WebFinger 验证 (check_webfinger!)             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Step 4: 账号处理与更新                                           │   │
│  │         (with_redis_lock 保护)                                   │   │
│  │                                                                  │   │
│  │  4.1 ProcessAccountService                                       │   │
│  │      ├── 创建或更新 Account 记录                                 │   │
│  │      ├── 更新 last_webfingered_at                                │   │
│  │      ├── 设置协议属性 (inbox, outbox, 公钥等)                   │   │
│  │      ├── 设置个人资料 (display_name, note, fields 等)            │   │
│  │      ├── 下载头像/横幅 (可选延迟处理)                             │   │
│  │      ├── 处理 Emoji                                              │   │
│  │      └── 检测重复账号 → 触发合并                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Step 5: 异步后续任务                                             │   │
│  │                                                                  │   │
│  │  5.1 协议变更 → PostUpgradeWorker                                │   │
│  │  5.2 密钥变更 → RefollowWorker (重新验证关注关系)                 │   │
│  │  5.3 精选集合 → SynchronizeFeaturedCollectionWorker              │   │
│  │  5.4 链接验证 → VerifyAccountLinksWorker (延迟 10 分钟)         │   │
│  │  5.5 媒体下载失败 → RedownloadAvatar/HeaderWorker (延迟重试)     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 入口场景

账号发现可从多个入口触发：

| 入口 | 触发方式 | 代码位置 |
|------|----------|----------|
| **账号搜索** | `AccountSearchService` with `resolve: true` | `app/services/account_search_service.rb:206-208` |
| **处理提及** | `ProcessMentionsService` → `ResolveAccountWorker` | `app/services/process_mentions_service.rb` |
| **后台刷新** | `AccountRefreshWorker` (定时触发) | `app/models/account.rb:268-272` |
| **手动刷新** | `Account#refresh!` | `app/models/account.rb:274-276` |
| **ActivityPub 处理** | 接收 Update 活动 | `app/lib/activitypub/activity/update.rb` |
| **URL 解析** | `ResolveURLService` | `app/services/resolve_url_service.rb` |
| **管理后台** | 管理员强制刷新 | `app/controllers/admin/accounts_controller.rb:103` |

---

## 5. 异常处理与边界情况

### 5.1 WebFinger 异常处理

| 异常类型 | 触发条件 | 处理方式 | 代码位置 |
|----------|----------|----------|----------|
| `Webfinger::GoneError` | 远端返回 410 | 挂起账号 + 异步删除 | `app/lib/webfinger.rb:77-78` |
| `Webfinger::RedirectError` | 重定向次数过多 | 抛异常或静默失败 | `app/services/resolve_account_service.rb:96` |
| `JSON::ParserError` | 响应 JSON 无效 | 抛异常或静默失败 | `app/lib/webfinger.rb:60-61` |
| `Addressable::URI::InvalidURIError` | URL 格式无效 | 抛异常或静默失败 | `app/lib/webfinger.rb:62-63` |

**账号删除流程：**

```ruby
# app/services/resolve_account_service.rb:134-137
def queue_deletion!
  @account.suspend!(origin: :remote)
  AccountDeletionWorker.perform_async(@account.id, { 'reserve_username' => false, 'skip_activitypub' => true })
end
```

### 5.2 一致性保障机制

**1. 双重检查锁定**

在 `AccountRefreshWorker` 中，任务执行前再次检查刷新阈值：

```ruby
# app/workers/account_refresh_worker.rb:10
return if account.nil? || account.last_webfingered_at > Account::BACKGROUND_REFRESH_INTERVAL.ago
```

这防止了以下竞态条件：
- 任务 1 调度刷新，设置延迟时间为 3 小时
- 在这 3 小时内，用户手动触发了刷新
- 任务 1 执行时，数据已经是新的，跳过即可

**2. 操作幂等性**

账号更新操作设计为幂等：
- `ProcessAccountService` 可以安全地重复调用
- 使用 `save_with_optional_media!` 处理媒体下载失败
- 关联记录更新使用 `rescue ActiveRecord::RecordNotUnique` 跳过重复

**3. 分布式锁+数据库事务**

关键更新操作同时使用：
1. **Redis 锁** (`with_redis_lock`) - 防止并发执行
2. **数据库事务** (Rails 隐式事务) - 确保数据一致性

**4. last_webfingered_at 更新时机**

该字段在以下场景更新：

| 场景 | 更新值 | 代码位置 |
|------|--------|----------|
| 正常处理账号 | `Time.now.utc` | `app/services/activitypub/process_account_service.rb:102` |
| 管理员强制刷新 | `nil` (触发下次全量刷新) | `app/controllers/admin/accounts_controller.rb:103` |
| 账号取消挂起 | `nil` | `app/services/unsuspend_account_service.rb:31` |
| 签名验证失败 | `nil` (触发重新验证) | `app/controllers/activitypub/inboxes_controller.rb:56` |

### 5.3 限流与防护

**ProcessAccountService 中的限流措施：**

```ruby
# app/services/activitypub/process_account_service.rb:48-57
if @account.nil?
  with_redis do |redis|
    # 子域名限制：每个公共后缀最多 10 个新账号
    return nil if redis.pfcount("unique_subdomains_for:#{PublicSuffix.domain(@domain, ignore_private: true)}") >= SUBDOMAINS_RATELIMIT
    
    # 单次请求限制：最多 400 个新发现
    discoveries = redis.incr("discovery_per_request:#{@options[:request_id]}")
    redis.expire("discovery_per_request:#{@options[:request_id]}", 5.minutes.seconds)
    return nil if discoveries > DISCOVERIES_PER_REQUEST
  end
end
```

**限流常量：**
- `SUBDOMAINS_RATELIMIT = 10` - 每个公共后缀的子域名限制
- `DISCOVERIES_PER_REQUEST = 400` - 单次请求的最大发现数

### 5.4 密钥变更处理

当账号的所有公钥都发生变化时，系统会触发特殊处理：

```ruby
# app/services/activitypub/process_account_service.rb:390-392
def all_public_keys_changed?
  !@old_public_keys.empty? && @account.keypairs.none? { |keypair| keypair.usable? && @old_public_keys.include?(keypair.public_key) }
end

# 触发的后续处理 (app/services/activitypub/process_account_service.rb:67-69)
after_key_change! if all_public_keys_changed? && !@options[:signed_with_known_key]
clear_tombstones! if all_public_keys_changed?
```

**处理逻辑：**
1. **RefollowWorker** - 重新验证关注关系（因为密钥变更可能意味着账号被盗）
2. **清除 Tombstones** - 删除旧的状态墓碑记录

---

## 附录：关键文件索引

| 文件路径 | 功能描述 |
|----------|----------|
| `app/lib/webfinger.rb` | WebFinger 协议实现 |
| `app/lib/entity_cache.rb` | 实体缓存单例 |
| `app/models/account.rb` | 账号模型（含刷新逻辑） |
| `app/models/concerns/account/finder_concern.rb` | 账号查找方法 |
| `app/models/concerns/account/merging.rb` | 账号合并逻辑 |
| `app/models/concerns/lockable.rb` | 分布式锁 concern |
| `app/services/resolve_account_service.rb` | 账号解析主服务 |
| `app/services/account_search_service.rb` | 账号搜索服务 |
| `app/services/activitypub/fetch_remote_account_service.rb` | 远程账号获取服务 |
| `app/services/activitypub/process_account_service.rb` | 账号数据处理服务 |
| `app/workers/account_refresh_worker.rb` | 后台刷新任务 |
| `app/workers/remote_account_refresh_worker.rb` | 远程账号刷新任务 |
| `app/workers/account_merging_worker.rb` | 账号合并任务 |
| `app/helpers/json_ld_helper.rb` | JSON-LD 处理辅助方法 |

---

*报告生成时间：2026-05-03*
*分析基于 Mastodon 代码库版本：当前工作目录版本*
