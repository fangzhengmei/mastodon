# Mastodon 有限可见状态联邦投递真实行为边界分析

## 重要纠偏声明

本文档纠偏了之前分析中的关键错误：

**`private`（仅关注者可见）与 `direct`/`limited`（私信/有限可见）的联邦行为完全不同。

---

## 一、五种可见性的完整行为对比表

| 可见性 | 枚举值 | to 字段 | cc 字段 | 关注者广播 | @提及投递 | LD-Signature | HTTP-Signature |
|--------|--------|---------|---------|-----------|-----------|--------------|-----------------|
| **public** | 0 | `[as:Public]` | followers_uri + 提及 | ✓ | ✓ | ✓ | ✓ |
| **unlisted** | 1 | `[followers_uri]` | as:Public + 提及 | ✓ | ✓ | ✓ | ✓ |
| **private** | 2 | `[followers_uri]` | 仅提及 | ✓ | ✓ | ✗ | ✓ |
| **direct** | 3 | `[提及账户URI]` | 空 | ✗ | ✓ | ✗ | ✓ |
| **limited** | 4 | `[提及账户URI]` | 空 | ✗ | ✓ | ✗ | ✓ |

**关键区分**：
- **`private`** = 仅关注者可见，但**仍会联邦**到所有关注者所在实例
- **`direct`/`limited`** = 不会广播给关注者，只投递到被@提及的账户

---

## 二、发送侧：Inbox 目标集合判定完整路径

### 2.1 核心判定逻辑

**文件**: `app/lib/status_reach_finder.rb:12-117`

```ruby
def inboxes
  (reached_account_inboxes + followers_inboxes + relay_inboxes).uniq
end
```

目标收件箱 = 涉及账户收件箱 ∪ 关注者收件箱 ∪ 中继收件箱

### 2.2 分路径详细分析

#### 路径 1：reached_account_inboxes（涉及账户）

```ruby
def reached_account_ids
  if @status.reblog?
    [reblog_of_account_id]
  else
    [
      replied_to_account_id,      # ⚠️ 有条件：仅 distributable?
      reblog_of_account_id,
      quote_of_account_id,
      mentioned_account_ids,    # ✓ 无条件！始终包含
      reblogs_account_ids,      # 有条件
      quotes_account_ids,       # 有条件
      favourites_account_ids,   # 有条件
      replies_account_ids,      # 有条件
    ].tap { |arr| arr.flatten!; arr.compact!; arr.uniq! }
  end
end

def mentioned_account_ids
  @status.mentions.pluck(:account_id)  # 没有任何条件判断！
end

def distributable?
  @status.public_visibility? || @status.unlisted_visibility?
end
```

**关键发现**：
- `mentioned_account_ids` **无条件**返回所有被@提及的账户 ID
- `replied_to_account_id` 等只有 `distributable?（即 public/unlisted）才返回
- 这意味着：**即使是私信/有限可见状态，被@提及的账户仍然会被投递！

#### 路径 2：followers_inboxes（关注者）

```ruby
def followers_scope
  if @status.in_reply_to_local_account? && distributable?
    # 回复本地账户的公开状态：作者关注者 + 原作者关注者
    @status.account.followers.or(@status.thread.account.followers.not_domain_blocked_by_account(@status.account))
  elsif @status.direct_visibility? || @status.limited_visibility?
    Account.none  # ⚠️ 私信/有限可见：关注者集合为空！
  else
    @status.account.followers  # private/public/unlisted：所有关注者
  end
end
```

**关键发现**：
- `direct`/`limited` → `Account.none` = **完全没有关注者广播**
- `private` → `@status.account.followers` = **所有关注者都会被投递**

#### 路径 3：relay_inboxes（中继）

```ruby
def relay_inboxes
  if @status.public_visibility?
    Relay.enabled.pluck(:inbox_url)
  else
    []  # 只有 public 状态会投递给中继
  end
end
```

### 2.3 不同可见性的实际投递目标

| 可见性 | 关注者广播 | 被@提及投递 | 中继投递 |
|--------|-------------|---------------|---------|
| **public** | ✓ 所有关注者 | ✓ | ✓ |
| **unlisted** | ✓ 所有关注者 | ✓ | ✗ |
| **private** | ✓ 所有关注者 | ✓ | ✗ |
| **direct** | ✗ 无 | ✓ 仅被@的人 | ✗ |
| **limited** | ✗ 无 | ✓ 仅被@的人 | ✗ |

### 2.4 ActivityPub to/cc 字段生成

**文件**: `app/lib/activitypub/tag_manager.rb:173-238`

```ruby
def to(status)
  case status.visibility
  when 'public'
    [COLLECTIONS[:public]]  # "https://www.w3.org/ns/activitystreams#Public"
  when 'unlisted', 'private'
    [followers_uri_for(status.account)]  # 关注者集合 URI
  when 'direct', 'limited'
    # 仅被@提及的账户 URI
    status.active_mentions.each_with_object([]) do |mention, result|
      result << uri_for(mention.account)
      result << followers_uri_for(mention.account) if mention.account.group?
    end.compact
  end
