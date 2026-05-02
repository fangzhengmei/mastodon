# Mastodon 内容可见性过滤机制分析报告（修订版 v3）

**基于代码库确凿证据分析**

---

## 核心发现摘要

### 两层过滤架构

Mastodon 采用**两层过滤架构**：

| 层级 | 位置 | 负责内容 | 代码证据 |
|-----|------|---------|---------|
| **第一层** | Rails 后端 | 所有事件的生成、visibility 过滤、silenced_at 过滤、基础过滤 | streaming/index.js:695-704 注释 |
| **第二层** | Streaming 服务 | 特定频道的额外过滤（blocks/mutes/domain blocks） | streaming/index.js:705-902 |

### 关键注释（确凿证据）

**streaming/index.js:695-704**：
```javascript
// Streaming only needs to apply filtering to some channels and only to
// some events. This is because majority of the filtering happens on the
// Ruby on Rails side when producing the event for streaming.
//
// The only events that require filtering from the streaming server are
// `update` and `status.update`, all other events are transmitted to the
// client as soon as they're received (pass-through).
```

**解读**：
- **大多数过滤发生在 Ruby on Rails 端**
- Streaming 服务只对特定频道和特定事件进行额外过滤
- 其他事件直接透传（pass-through）

---

## 第一部分：管理员处置层

### 1.1 Suspension（封禁）

#### 实现位置
**streaming/index.js:383**

#### 代码证据
```javascript
const result = await pgPool.query(
  'SELECT oauth_access_tokens.id, oauth_access_tokens.resource_owner_id, 
   users.account_id, users.chosen_languages, oauth_access_tokens.scopes, 
   COALESCE(user_roles.permissions, 0) AS permissions 
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

#### 关键分析

| 检查项 | SQL 条件 | 效果 |
|-------|---------|------|
| Token 有效性 | `oauth_access_tokens.revoked_at IS NULL` | 已撤销的 token 无效 |
| 用户禁用 | `users.disabled IS FALSE` | 被禁用的用户无法登录 |
| 账号封禁 | `accounts.suspended_at IS NULL` | 被封禁的账号无法登录 |

#### 过滤时机与位置

| 属性 | 说明 |
|-----|------|
| **过滤时机** | 认证阶段（获取 access token 时） |
| **过滤位置** | Streaming 服务的 `accountFromToken` 函数 |
| **优先级** | 最高优先级（未通过则无法进行任何后续操作） |

#### 对内容可见性的影响

- **直接效果**：被封禁的账号无法获取有效的 access token
- **间接效果**：无法登录、无法访问任何需要认证的 API
- **内容层面**：被封禁账号的内容是否可见取决于 Rails 后端的处理（本代码库未直接展示）

### 1.2 Silence（静默）

#### 代码库证据状态

**当前代码库中未发现** streaming 服务对 `silenced_at` 的检查逻辑。

#### 推断（基于架构）

根据两层过滤架构的注释（streaming/index.js:695-704）：
- "majority of the filtering happens on the Ruby on Rails side"
- 可以推断 `silenced_at` 的过滤**发生在 Rails 后端**

#### 对内容可见性的影响（基于 Mastodon 行为）

- 静默用户的帖子**不会出现在**：
  - 公共时间线
  - 联邦时间线
  - 非关注者的通知
- 静默用户的帖子**仍会出现在**：
  - 关注者的主页时间线
  - 用户自己的个人主页

### 1.3 管理员处置层总结

| 处置类型 | 过滤时机 | 过滤位置 | 数据库字段 | 代码证据 |
|---------|---------|---------|-----------|---------|
| **Suspend** | 认证阶段 | Streaming 服务 | `accounts.suspended_at` | streaming/index.js:383 |
| **Silence** | 事件生成阶段 | Rails 后端 | `accounts.silenced_at` | 推断（基于架构注释） |

---

## 第二部分：用户屏蔽层

### 2.1 关键配置：needsFiltering

#### 实现位置
**streaming/index.js:1091-1166**

#### 代码证据

```javascript
switch (name) {
case 'user':
  resolve({
    channelIds: channelsForUserStream(req),
    options: { needsFiltering: false },  // 不需要额外过滤
  });
  break;
case 'user:notification':
  resolve({
    channelIds: [`timeline:${req.accountId}:notifications`],
    options: { needsFiltering: false },  // 不需要额外过滤
  });
  break;
case 'direct':
  resolve({
    channelIds: [`timeline:direct:${req.accountId}`],
    options: { needsFiltering: false },  // 不需要额外过滤
  });
  break;
case 'list':
  resolve({
    channelIds: [`timeline:list:${params.list}`],
    options: { needsFiltering: false },  // 不需要额外过滤
  });
  break;
case 'public':
  resolveFeed('public', 'timeline:public', { needsFiltering: true });  // 需要额外过滤
  break;
case 'public:local':
  resolveFeed('public', 'timeline:public:local', { needsFiltering: true });  // 需要额外过滤
  break;
case 'hashtag':
  resolveFeed('hashtag', `timeline:hashtag:${normalizeHashtag(params.tag)}`, { needsFiltering: true });  // 需要额外过滤
  break;
// ... 其他公共/话题频道
}
```

#### 频道分类

| 频道类型 | 频道名称 | needsFiltering | 过滤位置 |
|---------|---------|---------------|---------|
| **用户时间线类** | `user` | `false` | Rails 后端 |
| | `user:notification` | `false` | Rails 后端 |
| | `direct` | `false` | Rails 后端 |
| | `list` | `false` | Rails 后端 |
| **公共时间线类** | `public` | `true` | Rails 后端 + Streaming 服务 |
| | `public:local` | `true` | Rails 后端 + Streaming 服务 |
| | `public:remote` | `true` | Rails 后端 + Streaming 服务 |
| | `hashtag` | `true` | Rails 后端 + Streaming 服务 |
| | `hashtag:local` | `true` | Rails 后端 + Streaming 服务 |

### 2.2 事件类型过滤

#### 实现位置
**streaming/index.js:705-709**

#### 代码证据

```javascript
if (!needsFiltering || (event !== 'update' && event !== 'status.update')) {
  // @ts-expect-error
  transmit(event, payload);
  return;
}
```

#### 关键分析

只有同时满足以下两个条件，才会在 Streaming 服务进行额外过滤：

| 条件 | 说明 |
|-----|------|
| `needsFiltering === true` | 公共时间线类频道 |
| `event === 'update'` 或 `event === 'status.update'` | 状态更新事件 |

**其他情况直接透传**：
- `needsFiltering === false` 的频道（用户时间线类）
- 其他事件类型（如 `delete`、`follow` 等）

### 2.3 Block（用户屏蔽）

#### 实现位置
**streaming/index.js:747-752**

#### 代码证据

```javascript
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
```

#### 关键分析：Block 是**双向**检查

```sql
WHERE (account_id = $1 AND target_account_id IN (...))
   OR (account_id = $2 AND target_account_id = $1)
