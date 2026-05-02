# Mastodon 受限作者与 Inbox 入口联动机制完整分析

## 重要概念区分：Silenced vs Suspended

| 状态 | 字段 | 含义 | 联邦影响 |
|------|------|------|----------|
| **Silenced** (受限/静音) | `silenced_at` | 用户仍可使用，但内容受限 | 影响 direct/limited 状态的受众 |
| **Suspended** (暂停/封禁) | `suspended_at` | 账户被禁用，无法使用 | 完全排除在联邦之外 |

---

## 一、受限作者 (Silenced) 的受众过滤机制

### 1.1 关键代码位置

**文件**: `app/lib/activitypub/tag_manager.rb:173-196`

```ruby
def to(status)
  case status.visibility
  when 'public'
    [COLLECTIONS[:public]]
  when 'unlisted', 'private'
    [followers_uri_for(status.account)]
  when 'direct', 'limited'
    if status.account.silenced?
      # ⚠️ 关键：受限作者的特殊过滤
      # Only notify followers if the account is locally silenced
      account_ids = status.active_mentions.pluck(:account_id)
      to = status.account.followers.where(id: account_ids).each_with_object([]) do |account, result|
        result << uri_for(account)
        result << followers_uri_for(account) if account.group?
      end
      to.concat(FollowRequest.where(target_account_id: status.account_id, account_id: account_ids).each_with_object([]) do |request, result|
        result << uri_for(request.account)
        result << followers_uri_for(request.account) if request.account.group?
      end).compact
    else
      # 非受限作者：所有被@提及的账户都包含在内
      status.active_mentions.each_with_object([]) do |mention, result|
        result << uri_for(mention.account)
        result << followers_uri_for(mention.account) if mention.account.group?
      end.compact
    end
  end
end
```

### 1.2 过滤逻辑对比表

| 作者状态 | 被@提及账户条件 | to 字段包含内容 |
|---------|----------------|----------------|
| **非受限 (正常) | 无条件 | 所有被@提及的账户 URI |
| **受限 (silenced)** | 必须同时是作者的**关注者** | 只有"被@提及 + 是作者关注者"的账户 URI |

### 1.3 FollowRequest 的特殊处理

受限作者的情况下，还会额外检查 **FollowRequest**：

```ruby
FollowRequest.where(target_account_id: status.account_id, account_id: account_ids)
```

这意味着：如果被@提及的账户**已经向作者发送了关注请求（但尚未被批准），也会被包含在 `to` 字段中。

### 1.4 双重过滤机制

发送侧存在**两层独立的过滤**：

#### 第一层：投递目标过滤 (`StatusReachFinder#inboxes`)

```ruby
# app/lib/status_reach_finder.rb:59-61
def mentioned_account_ids
  @status.mentions.pluck(:account_id)  # ⚠️ 无条件！
end

def inboxes_without_suspended_for(scope)
  scope.merge!(Account.without_suspended) unless unsafe?
  scope.inboxes  # 只排除 suspended，不排除 silenced！
end
```

**关键点**：
- `mentioned_account_ids` 是**无条件**的，不检查 silenced
- `inboxes_without_suspended_for` 只排除 `suspended` 的账户

**不排除 `silenced` 的账户

#### 第二层：to/cc 字段过滤 (`TagManager#to`)

```ruby
if status.account.silenced?
  # 只保留同时是作者关注者的被@提及账户
  status.account.followers.where(id: account_ids)
end
```

### 1.5 过滤不一致的后果

这种设计导致了一个**微妙的安全网机制：

| 场景 | 投递目标 (inboxes) | to/cc 字段 |
|------|---------------------|-------------|
| 受限作者 @ Bob（Bob是作者关注者） | ✅ 包含 Bob | ✅ 包含 Bob.uri |
| 受限作者 @ Bob（Bob**不是**作者关注者） | ✅ 包含 Bob | ❌ **不包含** Bob.uri |
| 非受限作者 @ Bob | ✅ 包含 Bob | ✅ 包含 Bob.uri |

**关键发现**：
- 投递会发送到 Bob 的实例
- 但 `to`/`cc` 字段**不包含** Bob 的 URI
- 接收侧会根据 `to`/`cc` 字段决定是否处理

---

