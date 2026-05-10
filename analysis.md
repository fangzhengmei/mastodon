# Mastodon 举报审核流程分析

## 一、举报进入审核队列的流程

### 1.1 用户举报入口
用户通过 API 接口提交举报：
- **控制器入口**: `app/controllers/api/v1/reports_controller.rb:9-17`
  ```ruby
  def create
    @report = ReportService.new.call(
      current_account,
      reported_account,
      report_params.merge(application: doorkeeper_token.application)
    )
  end
  ```
- **认证**: 需要 `write:reports` OAuth 权限

### 1.2 举报数据结构
**举报模型**: `app/models/report.rb`

**关键字段**:
- `account_id`: 举报人账号 ID
- `target_account_id`: 被举报账号 ID
- `status_ids`: 被举报的嘟文 ID 数组
- `category`: 举报分类（other/spam/legal/violation）
- `rule_ids`: 违规规则 ID 数组
- `comment`: 用户备注（最多 1000 字符）
- `forwarded`: 是否转发到远端实例
- `action_taken_at`: 管理员处理时间（nil 表示未处理）
- `assigned_account_id`: 分配给哪个管理员处理

**状态管理**:
- `unresolved` scope: `where(action_taken_at: nil)` - 未处理的举报（审核队列）
- `resolved` scope: `where.not(action_taken_at: nil)` - 已处理的举报

### 1.3 举报创建服务
**服务类**: `app/services/report_service.rb`

**创建流程**:
1. **创建举报记录**: `create_report!` (第 32-43 行)
   - 关联举报人、被举报人、嘟文、分类、规则
   - 设置 URI（仅本地账号生成）

2. **通知管理员**: `notify_staff!` (第 46-53 行)
   ```ruby
   def notify_staff!
     return if @report.unresolved_siblings?  # 如果已有未处理的同目标举报，不重复通知
     
     User.those_who_can(:manage_reports).includes(:account).find_each do |u|
       LocalNotificationWorker.perform_async(u.account_id, @report.id, 'Report', 'admin.report')
       AdminMailer.with(recipient: u.account).new_report(@report).deliver_later if u.allows_report_emails?
     end
   end
   ```
   - 向所有拥有 `manage_reports` 权限的用户发送通知
   - 同时发送邮件通知（如用户允许）

3. **可选转发到远端实例**: `forward_to_origin!` (第 55-60 行)
   - 通过 ActivityPub 协议将举报发送到目标账号的原始实例
   - 仅当目标是远端账号且用户选择转发时执行

### 1.4 审核队列管理
**过滤机制**: `app/models/report_filter.rb`

**过滤条件**:
- `resolved`: 是否已处理
- `account_id`: 按举报人筛选
- `target_account_id`: 按被举报人筛选
- `by_target_domain`: 按目标账号域名筛选
- `target_origin`: 本地 (`local`) 或远端 (`remote`) 账号

**默认显示未处理举报**:
```ruby
def results
  scope = Report.unresolved  # 默认只显示未处理的举报
  # 应用其他过滤条件...
end
```

**管理员控制器**: `app/controllers/admin/reports_controller.rb`
- `index`: 显示过滤后的举报列表
- `show`: 查看单个举报详情（包含操作历史）
- `assign_to_self`: 分配给自己处理
- `unassign`: 取消分配
- `resolve`: 标记为已处理
- `reopen`: 重新打开

---

## 二、举报转发到远端实例的完整路径

### 2.1 转发触发条件
**举报服务**: `app/services/report_service.rb:71-81`

```ruby
def forward?
  !@target_account.local? && ActiveModel::Type::Boolean.new.cast(@options[:forward])
end

def forward_to_origin?
  forward? && forward_to_domains.include?(@target_account.domain)
end

def forward_to_domains
  @forward_to_domains ||= (@options[:forward_to_domains] || [@target_account.domain]).filter_map do |domain|
    TagManager.instance.normalize_domain(domain)
  end.uniq
end
```

**转发条件总结**:
1. 目标账号必须是远端账号（`!target_account.local?`）
2. 用户必须勾选转发选项（`forward: true`）
3. `forward_to_domains` 包含目标实例域名（默认包含）

### 2.2 转发到目标账号原实例

**实现**: `app/services/report_service.rb:55-60`

```ruby
def forward_to_origin!
  return unless forward_to_origin?

  # Send report to the server where the account originates from
  ActivityPub::DeliveryWorker.perform_async(
    payload, 
    some_local_account.id, 
    @target_account.inbox_url
  )
end
```

**发送对象**:
- **目标**: `@target_account.inbox_url` - 被举报账号所在实例的 inbox
- **发送者**: `some_local_account` - 本实例的代表账号（`Account.representative`）
- **内容**: `payload` - 使用 `ActivityPub::FlagSerializer` 序列化的举报信息

**ActivityPub 格式**:
- 类型: `Flag`（举报）
- 包含: 举报人、被举报人、被举报的嘟文 URI、举报原因

### 2.3 转发到回复链相关实例

**实现**: `app/services/report_service.rb:62-69`

```ruby
def forward_to_replied_to!
  # Send report to servers to which the account was replying to, so they also have a chance to act
  inbox_urls = Account.remote
    .where(domain: forward_to_domains)
    .where(id: Status.where(id: reported_status_ids).where.not(in_reply_to_account_id: nil).select(:in_reply_to_account_id))
    .inboxes - [@target_account.inbox_url, @target_account.shared_inbox_url]

  inbox_urls.each do |inbox_url|
    ActivityPub::DeliveryWorker.perform_async(payload, some_local_account.id, inbox_url)
  end
end
```

**转发逻辑解析**:

```
被举报嘟文列表 (reported_status_ids)
       ↓
筛选有回复对象的嘟文 (in_reply_to_account_id IS NOT NULL)
       ↓
获取这些回复对象的账号 ID
       ↓
筛选其中的远端账号 (Account.remote)
       ↓
限制在用户指定的 forward_to_domains 列表中
       ↓
排除被举报账号自身的 inbox/shared_inbox
       ↓
向剩余的所有 inbox 发送 Flag 活动
```

**转发触发场景**:
假设存在以下对话链：
```
实例A: 账号X 发布嘟文 S1
实例B: 账号Y 回复 S1 → 嘟文 S2 (in_reply_to_account_id = X.id)
实例C: 账号Z 回复 S2 → 嘟文 S3 (in_reply_to_account_id = Y.id)
```

**场景**: 实例D的用户举报了实例B的账号Y的嘟文 S2
- **目标原实例**: 实例B（账号Y所在实例）
- **回复链实例**: 实例A（嘟文 S2 回复的 S1 来自实例A）

**用户可选的转发目标**: 用户可以通过 `forward_to_domains` 参数指定转发到哪些域名

### 2.4 转发路径总览

```
用户提交举报 (API: POST /api/v1/reports)
       ↓
ReportService.call()
       ↓
       ├─→ forward? 检查 (远端账号 + 用户勾选转发)
       │         ↓
       │    forward_to_domains 计算
       │         ↓
       ├─→ forward_to_origin!
       │    └─→ 发送 ActivityPub Flag 到 目标账号原实例 inbox
       │
       └─→ forward_to_replied_to!
            ├─→ 从 reported_status_ids 提取 in_reply_to_account_id
            ├─→ 筛选远端账号 + 域名匹配
            ├─→ 排除目标账号自身
            └─→ 向每个回复对象的实例 inbox 发送 Flag
```

### 2.5 前端显示提示

**举报详情页提示**: `app/views/admin/reports/show.html.haml:10-11`

```haml
- unless @report.local? || @report.target_account.local?
  .flash-message= t('admin.reports.forwarded_replies_explanation')
```

当以下条件满足时，管理员会看到转发说明提示：
- 举报人是远端账号（`!@report.local?`，举报来自其他实例）
- **且** 被举报人也是远端账号

这表明该举报可能已被转发到回复链相关实例。

---

## 三、管理员操作对账号限制的影响

### 3.1 管理员操作类型

#### 3.1.1 操作类型总览

**账号操作**: `app/models/admin/account_action.rb`

| 操作类型 | 内部类型 | 说明 | 适用范围 |
|---------|---------|------|---------|
| 仅警告 | `none` | 仅发送警告，不采取限制 | 仅本地账号 |
| 禁用账号 | `disable` | 禁用用户登录能力 | 仅本地账号 |
| 标记敏感 | `sensitive` | 账号所有媒体默认敏感 | 本地+远端 |
| 静音 | `silence` | 账号嘟文仅对粉丝可见 | 本地+远端 |
| 暂停/封禁 | `suspend` | 暂停账号，阻止所有交互 | 本地+远端 |

