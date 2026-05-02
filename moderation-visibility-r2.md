# Mastodon 内容审核与用户屏蔽机制分析报告（修订版）

**基于代码库实际实现分析

---

## 概述

本报告基于 Mastodon 代码库的实际实现，深入分析内容审核、用户屏蔽和内容可见性设置三个层面的规则机制。重点关注：
- 何时过滤（过滤时机）
- 在哪里过滤（过滤位置）
- 规则优先级如何叠加

---

## 第一部分：管理员处置层

### 1.1 管理员处置类型及数据库实现

#### 1.1.1 Suspension（封禁）

**实现证据**（streaming/index.js 第 383 行）：

```javascript
const result = await pgPool.query(
  'SELECT oauth_access_tokens.id, oauth_access_tokens.resource_owner_id, users.account_id, users.chosen_languages, oauth_access_tokens.scopes, COALESCE(user_roles.permissions, 0) AS permissions 
   FROM oauth_access_tokens 
   INNER JOIN users ON oauth_access_tokens.resource_owner_id = users.id 
   INNER JOIN accounts ON accounts.id = users.account_id 
   LEFT OUTER JOIN user_roles ON user_roles.id = users.role_id 
   WHERE oauth_access_tokens.token = $1 
     AND oauth_access_tokens.revoked_at IS NULL 
     AND users.disabled IS FALSE 
     AND accounts.suspended_at IS NULL 
   LIMIT 1', 
  [token]
);
```

**关键发现**：
- `accounts.suspended_at IS NULL` 检查在**认证阶段**就进行
- 被封禁的账号**无法获取 access token**，即无法登录
- 这是最高优先级的过滤，发生在所有其他过滤之前

**对内容可见性的影响**：
- 账号数据在数据库中标记 `suspended_at` 时间戳
- 公开资料页不可见
- 所有帖子、上传、粉丝关系公开移除
- 数据在后台保留 30 天，可恢复

#### 1.1.2 Silence（静默）

**数据库字段**：`accounts.silenced_at`

**实现原理**：
- 静默账号的 `silenced_at` 字段有值
- 静默不影响登录认证（与 suspend 不同）
- 影响仅体现在**时间线查询过滤**中

**对内容可见性的影响**：
- 帖子**不会出现在**：
  - 公共时间线
  - 联邦时间线
  - 非关注者的通知
- 帖子**仍会出现在**：
  - 关注者的主页时间线
  - 用户自己的个人主页

#### 1.1.3 Freeze（冻结）

**特性**：
- 仅适用于本地用户
- 阻止用户进行任何账号操作（发帖、互动等）
- 但所有内容保持不变且公开可见
- 完全可逆

#### 1.1.4 Sensitive 标记（媒体敏感）

**实现**：
- 强制该账户的所有媒体附件始终被标记为"敏感内容"
- 用户需要点击展开才能查看

### 1.2 管理员处置的过滤时机与位置

| 处置类型 | 过滤时机 | 过滤位置 | 数据库字段 |
|---------|---------|---------|-----------|
| **Suspend** | 认证阶段 | 登录时检查 `suspended_at` | `accounts.suspended_at` |
| **Silence** | 查询阶段 | 时间线查询时排除 | `accounts.silenced_at` |
| **Freeze** | 操作阶段 | 阻止用户操作 | - |
| **Sensitive** | 展示阶段 | 前端渲染时标记 | - |

### 1.3 管理员特殊权限

**关键发现**：管理员在审核界面可以绕过普通屏蔽规则。

虽然当前代码库中未直接展示这部分逻辑，但从业务逻辑推断：
- 普通界面：遵循普通用户的屏蔽规则
- 审核界面：可以看到被举报用户的**公开帖子**

---

## 第二部分：用户屏蔽层

### 2.1 核心数据模型

根据代码库中的三个关键表：

| 表名 | 用途 | 方向性 |
|-----|------|--------|
| `blocks` | 用户级屏蔽 | **双向**检查 |
| `mutes` | 用户级静音 | **单向**检查 |
| `account_domain_blocks` | 域级屏蔽 | 单向检查 |