## 二、三种 Inbox 入口的路由与处理差异

### 2.1 路由配置

**文件**: `config/routes.rb`

```ruby
# 1. 共享 Inbox (Shared Inbox)
resource :inbox, only: [:create], module: :activitypub  # POST /inbox

# 2. 用户 Inbox (User Inbox)
concern :account_resources do
  scope module: :activitypub do
    resource :inbox, only: [:create]  # POST /users/:username/inbox
  end
end

# 3. 实例演员 Inbox (Instance Actor Inbox)
resource :instance_actor, path: 'actor', only: [:show] do
  scope module: :activitypub do
    resource :inbox, only: [:create]  # POST /actor/inbox
  end
end
```

### 2.2 三种 Inbox 对比表

| Inbox 类型 | 路由 | `@account` 值 | `delivered_to_account_id` |
|------------|------|---------------|---------------------------|
| **共享 Inbox** | `POST /inbox` | `nil` | `nil` |
| **用户 Inbox** | `POST /users/:username/inbox` | 该用户 Account | 用户 ID |
| **实例演员 Inbox** | `POST /actor/inbox` | 实例演员 Account | 实例演员 ID |

### 2.3 参数传递链路

**文件**: `app/controllers/concerns/account_owned_concern.rb:16-22`

```ruby
def account_required?
  true
end

def set_account
  @account = username_param.present? ? Account.find_local!(username_param) : Account.local.find(account_id_param)
end
```

**文件**: `app/controllers/activitypub/inboxes_controller.rb:35-37, 75-77`

```ruby
def account_required?
  params[:account_username].present?
end

def process_payload
  # @account&.id 就是 delivered_to_account_id
  ActivityPub::ProcessingWorker.perform_async(
    signed_request_actor.id, 
    body, 
    @account&.id,  # ⚠️ 关键：用户 inbox 时有值，共享 inbox 时为 nil
    signed_request_actor.class.name
  )
end
```

---

## 三、接收侧：Inbox 入口与可见性保证的联动机制

### 3.1 核心访问控制检查

**文件**: `app/lib/activitypub/activity/create.rb:445-458`

```ruby
def related_to_local_activity?
  fetch? || followed_by_local_accounts? || requested_through_relay? ||
    responds_to_followed_account? || addresses_local_accounts?
end

def addresses_local_accounts?
  # ⚠️ 第一优先级检查
  return true if @options[:delivered_to_account_id]

  # 否则检查 to/cc 字段
  ActivityPub::TagManager.instance.uris_to_local_accounts((audience_to + audience_cc).uniq).exists?
end
```

### 3.2 联动机制流程图