**嘟文/集合操作**: `app/models/admin/moderation_action.rb`

| 操作类型 | 内部类型 | 说明 |
|---------|---------|------|
| 删除 | `delete` | 删除被举报的嘟文/集合 |
| 标记敏感 | `mark_as_sensitive` | 标记嘟文为敏感内容 |

### 3.2 举报页可触发的动作

#### 3.2.1 举报页动作入口

**控制器**: `app/controllers/admin/reports_controller.rb` + `app/controllers/admin/reports/actions_controller.rb`

**路由配置**: `config/routes/admin.rb:124-137`

```ruby
resources :reports, only: [:index, :show] do
  resources :actions, only: [:create], module: :reports do
    collection do
      post :preview   # POST /admin/reports/:report_id/actions/preview
    end
  end

  member do
    post :assign_to_self   # POST /admin/reports/:id/assign_to_self
    post :unassign         # POST /admin/reports/:id/unassign
    post :reopen           # POST /admin/reports/:id/reopen
    post :resolve          # POST /admin/reports/:id/resolve
  end
end
```

#### 3.2.2 举报详情页动作按钮

**视图**: `app/views/admin/reports/_actions.html.haml`

| 动作按钮 | 路由 | 目标 | 限制条件 |
|---------|------|------|---------|
| 标记已解决 | `resolve_admin_report_path` | 仅关闭举报 | 无 |
| 标记为敏感 | `preview_admin_report_actions_path` | 嘟文/集合 | 需要有媒体/预览卡的嘟文或集合 |
| 删除并解决 | `preview_admin_report_actions_path` | 嘟文/集合 | 需要有关联的嘟文或集合 |
| 静音 | `preview_admin_report_actions_path` | 目标账号 | 账号未被静音/本地暂停 |
| 暂停 | `preview_admin_report_actions_path` | 目标账号 | 账号未被本地暂停 |
| 自定义 | `new_admin_account_action_path` | 目标账号 | 无 |

#### 3.2.3 举报页动作执行流程

**控制器**: `app/controllers/admin/reports/actions_controller.rb:11-24`

```ruby
def create
  authorize @report, :show?

  case action_from_button
  when 'delete', 'mark_as_sensitive'
    # 嘟文/集合操作 → 使用 ModerationAction
    Admin::ModerationAction.new(moderation_action_params).save!
  when 'silence', 'suspend'
    # 账号限制操作 → 使用 AccountAction
    Admin::AccountAction.new(account_action_params).save!
  else
    return redirect_to admin_report_path(@report), alert: I18n.t('admin.reports.unknown_action_msg', action: action_from_button)
  end

  redirect_to admin_reports_path, notice: I18n.t('admin.reports.processed_msg', id: @report.id)
end
```

**共享参数**:
```ruby
def shared_params
  {
    current_account: current_account,
    report_id: @report.id,
    send_email_notification: !@report.spam?,  # 垃圾邮件举报不发送邮件
    text: params[:text],                       # 警告文本（仅本地账号）
    type: action_from_button,                  # 操作类型
  }
end
```

#### 3.2.4 举报页动作预览页

**视图**: `app/views/admin/reports/actions/preview.html.haml`

预览页显示的影响摘要：

| 操作类型 | 影响描述 |
|---------|---------|
| `delete` / `mark_as_sensitive` | 仅关闭当前举报 |
| `silence` / `suspend` | 关闭该账号所有未解决的举报 |

所有操作都会：
1. 记录一次 strike（警告/处罚记录）
2. 本地账号且非垃圾邮件举报 → 发送邮件通知

### 3.3 账号管理页可触发的动作

#### 3.3.1 账号管理页动作入口

**控制器**: `app/controllers/admin/accounts_controller.rb` + `app/controllers/admin/account_actions_controller.rb`

**路由配置**: `config/routes/admin.rb:141-170`

```ruby
resources :accounts, only: [:index, :show, :destroy], concerns: :batch do
  member do
    post :enable         # POST /admin/accounts/:id/enable
    post :unsensitive    # POST /admin/accounts/:id/unsensitive
    post :unsilence      # POST /admin/accounts/:id/unsilence
    post :unsuspend      # POST /admin/accounts/:id/unsuspend
    post :redownload     # POST /admin/accounts/:id/redownload
    post :remove_avatar  # POST /admin/accounts/:id/remove_avatar
    post :remove_header  # POST /admin/accounts/:id/remove_header
    post :memorialize    # POST /admin/accounts/:id/memorialize
    post :approve        # POST /admin/accounts/:id/approve
    post :reject         # POST /admin/accounts/:id/reject
    post :unblock_email  # POST /admin/accounts/:id/unblock_email
  end

  resource :action, only: [:new, :create], controller: 'account_actions'
  # GET  /admin/accounts/:account_id/action/new
  # POST /admin/accounts/:account_id/action
end
```

#### 3.3.2 账号管理页动作按钮

**视图**: `app/views/admin/accounts/_buttons.html.haml`

**可用动作按账号状态和类型区分**:

| 场景 | 可用动作 | 适用账号类型 |
|------|---------|-------------|
| **正常状态** | | |
| 发送警告 | `new_admin_account_action_path(account, type: 'none')` | 本地已批准 |
| 启用账号 | `enable_admin_account_path(account)` | 本地已禁用 |
| 禁用账号 | `new_admin_account_action_path(account, type: 'disable')` | 本地已批准 |
| 撤销敏感标记 | `unsensitive_admin_account_path(account)` | 已标记敏感的账号 |
| 标记敏感 | `new_admin_account_action_path(account, type: 'sensitive')` | 远端 或 本地已批准 |
| 撤销静音 | `unsilence_admin_account_path(account)` | 已静音的账号 |
| 静音 | `new_admin_account_action_path(account, type: 'silence')` | 远端 或 本地已批准 |
| 批准注册 | `approve_admin_account_path(account)` | 本地待审批 |
| 拒绝注册 | `reject_admin_account_path(account)` | 本地待审批 |
| 确认邮箱 | `admin_account_confirmation_path(account)` | 本地未确认 |
| 执行完整暂停 | `new_admin_account_action_path(account, type: 'suspend')` | 远端 或 本地已批准 |
| 设置纪念账号 | `memorialize_admin_account_path(account)` | 本地已批准 |
| 重新获取信息 | `redownload_admin_account_path(account)` | 远端账号 |
| **已暂停状态** | | |
| 撤销暂停 | `unsuspend_admin_account_path(account)` | 已暂停的账号 |
| 重新获取信息 | `redownload_admin_account_path(account)` | 远端发起的暂停 |
| 立即删除 | `admin_account_path(account)` DELETE | 有删除请求的账号 |

#### 3.3.3 账号管理页自定义动作表单

**视图**: `app/views/admin/account_actions/new.html.haml`

**表单字段**:

| 字段 | 说明 | 适用范围 |
|------|------|---------|
| `type` | 操作类型单选（none/disable/sensitive/silence/suspend） | 所有账号（远端账号选项较少） |
| `send_email_notification` | 是否发送邮件通知 | 仅本地账号 |
| `include_statuses` | 警告中是否包含举报的嘟文 | 本地账号 + 有关联举报 |
| `warning_preset_id` | 选择预设警告模板 | 仅本地账号 |
| `text` | 自定义警告文本 | 仅本地账号 |

### 3.4 两类页面动作对比

#### 3.4.1 动作范围对比

| 维度 | 举报详情页 | 账号管理页 |
|------|-----------|-----------|
| **操作上下文** | 针对具体举报 | 针对账号整体 |
| **关联举报** | 自动关联当前举报 | 可选关联（无默认关联） |
| **嘟文操作** | ✓ 删除、标记敏感 | ✗ 需单独进入嘟文管理 |
| **快捷操作** | ✓ 5个快捷按钮（含预览） | ✓ 更多独立动作按钮 |
| **自定义警告** | ✓ 跳转至账号动作表单 | ✓ 直接进入表单 |
| **撤销操作** | ✗ 无 | ✓ 撤销敏感、静音、暂停等 |
| **账号生命周期** | ✗ 无 | ✓ 批准、拒绝、纪念化等 |
| **账号内容** | ✗ 无 | ✓ 移除头像、移除头部等 |

#### 3.4.2 限制影响差异

