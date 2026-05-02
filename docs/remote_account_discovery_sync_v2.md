# Mastodon 远端用户账号发现与同步机制分析（修正版）

> **重要提示**: 本文档严格区分**代码事实**（可通过代码直接验证）与**合理推断**（基于代码逻辑的推导）。所有关于"不存在"的陈述均基于对代码库的全面检查。

---

## 1. WebFinger 协议在账号查找中的作用

### 1.1 WebFinger 协议概述（代码事实）

WebFinger 是一种用于在互联网上发现资源信息的协议，它使用简单的 HTTP 请求和 JSON 响应格式。在 Mastodon 中，WebFinger 是连接**用户友好的账号格式**与**机器可读的 ActivityPub URL** 的核心桥梁。

**核心作用**：
- 将 `@username@domain` 格式解析为 ActivityPub actor URL
- 验证账号身份的真实性
- 处理账号重定向

### 1.2 Mastodon 中的 WebFinger 实现（代码事实）

Mastodon 实现了完整的 WebFinger 协议栈，包括客户端和服务端两部分。

#### 1.2.1 Webfinger 类 - 客户端实现

**文件位置**: `app/lib/webfinger.rb`

**核心功能**：

1. **标准 WebFinger 请求**:
```ruby
# app/lib/webfinger.rb:111-117
def standard_url
  if @domain.end_with? '.onion'
    "http://#{@domain}/.well-known/webfinger?resource=#{@uri}"
  else
    "https://#{@domain}/.well-known/webfinger?resource=#{@uri}"
  end
end
```

2. **Host-Meta 备用机制**（当标准端点返回 404 时）:
```ruby
# app/lib/webfinger.rb:85-101
def body_from_host_meta
  host_meta_request.perform do |res|
    raise Webfinger::Error, "Request for #{@uri} returned HTTP #{res.code}" unless res.code == 200

    body_from_webfinger(url_from_template(res.body_with_limit), use_fallback: false)
  end
end
```

3. **响应验证**（确保响应包含必要字段）:
```ruby
# app/lib/webfinger.rb:44-47
def validate_response!
  raise Webfinger::Error, "Missing subject in response for #{@uri}" if subject.blank?
  raise Webfinger::Error, "Missing self link in response for #{@uri}" if self_link.blank?
end
```

#### 1.2.2 WebfingerController - 服务端实现

**文件位置**: `app/controllers/well_known/webfinger_controller.rb`

处理来自其他服务器的 WebFinger 请求：
```ruby
# app/controllers/well_known/webfinger_controller.rb:13-16
def show
  expires_in 3.days, public: true
  render json: @account, serializer: WebfingerSerializer, content_type: 'application/jrd+json'
end
```

#### 1.2.3 WebfingerResource - 资源解析器

**文件位置**: `app/lib/webfinger_resource.rb`

支持多种资源格式：
- Instance Actor URL
- 标准 URL（如 `https://domain/users/username`）
- Acct 格式（如 `acct:username@domain` 或 `username@domain`）

---

## 2. 账号查找与资料获取流程（代码事实）

### 2.1 完整的账号解析流程

Mastodon 使用多层服务架构来解析远端账号：

```
用户输入: @username@remote.domain
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. ResolveAccountService.call()                              │
│    - 解析输入，提取 username 和 domain                       │
│    - 检查本地是否已有记录: Account.find_remote(username,    │
│      domain)                                                 │
│    - 检查是否需要 WebFinger 更新: webfinger_update_due?()   │
└─────────────────────────────────────────────────────────────┘
         │
         ▼ 需要更新？
         │
         ├── 否 ──► 直接返回本地缓存
         │
         ▼ 是
         │
┌─────────────────────────────────────────────────────────────┐
│ 2. Webfinger 查询                                            │
│    - 向 remote.domain 发送 Webfinger 请求                    │
│    - 从响应中提取 ActivityPub actor URL (self link)          │
│    - 处理可能的账号重定向                                      │
│    - 检测账号删除 (HTTP 410 Gone)                            │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. ActivityPub::FetchRemoteAccountService.call(actor_url)   │
│    - 获取 ActivityPub actor JSON-LD 数据                     │
│    - 验证数据格式和类型                                        │
│    - 关键步骤: check_webfinger!() - WebFinger 验证循环       │
│      → 确保获取的 actor 确实与 WebFinger 响应匹配            │
│      → 防止域名劫持攻击                                       │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. ActivityPub::ProcessAccountService.call()                 │
│    - 获取分布式锁: with_redis_lock("process_account:#{uri}")│
│    - 创建或更新账号记录                                        │
│    - 更新 last_webfingered_at = Time.now.utc                │
│    - 处理公钥、头像、头图等媒体资源                            │
│    - 处理标签、精选集合等额外数据                              │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
   返回更新后的账号对象
```