end

def cc(status)
  cc = []
  cc << uri_for(status.reblog.account) if status.reblog?

  case status.visibility
  when 'public'
    cc << followers_uri_for(status.account)
  when 'unlisted'
    cc << COLLECTIONS[:public]
  end

  # ⚠️ direct/limited 不会在这里被排除！
  unless status.direct_visibility? || status.limited_visibility?
    # public/unlisted/private 会添加被@提及的账户到 cc
    cc.concat(status.active_mentions.each_with_object([]) do |mention, result|
      result << uri_for(mention.account)
      result << followers_uri_for(mention.account) if mention.account.group?
    end.compact)
  end

  cc
end
```

**JSON 示例对比：

**public 状态**：
```json
{
  "to": ["https://www.w3.org/ns/activitystreams#Public"],
  "cc": ["https://example.com/users/alice/followers", "https://other.com/users/bob"]
}
```

**private 状态**：
```json
{
  "to": ["https://example.com/users/alice/followers"],
  "cc": ["https://other.com/users/bob"]  // 被@提及的人
}
```

**direct 状态**：
```json
{
  "to": ["https://other.com/users/bob"],  // 仅被@提及的人
  "cc": []  // 空！
}
```

---

## 三、签名机制的可见性差异

### 3.1 签名判定逻辑

**文件**: `app/models/concerns/status/visibility.rb:31-35`

```ruby
def distributable?
  public_visibility? || unlisted_visibility?
end

alias sign? distributable?
```

**文件**: `app/services/concerns/payloadable.rb:13-29`

```ruby
def serialize_payload(record, serializer, options = {})
  # ...
  if object.respond_to?(:sign?) && object.sign? && signer && (always_sign || signing_enabled?)
    ActivityPub::LinkedDataSignature.new(payload).sign!(signer, sign_with: sign_with)
  else
    payload
  end
end
```

### 3.2 签名类型与可见性

| 可见性 | `sign?` 返回值 | Linked Data Signature | HTTP Signature |
|--------|-----------------|----------------------|-----------------|
| **public** | `true` | ✓ 嵌入 JSON | ✓ 请求头 |
| **unlisted** | `true` | ✓ 嵌入 JSON | ✓ 请求头 |
| **private** | `false` | ✗ | ✓ 请求头 |
| **direct** | `false` | ✗ | ✓ 请求头 |
| **limited** | `false` | ✗ | ✓ 请求头 |

**关键意义**：
- **HTTP Signatures**：所有投递都有，用于验证发送者身份
- **Linked Data Signatures**：仅 `public`/`unlisted` 才有，允许活动被中继/转发时仍可验证

**投递层签名（HTTP Signatures）：

**文件**: `app/lib/request.rb:100-108, 131`

```ruby
def on_behalf_of(actor, sign_with: nil)
  key_id = ActivityPub::TagManager.instance.key_uri_for(actor)
  keypair = sign_with.present? ? OpenSSL::PKey::RSA.new(sign_with) : actor.keypair
  @signing = HttpSignatureDraft.new(keypair, key_id)
  self
end

def headers
  (@signing ? @headers.merge('Signature' => signature) : @headers)
end
```

**文件**: `app/workers/activitypub/delivery_worker.rb:49-55`

```ruby
def build_request(http_client)
  Request.new(:post, @inbox_url, body: @json, http_client: http_client).tap do |request|
    request.on_behalf_of(@source_account, sign_with: @options[:sign_with])
    request.add_headers(HEADERS)
  end
end
```

---

## 四、接收侧：有限可见状态的完整处理流程

### 4.1 入口点与参数传递

**文件**: `app/controllers/activitypub/inboxes_controller.rb:35-37, 75-77`

```ruby
def account_required?
  params[:account_username].present?
end

def process_payload
  # @account&.id 就是 delivered_to_account_id
  ActivityPub::ProcessingWorker.perform_async(signed_request_actor.id, body, @account&.id, signed_request_actor.class.name)
end
```

**路由差异**：

| 路由 | `@account` 值 | `delivered_to_account_id` |
|------|----------------|---------------------------|
| `POST /inbox` | `nil` | `nil` |
| `POST /users/:username/inbox` | 该用户 Account | 用户 ID |

### 4.2 访问控制检查

**文件**: `app/lib/activitypub/activity/create.rb:17-19, 445-458`

```ruby
def create_status
  # 关键检查！如果不通过，直接 reject_payload!
  return reject_payload! if unsupported_object_type? || non_matching_uri_hosts?(@account.uri, object_uri) || tombstone_exists? || !related_to_local_activity?
  # ...