**举报页动作的自动行为**:
```
举报页执行账号限制操作 (silence/suspend):
       ↓
Admin::AccountAction 处理
       ↓
       ├─→ 执行限制操作 (silence! / suspend!)
       ├─→ 创建 AccountWarning (strike)
       ├─→ 自动关闭该账号 所有 未解决的举报
       ├─→ 记录审计日志
       └─→ 本地账号 → 发送邮件通知
```

**举报页执行嘟文操作 (delete/mark_as_sensitive)**:
```
举报页执行嘟文操作:
       ↓
Admin::ModerationAction 处理
       ↓
       ├─→ 仅处理举报关联的嘟文/集合
       ├─→ 创建 AccountWarning (strike)
       ├─→ 仅关闭 当前 举报
       └─→ 记录审计日志
```

**账号管理页动作**:
```
账号管理页执行账号限制操作:
       ↓
Admin::AccountAction 处理（无 report_id）
       ↓
       ├─→ 执行限制操作
       ├─→ 创建 AccountWarning (strike)
       ├─→ 不关闭任何举报（无关联）
       ├─→ 记录审计日志
       └─→ 本地账号 → 发送邮件通知
```

**关键差异总结**:
1. **举报页** = 上下文感知，自动关联举报、自动关闭相关举报
2. **账号管理页** = 独立操作，可执行完整的账号生命周期管理
3. **举报页的账号限制** 会关闭该账号所有未解决的举报
4. **举报页的嘟文操作** 仅关闭当前举报

### 3.5 具体限制操作详解

#### 3.5.1 禁用账号 (disable)
- **位置**: `app/models/admin/account_action.rb:85-89`
- **影响**: 禁用用户的登录能力
- **审计**: `log_action(:disable, target_account.user)`
- **撤销**: `accounts#enable`

#### 3.5.2 标记为敏感 (sensitive)
- **位置**: `app/models/admin/account_action.rb:91-95`
- **方法**: `target_account.sensitize!` - `app/models/concerns/account/sensitizes.rb:14-16`
- **影响**: 设置 `sensitized_at` 时间戳，账号所有媒体默认标记为敏感
- **审计**: `log_action(:sensitive, target_account)`
- **撤销**: `accounts#unsensitive`

#### 3.5.3 静音 (silence)
- **位置**: `app/models/admin/account_action.rb:97-101`
- **方法**: `target_account.silence!` - `app/models/concerns/account/silences.rb:15-17`
- **影响**: 设置 `silenced_at` 时间戳，账号嘟文仅对粉丝可见
- **审计**: `log_action(:silence, target_account)`
- **撤销**: `accounts#unsilence`

#### 3.5.4 暂停/封禁 (suspend)
- **位置**: `app/models/admin/account_action.rb:103-107`
- **方法**: `target_account.suspend!(origin: :local)` - `app/models/concerns/account/suspensions.rb:29-39`
- **影响**:
  - 创建删除请求
  - 设置 `suspended_at` 和 `suspension_origin`
  - 阻塞邮箱（可选）
  - 本地账号：强制断开所有流式连接
- **审计**: `log_action(:suspend, target_account)`
- **后台处理**: `Admin::SuspensionWorker`
- **撤销**: `accounts#unsuspend`

**暂停服务**: `app/services/suspend_account_service.rb`
```ruby
def call(account)
  return unless account.suspended?
  
  reject_remote_follows!    # 强制远端账号取消关注本地账号
  distribute_update_actor!  # 本地账号：向联邦网络广播更新
  unmerge_from_home_timelines!  # 从时间线移除
  unmerge_from_list_timelines!  # 从列表移除
  privatize_media_attachments!  # 媒体设为私有
  remove_from_trends!       # 从趋势移除
end
```

#### 3.5.5 删除嘟文 (delete)
**流程**: `app/models/admin/moderation_action.rb:34-51`

```ruby
def handle_delete!
  statuses.each { |status| authorize([:admin, status], :destroy?) }
  collections.each { |collection| authorize([:admin, collection], :destroy?) }

  ApplicationRecord.transaction do
    delete_statuses!      # 删除嘟文（discard_with_reblogs）
    delete_collections!   # 删除集合

    resolve_report!       # 关闭当前举报
    process_strike!(:delete_statuses)  # 创建警告

    create_tombstones! unless target_account.local?  # 远端账号创建墓碑
  end

  process_notification!

  # 后台清理
  RemovalWorker.push_bulk(status_ids) { |status_id| 
    [status_id, { 
      'preserve' => target_account.local?,   # 本地账号保留数据
      'immediate' => !target_account.local?  # 远端账号立即清理
    }]
  }
end
```

#### 3.5.6 标记嘟文为敏感 (mark_as_sensitive)
**流程**: `app/models/admin/moderation_action.rb:53-64`

```ruby
def handle_mark_as_sensitive!
  mark_statuses_as_sensitive!  # 本地用 UpdateStatusService，远端直接更新
  mark_collections_as_sensitive!

  resolve_report!              # 关闭当前举报
  process_strike!(:mark_statuses_as_sensitive)
  process_notification!
end
```

---

## 四、自定义账号动作：带/不带举报上下文的差异

### 4.1 两种进入路径概述

#### 4.1.1 路径一：从举报详情页进入（带举报上下文）

**触发位置**: `app/views/admin/reports/_actions.html.haml:45-46`

```haml
= link_to t('admin.accounts.custom'),
          new_admin_account_action_path(report.target_account_id, report_id: report.id),
          class: 'button'
```

**URL 格式**: `/admin/accounts/:account_id/action/new?report_id=:report_id`

**关键点**: URL 中带有 `report_id` 参数

#### 4.1.2 路径二：从账号管理页进入（不带举报上下文）

**触发位置**: `app/views/admin/accounts/_buttons.html.haml`

```haml
# 发送警告（仅本地）
= link_to t('admin.accounts.warn'), 
          new_admin_account_action_path(account.id, type: 'none'), 
          class: 'button'

# 其他操作（disable/sensitive/silence/suspend）
= link_to t('admin.accounts.sensitive'), 
          new_admin_account_action_path(account.id, type: 'sensitive'), 
          class: 'button'
```

**URL 格式**: `/admin/accounts/:account_id/action/new` 或 `/admin/accounts/:account_id/action/new?type=xxx`

**关键点**: URL 中**没有** `report_id` 参数

### 4.2 控制器层面的差异

**控制器**: `app/controllers/admin/account_actions_controller.rb`

#### 4.2.1 初始化差异 (`new` 方法)

```ruby
def new
  authorize @account, :show?

  @account_action  = Admin::AccountAction.new(
    type: params[:type], 
    report_id: params[:report_id],  # ← 关键：从 URL 参数传入
    send_email_notification: true, 
    include_statuses: true
  )
  @warning_presets = AccountWarningPreset.all
end
```

#### 4.2.2 重定向差异 (`create` 方法)

```ruby
def create
  # ... 创建 AccountAction ...
  
  if @account_action.save
    if @account_action.with_report?
      # 带上下文 → 跳转到举报列表
      redirect_to admin_reports_path, 
                  notice: I18n.t('admin.reports.processed_msg', id: resource_params[:report_id])
    else
      # 不带上下文 → 跳转到账号详情页
      redirect_to admin_account_path(@account.id)
    end
  end
end
```

### 4.3 表单视图层面的差异

**视图**: `app/views/admin/account_actions/new.html.haml`

#### 4.3.1 隐藏字段

```haml
= simple_form_for @account_action, url: admin_account_action_path(@account.id) do |f|
  = f.input :report_id,
            as: :hidden  # ← report_id 作为隐藏字段传递
```

#### 4.3.2 条件显示的字段

```haml
- if @account.local?
  .fields-group
    = f.input :send_email_notification, as: :boolean

  - if params[:report_id].present?  # ← 仅带上下文时显示
    .fields-group
      = f.input :include_statuses, as: :boolean  # 是否包含举报的嘟文

  .fields-group
    = f.input :warning_preset_id, collection: @warning_presets

  .fields-group
    = f.input :text, as: :text
```

**字段显示差异表**:

| 字段 | 带举报上下文 | 不带举报上下文 |
|------|-------------|---------------|
| `type`（操作类型） | ✓ | ✓ |
| `send_email_notification` | ✓（仅本地） | ✓（仅本地） |
| `include_statuses` | ✓ | ✗ |
| `warning_preset_id` | ✓（仅本地） | ✓（仅本地） |
| `text` | ✓（仅本地） | ✓（仅本地） |

### 4.4 核心业务逻辑差异：哪些举报被关闭

**关键模型**: `app/models/admin/account_action.rb`

#### 4.4.1 基类方法

**基类**: `app/models/admin/base_action.rb:35-41`

