# Mastodon 账号迁移机制完整分析（角色边界版）

## 目录

1. [概述](#概述)
2. [角色边界与职责划分](#角色边界与职责划分)
3. [前端引导流程与失败反馈闭环](#前端引导流程与失败反馈闭环)
4. [旧实例：发起迁移与分发 Move 信号](#旧实例发起迁移与分发-move-信号)
5. [新实例：维护别名与验证认领](#新实例维护别名与验证认领)
6. [关注者实例：协议入口链路与 Move 处理](#关注者实例协议入口链路与-move-处理)
   - [6.1 活动接收位置：InboxesController](#61-活动接收位置inboxescontroller)
   - [6.2 签名门禁：SignatureVerification](#62-签名门禁signatureverification)
   - [6.3 异步处理队列：ProcessingWorker](#63-异步处理队列processingworker)
   - [6.4 Different Actor 校验：ProcessCollectionService](#64-different-actor-校验processcollectionservice)
   - [6.5 业务层验证：Activity::Move](#65-业务层验证activitymove)
   - [6.6 MoveWorker：关注关系转移执行](#66-moveworker关注关系转移执行)
7. [失败发生层级 vs 用户可见性对照表](#失败发生层级-vs-用户可见性对照表)
8. [三方协同时序总结](#三方协同时序总结)
9. [关键数据结构](#关键数据结构)
10. [安全机制与错误处理](#安全机制与错误处理)
11. [相关文件索引](#相关文件索引)

---

## 概述

Mastodon 的账号迁移是基于 ActivityPub 协议的 `Move` 活动实现的分布式协作流程。整个流程涉及**三个核心角色**的紧密配合：

| 角色 | 定义 | 核心职责 |
|------|------|----------|
| **旧实例** | 源账号所在的实例 | 发起迁移请求、验证前置条件、分发 Move 活动、处理本地关注关系 |
| **新实例** | 目标账号所在的实例 | 维护 `also_known_as` 别名、通过 Update 活动宣告认领关系、被动验证 |
| **关注者实例** | 关注者所在的实例 | 接收 Move 活动、执行协议验证、重写本地关注关系 |

**核心安全机制：双向引用验证**

```
新账号 ──alsoKnownAs──► 旧账号 URI (主动认领)
旧账号 ──movedTo───────► 新账号 URI (被动指向)
```

只有当双向引用同时满足时，迁移才会被执行。

---

## 角色边界与职责划分

### 1. 旧实例（Source Instance）

#### 1.1 核心职责

| 职责类型 | 具体动作 | 代码位置 |
|----------|----------|----------|
| **用户交互** | 提供迁移表单页面 | `app/views/settings/migrations/show.html.haml` |
| **请求接收** | 处理迁移 POST 请求 | `app/controllers/settings/migrations_controller.rb` |
| **前置验证** | 验证密码/用户名、目标账号存在性、冷却期、also_known_as | `app/models/account_migration.rb` |
| **状态更新** | 设置 `moved_to_account` 重定向 | `app/services/move_service.rb:17-19` |
| **活动分发** | 分发 Update 活动（宣告迁移） | `app/services/move_service.rb:25-27` |
| **活动分发** | 分发 Move 活动（触发 follower 转移） | `app/services/move_service.rb:29-31` |
| **本地处理** | 处理同实例内的关注关系转移 | `app/workers/move_worker.rb:10-16` |

#### 1.2 不负责的事项

- ❌ 不验证新实例的真实意愿（通过 also_known_as 间接验证）
- ❌ 不处理跨实例的关注关系转移（由关注者实例处理）
- ❌ 不确保所有关注者都成功转移（异步、分布式、最终一致）

---

### 2. 新实例（Target Instance）

#### 2.1 核心职责

| 职责类型 | 具体动作 | 代码位置 |
|----------|----------|----------|
| **用户交互** | 提供别名管理页面 | `app/views/settings/aliases/index.html.haml` |
| **请求接收** | 处理别名添加/删除请求 | `app/controllers/settings/aliases_controller.rb` |
| **别名验证** | 验证目标账号存在、不是自己 | `app/models/account_alias.rb:50-56` |
| **状态更新** | 更新 `also_known_as` 数组 | `app/models/account_alias.rb:42-48` |
| **活动分发** | 分发 Update 活动（宣告别名关系） | `app/controllers/settings/aliases_controller.rb:17-18` |

#### 2.2 不负责的事项

- ❌ 不主动发起迁移（仅被动认领）
- ❌ 不处理关注关系转移（由关注者实例处理）
- ❌ 不知道哪些关注者会转移（分布式、无全局视图）

---

### 3. 关注者实例（Follower Instance）

#### 3.1 核心职责

| 职责类型 | 具体动作 | 代码位置 |
|----------|----------|----------|
| **活动接收** | 接收并解析 Move 活动 | `app/lib/activitypub/activity/move.rb` |
| **协议验证** | 验证 object==actor、7天冷却期、also_known_as 双向引用 | `app/lib/activitypub/activity/move.rb:6-25` |
| **状态更新** | 更新本地缓存的 `moved_to_account` | `app/lib/activitypub/activity/move.rb:18` |
| **关系转移** | 重写本地关注关系 | `app/workers/move_worker.rb` |
| **数据迁移** | 迁移列表成员、备注、屏蔽、静音 | `app/workers/move_worker.rb:96-156` |

#### 3.2 不负责的事项

- ❌ 不验证用户身份（仅验证协议签名和内容）
- ❌ 不回滚失败的迁移（原子性由单条操作保证）
- ❌ 不通知源实例转移结果（单向、最终一致）

---

### 4. 角色边界可视化

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           账号迁移角色边界图                                        │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│                              新实例 (Target Instance)                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐          │
│   │   用户操作层     │────►│   控制器层       │────►│    模型层        │          │
│   │  /settings/     │     │  Aliases-       │     │  AccountAlias   │          │
│   │    aliases      │     │  Controller     │     │                 │          │
│   └─────────────────┘     └─────────────────┘     └─────────────────┘          │
│                                                        │                          │
│                                                        ▼                          │
│   ┌─────────────────────────────────────────────────────────────────────┐        │
│   │                    新实例专属职责边界                                  │        │
│   ├─────────────────────────────────────────────────────────────────────┤        │
│   │ ✓ 维护 also_known_as 数组                                            │        │
│   │ ✓ 通过 AccountAlias 模型验证别名合法性                                │        │
│   │ ✓ 分发 Update 活动宣告别名关系                                        │        │
│   │ ✓ Actor 文档中包含 alsoKnownAs 字段                                   │        │
│   │                                                                       │        │
│   │ ✗ 不发起迁移（仅被动认领）                                            │        │
│   │ ✗ 不处理关注关系转移                                                  │        │
│   │ ✗ 不知道哪些关注者会转移                                              │        │
│   └─────────────────────────────────────────────────────────────────────┘        │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      ▲
                                      │ Update 活动 (alsoKnownAs)
                                      │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              旧实例 (Source Instance)                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐          │
│   │   用户操作层     │────►│   控制器层       │────►│    模型层        │          │
│   │  /settings/     │     │ Migrations-     │     │AccountMigration │          │
│   │   migration     │     │  Controller     │     │                 │          │
│   └─────────────────┘     └─────────────────┘     └─────────────────┘          │
│                                                        │                          │
│                                                        ▼                          │
│   ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐          │
│   │   分发层         │◄────│    服务层       │◄────│    验证层        │          │
│   │ MoveDistribution│     │  MoveService    │     │  模型验证         │          │
│   │ Worker          │     │                 │     │                 │          │
│   └─────────────────┘     └─────────────────┘     └─────────────────┘          │
│                                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐        │
│   │                    旧实例专属职责边界                                  │        │
│   ├─────────────────────────────────────────────────────────────────────┤        │
│   │ ✓ 提供迁移表单和用户交互                                              │        │
│   │ ✓ 验证密码/用户名身份                                                 │        │
│   │ ✓ 验证目标账号 also_known_as 包含源账号 URI                          │        │
│   │ ✓ 验证 30 天冷却期                                                   │        │
│   │ ✓ 设置 moved_to_account 重定向                                       │        │
│   │ ✓ 分发 Update 活动宣告迁移                                            │        │
│   │ ✓ 分发 Move 活动触发 follower 转移                                    │        │
│   │ ✓ 处理同实例内的关注关系转移                                          │        │
│   │                                                                       │        │
│   │ ✗ 不验证新实例的"真实"意愿（通过 also_known_as 间接验证）             │        │
│   │ ✗ 不处理跨实例的关注关系转移（由关注者实例处理）                       │        │
│   │ ✗ 不确保所有关注者都成功转移（异步、分布式、最终一致）                  │        │
│   └─────────────────────────────────────────────────────────────────────┘        │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      │ Move 活动 (actor, object, target)
                                      │ Update 活动 (movedTo)
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           关注者实例 (Follower Instance)                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐          │
│   │   协议层         │────►│   验证层         │────►│    执行层        │          │
│   │  ActivityPub    │     │  Activity::Move │     │   MoveWorker    │          │
│   │  Inbox 端点      │     │                 │     │                 │          │
│   └─────────────────┘     └─────────────────┘     └─────────────────┘          │
│                                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐        │
│   │                   关注者实例专属职责边界                               │        │
│   ├─────────────────────────────────────────────────────────────────────┤        │
│   │ ✓ 接收并解析 Move 活动                                               │        │
│   │ ✓ 验证 object == actor（防止假冒迁移）                               │        │
│   │ ✓ 验证 7 天处理冷却期                                                │        │
│   │ ✓ 验证目标账号 also_known_as 包含源账号 URI（核心验证）              │        │
│   │ ✓ 更新本地缓存的 moved_to_account                                    │        │
│   │ ✓ 重写本地关注关系（旧→新）                                          │        │
│   │ ✓ 迁移列表成员、账号备注、屏蔽、静音关系                               │        │
│   │                                                                       │        │
│   │ ✗ 不验证用户身份（仅验证协议签名和内容）                              │        │
│   │ ✗ 不回滚失败的迁移（原子性由单条操作保证）                            │        │
│   │ ✗ 不通知源实例转移结果（单向、最终一致）                              │        │
│   └─────────────────────────────────────────────────────────────────────┘        │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 前端引导流程与失败反馈闭环

### 1. 第一阶段：新账号添加别名（新实例）

#### 1.1 成功路径

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        新实例添加别名成功流程                                   │
└─────────────────────────────────────────────────────────────────────────────┘

用户操作                          前端反馈                          后端处理
─────────                        ─────────                        ─────────
     │                                │                                │
     │  1. 访问 /settings/aliases     │                                │
     │ ───────────────────────────►  │  显示别名列表和添加表单           │
     │                                │                                │
     │                                │                                │
     │  2. 输入旧账号地址              │                                │
     │     (如: olduser@old.com)     │                                │
     │ ───────────────────────────►  │                                │
     │                                │                                │
     │                                │  3. 提交表单                    │
     │ ◄───────────────────────────  │ ───────────────────────────►  │
     │                                │                                │
     │                                │                                │  4. AccountAlias 验证
     │                                │                                │     - WebFinger 解析
     │                                │                                │     - 目标账号存在?
     │                                │                                │     - 不是自己?
     │                                │                                │
     │                                │                                │  5. 更新 also_known_as
     │                                │                                │  6. 分发 Update 活动
     │                                │                                │
     │  7. 显示成功消息               │                                │
     │    "Alias created successfully"│                                │
     │ ◄───────────────────────────  │ ◄───────────────────────────  │
     │                                │                                │
     ▼                                ▼                                ▼
```

#### 1.2 失败路径与反馈闭环

| 失败场景 | 触发条件 | 前端反馈 | 代码位置 | 用户修复建议 |
|----------|----------|----------|----------|--------------|
| **账号不存在** | WebFinger 解析失败 | `could not be found` | `app/models/account_alias.rb:51-52` | 检查账号地址拼写、确认目标实例可访问 |
| **指向自己** | 别名是当前账号 | `cannot be current account` | `app/models/account_alias.rb:53-54` | 确保输入的是旧账号，不是新账号 |
| **网络错误** | 目标实例无法连接 | `could not be found` | `app/models/account_alias.rb:36-40` | 稍后重试、检查网络连接、确认目标实例在线 |
| **重复添加** | 别名已存在 | 数据库唯一约束错误 | `app/models/account_alias.rb:19` | 该别名已添加，无需重复操作 |

**错误处理流程**：
```ruby
# app/models/account_alias.rb:35-40
def set_uri
  target_account = ResolveAccountService.new.call(acct)
  self.uri = ActivityPub::TagManager.instance.uri_for(target_account) unless target_account.nil?
rescue Webfinger::Error, *Mastodon::HTTP_CONNECTION_ERRORS, Mastodon::Error
  # 异常被捕获，由后续验证逻辑添加错误信息
end

# app/models/account_alias.rb:50-56
def validate_target_account
  if uri.blank?
    errors.add(:acct, I18n.t('migrations.errors.not_found'))  # "could not be found"
  elsif ActivityPub::TagManager.instance.uri_for(account) == uri
    errors.add(:acct, I18n.t('migrations.errors.move_to_self'))  # "cannot be current account"
  end
end
```

---

### 2. 第二阶段：旧账号发起迁移（旧实例）

#### 2.1 成功路径

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        旧实例发起迁移成功流程                                   │
└─────────────────────────────────────────────────────────────────────────────┘

用户操作                          前端反馈                          后端处理
─────────                        ─────────                        ─────────
     │                                │                                │
     │  1. 访问 /settings/migration   │                                │
     │ ───────────────────────────►  │                                │
     │                                │  2. 检查状态                    │
     │                                │     - 是否已设置重定向?         │
     │                                │     - 是否在冷却期?             │
     │                                │                                │
     │                                │  3. 显示表单                    │
     │                                │     - 目标账号输入框            │
     │                                │     - 密码/用户名输入框         │
     │                                │     - 风险提示列表              │
     │ ◄───────────────────────────  │                                │
     │                                │                                │
     │                                │                                │
     │  4. 输入新账号地址和密码        │                                │
     │ ───────────────────────────►  │                                │
     │                                │                                │
     │                                │  5. 提交表单                    │
     │ ◄───────────────────────────  │ ───────────────────────────►  │
     │                                │                                │
     │                                │                                │  6. AccountMigration 验证
     │                                │                                │     - 密码/用户名正确?
     │                                │                                │     - 目标账号存在?
     │                                │                                │     - also_known_as 包含源账号?
     │                                │                                │     - 30天冷却期?
     │                                │                                │     - 不是迁移到自己?
     │                                │                                │
     │                                │                                │  7. MoveService 执行
     │                                │                                │     - 设置 moved_to_account
     │                                │                                │     - 异步处理本地关注
     │                                │                                │     - 分发 Update + Move 活动
     │                                │                                │
     │  8. 显示成功消息               │                                │
     │    "Your account is now       │                                │
     │     redirecting to ..."       │                                │
     │ ◄───────────────────────────  │ ◄───────────────────────────  │
     │                                │                                │
     ▼                                ▼                                ▼
```

#### 2.2 失败路径与反馈闭环

| 失败场景 | 触发条件 | 前端反馈 | 代码位置 | 用户修复建议 |
|----------|----------|----------|----------|--------------|
| **密码错误** | 密码验证失败 | 密码字段显示错误 | `app/models/account_migration.rb:46-47` | 检查密码是否正确 |
| **用户名错误** | OAuth 用户用户名验证失败 | 用户名字段显示错误 | `app/models/account_migration.rb:48-50` | 确认当前账号的用户名 |
| **目标账号不存在** | WebFinger 解析失败 | `could not be found` | `app/models/account_migration.rb:80-81` | 检查新账号地址、确认新实例可访问 |
| **缺少 also_known_as** | 新账号未添加旧账号为别名 | `is not an alias of this account` | `app/models/account_migration.rb:82-83` | **关键错误**：需要先在新账号添加旧账号为别名 |
| **已迁移过** | 已迁移到同一个账号 | `is the same account you have already moved to` | `app/models/account_migration.rb:84-85` | 无需重复操作 |
| **迁移到自己** | 目标账号是当前账号 | `cannot be current account` | `app/models/account_migration.rb:86` | 确保输入的是新账号地址 |
| **冷却期** | 30天内已迁移过 | `You are on cooldown. Available again in X days` | `app/models/account_migration.rb:89-91` + 视图 | 等待冷却期结束，页面会显示剩余天数 |

**核心验证逻辑**（`app/models/account_migration.rb`）：
```ruby
def validate_target_account
  if target_account.nil?
    errors.add(:acct, I18n.t('migrations.errors.not_found'))
  else
    # 核心验证：新账号的 also_known_as 必须包含旧账号 URI
    errors.add(:acct, I18n.t('migrations.errors.missing_also_known_as')) 
      unless target_account.also_known_as.include?(ActivityPub::TagManager.instance.uri_for(account))
    
    errors.add(:acct, I18n.t('migrations.errors.already_moved')) 
      if account.moved? && account.moved_to_account_id == target_account.id
    
    errors.add(:acct, I18n.t('migrations.errors.move_to_self')) 
      if account.id == target_account.id
  end
end
```

**前端预提示机制**（`app/views/settings/migrations/show.html.haml`）：

```haml
- unless on_cooldown?
  %p.hint= t('migrations.warning.before')  # "Before proceeding, please read these notes carefully:"

  %ul.hint
    %li.warning-hint= t('migrations.warning.followers')      # 粉丝会被转移
    %li.warning-hint= t('migrations.warning.other_data')     # 其他数据不会自动转移
    %li.warning-hint= t('migrations.warning.redirect')        # 个人资料会显示重定向
    %li.warning-hint= t('migrations.warning.backreference_required')  # 关键：新账号必须先配置反向引用
    %li.warning-hint= t('migrations.warning.cooldown')       # 迁移后有等待期
    %li.warning-hint= t('migrations.warning.disabled_account') # 当前账号之后不可完全使用
```

**关键提示翻译**：
- `backreference_required`: "The new account must first be configured to back-reference this one"
- 含义：新账号必须先配置反向引用（即添加旧账号为别名）

#### 2.3 冷却期状态展示

当用户处于冷却期时，前端会：

1. **禁用表单**：所有输入框和提交按钮变灰不可点击
2. **显示提示**：`"You have recently migrated your account. This function will become available again in X days."`
3. **显示历史**：展示过去的迁移记录

```haml
- if on_cooldown?
  %p.hint
    %span.warning-hint= t('migrations.on_cooldown', count: @cooldown.remaining_cooldown_days)
- else
  # 显示正常表单...

.actions
  = f.button :button, ..., disabled: on_cooldown?  # 冷却期禁用按钮
```

---

### 3. 第三阶段：异步任务执行（分布式）

#### 3.1 最终一致性模型

迁移的关注关系转移是**异步、分布式、最终一致**的：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         迁移的最终一致性模型                                    │
└─────────────────────────────────────────────────────────────────────────────┘

时间轴
──────►

T0: 用户在旧实例提交迁移表单
     │
     ▼
T1: 旧实例同步响应
     ✓ 前端显示成功消息
     ✓ moved_to_account 已设置
     ✓ Update 活动已分发
     ✓ Move 活动已入队
     ✗ 关注关系尚未转移（异步）
     │
     ▼
T2: 后台任务执行（旧实例）
     ✓ MoveWorker 处理本地关注关系
     ✓ 同实例内的关注已转移
     │
     ▼
T3: Move 活动分发（异步）
     ├──► 关注者实例 A 接收并处理
     │       ✓ 关注关系已转移
     │
     ├──► 关注者实例 B 接收并处理
     │       ✓ 关注关系已转移
     │
     └──► 关注者实例 C 暂时离线
             ✗ 活动丢失或延迟
             （依赖实例重试或后续刷新）
     │
     ▼
T4: 最终状态（时间不确定）
     ✓ 大部分关注关系已转移
     ✗ 少数可能因实例离线而延迟
     （用户可通过导出/导入手动补充）
```

#### 3.2 失败场景与不可见错误

这一阶段的错误**不会直接反馈给用户**，因为是异步分布式操作：

| 失败场景 | 影响范围 | 处理机制 | 用户感知 |
|----------|----------|----------|----------|
| **Sidekiq 任务失败** | 单条任务 | Sidekiq 自动重试 | 无感知，最终成功 |
| **关注者实例离线** | 该实例的所有关注者 | 依赖：1) 旧实例重试投递 2) 关注者实例后续刷新 | 无感知，可能延迟 |
| **协议验证失败** | 该关注者实例 | 活动被静默忽略 | **无感知，永远不会转移** |
| **数据库操作失败** | 单条关注关系 | 事务回滚、Sidekiq 重试 | 无感知，最终成功 |

**关键：协议验证失败是静默的**

当关注者实例收到 Move 活动，但验证失败时（如 `also_known_as` 不匹配），活动会被**静默忽略**，不会产生任何错误通知。

```ruby
# app/lib/activitypub/activity/move.rb:6-25
def perform
  return if origin_account.uri != object_uri           # 静默返回
  return unless mark_as_processing!                    # 静默返回
  
  target_account = ActivityPub::FetchRemoteAccountService.new.call(target_uri)
  
  if target_account.nil? || target_account.unavailable? || 
     !target_account.also_known_as.include?(origin_account.uri)
    unmark_as_processing!
    return                                              # 静默返回，无日志、无通知
  end
  
  # ... 执行迁移
end
```

**为什么这样设计？**

1. **安全**：防止恶意实例发送伪造的 Move 活动
2. **隐私**：不暴露内部验证逻辑给外部实例
3. **简洁**：失败的活动就是"不执行"，无需复杂的错误反馈协议

**用户如何发现问题？**

1. **观察粉丝数**：迁移后新旧账号的粉丝数变化是否符合预期
2. **手动抽查**：让几个好友确认是否已自动关注新账号
3. **手动补救**：导出旧账号的关注列表，在新账号导入

---

### 4. 失败反馈闭环总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         失败反馈闭环总览                                        │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ 阶段1: 新实例添加别名                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   用户输入 ──────►  WebFinger 解析 ──────►  验证 ──────►  结果             │
│   (old@old.com)      (跨实例网络请求)         (本地)                         │
│                                                                              │
│   可能的错误:                                                                 │
│   ✓ 账号不存在 ────────► 前端显示 "could not be found"                      │
│   ✓ 指向自己 ──────────► 前端显示 "cannot be current account"               │
│   ✓ 网络错误 ──────────► 前端显示 "could not be found"                      │
│   ✓ 重复添加 ──────────► 前端显示唯一约束错误                                │
│                                                                              │
│   反馈闭环: 同步、可见、可修复                                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 阶段2: 旧实例发起迁移                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   用户输入 ──────►  身份验证 ──────►  协议验证 ──────►  结果               │
│   (new@new.com)      (密码/用户名)           (also_known_as)                 │
│                              │                      │                        │
│                              │                      │  关键: 通过 WebFinger  │
│                              │                      │  获取新账号信息，验证  │
│                              │                      │  also_known_as 包含     │
│                              │                      │  旧账号 URI              │
│                                                                              │
│   可能的错误:                                                                 │
│   ✓ 密码错误 ──────────► 前端显示密码字段错误                                │
│   ✓ 目标不存在 ────────► 前端显示 "could not be found"                      │
│   ✓ 缺少别名 ──────────► 前端显示 "is not an alias of this account"         │
│   ✓ 已迁移过 ──────────► 前端显示 "is the same account you have already..."  │
│   ✓ 冷却期 ────────────► 前端禁用表单，显示剩余天数                          │
│                                                                              │
│   反馈闭环: 同步、可见、可修复                                                │
│                                                                              │
│   关键提示: 页面提前显示 "backreference_required" 警告                       │
│            引导用户先在新账号添加别名                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 阶段3: 异步任务执行（分布式）                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   旧实例              网络                  关注者实例                        │
│   ────────            ────                  ──────────                        │
│                                                                              │
│   Move 活动 ─────────► HTTP POST ─────────► Inbox 端点                      │
│   (异步分发)                                 (ActivityPub)                   │
│                                                      │                        │
│                                                      ▼                        │
│                                                协议验证                       │
│                                                - object == actor?            │
│                                                - 7天冷却期?                  │
│                                                - also_known_as?              │
│                                                      │                        │
│                              ┌───────────────────────┼───────────────────────┐
│                              │                       │                       │
│                              ▼                       ▼                       ▼
│                         验证成功               验证失败               实例离线
│                              │                       │                       │
│                              ▼                       ▼                       ▼
│                         执行迁移              静默忽略                活动丢失
│                         (MoveWorker)          (无反馈)              (无反馈)
│                                                                              │
│   可能的错误:                                                                 │
│   ✗ 协议验证失败 ──────► 静默忽略，无任何反馈                                │
│   ✗ 实例离线 ──────────► 活动丢失，依赖重试或后续刷新                        │
│   ✗ 数据库错误 ────────► Sidekiq 重试，最终成功                            │
│                                                                              │
│   反馈闭环: 异步、不可见、依赖最终一致性                                      │
│                                                                              │
│   用户补救:                                                                   │
│   - 观察粉丝数变化                                                            │
│   - 手动抽查好友是否已关注新账号                                              │
│   - 导出/导入关注列表作为补充                                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 旧实例：发起迁移与分发 Move 信号

### 1. 迁移验证流程（旧实例专属）

旧实例是迁移的**发起者和协调者**，负责执行最严格的前置验证。

#### 1.1 验证层级

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        旧实例迁移验证层级                                      │
└─────────────────────────────────────────────────────────────────────────────┘

Layer 5: 业务规则验证
┌─────────────────────────────────────────────────────────────────────────┐
│  validate_migration_cooldown                                              │
│  - 30天内是否已迁移过?                                                    │
│  - @cooldown.remaining_cooldown_days 显示剩余天数                         │
└─────────────────────────────────────────────────────────────────────────┘
                                      ▲
                                      │
Layer 4: 目标关系验证
┌─────────────────────────────────────────────────────────────────────────┐
│  validate_target_account (核心)                                          │
│  ├──► target_account.nil? ────────► "could not be found"               │
│  ├──► also_known_as 验证 ────────► "is not an alias of this account"   │
│  ├──► already_moved? ────────────► "is the same account..."             │
│  └──► move_to_self? ────────────► "cannot be current account"          │
└─────────────────────────────────────────────────────────────────────────┘
                                      ▲
                                      │
Layer 3: 目标账号解析
┌─────────────────────────────────────────────────────────────────────────┐
│  set_target_account                                                       │
│  - ResolveAccountService.new.call(acct, skip_cache: true)              │
│  - 跳过缓存，强制获取最新状态                                              │
│  - 捕获 Webfinger/网络异常，由验证层处理                                  │
└─────────────────────────────────────────────────────────────────────────┘
                                      ▲
                                      │
Layer 2: 身份验证
┌─────────────────────────────────────────────────────────────────────────┐
│  save_with_challenge                                                     │
│  ├──► 有密码? ──► current_user.valid_password?(current_password)        │
│  └──► 无密码? ──► account.username == current_username                  │
│         (OAuth 登录用户用用户名验证)                                       │
└─────────────────────────────────────────────────────────────────────────┘
                                      ▲
                                      │
Layer 1: 并发控制
┌─────────────────────────────────────────────────────────────────────────┐
│  with_redis_lock("account_migration:#{account.id}")                     │
│  - 防止同一账号并发发起迁移                                                │
│  - Redis 分布式锁                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 1.2 关键验证代码

```ruby
# app/models/account_migration.rb:45-57
def save_with_challenge(current_user)
  # Layer 2: 身份验证
  if current_user.encrypted_password.present?
    errors.add(:current_password, :invalid) unless current_user.valid_password?(current_password)
  else
    errors.add(:current_username, :invalid) unless account.username == current_username
  end

  return false unless errors.empty?

  # Layer 1: 并发控制
  with_redis_lock("account_migration:#{account.id}") do
    save  # 触发 Layer 3-5 验证
  end
end

# app/models/account_migration.rb:69-73
def set_target_account
  # Layer 3: 目标账号解析（跳过缓存）
  self.target_account = ResolveAccountService.new.call(acct, skip_cache: true)
rescue Webfinger::Error, *Mastodon::HTTP_CONNECTION_ERRORS, Mastodon::Error, Addressable::URI::InvalidURIError
  # 异常静默捕获，由 Layer 4 添加错误信息
end

# app/models/account_migration.rb:79-91
def validate_target_account
  # Layer 4: 目标关系验证
  if target_account.nil?
    errors.add(:acct, I18n.t('migrations.errors.not_found'))
  else
    # 核心：通过新账号的 also_known_as 验证"认领"关系
    errors.add(:acct, I18n.t('migrations.errors.missing_also_known_as')) 
      unless target_account.also_known_as.include?(ActivityPub::TagManager.instance.uri_for(account))
    
    errors.add(:acct, I18n.t('migrations.errors.already_moved')) 
      if account.moved? && account.moved_to_account_id == target_account.id
    
    errors.add(:acct, I18n.t('migrations.errors.move_to_self')) 
      if account.id == target_account.id
  end
end

# app/models/account_migration.rb:89-91
def validate_migration_cooldown
  # Layer 5: 冷却期验证
  errors.add(:base, I18n.t('migrations.errors.on_cooldown')) 
    if account.migrations.within_cooldown.exists?
end
```

### 2. MoveService 执行流程

验证通过后，`MigrationsController` 调用 `MoveService` 执行实际迁移：

```ruby
# app/controllers/settings/migrations_controller.rb:14-22
def create
  @migration = current_account.migrations.build(resource_params)
  
  if @migration.save_with_challenge(current_user)
    MoveService.new.call(@migration)
    redirect_to settings_migration_path, notice: I18n.t('migrations.moved_msg', ...)
  else
    render :show
  end
end
```

`MoveService` 执行四个关键步骤：

```ruby
# app/services/move_service.rb:3-32
class MoveService < BaseService
  def call(migration)
    @migration      = migration
    @source_account = migration.account
    @target_account = migration.target_account

    # 步骤1: 设置重定向（同步）
    update_redirect!
    
    # 步骤2: 处理本地关注关系（异步）
    process_local_relationships!
    
    # 步骤3: 分发 Update 活动（异步）
    distribute_update!
    
    # 步骤4: 分发 Move 活动（异步）
    distribute_move!
  end

  private

  def update_redirect!
    # 同步更新数据库：设置 moved_to_account
    @source_account.update!(moved_to_account: @target_account)
  end

  def process_local_relationships!
    # 异步处理同实例内的关注关系
    MoveWorker.perform_async(@source_account.id, @target_account.id)
  end

  def distribute_update!
    # 分发 Update 活动：宣告账号已迁移
    # Actor 文档中包含 movedTo 字段
    ActivityPub::UpdateDistributionWorker.perform_async(@source_account.id)
  end

  def distribute_move!
    # 分发 Move 活动：触发关注者实例重写关注关系
    ActivityPub::MoveDistributionWorker.perform_async(@migration.id)
  end
end
```

### 3. MoveDistributionWorker 分发机制

`MoveDistributionWorker` 负责将 Move 活动分发给所有相关方：

```ruby
# app/workers/activitypub/move_distribution_worker.rb:9-22
def perform(migration_id)
  @migration = AccountMigration.find(migration_id)
  @account   = @migration.account

  # 目标1: 所有关注者的 inbox
  ActivityPub::DeliveryWorker.push_bulk(inboxes, limit: 1_000) do |inbox_url|
    [signed_payload, @account.id, inbox_url]
  end

  # 目标2: 中继服务器（Relay）
  # 确保更广泛的传播
  ActivityPub::DeliveryWorker.push_bulk(Relay.enabled.pluck(:inbox_url)) do |inbox_url|
    [signed_payload, @account.id, inbox_url]
  end
end

private

def inboxes
  # 收集所有需要通知的 inbox:
  # 1. 关注者的 inbox
  # 2. 被屏蔽者的 inbox（让他们也知道迁移）
  @inboxes ||= (@migration.account.followers.inboxes + @migration.account.blocked_by.inboxes).uniq
end

def signed_payload
  # 使用 MoveSerializer 序列化
  @signed_payload ||= serialize_payload(@migration, ActivityPub::MoveSerializer, signer: @account).to_json
end
```

**为什么分发给被屏蔽者？**

- 被屏蔽者也可能是"关注者"（单向关注）
- 他们有权知道自己关注的账号已迁移
- 由他们自己决定是否关注新账号

### 4. Move 活动格式

`MoveSerializer` 生成标准的 ActivityPub Move 活动：

```ruby
# app/serializers/activitypub/move_serializer.rb
class ActivityPub::MoveSerializer < ActivityPub::Serializer
  attributes :id, :type, :target, :actor
  attribute :virtual_object, key: :object

  def id
    [ActivityPub::TagManager.instance.uri_for(object.account), '#moves/', object.id].join
  end

  def type
    'Move'
  end

  def target
    ActivityPub::TagManager.instance.uri_for(object.target_account)
  end

  def virtual_object
    ActivityPub::TagManager.instance.uri_for(object.account)
  end

  def actor
    ActivityPub::TagManager.instance.uri_for(object.account)
  end
end
```

**生成的 JSON**：
```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "id": "https://old.example.com/users/olduser#moves/123",
  "type": "Move",
  "actor": "https://old.example.com/users/olduser",
  "object": "https://old.example.com/users/olduser",
  "target": "https://new.example.com/users/newuser"
}
```

**关键字段含义**：

| 字段 | 值 | 含义 |
|------|-----|------|
| `actor` | 旧账号 URI | 活动发起者 |
| `object` | 旧账号 URI | 被迁移的对象（必须等于 actor） |
| `target` | 新账号 URI | 迁移目标 |

**为什么 `object` 必须等于 `actor`？**

- 防止恶意实例发送 `{actor: 攻击者, object: 受害者, target: 攻击者}` 这样的伪造活动
- 只有账号所有者才能迁移自己的账号
- 这是 `ActivityPub::Activity::Move` 的第一个验证点

---

## 新实例：维护别名与验证认领

### 1. 新实例的被动角色

新实例在迁移流程中是**被动的**：

- ❌ 不主动发起迁移
- ❌ 不处理关注关系转移
- ✓ 仅维护 `also_known_as` 别名关系
- ✓ 通过 Update 活动宣告别名关系
- ✓ 被其他实例查询时提供验证信息

### 2. 别名管理流程

#### 2.1 成功路径

```ruby
# app/controllers/settings/aliases_controller.rb:10-23
def index
  @alias = current_account.aliases.build
end

def create
  @alias = current_account.aliases.build(resource_params)

  if @alias.save
    # 保存成功后，分发 Update 活动
    ActivityPub::UpdateDistributionWorker.perform_async(current_account.id)
    redirect_to settings_aliases_path, notice: I18n.t('aliases.created_msg')
  else
    render :index
  end
end

def destroy
  @alias.destroy!
  redirect_to settings_aliases_path, notice: I18n.t('aliases.deleted_msg')
end
```

#### 2.2 AccountAlias 模型验证

```ruby
# app/models/account_alias.rb:15-56
class AccountAlias < ApplicationRecord
  belongs_to :account

  validates :acct, presence: true, domain: { acct: true }
  validates :uri, uniqueness: { scope: :account_id }  # 同一账号不能重复添加同一别名
  validate :validate_target_account

  before_validation :set_uri
  after_create :add_to_account      # 添加到 also_known_as
  after_destroy :remove_from_account # 从 also_known_as 移除

  private

  def set_uri
    # 解析目标账号，获取其 URI
    target_account = ResolveAccountService.new.call(acct)
    self.uri = ActivityPub::TagManager.instance.uri_for(target_account) unless target_account.nil?
  rescue Webfinger::Error, *Mastodon::HTTP_CONNECTION_ERRORS, Mastodon::Error
    # 异常由验证层处理
  end

  def add_to_account
    # 关键：将旧账号 URI 添加到新账号的 also_known_as
    account.update(also_known_as: account.also_known_as + [uri])
  end

  def remove_from_account
    # 移除时从 also_known_as 删除
    account.update(also_known_as: account.also_known_as.reject { |x| x == uri })
  end

  def validate_target_account
    if uri.blank?
      errors.add(:acct, I18n.t('migrations.errors.not_found'))
    elsif ActivityPub::TagManager.instance.uri_for(account) == uri
      errors.add(:acct, I18n.t('migrations.errors.move_to_self'))
    end
  end
end
```

### 3. also_known_as 的传播

当新账号添加别名后，会分发 Update 活动：

```ruby
# app/controllers/settings/aliases_controller.rb:17-18
ActivityPub::UpdateDistributionWorker.perform_async(current_account.id)
```

这使得全网实例可以在刷新新账号信息时获取最新的 `also_known_as`。

**Actor 文档中的表示**：
```json
{
  "type": "Person",
  "id": "https://new.example.com/users/newuser",
  "alsoKnownAs": [
    "https://old.example.com/users/olduser"
  ],
  "inbox": "https://new.example.com/users/newuser/inbox",
  "outbox": "https://new.example.com/users/newuser/outbox",
  // ... 其他字段
}
```

### 4. 新实例与旧实例的验证配合

| 验证时机 | 执行方 | 验证内容 | 数据来源 |
|----------|--------|----------|----------|
| 迁移发起时 | 旧实例 | `target_account.also_known_as.include?(源账号 URI)` | 旧实例通过 WebFinger + ActivityPub  fetch 获取新账号信息 |
| Move 处理时 | 关注者实例 | `target_account.also_known_as.include?(源账号 URI)` | 关注者实例通过 ActivityPub fetch 获取新账号信息 |

**关键**：新实例不需要"主动"做任何验证，它只需要：
1. 维护 `also_known_as` 数组
2. 在 Actor 文档中正确返回 `alsoKnownAs` 字段

验证逻辑由**旧实例**和**关注者实例**执行。

---

## 关注者实例：接收 Move 与重写关注关系

### 1. 关注者实例的核心角色

关注者实例是迁移的**实际执行者**：

- ✓ 接收 Move 活动
- ✓ 执行协议验证（最后一道防线）
- ✓ 重写本地关注关系
- ✓ 迁移相关数据（列表、备注、屏蔽、静音）

### 2. ActivityPub::Activity::Move 处理器

这是**最后一道验证防线**，也是最关键的安全验证：

```ruby
# app/lib/activitypub/activity/move.rb:1-44
class ActivityPub::Activity::Move < ActivityPub::Activity
  PROCESSING_COOLDOWN = 7.days.seconds  # 7天处理冷却期

  def perform
    # 验证1: object 必须等于 actor
    # 防止: {actor: 攻击者, object: 受害者, target: 攻击者}
    return if origin_account.uri != object_uri
    
    # 验证2: 7天内不能重复处理
    # 防止: 重复处理或攻击
    return unless mark_as_processing!

    # 获取目标账号信息
    target_account = ActivityPub::FetchRemoteAccountService.new.call(target_uri)

    # 验证3: 核心安全验证
    # - 目标账号存在且可用
    # - 目标账号的 also_known_as 包含源账号 URI
    # 这确保: 新账号"认领"了旧账号
    if target_account.nil? || target_account.unavailable? || 
       !target_account.also_known_as.include?(origin_account.uri)
      unmark_as_processing!
      return  # 静默失败，无错误通知
    end

    # 验证全部通过，执行迁移
    # 步骤1: 更新本地缓存的 moved_to_account
    origin_account.update(moved_to_account: target_account)

    # 步骤2: 异步处理关注关系转移
    MoveWorker.perform_async(origin_account.id, target_account.id)
  rescue
    unmark_as_processing!
    raise
  end

  private

  def origin_account
    @account  # Move 活动的 actor
  end

  def target_uri
    value_or_id(@json['target'])  # Move 活动的 target 字段
  end

  def mark_as_processing!
    # Redis 锁: 7天内不能重复处理
    redis.set("move_in_progress:#{@account.id}", true, nx: true, ex: PROCESSING_COOLDOWN)
  end

  def unmark_as_processing!
    redis.del("move_in_progress:#{@account.id}")
  end
end
```

### 3. 验证逻辑详解

#### 3.1 为什么需要三层验证？

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    关注者实例的三层验证逻辑                                    │
└─────────────────────────────────────────────────────────────────────────────┘

验证层1: object == actor
┌─────────────────────────────────────────────────────────────────────────┐
│  攻击场景:                                                                 │
│  攻击者控制实例 evil.com，发送:                                            │
│  {                                                                         │
│    "actor": "https://evil.com/users/attacker",                           │
│    "object": "https://good.com/users/victim",  ← 不等于 actor            │
│    "target": "https://evil.com/users/attacker"                           │
│  }                                                                         │
│                                                                           │
│  目的: 尝试"偷走"受害者的粉丝                                              │
│                                                                           │
│  防御: return if origin_account.uri != object_uri                         │
│                                                                           │
│  结果: 活动被静默忽略                                                      │
└─────────────────────────────────────────────────────────────────────────┘

验证层2: 7天冷却期
┌─────────────────────────────────────────────────────────────────────────┐
│  攻击场景:                                                                 │
│  攻击者控制实例，反复发送 Move 活动进行拒绝服务攻击                         │
│                                                                           │
│  或者:                                                                     │
│  正常用户在短时间内多次迁移（误操作）                                       │
│                                                                           │
│  防御: Redis 锁 "move_in_progress:{account_id}" 7天过期                  │
│                                                                           │
│  结果: 7天内同一账号的 Move 只处理一次                                    │
└─────────────────────────────────────────────────────────────────────────┘

验证层3: also_known_as 双向引用（核心）
┌─────────────────────────────────────────────────────────────────────────┐
│  攻击场景:                                                                 │
│  攻击者控制实例 old.com，发送:                                            │
│  {                                                                         │
│    "actor": "https://old.com/users/legit_user",                          │
│    "object": "https://old.com/users/legit_user",  ✓ 通过验证1           │
│    "target": "https://evil.com/users/attacker"                           │
│  }                                                                         │
│                                                                           │
│  目的: 攻击者控制了旧实例（或旧实例被攻破），想把合法用户的粉丝            │
│        转移到攻击者账号                                                    │
│                                                                           │
│  防御: 验证 target_account.also_known_as.include?(origin_account.uri)   │
│                                                                           │
│  关键: 攻击者控制了 old.com，但不控制 new.com                             │
│        无法在 new.com 的账号上添加 also_known_as                          │
│                                                                           │
│  结果: 活动被静默忽略                                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4. MoveWorker：关注关系转移执行

验证通过后，`MoveWorker` 执行实际的关注关系转移：

```ruby
# app/workers/move_worker.rb:3-400+
class MoveWorker
  include Sidekiq::Worker

  def perform(source_account_id, target_account_id)
    @source_account = Account.find(source_account_id)
    @target_account = Account.find(target_account_id)

    if @target_account.local? && @source_account.local?
      # 场景A: 源和目标都是本地账号
      # → 直接批量更新数据库（最高效）
      num_moved = rewrite_follows!
      @source_account.update_count!(:followers_count, -num_moved)
      @target_account.update_count!(:followers_count, num_moved)
    else
      # 场景B: 跨实例迁移
      # → 异步队列处理（需要与远程实例交互）
      queue_follow_unfollows!
    end

    # 其他数据迁移（两种场景都执行）
    copy_account_notes!    # 复制账号备注
    carry_blocks_over!     # 迁移屏蔽关系
    carry_mutes_over!      # 迁移静音关系

    raise @deferred_error unless @deferred_error.nil?
  rescue ActiveRecord::RecordNotFound
    true  # 账号已删除，静默处理
  end
```

#### 4.1 场景A：本地→本地迁移（rewrite_follows!）

**最高效的场景**：源账号和目标账号都在同一实例

```ruby
# app/workers/move_worker.rb:31-78
def rewrite_follows!
  num_moved = 0

  # 阶段1: 处理待处理的关注请求
  # 这些关注者已经请求关注新账号，先批准他们
  FollowRequest.where(account: @source_account.followers, target_account_id: @target_account.id).find_each do |follow_request|
    # 处理列表成员关系
    ListAccount.where(follow_id: follow_request.id).includes(:list).find_each do |list_account|
      list_account.list.accounts << @target_account
    rescue ActiveRecord::RecordInvalid
      nil
    end
    follow_request.authorize!
  end

  # 阶段2: 处理同时关注新旧账号的情况
  # 这些关注者已经关注了新账号，只需处理列表
  source_local_followers
    .where(account: @target_account.followers.local)
    .in_batches do |follows|
      ListAccount.where(follow: follows).includes(:list).find_each do |list_account|
        list_account.list.accounts << @target_account
      rescue ActiveRecord::RecordInvalid
        nil
      end
    end

  # 阶段3: 处理只关注旧账号的情况（最常见）
  # 批量更新数据库
  source_local_followers
    .where.not(account: @target_account.followers.local)
    .where.not(account_id: @target_account.id)
    .in_batches do |follows|
      # 批量更新列表成员
      ListAccount.where(follow: follows).in_batches.update_all(account_id: @target_account.id)
      
      # 批量更新关注关系：旧→新
      num_moved += follows.update_all(target_account_id: @target_account.id)

      # 手动清理缓存（update_all 不触发回调）
      Rails.cache.delete_multi(follows.flat_map do |follow|
        [
          ['relationships', follow.account_id, follow.target_account_id],
          ['relationships', follow.target_account_id, follow.account_id],
          ['relationships', follow.account_id, @target_account.id],
          ['relationships', @target_account.id, follow.account_id],
        ]
      end)
    end

  num_moved
end
```

**为什么分三个阶段？**

| 阶段 | 场景 | 处理方式 | 原因 |
|------|------|----------|------|
| 1 | 待处理请求 | 先批准，再迁移列表 | 确保列表成员关系正确 |
| 2 | 同时关注 | 只迁移列表 | 已经关注了新账号，无需重复关注 |
| 3 | 仅关注旧账号 | 批量更新 | 最常见场景，最高效 |

#### 4.2 场景B：跨实例迁移（queue_follow_unfollows!）

**需要与远程实例交互**的场景

```ruby
# app/workers/move_worker.rb:86-94
def queue_follow_unfollows!
  # bypass_locked: 如果新账号是本地账号，可以绕过锁定限制
  bypass_locked = @target_account.local?

  # 批量推送 UnfollowFollowWorker 任务
  @source_account.followers.local.select(:id).reorder(nil).find_in_batches do |accounts|
    UnfollowFollowWorker.push_bulk(accounts.map(&:id)) { |follower_id| 
      [follower_id, @source_account.id, @target_account.id, bypass_locked] 
    }
  rescue => e
    @deferred_error = e
  end
end
```

#### 4.3 UnfollowFollowWorker + FollowMigrationService

单条关注关系的迁移逻辑：

```ruby
# app/workers/unfollow_follow_worker.rb:3-17
class UnfollowFollowWorker
  include Sidekiq::Worker

  def perform(follower_account_id, old_target_account_id, new_target_account_id, bypass_locked = false)
    follower_account   = Account.find(follower_account_id)
    old_target_account = Account.find(old_target_account_id)
    new_target_account = Account.find(new_target_account_id)

    FollowMigrationService.new.call(
      follower_account, 
      new_target_account, 
      old_target_account, 
      bypass_locked: bypass_locked
    )
  rescue ActiveRecord::RecordNotFound, Mastodon::NotPermittedError
    true
  end
end
```

```ruby
# app/services/follow_migration_service.rb:3-62
class FollowMigrationService < FollowService
  def call(source_account, target_account, old_target_account, bypass_locked: false)
    @old_target_account = old_target_account

    # 读取原有关注设置
    @original_follow = source_account.active_relationships.find_by(target_account: old_target_account)
    reblogs          = @original_follow&.show_reblogs?
    notify           = @original_follow&.notify?
    languages        = @original_follow&.languages

    # 调用父类 FollowService，保留原有设置
    super(source_account, target_account, 
          reblogs: reblogs, notify: notify, languages: languages, 
          bypass_locked: bypass_locked, bypass_limit: true)
  end

  private

  # 场景A: 新账号锁定 → 创建关注请求
  def request_follow!
    follow_request = @source_account.request_follow!(@target_account, **follow_options)
    migrate_list_accounts!  # 迁移列表成员

    if @target_account.local?
      # 本地账号：发送通知，立即取消关注旧账号
      LocalNotificationWorker.perform_async(@target_account.id, follow_request.id, ...)
      UnfollowService.new.call(@source_account, @old_target_account, skip_unmerge: true)
    elsif @target_account.activitypub?
      # 远程账号：发送 ActivityPub Follow 活动
      ActivityPub::MigratedFollowDeliveryWorker.perform_async(
        build_json(follow_request), 
        @source_account.id, 
        @target_account.inbox_url, 
        @old_target_account.id
      )
    end

    follow_request
  end

  # 场景B: 直接关注（新账号未锁定或 bypass_locked）
  def direct_follow!
    follow = super
    migrate_list_accounts!
    # 立即取消关注旧账号
    UnfollowService.new.call(@source_account, @old_target_account, skip_unmerge: true)
    follow
  end

  # 迁移列表成员关系
  def migrate_list_accounts!
    ListAccount.where(follow_id: @original_follow.id).includes(:list).find_each do |list_account|
      list_account.list.accounts << @target_account
    rescue ActiveRecord::RecordInvalid
      nil
    end
  end
end
```

#### 4.4 其他数据迁移

`MoveWorker` 还处理以下数据迁移：

```ruby
# app/workers/move_worker.rb:96-156

# 复制账号备注
def copy_account_notes!
  @source_account.targeted_account_notes.find_each do |note|
    # 添加迁移提示前缀
    text = I18n.with_locale(note.account.user_locale.presence || I18n.default_locale) do
      I18n.t('move_handler.copy_account_note_text', acct: @source_account.acct)
    end

    new_note = @target_account.targeted_account_notes.find_by(account: note.account)
    if new_note.nil?
      # 新建备注
      @target_account.targeted_account_notes.create!(
        account: note.account, 
        comment: [text, note.comment].join("\n")
      )
    else
      # 合并备注
      new_note.update!(comment: [text, note.comment, "\n", new_note.comment].join("\n"))
    end
  end
end

# 迁移屏蔽关系
def carry_blocks_over!
  @source_account.blocked_by_relationships.where(account: Account.local).find_each do |block|
    unless skip_block_move?(block)
      BlockService.new.call(block.account, @target_account)
      add_account_note_if_needed!(block.account, 'move_handler.carry_blocks_over_text')
    end
  end
end

def skip_block_move?(block)
  # 跳过条件：已屏蔽新账号 或 正在关注新账号
  block.account.blocking?(@target_account) || block.account.following?(@target_account)
end

# 迁移静音关系（类似屏蔽）
def carry_mutes_over!
  @source_account.muted_by_relationships.where(account: Account.local).find_each do |mute|
    unless skip_mute_move?(mute)
      MuteService.new.call(mute.account, @target_account, notifications: mute.hide_notifications)
      add_account_note_if_needed!(mute.account, 'move_handler.carry_mutes_over_text')
    end
  end
end
```

---

## 三方协同时序总结

### 1. 完整时序图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        Mastodon 账号迁移三方协同时序图                                 │
└─────────────────────────────────────────────────────────────────────────────────────┘

时间轴
──────►
  │
  │   ┌──────────┐         ┌──────────┐         ┌──────────┐
  │   │  用户    │         │  新实例   │         │  旧实例   │
  │   └────┬─────┘         └────┬─────┘         └────┬─────┘
  │        │                    │                    │
  │        │ 1. 登录新账号        │                    │
  │        │───────────────────►│                    │
  │        │                    │                    │
  │        │ 2. 访问 /settings/aliases                │
  │        │───────────────────►│                    │
  │        │                    │                    │
  │        │ 3. 输入旧账号地址   │                    │
  │        │    (old@old.com)   │                    │
  │        │───────────────────►│                    │
  │        │                    │                    │
  │        │                    │ 4. WebFinger 解析  │
  │        │                    │    旧账号信息       │
  │        │                    │◄───────────────────│ (跨实例)
  │        │                    │                    │
  │        │                    │ 5. 验证通过         │
  │        │                    │    - 账号存在        │
  │        │                    │    - 不是自己        │
  │        │                    │                    │
  │        │                    │ 6. 更新 also_known_as│
  │        │                    │    + [旧账号 URI]    │
  │        │                    │                    │
  │        │                    │ 7. 分发 Update 活动  │
  │        │                    │    (Actor 包含       │
  │        │                    │     alsoKnownAs)     │
  │        │◄───────────────────│                    │
  │        │    "Alias created"  │                    │
  │        │                    │                    │
  ▼        │                    │                    │
           │ 8. 登录旧账号        │                    │
           │────────────────────────────────────────►│
           │                    │                    │
           │ 9. 访问 /settings/migration             │
           │────────────────────────────────────────►│
           │                    │                    │
           │                    │                    │ 10. 检查状态
           │                    │                    │     - 冷却期?
           │                    │                    │     - 已迁移?
           │◄────────────────────────────────────────│
           │    显示迁移表单      │                    │
           │    + 风险提示        │                    │
           │    + "backreference  │                    │
           │      required" 提示   │                    │
           │                    │                    │
           │ 11. 输入新账号地址    │                    │
           │     + 密码           │                    │
           │────────────────────────────────────────►│
           │                    │                    │
           │                    │                    │ 12. 身份验证
           │                    │                    │     - 密码/用户名
           │                    │                    │
           │                    │                    │ 13. WebFinger 解析
           │                    │                    │     新账号信息
           │                    │◄───────────────────│ (跨实例)
           │                    │                    │
           │                    │                    │ 14. 核心验证
           │                    │                    │     - also_known_as
           │                    │                    │       包含旧账号 URI?
           │                    │                    │     - 冷却期?
           │                    │                    │     - 不是自己?
           │                    │                    │
           │◄────────────────────────────────────────│ 15a. 验证失败
           │    显示错误消息      │                    │
           │    (如 "is not an   │                    │
           │     alias of this   │                    │
           │     account")       │                    │
           │                    │                    │
           │                    │                    │ 15b. 验证成功
           │                    │                    │
           │                    │                    │ 16. MoveService 执行
           │                    │                    │     - 设置 moved_to_account
           │                    │                    │     - MoveWorker 入队
           │                    │                    │     - UpdateDistributionWorker 入队
           │                    │                    │     - MoveDistributionWorker 入队
           │                    │                    │
           │◄────────────────────────────────────────│ 17. 同步响应
           │    "Your account is  │                    │
           │     now redirecting  │                    │
           │     to ..."          │                    │
           │                    │                    │
  ▼        │                    │                    │
           │                    │                    │
           │                    │                    │ 18. 异步任务执行
           │                    │                    │
           │                    │                    │     旧实例本地:
           │                    │                    │     - MoveWorker 处理本地关注
           │                    │                    │
           │                    │                    │     分布式:
           │                    │                    │     - MoveDistributionWorker 分发
           │                    │                    │     - UpdateDistributionWorker 分发
           │                    │                    │
           │                    │                    │ 19. 关注者实例接收
           │                    │                    │
           │                    │                    │     ┌─────────────────────────────┐
           │                    │                    │     │ 关注者实例处理流程:           │
           │                    │                    │     │                               │
           │                    │                    │     │ 1. 接收 Move 活动            │
           │                    │                    │     │ 2. 验证 object == actor       │
           │                    │                    │     │ 3. 验证 7 天冷却期           │
           │                    │                    │     │ 4. Fetch 新账号信息          │
           │                    │                    │     │ 5. 验证 also_known_as        │
           │                    │                    │     │    包含旧账号 URI?           │
           │                    │                    │     │                               │
           │                    │                    │     │ 验证通过:                    │
           │                    │                    │     │ - 更新 moved_to_account      │
           │                    │                    │     │ - MoveWorker 入队            │
           │                    │                    │     │                               │
           │                    │                    │     │ 验证失败:                    │
           │                    │                    │     │ - 静默忽略                   │
           │                    │                    │     │ - 无任何反馈                 │
           │                    │                    │     └─────────────────────────────┘
           │                    │                    │
           │                    │                    │ 20. 最终一致性达成
           │                    │                    │
           │                    │                    │     时间不确定，依赖:
           │                    │                    │     - Sidekiq 队列处理速度
           │                    │                    │     - 各实例在线状态
           │                    │                    │     - 网络延迟
           │                    │                    │
```