```
收到 ActivityPub 投递
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│                    InboxesController                             │
│  1. 验证 HTTP Signature                                  │
│  2. 根据路由设置 @account                                  │
│     - 共享 inbox: @account = nil                            │
│     - 用户 inbox: @account = 该用户 Account                 │
└───────────────────────────┬─────────────────────────────────┘
                        │
                        ▼ @account&.id 作为 delivered_to_account_id
┌─────────────────────────────────────────────────────────────┐
│              ProcessingWorker (异步)                          │
└───────────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│         ActivityPub::Activity::Create                          │
│                                                             │
│  related_to_local_activity? 检查                           │
│  ├─ fetch? → 非投递场景                                      │
│  ├─ followed_by_local_accounts? → 作者被本地账户关注      │
│  ├─ requested_through_relay? → 通过中继请求                 │
│  ├─ responds_to_followed_account? → 回复了关注者             │
│  └─ addresses_local_accounts? → ⚠️ 关键检查点              │
│     ├─ 有 delivered_to_account_id? → true ✓                 │
│     └─ 否则检查 to/cc 是否包含本地账户 URI               │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 两种 Inbox 入口的行为对比

#### 场景 A：投递到**共享 Inbox (`POST /inbox`)

| 条件 | `delivered_to_account_id` | `addresses_local_accounts?` | 结果 |
|------|---------------------------|----------------------------|------|
| 受限作者 @ Bob（Bob是关注者） | `nil` | 检查 to/cc → to 包含 Bob.uri → ✓ true | 处理 |
| 受限作者 @ Bob（Bob不是关注者） | `nil` | 检查 to/cc → to **不包含** Bob.uri → ✗ false | **拒绝** |
| 非受限作者 @ Bob | `nil` | 检查 to/cc → to 包含 Bob.uri → ✓ true | 处理 |

#### 场景 B：投递到**用户 Inbox** (`POST /users/bob/inbox`)

| 条件 | `delivered_to_account_id` | `addresses_local_accounts?` | 结果 |
|------|---------------------------|----------------------------|------|
| 受限作者 @ Bob（任何情况） | `bob.id` | **直接返回 true** | 处理 |
| 非受限作者 @ Bob | `bob.id` | **直接返回 true** | 处理 |

### 3.4 关键发现：安全网机制

**发送侧**：
- `StatusReachFinder` 会投递到**所有被@提及的账户（无条件）
- 不检查 silenced

**接收侧（共享 inbox）：
- 依赖 `to`/`cc` 字段来验证是否应该处理
- 受限作者的 `to` 字段被过滤，导致被@但非关注者的账户**无法通过检查

**接收侧（用户 inbox）**：
- `delivered_to_account_id` 存在时直接通过检查
- **不依赖** `to`/`cc` 字段

### 3.5 用户 Inbox 的后续处理：受众修正

即使通过了 `addresses_local_accounts?` 检查，还会有**额外的受众处理**：

**文件**: `app/lib/activitypub/activity/create.rb:124-169`

```ruby
def process_audience
  # 从 to/cc 解析受众
  accounts_in_audience = (audience_to + audience_cc).uniq.filter_map do |audience|
    account_from_uri(audience) unless ActivityPub::TagManager.instance.public_collection?(audience)
  end

  # ⚠️ 关键：如果投递到用户 inbox，该用户必须被添加到受众
  if @options[:delivered_to_account_id]
    accounts_in_audience << delivered_to_account
    accounts_in_audience.uniq!
  end

  accounts_in_audience.each do |account|
    # 跳过已经在 @ 提及的（非静默）
    next if @mentions.any? { |mention| mention.account_id == account.id }

    # 创建静默提及
    @mentions << Mention.new(account: account, silent: true)

    # 如果原本是 direct 但有额外的静默提及 → 升级为 limited
    @params[:visibility] = :limited if @params[:visibility] == :direct
  end
end
```

### 3.6 重复投递处理：状态已存在但新账户

**文件**: `app/lib/activitypub/activity/create.rb:27-35, 156-169`

```ruby
if @status.nil?
  process_status  # 新建状态
elsif @options[:delivered_to_account_id].present?
  postprocess_audience_and_deliver  # 状态已存在，但有新的投递目标
end

def postprocess_audience_and_deliver
  # 如果该账户还没有被提及，添加为静默提及
  return if @status.mentions.find_by(account_id: @options[:delivered_to_account_id])

  @status.mentions.create(account: delivered_to_account, silent: true)
  @status.update(visibility: :limited) if @status.direct_visibility?

  # 如果该本地账户关注了作者，插入到时间线
  return unless delivered_to_account.following?(@account)

  FeedInsertWorker.perform_async(@status.id, delivered_to_account.id, 'home')
end
```

---

## 四、完整链路示例分析

### 4.1 示例 1：受限作者 @ 非关注者（共享 inbox 路径

**前提**：
- Alice (instanceA, silenced) 发 direct 状态 @ Bob (instanceB)
- Bob **不是** Alice 的关注者
- instanceB 支持共享 inbox

**发送侧**：
1. `StatusReachFinder#mentioned_account_ids` → [Bob.id]
2. `StatusReachFinder#inboxes` → [instanceB/shared_inbox]
3. `TagManager#to` (silenced?) → **检查 Bob 是否是 Alice 的关注者 → 否 → **to = []**

**生成的 JSON**：
```json
{
  "to": [],  // ⚠️ 空！因为 Bob 不是 Alice 的关注者
  "cc": [],
  "object": { "content": "..." }
}
```

**接收侧 (instanceB)**：
1. 投递到 `POST /inbox` (共享 inbox)
2. `delivered_to_account_id` = `nil`
3. `addresses_local_accounts?`
   - 检查 `to`/`cc` → 空
   - **返回 false**
4. `related_to_local_activity?` → **false**
5. **拒绝处理** → reject_payload!**