### 2.2 关键服务详解

#### ResolveAccountService - 主入口

**文件位置**: `app/services/resolve_account_service.rb`

**核心决策逻辑**:
```ruby
# app/services/resolve_account_service.rb:115-120
def webfinger_update_due?
  return false if @options[:check_delivery_availability] && !DeliveryFailureTracker.available?(@domain)
  return false if @options[:skip_webfinger]

  @options[:skip_cache] || @account.nil? || @account.possibly_stale?
end
```

**需要执行 WebFinger 更新的条件**：
1. `skip_cache: true` - 强制跳过缓存
2. `@account.nil?` - 本地无记录（首次查找）
3. `@account.possibly_stale?` - 数据过期（超过 1 天或 `last_webfingered_at` 为 nil）

#### ActivityPub::ProcessAccountService - 数据存储

**文件位置**: `app/services/activitypub/process_account_service.rb`

**关键更新操作**:
```ruby
# app/services/activitypub/process_account_service.rb:101-112
def update_account
  @account.last_webfingered_at = Time.now.utc unless @options[:only_key]
  @account.protocol            = :activitypub

  set_suspension!
  set_immediate_protocol_attributes!  # inbox_url, outbox_url 等
  set_fetchable_key! unless @account.suspended? && @account.suspension_origin_local?
  set_immediate_attributes! unless @account.suspended?  # display_name, note 等
  set_fetchable_attributes! unless @options[:only_key] || @account.suspended?  # 头像、头图、统计数据等

  @account.save_with_optional_media!
end
```

---

## 3. 刷新触发链路分析（代码事实 ⚠️ 重要修正）

> **本节为关键修正内容**。以下分析严格基于代码位置和调用关系，区分**确实存在**的机制与**之前错误推断**的机制。

### 3.1 关键常量定义

**文件位置**: `app/models/account.rb`
```ruby
# app/models/account.rb:76-78
BACKGROUND_REFRESH_INTERVAL = 1.week.freeze
REFRESH_DEADLINE = 6.hours
STALE_THRESHOLD = 1.day
```

| 常量 | 值 | 用途 |
|------|-----|------|
| `STALE_THRESHOLD` | 1 天 | 决定 `possibly_stale?` 返回值，影响 `ResolveAccountService` 的同步决策 |
| `BACKGROUND_REFRESH_INTERVAL` | 1 周 | 决定 `schedule_refresh_if_stale!` 是否安排后台刷新 |
| `REFRESH_DEADLINE` | 6 小时 | 后台刷新任务的随机延迟范围 |

### 3.2 确实存在的刷新触发点

#### 触发点 1: ActivityPub Create 活动

**文件位置**: `app/lib/activitypub/activity/create.rb`

```ruby
# app/lib/activitypub/activity/create.rb:7-9
def perform
  @account.schedule_refresh_if_stale!

  dereference_object!
  create_status
end
```

**触发时机**：收到远端账号的 `Create` 活动时（通常是新嘟文）。

**执行逻辑**：
1. 首先调用 `@account.schedule_refresh_if_stale!`
2. 然后继续处理 `Create` 活动（创建嘟文等）

#### 触发点 2: ActivityPub Update 活动

**文件位置**: `app/lib/activitypub/activity/update.rb`

```ruby
# app/lib/activitypub/activity/update.rb:7-19
def perform
  @account.schedule_refresh_if_stale!

  dereference_object!

  if equals_or_includes_any?(@object['type'], %w(Application Group Organization Person Service))
    update_account
  elsif supported_object_type? || converted_object_type?
    update_status
  elsif equals_or_includes_any?(@object['type'], ['FeaturedCollection']) && Mastodon::Feature.collections_enabled?
    update_collection
  end
end
```

**关键代码** - 账号更新处理:
```ruby
# app/lib/activitypub/activity/update.rb:23-27
def update_account
  return reject_payload! if @account.uri != object_uri

  ActivityPub::ProcessAccountService.new.call(@account.username, @account.domain, @object, signed_with_known_key: true, request_id: @options[:request_id])
end
```