```

| 条件 | 参数绑定 | 含义 |
|-----|---------|------|
| 条件 1 | `account_id = $1` (req.accountId) | 当前用户屏蔽了目标用户 |
| 条件 2 | `account_id = $2` (payload.account.id) | 目标用户屏蔽了当前用户 |

**参数说明**：
- `$1` = `req.accountId`（当前用户 ID）
- `$2` = `payload.account.id`（帖子作者 ID）
- `targetAccountIds` = `[payload.account.id].concat(payload.mentions.map(item => item.id))`

**重要发现**：
1. **Block 检查包括被提及的用户**：`targetAccountIds` 包含作者和所有被提及的用户
2. **双向检查**：任一方向的屏蔽都会触发过滤

### 2.4 Mute（用户静音）

#### 实现位置
**streaming/index.js:753-758**

#### 代码证据

```sql
SELECT 1
FROM mutes
WHERE account_id = $1
  AND target_account_id IN (${placeholders(targetAccountIds, 2)})
```

#### 关键分析：Mute 是**单向**检查

| 条件 | 参数绑定 | 含义 |
|-----|---------|------|
| `account_id = $1` | `req.accountId` | 当前用户 |
| `target_account_id IN (...)` | `targetAccountIds` | 目标用户（作者 + 被提及者） |

**与 Block 的区别**：
- Block：检查**双向**关系
- Mute：只检查**单向**关系（当前用户是否静音了目标用户）

### 2.5 Domain Block（域屏蔽）

#### 实现位置
**streaming/index.js:762-765**

#### 代码证据

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

#### 关键分析

| 条件 | 说明 |
|-----|------|
| `account_id = $1` | 当前用户 |
| `domain = $2` | 帖子作者的域名 |

**域名提取**（streaming/index.js:738）：
```javascript
const accountDomain = payload.account.acct.split('@')[1];
```

- 本地用户：`acct === username`，`accountDomain = undefined`
- 远程用户：`acct === username@domain`，`accountDomain = domain`

### 2.6 过滤判断逻辑

#### 实现位置
**streaming/index.js:776-781**

#### 代码证据

```javascript
// Handling blocks & mutes and domain blocks: If one of those applies,
// then we don't transmit the payload of the event to the client
// @ts-expect-error
if (values[0].rows.length > 0 || (accountDomain && values[1].rows.length > 0)) {
  return;
}
```

#### 关键分析：**OR 逻辑**

| 查询结果索引 | 查询内容 | 条件 |
|-------------|---------|------|
| `values[0]` | blocks + mutes 查询 | `values[0].rows.length > 0` |
| `values[1]` | account_domain_blocks 查询（如果有 accountDomain） | `accountDomain && values[1].rows.length > 0` |

**逻辑**：
```
IF (blocks OR mutes 匹配) OR (domain block 匹配)
    不传输（return）