**结果**：Bob **不会收到投递被拒绝，状态**不会**被处理

### 4.2 示例 2：受限作者 @ 关注者（共享 inbox 路径）

**前提**：
- Alice (instanceA, silenced) 发 direct 状态 @ Bob (instanceB)
- Bob **是** Alice 的关注者

**发送侧**：
3. `TagManager#to` (silenced?) → 检查 Bob 是否是 Alice 的关注者 → 是 → **to = [Bob.uri]**

**生成的 JSON**：
```json
{
  "to": ["https://instanceB.com/users/bob"],  // ✓ 包含 Bob
  "cc": [],
  "object": { "content": "..." }
}
```

**接收侧**：
3. `addresses_local_accounts?`
   - 检查 `to`/`cc` → `to` 包含 Bob.uri
   - **返回 true**
4. **处理状态**

**结果**：Bob **会收到**

### 4.3 示例 3：受限作者 @ 非关注者（用户 inbox 路径）

**前提**：
- Alice (instanceA, silenced) 发 direct 状态 @ Bob (instanceB)
- Bob **不是** Alice 的关注者
- **投递到用户 inbox**（`POST /users/bob/inbox`）

**发送侧**：
- `to` = []（和示例 1 相同）

**接收侧**：
1. 投递到 `POST /users/bob/inbox`
2. `@account` = Bob
3. `delivered_to_account_id` = Bob.id
4. `addresses_local_accounts?`
   - 有 `delivered_to_account_id`? → **直接返回 true**
5. 处理状态

**受众处理**：
```ruby
# process_audience
accounts_in_audience = []  # 从 to/cc 解析，to 为空

# ⚠️ 关键：添加 delivered_to_account
if @options[:delivered_to_account_id]
  accounts_in_audience << delivered_to_account  # 添加 Bob
end

# 创建静默提及
@mentions << Mention.new(account: Bob, silent: true)

# 原本是 direct，但有额外的静默提及 → 升级为 limited
@params[:visibility] = :limited
```

**结果**：
- Bob **会收到**
- 状态的可见性在 Bob 的实例上被**升级为 limited**
- Bob 被添加为**静默提及**

### 4.4 关键发现：Inbox 入口的安全网漏洞？

通过以上示例揭示了一个**微妙的设计：

| 投递路径 | 受限作者 @ 非关注者 | 实际行为 |
|---------|---------------------|----------|
| 共享 Inbox | ❌ 被拒绝 | **预期行为 |
| 用户 Inbox | ✓ 被处理，可见性升级 | **绕过了发送侧的过滤？

**这是设计意图吗？

让我们重新审视代码中的注释：

```ruby
# app/lib/activitypub/tag_manager.rb:180-181
if status.account.silenced?
  # Only notify followers if the account is locally silenced
```

**关键理解**：
- 这是**本地实例**的政策，只影响**本地生成**的 to/cc 字段
- 但**远端实例**的用户 inbox 投递是**远端实例**的政策

**两种保护机制**：
1. **发送侧保护**：通过 `to`/`cc` 字段过滤，让共享 inbox 路径被拒绝
2. **接收侧保护**：即使用户 inbox 路径绕过，状态会被**升级为 limited**，并且只有被添加为**静默提及**

---

## 五、发送侧 Inbox URL 选择机制

### 5.1 preferred_inbox_url

**文件**: `app/models/account.rb:404-406`

```ruby
def preferred_inbox_url
  shared_inbox_url.presence || inbox_url
end
```

**优先级**：
1. 优先使用 `shared_inbox_url`（共享 inbox）
2. 如果没有，使用 `inbox_url`（用户 inbox）

### 5.2 Account.inboxes

**文件**: `app/models/account.rb:419-422`

```ruby
def inboxes
  urls = reorder(nil).activitypub.group(:preferred_inbox_url).pluck(
    Arel.sql("coalesce(nullif(accounts.shared_inbox_url, ''), accounts.inbox_url) AS preferred_inbox_url")
  )
  DeliveryFailureTracker.without_unavailable(urls)
end
```

**关键点**：
- 按 `preferred_inbox_url` 分组
- 同一实例的多个账户会**合并到同一个共享 inbox

### 5.3 实际投递行为