**触发时机**：收到远端账号的 `Update` 活动时。

**执行逻辑**：
1. **首先**调用 `@account.schedule_refresh_if_stale!`
2. **然后**根据活动类型分别处理：
   - **如果是账号资料更新**（类型为 `Person`/`Group`/`Organization`/`Application`/`Service`）：
     - **直接调用** `ActivityPub::ProcessAccountService` 更新账号
     - 这会**立即**设置 `last_webfingered_at = Time.now.utc`
   - 如果是嘟文更新：调用 `ProcessStatusUpdateService`
   - 如果是精选集合更新：调用 `ProcessFeaturedCollectionService`

**重要注意**: 对于账号资料的 `Update` 活动，`schedule_refresh_if_stale!` 的效果可能会被随后的直接更新覆盖，因为直接更新会立即设置 `last_webfingered_at`。

#### 触发点 3: 首次关注远端账号

**文件位置**: `app/lib/activitypub/activity/accept.rb`

```ruby
# app/lib/activitypub/activity/accept.rb:29-36
def accept_follow!(request)
  return if request.nil?

  is_first_follow = !request.target_account.followers.local.exists?
  request.authorize!

  RemoteAccountRefreshWorker.perform_async(request.target_account_id) if is_first_follow
end
```

**触发时机**：当远端账号接受本地用户的首次关注时。

**执行逻辑**：
- 检查是否为首次关注（本地没有其他用户关注该远端账号）
- 如果是首次关注，异步执行 `RemoteAccountRefreshWorker`

#### 触发点 4: 显式账号解析

**调用场景**：
1. **用户搜索远端账号**时
2. **手动调用** `account.refresh!` 时
3. **`AccountRefreshWorker` 执行**时

**相关代码**:
```ruby
# app/models/account.rb:274-276
def refresh!
  ResolveAccountService.new.call(acct) unless local?
end
```

```ruby
# app/workers/account_refresh_worker.rb:8-13
def perform(account_id)
  account = Account.find_by(id: account_id)
  return if account.nil? || account.last_webfingered_at > Account::BACKGROUND_REFRESH_INTERVAL.ago

  ResolveAccountService.new.call(account)
end
```

### 3.3 schedule_refresh_if_stale! 方法详解

**文件位置**: `app/models/account.rb`

```ruby
# app/models/account.rb:268-272
def schedule_refresh_if_stale!
  return unless last_webfingered_at.present? && last_webfingered_at <= BACKGROUND_REFRESH_INTERVAL.ago

  AccountRefreshWorker.perform_in(rand(REFRESH_DEADLINE), id)
end
```

**逻辑分析**：
| 条件 | 结果 |
|------|------|
| `last_webfingered_at` 为 nil | 不安排刷新（直接返回） |
| `last_webfingered_at` 不超过 1 周 | 不安排刷新 |
| `last_webfingered_at` 超过 1 周 | 安排 `AccountRefreshWorker` 在 0-6 小时内执行 |

**关键特点**：
- **不是立即刷新**：只是安排后台任务
- **有随机延迟**：0-6 小时，用于分散系统负载
- **条件严格**：只有超过 1 周未更新才会触发

### 3.4 两个不同的 RefreshWorker（重要区分）

#### AccountRefreshWorker

**文件位置**: `app/workers/account_refresh_worker.rb`

```ruby
# app/workers/account_refresh_worker.rb:8-13
def perform(account_id)
  account = Account.find_by(id: account_id)
  return if account.nil? || account.last_webfingered_at > Account::BACKGROUND_REFRESH_INTERVAL.ago

  ResolveAccountService.new.call(account)
end
```

**执行流程**：
```
ResolveAccountService
    │
    ├──► Webfinger 查询
    │
    └──► ActivityPub::FetchRemoteAccountService
            │
            └──► ActivityPub::ProcessAccountService
```

**特点**：完整的 WebFinger + ActivityPub 流程。

#### RemoteAccountRefreshWorker

**文件位置**: `app/workers/remote_account_refresh_worker.rb`

```ruby
# app/workers/remote_account_refresh_worker.rb:10-19
def perform(id)
  account = Account.find_by(id: id)
  return if account.nil? || account.local?

  ActivityPub::FetchRemoteAccountService.new.call(account.uri)
rescue Mastodon::UnexpectedResponseError => e
  response = e.response

  raise(e) unless response_error_unsalvageable?(response)
end
```

**执行流程**：
```
ActivityPub::FetchRemoteAccountService
    │
    └──► ActivityPub::ProcessAccountService
```