### 2.2 屏蔽/静音的过滤实现逻辑

**实现证据**（streaming/index.js 第 747-759 行）：

```javascript
const queries = [
  client.query(`SELECT 1
                FROM blocks
                WHERE (account_id = $1 AND target_account_id IN (${placeholders(targetAccountIds, 2)}))
                   OR (account_id = $2 AND target_account_id = $1)
                UNION
                SELECT 1
                FROM mutes
                WHERE account_id = $1
                  AND target_account_id IN (${placeholders(targetAccountIds, 2)})`, 
                [req.accountId, payload.account.id].concat(targetAccountIds)),
];
```

**关键发现**：

#### Block（屏蔽）是双向检查

```sql
WHERE (account_id = $1 AND target_account_id IN (...))
   OR (account_id = $2 AND target_account_id = $1)
```

这意味着：
- **条件1**：当前用户（`account_id = $1` 屏蔽了目标用户
- **条件2**：目标用户（`account_id = $2`）屏蔽了当前用户

**结论**：屏蔽是相互的。如果 A 屏蔽了 B，那么：
- A 看不到 B 的内容
- B 也看不到 A 的内容（在需要过滤的频道中）

#### Mute（静音）是单向检查

```sql
WHERE account_id = $1 AND target_account_id IN (...)
```

这意味着：
- 只检查当前用户是否静音了目标用户
- 目标用户静音当前用户不影响当前用户看到目标用户的内容

### 2.3 域屏蔽的实现

**实现证据**（streaming/index.js 第 762-765 行）：

```javascript
if (accountDomain) {
  queries.push(
    client.query(
      'SELECT 1 FROM account_domain_blocks WHERE account_id = $1 AND domain = $2', 
      [req.accountId, accountDomain]
    )
  );
}
```

**关键发现**：
- 域屏蔽检查 `account_domain_blocks` 表
- 检查条件：`account_id = $1 AND domain = $2`
- 只检查当前用户是否屏蔽了该域名

### 2.4 过滤结果判断逻辑

**实现证据**（streaming/index.js 第 776-781 行）：

```javascript
if (values[0].rows.length > 0 || (accountDomain && values[1].rows.length > 0)) {
  return;  // 不传输给客户端
}
```

**关键发现**：
- `values[0]` 是 blocks/mutes 查询结果
- `values[1]` 是 account_domain_blocks 查询结果（如果有 accountDomain）
- **只要任一查询返回结果，就不传输内容**
- 这是 OR 逻辑：屏蔽 OR 静音 OR 域屏蔽 → 过滤

### 2.5 哪些频道需要过滤

**实现证据**（streaming/index.js 第 1091-1167 行）：

| 频道 | needsFiltering | 过滤位置 |
|-----|---------------|---------|
| `user`（主页时间线） | `false` | Rails 后端 |
| `user:notification`（通知） | `false` | Rails 后端 |
| `list`（列表） | `false` | Rails 后端 |
| `direct`（私信） | `false` | Rails 后端 |
| `public`（公共时间线） | `true` | streaming 服务 |
| `public:local`（本地公共） | `true` | streaming 服务 |
| `public:remote`（远程公共） | `true` | streaming 服务 |
| `hashtag`（话题） | `true` | streaming 服务 |

**关键发现**：
- **用户时间线类**（user/notification/list/direct）的过滤主要在 **Rails 后端**进行
- **公共时间线类**（public/hashtag）的过滤在 **streaming 服务**进行额外过滤

### 2.6 自定义过滤器

**实现证据**（streaming/index.js 第 768-897 行）：

```javascript
// 首先检查 payload 是否已有 filtered 属性
if (Object.hasOwn(payload, "filtered")) {
  transmit(event, payload);
  return;
}

// 如果没有，从数据库加载自定义过滤器
queries.push(
  client.query(
    'SELECT filter.id AS id, filter.phrase AS title, filter.context AS context, 
            filter.expires_at AS expires_at, filter.action AS filter_action, 
            keyword.keyword AS keyword, keyword.whole_word AS whole_word 
     FROM custom_filter_keywords keyword 
     JOIN custom_filters filter ON keyword.custom_filter_id = filter.id 
     WHERE filter.account_id = $1 
       AND (filter.expires_at IS NULL OR filter.expires_at > NOW())', 
    [req.accountId]
  )
);
```