ELSE
    继续处理
```

### 2.7 完整过滤链（Streaming 服务层）

#### 实现位置
**streaming/index.js:714-902**

#### 完整流程

```
┌─────────────────────────────────────────────────────────────────┐
│  Streaming 服务过滤链（仅对 needsFiltering: true 的频道）         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  步骤 1：检查 Feed 访问设置                                        │
│  位置：streaming/index.js:714-718                                 │
│  ─────────────────────────────────────────────────────────────  │
│  const localPayload = payload.account.username === payload.account.acct;
│  if (localPayload ? filterLocal : filterRemote) {
│    return;  // 不传输
│  }
│                                                                  │
│  说明：                                                           │
│  - filterLocal/filterRemote 来自 getFeedAccessSettings()       │
│  - 检查 settings 表中的 'local_live_feed_access' 等设置         │
│  - 有 PERMISSION_VIEW_FEEDS 权限的用户跳过此检查                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  步骤 2：检查语言过滤                                              │
│  位置：streaming/index.js:720-726                                 │
│  ─────────────────────────────────────────────────────────────  │
│  if (Array.isArray(req.chosenLanguages) 
│      && req.chosenLanguages.indexOf(payload.language) === -1) {
│    return;  // 不传输
│  }
│                                                                  │
│  说明：                                                           │
│  - 检查帖子语言是否在用户的 chosen_languages 中                    │
│  - chosen_languages 从数据库 users 表获取                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  步骤 3：检查登录状态                                              │
│  位置：streaming/index.js:728-732                                 │
│  ─────────────────────────────────────────────────────────────  │
│  if (!req.accountId) {
│    transmit(event, payload);  // 直接传输
│    return;
│  }
│                                                                  │
│  说明：                                                           │
│  - 未登录用户不检查 blocks/mutes/domain blocks                   │
│  - 这是因为未登录用户无法设置这些关系                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  步骤 4：并行查询数据库                                            │
│  位置：streaming/index.js:740-771                                 │
│  ─────────────────────────────────────────────────────────────  │
│  查询 1：blocks + mutes（总是执行）                                │
│  ┌─────────────────────────────────────────────────────────────┐
│  │ SELECT 1 FROM blocks                                          │
│  │ WHERE (account_id = $1 AND target_account_id IN (...))      │
│  │    OR (account_id = $2 AND target_account_id = $1)          │
│  │ UNION                                                         │
│  │ SELECT 1 FROM mutes                                           │
│  │ WHERE account_id = $1                                         │
│  │   AND target_account_id IN (...)                              │
│  └─────────────────────────────────────────────────────────────┘
│                                                                  │
│  查询 2：account_domain_blocks（仅当有 accountDomain 时）        │
│  ┌─────────────────────────────────────────────────────────────┐
│  │ SELECT 1 FROM account_domain_blocks                           │
│  │ WHERE account_id = $1 AND domain = $2                        │
│  └─────────────────────────────────────────────────────────────┘
│                                                                  │
│  查询 3：custom_filters（仅当 payload 无 filtered 属性时）        │
│  ┌─────────────────────────────────────────────────────────────┐
│  │ SELECT filter.id, filter.phrase, filter.context,             │
│  │        filter.expires_at, filter.action AS filter_action,    │
│  │        keyword.keyword, keyword.whole_word                    │
│  │ FROM custom_filter_keywords keyword                           │
│  │ JOIN custom_filters filter ON keyword.custom_filter_id = filter.id
│  │ WHERE filter.account_id = $1                                  │
│  │   AND (filter.expires_at IS NULL OR filter.expires_at > NOW())
│  └─────────────────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  步骤 5：应用 blocks/mutes/domain blocks 过滤                     │
│  位置：streaming/index.js:776-781                                 │
│  ─────────────────────────────────────────────────────────────  │
│  if (values[0].rows.length > 0                                   │
│      || (accountDomain && values[1].rows.length > 0)) {
│    return;  // 不传输
│  }
│                                                                  │
│  说明：                                                           │
│  - OR 逻辑：任一条件满足即过滤                                     │
│  - values[0] = blocks/mutes 查询结果                              │
│  - values[1] = domain blocks 查询结果（如果有）                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  步骤 6：检查自定义过滤器                                          │
│  位置：streaming/index.js:783-897                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  6.1 检查是否已在 Rails 端处理                                     │
│  ─────────────────────────────────────────────────────────────  │
│  if (Object.hasOwn(payload, "filtered")) {
│    transmit(event, payload);  // 直接传输
│    return;
│  }
│                                                                  │
│  说明：如果 payload 已有 filtered 属性，Rails 端已处理              │
│                                                                  │
│  6.2 加载并应用自定义过滤器                                        │
│  ─────────────────────────────────────────────────────────────  │
│  - 从数据库加载用户的 custom_filters 和 custom_filter_keywords    │
│  - 构建正则表达式匹配关键词                                         │
│  - 检查 context 是否匹配（home, notifications, public, thread）   │
│  - 检查 filter_action：                                           │
│    • filter_action = 'warn' (0) → 添加 filtered 属性后传输        │
│    • filter_action = 'hide' (1) → 从代码看，实际上也是添加 filtered │
│                                                                  │
│  说明：                                                           │
│  - filter_action 枚举定义：streaming/index.js:810-815            │
│    // enum { warn: 0, hide: 1 }
│    filter_action: ['warn', 'hide'][filter.filter_action],
└─────────────────────────────────────────────────────────────────┘
```

### 2.8 用户屏蔽层总结

| 机制 | 方向性 | 检查时机 | 检查范围 | 代码位置 |
|-----|--------|---------|---------|---------|
| **Block** | 双向 | 登录后 | 作者 + 被提及者 | streaming/index.js:747-752 |
| **Mute** | 单向 | 登录后 | 作者 + 被提及者 | streaming/index.js:753-758 |
| **Domain Block** | 单向 | 登录后 | 作者域名 | streaming/index.js:762-765 |
| **Custom Filter** | 单向 | 登录后 | 内容关键词 | streaming/index.js:783-897 |

**过滤逻辑**：`Block OR Mute OR Domain Block` → 不传输

---

## 第三部分：内容可见性设置层

### 3.1 代码库证据状态

**当前代码库（streaming 服务）中未发现**对 `visibility` 字段的直接检查。

### 3.2 基于架构的推断

根据关键注释（streaming/index.js:695-704）：
```javascript
// This is because majority of the filtering happens on the
// Ruby on Rails side when producing the event for streaming.
```

可以推断：
- **Visibility 过滤发生在 Rails 后端**
- Streaming 服务接收的事件已经是经过 visibility 过滤的

### 3.3 可见性级别（基于 Mastodon 标准行为）

| 级别 | 枚举值 | 时间线出现 | 联邦传播 |
|-----|-------|-----------|---------|
| **Public** | `'public'` | 所有时间线 | 完全联邦 |
| **Unlisted** | `'unlisted'` | 主页时间线、个人主页 | 联邦但不进入公共时间线 |
| **Private** | `'private'` | 仅关注者主页 | 不联邦 |
| **Direct** | `'direct'` | 不进入时间线 | 不联邦 |

### 3.4 可见性层与其他层的衔接

```
┌─────────────────────────────────────────────────────────────────┐
│  可见性层（Rails 后端）                                           │
└─────────────────────────────────────────────────────────────────┘
                              │
              决定事件是否进入 Redis channel
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  事件进入 Redis channel（如 timeline:public）                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Streaming 服务订阅 Redis channel                                  │
│  - 对 needsFiltering: false 的频道：直接透传                       │
│  - 对 needsFiltering: true 的频道：额外过滤                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Streaming 服务额外过滤（仅 public/hashtag 类频道）               │
│  - blocks/mutes/domain blocks                                     │
│  - 语言过滤                                                        │
│  - 自定义过滤器                                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 第四部分：三层规则的完整衔接