**特点**：**跳过 WebFinger 查询**，直接使用已有的 `account.uri` 获取 ActivityPub 数据。

**用途对比**：
| Worker | 调用者 | 是否执行 WebFinger | 适用场景 |
|--------|--------|---------------------|----------|
| `AccountRefreshWorker` | `schedule_refresh_if_stale!` | ✅ 是 | 定期后台刷新，可能需要处理域名变更 |
| `RemoteAccountRefreshWorker` | 首次关注时 (`Accept` 活动) | ❌ 否 | 快速刷新，假设 `uri` 仍然有效 |

### 3.5 不存在的机制（之前错误推断 ⚠️）

基于对 `config/sidekiq.yml` 和整个代码库的检查，以下机制**并不存在**：

#### ❌ 不存在：定时扫描所有远端账号并刷新

**检查依据**：
- `config/sidekiq.yml` 中定义了所有定时任务
- 没有任何任务会扫描 `accounts` 表并为远端账号入队刷新任务

`config/sidekiq.yml` 中的定时任务列表：
| 任务名 | 执行频率 | 用途 |
|--------|----------|------|
| `scheduled_statuses_scheduler` | 每 5 分钟 | 处理预定发布的嘟文 |
| `trends_refresh_scheduler` | 每 5 分钟 | 刷新趋势数据 |
| `trends_review_notifications_scheduler` | 每 6 小时 | 趋势审查通知 |
| `indexing_scheduler` | 每分钟 | 索引任务 |
| `vacuum_scheduler` | 每天 (随机时间) | 数据库清理 |
| `follow_recommendations_scheduler` | 每天 (随机时间) | 关注推荐 |
| `user_cleanup_scheduler` | 每天 (随机时间) | 用户清理 |
| `ip_cleanup_scheduler` | 每天 (随机时间) | IP 清理 |
| `pghero_scheduler` | 每天 0 点 | 数据库监控 |
| `instance_refresh_scheduler` | 每小时 | 刷新**实例**信息（不是账号） |
| `accounts_statuses_cleanup_scheduler` | 每分钟 | 账号嘟文清理 |
| `suspended_user_cleanup_scheduler` | 每分钟 | 被暂停用户清理 |
| `software_update_check_scheduler` | 每 30 分钟 | 软件更新检查 |
| `auto_close_registrations_scheduler` | 每小时 | 自动关闭注册 |
| `fasp_follow_recommendation_cleanup_scheduler` | 每天 | FASP 推荐清理 |
| `collection_item_cleanup_scheduler` | 每小时 | 集合项目清理 |

**结论**：没有任何定时任务会主动扫描和刷新远端账号。

#### ❌ 不存在：每次 Create/Update 活动都会立即刷新账号

**事实**：
- `schedule_refresh_if_stale!` 有严格的条件：`last_webfingered_at` 超过 1 周
- 即使满足条件，也只是**安排后台任务**，不是立即刷新
- 后台任务有 0-6 小时的随机延迟

#### ❌ 不存在：schedule_refresh_if_stale! 会立即刷新

**事实**：
- `schedule_refresh_if_stale!` 只是调用 `AccountRefreshWorker.perform_in(rand(REFRESH_DEADLINE), id)`
- `perform_in` 表示"在指定时间后执行"，不是立即执行
- `rand(REFRESH_DEADLINE)` 是 0-6 小时的随机值

### 3.6 刷新触发链路完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    远端账号刷新触发链路（代码事实）                            │
└─────────────────────────────────────────────────────────────────────────────┘

触发源 A: ActivityPub Create 活动 (远端发布新嘟文)
=============================================
app/lib/activitypub/activity/create.rb:8
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ @account.schedule_refresh_if_stale!                         │
│                                                              │
│ 检查条件:                                                     │
│ last_webfingered_at.present? &&                             │
│ last_webfingered_at <= 1.week.ago                           │
└─────────────────────────────────────────────────────────────┘
         │
         ├── 条件不满足 ──► 不执行任何刷新操作
         │
         ▼ 条件满足
         │
┌─────────────────────────────────────────────────────────────┐
│ AccountRefreshWorker.perform_in(rand(6.hours), id)         │
│                                                              │
│ 注意: 这只是安排后台任务，不是立即执行                         │
│       实际执行时间是 0-6 小时后的随机时间                      │
└─────────────────────────────────────────────────────────────┘
         │
         ▼ (0-6 小时后)
         │