| 远端账户配置 | 实际投递目标 | `delivered_to_account_id` |
|--------------|--------------|---------------------------|
| 有 shared_inbox_url | 共享 inbox | `nil` |
| 只有 inbox_url（用户 inbox） | 用户 inbox | 用户 ID |

**这意味着**：
- **支持共享 inbox** 的实例：受限作者的过滤**会被正确拒绝
- **不支持共享 inbox**（只支持用户 inbox）的实例：受限作者的过滤**会被绕过

---

## 六、完整链路总结表

### 6.1 受限作者 (Silenced) 的 direct/limited 状态行为

| 维度 | 非受限作者 | 受限作者 (silenced) |
|------|------------|---------------------|
| **投递目标 (inboxes) | 所有被@提及的账户 | 所有被@提及的账户（无条件） |
| **to 字段内容** | 所有被@提及的账户 URI | 只有"被@提及 + 是作者关注者"的账户 URI |
| **共享 inbox 接收** | ✓ 处理 | ✗ 被@但非关注者：拒绝 |
| **用户 inbox 接收** | ✓ 处理 | ✓ 处理，但可见性升级为 limited |

### 6.2 两种 Inbox 入口的行为对比

| 检查点 | 共享 Inbox (`/inbox`) | 用户 Inbox (`/users/:username/inbox`) |
|--------|---------------------|---------------------------------------|
| `delivered_to_account_id` | `nil` | 该用户 ID |
| `addresses_local_accounts?` | 检查 to/cc 字段 | **直接返回 true** |
| 依赖 to/cc 字段 | ✅ 是 | ❌ 否 |
| 受限作者过滤效果 | ✅ 生效 | ⚠️ 部分绕过（但有后续处理） |
| 可见性升级 | 不适用 | 可能从 direct 升级为 limited |

### 6.3 关键代码位置索引

| 功能 | 文件路径 | 关键方法/行号 |
|------|----------|---------------|
| silenced 受众过滤 | `app/lib/activitypub/tag_manager.rb` | `to` (173-196行) |
| 投递目标计算 | `app/lib/status_reach_finder.rb` | `inboxes` (12行) |
| 被@提及账户 | `app/lib/status_reach_finder.rb` | `mentioned_account_ids` (59行) |
| 路由配置 | `config/routes.rb` | 第 103-109, 146 行 |
| @account 设置 | `app/controllers/concerns/account_owned_concern.rb` | `set_account` (20-22行) |
| delivered_to_account_id 传递 | `app/controllers/activitypub/inboxes_controller.rb` | `process_payload` (75-77行) |
| 访问控制检查 | `app/lib/activitypub/activity/create.rb` | `addresses_local_accounts?` (454-458行) |
| 受众处理 | `app/lib/activitypub/activity/create.rb` | `process_audience` (124-169行) |
| 重复投递处理 | `app/lib/activitypub/activity/create.rb` | `postprocess_audience_and_deliver` (156-169行) |
| Inbox URL 选择 | `app/models/account.rb` | `preferred_inbox_url` (404-406行) |

---

## 七、设计意图与安全考量

### 7.1 为什么这样设计？

**受限作者 (Silenced) 的设计意图：

1. **本地实例控制**：
   - 受限用户仍可发布内容
   - 但内容**不会**出现在公共时间线
   - 但**可以**与已知的关注者通信

2. **通过关注者验证**：
   - `to` 字段只包含同时是关注者的被@提及账户
   - 这是一种"熟人社交"模式

3. **两层保护**：
   - **第一层**：`to`/`cc` 字段过滤（发送侧）
   - **第二层**：接收侧的 `addresses_local_accounts?` 检查

### 7.2 边界情况处理

| 边界情况 | 行为 |
|---------|------|
| 受限作者 @ 自己 | 总是包含（自己是自己的关注者吗？需要确认） |
| 受限作者 @ 已发送关注请求但未被批准 | 通过 FollowRequest 检查，会被包含 |
| 受限作者 @ 多个账户，部分是关注者 | 只有关注者会在 `to` 字段 |
| 共享 inbox 投递受限作者的消息 | 非关注者被拒绝 |
| 用户 inbox 投递受限作者的消息 | 被处理，但可见性升级 |

### 7.3 安全假设