**关键发现**：

1. **检查顺序**：
   - 首先检查 `payload.filtered` 是否已存在（说明 Rails 端已处理）
   - 如果没有，才在 streaming 端处理

2. **filter_action 枚举**（第 810-815 行）：
   ```javascript
   // filter_action 是数据库列，值为整数
   // enum { warn: 0, hide: 1 }
   filter_action: ['warn', 'hide'][filter.filter_action],
   ```

3. **上下文过滤**：
   - `context` 字段是数组，决定过滤器应用于哪些场景
   - 可选值：`home`, `notifications`, `public`, `thread`

### 2.7 用户屏蔽层总结

| 机制 | 表名 | 方向性 | 过滤时机 | 过滤位置 |
|-----|------|---------|---------|---------|
| **Block** | `blocks` | 双向 | 内容传输前 | streaming 服务（公共频道）/ Rails（用户频道） |
| **Mute** | `mutes` | 单向 | 内容传输前 | streaming 服务（公共频道）/ Rails（用户频道） |
| **Domain Block** | `account_domain_blocks` | 单向 | 内容传输前 | streaming 服务 |
| **Custom Filter** | `custom_filters` | 单向 | 内容传输前/展示前 | Rails 后端 / streaming 服务 |

---

## 第三部分：内容可见性设置层

### 3.1 四种可见性级别

| 级别 | 数据库值 | 时间线出现 | 联邦传播 |
|-----|---------|-----------|---------|
| **Public** | `'public'` | 所有时间线 | 完全联邦 |
| **Unlisted** | `'unlisted'` | 主页时间线、个人主页 | 联邦但不进入公共时间线 |
| **Private** | `'private'` | 仅关注者主页 | 不联邦 |
| **Direct** | `'direct'` | 不进入时间线 | 不联邦 |

### 3.2 可见性过滤的实现位置

**关键发现**：

1. **用户时间线类频道**（`needsFiltering: false）：
   - 可见性过滤主要在 **Rails 后端**处理
   - 构建时间线时根据 `visibility` 字段过滤

2. **公共时间线类**（`needsFiltering: true`）：
   - 首先由 Rails 端生成事件时已过滤 `visibility = 'public'`
   - streaming 服务接收的事件已经是 `public` 可见性

### 3.3 可见性与其他过滤的关系

**可见性是基础过滤**：
- 一个 `private` 帖子根本不会进入公共时间线的事件流
- 因此其他过滤（屏蔽、静音）是在**可见性过滤之后**应用的

**过滤顺序**：
```
可见性过滤 → 屏蔽/静音过滤 → 自定义过滤器
```

---

## 第四部分：过滤优先级与叠加规则

### 4.1 完整过滤时机总览

#### 阶段 1：认证阶段（最高优先级）

| 检查 | 位置 | 代码位置 |
|-----|------|---------|
| `accounts.suspended_at IS NULL | 登录认证 | streaming/index.js:383 |
| `users.disabled IS FALSE` | 登录认证 | streaming/index.js:383 |

**效果**：未通过则无法获取 access token，无法登录。

#### 阶段 2：Rails 后端处理

**用户时间线（home timeline）：
1. 基于 `visibility` 字段过滤
2. 基于 `silenced_at` 过滤静默用户
3. 基于 `blocks` 表过滤屏蔽关系
4. 基于 `mutes` 表过滤静音关系
5. 基于 `account_domain_blocks` 表过滤域屏蔽
6. 应用自定义过滤器

**公共时间线**：
1. 过滤 `visibility = 'public'`
2. 过滤 `silenced_at IS NULL`
3. 生成事件推送到 Redis channel

#### 阶段 3：streaming 服务处理（公共频道）

**实现证据**（streaming/index.js 第 705-709 行）：