┌─────────────────────────────────────────────────────────────┐
│ AccountRefreshWorker.perform(account_id)                    │
│                                                              │
│ 再次检查条件:                                                 │
│ return if account.last_webfingered_at > 1.week.ago         │
│ (可能在此期间已通过其他方式更新)                               │
│                                                              │
│ 然后执行:                                                     │
│ ResolveAccountService.new.call(account)                     │
│     │                                                        │
│     ├──► Webfinger 查询                                      │
│     └──► ActivityPub 获取与处理                              │
│          └──► last_webfingered_at = Time.now.utc            │
└─────────────────────────────────────────────────────────────┘

═════════════════════════════════════════════════════════════════════════════

触发源 B: ActivityPub Update 活动
===================================
app/lib/activitypub/activity/update.rb:8
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 步骤 1: @account.schedule_refresh_if_stale!                 │
│         (同上：条件满足时安排 AccountRefreshWorker)           │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 步骤 2: 根据活动类型分别处理                                   │
└─────────────────────────────────────────────────────────────┘
         │
         ├── 类型: Person / Group / Organization / Application / Service
         │   (账号资料更新)
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ ActivityPub::ProcessAccountService.new.call(...)            │
│                                                              │
│ 直接更新账号，会立即执行:                                      │
│ @account.last_webfingered_at = Time.now.utc                 │
│                                                              │
│ 注意: 这会覆盖步骤 1 中可能安排的刷新任务的效果，              │
│      因为 last_webfingered_at 已被更新为当前时间              │
└─────────────────────────────────────────────────────────────┘
         │
         ├── 类型: Note / Article 等 (嘟文更新)
         │   └──► ActivityPub::ProcessStatusUpdateService
         │
         └── 类型: FeaturedCollection (精选集合更新)
             └──► ActivityPub::ProcessFeaturedCollectionService

═════════════════════════════════════════════════════════════════════════════

触发源 C: 首次关注远端账号
============================
app/lib/activitypub/activity/accept.rb:35
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 条件检查: is_first_follow =                                  │
│         !request.target_account.followers.local.exists?     │
│                                                              │
│ (本地没有其他用户关注该远端账号)                               │
└─────────────────────────────────────────────────────────────┘
         │
         ├── 否 ──► 不执行刷新
         │
         ▼ 是
         │
┌─────────────────────────────────────────────────────────────┐
│ RemoteAccountRefreshWorker.perform_async(request.target_account_id)│
│                                                              │
│ 注意: perform_async 表示"尽快异步执行"，但不是立即执行        │
└─────────────────────────────────────────────────────────────┘
         │
         ▼ (Sidekiq 调度后)
         │
┌─────────────────────────────────────────────────────────────┐
│ RemoteAccountRefreshWorker.perform(id)                      │
│                                                              │
│ 执行:                                                        │
│ ActivityPub::FetchRemoteAccountService.new.call(account.uri)│
│                                                              │
│ 特点: ⚠️ 跳过 WebFinger 查询                                 │
│       直接使用已有的 account.uri                              │
│                                                              │
│ 然后:                                                        │
│ ActivityPub::ProcessAccountService                          │
│   └──► last_webfingered_at = Time.now.utc                   │
└─────────────────────────────────────────────────────────────┘

═════════════════════════════════════════════════════════════════════════════

触发源 D: 显式账号解析
========================
触发场景:
- 用户搜索 @username@domain
- 手动调用 account.refresh!
- 其他显式解析场景
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ ResolveAccountService.new.call(uri, options)               │
│                                                              │
│ 检查是否需要 WebFinger 更新: webfinger_update_due?          │
│                                                              │
│ 需要更新的条件:                                               │
│ - @options[:skip_cache] == true (强制刷新)                   │
│ - @account.nil? (本地无记录)                                 │
│ - @account.possibly_stale? (超过 1 天未更新)                │
└─────────────────────────────────────────────────────────────┘
         │
         ├── 不需要更新 ──► 直接返回本地缓存
         │
         ▼ 需要更新
         │
┌─────────────────────────────────────────────────────────────┐
│ 完整的解析流程:                                               │
│                                                              │
│ 1. Webfinger 查询                                            │
│    ├── 获取 actor URL                                        │
│    ├── 处理重定向                                             │
│    └── 检测删除 (410 Gone)                                   │
│                                                              │
│ 2. ActivityPub::FetchRemoteAccountService                   │
│    ├── 获取 actor JSON-LD 数据                               │
│    └── check_webfinger! (验证循环)                           │
│                                                              │
│ 3. ActivityPub::ProcessAccountService                       │
│    └──► last_webfingered_at = Time.now.utc                  │
└─────────────────────────────────────────────────────────────┘