```ruby
def report
  @report ||= Report.find(report_id) if report_id.present?
end

def with_report?
  !report.nil?  # report_id 存在且能找到 Report 记录
end
```

#### 4.4.2 决定要关闭哪些举报的关键逻辑

**核心方法**: `app/models/admin/account_action.rb:131-137`

```ruby
def reports
  @reports ||= if type == 'none'
                 # 类型为 none（仅警告）:
                 # - 带上下文 → 只关闭当前举报
                 # - 不带上下文 → 不关闭任何举报
                 with_report? ? [report] : []
               else
                 # 类型为其他（disable/sensitive/silence/suspend）:
                 # - 无论是否带上下文 → 关闭该账号所有未解决的举报
                 target_account.targeted_reports.unresolved
               end
end
```

**逻辑解析图**:

```
reports 方法调用
       ↓
type == 'none'?
       ├─→ YES (仅警告)
       │         ↓
       │    with_report?
       │         ├─→ YES → 返回 [report] → 只关闭当前举报
       │         └─→ NO  → 返回 [] → 不关闭任何举报
       │
       └─→ NO (disable/sensitive/silence/suspend)
                 ↓
            返回 target_account.targeted_reports.unresolved
                 ↓
            关闭该账号所有未解决的举报
```

#### 4.4.3 执行关闭逻辑

**方法**: `app/models/admin/account_action.rb:70-83`

```ruby
def process_reports!
  # If we're doing "mark as resolved" on a single report,
  # then we want to keep other reports open in case they
  # contain new actionable information.
  #
  # Otherwise, we will mark all unresolved reports about
  # the account as resolved.

  reports.each do |report|
    authorize(report, :update?)
    log_action(:resolve, report)      # ← 记录审计日志
    report.resolve!(current_account)  # ← 设置 action_taken_at
  end
end
```

### 4.5 差异汇总表

假设场景：账号 B 有 3 个未解决的举报 R1、R2、R3

| 操作类型 | 进入路径 | report_id | `reports` 返回值 | 被关闭的举报 |
|---------|---------|-----------|-----------------|-------------|
| `none`（仅警告） | 举报详情页 R1 | ✓ R1 | `[R1]` | **仅 R1** |
| `none`（仅警告） | 账号管理页 | ✗ | `[]` | **无** |
| `silence` | 举报详情页 R1 | ✓ R1 | `[R1, R2, R3]` | **全部 R1, R2, R3** |
| `silence` | 账号管理页 | ✗ | `[R1, R2, R3]` | **全部 R1, R2, R3** |
| `suspend` | 举报详情页 R1 | ✓ R1 | `[R1, R2, R3]` | **全部 R1, R2, R3** |
| `suspend` | 账号管理页 | ✗ | `[R1, R2, R3]` | **全部 R1, R2, R3** |

**关键发现**:
1. **只有 `type == 'none'`（仅警告）时，路径差异才影响举报关闭范围**
2. **实际的限制操作（silence/suspend/sensitive/disable）无论从哪进入，都会关闭该账号所有未解决的举报**
3. 这是因为：限制操作是对账号整体的处理，应该清理所有相关举报；而仅警告可以是针对单个举报的温和处理

### 4.6 审计记录的差异

#### 4.6.1 AccountWarning (Strike) 的关联

**创建逻辑**: `app/models/admin/base_action.rb:45-53`

```ruby
def process_strike!(action = type)
  @warning = target_account.strikes.create!(
    account: current_account,
    report: report,           # ← 关联的举报（可能为 nil）
    action:,
    text: text_for_warning,
    status_ids: status_ids    # ← 关联的嘟文
  )
end
```

**status_ids 的计算**: `app/models/admin/account_action.rb:127-129`

```ruby
def status_ids
  report.status_ids if with_report? && include_statuses?
  # 只有带上下文且勾选 include_statuses 时，才会关联嘟文
end
```

#### 4.6.2 审计记录点

**执行流程**: `app/models/admin/account_action.rb:45-55`

```ruby
def process_action!
  ApplicationRecord.transaction do
    handle_type!         # 1. 执行限制操作 → 记录操作日志
    process_strike!      # 2. 创建 AccountWarning
    create_log!          # 3. 仅 none 类型且有文本时，记录 warning 创建
    process_reports!     # 4. 关闭举报 → 每个关闭的举报都记录 :resolve
  end

  process_notification!
  process_queue!
end
```

**各类操作的审计记录**:

| 步骤 | 记录内容 | 条件 |
|------|---------|------|
| `handle_type!` | `:disable` → User<br>`:sensitive` → Account<br>`:silence` → Account<br>`:suspend` → Account | 总是 |
| `create_log!` | `:create` → AccountWarning | 仅 `type == 'none'` 且有自定义文本 |
| `process_reports!` | `:resolve` → Report | 每个被关闭的举报 |

#### 4.6.3 不同路径下的审计记录对比

**场景**: 账号 B 有 R1、R2、R3 三个未解决举报，从 R1 详情页或账号管理页执行操作

| 操作类型 | 路径 | 审计记录 | 说明 |
|---------|------|---------|------|
| `none` + 文本 | 举报详情页 R1 | 1. `:create` → Warning(R1)<br>2. `:resolve` → Report(R1) | 仅警告 + 关闭 R1 |
| `none` + 文本 | 账号管理页 | 1. `:create` → Warning(nil) | 仅警告，不关闭任何举报 |
| `silence` | 举报详情页 R1 | 1. `:silence` → Account(B)<br>2. `:resolve` → Report(R1)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R3) | 静音 + 关闭所有举报 |
| `silence` | 账号管理页 | 1. `:silence` → Account(B)<br>2. `:resolve` → Report(R1)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R3) | **完全相同** |
| `suspend` | 举报详情页 R1 | 1. `:suspend` → Account(B)<br>2. `:resolve` → Report(R1)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R3) | 暂停 + 关闭所有举报 |
| `suspend` | 账号管理页 | 1. `:suspend` → Account(B)<br>2. `:resolve` → Report(R1)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R3) | **完全相同** |

### 4.7 页面展示的差异

#### 4.7.1 举报详情页的审计视图

**视图**: `app/views/admin/reports/show.html.haml:89-105`

```haml
- unless @action_logs.empty?
  %h3= t 'admin.reports.action_log'
  .report-notes
    = render @action_logs       # Admin::ActionLog

%hr.spacer/

%h3= t 'admin.reports.notes.title'
.report-notes
  = render @report_notes        # ReportNote
```

**数据来源**: `app/controllers/admin/reports_controller.rb:15-17`

```ruby
def show
  @report_notes = @report.notes.chronological.includes(:account)  # 该举报的备注
  @action_logs  = @report.history.includes(:target)               # 该举报关联的操作日志
end
```

#### 4.7.2 `Report#history` 的组成

**方法**: `app/models/report.rb:138-162`

```ruby
def history
  subquery = [
    # 1. 举报本身的操作（分配、解决等）
    Admin::ActionLog.where(target_type: 'Report', target_id: id),
    
    # 2. 被举报账号的操作（静音、暂停等）
    Admin::ActionLog.where(target_type: 'Account', target_id: target_account_id),
    
    # 3. 被举报嘟文的操作
    Admin::ActionLog.where(target_type: 'Status', target_id: status_ids),
    
    # 4. 与该举报关联的警告
    Admin::ActionLog.where(
      target_type: 'AccountWarning',
      target_id: AccountWarning.where(report_id: id).select(:id)
    ),
  ].reduce { |union, query| Arel::Nodes::UnionAll.new(union, query) }

  Admin::ActionLog.latest.from(Arel::Nodes::As.new(subquery, Admin::ActionLog.arel_table))
end
```

#### 4.7.3 不同路径下的展示对比

**场景**: 从 R1 详情页进入执行 `none` 警告（带文本），vs 从账号管理页执行

| 内容 | 从 R1 进入执行 none | 从账号页进入执行 none |
|------|---------------------|---------------------|
| **R1 详情页 Action Logs** | 1. `:create` → Warning(R1)<br>2. `:resolve` → Report(R1) | **不会显示**（Warning 不关联 R1） |
| **R1 详情页 Notes** | 无变化 | 无变化 |
| **账号页 Strikes 列表** | 显示 Warning（关联 R1） | 显示 Warning（无关联举报） |
| **R1 的 AccountWarning 关联** | `warning.report_id = R1.id` | `warning.report_id = nil` |
| **R1 状态** | `action_taken_at: 现在` | `action_taken_at: nil`（仍未解决） |