```javascript
// 只有 update 和 status.update 事件需要过滤
if (!needsFiltering || (event !== 'update' && event !== 'status.update')) {
  transmit(event, payload);
  return;
}
```

**过滤顺序**（streaming/index.js 第 747-902 行）：

```
1. 检查 feed 访问设置（filterLocal/filterRemote）
2. 检查语言过滤（chosenLanguages）
3. 检查 blocks 表（双向）
4. 检查 mutes 表（单向）
5. 检查 account_domain_blocks 表
6. 检查 custom_filters（如果 payload 没有 filtered 属性）
```

### 4.2 优先级详细说明

基于代码实现的优先级（从高到低）：

| 优先级 | 规则类型 | 过滤时机 | 代码位置 |
|-------|---------|---------|---------|
| **1** | Suspended（封禁） | 认证阶段 | streaming/index.js:383 |
| **2** | Visibility（可见性） | 事件生成阶段 | Rails 后端 |
| **3** | Silenced（静默） | 事件生成阶段 | Rails 后端 |
| **4** | Block（用户屏蔽） | 事件传输阶段 | streaming/index.js:747-759 |
| **5** | Mute（用户静音） | 事件传输阶段 | streaming/index.js:747-759 |
| **6** | Domain Block（域屏蔽） | 事件传输阶段 | streaming/index.js:762-765 |
| **7** | Custom Filter（自定义过滤器） | 事件传输/展示阶段 | streaming/index.js:768-897 |

### 4.3 关键纠正

#### 纠正 1：Block 是双向的

**之前的不准确描述**：Block 是单向的

**实际实现**（streaming/index.js:747-752）：

```sql
WHERE (account_id = $1 AND target_account_id IN (...))
   OR (account_id = $2 AND target_account_id = $1)
```

**正确理解**：
- A 屏蔽 B → A 看不到 B 的内容，B 也看不到 A 的内容
- 这是 OR 逻辑：只要任一方向的

#### 纠正 2：Mute 是单向的

**实现**（streaming/index.js:753-758）：

```sql
WHERE account_id = $1 AND target_account_id IN (...)
```

**正确理解**：
- A 静音 B → A 看不到 B 的内容
- B 静音 A → 不影响 A 看到 B 的内容

#### 纠正 3：过滤的 OR 逻辑

**实现**（streaming/index.js:776-781）：

```javascript
if (values[0].rows.length > 0 || (accountDomain && values[1].rows.length > 0)) {
  return;  // 不传输
}
```

**正确理解**：
- Block **OR** Mute **OR** Domain Block → 过滤
- 只要任一条件满足，内容就被过滤

#### 纠正 4：不同频道的过滤位置不同

**实现**（streaming/index.js:1091-1167）：

| 频道 | 过滤位置 |
|-----|---------|
| user, user:notification, list, direct | Rails 后端（主要） |
| public, public:local, public:remote, hashtag | streaming 服务 |

**正确理解**：
- 用户时间线的过滤在 Rails 后端进行
- 公共时间线的过滤在 streaming 服务进行额外检查

---

## 第五部分：完整过滤流程图

### 5.1 用户访问主页时间线（User Timeline）