═════════════════════════════════════════════════════════════════════════════

❌ 不存在的触发方式
==================
以下机制在代码中并不存在:

1. ❌ 定时扫描所有远端账号并刷新
   - config/sidekiq.yml 中没有相关任务
   - 没有代码会遍历 Account.remote 并入队刷新任务

2. ❌ 每次 Create/Update 活动都会立即刷新
   - schedule_refresh_if_stale! 有严格的条件 (超过 1 周)
   - 即使满足条件，也只是安排后台任务 (0-6 小时延迟)

3. ❌ schedule_refresh_if_stale! 会立即刷新
   - 它只调用 perform_in，安排延迟执行
   - 实际执行时间是随机的 0-6 小时后
```

---

## 4. 本地缓存同步策略（代码事实）

### 4.1 数据存储模型

远端账号信息存储在 `accounts` 表中，与本地账号使用相同的模型。

**关键字段区分**：
| 字段 | 本地账号 | 远端账号 |
|------|----------|----------|
| `domain` | NULL | 远端域名 |
| `private_key` | 有值 | NULL |
| `last_webfingered_at` | 可能为 NULL | 每次同步后更新 |

### 4.2 同步时机总结

基于代码事实，本地缓存的同步发生在以下时机：

#### 同步时机 1: 显式解析时
- 用户搜索远端账号
- 手动调用 `refresh!`
- `AccountRefreshWorker` 执行

**同步方式**：完整的 WebFinger + ActivityPub 流程。

#### 同步时机 2: 收到账号 Update 活动时
- 远端账号更新资料时，会发送 `Update` 活动
- 类型为 `Person`/`Group` 等的 `Update` 活动会直接触发 `ProcessAccountService`

**同步方式**：直接处理收到的 ActivityPub 数据，跳过 WebFinger（但数据已签名验证）。

#### 同步时机 3: 首次关注时
- 本地用户首次关注某个远端账号
- 触发 `RemoteAccountRefreshWorker`

**同步方式**：跳过 WebFinger，直接获取 ActivityPub 数据。

### 4.3 last_webfingered_at 字段的更新时机

**该字段在以下情况被更新为当前时间**：

1. **`ProcessAccountService#update_account` 执行时**（非 `only_key` 模式）
   ```ruby
   # app/services/activitypub/process_account_service.rb:102
   @account.last_webfingered_at = Time.now.utc unless @options[:only_key]
   ```

   **触发场景**：
   - `ResolveAccountService` 流程（WebFinger + ActivityPub）
   - `ActivityPub::FetchRemoteAccountService` 流程（仅 ActivityPub）
   - 收到账号资料的 `Update` 活动

2. **该字段不会在以下情况更新**：
   - `schedule_refresh_if_stale!` 被调用时（只是安排任务）
   - 收到 `Create` 活动时（除非触发了后台刷新任务并执行）
   - 收到嘟文的 `Update` 活动时

### 4.4 一致性保证机制

#### WebFinger 验证循环

**文件位置**: `app/services/activitypub/fetch_remote_actor_service.rb`

```ruby
# app/services/activitypub/fetch_remote_actor_service.rb:56-75
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

**验证流程**：
1. 从 ActivityPub actor 数据提取 `username` 和 `domain`
2. 执行 WebFinger 查询
3. **关键检查**：WebFinger 响应中的 `self` 链接必须等于获取的 actor URL
4. 处理最多一层重定向
5. **最终验证**：重定向后的信息必须一致

**目的**：防止域名劫持攻击，确保获取的数据确实属于目标域名。

#### 分布式锁

多个关键点使用 Redis 分布式锁防止并发更新：

```ruby
# app/services/resolve_account_service.rb:107-110
def fetch_account!
  with_redis_lock("resolve:#{@username}@#{@domain}") do
    @account = ActivityPub::FetchRemoteAccountService.new.call(actor_url, suppress_errors: @options[:suppress_errors])
  end