### 4.1 完整处理链

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Mastodon 内容可见性处理链                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                              │
              ┌───────────────────────────────┼───────────────────────────────┐
              │                               │                               │
              ▼                               ▼                               ▼
┌───────────────────────┐   ┌───────────────────────┐   ┌───────────────────────┐
│   管理员处置层         │   │   内容可见性层         │   │   用户屏蔽层           │
│                       │   │                       │   │                       │
│  - 认证时检查         │   │  - 事件生成时过滤      │   │  - Rails 后端过滤     │
│    suspended_at       │   │    visibility         │   │    (needsFiltering:   │
│                       │   │  - 事件生成时过滤      │   │     false 的频道)     │
│  优先级：最高          │   │    silenced_at        │   │  - Streaming 服务过滤  │
└───────────────────────┘   │                       │   │    (needsFiltering:    │
              │             │  优先级：基础过滤       │   │     true 的频道)       │
              │             └───────────┬───────────┘   └───────────┬───────────┘
              │                         │                               │
              │                         ▼                               │
              │             ┌───────────────────────┐                   │
              │             │   Rails 后端          │                   │
              │             │   事件生成            │                   │
              │             │                       │                   │
              │             │  1. visibility 过滤   │                   │
              │             │  2. silenced_at 过滤  │                   │
              │             │  3. blocks/mutes 过滤 │                   │
              │             │    (部分频道)          │                   │
              │             └───────────┬───────────┘                   │
              │                         │                               │
              │                         ▼                               │
              │             ┌───────────────────────┐                   │
              │             │   事件进入 Redis      │                   │
              │             │   channel             │                   │
              │             └───────────┬───────────┘                   │
              │                         │                               │
              │                         ▼                               │
              │             ┌───────────────────────┐                   │
              └────────────▶│   Streaming 服务      │◀──────────────────┘
                            │                       │
                            │  1. 认证检查           │
                            │     (suspended_at)     │
                            │                       │
                            │  2. 频道类型判断        │
                            │     - needsFiltering?  │
                            │     - 事件类型?         │
                            │                       │
                            │  3. 额外过滤（如需要）   │
                            │     - feed 访问设置     │
                            │     - 语言过滤          │
                            │     - blocks/mutes      │
                            │     - domain blocks     │
                            │     - 自定义过滤器       │
                            └───────────┬───────────┘
                                        │
                                        ▼
                            ┌───────────────────────┐
                            │   传输给客户端         │
                            │   或过滤掉             │
                            └───────────────────────┘
