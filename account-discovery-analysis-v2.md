# 账号发现机制与远端资料同步分析报告 (v2)

## 核心校正与补充

本报告重点校正以下内容：
1. **账号重定向判定条件** - 两层检测机制与回环验证
2. **最终一致性保障** - 缓存命中、刷新阈值与异步任务的协同

---

## 目录
1. [账号重定向判定机制](#1-账号重定向判定机制)
2. [阈值分层与缓存命中策略](#2-阈值分层与缓存命中策略)
3. [异步刷新任务与最终一致性](#3-异步刷新任务与最终一致性)
4. [关键数据流 v2](#4-关键数据流-v2)
5. [竞态条件与防护机制](#5-竞态条件与防护机制)

---

## 1. 账号重定向判定机制

### 1.1 两层重定向检测架构

Mastodon 采用**两层检测 + 回环验证**的架构来处理账号重定向：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    账号重定向检测架构                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  第一层: ResolveAccountService (基于 WebFinger subject)                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  输入: username@domain (用户请求的账号)                            │   │
│  │  检测: WebFinger 返回的 subject 与请求的账号是否不同               │   │
│  │  限制: 最多允许 1 次重定向                                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                    ↓                                      │
│  第二层: FetchRemoteActorService (基于 ActivityPub actor)               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  输入: actor URL (从 WebFinger self link 获取)                    │   │
│  │  检测: 从 actor JSON 解析 username + domain                       │   │
│  │  关键: 回环验证 (Loop Back Validation)                            │   │
│  │        WebFinger self_link_href 必须等于 ActivityPub actor.id    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 第一层：ResolveAccountService 重定向判定

**精确判定逻辑：** `app/services/resolve_account_service.rb:82-99`

```ruby
def process_webfinger!(uri)
  # 第 1 步: 对请求的账号执行 WebFinger 查询
  @webfinger = Webfinger.new("acct:#{uri}").perform
  confirmed_username, confirmed_domain = split_acct(@webfinger.subject)

  # 判定条件 A: subject 与请求的账号相同（不区分大小写）
  if confirmed_username.casecmp(@username).zero? && confirmed_domain.casecmp(@domain).zero?
    @username = confirmed_username
    @domain   = confirmed_domain
    return  # 不是重定向，直接返回
  end

  # 判定条件 B: subject 不同 → 检测到重定向
  # 对重定向后的账号执行第二次 WebFinger 查询
  @webfinger = Webfinger.new("acct:#{confirmed_username}@#{confirmed_domain}").perform
  @username, @domain = split_acct(@webfinger.subject)

  # 判定条件 C: 重定向验证
  # 第二次查询返回的 subject 必须与第一次返回的相同
  # 否则就是"过多重定向"
  raise Webfinger::RedirectError, "Too many webfinger redirects..." unless 
    confirmed_username.casecmp(@username).zero? && confirmed_domain.casecmp(@domain).zero?
rescue Webfinger::GoneError
  @gone = true  # 账号已删除标记
end
```

**重定向判定真值表：**

| 请求账号 | 第一次 WebFinger subject | 第二次 WebFinger subject | 判定结果 |
|---------|--------------------------|--------------------------|---------|
| `A@X` | `A@X` | 不执行 | 不是重定向 |
| `A@X` | `B@Y` | `B@Y` | 合法重定向 A→B |
| `A@X` | `B@Y` | `C@Z` | 过多重定向 (Error) |
| `A@X` | `a@x` (大小写不同) | 不执行 | 不是重定向 (casecmp 比较) |

### 1.3 第二层：FetchRemoteActorService 回环验证

**关键验证机制：** `app/services/activitypub/fetch_remote_actor_service.rb:56-75`

```ruby
def check_webfinger!
  # 从 ActivityPub actor 解析出的账号信息
  webfinger = Webfinger.new("acct:#{@username}@#{@domain}").perform
  confirmed_username, confirmed_domain = split_acct(webfinger.subject)

  if @username.casecmp(confirmed_username).zero? && @domain.casecmp(confirmed_domain).zero?
    # ⭐ 回环验证 (Loop Back Validation)
    # WebFinger 返回的 self link 必须与 ActivityPub actor 的 id 完全一致
    raise Error, "Webfinger response for #{@username}@#{@domain} does not loop back to #{@uri}" if 
      webfinger.self_link_href != @uri
    return
  end

  # 如果账号不同，尝试重定向（与第一层逻辑相同）
  webfinger = Webfinger.new("acct:#{confirmed_username}@#{confirmed_domain}").perform
  @username, @domain = split_acct(webfinger.subject)

  raise Webfinger::RedirectError, "Too many webfinger redirects..." unless 
    confirmed_username.casecmp(@username).zero? && confirmed_domain.casecmp(@domain).zero?
  
  # ⭐ 再次回环验证
  raise Error, "Webfinger response for #{@username}@#{@domain} does not loop back to #{@uri}" if 
    webfinger.self_link_href != @uri
end
```

**回环验证的目的：**
- 确保 WebFinger 记录和 ActivityPub 记录指向**同一个实体**
- 防止：WebFinger 声称某个账号，但 self link 指向不匹配的 actor

### 1.4 测试用例深度解析

基于 `spec/services/resolve_account_service_spec.rb` 的分析：

**场景 1：合法重定向 (Legitimate Redirection)**

```
输入: Foo@redirected.example.com

步骤 1: WebFinger 查询 redirected.example.com
        返回:
          subject: 'acct:foo@ap.example.com'  ← 与请求的不同
          links: [{ rel: 'self', href: 'https://ap.example.com/users/foo', type: 'application/activity+json' }]

判定: subject 不同 → 检测到重定向

步骤 2: WebFinger 查询 foo@ap.example.com
        返回:
          subject: 'acct:foo@ap.example.com'  ← 与第一次返回的相同

判定: 重定向验证通过

结果: 账号 foo@ap.example.com
```

**场景 2：配置错误重定向 (Misconfigured Redirection)**

这是一个**容易被误解**的场景：

```
输入: Foo@redirected.example.com

步骤 1: WebFinger 查询 redirected.example.com
        返回:
          subject: 'acct:Foo@redirected.example.com'  ← 与请求的大小写相同
          links: [{ rel: 'self', href: 'https://ap.example.com/users/foo', ... }]

第一层判定 (ResolveAccountService):
  - 'Foo'.casecmp('Foo').zero? → true
  - 'redirected.example.com'.casecmp('redirected.example.com').zero? → true
  - 结论: 不是重定向！

  但是！self_link_href = 'https://ap.example.com/users/foo'
  这指向了另一个域名的 actor

步骤 2: FetchRemoteActorService 获取 actor
        GET https://ap.example.com/users/foo
        返回:
          id: 'https://ap.example.com/users/foo'
          preferredUsername: 'foo'
          ...

        从 id 解析 domain: 'ap.example.com'
        所以 @username = 'foo', @domain = 'ap.example.com'

步骤 3: 回环验证 (check_webfinger!)
        WebFinger 查询 foo@ap.example.com
        返回:
          subject: 'acct:foo@ap.example.com'
          self_link_href: 'https://ap.example.com/users/foo'

        验证:
          - username/domain 匹配 ✓
          - self_link_href == actor.id ✓
          - 回环验证通过！

结果: 账号 foo@ap.example.com
```

**关键洞察：**
- 这个场景实际上**没有通过第一层的重定向检测**
- 但 WebFinger 的 self link 指向了另一个域名的 actor
- 系统通过**第二层的回环验证**来确保数据一致性

**场景 3：过多重定向 (Too Many Redirections)**

```
输入: Foo@redirected.example.com

步骤 1: WebFinger 查询 redirected.example.com
        返回 subject: 'acct:foo@evil.example.com'

第一层判定: subject 不同 → 检测到重定向

步骤 2: WebFinger 查询 foo@evil.example.com
        返回 subject: 'acct:foo@ap.example.com'

重定向验证:
  - 第一次返回的: 'foo@evil.example.com'
  - 第二次返回的: 'foo@ap.example.com'
  - 不相同!

结果: Webfinger::RedirectError → 解析失败
```

### 1.5 重定向流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        完整重定向检测流程                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  开始: 用户搜索 @username@domain                                            │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────────────┐                                                 │
│  │ ResolveAccountService   │                                                 │
│  │ #process_webfinger!     │                                                 │
│  └─────────────────────────┘                                                 │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ WF1: WebFinger 查询请求的账号                                         │  │
│  │      GET https://{domain}/.well-known/webfinger?resource=acct:{uri} │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 判定 1: WF1.subject == 请求的账号? (casecmp)                         │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│         │                                                                    │
│    ┌────┴────┐                                                               │
│    │         │                                                               │
│  Yes        No                                                               │
│    │         │                                                               │
│    │         ▼                                                               │
│    │  ┌────────────────────────────────────────────────────────────────┐ │
│    │  │ WF2: WebFinger 查询 WF1.subject                                 │ │
│    │  └────────────────────────────────────────────────────────────────┘ │
│    │         │                                                             │
│    │         ▼                                                             │
│    │  ┌────────────────────────────────────────────────────────────────┐ │
│    │  │ 判定 2: WF2.subject == WF1.subject?                            │ │
│    │  └────────────────────────────────────────────────────────────────┘ │
│    │         │                                                             │
│    │    ┌────┴────┐                                                        │
│    │    │         │                                                        │
│    │  Yes        No                                                        │
│    │    │         │                                                        │
│    │    │         ▼                                                        │
│    │    │  ┌──────────────────────────────────────────────────────────┐ │
│    │    │  │ 错误: Too many webfinger redirects                       │ │
│    │    │  │ 解析失败                                                  │ │
│    │    │  └──────────────────────────────────────────────────────────┘ │
│    │    │                                                                   │
│    ▼    ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 继续: 获取 ActivityPub Actor                                          │  │
│  │ actor_url = webfinger.self_link_href                                  │  │
│  │ GET {actor_url} (Accept: application/activity+json)                  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────────────┐                                                 │
│  │ FetchRemoteActorService │                                                 │
│  │ #check_webfinger!       │                                                 │
│  └─────────────────────────┘                                                 │
│         │                                                                    │
│         ▼                                                                    │
│  从 actor 解析:                                                              │
│    @username = actor.preferredUsername                                      │
│    @domain = URI.parse(actor.id).host                                       │
│    @uri = actor.id                                                           │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ WF3: WebFinger 查询 @username@@domain                                │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ ⭐ 回环验证 (Loop Back Validation)                                   │  │
│  │                                                                       │  │
│  │ 条件 A: WF3.subject == @username@@domain? (casecmp)                │  │
│  │ 条件 B: WF3.self_link_href == @uri (actor.id)?                      │  │
│  │                                                                       │  │
│  │ 两者都必须为 true，否则解析失败                                        │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 账号解析成功！                                                         │  │
│  │ 使用 @username@@domain 作为最终账号                                   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 阈值分层与缓存命中策略

### 2.1 三个关键时间阈值

`app/models/account.rb:76-78` 定义了三个核心阈值：

```ruby
BACKGROUND_REFRESH_INTERVAL = 1.week.freeze   # 后台刷新间隔
REFRESH_DEADLINE = 6.hours                     # 刷新任务延迟上限
STALE_THRESHOLD = 1.day                        # 数据陈旧阈值
```

### 2.2 阈值职责分层

| 阈值 | 值 | 判定方法 | 触发场景 | 设计目的 |
|------|-----|---------|---------|---------|
| **STALE_THRESHOLD** | 1 天 | `possibly_stale?` | 用户主动查询 | 主动刷新：确保用户看到较新的数据 |
| **BACKGROUND_REFRESH_INTERVAL** | 1 周 | `schedule_refresh_if_stale!` | 收到远程活动 | 被动刷新：后台静默更新 |
| **REFRESH_DEADLINE** | 6 小时 | `rand(REFRESH_DEADLINE)` | 调度延迟 | 负载分散：防止刷新风暴 |

### 2.3 缓存命中判定逻辑

**缓存命中 = 不需要执行 WebFinger + ActivityPub 获取**

在 `app/services/resolve_account_service.rb:115-120` 中：

```ruby
def webfinger_update_due?
  return false if @options[:check_delivery_availability] && !DeliveryFailureTracker.available?(@domain)
  return false if @options[:skip_webfinger]

  # ⭐ 缓存未命中（需要更新）的条件：
  @options[:skip_cache] ||      # 强制跳过缓存
    @account.nil? ||             # 账号不存在（新账号）
    @account.possibly_stale?    # 数据陈旧 (> 1 天)
end
```

**`possibly_stale?` 实现：** `app/models/account.rb:264-266`

```ruby
def possibly_stale?
  # last_webfingered_at 为空 或 超过 1 天
  last_webfingered_at.nil? || last_webfingered_at <= STALE_THRESHOLD.ago
end
```

### 2.4 缓存命中决策树

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     缓存命中决策流程                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  入口: ResolveAccountService.call(username@domain, options)             │
│         │                                                                 │
│         ▼                                                                 │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ Step 1: 检查本地账号是否存在                                        │ │
│  │         @account = Account.find_remote(@username, @domain)        │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│         │                                                                 │
│         ▼                                                                 │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ Step 2: 检查是否是本地账号或无需更新                                │ │
│  │                                                                     │ │
│  │ 条件: @account&.local? OR @domain.nil? OR !webfinger_update_due? │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│         │                                                                 │
│    ┌────┴────┐                                                           │
│    │         │                                                           │
│  Yes        No                                                           │
│    │         │                                                           │
│    │         ▼                                                           │
│    │  ┌─────────────────────────────────────────────────────────────┐ │ │
│    │  │ webfinger_update_due? 详细判定                               │ │ │
│    │  │                                                                 │ │ │
│    │  │ ┌─────────────────────────────────────────────────────────┐ │ │ │
│    │  │ │ 条件 1: skip_cache?                                      │ │ │ │
│    │  │ │           ↓ Yes                                          │ │ │ │
│    │  │ │           需要更新 (缓存未命中)                           │ │ │ │
│    │  │ └─────────────────────────────────────────────────────────┘ │ │ │
│    │  │                          │ No                                 │ │ │
│    │  │                          ▼                                    │ │ │
│    │  │ ┌─────────────────────────────────────────────────────────┐ │ │ │
│    │  │ │ 条件 2: @account.nil? (新账号)                          │ │ │ │
│    │  │ │           ↓ Yes                                          │ │ │ │
│    │  │ │           需要更新 (缓存未命中)                           │ │ │ │
│    │  │ └─────────────────────────────────────────────────────────┘ │ │ │
│    │  │                          │ No                                 │ │ │
│    │  │                          ▼                                    │ │ │
│    │  │ ┌─────────────────────────────────────────────────────────┐ │ │ │
│    │  │ │ 条件 3: @account.possibly_stale?                        │ │ │ │
│    │  │ │           last_webfingered_at <= 1天前?                  │ │ │ │
│    │  │ │           ↓ Yes                                          │ │ │ │
│    │  │ │           需要更新 (缓存未命中)                           │ │ │ │
│    │  │ │           ↓ No                                           │ │ │ │
│    │  │ │           缓存命中 → 直接返回 @account                   │ │ │ │
│    │  │ └─────────────────────────────────────────────────────────┘ │ │ │
│    │  └─────────────────────────────────────────────────────────────┘ │ │
│    │                                                                    │ │
│    ▼                                                                    ▼ │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ 缓存命中！                                                          │ │
│  │ 直接返回本地账号记录，不执行任何远端查询                            │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│  或                                                                       │
│                                                                           │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ 缓存未命中！                                                        │ │
│  │ 执行: process_webfinger! + fetch_account!                          │ │
│  │ 更新: last_webfingered_at = Time.now.utc                          │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.5 不同场景的缓存行为

| 场景 | 账号存在? | last_webfingered_at | webfinger_update_due? | 行为 |
|------|-----------|---------------------|----------------------|------|
| 新账号首次搜索 | No | nil | true | WebFinger + ActivityPub 获取 |
| 1 小时前刚更新 | Yes | 1 小时前 | false | 缓存命中，直接返回 |
| 2 天前更新 | Yes | 2 天前 | true | WebFinger + ActivityPub 获取 |
| 8 天前更新 | Yes | 8 天前 | true | WebFinger + ActivityPub 获取 |
| 使用 skip_cache: true | Yes | 任意 | true | 强制更新 |
| 使用 skip_webfinger: true | Yes | 任意 | false | 跳过 WebFinger，直接返回 |

### 2.6 EntityCache 与账号缓存的关系

**注意：** `EntityCache` 是另一个层级的缓存，与上述机制不同：

| 缓存层级 | 实现 | 内容 | 过期时间 | 用途 |
|---------|------|------|---------|------|
| **EntityCache** | `Rails.cache` | 数据库查询结果 (Account 部分字段) | 7 天 | 加速 `Account.find_remote` 查询 |
| **last_webfingered_at** | 数据库字段 | 上次刷新时间戳 | 无 | 决定是否需要**远端查询** |
| **Account 记录** | 数据库表 | 完整账号数据 | 无 | 实际的账号资料 |

**EntityCache.mention 实现：** `app/lib/entity_cache.rb:14-16`

```ruby
def mention(username, domain)
  Rails.cache.fetch(to_key(:mention, username, domain), expires_in: MAX_EXPIRATION) do
    # 注意：这只是数据库查询，不涉及任何 WebFinger 或 ActivityPub
    Account.select(:id, :username, :domain, :url).find_remote(username, domain)
  end
end
```

**关键区别：**
- EntityCache 缓存的是**数据库查询结果**
- last_webfingered_at 决定的是**是否需要从远端获取新数据**
- 两者是独立的缓存层级

---

## 3. 异步刷新任务与最终一致性

### 3.1 后台刷新触发时机

后台刷新任务在以下场景被触发：

**场景 A：收到远程用户的新状态** `app/lib/activitypub/activity/create.rb:8`

```ruby
def perform
  @account.schedule_refresh_if_stale!  # ← 这里
  dereference_object!
  create_status
end
```

**场景 B：收到远程用户的更新活动** `app/lib/activitypub/activity/update.rb:8`

```ruby
def perform
  @account.schedule_refresh_if_stale!  # ← 这里
  dereference_object!
  # ... 处理更新
end
```

### 3.2 后台刷新调度逻辑

`app/models/account.rb:268-272`

```ruby
def schedule_refresh_if_stale!
  # 条件: last_webfingered_at 存在 且 超过 1 周
  return unless last_webfingered_at.present? && last_webfingered_at <= BACKGROUND_REFRESH_INTERVAL.ago

  # 调度异步任务，随机延迟 0-6 小时
  AccountRefreshWorker.perform_in(rand(REFRESH_DEADLINE), id)
end
```

**阈值对比：**

| 方法 | 阈值 | 场景 |
|------|------|------|
| `possibly_stale?` | `STALE_THRESHOLD = 1 天` | 用户主动查询时 |
| `schedule_refresh_if_stale!` | `BACKGROUND_REFRESH_INTERVAL = 1 周` | 收到远程活动时 |

**设计意图：**
- 用户主动查询时，对数据新鲜度要求较高 → 1 天阈值
- 后台被动刷新时，优先考虑性能 → 1 周阈值 + 随机延迟

### 3.3 AccountRefreshWorker 双重检查

`app/workers/account_refresh_worker.rb:8-13`

```ruby
def perform(account_id)
  account = Account.find_by(id: account_id)
  
  # ⭐ 双重检查！
  # 在任务执行时再次检查是否需要刷新
  return if account.nil? || account.last_webfingered_at > Account::BACKGROUND_REFRESH_INTERVAL.ago

  ResolveAccountService.new.call(account)
end
```

### 3.4 最终一致性保障机制

#### 机制 1：双重检查锁定 (Double-Check)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        双重检查机制                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  时间线示例：                                                              │
│                                                                           │
│  T0: 账号 A 的 last_webfingered_at = 10 天前 (> 1 周)                   │
│         │                                                                 │
│         ▼                                                                 │
│  T1: 收到账号 A 的活动 → schedule_refresh_if_stale!                      │
│         │                                                                 │
│         ├─ 检查: 10天前 <= 1周前? → Yes                                  │
│         │                                                                 │
│         └─ 调度: AccountRefreshWorker.perform_in(3小时, accountA.id)   │
│         │                                                                 │
│         ▼                                                                 │
│  T2 (1 小时后): 用户主动搜索账号 A                                        │
│         │                                                                 │
│         ├─ ResolveAccountService.call(accountA)                          │
│         │                                                                 │
│         ├─ 检查: possibly_stale? (10天前 <= 1天前? → Yes)              │
│         │                                                                 │
│         ├─ 执行: WebFinger + ActivityPub 获取                            │
│         │                                                                 │
│         └─ 更新: last_webfingered_at = 现在                              │
│         │                                                                 │
│         ▼                                                                 │
│  T3 (3 小时后): AccountRefreshWorker 执行                                │
│         │                                                                 │
│         ├─ 获取账号 A                                                     │
│         │                                                                 │
│         ├─ ⭐ 双重检查: last_webfingered_at > 1周前?                    │
│         │     (现在 > 1周前? → Yes! 数据已更新)                         │
│         │                                                                 │
│         └─ return (跳过刷新)                                              │
│                                                                           │
│  结果：避免了重复刷新，节省了资源                                          │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 机制 2：随机延迟分散负载

```ruby
AccountRefreshWorker.perform_in(rand(REFRESH_DEADLINE), id)
# REFRESH_DEADLINE = 6 小时
```

**问题场景：** 假设 10000 个账号同时在 T0 超过 1 周阈值

**无随机延迟：**
- T0: 所有 10000 个任务同时调度
- T0: 所有任务同时执行 → 服务器负载峰值 → 可能超时/失败

**有随机延迟 (0-6 小时)：**
- 任务均匀分散在 6 小时窗口内
- 平均每小时约 1667 个任务
- 负载平滑，无峰值

#### 机制 3：分布式锁防止并发

**锁的使用场景：**

| 操作 | 锁键 | 代码位置 |
|------|------|----------|
| 解析账号 | `lock:resolve:{username}@{domain}` | `app/services/resolve_account_service.rb:108` |
| 处理账号 | `lock:process_account:{uri}` | `app/services/activitypub/process_account_service.rb:34` |

**ResolveAccountService 锁：**

```ruby
def fetch_account!
  with_redis_lock("resolve:#{@username}@#{@domain}") do
    @account = ActivityPub::FetchRemoteAccountService.new.call(actor_url, ...)
  end
  @account
end
```

**Lockable 实现：** `app/models/concerns/lockable.rb`

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

**锁保护的竞态场景：**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        分布式锁保护                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  场景：两个用户同时搜索同一个新账号 @new@remote.com                       │
│                                                                           │
│  无锁情况下：                                                              │
│  ┌─────────────────┐              ┌─────────────────┐                    │
│  │   进程 A         │              │   进程 B         │                    │
│  └────────┬────────┘              └────────┬────────┘                    │
│           │                                  │                             │
│           ▼                                  ▼                             │
│  find_remote → nil                find_remote → nil                      │
│           │                                  │                             │
│           ▼                                  ▼                             │
│  WebFinger 查询...                 WebFinger 查询...                      │
│           │                                  │                             │
│           ▼                                  ▼                             │
│  ActivityPub 获取...              ActivityPub 获取...                     │
│           │                                  │                             │
│           ▼                                  ▼                             │
│  Account.create!                    Account.create!                       │
│           │                                  │                             │
│           ▼                                  ▼                             │
│  成功 (id=123)                     ActiveRecord::RecordNotUnique!         │
│                                                                           │
│  问题：                                                                   │
│  - 重复的远端查询（浪费资源）                                             │
│  - 进程 B 可能失败（用户体验差）                                          │
│                                                                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  有锁情况下：                                                              │
│  ┌─────────────────┐              ┌─────────────────┐                    │
│  │   进程 A         │              │   进程 B         │                    │
│  └────────┬────────┘              └────────┬────────┘                    │
│           │                                  │                             │
│           ▼                                  ▼                             │
│  获取锁 lock:resolve:new@remote.com  尝试获取锁...                       │
│           │                                  │                             │
│           │                                  ▼                             │
│           │                            锁已被持有                          │
│           │                            等待/抛出 RaceConditionError       │
│           │                                  │                             │
│           ▼                                  │                             │
│  find_remote → nil                          │                             │
│           │                                  │                             │
│           ▼                                  │                             │
│  WebFinger + ActivityPub 获取               │                             │
│           │                                  │                             │
│           ▼                                  │                             │
│  Account.create! (id=123)                   │                             │
│           │                                  │                             │
│           ▼                                  │                             │
│  释放锁                                      │                             │
│                                              ▼                             │
│                                        获取锁成功                           │
│                                              │                             │
│                                              ▼                             │
│                                        find_remote → 返回 id=123          │
│                                              │                             │
│                                              ▼                             │
│                                        缓存命中，直接返回                  │
│                                                                           │
│  结果：                                                                    │
│  - 只执行一次远端查询                                                     │
│  - 两个进程都能成功返回（或一个成功一个明确失败）                         │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 机制 4：幂等操作设计

**ProcessAccountService 的幂等性保障：**

1. **last_webfingered_at 更新**
   ```ruby
   @account.last_webfingered_at = Time.now.utc unless @options[:only_key]
   ```
   - 每次处理都更新为当前时间
   - 重复调用不会产生问题

2. **媒体下载失败重试**
   ```ruby
   rescue Mastodon::UnexpectedResponseError, *Mastodon::HTTP_CONNECTION_ERRORS
     RedownloadAvatarWorker.perform_in(rand(PROCESSING_DELAY), @account.id)
   ```
   - 失败不影响主流程
   - 调度异步重试

3. **关联记录唯一约束处理**（在账号合并时）
   ```ruby
   klass.where(account_id: other_account.id).reorder(nil).find_each do |record|
     record.update_attribute(:account_id, id)
   rescue ActiveRecord::RecordNotUnique
     next  # 跳过重复记录
   end
   ```

4. **重复账号检测与合并**
   ```ruby
   def process_duplicate_accounts!
     return unless Account.where(uri: @account.uri).where.not(id: @account.id).exists?
     AccountMergingWorker.perform_async(@account.id)
   end
   ```

#### 机制 5：回环验证确保数据一致性

这是**最关键**的一致性保障：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        回环验证机制                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  验证目的：确保 WebFinger 记录和 ActivityPub 记录指向同一个实体           │
│                                                                           │
│  正常情况：                                                                │
│                                                                           │
│  用户查询: @alice@example.com                                             │
│       │                                                                   │
│       ▼                                                                   │
│  WebFinger 查询 example.com                                               │
│       │                                                                   │
│       ├─ subject: 'acct:alice@example.com'                               │
│       │                                                                   │
│       └─ links: [{ rel: 'self',                                          │
│       │           href: 'https://example.com/users/alice', ←──────┐    │
│       │           type: 'application/activity+json' }]              │    │
│       │                                                               │    │
│       ▼                                                               │    │
│  ActivityPub 获取 https://example.com/users/alice                    │    │
│       │                                                               │    │
│       ├─ id: 'https://example.com/users/alice'  ←───────────────────┘    │
│       │         (必须等于 self_link_href)                                  │
│       │                                                                     │
│       └─ preferredUsername: 'alice'                                        │
│                                                                             │
│  ⭐ 回环验证通过！                                                          │
│  Webfinger.self_link_href == ActivityPub.actor.id                         │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  异常情况（中间人攻击或配置错误）：                                         │
│                                                                           │
│  用户查询: @alice@example.com                                             │
│       │                                                                   │
│       ▼                                                                   │
│  WebFinger 查询 example.com                                               │
│       │                                                                   │
│       ├─ subject: 'acct:alice@example.com'                               │
│       │                                                                   │
│       └─ links: [{ rel: 'self',                                          │
│       │           href: 'https://evil.com/users/fake', ←───────┐       │
│       │           type: 'application/activity+json' }]          │       │
│       │                                                           │       │
│       ▼                                                           │       │
│  ActivityPub 获取 https://evil.com/users/fake                    │       │
│       │                                                           │       │
│       ├─ id: 'https://evil.com/users/fake'  ←────────────────────┘       │
│       │         (不等于 self_link_href?) 等等...                          │
│       │                                                                     │
│       └─ preferredUsername: 'fake'                                         │
│                                                                             │
│  等等！这里有个问题：                                                       │
│  Webfinger 的 self link 是 https://evil.com/users/fake                   │
│  我们获取的就是这个 URL 的 actor                                           │
│  actor.id 确实等于 self_link_href                                          │
│                                                                             │
│  ⚠️ 这就是为什么需要第二层 WebFinger 验证！                                 │
│                                                                             │
│  从 actor 解析出的账号信息：                                                │
│    @username = 'fake'                                                      │
│    @domain = 'evil.com'                                                    │
│                                                                             │
│  现在对 @fake@evil.com 执行 WebFinger 查询：                               │
│       │                                                                     │
│       └─ 返回的 self_link_href 应该等于                                    │
│          'https://evil.com/users/fake'                                     │
│                                                                             │
│  ⭐ 这个验证会通过，但用户搜索的是 @alice@example.com！                    │
│                                                                             │
│  这就是为什么第一层 ResolveAccountService 会检测：                         │
│    WebFinger (example.com) 返回的 subject 是不是 'alice@example.com'？   │
│                                                                             │
│  如果 example.com 的 WebFinger 返回的 subject 是 'alice@evil.com'         │
│  那才会被判定为重定向，然后去 evil.com 查询                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 关键数据流 v2

### 4.1 完整账号发现与同步流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    完整账号发现与同步流程 (v2)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  入口场景：                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 1. 用户搜索: @username@domain                                          │ │
│  │ 2. 处理提及: 文本中的 @username@domain                                 │ │
│  │ 3. 后台刷新: AccountRefreshWorker 执行                                │ │
│  │ 4. 收到活动: ActivityPub Create/Update                                 │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ Layer 1: ResolveAccountService                                        │ │
│  │                                                                       │ │
│  │ 职责：                                                                │ │
│  │ - 解析 username@domain                                                │ │
│  │ - 检查本地账号缓存                                                    │ │
│  │ - 执行 WebFinger 查询                                                 │ │
│  │ - 检测账号重定向                                                      │ │
│  │ - 调用 ActivityPub 层获取详细资料                                     │ │
│  │                                                                       │ │
│  │ 关键阈值：STALE_THRESHOLD = 1 天                                     │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│         │                                                                    │
│         ├─── 缓存命中 (不需要更新) ──────────────────────────────────────┐ │
│         │                                                                 │ │
│         ▼                                                                 ▼ │
│  ┌─────────────────────┐                                    ┌─────────────────────────────────┐ │
│  │ 直接返回本地账号记录 │                                    │ Layer 1a: WebFinger 查询       │ │
│  │                     │                                    │                                 │ │
│  │ 不执行任何远端查询   │                                    │ 流程：                          │ │
│  └─────────────────────┘                                    │  1. 标准 URL: /.well-known/webfinger │ │
│                                                              │  2. 404 时回退: host-meta → lrdd 模板  │ │
│                                                              │                                 │ │
│                                                              │ 重定向检测：                    │ │
│                                                              │  - subject 与请求账号不同?     │ │
│                                                              │  - 是 → 第二次 WebFinger 查询  │ │
│                                                              │  - 第二次必须等于第一次返回值   │ │
│                                                              └─────────────────────────────────┘ │
│                                                                              │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ Layer 2: ActivityPub::FetchRemoteAccountService                      │ │
│  │                                                                       │ │
│  │ 职责：                                                                │ │
│  │ - 从 WebFinger self link 获取 ActivityPub actor                      │ │
│  │ - 解析 actor JSON                                                    │ │
│  │ - 验证 context 和 type                                               │ │
│  │ - 执行回环验证 (Loop Back Validation)                                │ │
│  │                                                                       │ │
│  │ 关键验证：                                                            │ │
│  │ - supported_context? (ActivityPub 上下文)                           │ │
│  │ - expected_type? (Application/Group/Organization/Person/Service)   │ │
│  │ - check_webfinger! (回环验证)                                        │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ Layer 3: ActivityPub::ProcessAccountService                          │ │
│  │                                                                       │ │
│  │ 职责：                                                                │ │
│  │ - 创建或更新数据库中的 Account 记录                                   │ │
│  │ - 更新 last_webfingered_at 时间戳                                    │ │
│  │ - 设置账号属性 (inbox/outbox/公钥/个人资料等)                        │ │
│  │ - 下载头像/横幅 (或调度延迟下载)                                      │ │
│  │ - 检测重复账号并触发合并                                              │ │
│  │                                                                       │ │
│  │ 分布式锁：lock:process_account:{uri}                                 │ │
│  │ 限流保护：子域名限制、单次请求发现数限制                              │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 异步后续任务                                                           │ │
│  │                                                                       │ │
│  │ 调度的 Worker：                                                        │ │
│  │ - RedownloadAvatarWorker: 头像下载失败重试 (30秒-10分钟延迟)        │ │
│  │ - RedownloadHeaderWorker: 横幅下载失败重试                           │ │
│  │ - VerifyAccountLinksWorker: 链接验证 (10分钟延迟)                    │ │
│  │ - SynchronizeFeaturedCollectionWorker: 精选集合同步                   │ │
│  │ - AccountMergingWorker: 重复账号合并                                  │ │
│  │ - RefollowWorker: 密钥变更后重新验证关注关系                          │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 后台刷新触发流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    后台刷新触发流程                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  触发事件：收到远程用户的 ActivityPub 活动                                   │
│         │                                                                    │
│         ▼                                                                    │
│  ActivityPub::Activity::Create 或 Update#perform                           │
│         │                                                                    │
│         ▼                                                                    │
│  @account.schedule_refresh_if_stale!                                        │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 条件检查：                                                             │ │
│  │ last_webfingered_at.present? AND                                     │ │
│  │ last_webfingered_at <= BACKGROUND_REFRESH_INTERVAL.ago (1周前)      │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│         │                                                                    │
│    ┌────┴────┐                                                               │
│    │         │                                                               │
│   Yes        No                                                             │
│    │         │                                                               │
│    │         ▼                                                               │
│    │  不调度任何任务，直接返回                                               │
│    │                                                                         │
│    ▼                                                                         │
│  AccountRefreshWorker.perform_in(rand(6.hours), account.id)                │
│         │                                                                    │
│         │  (任务被调度到 Sidekiq，延迟 0-6 小时执行)                        │
│         │                                                                    │
│         ▼ (延迟时间后)                                                       │
│  AccountRefreshWorker#perform(account_id)                                   │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ ⭐ 双重检查！                                                         │ │
│  │ account = Account.find_by(id: account_id)                            │ │
│  │ return if account.nil? OR                                            │ │
│  │           account.last_webfingered_at > 1周前                        │ │
│  │                                                                       │ │
│  │ 注意：这里用的是 > (大于)，调度时用的是 <= (小于等于)                │ │
│  │ 所以如果在等待期间数据已更新，这里会跳过                              │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│         │                                                                    │
│    ┌────┴────┐                                                               │
│    │         │                                                               │
│   Yes        No                                                             │
│    │         │                                                               │
│    │         ▼                                                               │
│    │  数据已更新，跳过刷新                                                   │
│    │                                                                         │
│    ▼                                                                         │
│  ResolveAccountService.new.call(account)                                    │
│         │                                                                    │
│         ▼                                                                    │
│  执行完整的 WebFinger + ActivityPub 获取流程                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 竞态条件与防护机制

### 5.1 潜在竞态场景汇总

| 场景 | 可能的问题 | 防护机制 |
|------|-----------|---------|
| 多个用户同时搜索同一新账号 | 重复查询、数据不一致 | 分布式锁 (`with_redis_lock`) |
| 后台任务等待期间用户主动刷新 | 重复刷新、资源浪费 | 双重检查 (`AccountRefreshWorker#perform`) |
| 同一账号的并发更新 | 数据覆盖、丢失 | 分布式锁 + 数据库事务 |
| 账号迁移导致的重复记录 | 数据不一致 | `AccountMergingWorker` |
| 远端配置错误 | WebFinger 与 ActivityPub 不一致 | 回环验证 (`check_webfinger!`) |

### 5.2 时间线竞态分析

**场景：用户主动刷新与后台任务的交互**

```
时间线：
─────────────────────────────────────────────────────────────────────────►

T0: 账号 A 的状态
    - last_webfingered_at = 10 天前 (超过 1 周阈值)
    - 数据状态: 陈旧

T1: 收到账号 A 的新状态 (ActivityPub::Activity::Create)
    │
    └─► @account.schedule_refresh_if_stale!
         │
         ├─ 检查: 10天前 <= 1周前? → Yes
         │
         └─ 调度: AccountRefreshWorker.perform_in(3小时, accountA.id)
              (任务将在 T1+3h 执行)

T2 (T1 + 1 小时): 用户主动搜索账号 A
    │
    └─► ResolveAccountService.call('A@remote.com')
         │
         ├─ 检查: webfinger_update_due?
         │     - @account.possibly_stale? → 10天前 <= 1天前? → Yes
         │
         ├─ 执行: process_webfinger! + fetch_account!
         │
         ├─ 获取到新数据
         │
         └─ 更新: last_webfingered_at = T2 (现在)
              (数据现在是新鲜的)

T3 (T1 + 3 小时): AccountRefreshWorker 执行
    │
    └─► AccountRefreshWorker#perform(accountA_id)
         │
         ├─ account = Account.find_by(id: accountA_id) → 找到
         │
         ├─ ⭐ 双重检查:
         │     account.last_webfingered_at > BACKGROUND_REFRESH_INTERVAL.ago?
         │     T2 > 1周前? → Yes! (T2 是 2 小时前)
         │
         └─ return (跳过刷新)

结果：
- 用户在 T2 获取了最新数据
- 后台任务在 T3 检测到数据已更新，跳过
- 无重复查询，无资源浪费
- 最终一致性得到保障
```

### 5.3 分布式锁与数据库事务的协同

Mastodon 使用**两层防护**：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    两层防护机制                                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  第一层：Redis 分布式锁                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 位置: ResolveAccountService#fetch_account!                      │   │
│  │       ProcessAccountService#call (with_redis_lock block)       │   │
│  │                                                                   │   │
│  │ 作用: 防止同一账号被并发处理                                      │   │
│  │ 范围: 跨进程、跨服务器                                            │   │
│  │ 粒度: 账号级别 (username@domain 或 uri)                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│         │                                                                 │
│         ▼                                                                 │
│  第二层：ActiveRecord 数据库事务                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 位置: Rails 隐式事务 (保存操作自动包装)                           │   │
│  │                                                                   │   │
│  │ 作用: 确保单条记录的原子性更新                                    │   │
│  │ 范围: 单数据库连接                                                │   │
│  │ 粒度: 单条或多条 SQL 语句                                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  为什么需要两层？                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 场景：两个进程同时处理同一账号                                     │   │
│  │                                                                   │   │
│  │ 只有数据库事务的情况：                                            │   │
│  │ - 进程 A: SELECT → 不存在                                        │   │
│  │ - 进程 B: SELECT → 不存在                                        │   │
│  │ - 进程 A: INSERT → 成功                                          │   │
│  │ - 进程 B: INSERT → RecordNotUnique (失败)                        │   │
│  │                                                                   │   │
│  │ 问题：                                                            │   │
│  │ - 两个进程都执行了完整的 WebFinger + ActivityPub 查询            │   │
│  │ - 浪费了远端服务器资源                                            │   │
│  │ - 进程 B 的用户体验差（看到错误）                                 │   │
│  │                                                                   │   │
│  │ 加上分布式锁的情况：                                              │   │
│  │ - 进程 A: 获取锁 → 执行查询 → INSERT → 释放锁                   │   │
│  │ - 进程 B: 等待锁 → 锁释放后 → SELECT → 发现已存在 → 返回        │   │
│  │                                                                   │   │
│  │ 优势：                                                            │   │
│  │ - 只执行一次远端查询                                              │   │
│  │ - 两个进程都能成功返回                                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.4 最终一致性保证的时序图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    最终一致性时序图                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  图例：                                                                      │
│  ┌──────┐   用户操作                                                        │
│  └──────┘                                                                  │
│  ┌──────────────────┐   异步任务调度                                        │
│  └──────────────────┘                                                        │
│  ════════════════════   数据库操作                                          │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  时间    事件                                                                 │
│  ────    ────                                                                 │
│                                                                              │
│  T0      ┌──────────────────────────────────────────────────────────────┐  │
│          │ 账号 A: last_webfingered_at = 10 天前                        │  │
│          │         数据状态: 陈旧                                         │  │
│          └──────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  T1      ┌──────────────────────────────────────────────────────────────┐  │
│          │ 收到账号 A 的 ActivityPub Create 活动                         │  │
│          │                                                                 │  │
│          │ schedule_refresh_if_stale!                                    │  │
│          │   → 10天前 <= 1周前? → Yes                                    │  │
│          │   → 调度 AccountRefreshWorker (3小时后执行)                   │  │
│          └──────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  T2      ┌──────────────────────────────────────────────────────────────┐  │
│          │ (1小时后) 用户主动搜索 @A@remote                               │  │
│          │                                                                 │  │
│          │ ResolveAccountService.call                                     │  │
│          │   ├─ webfinger_update_due?                                     │  │
│          │   │   └─ 10天前 <= 1天前? → Yes (需要更新)                   │  │
│          │   │                                                            │  │
│          │   ├─ 获取 Redis 锁: lock:resolve:A@remote                    │  │
│          │   │                                                            │  │
│          │   ├─ WebFinger 查询 → 获取新数据                              │  │
│          │   │                                                            │  │
│          │   ├─ ActivityPub 获取 → 获取新资料                            │  │
│          │   │                                                            │  │
│          │   └─ ════════════════════════════════════════════            │  │
│          │         数据库:                                                 │  │
│          │         UPDATE accounts                                        │  │
│          │         SET last_webfingered_at = 'T2'                        │  │
│          │         WHERE id = accountA_id                                 │  │
│          │         ════════════════════════════════════════════            │  │
│          │                                                                 │  │
│          │   └─ 释放 Redis 锁                                             │  │
│          │                                                                 │  │
│          │ 用户获得最新数据 ✓                                              │  │
│          └──────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  T3      ┌──────────────────────────────────────────────────────────────┐  │
│          │ (3小时后) AccountRefreshWorker 执行                           │  │
│          │                                                                 │  │
│          │ perform(accountA_id)                                           │  │
│          │   ├─ account = Account.find_by(id: accountA_id)              │  │
│          │   │                                                            │  │
│          │   ├─ ⭐ 双重检查:                                              │  │
│          │   │   account.last_webfingered_at > 1周前?                   │  │
│          │   │   'T2' > 1周前? → Yes! (T2 是 2 小时前)                 │  │
│          │   │                                                            │  │
│          │   └─ return (跳过刷新)                                         │  │
│          │                                                                 │  │
│          │ 无重复查询 ✓                                                   │  │
│          │ 数据已保持最新 ✓                                               │  │
│          └──────────────────────────────────────────────────────────────┘  │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  最终一致性保障总结：                                                        │
│                                                                              │
│  1. 用户主动操作优先获得最新数据                                             │
│  2. 后台任务通过双重检查避免重复工作                                         │
│  3. 分布式锁防止并发冲突                                                     │
│  4. 所有操作都是幂等的，可以安全重试                                         │
│  5. 回环验证确保 WebFinger 与 ActivityPub 数据一致                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 附录：关键文件索引 (v2)

### 核心服务

| 文件路径 | 职责 | 关键代码 |
|----------|------|----------|
| `app/services/resolve_account_service.rb` | 账号解析主入口 | `process_webfinger!`, `webfinger_update_due?` |
| `app/services/activitypub/fetch_remote_actor_service.rb` | ActivityPub actor 获取 | `check_webfinger!` (回环验证) |
| `app/services/activitypub/process_account_service.rb` | 账号数据处理 | `update_account`, `process_duplicate_accounts!` |

### 阈值与刷新

| 文件路径 | 职责 | 关键代码 |
|----------|------|----------|
| `app/models/account.rb` | 账号模型 | `possibly_stale?`, `schedule_refresh_if_stale!` |
| `app/workers/account_refresh_worker.rb` | 后台刷新任务 | `perform` (双重检查) |
| `app/workers/remote_account_refresh_worker.rb` | 远程账号刷新 | `perform` (从 URI 直接刷新) |

### 锁与一致性

| 文件路径 | 职责 | 关键代码 |
|----------|------|----------|
| `app/models/concerns/lockable.rb` | 分布式锁 | `with_redis_lock` |
| `app/workers/account_merging_worker.rb` | 重复账号合并 | `perform` |
| `app/models/concerns/account/merging.rb` | 账号合并逻辑 | `merge_with!` |

### 协议实现

| 文件路径 | 职责 | 关键代码 |
|----------|------|----------|
| `app/lib/webfinger.rb` | WebFinger 协议 | `Response`, `perform` |
| `app/lib/entity_cache.rb` | 实体缓存 | `mention`, `status`, `emoji` |
| `app/helpers/json_ld_helper.rb` | JSON-LD 处理 | `fetch_resource`, `load_jsonld_context` |

---

*报告版本：v2*
*生成时间：2026-05-03*
*主要更新：重定向判定条件校正、最终一致性保障机制分析*