end
```

```ruby
# app/services/activitypub/process_account_service.rb:34
with_redis_lock("process_account:#{@uri}") do
```

---

## 5. 关键代码位置汇总

| 功能 | 文件路径 | 关键方法/类 |
|------|----------|-------------|
| WebFinger 客户端 | `app/lib/webfinger.rb` | `Webfinger` 类, `perform()` |
| WebFinger 服务端控制器 | `app/controllers/well_known/webfinger_controller.rb` | `WebfingerController` |
| WebFinger 资源解析 | `app/lib/webfinger_resource.rb` | `WebfingerResource` 类 |
| 账号解析主服务 | `app/services/resolve_account_service.rb` | `ResolveAccountService`, `webfinger_update_due?` |
| ActivityPub Actor 获取 | `app/services/activitypub/fetch_remote_actor_service.rb` | `check_webfinger!` |
| 账号数据处理 | `app/services/activitypub/process_account_service.rb` | `update_account`, `last_webfingered_at` 更新 |
| 账号模型 | `app/models/account.rb` | `schedule_refresh_if_stale!`, `possibly_stale?`, `refresh!` |
| ActivityPub Create 处理 | `app/lib/activitypub/activity/create.rb` | `perform` 方法第 8 行 |
| ActivityPub Update 处理 | `app/lib/activitypub/activity/update.rb` | `perform`, `update_account` |
| ActivityPub Accept 处理 | `app/lib/activitypub/activity/accept.rb` | `accept_follow!` |
| 账号刷新 Worker | `app/workers/account_refresh_worker.rb` | `AccountRefreshWorker` |
| 远端账号刷新 Worker | `app/workers/remote_account_refresh_worker.rb` | `RemoteAccountRefreshWorker` |
| Sidekiq 定时任务配置 | `config/sidekiq.yml` | `:scheduler: :schedule:` 部分 |

---

## 6. 常见误解澄清

> **本节澄清之前报告中的错误以及常见的理解偏差**

### 误解 1: "Mastodon 会定时扫描所有远端账号并刷新"

**事实**：❌ 不存在这样的机制

- `config/sidekiq.yml` 中没有相关的定时任务
- 没有代码会遍历 `Account.remote` 并为每个账号入队刷新任务
- `instance_refresh_scheduler` 只是刷新实例信息，不是账号信息

### 误解 2: "每次收到远端账号的活动都会立即刷新账号"

**事实**：❌ 不是立即刷新，也不是每次都刷新

- `schedule_refresh_if_stale!` 有严格的条件：`last_webfingered_at` 超过 **1 周**
- 即使满足条件，也只是**安排后台任务**，不是立即执行
- 后台任务有 **0-6 小时**的随机延迟

### 误解 3: "schedule_refresh_if_stale! 会立即刷新账号"

**事实**：❌ 它只是安排后台任务

```ruby
# 实际代码
AccountRefreshWorker.perform_in(rand(REFRESH_DEADLINE), id)
```

- `perform_in` = "在指定时间后执行"
- `rand(REFRESH_DEADLINE)` = 0-6 小时的随机值

### 误解 4: "AccountRefreshWorker 和 RemoteAccountRefreshWorker 是一样的"

**事实**：❌ 它们有重要区别

| 特性 | AccountRefreshWorker | RemoteAccountRefreshWorker |
|------|----------------------|---------------------------|
| 调用者 | `schedule_refresh_if_stale!` | 首次关注时 |
| 执行 WebFinger | ✅ 是 | ❌ 否 |
| 适用场景 | 可能有域名变更 | 假设 URI 仍然有效 |

### 误解 5: "收到 Create 活动时会刷新账号"

**事实**：⚠️ 只有在特定条件下才会安排刷新

- 收到 `Create` 活动时，会调用 `schedule_refresh_if_stale!`
- 但这**只有**在 `last_webfingered_at` 超过 1 周时才会安排后台刷新
- 实际刷新发生在 0-6 小时后（如果后台任务执行时条件仍然满足）

### 事实: "收到账号 Update 活动时会立即更新账号"

**✅ 这是正确的**

```ruby
# app/lib/activitypub/activity/update.rb:23-27
def update_account
  return reject_payload! if @account.uri != object_uri

  ActivityPub::ProcessAccountService.new.call(@account.username, @account.domain, @object, signed_with_known_key: true, request_id: @options[:request_id])