**共享 inbox 是现代 ActivityPub 实现的标准配置，它：
1. 减少投递次数（同一实例只投递一次
2. 依赖 `to`/`cc` 字段来区分受众

用户 inbox 是较旧或较不常见，它：
1. 每个账户单独投递
2. 不依赖 `to`/`cc` 字段（因为投递目标明确

**设计权衡**：
- 共享 inbox + `to`/`cc` 过滤 = **高效且安全
- 用户 inbox + 可见性升级 = **兼容但有额外处理

---

## 八、完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              发送侧 (Instance A)                                        │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  状态: Alice (silenced?) 发 direct 状态 @ Bob                                               │
│                                                                                         │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ StatusReachFinder#inboxes                                                       │  │
│  │   ├─ reached_account_ids = [Bob.id] （无条件，不检查 silenced                   │  │
│  │   └─ inboxes_without_suspended_for → 只排除 suspended，不排除 silenced    │  │
│  │   → 投递目标: [Bob.instance.shared_inbox] 或 [Bob.user_inbox]                     │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                                  │
│                                      ▼                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ TagManager#to (direct/limited + silenced?)                                  │  │
│  │   ├─ account_ids = status.active_mentions.pluck(:account_id)                    │  │
│  │   └─ status.account.followers.where(id: account_ids)  ⚠️ 关键过滤           │  │
│  │                                                                              │  │
│  │   条件: Bob 是 Alice 的关注者?                                                  │  │
│  │   ├─ 是 → to = [Bob.uri]                                                    │  │
│  │   └─ 否 → to = []  ⚠️ 空!                                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                                  │
│                                      ▼                                                  │
│  投递到:                                                                                │
│    - 共享 inbox: POST https://instanceB.com/inbox                                      │
│    - 或用户 inbox: POST https://instanceB.com/users/bob/inbox                            │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      │  HTTP POST
                                      │  Content-Type: application/activity+json
                                      │  Signature: ...
                                      │  Body: { "to": [...], "object": {...} }
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              接收侧 (Instance B)                                        │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ InboxesController#create                                                      │  │
│  │                                                                              │  │
│  │ 路由决定 @account:                                                              │  │
│  │   ├─ 共享 inbox (/inbox): @account = nil                                    │  │
│  │   └─ 用户 inbox (/users/bob/inbox): @account = Bob                         │  │
│  │                                                                              │  │
│  │ delivered_to_account_id = @account&.id                                          │  │
│  │   ├─ 共享 inbox: nil                                                        │  │
│  │   └─ 用户 inbox: Bob.id                                                      │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                                  │
│                                      ▼                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ ActivityPub::Activity::Create#related_to_local_activity?                     │  │
│  │                                                                              │  │
│  │ addresses_local_accounts?                                                    │  │
│  │   ├─ 有 delivered_to_account_id?                                            │  │
│  │   │   ├─ 共享 inbox: 否 → 继续检查 to/cc                                    │  │
│  │   │   └─ 用户 inbox: 是 → ⚠️ 直接返回 true!                                 │  │
│  │   │                                                                          │  │
│  │   └─ 检查 to/cc 字段是否包含本地账户 URI                                    │  │
│  │       ├─ 是 → true ✓ 处理                                                      │  │
│  │       └─ 否 → false ✗ reject_payload!                                      │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                                  │
│              ┌───────────────────────┴───────────────────────┐                          │
│              ▼                                           ▼                          │
│  ┌──────────────────────┐              ┌──────────────────────────────┐              │
│  │ 共享 inbox 路径       │              │ 用户 inbox 路径                  │              │
│  │                      │              │                                │              │
│  │ to = [Bob.uri]       │              │ delivered_to_account_id = Bob.id  │              │
│  │ → 处理 ✓             │              │ → 直接通过 ✓                   │              │
│  │                      │              │                                │              │
│  │ to = []              │              │ 后续处理:                      │              │
│  │ → 拒绝 ✗             │              │   process_audience             │              │
│  │                      │              │   ├─ 添加 Bob 为静默提及      │              │
│  │                      │              │   └─ visible: direct → limited │              │
│  │                      │              │                                │              │
│  │                      │              │ postprocess_audience_and_deliver │              │
│  │                      │              │   (如果状态已存在)               │              │
│  └──────────────────────┘              └──────────────────────────────┘              │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```