### 4.8 最小可复核流程对照

#### 4.8.1 场景设定

**前置条件**:
- 账号 B 有 3 个未解决举报：R1、R2、R3
- R1 关联嘟文 S1、S2
- 管理员 M 执行操作

#### 4.8.2 流程一：从举报详情页 R1 执行仅警告 (`type=none`)

**执行步骤**:

| 步骤 | 操作 | 代码路径 | 预期结果 |
|------|------|---------|---------|
| 1 | 访问 R1 详情页点击"自定义" | `_actions.html.haml:45` | 跳转至 `/admin/accounts/B/action/new?report_id=R1` |
| 2 | 表单中 `report_id` 作为隐藏字段 | `new.html.haml:12-13` | `report_id = R1` |
| 3 | 选择操作类型 `none`，填写文本 | `new.html.haml:16-23` | `type = 'none'` |
| 4 | 勾选 `include_statuses`（显示因为有 report_id） | `new.html.haml:33-37` | `include_statuses = true` |
| 5 | 提交表单 | `account_actions_controller#create` | 创建 AccountAction |

**数据变化验证**:

| 验证项 | SQL 查询 | 预期值 |
|--------|----------|--------|
| AccountWarning 关联 | `SELECT report_id FROM account_warnings ORDER BY id DESC LIMIT 1` | `R1` |
| AccountWarning 嘟文 | `SELECT status_ids FROM account_warnings ORDER BY id DESC LIMIT 1` | `[S1, S2]` |
| R1 状态 | `SELECT action_taken_at FROM reports WHERE id = R1` | **NOT NULL** |
| R2 状态 | `SELECT action_taken_at FROM reports WHERE id = R2` | **NULL** |
| R3 状态 | `SELECT action_taken_at FROM reports WHERE id = R3` | **NULL** |
| Action Log 数量 | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'Report' AND target_id IN (R1,R2,R3)` | **1**（仅 R1） |

**R1 详情页展示**:
```
操作日志 (Action Logs):
  1. admin 创建了警告 #W1
  2. admin 解决了举报 #R1

备注 (Notes):
  (空)
```

#### 4.8.3 流程二：从账号管理页执行仅警告 (`type=none`)

**执行步骤**:

| 步骤 | 操作 | 代码路径 | 预期结果 |
|------|------|---------|---------|
| 1 | 访问账号 B 详情页点击"发送警告" | `_buttons.html.haml:15` | 跳转至 `/admin/accounts/B/action/new?type=none` |
| 2 | 表单中无 `report_id` 隐藏字段 | `new.html.haml:12-13` | `report_id = nil` |
| 3 | 操作类型已预设为 `none` | `new.html.haml:16-23` | `type = 'none'` |
| 4 | **不显示** `include_statuses` 选项 | `new.html.haml:33-37` | 条件 `params[:report_id].present?` 不满足 |
| 5 | 提交表单 | `account_actions_controller#create` | 创建 AccountAction |

**数据变化验证**:

| 验证项 | SQL 查询 | 预期值 |
|--------|----------|--------|
| AccountWarning 关联 | `SELECT report_id FROM account_warnings ORDER BY id DESC LIMIT 1` | **NULL** |
| AccountWarning 嘟文 | `SELECT status_ids FROM account_warnings ORDER BY id DESC LIMIT 1` | **NULL** |
| R1 状态 | `SELECT action_taken_at FROM reports WHERE id = R1` | **NULL**（仍未解决） |
| R2 状态 | `SELECT action_taken_at FROM reports WHERE id = R2` | **NULL** |
| R3 状态 | `SELECT action_taken_at FROM reports WHERE id = R3` | **NULL** |
| Action Log 数量 | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'Report' AND target_id IN (R1,R2,R3)` | **0** |

**R1 详情页展示**:
```
操作日志 (Action Logs):
  (空，因为 Warning 不关联 R1)

备注 (Notes):
  (空)
```

**账号页 Strikes 列表展示**:
```
之前的警告 (Previous Strikes):
  · 警告 - 无关联举报（自定义文本内容）
```

#### 4.8.4 流程三：从举报详情页 R1 执行静音 (`type=silence`)

**执行步骤**:

| 步骤 | 操作 | 代码路径 | 预期结果 |
|------|------|---------|---------|
| 1 | 访问 R1 详情页点击"自定义" | `_actions.html.haml:45` | 跳转至 `/admin/accounts/B/action/new?report_id=R1` |
| 2 | 选择操作类型 `silence` | `new.html.haml:16-23` | `type = 'silence'` |
| 3 | 勾选 `include_statuses` | `new.html.haml:33-37` | `include_statuses = true` |
| 4 | 提交表单 | `account_actions_controller#create` | 创建 AccountAction |

**数据变化验证**:

| 验证项 | SQL 查询 | 预期值 |
|--------|----------|--------|
| 账号状态 | `SELECT silenced_at FROM accounts WHERE id = B` | **NOT NULL** |
| AccountWarning 关联 | `SELECT report_id FROM account_warnings ORDER BY id DESC LIMIT 1` | `R1` |
| AccountWarning 嘟文 | `SELECT status_ids FROM account_warnings ORDER BY id DESC LIMIT 1` | `[S1, S2]` |
| R1 状态 | `SELECT action_taken_at FROM reports WHERE id = R1` | **NOT NULL** |
| R2 状态 | `SELECT action_taken_at FROM reports WHERE id = R2` | **NOT NULL** |
| R3 状态 | `SELECT action_taken_at FROM reports WHERE id = R3` | **NOT NULL** |
| Action Log 数量 | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'Report' AND target_id IN (R1,R2,R3)` | **3**（全部关闭） |

**R1 详情页展示**:
```
操作日志 (Action Logs):
  1. admin 静音了账号 @bob@example.com
  2. admin 解决了举报 #R3
  3. admin 解决了举报 #R2
  4. admin 解决了举报 #R1
  (按倒序排列，最新的在最前)

备注 (Notes):
  (空)
```

#### 4.8.5 流程四：从账号管理页执行静音 (`type=silence`)

**执行步骤**:

| 步骤 | 操作 | 代码路径 | 预期结果 |
|------|------|---------|---------|
| 1 | 访问账号 B 详情页点击"静音" | `_buttons.html.haml:27` | 跳转至 `/admin/accounts/B/action/new?type=silence` |
| 2 | 操作类型已预设为 `silence` | `new.html.haml:16-23` | `type = 'silence'` |
| 3 | **不显示** `include_statuses` 选项 | `new.html.haml:33-37` | 无 report_id |
| 4 | 提交表单 | `account_actions_controller#create` | 创建 AccountAction |

**数据变化验证**:

| 验证项 | SQL 查询 | 预期值 |
|--------|----------|--------|
| 账号状态 | `SELECT silenced_at FROM accounts WHERE id = B` | **NOT NULL** |
| AccountWarning 关联 | `SELECT report_id FROM account_warnings ORDER BY id DESC LIMIT 1` | **NULL** |
| AccountWarning 嘟文 | `SELECT status_ids FROM account_warnings ORDER BY id DESC LIMIT 1` | **NULL** |
| R1 状态 | `SELECT action_taken_at FROM reports WHERE id = R1` | **NOT NULL** |
| R2 状态 | `SELECT action_taken_at FROM reports WHERE id = R2` | **NOT NULL** |
| R3 状态 | `SELECT action_taken_at FROM reports WHERE id = R3` | **NOT NULL** |
| Action Log 数量 | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'Report' AND target_id IN (R1,R2,R3)` | **3**（全部关闭） |

**R1 详情页展示**:
```
操作日志 (Action Logs):
  1. admin 静音了账号 @bob@example.com
  2. admin 解决了举报 #R3
  3. admin 解决了举报 #R2
  4. admin 解决了举报 #R1
  (与流程三完全相同)

备注 (Notes):
  (空)
```

#### 4.8.6 四种流程对比总结

| 对比项 | 流程一<br>举报页+none | 流程二<br>账号页+none | 流程三<br>举报页+silence | 流程四<br>账号页+silence |
|--------|---------------------|---------------------|-------------------------|-------------------------|
| `report_id` | ✓ R1 | ✗ | ✓ R1 | ✗ |
| 账号限制 | 无 | 无 | 静音 | 静音 |
| 关闭举报数 | 仅 R1 | 无 | R1+R2+R3 | R1+R2+R3 |
| Warning 关联 R1 | ✓ | ✗ | ✓ | ✗ |
| Warning 含嘟文 | ✓ S1,S2 | ✗ | ✓ S1,S2 | ✗ |
| R1 详情页日志 | 显示 Warning 创建+R1 解决 | 不显示 | 显示静音+3 个举报解决 | 显示静音+3 个举报解决 |
| 重定向目标 | `/admin/reports` | `/admin/accounts/B` | `/admin/reports` | `/admin/accounts/B` |

---

## 五、审计记录机制

### 5.1 审计视图的组成

#### 5.1.1 举报详情页审计区域

**视图**: `app/views/admin/reports/show.html.haml:89-105`

```haml
- unless @action_logs.empty?
  %hr.spacer/

  %h3= t 'admin.reports.action_log'      # 操作日志

  .report-notes
    = render @action_logs                 # Admin::ActionLog 列表

