# Mastodon 全文搜索权限裁剪分析报告

## 1. 概述

Mastodon 的全文搜索采用了**多层权限控制机制**，确保搜索结果只对有权限的用户可见。权限控制发生在多个关键环节：

1. **OAuth 令牌形态识别**：区分无 token、无 token+session 登录、client credentials token、resource owner token
2. **全局未认证访问开关**：`DISALLOW_UNAUTHENTICATED_API_ACCESS` 和 `limited_federation_mode` 可在 API 入口层拦截
3. **登录态门禁**：状态全文搜索链路要求 `@account.present?`
4. **索引构建阶段**：通过 `searchable_by` 字段预计算可访问用户列表
5. **查询执行阶段**：根据 `in:` 参数选择不同索引并应用权限过滤
6. **结果过滤阶段**：对返回结果进行二次精细权限检查

## 2. 整体架构流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OAuth 令牌形态识别 (Doorkeeper + Devise)                   │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  四种身份场景:                                                        │    │
│  │  - 无 token + 未登录 session: doorkeeper_token=nil, current_user=nil│    │
│  │  - 无 token + 已登录 session: doorkeeper_token=nil, current_user=super│    │
│  │  - client credentials: resource_owner_id=nil, current_user=nil      │    │
│  │  - resource owner: resource_owner_id=用户ID, current_user=User对象   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    全局开关层 (API BaseController)                           │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  require_authenticated_user! (if disallow_unauthenticated_api_access?) │    │
│  │  - 检查 current_user 是否存在                                          │    │
│  │  - 无 token+未登录、client credentials: 401                            │    │
│  │  - 无 token+已登录、resource owner: 通过                               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         搜索请求入口 (Search Controller)                     │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  authorize_if_got_token! :read, :'read:search'                       │    │
│  │  - 有 token → 验证 scope                                            │    │
│  │  - 无 token → 放行（包括 session 登录场景）                            │    │
│  │                                                                   │    │
│  │  user_signed_in? 额外检查:                                          │    │
│  │  - 未登录时限制分页和远程解析                                         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      登录态门禁 (SearchService)                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  分类型的 *_searchable? 检查:                                        │    │
│  │  - status_searchable? = ... && @account.present?                    │    │
│  │  - account_searchable? = 仅检查搜索类型（无登录要求）                 │    │
│  │  - hashtag_searchable? = 仅检查搜索类型（无登录要求）                 │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        查询参数解析 (SearchQueryParser)                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  in: 参数解析 (SearchQueryTransformer#indexes)                        │    │
│  │  - in:public  → 只搜索 PublicStatusesIndex                           │    │
│  │  - in:library → 只搜索 StatusesIndex                                 │    │
│  │  - 无参数     → 同时搜索两个索引                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        索引构建阶段 (权限预计算)                             │
│  ┌──────────────────────┐    ┌──────────────────────────────────┐          │
│  │ PublicStatusesIndex  │    │         StatusesIndex             │          │
│  │ - 公开可见性状态      │    │ - 所有非转发状态                   │          │
│  │ - 作者可索引          │    │ - 含 searchable_by 字段           │          │
│  │ - 无 searchable_by    │    │ - 预计算可访问用户ID列表           │          │
│  └──────────────────────┘    └──────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        查询执行阶段 (ES查询过滤)                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  default_filter 权限过滤 (SearchQueryTransformer)                    │    │
│  │  - PublicStatusesIndex: 无条件可见                                   │    │
│  │  - StatusesIndex: searchable_by 包含当前用户ID                       │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        结果过滤阶段 (二次检查)                               │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  StatusFilter + StatusPolicy 精细检查                                 │    │
│  │  - 作者不可用: 过滤                                                 │    │
│  │  - 可见性级别检查: 私信/私有/公开                                    │    │
│  │  - 拉黑/静音关系检查                                                │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 3. OAuth 令牌形态与 Session 登录分析

### 3.1 四种身份场景的定义

Mastodon 结合了 Doorkeeper（OAuth 2.0）和 Devise（Session 认证），存在四种身份场景：

**授权流程配置**：`config/initializers/doorkeeper.rb:170`
```ruby
grant_flows %w(authorization_code client_credentials)
```

| 场景 | OAuth Token | Session 状态 | 身份来源 | 典型场景 |
|------|-------------|--------------|----------|----------|
| **无 token + 未登录** | 无 | 未登录 | 无 | 未认证的浏览器/脚本访问 |
| **无 token + 已登录** | 无 | 已登录 | Devise session | 浏览器中已登录的用户直接访问 API |
| **client credentials** | 有 | 无关 | Doorkeeper | 服务器到服务器调用，无用户上下文 |
| **resource owner** | 有 | 无关 | Doorkeeper + 用户授权 | 第三方应用代表用户操作 |

### 3.2 身份识别与 current_user/current_account 求值链

**核心求值逻辑**：`app/controllers/api/base_controller.rb:43-51`

```ruby
def current_resource_owner
  @current_user ||= User.find(doorkeeper_token.resource_owner_id) if doorkeeper_token
end

def current_user
  current_resource_owner || super  # 关键：|| super
rescue ActiveRecord::RecordNotFound
  nil
end
```

**current_account 求值**：`app/controllers/application_controller.rb:119-123`

```ruby
def current_account
  return @current_account if defined?(@current_account)

  @current_account = current_user&.account
end
```

**关键理解**：
- `current_user` 的求值顺序是：**先检查 OAuth token，再回退到 Devise session（`super`）**
- `super` 会调用 Devise 的 `current_user` 方法，从 session 中获取登录用户
- 这意味着：**即使没有 OAuth token，只要 session 已登录，`current_user` 也会有值**

### 3.3 四种场景的求值结果详解

#### 场景 1：无 token + 未登录 session

```
请求 → doorkeeper_token = nil
           ↓
    current_resource_owner = nil (doorkeeper_token 为 nil)
           ↓
    current_user = nil || super
           ↓
    super (Devise current_user) = nil (session 未登录)
           ↓
    current_user = nil
           ↓
    current_account = nil&.account = nil
```

| 变量 | 值 | 来源 |
|------|-----|------|
| `doorkeeper_token` | `nil` | 无 OAuth token |
| `current_resource_owner` | `nil` | `doorkeeper_token` 为 nil |
| `current_user` | `nil` | `super` 返回 nil |
| `current_account` | `nil` | `current_user` 为 nil |

#### 场景 2：无 token + 已登录 session（浏览器用户）

```
请求 → doorkeeper_token = nil
           ↓
    current_resource_owner = nil (doorkeeper_token 为 nil)
           ↓
    current_user = nil || super
           ↓
    super (Devise current_user) = User对象 (从 session cookie 读取)
           ↓
    current_user = User对象
           ↓
    current_account = User对象.account = Account对象
```

| 变量 | 值 | 来源 |
|------|-----|------|
| `doorkeeper_token` | `nil` | 无 OAuth token |
| `current_resource_owner` | `nil` | `doorkeeper_token` 为 nil |
| `current_user` | User 对象 | `super`（Devise session） |
| `current_account` | Account 对象 | `current_user.account` |

**重要发现**：
- 这是浏览器中已登录用户访问 API 的常见场景
- 用户通过表单登录后，session cookie 被设置
- 后续 API 调用通过 cookie 中的 session ID 进行认证
- 不需要 OAuth token 也能获得完整的用户身份

#### 场景 3：client credentials token

```
请求 → doorkeeper_token 存在 (client_credentials 类型)
           ↓
    current_resource_owner = User.find(nil) → nil (resource_owner_id 为 nil)
           ↓
    current_user = nil || super
           ↓
    super 可能返回 User对象 或 nil (取决于 session)
           ↓
    current_user = (取决于 session)
           ↓
    current_account = (取决于 current_user)
```

| 变量 | 值 | 来源 |
|------|-----|------|
| `doorkeeper_token` | 存在 | Client Credentials Grant |
| `current_resource_owner` | `nil` | `resource_owner_id` 为 nil |
| `current_user` | 取决于 session | `super`（Devise session） |
| `current_account` | 取决于 session | `current_user.account` |

**关键理解**：
- `client credentials` 的 `resource_owner_id` 始终为 `nil`
- 但 `current_user` 仍可能通过 `super` 从 session 获得值
- 这意味着：**client credentials + 已登录 session = 完整用户身份**
- 但纯 client credentials 调用（无 session）的 `current_user` 为 `nil`

#### 场景 4：resource owner token

```
请求 → doorkeeper_token 存在 (authorization_code 类型)
           ↓
    current_resource_owner = User.find(123) → User对象 (resource_owner_id = 123)
           ↓
    current_user = User对象 || super
           ↓
    current_user = User对象 (|| 短路，super 不执行)
           ↓
    current_account = User对象.account = Account对象
```

| 变量 | 值 | 来源 |
|------|-----|------|
| `doorkeeper_token` | 存在 | Authorization Code Grant |
| `current_resource_owner` | User 对象 | `User.find(resource_owner_id)` |
| `current_user` | User 对象 | `current_resource_owner`（短路 super） |
| `current_account` | Account 对象 | `current_user.account` |

### 3.4 四种身份场景的对比总结

| 场景 | doorkeeper_token | resource_owner_id | current_user 来源 | current_user | current_account |
|------|------------------|-------------------|-------------------|--------------|-----------------|
| **无 token + 未登录** | `nil` | N/A | 无 | `nil` | `nil` |
| **无 token + 已登录** | `nil` | N/A | Devise session (`super`) | User 对象 | Account 对象 |
| **client credentials** | 存在 | `nil` | 取决于 session | 取决于 session | 取决于 session |
| **resource owner** | 存在 | 用户 ID | Doorkeeper | User 对象 | Account 对象 |

### 3.5 四种身份场景对状态搜索结果的影响

让我们跟踪四种场景在状态搜索链路中的表现：

#### 场景 A：无 token + 未登录

```
请求 → current_user = nil
           ↓
    SearchService#call(account: nil)
           ↓
    status_searchable? = ... && @account.present?
           ↓
    @account.nil? → false
           ↓
    perform_statuses_search! 不执行
           ↓
    results[:statuses] = []
```

**结果**：状态搜索返回空数组

#### 场景 B：无 token + 已登录（浏览器用户）

```
请求 → current_user = User对象 (来自 super)
           ↓
    current_account = Account对象
           ↓
    SearchService#call(account: Account对象)
           ↓
    status_searchable? = ... && @account.present?
           ↓
    @account.present? → true
           ↓
    perform_statuses_search! 执行
           ↓
    进入完整搜索链路
           ↓
    返回搜索结果
```

**结果**：完整的状态搜索结果（与 resource owner token 相同）

#### 场景 C：client credentials token（无 session）

```
请求 → doorkeeper_token 存在
           ↓
    authorize_if_got_token! → 验证 scope（read:search）
           ↓
    scope 验证通过
           ↓
    current_resource_owner = User.find(nil) → nil
           ↓
    current_user = nil || super
           ↓
    super = nil (无 session)
           ↓
    current_user = nil
           ↓
    current_account = nil
           ↓
    SearchService#call(account: nil)
           ↓
    status_searchable? = ... && @account.present?
           ↓
    @account.nil? → false
           ↓
    perform_statuses_search! 不执行
           ↓
    results[:statuses] = []
```

**结果**：状态搜索返回空数组

#### 场景 D：resource owner token

```
请求 → doorkeeper_token 存在
           ↓
    authorize_if_got_token! → 验证 scope（read:search）
           ↓
    scope 验证通过
           ↓
    current_resource_owner = User.find(123) → User对象
           ↓
    current_user = User对象
           ↓
    current_account = Account对象
           ↓
    SearchService#call(account: Account对象)
           ↓
    status_searchable? = ... && @account.present?
           ↓
    @account.present? → true
           ↓
    perform_statuses_search! 执行
           ↓
    进入完整搜索链路
           ↓
    返回搜索结果
```

**结果**：完整的状态搜索结果

### 3.6 四种场景的最终对比

| 场景 | OAuth 认证 | Session 认证 | current_user | current_account | status_searchable? | 状态搜索结果 |
|------|-----------|-------------|--------------|-----------------|-------------------|-------------|
| 无 token + 未登录 | ❌ | ❌ | `nil` | `nil` | ❌ | `[]` |
| 无 token + 已登录 | ❌ | ✅ | User 对象 | Account 对象 | ✅ | 完整结果 |
| client credentials | ✅ | ❌ | `nil` | `nil` | ❌ | `[]` |
| resource owner | ✅ | 无关 | User 对象 | Account 对象 | ✅ | 完整结果 |

**重要结论**：
- **无 token + 已登录 session** 与 **resource owner token** 具有相同的权限
- 这是 Mastodon 前端 Web 应用的标准认证方式
- `client credentials` 即使通过了 OAuth 认证，没有 session 的话仍无法搜索状态
- 身份识别的核心是 `current_user` 是否存在，而不是 token 类型

## 4. 全局未认证访问开关

### 4.1 开关定义与触发条件

**文件位置**：`app/controllers/api/base_controller.rb:94-96`

```ruby
def disallow_unauthenticated_api_access?
  ENV['DISALLOW_UNAUTHENTICATED_API_ACCESS'] == 'true' || Rails.configuration.x.mastodon.limited_federation_mode
end
```

两个开关是 **OR** 关系，任一启用即触发强制认证：

| 开关 | 配置方式 | 说明 |
|------|----------|------|
| `DISALLOW_UNAUTHENTICATED_API_ACCESS` | 环境变量 | 显式禁止所有未认证 API 访问 |
| `limited_federation_mode` | 环境变量 `LIMITED_FEDERATION_MODE` 或 `WHITELIST_MODE` | 有限联邦模式，仅与白名单域名通信 |

**配置文件位置**：`config/mastodon.yml:4`

```yaml
limited_federation_mode: <%= (ENV.fetch('LIMITED_FEDERATION_MODE', nil) || ENV.fetch('WHITELIST_MODE', nil)) == 'true' %>
```

### 4.2 开关对搜索入口的影响

**文件位置**：`app/controllers/api/base_controller.rb:14-17`

```ruby
skip_before_action :require_functional!, unless: :limited_federation_mode?

before_action :require_authenticated_user!, if: :disallow_unauthenticated_api_access?
before_action :require_not_suspended!
```

当任一开关启用时，`before_action :require_authenticated_user!` 会被触发：

**文件位置**：`app/controllers/api/base_controller.rb:57-59`

```ruby
def require_authenticated_user!
  render json: { error: 'This method requires an authenticated user' }, status: 401 unless current_user
end
```

### 4.3 开关与四种身份场景的交互

让我们分析全局开关启用时，四种身份场景的行为：

#### 场景 A：全局开关启用 + 无 token + 未登录

```
请求 → require_authenticated_user!
           ↓
    current_user.nil?
           ↓
    返回 401: "This method requires an authenticated user"
```

**结果**：直接 401 拒绝

#### 场景 B：全局开关启用 + 无 token + 已登录

```
请求 → current_user = User对象 (来自 super)
           ↓
    require_authenticated_user!
           ↓
    current_user.present? → 通过
           ↓
    继续执行后续逻辑
```

**结果**：通过认证检查，继续执行

#### 场景 C：全局开关启用 + client credentials + 未登录

```
请求 → doorkeeper_token 存在（client credentials）
           ↓
    current_resource_owner = User.find(nil) → nil
           ↓
    current_user = nil || super
           ↓
    super = nil (无 session)
           ↓
    current_user = nil
           ↓
    require_authenticated_user!
           ↓
    current_user.nil?
           ↓
    返回 401: "This method requires an authenticated user"
```

**关键发现**：
- `client credentials token` 即使通过了 OAuth 认证
- 但 `current_user` 仍为 `nil`（无 session 时）
- 全局开关启用时，也会被 `require_authenticated_user!` 拦截

#### 场景 D：全局开关启用 + resource owner token

```
请求 → doorkeeper_token 存在（resource owner）
           ↓
    current_resource_owner = User.find(123) → User对象
           ↓
    current_user = User对象
           ↓
    require_authenticated_user!
           ↓
    current_user.present? → 通过
           ↓
    继续执行后续逻辑
```

**结果**：通过认证检查，继续执行

### 4.4 开关与四种场景的对照表

| 全局开关 | 身份场景 | current_user | 开关行为 |
|----------|----------|--------------|----------|
| **关** | 无 token + 未登录 | `nil` | 不拦截 |
| **关** | 无 token + 已登录 | User 对象 | 不拦截 |
| **关** | client credentials | 取决于 session | 不拦截 |
| **关** | resource owner | User 对象 | 不拦截 |
| **开** | 无 token + 未登录 | `nil` | ❌ 401 |
| **开** | 无 token + 已登录 | User 对象 | ✅ 通过 |
| **开** | client credentials (无 session) | `nil` | ❌ 401 |
| **开** | client credentials (有 session) | User 对象 | ✅ 通过 |
| **开** | resource owner | User 对象 | ✅ 通过 |

**关键理解**：
- 全局开关检查的是 `current_user` 是否存在
- 与 token 类型无关，只看最终的 `current_user` 值
- `无 token + 已登录 session` 和 `resource owner token` 都能通过
- `client credentials` 只有在有 session 登录时才能通过

## 5. 入口权限矩阵

### 5.1 搜索类型的 *_searchable? 方法

**文件位置**：`app/services/search_service.rb:86-96`

```ruby
def status_searchable?
  Chewy.enabled? && status_search? && @account.present?
end

def account_searchable?
  account_search?
end

def hashtag_searchable?
  hashtag_search?
end
```

**关键区别**：
- `status_searchable?`：额外检查 `@account.present?` → **需要登录**
- `account_searchable?`：只检查搜索类型 → **无需登录**
- `hashtag_searchable?`：只检查搜索类型 → **无需登录**

### 5.2 完整入口权限矩阵（四种身份场景）

矩阵维度：
- **全局开关**：关（默认）/ 开（DISALLOW_UNAUTHENTICATED_API_ACCESS 或 limited_federation_mode）
- **身份场景**：无 token+未登录 / 无 token+已登录 / client credentials / resource owner
- **搜索类型**：statuses / accounts / hashtags

#### 矩阵 1：全局开关关闭（默认配置）

| 身份场景 | current_user | current_account | statuses | accounts | hashtags | 说明 |
|----------|--------------|-----------------|----------|----------|----------|------|
| **无 token + 未登录** | `nil` | `nil` | ❌ 空数组 | ✅ 可搜索 | ✅ 可搜索 | 状态搜索被 `@account.present?` 拦截 |
| **无 token + 已登录** | User 对象 | Account 对象 | ✅ 完整结果 | ✅ 可搜索 | ✅ 可搜索 | 浏览器已登录用户，通过 session 认证 |
| **client credentials (无 session)** | `nil` | `nil` | ❌ 空数组 | ✅ 可搜索 | ✅ 可搜索 | scope 验证通过，但 `current_user` 为 `nil` |
| **client credentials (有 session)** | User 对象 | Account 对象 | ✅ 完整结果 | ✅ 可搜索 | ✅ 可搜索 | 不常见场景，token + session 双重认证 |
| **resource owner** | User 对象 | Account 对象 | ✅ 完整结果 | ✅ 可搜索 | ✅ 可搜索 | OAuth 用户授权，完整用户上下文 |

#### 矩阵 2：全局开关开启

| 身份场景 | current_user | statuses | accounts | hashtags | 说明 |
|----------|--------------|----------|----------|----------|------|
| **无 token + 未登录** | `nil` | ❌ 401 | ❌ 401 | ❌ 401 | 被 `require_authenticated_user!` 拦截 |
| **无 token + 已登录** | User 对象 | ✅ 完整结果 | ✅ 可搜索 | ✅ 可搜索 | session 认证通过 |
| **client credentials (无 session)** | `nil` | ❌ 401 | ❌ 401 | ❌ 401 | `current_user` 为 `nil`，被拦截 |
| **client credentials (有 session)** | User 对象 | ✅ 完整结果 | ✅ 可搜索 | ✅ 可搜索 | session 认证通过 |
| **resource owner** | User 对象 | ✅ 完整结果 | ✅ 可搜索 | ✅ 可搜索 | OAuth 认证通过 |

### 5.3 矩阵解读

#### 关键差异：无 token + 未登录 vs 无 token + 已登录

这是最容易混淆的两种场景：

| 维度 | 无 token + 未登录 | 无 token + 已登录 |
|------|------------------|------------------|
| OAuth token | 无 | 无 |
| Session cookie | 无/无效 | 有效 |
| `current_user` | `nil` | User 对象 |
| `current_account` | `nil` | Account 对象 |
| `user_signed_in?` | `false` | `true` |
| 全局开关（开） | ❌ 401 | ✅ 通过 |
| `status_searchable?` | ❌ | ✅ |
| 状态搜索结果 | 空数组 | 完整结果 |

**典型场景**：
- **无 token + 未登录**：匿名脚本访问 API，或未登录浏览器
- **无 token + 已登录**：用户在浏览器中登录后，前端 JavaScript 调用 API

#### 全局开关关闭时的行为（默认）

- **无 token + 未登录**：API 允许访问，但状态搜索被 `@account.present?` 拦截，返回空数组；账户和标签搜索可用
- **无 token + 已登录**：完整权限，所有搜索类型可用
- **client credentials (无 session)**：OAuth 认证通过，但状态搜索仍为空（`current_user` 为 `nil`）
- **resource owner**：完整权限，所有搜索类型可用

#### 全局开关开启时的行为

- **无 token + 未登录**：被 `require_authenticated_user!` 拦截，返回 401
- **无 token + 已登录**：通过 `current_user` 检查，完整权限
- **client credentials (无 session)**：被拦截，返回 401（`current_user` 为 `nil`）
- **resource owner**：通过检查，完整权限

### 5.4 user_signed_in? 的额外限制

**文件位置**：`app/controllers/api/v2/search_controller.rb:12-15`

```ruby
with_options unless: :user_signed_in? do
  before_action :query_pagination_error, if: :pagination_requested?
  before_action :remote_resolve_error, if: :remote_resolve_requested?
end
```

即使 `current_user` 存在（通过 token 或 session），`user_signed_in?` 还会对未登录用户施加额外限制：

| 限制项 | 未登录用户 | 已登录用户 |
|--------|-----------|-----------|
| 分页 (`offset` 参数) | ❌ 401 错误 | ✅ 可用 |
| 远程解析 (`resolve` 参数) | ❌ 401 错误 | ✅ 可用 |

**关键理解**：
- `user_signed_in?` 是 Devise 提供的方法，检查的是 session 登录状态
- 即使 `current_user` 存在（通过 resource owner token），`user_signed_in?` 也可能返回 `false`
- 这意味着：**resource owner token 认证的用户可能无法使用分页和远程解析**
- 只有 **无 token + 已登录 session** 的用户才能完全绕过这些限制

### 5.5 四种身份场景的完整权限对比

| 场景 | current_user | user_signed_in? | 全局开关(开) | status_searchable? | 分页 | 远程解析 |
|------|--------------|-----------------|-------------|-------------------|------|----------|
| 无 token + 未登录 | `nil` | `false` | ❌ 401 | ❌ | ❌ | ❌ |
| 无 token + 已登录 | User 对象 | `true` | ✅ | ✅ | ✅ | ✅ |
| client credentials (无 session) | `nil` | `false` | ❌ 401 | ❌ | ❌ | ❌ |
| client credentials (有 session) | User 对象 | `true` | ✅ | ✅ | ✅ | ✅ |
| resource owner | User 对象 | `false` | ✅ | ✅ | ❌ | ❌ |

**发现**：
- **无 token + 已登录 session** 是权限最完整的场景（浏览器标准用户）
- **resource owner token** 虽然能搜索状态，但无法使用分页和远程解析
- 这是 Mastodon 前端（Web UI）和 API 客户端的权限差异

## 6. 登录态门禁：状态搜索链路的登录要求

### 6.1 控制器层的 token 检查

**文件位置**：`app/controllers/api/v2/search_controller.rb:9-15`

```ruby
before_action -> { authorize_if_got_token! :read, :'read:search' }
before_action :validate_search_params!

with_options unless: :user_signed_in? do
  before_action :query_pagination_error, if: :pagination_requested?
  before_action :remote_resolve_error, if: :remote_resolve_requested?
end
```

**文件位置**：`app/controllers/api/base_controller.rb:90-92`

```ruby
def authorize_if_got_token!(*scopes)
  doorkeeper_authorize!(*scopes) if doorkeeper_token
end
```

**关键行为**：
- `authorize_if_got_token!` 是"有 token 才检查"的逻辑
- 如果请求不带 token，此方法直接放行（不报错）
- 如果带了 client credentials token，scope 验证通过但 `current_user` 可能仍为 `nil`
- 无 token 但有 session 的用户可以完全通过此检查

### 6.2 SearchService 层的关键门禁

**文件位置**：`app/services/search_service.rb:86-96`

```ruby
def status_searchable?
  Chewy.enabled? && status_search? && @account.present?
end

def account_searchable?
  account_search?
end

def hashtag_searchable?
  hashtag_search?
end
```

这是**状态全文搜索链路的关键门禁**。让我们查看调用逻辑：

**文件位置**：`app/services/search_service.rb:16-27`

```ruby
default_results.tap do |results|
  next if @query.blank? || @limit.zero?

  if url_query?
    results.merge!(url_resource_results) unless url_resource.nil? || @offset.positive? || (@options[:type].present? && url_resource_symbol != @options[:type].to_sym)
  elsif @query.present?
    results[:accounts] = perform_accounts_search! if account_searchable?
    results[:statuses] = perform_statuses_search! if status_searchable?  # 关键条件
    results[:hashtags] = perform_hashtags_search! if hashtag_searchable?
  end
end
```

### 6.3 四种身份场景的登录态门禁检查

`status_searchable?` 方法的三个条件：

| 条件 | 说明 | 无 token+未登录 | 无 token+已登录 | client credentials (无 session) | resource owner |
|------|------|----------------|----------------|-------------------------------|----------------|
| `Chewy.enabled?` | ES 是否启用 | 可能为 true | 可能为 true | 可能为 true | 可能为 true |
| `status_search?` | 搜索类型 | 可能为 true | 可能为 true | 可能为 true | 可能为 true |
| `@account.present?` | 登录账户 | **false** | **true** | **false** | **true** |

**关键结论**：
- `无 token + 未登录` 和 `client credentials (无 session)`：`@account.nil?` → 跳过状态搜索
- `无 token + 已登录` 和 `resource owner`：`@account.present?` → 执行状态搜索

### 6.4 门禁的设计意图

这个门禁设计有以下考虑：

1. **隐私保护**：状态搜索可能包含用户的私人互动（如点赞、收藏的状态）
2. **避免滥用**：未登录用户无法大规模搜索状态内容
3. **性能优化**：减少无效的 Elasticsearch 查询
4. **与索引设计匹配**：`StatusesIndex` 的 `searchable_by` 字段需要用户 ID 进行过滤
5. **身份明确性**：只有具有 `current_account` 的身份才能进行个性化搜索

## 7. 索引更新阶段（权限预计算）

### 7.1 触发机制

当内容（Status、Account、Tag）被创建或更新时，Chewy gem 的 Mastodon 策略会捕获这些变更：

**文件位置**：`lib/chewy/strategy/mastodon.rb:12-27`

```ruby
def update(type, objects, _options = {})
  @stash[type].concat(type.root.id ? Array.wrap(objects) : type.adapter.identify(objects)) if Chewy.enabled?
end

def leave
  RedisConnection.with do |redis|
    redis.pipelined do |pipeline|
      @stash.each do |type, ids|
        ids = ids&.compact
        next if ids.blank?

        pipeline.sadd("chewy:queue:#{type.name}", ids)
      end
    end
  end
end
```

- 当模型发生变更时，`update` 方法被调用
- 变更记录被添加到 Redis 队列 `chewy:queue:{index_name}`
- 由 `Scheduler::IndexingScheduler` 定期处理队列并更新 Elasticsearch 索引

### 7.2 双索引设计

Mastodon 使用两个独立的状态索引，这是理解 `in:public` 和 `in:library` 差异的基础：

#### PublicStatusesIndex（公开状态索引）

**文件位置**：`app/chewy/public_statuses_index.rb:55-68`

```ruby
index_scope ::Status.unscoped
  .kept
  .indexable  # 关键：public_visibility + 作者 indexable: true
  .includes(:media_attachments, :preloadable_poll, :tags, preview_cards_status: :preview_card)

root date_detection: false do
  field(:id, type: 'long')
  field(:account_id, type: 'long')
  field(:text, type: 'text', analyzer: 'verbatim', value: ->(status) { status.searchable_text }) { field(:stemmed, type: 'text', analyzer: 'content') }
  field(:tags, type: 'text', analyzer: 'hashtag', value: ->(status) { status.tags.map(&:display_name) })
  field(:language, type: 'keyword')
  field(:properties, type: 'keyword', value: ->(status) { status.searchable_properties })
  field(:created_at, type: 'date', value: ->(status) { clamp_date(status.created_at) })
end
```

**PublicStatusesIndex 特点**：
- 仅索引**公开可见性**（`public_visibility`）的状态
- 要求作者账户设置为可索引（`indexable: true`）
- **不包含 `searchable_by` 字段**
- 设计目标：存储可被公开搜索的内容

让我们验证 `indexable` scope 的定义：

**文件位置**：`app/models/concerns/status/search_concern.rb:6-8`

```ruby
included do
  scope :indexable, -> { without_reblogs.public_visibility.joins(:account).where(account: { indexable: true }) }
end
```

#### StatusesIndex（私有状态索引）

**文件位置**：`app/chewy/statuses_index.rb:55-66`

```ruby
index_scope ::Status.unscoped.kept.without_reblogs.includes(:media_attachments, :local_mentioned, :local_favorited, :local_reblogged, :local_bookmarked, :tags, preview_cards_status: :preview_card, preloadable_poll: :local_voters), 
  delete_if: ->(status) { status.searchable_by.empty? }

root date_detection: false do
  field(:id, type: 'long')
  field(:account_id, type: 'long')
  field(:text, type: 'text', analyzer: 'verbatim', value: ->(status) { status.searchable_text }) { field(:stemmed, type: 'text', analyzer: 'content') }
  field(:tags, type: 'text', analyzer: 'hashtag',  value: ->(status) { status.tags.map(&:display_name) })
  field(:searchable_by, type: 'long', value: ->(status) { status.searchable_by })  # 关键字段
  field(:language, type: 'keyword')
  field(:properties, type: 'keyword', value: ->(status) { status.searchable_properties })
  field(:created_at, type: 'date', value: ->(status) { clamp_date(status.created_at) })
end
```

**StatusesIndex 特点**：
- 索引所有**非转发**的状态（包括私有可见性）
- 如果 `searchable_by` 为空，则不索引该状态
- **包含 `searchable_by` 字段**，存储有权限访问该状态的用户 ID 列表
- 设计目标：用户可以搜索自己互动过的非公开状态

### 7.3 searchable_by 字段计算

**文件位置**：`app/models/concerns/status/search_concern.rb:10-24`

```ruby
def searchable_by
  @searchable_by ||= begin
    ids = []

    ids << account_id if local?           # 状态作者（仅限本地账户）

    ids += local_mentioned.pluck(:id)     # 被@提及的本地用户
    ids += local_favorited.pluck(:id)     # 点赞的本地用户
    ids += local_reblogged.pluck(:id)     # 转发的本地用户
    ids += local_bookmarked.pluck(:id)    # 收藏的本地用户
    ids += preloadable_poll.local_voters.pluck(:id) if preloadable_poll.present?  # 投票的本地用户

    ids.uniq
  end
end
```

**权限裁剪环节 1（索引构建阶段）**：
- 预计算哪些用户有权限搜索到该状态
- 基于历史互动（点赞、转发、收藏、投票、提及）
- 仅限本地用户，避免索引膨胀
- 如果没有任何本地用户互动过，`searchable_by` 为空，状态不会被索引到 `StatusesIndex`

## 8. in:public 与 in:library 查询参数分析

### 8.1 索引选择逻辑

**文件位置**：`app/lib/search_query_transformer.rb:57-66`

```ruby
def indexes
  case @flags['in']
  when 'library'
    [StatusesIndex]
  when 'public'
    [PublicStatusesIndex]
  else
    [PublicStatusesIndex, StatusesIndex]
  end
end
```

`in:` 参数是通过 `PrefixClause` 解析的：

**文件位置**：`app/lib/search_query_transformer.rb:146-188`

```ruby
class PrefixClause
  def initialize(prefix, operator, term, options = {})
    @prefix = prefix
    @negated = operator == '-'
    @options = options
    @operator = :filter

    case prefix
    # ... 其他前缀
    when 'in'
      @operator = :flag  # 特殊处理：标记为 flag 而非 filter
      @term = term
    # ...
    end
  end
end
```

然后在 `Query` 类中提取 flags：

**文件位置**：`app/lib/search_query_transformer.rb:41-43`

```ruby
def flags_from_clauses!
  @flags = clauses_by_operator.fetch(:flag, []).to_h { |clause| [clause.prefix, clause.term] }
end
```

### 8.2 in:public 与 in:library 的详细对比

| 特性 | in:public | in:library | 默认（无参数） |
|------|-----------|------------|----------------|
| 搜索索引 | PublicStatusesIndex | StatusesIndex | 两个索引 |
| 内容范围 | 公开可见性状态 | 用户互动过的状态 | 全部 |
| 索引条件 | public_visibility + 作者 indexable | 非转发 + searchable_by 非空 | - |
| 权限过滤方式 | 无（索引时已筛选） | searchable_by 匹配 | 组合过滤 |
| 典型用例 | 搜索公开推文 | 搜索自己点赞/收藏过的内容 | 综合搜索 |

### 8.3 in:public 场景的两层权限概念澄清

**重要澄清**：`in:public` 场景下存在两个独立的权限概念，需要分开理解：

#### 概念 A：索引层的"公开可见性"

这是指**被索引的内容本身的属性**：

- **位置**：索引构建阶段（`PublicStatusesIndex` 的 `index_scope`）
- **逻辑**：`public_visibility` + 作者 `indexable: true`
- **含义**：这个状态**内容本身**是公开的，可以被任何用户看到

```ruby
# app/models/concerns/status/search_concern.rb:6-8
scope :indexable, -> { 
  without_reblogs
    .public_visibility           # 状态可见性为 public
    .joins(:account)
    .where(account: { indexable: true })  # 作者允许被索引
}
```

#### 概念 B：状态搜索链路的"登录要求"

这是指**能否进入搜索链路**的门禁：

- **位置**：`SearchService#status_searchable?`
- **逻辑**：`@account.present?`
- **含义**：即使搜索的是公开内容，**用户也必须登录才能使用状态搜索功能**

```ruby
# app/services/search_service.rb:86-88
def status_searchable?
  Chewy.enabled? && status_search? && @account.present?  # 必须有登录账户
end
```

#### 两层概念与四种身份场景的关系

| 身份场景 | 索引层公开可见性 (A) | 搜索链路登录要求 (B) | 最终结果 |
|----------|---------------------|---------------------|----------|
| 无 token + 未登录 | 始终可用（索引定义） | ❌ `@account.present? = false` | 空数组 |
| 无 token + 已登录 | 始终可用（索引定义） | ✅ `@account.present? = true` | 完整结果 |
| client credentials (无 session) | 始终可用（索引定义） | ❌ `@account.present? = false` | 空数组 |
| resource owner | 始终可用（索引定义） | ✅ `@account.present? = true` | 完整结果 |

**关键理解**：
- `in:public` 只影响**搜索哪些索引**（概念 A）
- 登录态门禁影响**能否进入搜索链路**（概念 B）
- 这是两个**独立**的权限控制，互不影响
- 即使使用 `in:public` 搜索"公开索引"，用户仍需登录才能通过 `@account.present?` 门禁
- `无 token + 已登录 session` 和 `resource owner token` 都能通过门禁

#### in:public 场景的完整流程

```
用户发送 in:public 搜索请求
           ↓
    身份场景判断
      ├── 无 token + 未登录 → @account = nil → status_searchable? = false → 结果空
      ├── client credentials (无 session) → @account = nil → status_searchable? = false → 结果空
      ├── 无 token + 已登录 → @account 有值 → 继续
      └── resource owner → @account 有值 → 继续
                                ↓
                          全局开关检查（第 0 层）
                                ↓
                          authorize_if_got_token!（第 1 层）
                                ↓
                          解析 in:public
                                ↓
                          indexes = [PublicStatusesIndex]
                                ↓
                          ES 查询公开索引
                                ↓
                          StatusFilter 二次检查
                                ↓
                          返回公开状态结果
```

### 8.4 不同参数下的权限裁剪环节

#### 场景 1：in:public 查询

**索引选择**：`[PublicStatusesIndex]`

**default_filter 行为**：

**文件位置**：`app/lib/search_query_transformer.rb:159-189`

```ruby
def default_filter
  {
    bool: {
      should: [
        {
          term: {
            _index: PublicStatusesIndex.index_name,  # 命中这个分支
          },
        },
        {
          bool: {
            must: [
              { term: { _index: StatusesIndex.index_name } },
              { term: { searchable_by: @options[:current_account].id } },
            ],
          },
        },
      ],
      minimum_should_match: 1,
    },
  }
end
```

**in:public 权限裁剪环节**：

| 环节 | 是否生效 | 说明 |
|------|----------|------|
| 0. 身份场景识别 | 生效 | 只有提供 `@account` 的场景能通过门禁 |
| 1. 全局开关 | 可能生效 | 开关启用时需要 `current_user` 存在 |
| 2. 登录态门禁 | **生效** | 仍需 `@account.present?` 才能进入状态搜索链路 |
| 3. 索引构建阶段 | 已预筛选 | PublicStatusesIndex 只包含公开可见性状态 |
| 4. 查询执行阶段 | 简化 | 只命中 `_index == PublicStatusesIndex` 分支，无额外权限过滤 |
| 5. 结果过滤阶段 | **完整生效** | StatusFilter + StatusPolicy 仍会检查：<br>- 作者是否拉黑当前用户<br>- 作者是否拉黑当前用户域名<br>- 当前用户是否拉黑/静音作者 |

**注意**：即使使用 `in:public`，结果过滤阶段仍然会进行完整的权限检查，确保：
- 被作者拉黑的用户看不到该作者的公开状态
- 用户自己拉黑的作者不会出现在搜索结果中

#### 场景 2：in:library 查询

**索引选择**：`[StatusesIndex]`

**default_filter 行为**：
- 必须同时满足两个条件：
  1. `_index == StatusesIndex.index_name`
  2. `searchable_by` 包含当前用户 ID

**in:library 权限裁剪环节**：

| 环节 | 是否生效 | 说明 |
|------|----------|------|
| 0. 身份场景识别 | 生效 | 只有提供 `@account` 的场景能通过门禁 |
| 1. 全局开关 | 可能生效 | 开关启用时需要 `current_user` 存在 |
| 2. 登录态门禁 | 生效 | 必须登录才能使用 |
| 3. 索引构建阶段 | 生效 | `searchable_by` 预计算可访问用户列表 |
| 4. 查询执行阶段 | **关键过滤** | ES 查询时要求 `searchable_by` 包含当前用户 ID |
| 5. 结果过滤阶段 | **完整生效** | 额外检查动态关系（拉黑、静音等） |

**in:library 的核心逻辑**：
- 索引阶段：状态 A 被用户 B 点赞 → `searchable_by` 包含 B 的 ID
- 查询阶段：用户 B 搜索 → 只有 `searchable_by` 包含 B 的 ID 的状态被返回
- 结果阶段：再检查是否存在拉黑等动态关系

#### 场景 3：默认查询（无 in: 参数）

**索引选择**：`[PublicStatusesIndex, StatusesIndex]`

**default_filter 行为**：
- `should` 条件满足任一即可：
  - 来自 PublicStatusesIndex（无条件）
  - 来自 StatusesIndex 且 `searchable_by` 包含当前用户 ID

**默认查询的权限裁剪环节**：
- 结合了 `in:public` 和 `in:library` 的所有环节
- 返回结果是两个索引的并集

## 9. 查询执行阶段（搜索请求处理）

### 9.1 搜索服务入口

**文件位置**：`app/services/search_service.rb:6-27`

```ruby
def call(query, account, limit, options = {})
  @query     = query&.strip&.gsub(QUOTE_EQUIVALENT_CHARACTERS, '"')
  @account   = account
  @options   = options
  @limit     = limit.to_i
  @offset    = options[:type].blank? ? 0 : options[:offset].to_i
  @resolve   = options[:resolve] || false
  @following = options[:following] || false

  default_results.tap do |results|
    next if @query.blank? || @limit.zero?

    if url_query?
      results.merge!(url_resource_results) unless url_resource.nil? || @offset.positive? || (@options[:type].present? && url_resource_symbol != @options[:type].to_sym)
    elsif @query.present?
      results[:accounts] = perform_accounts_search! if account_searchable?
      results[:statuses] = perform_statuses_search! if status_searchable?
      results[:hashtags] = perform_hashtags_search! if hashtag_searchable?
    end
  end
end
```

### 9.2 状态搜索服务

**文件位置**：`app/services/statuses_search_service.rb:27-38`

```ruby
def status_search_results
  request             = parsed_query.request
  results             = request.collapse(field: :id).order(id: { order: :desc }).limit(@limit).offset(@offset).objects.compact
  account_ids         = results.map(&:account_id)
  account_domains     = results.map(&:account_domain)

  @account.preload_relations!(account_ids, account_domains)

  results.reject { |status| StatusFilter.new(status, @account).filtered? }
rescue Faraday::ConnectionFailed, Parslet::ParseFailed, Errno::ENETUNREACH
  []
end
```

### 9.3 查询构建与权限过滤

**文件位置**：`app/lib/search_query_transformer.rb:25-98`

```ruby
def request
  search = Chewy::Search::Request.new(*indexes).filter(default_filter)

  must_clauses.each { |clause| search = search.query.must(clause.to_query) }
  must_not_clauses.each { |clause| search = search.query.must_not(clause.to_query) }
  filter_clauses.each { |clause| search = search.filter(**clause.to_query) }

  search
end
```

**权限裁剪环节 2（查询执行阶段）**：
- 根据 `in:` 参数选择索引
- 应用 `default_filter` 进行权限过滤
- 对于 `StatusesIndex`，要求 `searchable_by` 包含当前用户 ID

## 10. 结果过滤阶段（二次过滤）

### 10.1 StatusFilter 过滤器

**文件位置**：`app/services/statuses_search_service.rb:35`

```ruby
results.reject { |status| StatusFilter.new(status, @account).filtered? }
```

**权限裁剪环节 3（结果过滤阶段）**：对 Elasticsearch 返回的结果进行二次精细过滤。

### 10.2 StatusFilter 详细实现

**文件位置**：`app/lib/status_filter.rb:11-71`

```ruby
def filtered?
  return false if !account.nil? && account.id == status.account_id  # 作者自己始终可见

  blocked_by_policy? || (account_present? && filtered_status?) || silenced_account?
end

def blocked_by_policy?
  !policy_allows_show?
end

def policy_allows_show?
  StatusPolicy.new(account, status).show?
end

def filtered_status?
  blocking_account? || blocking_domain? || muting_account?
end
```

### 10.3 StatusPolicy 权限检查

**文件位置**：`app/policies/status_policy.rb:4-14`

```ruby
def show?
  return false if author.unavailable?

  if requires_mention?  # 私信或限定可见
    owned? || mention_exists?
  elsif private?        # 仅关注者可见
    owned? || following_author? || mention_exists?
  else                  # 公开状态
    current_account.nil? || (!author_blocking? && !author_blocking_domain?)
  end
end
```

**二次过滤检查的内容**：
1. **作者不可用**：作者账户被暂停或删除则不可见
2. **私信/限定可见**：必须是作者或被@提及
3. **仅关注者可见**：必须是作者、关注者或被@提及
4. **公开状态**：不能被作者拉黑或域名拉黑
5. **用户过滤设置**：检查是否被当前用户拉黑、域名拉黑或静音

**为什么需要二次过滤**：
- `searchable_by` 是静态预计算的，无法反映实时的拉黑/静音关系
- 索引更新有延迟，二次过滤可以弥补
- 提供额外的安全保障，即使索引权限计算有误也不会泄露隐私

## 11. 权限裁剪环节完整总结

### 11.1 整体环节表（含四种身份场景）

| 环节 | 阶段 | 实现位置 | 检查内容 | 无 token+未登录 | 无 token+已登录 | client credentials (无 session) | resource owner |
|------|------|----------|----------|----------------|----------------|-------------------------------|----------------|
| 0 | 身份识别 | Doorkeeper + Devise | `current_user` 求值链 | `nil` | User 对象 | `nil` | User 对象 |
| 1 | 全局开关 | `Api::BaseController` | `disallow_unauthenticated_api_access?` | 可能 401 | 通过 | 可能 401 | 通过 |
| 2 | 登录态门禁 | `SearchService#status_searchable?` | `@account.present?` | ❌ | ✅ | ❌ | ✅ |
| 3 | 索引构建 | `Status#searchable_by` | 预计算可访问用户列表 | N/A | ✅ 生效 | N/A | ✅ 生效 |
| 4 | 查询执行 | `SearchQueryTransformer#indexes` | 根据 `in:` 参数选择索引 | N/A | 根据参数 | N/A | 根据参数 |
| 5 | 查询执行 | `SearchQueryTransformer#default_filter` | ES查询时的权限过滤 | N/A | ✅ 生效 | N/A | ✅ 生效 |
| 6 | 结果过滤 | `StatusFilter` + `StatusPolicy` | 二次检查：可见性、拉黑/静音 | N/A | ✅ 完整检查 | N/A | ✅ 完整检查 |

### 11.2 不同场景的权限裁剪流程

#### 场景 A：默认配置 + 无 token + 未登录

```
用户请求 → doorkeeper_token = nil
                    ↓
           current_user = nil || super = nil
                    ↓
           current_account = nil
                    ↓
           SearchService#call(account: nil)
                    ↓
           status_searchable? = ... && @account.present?
                    ↓
           @account.nil? → false
                    ↓
           perform_statuses_search! 不执行
                    ↓
           results[:statuses] = []
           results[:accounts] = 可能有结果
           results[:hashtags] = 可能有结果
```

**结果**：API 允许访问，但状态搜索结果为空

#### 场景 B：默认配置 + 无 token + 已登录（浏览器用户）

```
用户请求 → doorkeeper_token = nil
                    ↓
           current_user = nil || super = User对象 (session)
                    ↓
           current_account = Account对象
                    ↓
           SearchService#call(account: Account对象)
                    ↓
           status_searchable? = ... && @account.present?
                    ↓
           @account.present? → true
                    ↓
           perform_statuses_search! 执行
                    ↓
           进入完整搜索链路
                    ↓
           返回搜索结果
```

**结果**：完整的状态搜索结果

#### 场景 C：全局开关启用 + 无 token + 未登录

```
用户请求 → require_authenticated_user!
                    ↓
           current_user.nil?
                    ↓
           返回 401: "This method requires an authenticated user"
```

**结果**：API 入口直接拒绝，返回 401

#### 场景 D：全局开关启用 + client credentials + 无 session

```
用户请求 → doorkeeper_token 存在
                    ↓
           current_resource_owner = User.find(nil) → nil
                    ↓
           current_user = nil || super = nil
                    ↓
           require_authenticated_user!
                    ↓
           current_user.nil?
                    ↓
           返回 401: "This method requires an authenticated user"
```

**结果**：API 入口直接拒绝，返回 401

#### 场景 E：resource owner token + in:public

```
用户请求 → doorkeeper_token 存在（resource owner）
                    ↓
           current_resource_owner = User.find(123) → User对象
                    ↓
           current_user = User对象
                    ↓
           current_account = Account对象
                    ↓
           @account.present? == true
                    ↓
           解析 in:public → indexes = [PublicStatusesIndex]
                    ↓
           default_filter: _index 匹配即可
                    ↓
           ES 返回公开状态
                    ↓
           StatusFilter 二次检查（拉黑/静音）
                    ↓
           返回过滤后的公开状态
```

#### 场景 F：resource owner token + in:library

```
用户请求 → 全局开关检查通过
                    ↓
           @account.present? == true
                    ↓
           解析 in:library → indexes = [StatusesIndex]
                    ↓
           default_filter: searchable_by 包含当前用户ID
                    ↓
           ES 返回用户互动过的状态
                    ↓
           StatusFilter 二次检查
                    ↓
           返回最终结果
```

## 12. 统一结论

### 12.1 权限控制的层次结构

Mastodon 的状态全文搜索权限控制是一个**多层递进**的体系，各层相互独立但协同工作：

```
┌─────────────────────────────────────────────────────────────────────────┐
│  第 0 层：身份场景识别（OAuth + Session）                                 │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  四种场景:                                                          │  │
│  │  - 无 token + 未登录: current_user = nil                          │  │
│  │  - 无 token + 已登录: current_user = User对象 (super)              │  │
│  │  - client credentials (无 session): current_user = nil            │  │
│  │  - resource owner: current_user = User对象 (Doorkeeper)           │  │
│  │                                                                   │  │
│  │  关键: current_user = current_resource_owner || super             │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  第 1 层：全局未认证访问开关                                              │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  触发条件：DISALLOW_UNAUTHENTICATED_API_ACCESS 或 limited_federation_mode │  │
│  │  检查逻辑：require_authenticated_user! → current_user 是否存在    │  │
│  │  效果：无 token+未登录、client credentials (无 session) 返回 401   │  │
│  │  范围：整个 API，包括搜索、时间线等所有端点                        │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  第 2 层：分类型搜索门禁                                                 │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  status_searchable? = ... && @account.present?                   │  │
│  │  account_searchable? = 仅检查搜索类型                             │  │
│  │  hashtag_searchable? = 仅检查搜索类型                             │  │
│  │                                                                   │  │
│  │  关键区别：只有状态搜索需要 @account.present?                     │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  第 3 层：in: 参数索引选择                                               │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  触发条件：查询字符串中的 in:public 或 in:library                  │  │
│  │  效果：决定搜索 PublicStatusesIndex 还是 StatusesIndex             │  │
│  │  范围：搜索范围（公开内容 vs 互动内容）                             │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  第 4 层：ES 查询过滤                                                    │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  触发条件：SearchQueryTransformer#default_filter                   │  │
│  │  效果：StatusesIndex 需要 searchable_by 匹配当前用户               │  │
│  │  范围：索引级别的权限过滤                                          │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  第 5 层：结果二次过滤                                                   │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  触发条件：StatusFilter + StatusPolicy                             │  │
│  │  效果：检查可见性级别、拉黑/静音等动态关系                          │  │
│  │  范围：最终结果的精细检查                                          │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 12.2 关键澄清：避免常见误解

#### 误解 1："无 token 就一定是未登录"

**事实**：
- `current_user = current_resource_owner || super`
- `super` 会从 Devise session 中获取登录用户
- **无 token + 已登录 session** 也是一种合法的已认证状态
- 这是 Mastodon 前端 Web 应用的标准认证方式

#### 误解 2："client credentials token 可以搜索状态"

**事实**：
- `client credentials token` 通过 OAuth 认证和 scope 验证
- 但 `resource_owner_id` 为 `nil`，导致 `current_resource_owner` 为 `nil`
- 如果没有 session 登录，`super` 也返回 `nil`
- `status_searchable?` 中的 `@account.present?` 检查失败
- **状态搜索结果为空数组**

#### 误解 3："全局开关只拦截无 token 的请求"

**事实**：
- 全局开关检查的是 `current_user`
- `client credentials (无 session)` 的 `current_user` 为 `nil`
- 全局开关启用时，`client credentials` 也会被拦截并返回 401
- 全局开关要求的是"用户认证"，而非"应用认证"

#### 误解 4："in:public 允许未登录用户搜索"

**事实**：
- `in:public` 只影响**搜索哪些索引**
- 登录态门禁（`@account.present?`）是**独立**的检查
- 即使使用 `in:public`，用户仍需登录才能通过门禁
- `无 token + 未登录` 和 `client credentials (无 session)` 都无法通过此门禁

#### 误解 5："resource owner token 与 session 登录权限相同"

**事实**：
- `resource owner token`：`current_user` 存在，但 `user_signed_in?` 可能为 `false`
- `无 token + 已登录 session`：`current_user` 存在，且 `user_signed_in? = true`
- **差异**：`user_signed_in?` 为 `false` 时，无法使用分页和远程解析
- 这是前端 Web UI 与 API 客户端的权限差异

### 12.3 入口权限矩阵总结回顾

#### 全局开关关闭（默认）

| 身份场景 | current_user | statuses | accounts | hashtags |
|----------|--------------|----------|----------|----------|
| 无 token + 未登录 | `nil` | ❌ 空数组 | ✅ 可搜索 | ✅ 可搜索 |
| 无 token + 已登录 | User 对象 | ✅ 完整结果 | ✅ 可搜索 | ✅ 可搜索 |
| client credentials (无 session) | `nil` | ❌ 空数组 | ✅ 可搜索 | ✅ 可搜索 |
| client credentials (有 session) | User 对象 | ✅ 完整结果 | ✅ 可搜索 | ✅ 可搜索 |
| resource owner | User 对象 | ✅ 完整结果 | ✅ 可搜索 | ✅ 可搜索 |

#### 全局开关开启

| 身份场景 | current_user | statuses | accounts | hashtags |
|----------|--------------|----------|----------|----------|
| 无 token + 未登录 | `nil` | ❌ 401 | ❌ 401 | ❌ 401 |
| 无 token + 已登录 | User 对象 | ✅ 完整结果 | ✅ 可搜索 | ✅ 可搜索 |
| client credentials (无 session) | `nil` | ❌ 401 | ❌ 401 | ❌ 401 |
| client credentials (有 session) | User 对象 | ✅ 完整结果 | ✅ 可搜索 | ✅ 可搜索 |
| resource owner | User 对象 | ✅ 完整结果 | ✅ 可搜索 | ✅ 可搜索 |

### 12.4 设计意图总结

Mastodon 的这种多层权限设计体现了以下设计哲学：

1. **身份明确性优先**：
   - `client credentials` 是"应用认证"，不代表用户身份
   - 只有 `resource owner` 或 `session 登录` 才是明确的"用户授权"
   - 状态搜索需要明确的用户身份来进行个性化权限判断

2. **安全分层防御**：
   - 多层过滤确保即使某一层出现问题，其他层仍能提供保护
   - 二次过滤弥补索引更新延迟的问题
   - 全局开关提供实例级别的访问控制

3. **数据分类保护**：
   - 状态搜索需要登录（可能包含私人互动）
   - 账户/标签搜索无需登录（公开数据）
   - `in:library` 搜索需要更强的权限控制（个人互动历史）

4. **灵活配置**：
   - 全局开关允许实例管理员完全禁止未认证访问
   - 默认配置下提供相对开放的体验（账户/标签可搜索）
   - `in:` 参数允许用户精确控制搜索范围

5. **隐私保护**：
   - 未登录用户无法搜索状态（即使是公开状态）
   - `StatusesIndex` 通过 `searchable_by` 限制私有内容的可见性
   - 拉黑/静音关系在结果阶段强制执行

## 13. 技术设计特点

### 13.1 多层过滤的优势

1. **性能优化**：
   - 身份识别：在最早阶段确定身份
   - 全局开关：拦截无效请求
   - 登录态门禁：在服务层拦截状态搜索
   - 索引阶段预计算权限，减少查询时的计算量
   - ES 查询阶段过滤大部分无权限内容
   - 结果阶段只对少量候选结果进行精细检查

2. **安全性保障**：
   - `client credentials (无 session)` 无法伪装成用户
   - 全局开关可完全禁止未认证访问
   - 登录态门禁确保状态搜索需要登录
   - `in:public` 和 `in:library` 提供明确的内容隔离
   - 即使索引权限计算有误，结果阶段的二次过滤仍能保障安全
   - 动态变化的关系（如拉黑、关注）在结果阶段实时检查

3. **灵活性**：
   - 管理员可通过全局开关控制整体访问策略
   - `searchable_by` 字段可以覆盖复杂的权限场景
   - `in:` 参数允许用户精确控制搜索范围
   - `StatusPolicy` 可以实现精细化的权限规则

### 13.2 潜在注意事项

1. **索引更新延迟**：
   - 权限关系变更（如关注、拉黑）不会立即反映在 `searchable_by` 字段中
   - 需要等待状态重新索引才能更新权限信息
   - 但二次过滤可以弥补这一延迟

2. **索引膨胀**：
   - 对于热门状态，`searchable_by` 可能包含大量用户 ID
   - 但 Mastodon 限制了只索引本地用户，减轻了这个问题

3. **登录态门禁的严格性**：
   - 即使使用 `in:public` 只搜索公开内容，也需要登录
   - 即使使用 `client credentials token` 通过了 OAuth 认证，状态搜索仍为空
   - 这是设计决策，可能限制了某些自动化场景
   - 但增强了隐私保护和防止滥用

## 14. 关键代码位置汇总

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| current_resource_owner | `app/controllers/api/base_controller.rb` | 43-45 |
| current_user 重定义（含 super） | `app/controllers/api/base_controller.rb` | 47-51 |
| require_authenticated_user! | `app/controllers/api/base_controller.rb` | 57-59 |
| require_client_credentials! | `app/controllers/api/base_controller.rb` | 53-55 |
| authorize_if_got_token! | `app/controllers/api/base_controller.rb` | 90-92 |
| 全局开关判断 | `app/controllers/api/base_controller.rb` | 94-96 |
| current_account | `app/controllers/application_controller.rb` | 119-123 |
| user_signed_in? 额外限制 | `app/controllers/api/v2/search_controller.rb` | 12-15 |
| 状态搜索门禁 | `app/services/search_service.rb` | 86-88 |
| 账户搜索门禁 | `app/services/search_service.rb` | 90-92 |
| 标签搜索门禁 | `app/services/search_service.rb` | 94-96 |
| 搜索服务入口 | `app/services/search_service.rb` | 6-27 |
| 状态搜索服务 | `app/services/statuses_search_service.rb` | 27-38 |
| 索引选择逻辑 | `app/lib/search_query_transformer.rb` | 57-66 |
| 查询默认过滤器 | `app/lib/search_query_transformer.rb` | 68-98 |
| in: 参数解析 | `app/lib/search_query_transformer.rb` | 146-188 |
| 状态过滤器 | `app/lib/status_filter.rb` | 11-71 |
| 状态权限策略 | `app/policies/status_policy.rb` | 4-14 |
| 状态搜索扩展 | `app/models/concerns/status/search_concern.rb` | 6-24 |
| 公开状态索引 | `app/chewy/public_statuses_index.rb` | 55-68 |
| 私有状态索引 | `app/chewy/statuses_index.rb` | 55-66 |
| API 控制器 | `app/controllers/api/v2/search_controller.rb` | 9-15, 65-72 |
| Chewy 策略 | `lib/chewy/strategy/mastodon.rb` | 12-27 |
| 索引调度器 | `app/workers/scheduler/indexing_scheduler.rb` | 13-31 |
| OAuth 配置 | `config/initializers/doorkeeper.rb` | 170 |
| 配置定义 | `config/mastodon.yml` | 4 |