end

def related_to_local_activity?
  fetch? || followed_by_local_accounts? || requested_through_relay? ||
    responds_to_followed_account? || addresses_local_accounts?
end

def addresses_local_accounts?
  # ⚠️ 如果是关键！
  return true if @options[:delivered_to_account_id]

  # 否则检查 to/cc 是否包含本地账户 URI
  ActivityPub::TagManager.instance.uris_to_local_accounts((audience_to + audience_cc).uniq).exists?
end
```

**关键逻辑解读**：

1. **如果是直接投递到用户 inbox**（`delivered_to_account_id` 存在）：
   - `addresses_local_accounts?` 直接返回 `true`
   - 活动**会被处理

2. **如果是共享 inbox 投递**：
   - 需要检查 `to`/`cc` 字段是否包含任何本地账户 URI
   - 对于 `direct` 状态，`to` 包含被@提及的账户 URI
   - 如果本地账户在 `to` 中，活动会被处理

### 4.3 受众处理逻辑

**文件**: `app/lib/activitypub/activity/create.rb:124-169`

```ruby
def process_audience
  # 1. 从 to/cc 解析已知的本地账户
  accounts_in_audience = (audience_to + audience_cc).uniq.filter_map do |audience|
    account_from_uri(audience) unless ActivityPub::TagManager.instance.public_collection?(audience)
  end

  # 2. ⚠️ 关键：如果是直接投递到某用户的 inbox，该用户必须被添加到受众
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

  # 计算被静默的账户（被 tag 但不在受众中）
  @silenced_account_ids = @mentions.filter_map { |mention| mention.account_id if mention.account.local? } - accounts_in_audience.map(&:id)
end
```

### 4.4 重复处理：状态已存在但投递到新账户

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

### 4.5 接收侧处理流程图

```
收到 ActivityPub 投递
        │
        ▼
┌───────────────────────┐
│ InboxesController     │
│ 验证 HTTP Signature    │
│ 获取 @account         │
│ （来自路由参数）     │
└───────────┬───────────┘
            │
            ▼ delivered_to_account_id = @account&.id
┌───────────────────────┐
│ ProcessingWorker  │
│ 异步处理活动        │
└───────────┬───────────┘
            │
            ▼
┌────────────────────────────────────────────────────────┐
│ ProcessCollectionService                             │
│ 解析 JSON-LD                                         │
│ 验证 LD-Signature（如果有）                           │
│ 根据 activity type 分发                                │
└───────────────────┬────────────────────────────────────┘
                    │ type: "Create"
                    ▼
┌────────────────────────────────────────────────────────┐
│ ActivityPub::Activity::Create                        │
│                                                        │
│ 1. related_to_local_activity? 检查                     │
│    ├─ fetch?（非投递场景）                              │
│    ├─ followed_by_local_accounts?                      │
│    ├─ requested_through_relay?                         │
│    ├─ responds_to_followed_account?                     │
│    └─ addresses_local_accounts?                        │
│       ├─ 有 delivered_to_account_id? → true            │
│       └─ 否则检查 to/cc 是否有本地账户               │
│                                                        │
│ 2. process_audience                                   │
│    ├─ 从 to/cc 解析受众                              │
│    ├─ 有 delivered_to_account_id? → 添加到受众     │
│    └─ 创建静默提及                                      │
│    └─ 有额外受众？direct → limited                  │
│                                                        │
│ 3. 新建或更新状态                                      │
└────────────────────────────────────────────────────────┘
```

---

## 五、典型场景行为边界分析

### 场景 1：Alice (instanceA) 发私信 @Bob (instanceB)

**发送侧判定**：
- 可见性：`direct`
- `followers_scope` → `Account.none`（无关注者广播）
- `mentioned_account_ids` → [Bob.id]
- **实际投递**：仅 Bob 所在的 instanceB

**生成的 JSON**：
```json
{
  "to": ["https://instanceB.com/users/bob"],
  "cc": [],
  "object": { "content": "私内容" }
}
```

**接收侧（instanceB）**：
- 投递到 `/users/bob/inbox`
- `delivered_to_account_id` = Bob.id
- `addresses_local_accounts?` → true
- 处理：创建状态，Bob 作为静默提及（或非静默如果在 tag 中）

---

### 场景 2：Alice (instanceA) 发 private 状态（仅关注者可见）

**发送侧判定**：
- 可见性：`private`
- `followers_scope` → Alice.followers（所有关注者）
- `mentioned_account_ids` → 被@的账户（如有）
- **实际投递**：所有关注者所在实例 + 被@提及的账户

**生成的 JSON**：
```json
{
  "to": ["https://instanceA.com/users/alice/followers"],
  "cc": ["https://other.com/users/mentioned"],  // 如果有 @ 提及
  "object": { "content": "仅关注者可见的内容" }
}
```

**关键区别**：`private` 状态**会**联邦到所有关注者的实例！

---

### 场景 3：Alice (instanceA) 发 private 状态 @Bob (instanceB)，Charlie (instanceC) 关注 Alice

**发送侧**：
- `followers_inboxes` → [instanceB, instanceC, ...]（所有关注者的实例）
- `reached_account_inboxes` → [instanceB]（Bob 被@提及）
- **合并后**：instanceB, instanceC, ...

**instanceC 接收侧**：
- Charlie 是 Alice 的关注者
- 投递到 Charlie 的 inbox 或共享 inbox
- 状态会被处理并插入到 Charlie 的时间线

---

### 场景 4：Alice (instanceA) 发 direct 状态 @Bob (instanceB)，Charlie (instanceC) 关注 Alice

**发送侧**：
- `followers_inboxes` → `[]`（空！）
- `reached_account_inboxes` → [instanceB]（仅 Bob）
- **实际投递**：仅 instanceB

**Charlie 的情况**：
- ❌ 完全不会收到投递
- ❌ 不会知道这条状态的存在

---

### 场景 5：共享 inbox 投递 direct 状态

**发送侧**：投递到 `https://instanceB.com/inbox`（共享 inbox）
- `delivered_to_account_id` = `nil`