%hr.spacer/

%h3= t 'admin.reports.notes.title'        # 备注

%p= t 'admin.reports.notes_description_html'

.report-notes
  = render @report_notes                  # ReportNote 列表
```

**举报页审计区域包含两部分**:
1. **Action Logs (操作日志)**: 结构化的管理员操作记录
2. **Notes (备注)**: 管理员自由文本备注

#### 5.1.2 操作日志 (Action Logs)

**控制器赋值**: `app/controllers/admin/reports_controller.rb:17`

```ruby
def show
  authorize @report, :show?

  @report_note  = @report.notes.new
  @report_notes = @report.notes.chronological.includes(:account)
  @action_logs  = @report.history.includes(:target)  # ← 操作日志
  @form         = Admin::StatusBatchAction.new
  @statuses     = @report.statuses.with_includes
end
```

**历史查询**: `app/models/report.rb:138-162`

```ruby
def history
  subquery = [
    # 1. 举报本身的操作
    Admin::ActionLog.where(target_type: 'Report', target_id: id),
    
    # 2. 被举报账号的操作
    Admin::ActionLog.where(target_type: 'Account', target_id: target_account_id),
    
    # 3. 被举报嘟文的操作
    Admin::ActionLog.where(target_type: 'Status', target_id: status_ids),
    
    # 4. 与该举报关联的警告记录
    Admin::ActionLog.where(
      target_type: 'AccountWarning',
      target_id: AccountWarning.where(report_id: id).select(:id)
    ),
  ].reduce { |union, query| Arel::Nodes::UnionAll.new(union, query) }

  Admin::ActionLog.latest.from(Arel::Nodes::As.new(subquery, Admin::ActionLog.arel_table))
end
```

**操作日志包含的内容**:

| 关联类型 | target_type | 记录的操作 |
|---------|-------------|-----------|
| 举报本身 | `Report` | assigned_to_self, unassigned, reopen, resolve |
| 目标账号 | `Account` | sensitive, silence, suspend, unsensitive, unsilence, unsuspend, memorialize 等 |
| 关联嘟文 | `Status` | destroy, update |
| 关联警告 | `AccountWarning` | create |

#### 5.1.3 备注 (Notes)

**举报备注模型**: `app/models/report_note.rb`

```ruby
class ReportNote < ApplicationRecord
  CONTENT_SIZE_LIMIT = 2_000

  belongs_to :account           # 撰写备注的管理员
  belongs_to :report, inverse_of: :notes, touch: true

  scope :chronological, -> { reorder(id: :asc) }  # 按时间正序排列

  validates :content, presence: true, length: { maximum: CONTENT_SIZE_LIMIT }
end
```

**举报备注控制器**: `app/controllers/admin/report_notes_controller.rb:7-31`

```ruby
def create
  authorize :report_note, :create?

  @report_note = current_account.report_notes.new(resource_params)
  @report      = @report_note.report

  if @report_note.save
    if params[:create_and_resolve]
      @report.resolve!(current_account)   # 创建备注并解决举报
      log_action :resolve, @report
    elsif params[:create_and_unresolve]
      @report.unresolve!                  # 创建备注并重新打开举报
      log_action :reopen, @report
    end

    redirect_to after_create_redirect_path, notice: I18n.t('admin.report_notes.created_msg')
  else
    # 重新渲染页面...
  end
end
```

**举报备注的三种操作**:

| 按钮 | 行为 |
|------|------|
| 创建并解决 | 创建备注 + 关闭举报 + 记录 resolve 操作日志 |
| 创建并重新打开 | 创建备注 + 重新打开举报 + 记录 reopen 操作日志 |
| 仅创建 | 仅创建备注，不改变举报状态 |

#### 5.1.4 账号管理页的备注系统

**账号审核备注模型**: `app/models/account_moderation_note.rb`

```ruby
class AccountModerationNote < ApplicationRecord
  CONTENT_SIZE_LIMIT = 2_000

  belongs_to :account           # 撰写备注的管理员
  belongs_to :target_account, class_name: 'Account'  # 被备注的账号

  scope :chronological, -> { reorder(id: :asc) }

  validates :content, presence: true, length: { maximum: CONTENT_SIZE_LIMIT }
end
```

**控制器**: `app/controllers/admin/account_moderation_notes_controller.rb`

与举报备注不同，账号备注：
- 不关联具体举报
- 不改变账号状态
- 仅用于记录账号相关的审核信息

#### 5.1.5 Notes 与 Action Logs 对比

| 特性 | Notes (备注) | Action Logs (操作日志) |
|------|-------------|----------------------|
| **类型** | ReportNote / AccountModerationNote | Admin::ActionLog |
| **创建方式** | 管理员主动填写 | 操作自动触发 |
| **内容** | 自由文本（最多 2000 字） | 结构化（action + target 多态关联） |
| **是否可删除** | ✓ 管理员可删除 | ✗ 不可删除 |
| **显示区域** | 独立的 Notes 区块 | Action Log 区块 |
| **排序** | chronological (正序，旧→新) | latest (倒序，新→旧) |
| **关联对象** | 举报 或 账号 | 任意多态对象 |
| **审计覆盖** | 人工补充信息 | 系统自动记录所有操作 |

### 5.2 审计记录模型

**模型**: `app/models/admin/action_log.rb`

**存储表**: `admin_action_logs`

**关键字段**:

| 字段 | 说明 |
|------|------|
| `account_id` | 执行操作的管理员账号 |
| `action` | 操作类型（字符串，如 :resolve, :suspend, :destroy） |
| `target_type` | 目标对象类型（多态） |
| `target_id` | 目标对象 ID |
| `human_identifier` | 人类可读的目标标识 |
| `permalink` | 目标对象链接 |
| `route_param` | 路由参数 |
| `created_at` | 操作时间 |

### 5.3 记录创建方式

**Concern**: `app/controllers/concerns/accountable_concern.rb`

```ruby
def log_action(action, target)
  current_account
    .action_logs
    .create(action:, target:)
end
```

**自动填充字段** (before_validation 钩子):
1. `set_human_identifier`: 调用目标对象的 `to_log_human_identifier`
2. `set_route_param`: 调用目标对象的 `to_log_route_param`
3. `set_permalink`: 调用目标对象的 `to_log_permalink`

### 5.4 审计记录点总览

#### 5.4.1 举报相关的记录点

**举报控制器**: `app/controllers/admin/reports_controller.rb`

| 操作 | action 值 | 目标类型 |
|------|----------|---------|
| 分配给自己 | `:assigned_to_self` | Report |
| 取消分配 | `:unassigned` | Report |
| 重新打开 | `:reopen` | Report |
| 标记解决 | `:resolve` | Report |

**举报备注控制器**: `app/controllers/admin/report_notes_controller.rb`

| 操作 | action 值 | 目标类型 |
|------|----------|---------|
| 创建备注并解决 | `:resolve` | Report |
| 创建备注并重新打开 | `:reopen` | Report |

#### 5.4.2 账号操作相关的记录点

**账号操作模型**: `app/models/admin/account_action.rb`

| 操作类型 | action 值 | 目标类型 |
|---------|----------|---------|
| 禁用账号 | `:disable` | User |
| 标记敏感 | `:sensitive` | Account |
| 静音 | `:silence` | Account |
| 暂停 | `:suspend` | Account |
| 解决关联举报 | `:resolve` | Report |
| 创建警告（仅none且有文本） | `:create` | AccountWarning |

**账号控制器**: `app/controllers/admin/accounts_controller.rb`

| 操作 | action 值 | 目标类型 |
|------|----------|---------|
| 设置纪念账号 | `:memorialize` | Account |
| 启用账号 | `:enable` | User |
| 批准注册 | `:approve` | User |
| 拒绝注册 | `:reject` | User |
| 撤销敏感 | `:unsensitive` | Account |
| 撤销静音 | `:unsilence` | Account |
| 撤销暂停 | `:unsuspend` | Account |
| 移除头像 | `:remove_avatar` | User |
| 移除头部 | `:remove_header` | User |
| 解除邮箱封锁 | `:unblock_email` | Account |

#### 5.4.3 嘟文操作相关的记录点

**嘟文操作模型**: `app/models/admin/moderation_action.rb`

| 操作 | action 值 | 目标类型 |
|------|----------|---------|
| 删除嘟文 | `:destroy` | Status |
| 删除集合 | `:destroy` | Collection |
| 更新嘟文（标记敏感） | `:update` | Status |
| 更新集合（标记敏感） | `:update` | Collection |
| 解决举报 | `:resolve` | Report |

---

## 六、本地审核与远端账号处理的边界

### 6.1 本地/远端账号判断标准

**判断逻辑**: `app/models/account.rb:208-214`

```ruby
def local?
  domain.nil?  # 无域名 → 本地账号