```
┌─────────────────────────────────────────────────────────────────┐
│  用户请求主页时间线                                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  阶段 1：认证检查（Rails 中间件）                                 │
│  ─────────────────────────────────────────────────────────────  │
│  • 检查 access token 是否有效                                     │
│  • 检查 accounts.suspended_at IS NULL                             │
│  • 检查 users.disabled IS FALSE                                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  阶段 2：Rails 后端查询过滤                                      │
│  ─────────────────────────────────────────────────────────────  │
│  1. 基于 visibility 过滤：                                      │
│     • visibility IN ('public', 'unlisted', 'private')         │
│     • 对于 private：检查当前用户是否是关注者                        │
│                                                                  │
│  2. 基于 silenced_at 过滤：                                     │
│     • 静默用户的帖子不出现在公共时间线，但主页时间线仍可见            │
│                                                                  │
│  3. 基于 blocks 过滤（双向）：                                   │
│     • 排除当前用户屏蔽的账号                                       │
│     • 排除屏蔽当前用户的账号                                       │
│                                                                  │
│  4. 基于 mutes 过滤（单向）：                                    │
│     • 排除当前用户静音的账号                                       │
│                                                                  │
│  5. 基于 account_domain_blocks 过滤：                             │
│     • 排除当前用户屏蔽的域名                                       │
│                                                                  │
│  6. 应用自定义过滤器：                                           │
│     • 检查 context = 'home' 的过滤器                              │
│     • filter_action = 'hide' → 从结果中移除                      │
│     • filter_action = 'warn' → 标记为 filtered                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  阶段 3：返回结果给客户端                                        │
│  ─────────────────────────────────────────────────────────────  │
│  • 包含 filtered 属性的状态                                      │
│  • 前端根据 filtered 决定如何展示                                    │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 用户访问公共时间线（Public Timeline）

```
┌─────────────────────────────────────────────────────────────────┐
│  用户订阅公共时间线 WebSocket 连接                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  阶段 1：认证检查（streaming/index.js:383）                      │
│  ─────────────────────────────────────────────────────────────  │
│  • 检查 access token                                            │
│  • 检查 accounts.suspended_at IS NULL                             │
│  • 检查 users.disabled IS FALSE                                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  阶段 2：订阅 Redis Channel                                       │
│  ─────────────────────────────────────────────────────────────  │
│  • 订阅 timeline:public 或 timeline:public:local 等              │
│  • 这些 channel 中的事件已由 Rails 端预过滤                        │
│  • 预过滤条件：visibility = 'public' AND silenced_at IS NULL    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  阶段 3：接收事件（streaming/index.js:688-903）                │
│  ─────────────────────────────────────────────────────────────  │
│  检查是否需要过滤：                                                │
│  • needsFiltering = true（公共时间线需要）                       │
│  • event 必须是 'update' 或 'status.update'                      │
│                                                                  │
│  如果不需要过滤 → 直接传输                                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  阶段 4：应用过滤逻辑（streaming/index.js:714-902）              │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  4.1 检查 feed 访问设置：                                         │
│      • filterLocal：检查本地访问设置                              │
│      • filterRemote：检查远程访问设置                              │
│                                                                  │
│  4.2 检查语言过滤：                                               │
│      • payload.language 是否在 chosenLanguages 中                  │
│                                                                  │
│  4.3 查询数据库（并行执行）：                                     │
│      ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│      │ blocks 查询    │  │ mutes 查询     │  │ domain_blocks   │
│      │ (双向检查)    │  │ (单向检查)      │  │ 查询            │
│      └────────┬────────┘  └────────┬────────┘  └────────┬────────┘
│               │                      │                      │
│               └──────────────────────┬───┴──────────────────────┘
│                                  ▼
│              如果任一查询有结果 → 不传输（return）
│                                                                  │
│  4.4 检查自定义过滤器：                                           │
│      • 如果 payload 已有 filtered 属性 → 直接传输                 │
│      • 否则加载用户的 custom_filters                                 │
│      • 构建正则表达式匹配关键词                                     │
│      • 匹配成功 → 添加 filtered 属性                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  阶段 5：传输给客户端                                            │
│  ─────────────────────────────────────────────────────────────  │
│  • 未被过滤的事件通过 WebSocket 传输                              │
│  • 包含 filtered 属性供前端处理                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 第六部分：核心表结构参考

### 6.1 关键数据库表

#### accounts 表关键字段

| 字段 | 类型 | 用途 |
|-----|------|-----|
| `id` | bigint | 主键 |
| `suspended_at` | datetime | 封禁时间戳，NULL 表示未封禁 |
| `silenced_at` | datetime | 静默时间戳，NULL 表示未静默 |
| `domain` | string | 域名，NULL 表示本地用户 |
| `locked` | boolean | 账号是否锁定（需要审核关注请求） |

#### statuses 表关键字段