end
```

- 当收到类型为 `Person`/`Group` 等的 `Update` 活动时
- 会**直接调用** `ProcessAccountService`
- 这会**立即**更新 `last_webfingered_at = Time.now.utc`

---

## 7. 总结

### 7.1 刷新触发机制的实际行为

基于代码事实，Mastodon 的远端账号刷新机制可以总结为：

#### 被动刷新（基于收到的活动）

1. **Create 活动**:
   - 调用 `schedule_refresh_if_stale!`
   - 只有超过 1 周未更新时，安排 0-6 小时后的后台刷新

2. **账号 Update 活动**:
   - 首先调用 `schedule_refresh_if_stale!`
   - **然后直接更新账号**（设置 `last_webfingered_at`）
   - 这会使之前安排的刷新任务变得不必要

3. **嘟文 Update 活动**:
   - 调用 `schedule_refresh_if_stale!`
   - 不直接更新账号资料

#### 主动刷新（基于显式操作）

1. **用户搜索**:
   - 触发 `ResolveAccountService`
   - 如果数据过期（超过 1 天）或本地无记录，执行完整刷新

2. **首次关注**:
   - 触发 `RemoteAccountRefreshWorker`
   - 跳过 WebFinger，直接获取 ActivityPub 数据

3. **手动调用 refresh!**:
   - 触发完整的 `ResolveAccountService` 流程

### 7.2 关键设计决策分析

#### 1. 为什么使用被动刷新为主？

**设计意图**：
- 减少对远端服务器的请求压力
- 只有当账号"活跃"时（发布内容、被关注、被搜索）才需要刷新
- 不活跃的账号不需要保持最新数据

**实际效果**：
- 如果一个远端账号不再发布内容，也没有本地用户关注或搜索它
- 它的本地缓存可能永远不会刷新
- 这是合理的设计权衡

#### 2. 为什么 schedule_refresh_if_stale! 的阈值是 1 周？

**设计意图**：
- 账号资料（头像、显示名、简介等）不会频繁变更
- 1 周是平衡数据新鲜度和系统负载的合理阈值

**与 possibly_stale? (1 天) 的区别**：
- `possibly_stale?` (1 天): 用于显式解析场景（用户搜索）
- `BACKGROUND_REFRESH_INTERVAL` (1 周): 用于后台被动刷新

这意味着：
- 用户搜索时，如果数据超过 1 天，会同步刷新
- 后台被动刷新只有超过 1 周才会触发

#### 3. 为什么有两种不同的 RefreshWorker？

**AccountRefreshWorker** (执行 WebFinger):
- 用于后台被动刷新场景
- 需要处理可能的域名变更或账号迁移
- WebFinger 可以检测这些变更

**RemoteAccountRefreshWorker** (跳过 WebFinger):
- 用于首次关注场景
- 此时刚刚完成了 `Accept` 活动的交互
- 假设 `account.uri` 仍然有效
- 跳过 WebFinger 可以提高响应速度

### 7.3 系统依赖与边界条件

#### 依赖项

1. **WebFinger 协议**: 账号发现的核心协议
2. **ActivityPub 协议**: 数据获取和更新的核心协议
3. **Redis**: 分布式锁、任务队列
4. **Sidekiq**: 后台任务处理
5. **Sidekiq Scheduler**: 定时任务（但没有账号刷新相关任务）

#### 边界条件

1. **远端服务器不可达**:
   - WebFinger 请求失败 → `ResolveAccountService` 可能返回缓存或失败
   - ActivityPub 请求失败 → 可能安排重试任务

2. **远端服务器返回 410 Gone**:
   - WebFinger 返回 410 → 标记本地账号为删除
   - 安排 `AccountDeletionWorker`

3. **数据签名验证失败**:
   - 可能导致数据不被处理
   - 具体行为取决于签名验证逻辑

4. **并发更新**:
   - 通过 Redis 分布式锁防止数据竞争

---

## 附录: 验证方法

读者可以通过以下方法验证本文档中的陈述：

### 验证 schedule_refresh_if_stale! 的调用位置

```ruby
# 在 Rails console 中执行
# 搜索调用 schedule_refresh_if_stale! 的位置

# 或者检查具体文件:
# 1. app/lib/activitypub/activity/create.rb 第 8 行
# 2. app/lib/activitypub/activity/update.rb 第 8 行
```

### 验证定时任务配置

查看文件 `config/sidekiq.yml` 中的 `:scheduler: :schedule:` 部分，确认没有账号刷新相关的定时任务。

### 验证两个 RefreshWorker 的区别

比较两个文件：
- `app/workers/account_refresh_worker.rb` - 调用 `ResolveAccountService`
- `app/workers/remote_account_refresh_worker.rb` - 调用 `ActivityPub::FetchRemoteAccountService`
