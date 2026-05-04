# OAuth 应用注册与授权流程分析

## 1. 概述

Mastodon 使用 [Doorkeeper](https://github.com/doorkeeper-gem/doorkeeper) gem 作为 OAuth 2.0 提供者实现，提供完整的 OAuth 应用注册、授权和令牌管理功能。

## 2. OAuth 应用注册流程

### 2.1 应用注册入口

用户通过设置页面的应用管理界面进行 OAuth 应用注册，主要控制器为 `Settings::ApplicationsController`。

**关键代码位置：**
- `app/controllers/settings/applications_controller.rb`

### 2.2 注册流程步骤

1. **创建新应用表单** (`new` 动作)
   - 用户填写应用名称、重定向 URI、网站 URL
   - 默认预填 `redirect_uri` 为 `Doorkeeper.configuration.native_redirect_uri`
   - 默认权限范围为 `profile`

2. **提交创建** (`create` 动作)
   ```ruby
   @application = current_user.applications.build(application_params)
   if @application.save
     redirect_to settings_applications_path, notice: I18n.t('applications.created')
   end
   ```
   - 应用与当前用户关联（通过 `enable_application_owner` 配置）
   - 自动生成客户端 ID (`uid`) 和客户端密钥 (`secret`)

3. **应用参数** (`application_params`)
   ```ruby
   params.expect(doorkeeper_application: [:name, :redirect_uri, :website, scopes: []])
   ```
   - 必需字段：`name`, `redirect_uri`, `scopes`
   - 可选字段：`website`

### 2.3 应用模型

应用基于 `Doorkeeper::Application` 模型，核心属性包括：
- `name` - 应用名称
- `uid` - 客户端 ID（自动生成）
- `secret` - 客户端密钥（自动生成）
- `redirect_uri` - 重定向 URI
- `scopes` - 权限范围
- `website` - 应用网站 URL
- `superapp` - 是否为超级应用（跳过授权确认）

### 2.4 重定向 URI 安全限制

**配置位置：** `config/initializers/doorkeeper.rb:151`

```ruby
forbid_redirect_uri { |uri| %w(data vbscript javascript).include?(uri.scheme.to_s.downcase) }
```

- 禁止使用 `data:`、`vbscript:`、`javascript:` 等危险协议
- 开发环境允许 HTTP，生产环境强制 HTTPS（通过 `force_ssl_in_redirect_uri` 配置）

## 3. OAuth 授权流程

### 3.1 支持的授权类型

**配置位置：** `config/initializers/doorkeeper.rb:170`

```ruby
grant_flows %w(authorization_code client_credentials)
```

Mastodon 支持两种 OAuth 2.0 授权流程：
1. **授权码流程 (Authorization Code Flow)** - 适用于第三方应用
2. **客户端凭证流程 (Client Credentials Flow)** - 适用于服务器到服务器通信

**禁用的流程：**
- 密码模式 (Resource Owner Password Credentials) - 已禁用（返回 `nil`）
- 简化模式 (Implicit) - 未启用

### 3.2 授权码流程详解

#### 3.2.1 授权请求

**控制器：** `app/controllers/oauth/authorizations_controller.rb`

```ruby
class OAuth::AuthorizationsController < Doorkeeper::AuthorizationsController
  prepend_before_action :store_current_location
  layout 'modal'
  
  private
  
  def can_authorize_response?
    !truthy_param?('force_login') && super
  end
end
```

**授权端点：** `GET /oauth/authorize`

**请求参数：**
- `client_id` - 客户端 ID
- `redirect_uri` - 重定向 URI
- `response_type` - 必须为 `code`
- `scope` - 请求的权限范围（空格分隔）
- `state` - 防 CSRF 状态值
- `force_login` - 可选，强制用户重新登录

#### 3.2.2 PKCE 支持

**配置位置：** `config/initializers/doorkeeper.rb:56`

```ruby
pkce_code_challenge_methods ['S256']
```

Mastodon 支持 Proof Key for Code Exchange (PKCE)，增强公共客户端的安全性：
- 仅支持 `S256` 哈希方法
- 客户端需生成 `code_verifier` 并发送 `code_challenge`

#### 3.2.3 授权确认页面

**视图位置：** `app/views/oauth/authorizations/new.html.haml`

页面显示：
- 应用名称和网站
- 请求的权限范围（分组显示）
- 授权和拒绝按钮

#### 3.2.4 超级应用跳过授权

**配置位置：** `config/initializers/doorkeeper.rb:175-177`

```ruby
skip_authorization do |_resource_owner, client|
  client.application.superapp?
end
```

- `superapp: true` 的应用（如内置 Web 应用）会跳过用户授权确认步骤
- 自动授权所有请求的权限范围

### 3.3 令牌发放

**控制器：** `app/controllers/oauth/tokens_controller.rb`

```ruby
class OAuth::TokensController < Doorkeeper::TokensController
  def revoke
    unsubscribe_for_token if token.present? && authorized? && token.accessible?
    super
  end
  
  private
  
  def unsubscribe_for_token
    Web::PushSubscription.where(access_token_id: token.id).delete_all
  end
end
```

**令牌端点：** `POST /oauth/token`

**支持的 grant_type：**
1. `authorization_code` - 使用授权码换取令牌
2. `client_credentials` - 客户端凭证模式
3. `refresh_token` - 刷新令牌（需启用 `use_refresh_token`，当前未启用）

### 3.4 令牌配置

**配置位置：** `config/initializers/doorkeeper.rb`

```ruby
access_token_expires_in nil    # 令牌永不过期
reuse_access_token             # 同一用户同一应用重用令牌
```

**关键特性：**
- **令牌永不过期**：`access_token_expires_in nil`，用户需手动撤销
- **令牌重用**：`reuse_access_token`，同一用户对同一应用的授权会返回相同的 access token
- **无刷新令牌**：`use_refresh_token` 被注释掉，不支持刷新令牌机制

### 3.5 令牌撤销

**端点：** `POST /oauth/revoke`

撤销令牌时会：
1. 使 access token 失效
2. 删除与该 token 关联的 Web Push 订阅
3. 关闭相关的流式连接（在 `AuthorizedApplicationsController#destroy` 中）

## 4. Access Token 权限范围 (Scopes)

### 4.1 范围定义

**配置位置：** `config/initializers/doorkeeper.rb:72-119`

```ruby
default_scopes  :read
optional_scopes :profile,
                :write,
                :'write:accounts',
                # ... 更多细粒度范围
                :read,
                :'read:accounts',
                # ... 更多细粒度范围
                :follow,
                :push,
                :'admin:read',
                :'admin:write'
```

### 4.2 范围分类详解

#### 4.2.1 默认范围

| 范围 | 说明 |
|------|------|
| `read` | 只读访问（默认） |

#### 4.2.2 基础范围

| 范围 | 说明 |
|------|------|
| `profile` | 访问用户个人资料信息 |
| `write` | 写入权限（包含所有 write:* 子范围） |
| `read` | 读取权限（包含所有 read:* 子范围） |
| `follow` | 关注/取消关注用户 |
| `push` | Web Push 通知订阅 |

#### 4.2.3 细粒度写入范围

| 范围 | 说明 |
|------|------|
| `write:accounts` | 修改账户信息（更新资料、设置等） |
| `write:blocks` | 管理黑名单 |
| `write:bookmarks` | 管理书签 |
| `write:collections` | 管理收藏集 |
| `write:conversations` | 管理私信对话 |
| `write:favourites` | 管理收藏 |
| `write:filters` | 管理过滤器 |
| `write:follows` | 管理关注关系 |
| `write:lists` | 管理列表 |
| `write:media` | 上传媒体文件 |
| `write:mutes` | 管理静音 |
| `write:notifications` | 管理通知设置 |
| `write:reports` | 提交举报 |
| `write:statuses` | 发布、编辑、删除嘟文 |

#### 4.2.4 细粒度读取范围

| 范围 | 说明 |
|------|------|
| `read:accounts` | 查看账户信息 |
| `read:blocks` | 查看黑名单 |
| `read:bookmarks` | 查看书签 |
| `read:collections` | 查看收藏集 |
| `read:favourites` | 查看收藏 |
| `read:filters` | 查看过滤器 |
| `read:follows` | 查看关注关系 |
| `read:lists` | 查看列表 |
| `read:mutes` | 查看静音 |
| `read:notifications` | 查看通知 |
| `read:search` | 搜索功能 |
| `read:statuses` | 查看嘟文 |

#### 4.2.5 管理员范围

| 范围 | 说明 |
|------|------|
| `admin:read` | 管理员只读权限（包含所有 admin:read:* 子范围） |
| `admin:read:accounts` | 查看用户账户管理信息 |
| `admin:read:reports` | 查看举报 |
| `admin:read:domain_allows` | 查看域名允许列表 |
| `admin:read:domain_blocks` | 查看域名封禁列表 |
| `admin:read:ip_blocks` | 查看 IP 封禁列表 |
| `admin:read:email_domain_blocks` | 查看邮箱域名封禁 |
| `admin:read:canonical_email_blocks` | 查看规范化邮箱封禁 |
| `admin:write` | 管理员写入权限（包含所有 admin:write:* 子范围） |
| `admin:write:accounts` | 管理用户账户 |
| `admin:write:reports` | 处理举报 |
| `admin:write:domain_allows` | 管理域名允许列表 |
| `admin:write:domain_blocks` | 管理域名封禁列表 |
| `admin:write:ip_blocks` | 管理 IP 封禁列表 |
| `admin:write:email_domain_blocks` | 管理邮箱域名封禁 |
| `admin:write:canonical_email_blocks` | 管理规范化邮箱封禁 |

### 4.3 范围层级关系（继承机制）

Doorkeeper 支持范围的层级关系，使用冒号 `:` 分隔，实现粗粒度范围到细粒度范围的自动继承。

#### 4.3.1 继承规则

| 粗粒度范围 | 自动包含的细粒度范围 |
|-----------|---------------------|
| `write` | `write:accounts`, `write:blocks`, `write:bookmarks`, `write:collections`, `write:conversations`, `write:favourites`, `write:filters`, `write:follows`, `write:lists`, `write:media`, `write:mutes`, `write:notifications`, `write:reports`, `write:statuses` |
| `read` | `read:accounts`, `read:blocks`, `read:bookmarks`, `read:collections`, `read:favourites`, `read:filters`, `read:follows`, `read:lists`, `read:mutes`, `read:notifications`, `read:search`, `read:statuses` |
| `admin:read` | `admin:read:accounts`, `admin:read:reports`, `admin:read:domain_allows`, `admin:read:domain_blocks`, `admin:read:ip_blocks`, `admin:read:email_domain_blocks`, `admin:read:canonical_email_blocks` |
| `admin:write` | `admin:write:accounts`, `admin:write:reports`, `admin:write:domain_allows`, `admin:write:domain_blocks`, `admin:write:ip_blocks`, `admin:write:email_domain_blocks`, `admin:write:canonical_email_blocks` |

#### 4.3.2 继承机制示例

**场景 1：Token 具有 `write` 范围**
- 自动拥有所有 `write:*` 细粒度权限
- 可以访问任何需要 `write` 或 `write:statuses` 或 `write:favourites` 等的接口

**场景 2：Token 仅具有 `write:statuses` 范围**
- 只能访问需要 `write:statuses` 的接口
- 不能访问需要 `write:favourites` 或其他细粒度写入权限的接口
- 不能访问需要 `write` 粗粒度范围的接口（但实际上 `write` 包含 `write:statuses`，所以应该可以？需要验证）

#### 4.3.3 流式服务中的范围优先级

**位置：** `streaming/index.js:462-487`

```javascript
const checkScopes = (req, logger, channelName) => new Promise((resolve, reject) => {
  // The `read` scope has the highest priority, if the token has it
  // then it can access all streams
  const requiredScopes = ['read'];
  
  // When accessing specifically the notifications stream,
  // we need a read:notifications, while in all other cases,
  // we can allow access with read:statuses.
  if (channelName === 'user:notification') {
    requiredScopes.push('read:notifications');
  } else {
    requiredScopes.push('read:statuses');
  }
  
  if (req.scopes && requiredScopes.some(requiredScope => req.scopes.includes(requiredScope))) {
    resolve();
    return;
  }
  
  reject(new AuthenticationError('Access token does not have the required scopes'));
});
```

**关键点：**
1. **`read` 范围优先级最高**：如果 token 具有 `read` 范围，可以访问所有流式 API
2. **细粒度范围作为备选**：
   - 通知流需要 `read:notifications`
   - 其他流需要 `read:statuses`
3. **匹配规则**：`requiredScopes.some(...)` - 只要 token 具有其中任意一个范围即可通过

### 4.4 范围强制配置

**配置位置：** `config/initializers/doorkeeper.rb:61`

```ruby
enforce_configured_scopes
```

此配置确保：
- 应用只能请求 `default_scopes` 或 `optional_scopes` 中定义的范围
- 防止应用请求未授权的自定义范围

## 5. 权限范围校验机制

### 5.1 控制器级别的权限校验

#### 5.1.1 基础控制器

**位置：** `app/controllers/api/base_controller.rb`

```ruby
def doorkeeper_authorize!(*scopes)
  # Doorkeeper 提供的权限校验方法
end

def authorize_if_got_token!(*scopes)
  doorkeeper_authorize!(*scopes) if doorkeeper_token
end
```

#### 5.1.2 权限校验方法

**`doorkeeper_authorize!(*scopes)`**
- 强制校验 access token 是否具有指定的权限范围
- 如果 token 不存在或权限不足，返回 401 或 403 错误

**`authorize_if_got_token!(*scopes)`**
- 仅当存在 access token 时才进行权限校验
- 允许未登录用户访问（用于公开可读的 API）

### 5.2 API 控制器中的权限校验示例

#### 5.2.1 嘟文控制器

**位置：** `app/controllers/api/v1/statuses_controller.rb:8-9`

```ruby
before_action -> { authorize_if_got_token! :read, :'read:statuses' }, except: [:create, :update, :destroy]
before_action -> { doorkeeper_authorize! :write, :'write:statuses' }, only:   [:create, :update, :destroy]
```

**权限策略：**
- **读取操作**（index, show, context）：
  - 使用 `authorize_if_got_token!`
  - 允许未登录用户访问公开嘟文
  - 登录用户需要 `read` 或 `read:statuses` 范围
  
- **写入操作**（create, update, destroy）：
  - 使用 `doorkeeper_authorize!`
  - 必须登录且具有 `write` 或 `write:statuses` 范围

#### 5.2.2 通知控制器

**位置：** `app/controllers/api/v1/notifications_controller.rb`（根据 `doorkeeper_authorize!` 搜索结果）

```ruby
before_action -> { doorkeeper_authorize! :read, :'read:notifications' }
```

### 5.3 Scope 校验完整链路

Scope 校验发生在 OAuth 流程的多个阶段，从授权请求到 API 接口调用形成一条完整的校验链。

#### 5.3.1 阶段一：授权请求校验

**端点：** `GET /oauth/authorize`

**校验内容：**
1. **应用范围限制**：请求的 scope 必须是应用注册时允许的范围的子集
   - 应用在 `Doorkeeper::Application` 的 `scopes` 字段定义了允许的范围
   - 例如：应用注册时 `scopes: 'read write'`，则只能请求 `read`、`write` 或其子范围

2. **配置范围强制**：请求的 scope 必须在 `doorkeeper.rb` 配置的 `default_scopes` 或 `optional_scopes` 中
   ```ruby
   # config/initializers/doorkeeper.rb:61
   enforce_configured_scopes
   ```

**校验失败处理：**
- 如果请求的 scope 不在允许范围内，Doorkeeper 会返回错误

#### 5.3.2 阶段二：用户授权确认

**界面：** `app/views/oauth/authorizations/new.html.haml`

**流程：**
1. 用户看到应用请求的所有 scope 列表
2. 用户可以选择授权或拒绝
3. 如果授权，scope 被记录到 `Doorkeeper::AccessGrant` 的 `scopes` 字段

**关键点：**
- 用户只能授权应用请求的 scope，不能增加或减少
- 授权的 scope 会被完整记录到授权码中

#### 5.3.3 阶段三：令牌签发校验

**端点：** `POST /oauth/token`

**校验逻辑：**

**授权码模式：**
- 令牌的 scope 来自于 `AccessGrant` 的 `scopes` 字段
- 不能请求超出授权码范围的 scope

**测试验证** (`spec/requests/oauth/token_spec.rb:32-65`)：
```ruby
shared_examples 'original scope request preservation' do
  it 'returns all scopes requested for the given code' do
    subject
    expect(response).to have_http_status(200)
    expect(response.parsed_body[:scope]).to eq 'read write'
  end
end

context 'with scopes specified' do
  context 'when the scopes were requested for this code' do
    let(:scope) { 'write' }
    it_behaves_like 'original scope request preservation'
  end
  
  context 'when the scope was not requested for the code' do
    let(:scope) { 'follow' }
    it_behaves_like 'original scope request preservation'
  end
end
```

**关键点：**
- 即使在令牌请求时指定了不同的 scope，返回的令牌仍然使用授权码原始的 scope
- 这确保了令牌不会超出用户授权的范围

**客户端凭证模式：**
- 可以请求应用允许的任意 scope 子集
- 如果请求的 scope 不属于应用，返回 400 错误

**测试验证** (`spec/requests/oauth/token_spec.rb:95-104`)：
```ruby
context 'when some scopes do not belong to the application' do
  let(:scope) { 'read write push' }
  
  it 'returns an error' do
    subject
    expect(response).to have_http_status(400)
  end
end
```

#### 5.3.4 阶段四：API 接口鉴权

**位置：** 各 API 控制器的 `before_action`

**校验方法：** `doorkeeper_authorize!(*scopes)`

这是最终的权限校验点，在每个 API 请求时执行。

### 5.4 多范围请求的匹配规则

#### 5.4.1 OR 匹配规则

`doorkeeper_authorize!` 方法接收多个 scope 参数，使用 **OR** 逻辑进行匹配。

**示例 1：嘟文控制器** (`app/controllers/api/v1/statuses_controller.rb:8-9`)
```ruby
before_action -> { authorize_if_got_token! :read, :'read:statuses' }, except: [:create, :update, :destroy]
before_action -> { doorkeeper_authorize! :write, :'write:statuses' }, only:   [:create, :update, :destroy]
```

**匹配逻辑：**
- 对于读取操作：Token 具有 `read` **OR** `read:statuses` 即可通过
- 对于写入操作：Token 具有 `write` **OR** `write:statuses` 即可通过

**示例 2：用户凭证控制器** (`app/controllers/api/v1/accounts/credentials_controller.rb:4-5`)
```ruby
before_action -> { doorkeeper_authorize! :profile, :read, :'read:accounts' }, except: [:update]
before_action -> { doorkeeper_authorize! :write, :'write:accounts' }, only: [:update]
```

**匹配逻辑：**
- 读取用户凭证：`profile` **OR** `read` **OR** `read:accounts`
- 更新用户凭证：`write` **OR** `write:accounts`

#### 5.4.2 匹配规则详解

| API 要求 | Token 具有 | 结果 |
|---------|-----------|------|
| `:read, :'read:statuses'` | `read` | ✅ 通过（OR 匹配） |
| `:read, :'read:statuses'` | `read:statuses` | ✅ 通过（OR 匹配） |
| `:read, :'read:statuses'` | `write` | ❌ 拒绝 |
| `:write, :'write:statuses'` | `write` | ✅ 通过 |
| `:write, :'write:statuses'` | `write:statuses` | ✅ 通过 |
| `:write, :'write:statuses'` | `write:favourites` | ❌ 拒绝（只匹配具体列出的 scope） |

### 5.5 粗粒度与细粒度的继承关系详解

#### 5.5.1 Doorkeeper 的 Scope 继承机制

**关键点：** `doorkeeper_authorize!` 列出的 scope 之间是 OR 关系，同时粗粒度 scope 具有继承特性。

**继承规则：**
- 粗粒度 scope（如 `write`）自动包含所有同名的细粒度子 scope（如 `write:statuses`、`write:favourites` 等）
- 当 API 要求 `write` 时，具有 `write` **或** 任何 `write:*` 子 scope 的 token 都能通过
- 当 API 要求 `write:statuses` 时，具有 `write:statuses` **或** `write` 的 token 都能通过

#### 5.5.2 继承关系示例分析

**示例分析 1：API 要求 `doorkeeper_authorize! :read, :'read:statuses'`**

| Token Scope | 是否通过 | 原因 |
|------------|---------|------|
| `read` | ✅ | 直接匹配 `:read` |
| `read:statuses` | ✅ | 直接匹配 `:'read:statuses'` |
| `read:accounts` | ❌ | 不匹配任何列出的 scope |
| `write` | ❌ | 不匹配任何列出的 scope |

**示例分析 2：API 要求 `doorkeeper_authorize! :write`**

| Token Scope | 是否通过 | 原因 |
|------------|---------|------|
| `write` | ✅ | 直接匹配 |
| `write:statuses` | ✅ | `write` 包含所有 `write:*` 子范围 |
| `write:favourites` | ✅ | `write` 包含所有 `write:*` 子范围 |
| `read` | ❌ | 不匹配 |

**示例分析 3：API 要求 `doorkeeper_authorize! :'write:statuses'`**

| Token Scope | 是否通过 | 原因 |
|------------|---------|------|
| `write` | ✅ | `write` 包含 `write:statuses` |
| `write:statuses` | ✅ | 直接匹配 |
| `write:favourites` | ❌ | `write:favourites` 不包含 `write:statuses` |
| `read` | ❌ | 不匹配 |

#### 5.5.3 继承机制核心结论

**这说明：**
1. **粗粒度包容细粒度**：当 API 要求粗粒度 scope（如 `:write`）时，具有该粗粒度或任意细粒度子 scope 的 token 都能通过
2. **细粒度继承父粗粒度**：当 API 要求细粒度 scope（如 `:'write:statuses'`）时，具有该细粒度或其父粗粒度 scope 的 token 能通过
3. **细粒度之间互不包含**：细粒度 scope 之间**不**互相包含（如 `write:statuses` 不包含 `write:favourites`）

### 5.6 权限校验错误处理

#### 5.6.1 401 Unauthorized vs 403 Forbidden

**位置：** `app/controllers/api/base_controller.rb:23-29`

```ruby
def doorkeeper_unauthorized_render_options(error: nil)
  { json: { error: error.try(:description) || 'Not authorized' } }
end

def doorkeeper_forbidden_render_options(*)
  { json: { error: 'This action is outside the authorized scopes' } }
end
```

#### 5.6.2 错误场景详解

| 错误类型 | HTTP 状态码 | 触发条件 | 错误消息 |
|---------|------------|---------|---------|
| **Unauthorized** | 401 | Token 不存在、无效、已撤销或格式错误 | `"Not authorized"` 或具体错误描述 |
| **Forbidden** | 403 | Token 有效，但权限范围不足 | `"This action is outside the authorized scopes"` |

**关键区别：**
- **401**：认证失败 —— 系统无法识别你是谁
- **403**：授权失败 —— 系统知道你是谁，但你没有权限执行该操作

#### 5.6.3 测试验证

**位置：** `spec/support/examples/api.rb:3-12`

```ruby
RSpec.shared_examples 'forbidden for wrong scope' do |wrong_scope|
  let(:scopes) { wrong_scope }

  it 'returns http forbidden' do
    subject if request.nil?
    expect(response).to have_http_status(403)
  end
end
```

**使用示例** (`spec/requests/api/v1/statuses_spec.rb:51`)：
```ruby
it_behaves_like 'forbidden for wrong scope', 'write write:statuses'
```

这说明：
- 当 API 需要 `read` 或 `read:statuses` 时
- 使用 `write` 或 `write:statuses` 的 token 会收到 **403** 错误
- 而不是 401，因为 token 本身是有效的，只是权限不足

#### 5.6.4 流式服务中的错误处理

**位置：** `streaming/index.js`

**Token 不存在：**
```javascript
if (!authorization && !accessToken) {
  reject(new AuthenticationError('Missing access token'));
}
```

**Token 无效：**
```javascript
if (result.rows.length === 0) {
  throw new AuthenticationError('Invalid access token');
}
```

**Scope 不足：**
```javascript
if (req.scopes && requiredScopes.some(requiredScope => req.scopes.includes(requiredScope))) {
  resolve();
} else {
  reject(new AuthenticationError('Access token does not have the required scopes'));
}
```

**流式服务的 HTTP 状态码：**
- 所有认证错误都返回 **401**（WebSocket 连接的特殊处理）
- 错误消息用于区分具体原因：
  - `'Missing access token'`
  - `'Invalid access token'`
  - `'Access token does not have the required scopes'`

### 5.7 原权限校验错误处理

**位置：** `app/controllers/api/base_controller.rb:23-29`

```ruby
def doorkeeper_unauthorized_render_options(error: nil)
  { json: { error: error.try(:description) || 'Not authorized' } }
end

def doorkeeper_forbidden_render_options(*)
  { json: { error: 'This action is outside the authorized scopes' } }
end
```

**错误响应：**
- **401 Unauthorized**：Token 无效或不存在
  ```json
  { "error": "Not authorized" }
  ```
  
- **403 Forbidden**：Token 权限不足
  ```json
  { "error": "This action is outside the authorized scopes" }
  ```

### 5.4 资源所有者认证

**位置：** `app/controllers/api/base_controller.rb:43-51`

```ruby
def current_resource_owner
  @current_user ||= User.find(doorkeeper_token.resource_owner_id) if doorkeeper_token
end

def current_user
  current_resource_owner || super
rescue ActiveRecord::RecordNotFound
  nil
end
```

- `current_resource_owner` 从 access token 获取关联的用户
- `current_user` 优先使用 OAuth 认证的用户，回退到 Devise session 认证

### 5.5 客户端凭证模式的特殊处理

**位置：** `app/controllers/api/base_controller.rb:53-55`

```ruby
def require_client_credentials!
  render json: { error: 'This method requires an client credentials authentication' }, status: 403 if doorkeeper_token.resource_owner_id.present?
end
```

某些 API 端点可能要求纯客户端凭证认证（无用户关联），此方法用于确保 token 不关联任何用户。

## 6. 授权应用管理

### 6.1 用户已授权应用列表

**控制器：** `app/controllers/oauth/authorized_applications_controller.rb`

**功能：**
- 列出用户已授权的所有应用
- 显示每个应用的权限范围
- 允许用户撤销应用授权

### 6.2 撤销授权

```ruby
def destroy
  Web::PushSubscription.unsubscribe_for(params[:id], current_resource_owner)
  Doorkeeper::Application.find_by(id: params[:id])&.close_streaming_sessions(current_resource_owner)
  super
end
```

撤销授权时会：
1. 删除该应用的 Web Push 订阅
2. 关闭相关的流式连接
3. 删除 access token

## 7. 安全特性

### 7.1 令牌安全性

- **永不过期**：令牌不会自动过期，用户必须手动撤销
- **令牌重用**：同一用户对同一应用只生成一个令牌，减少令牌数量
- **无刷新令牌**：简化实现，但用户需重新授权来更新权限

### 7.2 PKCE 支持

支持 PKCE 增强公共客户端（如移动端、桌面应用）的安全性，防止授权码拦截攻击。

### 7.3 重定向 URI 验证

- 禁止危险协议（data:, vbscript:, javascript:）
- 生产环境强制 HTTPS
- 精确匹配重定向 URI（Doorkeeper 默认行为）

### 7.4 范围强制

`enforce_configured_scopes` 确保应用只能请求预定义的权限范围，防止权限逃逸。

## 8. 关键代码文件汇总

| 文件路径 | 功能说明 |
|----------|----------|
| `config/initializers/doorkeeper.rb` | Doorkeeper 核心配置，定义范围、授权流程等 |
| `app/controllers/settings/applications_controller.rb` | OAuth 应用注册和管理控制器 |
| `app/controllers/oauth/authorizations_controller.rb` | OAuth 授权控制器 |
| `app/controllers/oauth/tokens_controller.rb` | OAuth 令牌控制器 |
| `app/controllers/oauth/authorized_applications_controller.rb` | 用户已授权应用管理 |
| `app/controllers/api/base_controller.rb` | API 基础控制器，包含权限校验方法 |
| `app/controllers/api/v1/statuses_controller.rb` | 嘟文 API，权限校验示例 |
| `db/seeds/01_web_app.rb` | 内置 Web 应用种子数据 |

## 9. 总结

Mastodon 的 OAuth 2.0 实现具有以下特点：

1. **完整的 OAuth 2.0 支持**：基于 Doorkeeper gem，实现了授权码流程和客户端凭证流程
2. **细粒度权限控制**：通过层级化的 scope 设计，支持从粗粒度（write, read）到细粒度（write:statuses, read:notifications）的权限控制
3. **开发者友好**：允许用户自主注册和管理 OAuth 应用，自动生成客户端凭证
4. **安全设计**：支持 PKCE、重定向 URI 验证、范围强制等安全特性
5. **灵活的权限校验**：提供 `doorkeeper_authorize!` 和 `authorize_if_got_token!` 两种校验方式，适应不同的 API 安全需求

权限范围的设计遵循 OAuth 2.0 最佳实践，通过分层 scope 机制实现了最小权限原则，同时保持了良好的易用性。