```

### 4.2 不同频道的过滤差异

#### 场景 A：用户访问主页时间线（`user` 频道）

| 阶段 | 过滤位置 | 过滤内容 | 代码证据 |
|-----|---------|---------|---------|
| 1 | Rails 后端 | visibility 过滤 | 推断（架构注释） |
| 2 | Rails 后端 | silenced_at 过滤 | 推断（架构注释） |
| 3 | Rails 后端 | blocks/mutes 过滤 | 推断（架构注释） |
| 4 | Streaming 服务 | **直接透传**（needsFiltering: false） | streaming/index.js:1091-1095 |

**关键代码**（streaming/index.js:1091-1095）：
```javascript
case 'user':
  resolve({
    channelIds: channelsForUserStream(req),
    options: { needsFiltering: false },  // 不需要额外过滤
  });
  break;
```

#### 场景 B：用户访问公共时间线（`public` 频道）

| 阶段 | 过滤位置 | 过滤内容 | 代码证据 |
|-----|---------|---------|---------|
| 1 | Rails 后端 | visibility = 'public' | 推断（架构注释） |
| 2 | Rails 后端 | silenced_at IS NULL | 推断（架构注释） |
| 3 | Streaming 服务 | **额外过滤**（needsFiltering: true） | streaming/index.js:1105-1107 |

**额外过滤内容**：
- feed 访问设置（streaming/index.js:714-718）
- 语言过滤（streaming/index.js:720-726）
- blocks（双向）（streaming/index.js:747-752）
- mutes（单向）（streaming/index.js:753-758）
- domain blocks（streaming/index.js:762-765）
- 自定义过滤器（streaming/index.js:783-897）

### 4.3 优先级最终确认

基于确凿代码证据的优先级（从高到低）：

| 优先级 | 规则 | 过滤时机 | 过滤位置 | 代码证据 |
|-------|------|---------|---------|---------|
| **1** | Suspended（封禁） | 认证阶段 | Streaming 服务 | streaming/index.js:383 |
| **2** | Visibility（可见性） | 事件生成阶段 | Rails 后端 | 推断（架构注释 streaming/index.js:695-704） |
| **3** | Silenced（静默） | 事件生成阶段 | Rails 后端 | 推断（架构注释） |
| **4** | Blocks（用户屏蔽） | 事件传输阶段 | Streaming 服务（公共频道） | streaming/index.js:747-752 |
| **5** | Mutes（用户静音） | 事件传输阶段 | Streaming 服务（公共频道） | streaming/index.js:753-758 |
| **6** | Domain Blocks（域屏蔽） | 事件传输阶段 | Streaming 服务（公共频道） | streaming/index.js:762-765 |
| **7** | Custom Filters（自定义） | 事件传输阶段 | Streaming 服务（公共频道） | streaming/index.js:783-897 |

### 4.4 关键纠正

#### 纠正 1：用户时间线类频道不在 Streaming 服务过滤

**之前可能的误解**：所有频道都在 Streaming 服务过滤

**实际实现**（streaming/index.js:1091-1166）：
- `user`, `user:notification`, `direct`, `list` 频道：`needsFiltering: false`
- 这些频道的过滤**完全在 Rails 后端**进行

#### 纠正 2：公共频道的过滤是**额外**过滤

**关键注释**（streaming/index.js:695-704）：
```javascript
// Streaming only needs to apply filtering to some channels...
// This is because majority of the filtering happens on the
// Ruby on Rails side when producing the event for streaming.
```

**正确理解**：
- Streaming 服务的过滤是**在 Rails 后端过滤之后**的额外过滤
- 不是替代 Rails 后端的过滤

#### 纠正 3：Blocks 的检查范围包括被提及用户

**代码证据**（streaming/index.js:736）：
```javascript
const targetAccountIds = [payload.account.id].concat(payload.mentions.map(item => item.id));
```

**正确理解**：
- Block/Mute 检查的是 `targetAccountIds`
- `targetAccountIds` = `[帖子作者ID] + [所有被提及用户的ID]`
- 这意味着：如果帖子提及了一个你屏蔽的用户，这个帖子也会被过滤

---

## 第五部分：代码证据索引

### 5.1 核心代码位置

| 功能 | 文件位置 | 行号 | 说明 |
|-----|---------|-----|------|
| 认证时检查 suspended_at | streaming/index.js | 383 | 账号封禁检查 |
| 关键架构注释 | streaming/index.js | 695-704 | 说明大部分过滤在 Rails 端 |
| 事件类型过滤判断 | streaming/index.js | 705-709 | 只有 update/status.update 需要过滤 |
| Feed 访问设置检查 | streaming/index.js | 714-718 | filterLocal/filterRemote |
| 语言过滤 | streaming/index.js | 720-726 | chosenLanguages 检查 |
| 未登录用户跳过检查 | streaming/index.js | 728-732 | 无 accountId 直接传输 |
| 目标账号 ID 构建 | streaming/index.js | 736 | 作者 + 被提及者 |
| Blocks 双向检查 SQL | streaming/index.js | 747-752 | OR 条件的双向检查 |
| Mutes 单向检查 SQL | streaming/index.js | 753-758 | 单向检查 |
| Domain Blocks 检查 | streaming/index.js | 762-765 | 域名检查 |
| 过滤判断（OR 逻辑） | streaming/index.js | 776-781 | 任一条件满足即过滤 |
| 自定义过滤器预处理检查 | streaming/index.js | 783-789 | 检查 payload.filtered |
| filter_action 枚举映射 | streaming/index.js | 810-815 | 0=warn, 1=hide |
| needsFiltering 配置 | streaming/index.js | 1091-1166 | 各频道的过滤配置 |
| PERMISSION_VIEW_FEEDS | streaming/index.js | 23 | 权限常量定义 |
| getFeedAccessSettings | streaming/index.js | 622-650 | Feed 访问权限检查 |

### 5.2 关键 SQL 查询

#### 认证查询（streaming/index.js:383）

```sql
SELECT oauth_access_tokens.id, 
       oauth_access_tokens.resource_owner_id, 
       users.account_id, 
       users.chosen_languages, 
       oauth_access_tokens.scopes, 
       COALESCE(user_roles.permissions, 0) AS permissions 
