# 账号发现机制与远端资料同步分析报告 (v3)

## 核心校正与补充

本报告重点校正以下内容：
1. **缓存边界** - 明确 EntityCache、Account.find_remote、ResolveAccountService 的职责划分
2. **锁与事务** - 分析 RedisLock、Sidekiq Lock、ActiveRecord 事务的真实关系（代码库中未使用 savepoint）
3. **失败路径** - 分析退化机制、指数退避、不可用域名标记、异步重试等恢复策略

---

## 目录
1. [缓存边界：EntityCache vs 直接查询 vs 远端发现](#1-缓存边界entitycache-vs-直接查询-vs-远端发现)
2. [锁层级与事务边界](#2-锁层级与事务边界)
3. [失败路径：退化与恢复机制](#3-失败路径退化与恢复机制)
4. [关键数据流 v3](#4-关键数据流-v3)
5. [并发一致性保障的完整图景](#5-并发一致性保障的完整图景)

---

## 1. 缓存边界：EntityCache vs 直接查询 vs 远端发现

### 1.1 三层查询架构

Mastodon 采用**三层查询架构**，每层有明确的职责边界：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        三层查询架构                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Layer 3: 远端发现 (ResolveAccountService)                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 职责：从远端服务器获取新账号或刷新已有账号                              │ │
│  │                                                                       │ │
│  │ 触发条件：                                                             │ │
│  │ - 新账号（本地数据库不存在）                                          │ │
│  │ - 已有账号但 last_webfingered_at 超过 STALE_THRESHOLD (1天)         │ │
│  │ - 强制刷新（skip_cache: true）                                        │ │
│  │                                                                       │ │
│  │ 执行操作：                                                             │ │
│  │ - WebFinger 查询                                                      │ │
│  │ - ActivityPub actor 获取                                              │ │
│  │ - 创建/更新 Account 记录                                              │ │
│  │ - 更新 last_webfingered_at                                            │ │
│  │                                                                       │ │
│  │ 开销：高（涉及网络请求）                                              │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                    ↑                                         │
│                                    │ 缓存未命中时才进入                      │
│                                    │                                         │
│  Layer 2: 数据库直接查询 (Account.find_remote)                             │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 职责：从本地数据库查询已有账号                                         │ │
│  │                                                                       │ │
│  │ 实现：                                                                 │ │
│  │   Account                                                             │ │
│  │     .with_username(username)                                          │ │
│  │     .with_domain(domain)                                              │ │
│  │     .order(id: :asc)                                                  │ │
│  │     .take                                                             │ │
│  │                                                                       │ │
│  │ 触发场景：                                                             │ │
│  │ - 处理提及 (ProcessMentionsService)                                   │ │
│  │ - ResolveAccountService 内部检查                                      │ │
│  │ - 账号搜索 (AccountSearchService)                                     │ │
│  │                                                                       │ │
│  │ 开销：中（数据库查询）                                                │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                    ↑                                         │
│                                    │ 缓存未命中时才进入                      │
│                                    │                                         │
│  Layer 1: EntityCache (Rails.cache)                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 职责：缓存数据库查询结果，减少重复查询                                 │ │
│  │                                                                       │ │
│  │ 缓存类型：                                                             │ │
│  │ - mention(username, domain) → Account 部分字段                       │ │
│  │ - status(url) → Status 对象                                           │ │
│  │ - emoji(shortcodes, domain) → CustomEmoji 列表                       │ │
│  │                                                                       │ │
│  │ 过期时间：MAX_EXPIRATION = 7 天                                      │ │
│  │                                                                       │ │
│  │ 触发场景：                                                             │ │
│  │ - 文本格式化 (TextFormatter#link_to_mention)                         │ │
│  │ - 账号从文本解析 (Account.from_text)                                  │ │
│  │ - 公告提及解析 (Announcement#mentions)                                │ │
│  │ - 状态 URL 解析 (Status#unmangle_mentions_and_links)                │ │
│  │ - 自定义表情查询 (CustomEmoji.from_text)                              │ │
│  │                                                                       │ │
│  │ 开销：低（内存缓存）                                                  │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 EntityCache 详细分析

**实现位置：** `app/lib/entity_cache.rb`

```ruby
class EntityCache
  include Singleton
  
  MAX_EXPIRATION = 7.days.freeze

  def mention(username, domain)
    Rails.cache.fetch(to_key(:mention, username, domain), expires_in: MAX_EXPIRATION) do
      # ⚠️ 注意：这里只查询数据库，不涉及任何远端请求
      Account.select(:id, :username, :domain, :url).find_remote(username, domain)
    end
  end

  def status(url)
    Rails.cache.fetch(to_key(:status, url), expires_in: MAX_EXPIRATION) do
      # 同样只查询数据库
      FetchRemoteStatusService.new.call(url)
    end
  end

  def to_key(type, *ids)
    "#{type}:#{ids.compact.map(&:downcase).join(':')}"
  end
end
```

**缓存键格式：**
- mention: `mention:{username}:{domain}` → 例如 `mention:alice:example.com`
- status: `status:{url}`
- emoji: `emoji:{shortcode}:{domain}`

**关键洞察：**
> EntityCache 缓存的是**数据库查询结果**，不是远端数据。
> 它的存在只是为了减少重复的 SQL 查询，与 "是否需要从远端刷新数据" 是完全独立的问题。

### 1.3 各场景的查询路径选择

#### 场景 A：文本格式化（显示提及）

**路径：** Layer 1 (EntityCache) → 结束

```ruby
# app/lib/text_formatter.rb:142-144
def link_to_mention(entity)
  # ...
  if preloaded_accounts?
    # 从预加载数据中查找
    account = # ...
  else
    # ⭐ 使用 EntityCache
    account = entity_cache.mention(username, domain)
  end
  # ...
end
```

**为什么用 EntityCache？**
- 文本格式化是高频操作（渲染时间线、通知等）
- 只需要账号的基本信息（用于生成链接）
- 不需要最新数据，7 天过期可接受
- 如果账号不存在，显示为纯文本 `@username`

#### 场景 B：处理提及（发布新嘟文时）

**路径：** Layer 2 (Account.find_remote) → Layer 3 (ResolveAccountService)

```ruby
# app/services/process_mentions_service.rb:35-48
def scan_text!
  @status.text = @status.text.gsub(Account::MENTION_RE) do |match|
    # ...
    # ⭐ Layer 2: 直接查询数据库
    mentioned_account = Account.find_remote(username, domain)

    # 如果账号不存在或协议不兼容，尝试解析
    if mention_undeliverable?(mentioned_account)
      begin
        # ⭐ Layer 3: 从远端解析
        mentioned_account = ResolveAccountService.new.call(Regexp.last_match(1))
      rescue Webfinger::Error, *Mastodon::HTTP_CONNECTION_ERRORS, Mastodon::UnexpectedResponseError
        mentioned_account = nil
      end
    end
    # ...
  end
end
```

**为什么不用 EntityCache？**
- 发布嘟文时需要确保提及的账号是**可投递的**
- EntityCache 可能返回 7 天前的数据
- 需要检查 `activitypub?` 协议兼容性
- 如果账号不存在，需要尝试从远端解析

#### 场景 C：账号搜索

**路径：** Layer 2 (Account.find_remote) → Layer 3 (ResolveAccountService) (可选)

```ruby
# app/services/account_search_service.rb:201-217
def exact_match
  # ...
  match = if options[:resolve]
            # ⭐ 用户明确要求解析时，走 Layer 3
            ResolveAccountService.new.call(query)
          elsif domain_is_local?
            # ⭐ 本地账号，Layer 2
            Account.find_local(query_username)
          else
            # ⭐ 远程账号，Layer 2
            Account.find_remote(query_username, query_domain)
          end
  # ...
end
```

**关键点：**
- 默认情况下，搜索只查询本地数据库
- 只有当用户明确要求 `resolve: true` 时，才会从远端解析

#### 场景 D：账号解析主入口

**路径：** Layer 2 → Layer 3 (有条件)

```ruby
# app/services/resolve_account_service.rb:17-54
def call(uri, options = {})
  # ...
  # ⭐ Layer 2: 先检查本地
  @account ||= Account.find_remote(@username, @domain)

  # ⭐ 判断是否需要 Layer 3
  return @account if @account&.local? || @domain.nil? || !webfinger_update_due?

  # ⭐ Layer 3: 执行 WebFinger + ActivityPub
  process_webfinger!(@uri)
  # ...
  fetch_account!
end

def webfinger_update_due?
  # 缓存未命中（需要更新）的条件：
  @options[:skip_cache] ||      # 强制跳过缓存
    @account.nil? ||             # 新账号
    @account.possibly_stale?    # 数据陈旧 (> 1 天)
end
```

### 1.4 缓存边界决策树

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    缓存边界决策树                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  输入: username@domain                                                       │
│       │                                                                      │
│       ▼                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ Q1: 操作目的是什么？                                                   │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│       │                                                                      │
│    ┌──┴────────────────────────────────────────────────────────────────┐   │
│    │                                                                     │   │
│    ▼                                                                     ▼   │
│  显示/格式化                                                         投递/解析  │
│  (文本渲染、链接生成)                                               (发布嘟文、搜索解析) │
│    │                                                                     │   │
│    ▼                                                                     ▼   │
│  ┌───────────────┐                                                ┌───────────────┐ │
│  │ Layer 1       │                                                │ Layer 2       │ │
│  │ EntityCache   │                                                │ Account.      │ │
│  │               │                                                │ find_remote   │ │
│  │ 过期: 7天     │                                                │ 无缓存         │ │
│  │ 只返回已有数据 │                                                │ 实时查询数据库 │ │
│  └───────────────┘                                                └───────────────┘ │
│    │                                                                     │   │
│    │                                                                     │   │
│    ▼                                                                     ▼   │
│  缓存命中? ──Yes──▶ 返回缓存数据                                         │   │
│    │                                                                        │   │
│    No                                                                      │   │
│    │                                                                        │   │
│    ▼                                                                        │   │
│  Layer 2 查询 ──▶ 存在? ──Yes──▶ 存入 EntityCache ──▶ 返回             │   │
│                   │                                        │              │   │
│                   No                                       │              │   │
│                   │                                        │              │   │
│                   ▼                                        │              │   │
│                 返回 nil                                   │              │   │
│                                                            │              │   │
│                                                            │              │   │
│                              ┌─────────────────────────────┘              │   │
│                              │                                            │   │
│                              ▼                                            │   │
│                    ┌───────────────────────────┐                         │   │
│                    │ Q2: 需要从远端获取吗？   │                         │   │
│                    └───────────────────────────┘                         │   │
│                              │                                            │   │
│              ┌───────────────┴───────────────┐                         │   │
│              │                               │                         │   │
│              ▼                               ▼                         │   │
│         不需要 (skip_webfinger)         需要 (默认)                     │   │
│              │                               │                         │   │
│              ▼                               ▼                         │   │
│         返回 nil 或已有账号         ┌─────────────────┐              │   │
│                                      │ Layer 3         │              │   │
│                                      │ ResolveAccount  │              │   │
│                                      │ Service         │              │   │
│                                      │                 │              │   │
│                                      │ 触发条件:       │              │   │
│                                      │ - 新账号        │              │   │
│                                      │ - > 1天未更新   │              │   │
│                                      │ - skip_cache    │              │   │
│                                      │                 │              │   │
│                                      │ 执行:           │              │   │
│                                      │ - WebFinger     │              │   │
│                                      │ - ActivityPub   │              │   │
│                                      │ - 创建/更新账号 │              │   │
│                                      └─────────────────┘              │   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.5 缓存失效机制

**EntityCache 失效：**

| 缓存类型 | 失效方式 | 代码位置 |
|---------|---------|----------|
| emoji | 主动删除 | `app/models/custom_emoji.rb:105` |
| mention | 依赖 TTL 过期 | 无主动失效 |
| status | 依赖 TTL 过期 | 无主动失效 |

**CustomEmoji 的主动失效：**

```ruby
# app/models/custom_emoji.rb:103-106
def remove_entity_cache
  Rails.cache.delete(EntityCache.instance.to_key(:emoji, shortcode, domain))
end
```

**关键问题：**
> EntityCache.mention 没有主动失效机制！
> 这意味着：
> 1. 账号资料更新后，EntityCache 可能返回旧数据（最多 7 天）
> 2. 但这是可接受的，因为 EntityCache 只用于显示/格式化
> 3. 实际的投递和解析操作不依赖 EntityCache

---

## 2. 锁层级与事务边界

### 2.1 三层锁架构

Mastodon 采用**三层锁架构**来保障并发一致性：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        三层锁架构                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ⚠️ 重要：代码库中未使用 SAVEPOINT！                                        │
│  以下分析基于实际代码，无 savepoint。                                        │
│                                                                              │
│  Layer 1: Sidekiq 任务锁 (sidekiq-lock gem)                                │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 职责：防止同一 Worker 任务被多次调度执行                              │ │
│  │                                                                       │ │
│  │ 配置示例：                                                            │ │
│  │   sidekiq_options lock: :until_executed, lock_ttl: 1.day.to_i      │ │
│  │                                                                       │ │
│  │ 锁策略：                                                              │ │
│  │ - :until_executed → 任务开始执行时释放锁                             │ │
│  │ - :until_expired → 锁过期后才释放                                    │ │
│  │ - :until_and_while_executing → 执行前后都加锁                        │ │
│  │                                                                       │ │
│  │ 使用场景：                                                            │ │
│  │ - AccountRefreshWorker     (lock: :until_executed, ttl: 1天)       │ │
│  │ - AccountDeletionWorker     (lock: :until_executed, ttl: 1周)       │ │
│  │ - QuoteRefreshWorker       (lock: :until_executed, ttl: 1天)       │ │
│  │ - SynchronizeFeaturedCollectionWorker (lock: :until_executed)        │ │
│  │                                                                       │ │
│  │ 粒度：任务级别（基于 Worker 类 + 参数）                               │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                    ↓                                         │
│  Layer 2: Redis 分布式锁 (RedisLock + Lockable)                            │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 职责：防止同一账号被并发处理（应用层锁）                              │ │
│  │                                                                       │ │
│  │ 实现：                                                                │ │
│  │   # app/models/concerns/lockable.rb                                  │ │
│  │   def with_redis_lock(lock_name, autorelease: 15.minutes, ...)      │ │
│  │     RedisLock.acquire(redis: redis, key: "lock:#{lock_name}", ...)  │ │
│  │   end                                                                 │ │
│  │                                                                       │ │
│  │ 锁键格式：                                                            │ │
│  │ - 解析账号: lock:resolve:{username}@{domain}                         │ │
│  │ - 处理账号: lock:process_account:{uri}                                │ │
│  │ - 创建状态: lock:create:{object_uri}                                 │ │
│  │ - 分发状态: lock:distribute:{status_id}                              │ │
│  │ - 关系操作: lock:relationship:{follower_id}:{followee_id}           │ │
│  │                                                                       │ │
│  │ 自动释放时间：默认 15 分钟                                            │ │
│  │                                                                       │ │
│  │ 使用场景：                                                            │ │
│  │ - ResolveAccountService#fetch_account!                               │ │
│  │ - ProcessAccountService#call (with_redis_lock block)                │ │
│  │ - ActivityPub::Activity::Create                                      │ │
│  │ - PostStatusService                                                   │ │
│  │                                                                       │ │
│  │ 粒度：操作级别（基于业务标识）                                         │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                    ↓                                         │
│  Layer 3: ActiveRecord 数据库事务                                           │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 职责：确保数据库操作的原子性和一致性                                  │ │
│  │                                                                       │ │
│  │ 实现：                                                                │ │
│  │   - 隐式事务：ActiveRecord 的 save/update 默认在事务中执行           │ │
│  │   - 显式事务：Model.transaction do ... end                            │ │
│  │                                                                       │ │
│  │ 使用场景：                                                            │ │
│  │ - ProcessMentionsService (Status.transaction do)                     │ │
│  │ - 迁移任务                                                            │ │
│  │ - 需要原子性的批量操作                                                │ │
│  │                                                                       │ │
│  │ 粒度：数据库连接级别                                                  │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 各层锁的真实关系

#### 责任划分

| 层级 | 锁类型 | 保护范围 | 失败处理 |
|------|--------|---------|---------|
| Layer 1 | Sidekiq Lock | 防止同一任务被多次调度 | 重复任务被静默忽略 |
| Layer 2 | RedisLock | 防止同一业务操作并发执行 | 抛出 `Mastodon::RaceConditionError` |
| Layer 3 | ActiveRecord Transaction | 确保数据库操作原子性 | 回滚事务 |

#### 关键洞察

> **三层锁是独立的，没有嵌套或依赖关系！**
>
> 例如：
> 1. Sidekiq Lock 防止同一个 `AccountRefreshWorker` 被调度两次
> 2. 但两个不同的 Worker 可能同时处理同一账号
> 3. 这时需要 RedisLock 来保护
> 4. RedisLock 内部的数据库操作由 ActiveRecord 事务保护

### 2.3 典型场景的锁时序

#### 场景：ResolveAccountService 处理新账号

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              ResolveAccountService 锁时序                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  时间    事件                                                                 │
│  ────    ────                                                                 │
│                                                                              │
│  T0      入口: ResolveAccountService.call('alice@remote.com')              │
│           │                                                                  │
│           ▼                                                                  │
│  T1      Layer 2 查询: Account.find_remote('alice', 'remote.com')          │
│           │                                                                  │
│           └─▶ 返回 nil (新账号)                                              │
│               │                                                              │
│               ▼                                                              │
│  T2      process_webfinger!                                                 │
│           │                                                                  │
│           ├─ WebFinger 查询: GET https://remote.com/.well-known/webfinger  │
│           │                                                                  │
│           └─ 获取 self_link: 'https://remote.com/users/alice'              │
│               │                                                              │
│               ▼                                                              │
│  T3      fetch_account! (进入 with_redis_lock)                              │
│           │                                                                  │
│           ├─ 获取 RedisLock: lock:resolve:alice@remote.com                 │
│           │   (自动释放: 15分钟)                                            │
│           │                                                                  │
│           ├─ ActivityPub::FetchRemoteAccountService.call                    │
│           │   │                                                             │
│           │   ├─ GET https://remote.com/users/alice                        │
│           │   │                                                             │
│           │   └─ check_webfinger! (可选)                                   │
│           │       │                                                         │
│           │       └─ 回环验证: self_link_href == actor.id                 │
│           │                                                                  │
│           ├─ ActivityPub::ProcessAccountService.call                        │
│           │   │                                                             │
│           │   ├─ 进入 with_redis_lock: lock:process_account:{uri}         │
│           │   │                                                             │
│           │   ├─ Layer 3 数据库操作:                                        │
│           │   │   │                                                        │
│           │   │   ├─ 创建/更新 Account 记录                                │
│           │   │   │   (隐式事务)                                           │
│           │   │   │                                                        │
│           │   │   └─ 更新 last_webfingered_at = Time.now.utc              │
│           │   │                                                            │
│           │   └─ 释放 process_account 锁                                    │
│           │                                                                  │
│           └─ 释放 resolve 锁                                                │
│               │                                                              │
│               ▼                                                              │
│  T4      返回 Account 对象                                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.4 锁与事务的边界分析

#### RedisLock 与 ActiveRecord 事务的关系

**RedisLock 不管理事务！**

```ruby
# app/models/concerns/lockable.rb
def with_redis_lock(lock_name, autorelease: 15.minutes, raise_on_failure: true)
  with_redis do |redis|
    RedisLock.acquire(...) do |lock|
      if lock.acquired?
        yield  # ⚠️ 这里没有包装在事务中！
      elsif raise_on_failure
        raise Mastodon::RaceConditionError, "..."
      end
    end
  end
end
```

**关键洞察：**
> RedisLock 只是**应用层的互斥锁**，不负责数据库事务。
> 如果 `yield` 块中的数据库操作失败，RedisLock 会自动释放（因为 block 退出），但事务回滚需要 ActiveRecord 自己处理。

#### 显式事务的使用场景

**ProcessMentionsService 是少数显式使用事务的场景：**

```ruby
# app/services/process_mentions_service.rb:17-21
def call(status)
  # ...
  Status.transaction do
    scan_text!
    assign_mentions!
  end
end
```

**为什么需要显式事务？**
- 涉及多个操作：扫描文本 + 分配提及 + 保存状态
- 需要确保所有提及要么全部创建成功，要么全部失败
- 防止部分提及创建、部分失败的不一致状态

### 2.5 无 Savepoint 的设计决策

**代码库中没有显式使用 SAVEPOINT。** 这是一个设计选择：

| 特性 | 用途 | Mastodon 的替代方案 |
|------|------|---------------------|
| SAVEPOINT | 嵌套事务的部分回滚 | 不使用 |
| `rescue ActiveRecord::RecordNotUnique` | 跳过重复记录 | 广泛使用 |
| `rescue` + 继续执行 | 部分失败不影响整体 | 广泛使用 |

**代码示例：**

```ruby
# app/models/concerns/account/merging.rb:22-28
# 账号合并时，跳过重复的关联记录
owned_classes.each do |klass|
  klass.where(account_id: other_account.id).reorder(nil).find_each do |record|
    record.update_attribute(:account_id, id)
  rescue ActiveRecord::RecordNotUnique
    next  # ⭐ 跳过重复，不回滚整个事务
  end
end
```

**设计哲学：**
> Mastodon 倾向于**优雅降级**而非**严格事务**。
> 当部分操作失败时，通过 `rescue` 跳过继续执行，而不是使用 savepoint 回滚到中间点。

---

## 3. 失败路径：退化与恢复机制

### 3.1 失败处理策略总览

Mastodon 采用**多层失败处理**策略：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        失败处理策略层级                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Layer 1: 静默失败 (suppress_errors)                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 职责：非关键操作失败时不抛出异常，返回 nil                            │ │
│  │                                                                       │ │
│  │ 使用场景：                                                            │ │
│  │ - ResolveAccountService (默认 suppress_errors: true)                │ │
│  │ - FetchRemoteAccountService (默认 suppress_errors: true)            │ │
│  │                                                                       │ │
│  │ 效果：操作失败时用户看不到错误，行为上表现为"账号不存在"             │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  Layer 2: 指数退避重试 (ExponentialBackoff)                                │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 职责：临时失败时自动重试，延迟时间指数增长                            │ │
│  │                                                                       │ │
│  │ 实现：                                                                │ │
│  │   # app/workers/concerns/exponential_backoff.rb                      │ │
│  │   sidekiq_retry_in do |count|                                        │ │
│  │     15 + (10 * (count**4)) + rand(10 * (count**4))                 │ │
│  │   end                                                                 │ │
│  │                                                                       │ │
│  │ 重试延迟：                                                            │ │
│  │ - 第 1 次重试: ~15-25 秒                                             │ │
│  │ - 第 2 次重试: ~15-175 秒                                            │ │
│  │ - 第 3 次重试: ~15-825 秒                                            │ │
│  │ - ...                                                                 │ │
│  │                                                                       │ │
│  │ 使用场景：                                                            │ │
│  │ - FetchRepliesWorker        (retry: 3)                               │ │
│  │ - ProcessFeaturedItemWorker (retry: 3)                               │ │
│  │ - RemoteAccountRefreshWorker (retry: 3)                              │ │
│  │ - RefetchAndVerifyQuoteWorker (retry: 5)                            │ │
│  │                                                                       │ │
│  │ 适用错误：临时网络故障、远端服务暂时不可用                            │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  Layer 3: 异步延迟重试 (专用 Worker)                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 职责：特定操作失败时，调度专用 Worker 延迟重试                        │ │
│  │                                                                       │ │
│  │ 使用场景：                                                            │ │
│  │ - 头像下载失败 → RedownloadAvatarWorker                              │ │
│  │ - 横幅下载失败 → RedownloadHeaderWorker                              │ │
│  │ - 链接验证失败 → VerifyAccountLinksWorker                            │ │
│  │                                                                       │ │
│  │ 延迟策略：                                                            │ │
│  │ - PROCESSING_DELAY = (30.seconds)..(10.minutes)  (随机延迟)        │ │
│  │ - VERIFY_DELAY = 10.minutes                                          │ │
│  │                                                                       │ │
│  │ 适用错误：媒体下载超时、CDN 暂时不可用                                │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  Layer 4: 不可用域名标记 (DeliveryFailureTracker)                          │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 职责：持续失败的域名被标记为"不可用"，跳过后续投递                    │ │
│  │                                                                       │ │
│  │ 阈值：                                                                │ │
│  │ - FAILURE_THRESHOLDS = { days: 7, minutes: 5 }                      │ │
│  │                                                                       │ │
│  │ 状态流转：                                                            │ │
│  │   正常 → 警告（部分失败）→ 不可用（超过阈值）→ 恢复（成功一次）     │ │
│  │                                                                       │ │
│  │ 适用错误：域名持续故障、SSL 证书过期、DNS 解析失败                   │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  Layer 5: 死信队列跳过 (dead: false)                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 职责：失败次数用尽后，不加入死信队列，直接丢弃                        │ │
│  │                                                                       │ │
│  │ 配置：sidekiq_options dead: false                                     │ │
│  │                                                                       │ │
│  │ 使用场景：                                                            │ │
│  │ - AccountRefreshWorker                                                │ │
│  │ - QuoteRefreshWorker                                                  │ │
│  │                                                                       │ │
│  │ 设计意图：这些是"最佳努力"操作，失败不影响核心功能                   │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 DeliveryFailureTracker 详细分析

**实现位置：** `app/lib/delivery_failure_tracker.rb`

```ruby
class DeliveryFailureTracker
  FAILURE_THRESHOLDS = {
    days: 7,        # 7 天内失败天数
    minutes: 5,     # 5 分钟内失败次数
  }.freeze

  def track_failure!
    # 记录失败时间
    redis.sadd(exhausted_deliveries_key, failure_time)
    # 超过阈值则标记为不可用
    UnavailableDomain.create(domain: @host) if reached_failure_threshold?
  end

  def track_success!
    # 成功一次就清除所有失败记录
    redis.del(exhausted_deliveries_key)
    UnavailableDomain.find_by(domain: @host)&.destroy
  end

  def available?
    !UnavailableDomain.exists?(domain: @host)
  end
end
```

**状态流转：**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              不可用域名状态流转                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────┐        失败天数 < 7        ┌──────────┐                      │
│  │          │ ─────────────────────────▶ │          │                      │
│  │  正常    │                              │  警告    │                      │
│  │          │ ◀───────────────────────── │          │                      │
│  └──────────┘        成功一次             └──────────┘                      │
│       │                                          │                          │
│       │                                          │ 失败天数 >= 7            │
│       │                                          ▼                          │
│       │                                    ┌──────────┐                      │
│       │                                    │          │                      │
│       │                                    │ 不可用   │                      │
│       │                                    │          │                      │
│       │                                    └──────────┘                      │
│       │                                          │                          │
│       │                                          │ 成功一次                 │
│       │                                          ▼                          │
│       └──────────────────────────────────────────┘                          │
│                                                                              │
│  注意：成功一次就能从"不可用"恢复到"正常"，跳过"警告"阶段                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**在解析流程中的使用：**

```ruby
# app/services/resolve_account_service.rb:115-117
def webfinger_update_due?
  # ⭐ 如果域名被标记为不可用，跳过更新
  return false if @options[:check_delivery_availability] && !DeliveryFailureTracker.available?(@domain)
  # ...
end
```

### 3.3 ResolveAccountService 的失败处理

**多层错误处理：**

```ruby
# app/services/resolve_account_service.rb:17-58
def call(uri, options = {})
  # ...
  process_webfinger!(@uri)
  # ...
  fetch_account!
rescue Webfinger::Error => e
  Rails.logger.debug { "Webfinger query for #{@uri} failed: #{e}" }
  raise unless @options[:suppress_errors]  # ⭐ 默认静默失败
end

# 内部 fetch_account!
def fetch_account!
  with_redis_lock("resolve:#{@username}@#{@domain}") do
    @account = ActivityPub::FetchRemoteAccountService.new.call(
      actor_url, 
      suppress_errors: @options[:suppress_errors]  # ⭐ 传递下去
    )
  end
end
```

**Webfinger 错误类型：**

| 错误类型 | 触发条件 | 默认处理 |
|---------|---------|---------|
| `Webfinger::Error` | 通用错误（JSON 无效、网络错误等） | 静默返回 nil |
| `Webfinger::GoneError` | HTTP 410 (账号已删除) | 触发账号删除流程 |
| `Webfinger::RedirectError` | 重定向次数过多 | 静默返回 nil |

**410 Gone 的特殊处理：**

```ruby
# app/services/resolve_account_service.rb:44-48
if gone_from_origin? && not_yet_deleted?
  queue_deletion!
  return
end

def queue_deletion!
  @account.suspend!(origin: :remote)
  AccountDeletionWorker.perform_async(@account.id, { 'reserve_username' => false, 'skip_activitypub' => true })
end
```

### 3.4 媒体下载的延迟重试

**ProcessAccountService 中的媒体处理：**

```ruby
# app/services/activitypub/process_account_service.rb:164-186
def set_fetchable_attributes!
  # 头像下载
  begin
    avatar_url, avatar_description = image_url_and_description('icon')
    @account.avatar_remote_url = avatar_url || '' unless skip_download?
    @account.avatar = nil if @account.avatar_remote_url.blank?
    @account.avatar_description = avatar_description || ''
  rescue Mastodon::UnexpectedResponseError, *Mastodon::HTTP_CONNECTION_ERRORS
    # ⭐ 失败时调度延迟重试
    RedownloadAvatarWorker.perform_in(rand(PROCESSING_DELAY), @account.id)
  end

  # 横幅下载（同样的模式）
  begin
    # ...
  rescue Mastodon::UnexpectedResponseError, *Mastodon::HTTP_CONNECTION_ERRORS
    RedownloadHeaderWorker.perform_in(rand(PROCESSING_DELAY), @account.id)
  end
end
```

**延迟范围：**
- `PROCESSING_DELAY = (30.seconds)..(10.minutes)`
- 随机延迟，避免同时重试造成服务器压力

### 3.5 失败路径的最终一致性保障

#### 场景：账号解析时 WebFinger 失败

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              WebFinger 失败的处理流程                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  时间    事件                                                                 │
│  ────    ────                                                                 │
│                                                                              │
│  T0      用户搜索 @alice@remote.com                                          │
│           │                                                                  │
│           ▼                                                                  │
│  T1      ResolveAccountService.call                                          │
│           │                                                                  │
│           ▼                                                                  │
│  T2      WebFinger 查询失败（网络超时）                                      │
│           │                                                                  │
│           ├─ 日志记录: "Webfinger query for alice@remote.com failed: ..."  │
│           │                                                                  │
│           └─ suppress_errors: true → 返回 nil                               │
│               │                                                              │
│               ▼                                                              │
│  T3      用户看到"账号不存在"或搜索结果为空                                  │
│           │                                                                  │
│           │ 恢复路径：                                                       │
│           │                                                                  │
│           │ 路径 A: 用户稍后再次搜索                                         │
│           │   └─ 如果网络已恢复，解析成功                                   │
│           │                                                                  │
│           │ 路径 B: 后台收到该用户的 ActivityPub 活动                       │
│           │   └─ ActivityPub::Activity::Create                              │
│           │       └─ @account.schedule_refresh_if_stale!                   │
│           │           └─ 如果 last_webfingered_at 超过 1 周，调度刷新     │
│           │                                                                  │
│           │ 路径 C: 远程用户主动 @ 本地用户                                  │
│           │   └─ ProcessMentionsService                                     │
│           │       └─ 检测到账号不可投递                                      │
│           │           └─ 尝试 ResolveAccountService                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 场景：域名被标记为不可用

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              不可用域名的退化与恢复                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  退化状态：                                                                  │
│  - 域名被加入 UnavailableDomain 表                                          │
│  - 投递到该域名的活动被跳过                                                  │
│  - ResolveAccountService 可能跳过更新（取决于 check_delivery_availability） │
│                                                                              │
│  恢复条件：                                                                  │
│  - 对该域名的任意一次投递成功                                                │
│  - DeliveryFailureTracker.track_success! 被调用                            │
│                                                                              │
│  恢复效果：                                                                  │
│  - UnavailableDomain 记录被删除                                             │
│  - Redis 中的失败记录被清除                                                 │
│  - 域名恢复为"正常"状态                                                     │
│                                                                              │
│  ⚠️ 注意：一次成功就能完全恢复，无需累积成功次数                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 关键数据流 v3

### 4.1 完整账号发现与同步流程（含失败路径）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    完整账号发现与同步流程 (v3)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  入口场景：                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 1. 用户搜索: @username@domain (可选 resolve: true)                    │ │
│  │ 2. 处理提及: ProcessMentionsService (发布嘟文时)                       │ │
│  │ 3. 后台刷新: AccountRefreshWorker (收到活动时调度)                     │ │
│  │ 4. ActivityPub 接收: Update 活动触发直接更新                          │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ Layer 1: 缓存/直接查询                                                │ │
│  │                                                                       │ │
│  │ 根据操作目的选择：                                                     │ │
│  │ - 显示/格式化 → EntityCache.mention (TTL: 7天)                       │ │
│  │ - 投递/解析 → Account.find_remote (实时数据库查询)                    │ │
│  │                                                                       │ │
│  │ 注意：EntityCache 只缓存数据库查询结果，不涉及远端请求                │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│         │                                                                    │
│         ├── 账号存在且不需要刷新 ──▶ 直接返回 ──▶ 结束                    │
│         │                                                                    │
│         ▼ (账号不存在 或 > 1天未更新)                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ Layer 2: 远端发现 (ResolveAccountService)                            │ │
│  │                                                                       │ │
│  │ 锁保护：Sidekiq Lock (任务级别) + RedisLock (操作级别)              │ │
│  │                                                                       │ │
│  │ 步骤：                                                                │ │
│  │ 1. WebFinger 查询                                                     │ │
│  │    - 标准 URL: /.well-known/webfinger                                │ │
│  │    - 回退机制: 404 时尝试 host-meta → lrdd 模板                     │ │
│  │    - 重定向检测: subject 不同时执行二次验证                          │ │
│  │    - 回环验证: self_link_href 必须等于最终确认                       │ │
│  │                                                                       │ │
│  │ 2. ActivityPub 获取                                                   │ │
│  │    - GET actor_url (Accept: application/activity+json)               │ │
│  │    - 验证 context, type, preferredUsername                          │ │
│  │    - 可选: check_webfinger! (二次回环验证)                           │ │
│  │                                                                       │ │
│  │ 失败处理：                                                            │ │
│  │ - Webfinger::Error → 日志记录 + suppress_errors (默认返回 nil)      │ │
│  │ - Webfinger::GoneError → 触发账号删除流程                            │ │
│  │ - 网络错误 → 依赖指数退避重试 (Worker 场景)                          │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│         │                                                                    │
│         ▼ (获取成功)                                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ Layer 3: 账号处理 (ProcessAccountService)                            │ │
│  │                                                                       │ │
│  │ 锁保护：RedisLock (lock:process_account:{uri})                       │ │
│  │ 事务：ActiveRecord 隐式事务                                          │ │
│  │                                                                       │ │
│  │ 步骤：                                                                │ │
│  │ 1. 创建或更新 Account 记录                                           │ │
│  │    - 新账号: INSERT                                                   │ │
│  │    - 已有账号: UPDATE                                                 │ │
│  │                                                                       │ │
│  │ 2. 更新时间戳                                                        │ │
│  │    - last_webfingered_at = Time.now.utc                             │ │
│  │                                                                       │ │
│  │ 3. 设置属性                                                          │ │
│  │    - 协议属性: inbox/outbox/publicKey/endpoints                     │ │
│  │    - 个人资料: display_name/note/fields/avatar/header               │ │
│  │                                                                       │ │
│  │ 4. 媒体下载（可选延迟重试）                                          │ │
│  │    - 头像下载失败 → RedownloadAvatarWorker (30秒-10分钟延迟)       │ │
│  │    - 横幅下载失败 → RedownloadHeaderWorker (30秒-10分钟延迟)       │ │
│  │                                                                       │ │
│  │ 5. 异步任务调度                                                      │ │
│  │    - 精选集合: SynchronizeFeaturedCollectionWorker                   │ │
│  │    - 链接验证: VerifyAccountLinksWorker (10分钟延迟)                 │ │
│  │    - 重复账号: AccountMergingWorker (如果存在相同 uri)               │ │
│  │    - 密钥变更: RefollowWorker (重新验证关注关系)                     │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 失败退化与恢复路径                                                    │ │
│  │                                                                       │ │
│  │ 临时失败（网络超时、服务暂时不可用）：                                 │ │
│  │ - Worker 场景: ExponentialBackoff (指数退避重试)                    │ │
│  │ - 媒体下载: 专用 Worker 延迟重试                                     │ │
│  │ - 用户场景: 下次搜索或收到活动时重试                                 │ │
│  │                                                                       │ │
│  │ 永久失败（410 Gone、404 Not Found）：                                 │ │
│  │ - 410: 触发账号删除流程                                              │ │
│  │ - 其他: suppress_errors 返回 nil，不影响其他功能                     │ │
│  │                                                                       │ │
│  │ 持续失败（域名级故障）：                                               │ │
│  │ - 7 天内失败天数 >= 7 → 标记为 UnavailableDomain                    │ │
│  │ - 投递被跳过                                                          │ │
│  │ - 恢复: 任意一次成功投递 → 清除标记                                  │ │
│  │                                                                       │ │
│  │ 最终一致性保障：                                                      │ │
│  │ - 双重检查: 任务调度时和执行时都检查刷新阈值                         │ │
│  │ - 多入口恢复: 用户搜索、后台活动、提及处理都可能触发解析            │ │
│  │ - 幂等操作: ProcessAccountService 可安全重复执行                    │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 并发一致性保障的完整图景

### 5.1 各层保障机制汇总

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    并发一致性保障机制汇总                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ═══════════════════════════════════════════════════════════════════════    │
│  一、锁机制                                                                   │
│  ═══════════════════════════════════════════════════════════════════════    │
│                                                                              │
│  1. Sidekiq 任务锁 (Layer 1)                                                │
│     ┌──────────────────────────────────────────────────────────────────┐   │
│     │ 配置: sidekiq_options lock: :until_executed, lock_ttl: 1.day    │   │
│     │ 范围: 同一 Worker 类 + 同一参数                                    │   │
│     │ 目的: 防止同一个刷新任务被多次调度                                  │   │
│     │ 失效: 任务开始执行时释放锁                                         │   │
│     └──────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  2. Redis 分布式锁 (Layer 2)                                                │
│     ┌──────────────────────────────────────────────────────────────────┐   │
│     │ 实现: RedisLock.acquire + Lockable#with_redis_lock               │   │
│     │ 范围: lock:{业务标识} (如 lock:resolve:alice@example.com)        │   │
│     │ 目的: 防止同一账号被并发处理                                        │   │
│     │ 自动释放: 默认 15 分钟                                             │   │
│     │ 失败处理: 抛出 Mastodon::RaceConditionError                       │   │
│     └──────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  3. ActiveRecord 事务 (Layer 3)                                             │
│     ┌──────────────────────────────────────────────────────────────────┐   │
│     │ 实现: 隐式事务 (save/update) + 显式事务 (Model.transaction)       │   │
│     │ 范围: 数据库连接级别                                                │   │
│     │ 目的: 确保数据库操作原子性                                          │   │
│     │ 失败处理: 回滚事务                                                  │   │
│     └──────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ═══════════════════════════════════════════════════════════════════════    │
│  二、缓存与时间戳                                                            │
│  ═══════════════════════════════════════════════════════════════════════    │
│                                                                              │
│  1. EntityCache (查询缓存)                                                  │
│     ┌──────────────────────────────────────────────────────────────────┐   │
│     │ 过期: 7 天                                                         │   │
│     │ 内容: 数据库查询结果                                                │   │
│     │ 用途: 显示/格式化场景                                              │   │
│     │ 注意: 不涉及远端数据刷新                                            │   │
│     └──────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  2. last_webfingered_at (刷新阈值)                                         │
│     ┌──────────────────────────────────────────────────────────────────┐   │
│     │ STALE_THRESHOLD = 1 天 (用户主动查询时)                           │   │
│     │ BACKGROUND_REFRESH_INTERVAL = 1 周 (后台刷新时)                  │   │
│     │ 用途: 决定是否需要从远端获取新数据                                  │   │
│     └──────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ═══════════════════════════════════════════════════════════════════════    │
│  三、失败与恢复                                                              │
│  ═══════════════════════════════════════════════════════════════════════    │
│                                                                              │
│  1. 指数退避重试                                                            │
│     ┌──────────────────────────────────────────────────────────────────┐   │
│     │ 实现: ExponentialBackoff concern                                  │   │
│     │ 公式: 15 + (10 * (count**4)) + rand(10 * (count**4))           │   │
│     │ 适用: 临时网络故障                                                 │   │
│     └──────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  2. 专用延迟 Worker                                                          │
│     ┌──────────────────────────────────────────────────────────────────┐   │
│     │ RedownloadAvatarWorker: 30秒-10分钟随机延迟                      │   │
│     │ RedownloadHeaderWorker: 30秒-10分钟随机延迟                      │   │
│     │ VerifyAccountLinksWorker: 10分钟固定延迟                         │   │
│     │ 适用: 媒体下载、链接验证等非关键操作                               │   │
│     └──────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  3. 不可用域名标记                                                          │
│     ┌──────────────────────────────────────────────────────────────────┐   │
│     │ 阈值: 7 天内失败天数 >= 7                                          │   │
│     │ 存储: UnavailableDomain 表 + Redis Set                            │   │
│     │ 恢复: 任意一次成功投递                                             │   │
│     │ 适用: 持续故障的域名                                               │   │
│     └──────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ═══════════════════════════════════════════════════════════════════════    │
│  四、幂等性与一致性                                                          │
│  ═══════════════════════════════════════════════════════════════════════    │
│                                                                              │
│  1. 双重检查                                                                │
│     ┌──────────────────────────────────────────────────────────────────┐   │
│     │ 调度时: last_webfingered_at <= 1周前? → 调度任务                  │   │
│     │ 执行时: last_webfingered_at > 1周前? → 跳过执行                  │   │
│     │ 目的: 防止任务等待期间数据已被更新                                 │   │
│     └──────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  2. 操作幂等                                                                 │
│     ┌──────────────────────────────────────────────────────────────────┐   │
│     │ ProcessAccountService: 可安全重复执行                             │   │
│     │  - last_webfingered_at 每次更新为当前时间                         │   │
│     │  - 媒体下载失败调度重试                                            │   │
│     │  - 重复记录通过 rescue RecordNotUnique 跳过                       │   │
│     └──────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  3. 多入口恢复                                                               │
│     ┌──────────────────────────────────────────────────────────────────┐   │
│     │ 账号解析可从多个入口触发：                                          │   │
│     │  - 用户搜索 (resolve: true)                                        │   │
│     │  - 处理提及 (账号不存在时)                                          │   │
│     │  - 后台刷新 (收到活动时)                                            │   │
│     │  - ActivityPub Update 活动                                         │   │
│     │ 目的: 即使一个入口失败，其他入口仍可能恢复一致性                   │   │
│     └──────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 最终一致性的时间线保证

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    最终一致性时间线保证                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  场景：远程账号 @alice@remote.com 资料更新，但本地缓存陈旧                 │
│                                                                              │
│  时间线：                                                                    │
│  ───────                                                                     │
│                                                                              │
│  T0 (远端更新)                                                               │
│      - 远程服务器更新了 alice 的个人资料                                    │
│      - 本地数据库: last_webfingered_at = 3 天前                            │
│      - EntityCache: 可能有 3 天前的缓存                                    │
│                                                                              │
│  T1 (用户查看时间线)                                                         │
│      ┌──────────────────────────────────────────────────────────────────┐ │
│      │ TextFormatter 渲染 @alice 提及                                    │ │
│      │   │                                                                 │ │
│      │   ├─ EntityCache.mention('alice', 'remote.com')                  │ │
│      │   │   │                                                             │ │
│      │   │   └─ 缓存命中 → 返回 3 天前的账号数据                         │ │
│      │   │                                                                 │ │
│      │   └─ 链接指向正确的账号页面（URL 是正确的）                         │ │
│      │                                                                     │ │
│      │ 影响：用户可能看到旧的显示名称，但链接是正确的                     │ │
│      │       这是可接受的退化（7 天 TTL）                                 │ │
│      └──────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  T2 (用户主动搜索 @alice@remote.com)                                        │
│      ┌──────────────────────────────────────────────────────────────────┐ │
│      │ AccountSearchService 执行                                         │ │
│      │   │                                                                 │ │
│      │   ├─ exact_match                                                   │ │
│      │   │   │                                                             │ │
│      │   │   ├─ 如果 options[:resolve] = true                            │ │
│      │   │   │   └─ ResolveAccountService.call                           │ │
│      │   │   │       │                                                    │ │
│      │   │   │       ├─ webfinger_update_due?                            │ │
│      │   │   │       │   last_webfingered_at (3天前) <= 1天前? → Yes   │ │
│      │   │   │       │                                                    │ │
│      │   │   │       └─ 执行 WebFinger + ActivityPub                     │ │
│      │   │   │           │                                                │ │
│      │   │   │           └─ 获取新资料                                   │ │
│      │   │   │               │                                            │ │
│      │   │   │               └─ 更新 last_webfingered_at = 现在         │ │
│      │   │   │                                                            │ │
│      │   │   └─ 否则                                                       │ │
│      │   │       └─ Account.find_remote (直接返回本地数据)               │ │
│      │   │                                                                 │ │
│      │   └─ 用户获得最新数据 ✓                                            │ │
│      └──────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  T3 (收到远程账号的新嘟文)                                                   │
│      ┌──────────────────────────────────────────────────────────────────┐ │
│      │ ActivityPub::Activity::Create 处理                                │ │
│      │   │                                                                 │ │
│      │   ├─ @account.schedule_refresh_if_stale!                          │ │
│      │   │   │                                                             │ │
│      │   │   ├─ 检查: last_webfingered_at <= 1周前?                     │ │
│      │   │   │   3天前 <= 1周前? → Yes                                   │ │
│      │   │   │                                                             │ │
│      │   │   └─ 调度 AccountRefreshWorker.perform_in(rand(6.hours), id)  │ │
│      │   │       │                                                         │ │
│      │   │       └─ 任务将在 0-6 小时内随机时间执行                      │ │
│      │   │                                                                 │ │
│      │   └─ 嘟文正常创建，但账号资料仍可能陈旧                            │ │
│      │                                                                     │ │
│      │ 注意：这是后台刷新，不影响当前操作                                  │ │
│      └──────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  T4 (几小时后，AccountRefreshWorker 执行)                                   │
│      ┌──────────────────────────────────────────────────────────────────┐ │
│      │ AccountRefreshWorker#perform                                      │ │
│      │   │                                                                 │ │
│      │   ├─ account = Account.find_by(id: account_id)                   │ │
│      │   │   │                                                             │ │
│      │   │   └─ ⭐ 双重检查                                              │ │
│      │   │       │                                                         │ │
│      │   │       ├─ 情况 A: 上次刷新后用户已主动搜索过                   │ │
│      │   │       │   │                                                     │ │
│      │   │       │   ├─ last_webfingered_at = T2 (现在 - 2小时)         │ │
│      │   │       │   │                                                     │ │
│      │   │       │   └─ 检查: last_webfingered_at > 1周前? → Yes!       │ │
│      │   │       │       │                                                 │ │
│      │   │       │       └─ return (跳过刷新)                            │ │
│      │   │       │                                                         │ │
│      │   │       └─ 情况 B: 期间没有其他刷新                             │ │
│      │   │           │                                                     │ │
│      │   │           ├─ last_webfingered_at = T0 (现在 - 3天 - 几小时)  │ │
│      │   │           │                                                     │ │
│      │   │           └─ 检查: last_webfingered_at > 1周前? → No         │ │
│      │   │               │                                                 │ │
│      │   │               └─ ResolveAccountService.new.call(account)      │ │
│      │   │                   │                                             │ │
│      │   │                   └─ 获取最新资料 ✓                            │ │
│      │   │                                                                 │ │
│      │   └─ 最终一致性保证：要么用户已主动更新，要么后台更新              │ │
│      │                                                                     │ │
│      │ 最坏情况延迟：6 小时 (最大随机延迟) + 任务执行时间                 │ │
│      └──────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ═══════════════════════════════════════════════════════════════════════    │
│  最终一致性保证总结：                                                        │
│  ═══════════════════════════════════════════════════════════════════════    │
│                                                                              │
│  1. EntityCache 数据最多陈旧 7 天（仅影响显示）                            │
│  2. 用户主动搜索时，数据最多陈旧 1 天                                       │
│  3. 后台刷新时，数据最多陈旧 1 周 + 6 小时随机延迟                        │
│  4. 双重检查确保不会重复刷新                                                │
│  5. 多入口（用户搜索、后台活动、提及处理）确保即使一个入口失败，          │
│     其他入口仍能恢复一致性                                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 附录：关键代码索引 (v3)

### 缓存相关

| 文件路径 | 职责 | 关键代码 |
|----------|------|----------|
| `app/lib/entity_cache.rb` | EntityCache 实现 | `mention`, `status`, `emoji` |
| `app/lib/text_formatter.rb` | 文本格式化 | `link_to_mention` (使用 EntityCache) |
| `app/models/account.rb` | 账号模型 | `from_text` (使用 EntityCache), `find_remote` (直接查询) |
| `app/models/custom_emoji.rb` | 自定义表情 | `from_text` (使用 EntityCache), `remove_entity_cache` |

### 锁与事务

| 文件路径 | 职责 | 关键代码 |
|----------|------|----------|
| `app/models/concerns/lockable.rb` | RedisLock 封装 | `with_redis_lock` |
| `app/workers/concerns/exponential_backoff.rb` | 指数退避 | `sidekiq_retry_in` |
| `app/workers/account_refresh_worker.rb` | 后台刷新任务 | `sidekiq_options lock: :until_executed` |
| `app/services/process_mentions_service.rb` | 处理提及 | `Status.transaction do` (显式事务) |

### 失败处理

| 文件路径 | 职责 | 关键代码 |
|----------|------|----------|
| `app/lib/delivery_failure_tracker.rb` | 不可用域名跟踪 | `track_failure!`, `track_success!`, `available?` |
| `app/services/resolve_account_service.rb` | 账号解析 | `rescue Webfinger::Error`, `suppress_errors` |
| `app/services/activitypub/process_account_service.rb` | 账号处理 | `RedownloadAvatarWorker`, `RedownloadHeaderWorker` |
| `app/workers/remote_account_refresh_worker.rb` | 远程刷新 | `include ExponentialBackoff` |

---

*报告版本：v3*
*生成时间：2026-05-03*
*主要更新：缓存边界校正、锁与事务关系分析、失败路径退化与恢复机制*