**接收侧**：
- `addresses_local_accounts?` 检查 `to`/`cc`
- `to` = [Bob.uri]
- 如果 Bob 是本地账户 → 处理

---

## 六、完整链路对比总结

### 6.1 不会走关注者广播的场景

| 可见性 | 关注者集合 | 是否广播给关注者 |
|--------|-----------|----------------|
| **direct** | `Account.none` | ❌ 否 |
| **limited** | `Account.none` | ❌ 否 |
| **private** | `account.followers | ✓ 是 |
| **public** | `account.followers` | ✓ 是 |
| **unlisted** | `account.followers` | ✓ 是 |

### 6.2 仍会定向投递的场景

**无论可见性如何，以下账户都会被投递**：

1. **被@提及的账户**（`mentioned_account_ids` - 无条件）
2. **被回复的原作者**（仅 `public`/`unlisted`）
3. **public 状态的所有关注者**
4. **private 状态的所有关注者**（关键！容易被误解）

### 6.3 关键代码位置索引

| 功能 | 文件路径 | 关键方法/行号 |
|------|----------|---------------|
| 目标收件箱计算 | `app/lib/status_reach_finder.rb` | `inboxes` (12行) |
| 关注者范围判定 | `app/lib/status_reach_finder.rb` | `followers_scope` (104行) |
| 被@提及账户 | `app/lib/status_reach_finder.rb` | `mentioned_account_ids` (59行) |
| to/cc 生成 | `app/lib/activitypub/tag_manager.rb` | `to` (173行), `cc` (205行) |
| 签名判定 | `app/models/concerns/status/visibility.rb` | `sign?` (35行) |
| 接收侧访问控制 | `app/lib/activitypub/activity/create.rb` | `related_to_local_activity?` (445行) |
| 接收侧受众处理 | `app/lib/activitypub/activity/create.rb` | `process_audience` (124行) |
| 重复投递处理 | `app/lib/activitypub/activity/create.rb` | `postprocess_audience_and_deliver` (156行) |

---

## 七、常见误解与澄清

### 误解 1："private 状态不会联邦"

**❌ 错误理解**：`private` = 不联邦，只在本地可见

**✓ 正确理解**：
- `private` 会**联邦到所有关注者所在的实例
- 区别在于：
  - `to` = 是 `followers_uri`，不是 `as:Public`
  - 不会出现在公共时间线
  - 不会投递给中继
  - 没有 Linked Data Signature

### 误解 2："direct 状态只在 to/cc 控制投递

**❌ 错误理解**：`direct` 的投递完全由 `to`/`cc` 字段决定

**✓ 正确理解**：
- 投递目标由 `StatusReachFinder` 决定
- `mentioned_account_ids` 是**无条件**的
- 即使 `to`/`cc` 为空，只要有 `@` 提及就会投递
- `to`/`cc` 影响的是**接收侧**的受众解析，不影响**发送侧**的投递目标

### 误解 3："有限可见状态无法联邦"

**❌ 错误理解**：`limited` = 完全不联邦

**✓ 正确理解**：
- `limited` **不会**广播给所有关注者
- 但**会**投递给被@提及的账户
- 行为与 `direct` 相同

### 关键区分速记

```
public    = 完全公开（公共时间线 + 关注者 + 中继
unlisted  = 半公开（关注者 + as:Public在cc，但不推广）
private   = 仅关注者（关注者联邦，但无公共访问）
direct    = 私信（仅被@的人，无关注者广播）
limited   = 有限可见（同 direct，用于更灵活的受众）
```