FROM oauth_access_tokens 
INNER JOIN users ON oauth_access_tokens.resource_owner_id = users.id 
INNER JOIN accounts ON accounts.id = users.account_id 
LEFT OUTER JOIN user_roles ON user_roles.id = users.role_id 
WHERE oauth_access_tokens.token = $1 
  AND oauth_access_tokens.revoked_at IS NULL 
  AND users.disabled IS FALSE 
  AND accounts.suspended_at IS NULL 
LIMIT 1
```

#### Blocks/Mutes 查询（streaming/index.js:747-759）

```sql
SELECT 1
FROM blocks
WHERE (account_id = $1 AND target_account_id IN (...))
   OR (account_id = $2 AND target_account_id = $1)
UNION
SELECT 1
FROM mutes
WHERE account_id = $1
  AND target_account_id IN (...)
```

#### Domain Blocks 查询（streaming/index.js:762-765）

```sql
SELECT 1 FROM account_domain_blocks 
WHERE account_id = $1 AND domain = $2
```

#### Custom Filters 查询（streaming/index.js:768-771）

```sql
SELECT filter.id AS id, 
       filter.phrase AS title, 
       filter.context AS context, 
       filter.expires_at AS expires_at, 
       filter.action AS filter_action, 
       keyword.keyword AS keyword, 
       keyword.whole_word AS whole_word 