| 字段 | 类型 | 用途 |
|-----|------|-----|
| `id` | bigint | 主键 |
| `visibility` | integer/enum | 可见性：public, unlisted, private, direct |
| `sensitive` | boolean | 是否标记为敏感内容 |
| `spoiler_text` | string | 内容警告文本 |
| `deleted_at` | datetime | 删除时间戳（软删除） |
| `account_id` | bigint | 作者账号 ID |

#### blocks 表

| 字段 | 类型 | 用途 |
|-----|------|-----|
| `id` | bigint | 主键 |
| `account_id` | bigint | 发起屏蔽的用户 ID |
| `target_account_id` | bigint | 被屏蔽的用户 ID |
| `created_at` | datetime | 创建时间 |

**关键**：查询时**双向**检查

#### mutes 表

| 字段 | 类型 | 用途 |
|-----|------|-----|
| `id` | bigint | 主键 |
| `account_id` | bigint | 发起静音的用户 ID |
| `target_account_id` | bigint | 被静音的用户 ID |
| `hide_notifications` | boolean | 是否同时隐藏通知 |

**关键**：查询时**单向**检查

#### account_domain_blocks 表

| 字段 | 类型 | 用途 |
|-----|------|-----|
| `id` | bigint | 主键 |
| `account_id` | bigint | 发起屏蔽的用户 ID |
| `domain` | string | 被屏蔽的域名 |

#### custom_filters 表

| 字段 | 类型 | 用途 |
|-----|------|-----|
| `id` | bigint | 主键 |
| `account_id` | bigint | 用户 ID |
| `phrase` | string | 过滤器标题 |
| `context` | string[] | 应用场景：home, notifications, public, thread |
| `action` | integer | 0=warn, 1=hide |
| `expires_at` | datetime | 过期时间 |

#### custom_filter_keywords 表

| 字段 | 类型 | 用途 |
|-----|------|-----|
| `id` | bigint | 主键 |
| `custom_filter_id` | bigint | 关联的过滤器 ID |
| `keyword` | string | 关键词 |
| `whole_word` | boolean | 是否整词匹配 |

---

## 第七部分：总结

### 7.1 核心设计原则

1. **分层过滤**：
   - 认证层（suspend）→ 数据层（visibility）→ 关系层（block/mute）→ 内容层（custom filter）

2. **不同频道不同处理**：
   - 用户时间线：主要在 Rails 后端过滤
   - 公共时间线：在 streaming 服务额外过滤

3. **双向屏蔽 vs 单向静音**：
   - Block：双向检查，保护双方
   - Mute：单向检查，保护发起方

4. **OR 过滤逻辑**：
   - 任一过滤条件满足任一，内容即被过滤
   - 不是 AND 逻辑

### 7.2 优先级最终确认

| 优先级 | 规则 | 实现位置 | 方向性 |
|-------|------|---------|--------|
| **1** | Suspended（封禁） | 认证阶段 | - |
| **2** | Visibility（可见性） | 事件生成 | - |
| **3** | Silenced（静默） | 事件生成 | - |
| **4** | Block（用户屏蔽） | 事件传输 | **双向** |
| **5** | Mute（用户静音） | 事件传输 | **单向** |
| **6** | Domain Block（域屏蔽） | 事件传输 | 单向 |
| **7** | Custom Filter（自定义） | 事件传输/展示 | 单向 |

### 7.3 代码证据索引

| 功能 | 文件位置 | 行号 |
|-----|---------|-----|
| 认证时检查 suspended_at | streaming/index.js | 383 |
| Block 双向检查 SQL | streaming/index.js | 747-752 |
| Mute 单向检查 SQL | streaming/index.js | 753-758 |
| Domain Block 检查 SQL | streaming/index.js | 762-765 |
| 过滤结果判断（OR 逻辑） | streaming/index.js | 776-781 |
| 自定义过滤器处理 | streaming/index.js | 768-897 |
| 频道过滤配置 | streaming/index.js | 1091-1167 |
| filter_action 枚举映射 | streaming/index.js | 810-815 |

---

**报告生成日期**：2026-05-02

**数据来源**：
- `g:/fangzheng/solo-dogfeeding/code/17727-mastodon/streaming/index.js
- Mastodon 官方文档和代码架构