end

def remote?
  !domain.nil? # 有域名 → 远端账号
end
```

### 6.2 举报阶段的差异

**举报转发机制**: `app/services/report_service.rb:71-81`

```ruby
def forward?
  !@target_account.local? && ActiveModel::Type::Boolean.new.cast(@options[:forward])
end

def forward_to_origin?
  forward? && forward_to_domains.include?(@target_account.domain)
end
```
- **本地目标**: 不转发
- **远端目标**: 用户可选择将举报转发到原实例（通过 ActivityPub 的 Flag 活动）

**嘟文关联处理**: `app/services/report_service.rb:83-94`
- **本地举报人**: 使用严格的可见性过滤 `AccountStatusesFilter`
- **远端举报人**: 放宽可见性检查（因为可能已匿名化）

### 6.3 审核操作的边界

#### 6.3.1 可用操作类型差异

| 操作类型 | 本地账号 | 远端账号 | 说明 |
|---------|---------|---------|------|
| `none` | ✓ | ✗ | 远端账号无法仅发送警告 |
| `disable` | ✓ | ✗ | 远端账号无本地 User 记录 |
| `sensitive` | ✓ | ✓ | 仅影响本地显示 |
| `silence` | ✓ | ✓ | 仅影响本地显示和传播 |
| `suspend` | ✓ | ✓ | 本地封禁，远端账号仍可在原实例使用 |

#### 6.3.2 暂停操作的深层差异

**暂停来源标记**: `app/models/concerns/account/suspensions.rb:16-18`

```ruby
def suspended_locally?
  suspended? && suspension_origin_local?
end
```

**远端账号暂停的限制**: `app/services/suspend_account_service.rb:23-42`

```ruby
def reject_remote_follows!
  return if @account.local? || !@account.activitypub? || @account.suspension_origin_remote?
  
  # 注释说明：
  # 当暂停远端账号时，该账号在其原始实例上实际上并没有被暂停。
  # 也就是说，与本地暂停的账号不同，它继续可以访问其首页和其他内容。
  # 为了防止它能够继续访问因为它关注本地账号而收到的嘟文，
  # 我们必须强制它取消关注这些账号。
  # 不幸的是，此操作没有对应操作，即你不能强制远端账号重新关注你，
  # 所以这部分是不可逆的。
end
```

**本地账号暂停的联邦广播**: `app/services/suspend_account_service.rb:44-52`

```ruby
def distribute_update_actor!
  return unless @account.local?  # 仅本地账号向联邦网络广播
  # 向所有已知实例发送 Update Actor 活动
end
```

#### 6.3.3 嘟文处理差异

**删除操作**: `app/models/admin/moderation_action.rb:45-51`

```ruby
# 远端账号：创建墓碑记录
create_tombstones! unless target_account.local?

# 后台清理参数差异
RemovalWorker.push_bulk(status_ids) { |status_id| 
  [status_id, { 'preserve' => target_account.local?, 'immediate' => !target_account.local? }]
}
```
- 本地账号: `preserve: true` - 保留数据用于恢复
- 远端账号: `immediate: true` - 立即清理

**标记敏感操作**: `app/models/admin/moderation_action.rb:92-96`

```ruby
if target_account.local?
  UpdateStatusService.new.call(status, representative_account.id, sensitive: true)
else
  status.update(sensitive: true)  # 仅更新本地记录
end
```
- 本地账号: 调用完整服务，可能触发联邦更新
- 远端账号: 直接更新本地数据库，不影响原实例

#### 6.3.4 通知差异

**警告通知**: `app/models/admin/base_action.rb:62-64`

```ruby
def warnable?
  send_email_notification? && target_account.local?  # 仅本地账号能收到通知
end
```
- 本地账号: 发送邮件和站内通知
- 远端账号: 无法发送通知（没有本地 User 记录）

### 6.4 处理边界总结

| 能力维度 | 本地账号 | 远端账号 |
|---------|---------|---------|
| **数据所有权** | 完整控制 | 仅本地缓存副本 |
| **账号禁用** | 可禁用登录 | 无法禁用（不在本实例） |
| **联邦广播** | 可向全网广播操作 | 无法向原实例推送 |
| **邮件通知** | 可发送 | 无法发送 |
| **操作可逆性** | 多数操作可逆 | 部分不可逆（如强制取消关注） |
| **举报转发** | 不需要 | 用户可选转发到原实例 |
| **实际限制范围** | 全局有效 | 仅在本实例生效 |

**核心原则**:
1. 本地管理员对本地账号拥有完整控制权
2. 本地管理员对远端账号的限制仅在本实例生效
3. 远端账号在其原实例上的行为不受本地操作影响
4. ActivityPub 协议提供了协作机制（举报转发），但不强制远端实例执行

---

## 七、端到端处理时间线

### 7.1 场景设定

**角色**:
- 用户 A: 本地用户 (@alice@local.example)
- 用户 B: 远端用户 (@bob@remote.example)
- 管理员 M: 本地实例管理员 (@admin@local.example)

**事件**: 用户 A 举报了用户 B 的一条回复性嘟文，管理员 M 处理举报并暂停账号 B。

---

### 7.2 详细时间线

#### T1: 用户提交举报

**操作**: 用户 A 在前端点击"举报"，选择转发到原实例

**代码路径**:
1. **前端请求**: `POST /api/v1/reports`
   - 参数: `account_id` (B), `status_ids` ([S2]), `forward: true`

2. **控制器**: `app/controllers/api/v1/reports_controller.rb:9-17`
   ```ruby
   @report = ReportService.new.call(
     current_account,      # A
     reported_account,     # B
     report_params.merge(application: doorkeeper_token.application)
   )
   ```

3. **服务层**: `app/services/report_service.rb:6-28`
   ```ruby
   def call(source_account, target_account, options = {})
     create_report!     # 创建 Report 记录
     notify_staff!      # 通知管理员
     
     if forward?
       forward_to_origin!     # 转发到 B 的原实例
       forward_to_replied_to! # 转发到回复链上的其他实例
     end
   end
   ```

**此时系统状态**:
- Report 表新增一条记录 (id=R1)
  - `account_id`: A
  - `target_account_id`: B
  - `status_ids`: [S2]
  - `action_taken_at`: nil (未处理)
  - `forwarded`: true
- 管理员 M 收到站内通知和邮件
- ActivityPub::DeliveryWorker 排队发送 Flag 到:
  - remote.example 的 inbox (B 的原实例)
  - S2 回复对象所在实例的 inbox (回复链实例)

---

#### T2: 管理员查看举报队列

**操作**: 管理员 M 登录后台，访问 `/admin/reports`

**代码路径**:
1. **控制器**: `app/controllers/admin/reports_controller.rb:7-10`
   ```ruby
   def index
     authorize :report, :index?
     @reports = filtered_reports.page(params[:page])
   end
   ```

2. **过滤逻辑**: `app/models/report_filter.rb:18-26`
   ```ruby
   def results
     scope = Report.unresolved  # 默认只显示未处理
     # 应用过滤条件
   end
   ```

**此时系统状态**:
- 列表显示 R1（因为 `action_taken_at` 为 nil）
- 可以看到: 举报人 A、被举报人 B、举报的嘟文 S2、已转发标记

---

#### T3: 管理员查看举报详情并添加备注

**操作**: 管理员 M 点击 R1，查看详情后添加备注

**代码路径**:
1. **控制器**: `app/controllers/admin/reports_controller.rb:12-20`
   ```ruby
   def show
     @report_notes = @report.notes.chronological.includes(:account)  # 空
     @action_logs  = @report.history.includes(:target)               # 空
   end
   ```

2. **创建备注**: `POST /admin/report_notes`
   - 控制器: `app/controllers/admin/report_notes_controller.rb:7-31`
   ```ruby
   def create
     @report_note = current_account.report_notes.new(resource_params)
     if @report_note.save
       # 选择"仅创建备注"，不改变举报状态
     end
   end
   ```

**此时系统状态**:
- ReportNote 表新增记录 (id=N1)
  - `account_id`: M
  - `report_id`: R1
  - `content`: "需要进一步调查..."
- Admin::ActionLog: 无新增（仅创建备注不记录操作日志）
- 举报 R1 状态: 仍未解决

---

#### T4: 管理员分配举报并执行暂停

**操作**: 管理员 M 将举报分配给自己，然后点击"暂停"按钮

**代码路径**:
1. **分配给自己**: `POST /admin/reports/:id/assign_to_self`
   - 控制器: `app/controllers/admin/reports_controller.rb:22-27`
   ```ruby
   def assign_to_self
     @report.update!(assigned_account_id: current_account.id)
     log_action :assigned_to_self, @report  # ← 记录操作日志
   end
   ```

2. **点击暂停按钮**: 提交到 `POST /admin/reports/:report_id/actions/preview`
   - 控制器: `app/controllers/admin/reports/actions_controller.rb:6-9`
   ```ruby
   def preview
     authorize @report, :show?
     @moderation_action = 'suspend'  # 从按钮判断
   end
   ```

3. **预览页确认**: `app/views/admin/reports/actions/preview.html.haml`
   - 显示: 将暂停账号 B、关闭该账号所有未解决举报、记录 strike、发送邮件（仅本地）

4. **确认执行**: `POST /admin/reports/:report_id/actions`
   - 控制器: `app/controllers/admin/reports/actions_controller.rb:11-24`
   ```ruby
   def create
     case action_from_button
     when 'silence', 'suspend'
       Admin::AccountAction.new(account_action_params).save!
     end
   end
   ```

5. **账号操作处理**: `app/models/admin/account_action.rb:45-55`
   ```ruby
   def process_action!
     ApplicationRecord.transaction do
       handle_type!         # 执行 suspend!
       process_strike!      # 创建 AccountWarning
       create_log!          # 记录警告创建（如有文本）
       process_reports!     # 关闭该账号所有未解决举报
     end
     process_notification!  # 发送通知（仅本地）
     process_queue!         # 启动后台任务
   end
   ```

6. **执行暂停**: `app/models/admin/account_action.rb:103-107`
   ```ruby
   def handle_suspend!
     authorize(target_account, :suspend?)
     log_action(:suspend, target_account)        # ← 记录暂停操作
     target_account.suspend!(origin: :local)     # 设置 suspended_at
   end
   ```

7. **关闭关联举报**: `app/models/admin/account_action.rb:70-83`
   ```ruby
   def process_reports!
     reports.each do |report|                    # 该账号所有未解决举报
       authorize(report, :update?)
       log_action(:resolve, report)              # ← 记录举报解决
       report.resolve!(current_account)          # 设置 action_taken_at
     end
   end
   ```

8. **后台任务**: `app/models/admin/account_action.rb:123-125`
   ```ruby
   def process_queue!
     queue_suspension_worker! if type == 'suspend'
   end
   ```

**此时系统状态**:
- Account 表 (B):
  - `suspended_at`: 当前时间
  - `suspension_origin`: 'local'
- Admin::ActionLog 表新增 3 条记录:
  1. `action: :assigned_to_self`, `target: Report#R1`
  2. `action: :suspend`, `target: Account#B`
  3. `action: :resolve`, `target: Report#R1`