FROM custom_filter_keywords keyword 
JOIN custom_filters filter ON keyword.custom_filter_id = filter.id 
WHERE filter.account_id = $1 
  AND (filter.expires_at IS NULL OR filter.expires_at > NOW())
```

---

## 第六部分：总结

### 6.1 核心架构发现

1. **两层过滤架构**：
   - Rails 后端：负责大多数过滤（visibility、silenced_at、基础 blocks/mutes）
   - Streaming 服务：仅对公共/话题类频道进行**额外**过滤

2. **频道分类**：
   - 用户时间线类（`needsFiltering: false`）：过滤完全在 Rails 后端
   - 公共时间线类（`needsFiltering: true`）：Rails 后端 + Streaming 服务双重过滤

3. **事件类型限制**：
   - 只有 `update` 和 `status.update` 事件需要在 Streaming 服务过滤
   - 其他事件直接透传

### 6.2 关键机制发现

1. **Block 是双向检查**：
   - 检查当前用户是否屏蔽了目标用户
   - **同时**检查目标用户是否屏蔽了当前用户
   - 任一条件满足即过滤

2. **Mute 是单向检查**：
   - 只检查当前用户是否静音了目标用户

3. **检查范围包括被提及用户**：
   - `targetAccountIds` = 作者 + 所有被提及用户
   - 如果帖子提及了一个你屏蔽的用户，帖子也会被过滤

4. **OR 过滤逻辑**：
   - `Block OR Mute OR Domain Block` → 不传输

### 6.3 优先级最终确认

| 优先级 | 规则 | 过滤位置 | 确凿证据 |
|-------|------|---------|---------|
| 1 | Suspended（封禁） | Streaming 认证 | streaming/index.js:383 |
| 2 | Visibility（可见性） | Rails 后端 | 架构注释推断 |
| 3 | Silenced（静默） | Rails 后端 | 架构注释推断 |
| 4 | Blocks（用户屏蔽） | Streaming（公共频道） | streaming/index.js:747-752 |
| 5 | Mutes（用户静音） | Streaming（公共频道） | streaming/index.js:753-758 |
| 6 | Domain Blocks（域屏蔽） | Streaming（公共频道） | streaming/index.js:762-765 |
| 7 | Custom Filters（自定义） | Streaming（公共频道） | streaming/index.js:783-897 |

---

**报告生成日期**：2026-05-02

**数据来源**：
- `g:/fangzheng/solo-dogfeeding/code/17727-mastodon/streaming/index.js`（确凿代码证据）
- 基于代码注释的合理推断