- AccountWarning 表新增记录 (strike)
  - `account_id`: M
  - `target_account_id`: B
  - `report_id`: R1
  - `action`: 'suspend'
- Report 表 (R1):
  - `action_taken_at`: 当前时间
  - `action_taken_by_account_id`: M
- Admin::SuspensionWorker 排队执行
- 通知: 无（B 是远端账号，无法发送邮件）

---

#### T5: 后台任务执行暂停清理

**操作**: `Admin::SuspensionWorker` 执行

**代码路径**:
1. **Worker**: 调用 `SuspendAccountService.call(B)`

2. **服务**: `app/services/suspend_account_service.rb:8-19`
   ```ruby
   def call(account)
     return unless account.suspended?
     
     reject_remote_follows!    # 强制 B 取消关注本地账号（不可逆）
     distribute_update_actor!  # 跳过（B 是远端账号）
     unmerge_from_home_timelines!  # 从本地时间线移除 B 的内容
     unmerge_from_list_timelines!  # 从列表移除
     privatize_media_attachments!  # 媒体设为私有
     remove_from_trends!       # 从趋势移除
   end
   ```

**此时系统状态**:
- B 的所有本地粉丝的时间线已移除 B 的嘟文
- B 关注的所有本地账号已被强制取消关注（B 无法再接收这些账号的新嘟文）
- B 的媒体附件已设为私有
- B 已从趋势中移除

---

#### T6: 管理员验证处理结果

**操作**: 管理员 M 再次查看 R1 详情页

**代码路径**:
1. **控制器**: `app/controllers/admin/reports_controller.rb:12-20`
   ```ruby
   def show
     @report_notes = @report.notes.chronological.includes(:account)  # [N1]
     @action_logs  = @report.history.includes(:target)               # 包含所有关联操作
   end
   ```

2. **视图**: `app/views/admin/reports/show.html.haml:89-105`
   - Action Log 区块显示:
     1. M 暂停了账号 B
     2. M 解决了举报 R1
     3. M 将举报 R1 分配给自己
   - Notes 区块显示:
     1. N1: "需要进一步调查..."

**此时审计视图显示**:

```
┌─────────────────────────────────────────────────────┐
│  操作日志 (Action Logs)                              │
├─────────────────────────────────────────────────────┤
│  T4 · admin 暂停了账号 @bob@remote.example          │
│  T4 · admin 解决了举报 #R1                           │
│  T3 · admin 将举报 #R1 分配给自己                    │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  备注 (Notes)                                        │
├─────────────────────────────────────────────────────┤
│  T3 · admin: 需要进一步调查...                        │
└─────────────────────────────────────────────────────┘
```

---

### 7.3 时间线可视化

```
时间轴: T1 ────────── T2 ───────── T3 ───────── T4 ───────── T5 ───────── T6
         │              │               │              │              │              │
         ▼              ▼               ▼              ▼              ▼              ▼
   用户A举报     管理员查看     管理员添加     管理员暂停     后台清理     管理员验证
   (创建Report)  (未解决列表)   (ReportNote)   (AccountAction) (SuspendService)  (审计视图)
         │              │               │              │              │              │
         │              │               │              ├─→ 暂停账号 B   │              │
         │              │               │              ├─→ 记录操作日志 │              │
         ├─→ 通知管理员  │               │              ├─→ 关闭举报 R1  │              │
         ├─→ 转发Flag    │               │              └─→ 创建 Strike ─┼─→ 时间线清理  │
         └─→ 转发回复链  │               │                              └─→ 强制取关   │
                        └─→ 显示R1       └─→ 仅创建备注 ──────────────────────────────┘
                                       (不改变状态)        (自动记录)
```

### 7.4 关键时间点的数据变化

| 时间点 | Report#R1 | ActionLog | ReportNote | Account#B |
|-------|-----------|-----------|------------|-----------|
| T1 | `action_taken_at: nil` | 空 | 空 | 正常 |
| T3 | `action_taken_at: nil` | 空 | N1 已创建 | 正常 |
| T4 | `action_taken_at: 现在` | 3 条记录 | N1 | `suspended_at: 现在` |
| T5 | 不变 | 不变 | 不变 | 后台清理完成 |
| T6 | 不变 | 不变 | 不变 | 已暂停 |

### 7.5 远端账号 B 的实际状态

**在本地实例 (local.example)**:
- B 已被暂停 (`suspended_at` 已设置)
- B 的嘟文不会出现在任何本地用户的时间线
- B 无法再接收本地账号的新嘟文（已被强制取消关注）
- B 的媒体附件已私有

**在原实例 (remote.example)**:
- B 完全不受影响
- B 可以正常登录、发布嘟文、与其他用户互动
- B 甚至不知道自己在 local.example 被暂停了

**这就是联邦社交网络的处理边界**：
> 本地管理员的决定只在本地实例生效，远端实例拥有完全的自主权。ActivityPub 提供了协作机制（如举报转发），但不强制任何实例必须执行。
