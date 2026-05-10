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

## 三、自定义账号动作：本地账号 vs 远端账号

### 3.1 账号类型判断标准

**判断逻辑**: `app/models/account.rb:208-214`

```ruby
def local?
  domain.nil?  # 无域名 → 本地账号
end

def remote?
  !domain.nil? # 有域名 → 远端账号
end
```

### 3.2 动作类型可用性差异

**核心方法**: `app/models/admin/account_action.rb:22-28`

```ruby
def types_for_account(account)
  if account.local?
    TYPES  # ['none', 'disable', 'sensitive', 'silence', 'suspend']
  else
    TYPES - %w(none disable)  # ['sensitive', 'silence', 'suspend']
  end
end
```

**动作类型可用性对比**:

| 动作类型 | 内部类型 | 本地账号 | 远端账号 | 说明 |
|---------|---------|---------|---------|------|
| 仅警告 | `none` | ✓ | ✗ | 仅本地账号可发送警告 |
| 禁用账号 | `disable` | ✓ | ✗ | 远端账号无本地 User 记录 |
| 标记敏感 | `sensitive` | ✓ | ✓ | 仅影响本地显示 |
| 静音 | `silence` | ✓ | ✓ | 仅影响本地显示和传播 |
| 暂停/封禁 | `suspend` | ✓ | ✓ | 本地封禁，远端不受影响 |

### 3.3 进入路径：举报详情页 vs 账号管理页

#### 3.3.1 路径一：从举报详情页进入（带举报上下文）

**触发位置**: `app/views/admin/reports/_actions.html.haml:45-46`

```haml
= link_to t('admin.accounts.custom'),
          new_admin_account_action_path(report.target_account_id, report_id: report.id),
          class: 'button'
```

**URL 格式**: `/admin/accounts/:account_id/action/new?report_id=:report_id`

**关键点**: URL 中带有 `report_id` 参数

#### 3.3.2 路径二：从账号管理页进入（不带举报上下文）

**触发位置**: `app/views/admin/accounts/_buttons.html.haml`

**本地账号可触发的动作**:
```haml
- if account.local? && account.user_approved?
  = link_to t('admin.accounts.warn'), new_admin_account_action_path(account.id, type: 'none'), class: 'button'
  = link_to t('admin.accounts.disable'), new_admin_account_action_path(account.id, type: 'disable'), class: 'button'
- if !account.local? || account.user_approved?
  = link_to t('admin.accounts.sensitive'), new_admin_account_action_path(account.id, type: 'sensitive'), class: 'button'
  = link_to t('admin.accounts.silence'), new_admin_account_action_path(account.id, type: 'silence'), class: 'button'
  = link_to t('admin.accounts.perform_full_suspension'), new_admin_account_action_path(account.id, type: 'suspend'), class: 'button'
```

**URL 格式**: `/admin/accounts/:account_id/action/new?type=xxx`

**关键点**: URL 中**没有** `report_id` 参数

---

## 四、本地账号：自定义动作详细分析

### 4.1 可用动作类型

本地账号支持全部 5 种动作：
- `none`（仅警告）
- `disable`（禁用账号）
- `sensitive`（标记敏感）
- `silence`（静音）
- `suspend`（暂停/封禁）

### 4.2 表单字段展示对比

**视图**: `app/views/admin/account_actions/new.html.haml`

**控制器初始化**: `app/controllers/admin/account_actions_controller.rb:7-12`

```ruby
def new
  @account_action = Admin::AccountAction.new(
    type: params[:type],
    report_id: params[:report_id],
    send_email_notification: true,
    include_statuses: true
  )
  @warning_presets = AccountWarningPreset.all
end
```

**表单字段逐项对比**：

| 字段 | 带举报上下文 (report_id=R1) | 不带举报上下文 | 代码位置 |
|------|---------------------------|---------------|---------|
| `report_id` 隐藏字段 | ✓ (值为 R1) | ✓ (值为 nil) | `new.html.haml:12-13` |
| `type` 操作类型 | ✓ (可选全部 5 种) | ✓ (可选全部 5 种) | `new.html.haml:15-23` |
| `send_email_notification` | ✓ (默认勾选) | ✓ (默认勾选) | `new.html.haml:28-31` |
| `include_statuses` | ✓ (条件显示) | ✗ (不显示) | `new.html.haml:33-37` |
| `warning_preset_id` | ✓ (可选预设) | ✓ (可选预设) | `new.html.haml:41-46` |
| `text` 自定义文本 | ✓ (可填写) | ✓ (可填写) | `new.html.haml:48-52` |

**关键差异代码**:

```haml
- if @account.local?
  .fields-group
    = f.input :send_email_notification, as: :boolean

  - if params[:report_id].present?  # ← 仅带上下文时显示
    .fields-group
      = f.input :include_statuses, as: :boolean

  .fields-group
    = f.input :warning_preset_id, collection: @warning_presets

  .fields-group
    = f.input :text, as: :text
```

### 4.3 举报关闭范围差异

**核心逻辑**: `app/models/admin/account_action.rb:131-137`

```ruby
def reports
  @reports ||= if type == 'none'
                 with_report? ? [report] : []
               else
                 target_account.targeted_reports.unresolved
               end
end
```

**本地账号举报关闭范围对比**：

假设场景：账号 B 有 3 个未解决举报 R1、R2、R3

| 操作类型 | 进入路径 | report_id | `reports` 返回值 | 被关闭的举报 |
|---------|---------|-----------|-----------------|-------------|
| `none`（仅警告） | 举报详情页 R1 | ✓ R1 | `[R1]` | **仅 R1** |
| `none`（仅警告） | 账号管理页 | ✗ | `[]` | **无** |
| `disable` | 举报详情页 R1 | ✓ R1 | `[R1, R2, R3]` | **全部** |
| `disable` | 账号管理页 | ✗ | `[R1, R2, R3]` | **全部** |
| `sensitive` | 举报详情页 R1 | ✓ R1 | `[R1, R2, R3]` | **全部** |
| `sensitive` | 账号管理页 | ✗ | `[R1, R2, R3]` | **全部** |
| `silence` | 举报详情页 R1 | ✓ R1 | `[R1, R2, R3]` | **全部** |
| `silence` | 账号管理页 | ✗ | `[R1, R2, R3]` | **全部** |
| `suspend` | 举报详情页 R1 | ✓ R1 | `[R1, R2, R3]` | **全部** |
| `suspend` | 账号管理页 | ✗ | `[R1, R2, R3]` | **全部** |

**关键发现**:
1. **只有 `type == 'none'`（仅警告）时，路径差异才影响举报关闭范围**
2. **其他动作（disable/sensitive/silence/suspend）无论从哪进入，都会关闭该账号所有未解决的举报**

### 4.4 审计记录差异

#### 4.4.1 执行流程与记录点

**执行流程**: `app/models/admin/account_action.rb:45-55`

```ruby
def process_action!
  ApplicationRecord.transaction do
    handle_type!         # 1. 执行限制操作 → 记录操作日志
    process_strike!      # 2. 创建 AccountWarning
    create_log!          # 3. 仅 none 类型且有文本时，记录 warning 创建
    process_reports!     # 4. 关闭举报 → 每个关闭的举报都记录 :resolve
  end

  process_notification! # 5. 发送通知（仅本地账号 + 勾选邮件）
  process_queue!
end
```

#### 4.4.2 各类操作的审计记录

| 步骤 | 记录内容 | 条件 | 代码位置 |
|------|---------|------|---------|
| `handle_type!` | `:disable` → User | `type == 'disable'` | `account_action.rb:85-89` |
| `handle_type!` | `:sensitive` → Account | `type == 'sensitive'` | `account_action.rb:91-95` |
| `handle_type!` | `:silence` → Account | `type == 'silence'` | `account_action.rb:97-101` |
| `handle_type!` | `:suspend` → Account | `type == 'suspend'` | `account_action.rb:103-107` |
| `create_log!` | `:create` → AccountWarning | `type == 'none'` 且有自定义文本 | `account_action.rb:109-113` |
| `process_reports!` | `:resolve` → Report | 每个被关闭的举报 | `account_action.rb:70-83` |

#### 4.4.3 AccountWarning 的关联差异

**创建逻辑**: `app/models/admin/base_action.rb:45-53`

```ruby
def process_strike!(action = type)
  @warning = target_account.strikes.create!(
    account: current_account,
    report: report,           # ← 关联的举报（带上下文时为 R1，否则为 nil）
    action:,
    text: text_for_warning,
    status_ids: status_ids    # ← 关联的嘟文（带上下文且勾选 include_statuses 时）
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

#### 4.4.4 审计记录对比（本地账号）

**场景**: 账号 B 有 R1、R2、R3 三个未解决举报，从 R1 详情页或账号管理页执行操作

| 操作类型 | 路径 | 审计记录 | 说明 |
|---------|------|---------|------|
| `none` + 文本 | 举报详情页 R1 | 1. `:create` → Warning(R1)<br>2. `:resolve` → Report(R1) | 仅警告 + 关闭 R1 |
| `none` + 文本 | 账号管理页 | 1. `:create` → Warning(nil) | 仅警告，不关闭任何举报 |
| `disable` | 举报详情页 R1 | 1. `:disable` → User(B)<br>2. `:resolve` → Report(R1)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R3) | 禁用 + 关闭所有举报 |
| `disable` | 账号管理页 | 1. `:disable` → User(B)<br>2. `:resolve` → Report(R1)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R3) | **审计记录完全相同** |
| `silence` | 举报详情页 R1 | 1. `:silence` → Account(B)<br>2. `:resolve` → Report(R1)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R3) | 静音 + 关闭所有举报 |
| `silence` | 账号管理页 | 1. `:silence` → Account(B)<br>2. `:resolve` → Report(R1)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R3) | **审计记录完全相同** |

### 4.5 页面展示差异

#### 4.5.1 举报详情页的审计视图组成

**数据来源**: `app/controllers/admin/reports_controller.rb:15-17`

```ruby
def show
  @report_notes = @report.notes.chronological.includes(:account)
  @action_logs  = @report.history.includes(:target)
end
```

**Report#history 的组成**: `app/models/report.rb:138-162`

```ruby
def history
  subquery = [
    # 1. 举报本身的操作
    Admin::ActionLog.where(target_type: 'Report', target_id: id),
    
    # 2. 被举报账号的操作
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

#### 4.5.2 本地账号页面展示对比

**场景**: 账号 B 本地账号，有 R1、R2、R3 三个未解决举报，R1 关联嘟文 S1、S2

| 操作 | 路径 | R1 详情页 Action Logs 显示 | 说明 |
|------|------|---------------------------|------|
| `none` + 文本 | 举报页 R1 | 1. `:create` → Warning(R1)<br>2. `:resolve` → Report(R1) | Warning 关联 R1，故显示 |
| `none` + 文本 | 账号页 | **无 Warning 日志**<br>**无 resolve 日志** | Warning 不关联 R1，且 R1 未关闭 |
| `disable` | 举报页 R1 | 1. `:disable` → User(B)<br>2. `:resolve` → Report(R3)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R1) | 按倒序排列 |
| `disable` | 账号页 | 1. `:disable` → User(B)<br>2. `:resolve` → Report(R3)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R1) | **完全相同** |
| `silence` | 举报页 R1 | 1. `:silence` → Account(B)<br>2. `:resolve` → Report(R3)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R1) | 按倒序排列 |
| `silence` | 账号页 | 1. `:silence` → Account(B)<br>2. `:resolve` → Report(R3)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R1) | **完全相同** |

#### 4.5.3 通知机制差异

**通知判断**: `app/models/admin/base_action.rb:62-64`

```ruby
def warnable?
  send_email_notification? && target_account.local?
end
```

**本地账号通知**:
- 发送邮件: `UserMailer.warning(target_account.user, warning).deliver_later!`
- 发送站内通知: `LocalNotificationWorker.perform_async(...)`

**通知前提**:
1. 目标账号必须是本地账号
2. 管理员必须勾选 `send_email_notification`

---

## 五、远端账号：自定义动作详细分析

### 5.1 可用动作类型

**远端账号仅支持 3 种动作**（`app/models/admin/account_action.rb:26`）:
- `sensitive`（标记敏感）
- `silence`（静音）
- `suspend`（暂停/封禁）

**不可用动作**:
- `none`（仅警告）: 远端账号无法接收警告
- `disable`（禁用账号）: 远端账号无本地 User 记录

### 5.2 表单字段展示对比

**视图**: `app/views/admin/account_actions/new.html.haml:25-53`

```haml
- if @account.local?
  %hr.spacer/

  .fields-group
    = f.input :send_email_notification, as: :boolean

  - if params[:report_id].present?
    .fields-group
      = f.input :include_statuses, as: :boolean

  %hr.spacer/

  - unless @warning_presets.empty?
    .fields-group
      = f.input :warning_preset_id, collection: @warning_presets

  .fields-group
    = f.input :text, as: :text
```

**关键**: 整个 `if @account.local?` 块对于远端账号**完全不执行**

**远端账号表单字段对比**：

| 字段 | 带举报上下文 (report_id=R1) | 不带举报上下文 | 说明 |
|------|---------------------------|---------------|------|
| `report_id` 隐藏字段 | ✓ (值为 R1) | ✓ (值为 nil) | 始终存在 |
| `type` 操作类型 | ✓ (仅 3 种: sensitive/silence/suspend) | ✓ (仅 3 种) | 无 none/disable |
| `send_email_notification` | ✗ | ✗ | 远端账号不显示 |
| `include_statuses` | ✗ | ✗ | 条件 `@account.local?` 不满足 |
| `warning_preset_id` | ✗ | ✗ | 远端账号不显示 |
| `text` 自定义文本 | ✗ | ✗ | 远端账号不显示 |

**远端账号 vs 本地账号表单差异总结**：

| 账号类型 | 可用 type | send_email | include_statuses | warning_preset | 自定义文本 |
|---------|-----------|-----------|-----------------|---------------|-----------|
| 本地账号 | 5 种 | ✓ | ✓ (带 report_id) | ✓ | ✓ |
| 远端账号 | 3 种 | ✗ | ✗ | ✗ | ✗ |

### 5.3 举报关闭范围差异

**核心逻辑**（与本地账号相同）: `app/models/admin/account_action.rb:131-137`

```ruby
def reports
  @reports ||= if type == 'none'
                 with_report? ? [report] : []
               else
                 target_account.targeted_reports.unresolved
               end
end
```

**远端账号的特殊情况**:
- 由于远端账号无法使用 `type == 'none'`，所以**路径差异不影响举报关闭范围**
- 所有可用动作（sensitive/silence/suspend）都会关闭该账号所有未解决的举报

**远端账号举报关闭范围对比**：

假设场景：账号 B（远端）有 3 个未解决举报 R1、R2、R3

| 操作类型 | 进入路径 | report_id | `reports` 返回值 | 被关闭的举报 |
|---------|---------|-----------|-----------------|-------------|
| `sensitive` | 举报详情页 R1 | ✓ R1 | `[R1, R2, R3]` | **全部** |
| `sensitive` | 账号管理页 | ✗ | `[R1, R2, R3]` | **全部** |
| `silence` | 举报详情页 R1 | ✓ R1 | `[R1, R2, R3]` | **全部** |
| `silence` | 账号管理页 | ✗ | `[R1, R2, R3]` | **全部** |
| `suspend` | 举报详情页 R1 | ✓ R1 | `[R1, R2, R3]` | **全部** |
| `suspend` | 账号管理页 | ✗ | `[R1, R2, R3]` | **全部** |

**关键发现**:
- **远端账号的所有可用动作，无论从哪进入，都会关闭该账号所有未解决的举报**
- 这是因为远端账号无法使用 `type == 'none'`，所以路径差异对远端账号**没有实际影响**

### 5.4 审计记录差异

#### 5.4.1 执行流程

**执行流程**: `app/models/admin/account_action.rb:45-55`

```ruby
def process_action!
  ApplicationRecord.transaction do
    handle_type!         # 1. 执行限制操作
    process_strike!      # 2. 创建 AccountWarning
    create_log!          # 3. 仅 none 类型 → 远端账号永远不执行
    process_reports!     # 4. 关闭举报
  end

  process_notification! # 5. 远端账号不执行
  process_queue!
end
```

#### 5.4.2 远端账号审计记录对比

**场景**: 账号 B（远端）有 R1、R2、R3 三个未解决举报

| 操作类型 | 路径 | 审计记录 | 说明 |
|---------|------|---------|------|
| `sensitive` | 举报详情页 R1 | 1. `:sensitive` → Account(B)<br>2. `:resolve` → Report(R1)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R3) | 无 `:create` → Warning 记录 |
| `sensitive` | 账号管理页 | 1. `:sensitive` → Account(B)<br>2. `:resolve` → Report(R1)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R3) | **完全相同** |
| `silence` | 举报详情页 R1 | 1. `:silence` → Account(B)<br>2. `:resolve` → Report(R1)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R3) | 无 `:create` → Warning 记录 |
| `silence` | 账号管理页 | 1. `:silence` → Account(B)<br>2. `:resolve` → Report(R1)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R3) | **完全相同** |
| `suspend` | 举报详情页 R1 | 1. `:suspend` → Account(B)<br>2. `:resolve` → Report(R1)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R3) | 无 `:create` → Warning 记录 |
| `suspend` | 账号管理页 | 1. `:suspend` → Account(B)<br>2. `:resolve` → Report(R1)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R3) | **完全相同** |

#### 5.4.3 AccountWarning 的关联

虽然远端账号没有 `text` 字段，但仍会创建 AccountWarning：

```ruby
def process_strike!(action = type)
  @warning = target_account.strikes.create!(
    account: current_account,
    report: report,           # ← 带上下文时为 R1，否则为 nil
    action:,
    text: text_for_warning,   # ← 远端账号为空字符串
    status_ids: status_ids    # ← 远端账号不填（因为 include_statuses 不显示）
  )
end
```

**远端账号的 AccountWarning**:
- `report_id`: 带上下文时为 R1，否则为 nil
- `text`: 空字符串（因为不显示自定义文本字段）
- `status_ids`: nil（因为 include_statuses 字段不显示）

### 5.5 页面展示差异

#### 5.5.1 举报详情页展示对比

**场景**: 账号 B（远端）有 R1、R2、R3 三个未解决举报

| 操作 | 路径 | R1 详情页 Action Logs 显示 | 说明 |
|------|------|---------------------------|------|
| `sensitive` | 举报页 R1 | 1. `:sensitive` → Account(B)<br>2. `:resolve` → Report(R3)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R1) | 无 Warning 创建日志 |
| `sensitive` | 账号页 | 1. `:sensitive` → Account(B)<br>2. `:resolve` → Report(R3)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R1) | **完全相同** |
| `silence` | 举报页 R1 | 1. `:silence` → Account(B)<br>2. `:resolve` → Report(R3)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R1) | 无 Warning 创建日志 |
| `silence` | 账号页 | 1. `:silence` → Account(B)<br>2. `:resolve` → Report(R3)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R1) | **完全相同** |
| `suspend` | 举报页 R1 | 1. `:suspend` → Account(B)<br>2. `:resolve` → Report(R3)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R1) | 无 Warning 创建日志 |
| `suspend` | 账号页 | 1. `:suspend` → Account(B)<br>2. `:resolve` → Report(R3)<br>3. `:resolve` → Report(R2)<br>4. `:resolve` → Report(R1) | **完全相同** |

#### 5.5.2 通知机制

**远端账号不发送通知**:

```ruby
def warnable?
  send_email_notification? && target_account.local?
  # 远端账号: target_account.local? = false → 永远不执行通知
end
```

**远端账号通知情况**:
- **无邮件通知**: 远端账号无本地 User 记录
- **无站内通知**: 远端账号无法在本实例登录

---

## 六、最小可复核流程对照

### 6.1 场景设定

**共同前置条件**:
- 账号 B（本地或远端）有 3 个未解决举报：R1、R2、R3
- R1 关联嘟文 S1、S2
- 管理员 M 执行操作

---

### 6.2 本地账号复核流程

#### 流程一：从举报详情页 R1 执行仅警告 (`type=none`)

**适用范围**: 仅本地账号

**执行步骤**:

| 步骤 | 操作 | 代码路径 | 预期结果 |
|------|------|---------|---------|
| 1 | 访问 R1 详情页点击"自定义" | `_actions.html.haml:45` | 跳转至 `/admin/accounts/B/action/new?report_id=R1` |
| 2 | 表单字段检查 | `new.html.haml` | 显示: type(5种), send_email, include_statuses, warning_preset, text |
| 3 | 选择操作类型 `none` | `new.html.haml:15-23` | `type = 'none'` |
| 4 | 勾选 `include_statuses` | `new.html.haml:33-37` | `include_statuses = true` |
| 5 | 填写自定义文本 | `new.html.haml:48-52` | `text = '警告内容'` |
| 6 | 提交表单 | `account_actions_controller#create` | 创建 AccountAction |
| 7 | 重定向 | `account_actions_controller:22-23` | 跳转至 `/admin/reports` |

**数据变化验证**:

| 验证项 | SQL 查询 | 预期值 |
|--------|----------|--------|
| AccountWarning.report_id | `SELECT report_id FROM account_warnings ORDER BY id DESC LIMIT 1` | `R1` |
| AccountWarning.status_ids | `SELECT status_ids FROM account_warnings ORDER BY id DESC LIMIT 1` | `[S1, S2]` |
| R1 状态 | `SELECT action_taken_at FROM reports WHERE id = R1` | **NOT NULL** |
| R2 状态 | `SELECT action_taken_at FROM reports WHERE id = R2` | **NULL** |
| R3 状态 | `SELECT action_taken_at FROM reports WHERE id = R3` | **NULL** |
| Action Log 数量 (Report) | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'Report' AND target_id IN (R1,R2,R3)` | **1**（仅 R1） |
| Action Log 数量 (Warning) | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'AccountWarning'` | **1**（create 记录） |

**R1 详情页展示**:
```
操作日志 (Action Logs):
  1. admin 创建了警告 #W1
  2. admin 解决了举报 #R1

备注 (Notes):
  (空)
```

---

#### 流程二：从账号管理页执行仅警告 (`type=none`)

**适用范围**: 仅本地账号

**执行步骤**:

| 步骤 | 操作 | 代码路径 | 预期结果 |
|------|------|---------|---------|
| 1 | 访问账号 B 详情页点击"发送警告" | `_buttons.html.haml:15` | 跳转至 `/admin/accounts/B/action/new?type=none` |
| 2 | 表单字段检查 | `new.html.haml` | 显示: type(5种), send_email, warning_preset, text<br>**不显示 include_statuses** |
| 3 | 操作类型已预设为 `none` | `new.html.haml:15-23` | `type = 'none'` |
| 4 | 填写自定义文本 | `new.html.haml:48-52` | `text = '警告内容'` |
| 5 | 提交表单 | `account_actions_controller#create` | 创建 AccountAction |
| 6 | 重定向 | `account_actions_controller:24-25` | 跳转至 `/admin/accounts/B` |

**数据变化验证**:

| 验证项 | SQL 查询 | 预期值 |
|--------|----------|--------|
| AccountWarning.report_id | `SELECT report_id FROM account_warnings ORDER BY id DESC LIMIT 1` | **NULL** |
| AccountWarning.status_ids | `SELECT status_ids FROM account_warnings ORDER BY id DESC LIMIT 1` | **NULL** |
| R1 状态 | `SELECT action_taken_at FROM reports WHERE id = R1` | **NULL**（仍未解决） |
| R2 状态 | `SELECT action_taken_at FROM reports WHERE id = R2` | **NULL** |
| R3 状态 | `SELECT action_taken_at FROM reports WHERE id = R3` | **NULL** |
| Action Log 数量 (Report) | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'Report' AND target_id IN (R1,R2,R3)` | **0** |
| Action Log 数量 (Warning) | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'AccountWarning'` | **1**（create 记录） |

**R1 详情页展示**:
```
操作日志 (Action Logs):
  (空，因为 Warning 不关联 R1，且 R1 未关闭)

备注 (Notes):
  (空)
```

**账号页 Strikes 列表展示**:
```
之前的警告 (Previous Strikes):
  · 警告 - 无关联举报（自定义文本内容）
```

---

#### 流程三：从举报详情页 R1 执行静音 (`type=silence`)

**适用范围**: 本地账号 + 远端账号

**执行步骤**:

| 步骤 | 操作 | 代码路径 | 预期结果 |
|------|------|---------|---------|
| 1 | 访问 R1 详情页点击"自定义" | `_actions.html.haml:45` | 跳转至 `/admin/accounts/B/action/new?report_id=R1` |
| 2 | 表单字段检查（本地账号） | `new.html.haml` | 显示: type(5种), send_email, include_statuses, warning_preset, text |
| 3 | 选择操作类型 `silence` | `new.html.haml:15-23` | `type = 'silence'` |
| 4 | 勾选 `include_statuses` | `new.html.haml:33-37` | `include_statuses = true` |
| 5 | 提交表单 | `account_actions_controller#create` | 创建 AccountAction |
| 6 | 重定向 | `account_actions_controller:22-23` | 跳转至 `/admin/reports` |

**数据变化验证**:

| 验证项 | SQL 查询 | 预期值 |
|--------|----------|--------|
| 账号状态 | `SELECT silenced_at FROM accounts WHERE id = B` | **NOT NULL** |
| AccountWarning.report_id | `SELECT report_id FROM account_warnings ORDER BY id DESC LIMIT 1` | `R1` |
| AccountWarning.status_ids | `SELECT status_ids FROM account_warnings ORDER BY id DESC LIMIT 1` | `[S1, S2]` |
| R1 状态 | `SELECT action_taken_at FROM reports WHERE id = R1` | **NOT NULL** |
| R2 状态 | `SELECT action_taken_at FROM reports WHERE id = R2` | **NOT NULL** |
| R3 状态 | `SELECT action_taken_at FROM reports WHERE id = R3` | **NOT NULL** |
| Action Log 数量 (Report) | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'Report' AND target_id IN (R1,R2,R3)` | **3**（全部关闭） |
| Action Log 数量 (Account) | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'Account' AND target_id = B` | **1**（silence） |
| Action Log 数量 (Warning) | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'AccountWarning'` | **0**（非 none 类型） |

**R1 详情页展示**:
```
操作日志 (Action Logs):
  1. admin 静音了账号 @bob@local.example
  2. admin 解决了举报 #R3
  3. admin 解决了举报 #R2
  4. admin 解决了举报 #R1
  (按倒序排列，最新的在最前)

备注 (Notes):
  (空)
```

---

#### 流程四：从账号管理页执行静音 (`type=silence`)

**适用范围**: 本地账号 + 远端账号

**执行步骤**:

| 步骤 | 操作 | 代码路径 | 预期结果 |
|------|------|---------|---------|
| 1 | 访问账号 B 详情页点击"静音" | `_buttons.html.haml:27` | 跳转至 `/admin/accounts/B/action/new?type=silence` |
| 2 | 表单字段检查（本地账号） | `new.html.haml` | 显示: type(5种), send_email, warning_preset, text<br>**不显示 include_statuses** |
| 3 | 操作类型已预设为 `silence` | `new.html.haml:15-23` | `type = 'silence'` |
| 4 | 提交表单 | `account_actions_controller#create` | 创建 AccountAction |
| 5 | 重定向 | `account_actions_controller:24-25` | 跳转至 `/admin/accounts/B` |

**数据变化验证**:

| 验证项 | SQL 查询 | 预期值 |
|--------|----------|--------|
| 账号状态 | `SELECT silenced_at FROM accounts WHERE id = B` | **NOT NULL** |
| AccountWarning.report_id | `SELECT report_id FROM account_warnings ORDER BY id DESC LIMIT 1` | **NULL** |
| AccountWarning.status_ids | `SELECT status_ids FROM account_warnings ORDER BY id DESC LIMIT 1` | **NULL** |
| R1 状态 | `SELECT action_taken_at FROM reports WHERE id = R1` | **NOT NULL** |
| R2 状态 | `SELECT action_taken_at FROM reports WHERE id = R2` | **NOT NULL** |
| R3 状态 | `SELECT action_taken_at FROM reports WHERE id = R3` | **NOT NULL** |
| Action Log 数量 (Report) | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'Report' AND target_id IN (R1,R2,R3)` | **3**（全部关闭） |
| Action Log 数量 (Account) | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'Account' AND target_id = B` | **1**（silence） |
| Action Log 数量 (Warning) | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'AccountWarning'` | **0**（非 none 类型） |

**R1 详情页展示**:
```
操作日志 (Action Logs):
  1. admin 静音了账号 @bob@local.example
  2. admin 解决了举报 #R3
  3. admin 解决了举报 #R2
  4. admin 解决了举报 #R1
  (与流程三完全相同)

备注 (Notes):
  (空)
```

---

### 6.3 远端账号复核流程

#### 流程五：从举报详情页 R1 执行静音 (`type=silence`)

**适用范围**: 远端账号

**执行步骤**:

| 步骤 | 操作 | 代码路径 | 预期结果 |
|------|------|---------|---------|
| 1 | 访问 R1 详情页点击"自定义" | `_actions.html.haml:45` | 跳转至 `/admin/accounts/B/action/new?report_id=R1` |
| 2 | 表单字段检查（远端账号） | `new.html.haml` | 仅显示: type(3种: sensitive/silence/suspend)<br>**不显示**: send_email, include_statuses, warning_preset, text |
| 3 | 选择操作类型 `silence` | `new.html.haml:15-23` | `type = 'silence'` |
| 4 | 提交表单 | `account_actions_controller#create` | 创建 AccountAction |
| 5 | 重定向 | `account_actions_controller:22-23` | 跳转至 `/admin/reports` |

**数据变化验证**:

| 验证项 | SQL 查询 | 预期值 |
|--------|----------|--------|
| 账号状态 | `SELECT silenced_at FROM accounts WHERE id = B` | **NOT NULL** |
| AccountWarning.report_id | `SELECT report_id FROM account_warnings ORDER BY id DESC LIMIT 1` | `R1` |
| AccountWarning.status_ids | `SELECT status_ids FROM account_warnings ORDER BY id DESC LIMIT 1` | **NULL**（远端账号无 include_statuses） |
| AccountWarning.text | `SELECT text FROM account_warnings ORDER BY id DESC LIMIT 1` | **空字符串**（远端账号无 text 字段） |
| R1 状态 | `SELECT action_taken_at FROM reports WHERE id = R1` | **NOT NULL** |
| R2 状态 | `SELECT action_taken_at FROM reports WHERE id = R2` | **NOT NULL** |
| R3 状态 | `SELECT action_taken_at FROM reports WHERE id = R3` | **NOT NULL** |
| Action Log 数量 (Report) | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'Report' AND target_id IN (R1,R2,R3)` | **3**（全部关闭） |
| Action Log 数量 (Account) | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'Account' AND target_id = B` | **1**（silence） |
| Action Log 数量 (Warning) | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'AccountWarning'` | **0**（非 none 类型） |

**R1 详情页展示**:
```
操作日志 (Action Logs):
  1. admin 静音了账号 @bob@remote.example
  2. admin 解决了举报 #R3
  3. admin 解决了举报 #R2
  4. admin 解决了举报 #R1

备注 (Notes):
  (空)
```

---

#### 流程六：从账号管理页执行静音 (`type=silence`)

**适用范围**: 远端账号

**执行步骤**:

| 步骤 | 操作 | 代码路径 | 预期结果 |
|------|------|---------|---------|
| 1 | 访问账号 B 详情页点击"静音" | `_buttons.html.haml:27` | 跳转至 `/admin/accounts/B/action/new?type=silence` |
| 2 | 表单字段检查（远端账号） | `new.html.haml` | 仅显示: type(3种)<br>**不显示**: 其他所有字段 |
| 3 | 操作类型已预设为 `silence` | `new.html.haml:15-23` | `type = 'silence'` |
| 4 | 提交表单 | `account_actions_controller#create` | 创建 AccountAction |
| 5 | 重定向 | `account_actions_controller:24-25` | 跳转至 `/admin/accounts/B` |

**数据变化验证**:

| 验证项 | SQL 查询 | 预期值 |
|--------|----------|--------|
| 账号状态 | `SELECT silenced_at FROM accounts WHERE id = B` | **NOT NULL** |
| AccountWarning.report_id | `SELECT report_id FROM account_warnings ORDER BY id DESC LIMIT 1` | **NULL** |
| AccountWarning.status_ids | `SELECT status_ids FROM account_warnings ORDER BY id DESC LIMIT 1` | **NULL** |
| AccountWarning.text | `SELECT text FROM account_warnings ORDER BY id DESC LIMIT 1` | **空字符串** |
| R1 状态 | `SELECT action_taken_at FROM reports WHERE id = R1` | **NOT NULL** |
| R2 状态 | `SELECT action_taken_at FROM reports WHERE id = R2` | **NOT NULL** |
| R3 状态 | `SELECT action_taken_at FROM reports WHERE id = R3` | **NOT NULL** |
| Action Log 数量 (Report) | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'Report' AND target_id IN (R1,R2,R3)` | **3**（全部关闭） |
| Action Log 数量 (Account) | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'Account' AND target_id = B` | **1**（silence） |
| Action Log 数量 (Warning) | `SELECT COUNT(*) FROM admin_action_logs WHERE target_type = 'AccountWarning'` | **0** |

**R1 详情页展示**:
```
操作日志 (Action Logs):
  1. admin 静音了账号 @bob@remote.example
  2. admin 解决了举报 #R3
  3. admin 解决了举报 #R2
  4. admin 解决了举报 #R1
  (与流程五完全相同)

备注 (Notes):
  (空)
```

---

### 6.4 全部流程对比总结

#### 按账号类型分类的可用流程

| 流程 | 账号类型 | 进入路径 | 操作类型 | 可用 |
|------|---------|---------|---------|------|
| 流程一 | 本地 | 举报页 R1 | none | ✓ |
| 流程二 | 本地 | 账号页 | none | ✓ |
| 流程三 | 本地 | 举报页 R1 | silence | ✓ |
| 流程四 | 本地 | 账号页 | silence | ✓ |
| 流程五 | 远端 | 举报页 R1 | silence | ✓ |
| 流程六 | 远端 | 账号页 | silence | ✓ |
| (远端 none) | 远端 | 任意 | none | ✗ |
| (远端 disable) | 远端 | 任意 | disable | ✗ |

#### 核心差异对照表

**本地账号对比**:

| 对比项 | 流程一<br>举报页+none | 流程二<br>账号页+none | 流程三<br>举报页+silence | 流程四<br>账号页+silence |
|--------|---------------------|---------------------|-------------------------|-------------------------|
| `report_id` | ✓ R1 | ✗ | ✓ R1 | ✗ |
| 账号限制 | 无 | 无 | 静音 | 静音 |
| 关闭举报数 | 仅 R1 | 无 | R1+R2+R3 | R1+R2+R3 |
| Warning.report_id | ✓ R1 | ✗ | ✓ R1 | ✗ |
| Warning.status_ids | ✓ S1,S2 | ✗ | ✓ S1,S2 | ✗ |
| include_statuses 字段 | ✓ 显示 | ✗ 不显示 | ✓ 显示 | ✗ 不显示 |
| Warning create 日志 | ✓ | ✓ | ✗ | ✗ |
| R1 详情页日志 | Warning创建 + R1解决 | 不显示 | 静音 + 3个解决 | 静音 + 3个解决 |
| 重定向目标 | `/admin/reports` | `/admin/accounts/B` | `/admin/reports` | `/admin/accounts/B` |

**远端账号对比**:

| 对比项 | 流程五<br>举报页+silence | 流程六<br>账号页+silence |
|--------|-------------------------|-------------------------|
| `report_id` | ✓ R1 | ✗ |
| 账号限制 | 静音 | 静音 |
| 关闭举报数 | R1+R2+R3 | R1+R2+R3 |
| Warning.report_id | ✓ R1 | ✗ |
| Warning.status_ids | ✗ | ✗ |
| Warning.text | 空字符串 | 空字符串 |
| include_statuses 字段 | ✗ 不显示 | ✗ 不显示 |
| R1 详情页日志 | 静音 + 3个解决 | 静音 + 3个解决 |
| 重定向目标 | `/admin/reports` | `/admin/accounts/B` |

**关键发现总结**:

1. **只有本地账号使用 `type == 'none'` 时，路径差异才产生实际影响**:
   - 举报页进入 → 关闭当前举报 + Warning 关联该举报
   - 账号页进入 → 不关闭任何举报 + Warning 不关联任何举报

2. **对于限制操作（silence/suspend/sensitive/disable）**:
   - 本地账号: 无论从哪进入，都会关闭该账号所有未解决的举报
   - 远端账号: 无论从哪进入，都会关闭该账号所有未解决的举报
   - 唯一差异是 `AccountWarning.report_id` 是否关联

3. **远端账号的特殊限制**:
   - 无法使用 `none`（仅警告）和 `disable`（禁用账号）
   - 表单只显示 `type` 字段，其他字段全部隐藏
   - 不发送任何通知（无邮件、无站内通知）

---

## 七、审计记录机制

### 7.1 审计视图的组成

#### 7.1.1 举报详情页审计区域

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

#### 7.1.2 操作日志 (Action Logs)

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

#### 7.1.3 备注 (Notes)

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

#### 7.1.4 账号管理页的备注系统

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

#### 7.1.5 Notes 与 Action Logs 对比

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

### 7.2 审计记录模型

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

### 7.3 记录创建方式

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

### 7.4 审计记录点总览

#### 7.4.1 举报相关的记录点

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

#### 7.4.2 账号操作相关的记录点

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

#### 7.4.3 嘟文操作相关的记录点

**嘟文操作模型**: `app/models/admin/moderation_action.rb`

| 操作 | action 值 | 目标类型 |
|------|----------|---------|
| 删除嘟文 | `:destroy` | Status |
| 删除集合 | `:destroy` | Collection |
| 更新嘟文（标记敏感） | `:update` | Status |
| 更新集合（标记敏感） | `:update` | Collection |
| 解决举报 | `:resolve` | Report |

---

## 八、本地审核与远端账号处理的边界

### 8.1 本地/远端账号判断标准

**判断逻辑**: `app/models/account.rb:208-214`

```ruby
def local?
  domain.nil?  # 无域名 → 本地账号
end

def remote?
  !domain.nil? # 有域名 → 远端账号
end
```

### 8.2 举报阶段的差异

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

### 8.3 审核操作的边界

#### 8.3.1 可用操作类型差异

| 操作类型 | 本地账号 | 远端账号 | 说明 |
|---------|---------|---------|------|
| `none` | ✓ | ✗ | 远端账号无法仅发送警告 |
| `disable` | ✓ | ✗ | 远端账号无本地 User 记录 |
| `sensitive` | ✓ | ✓ | 仅影响本地显示 |
| `silence` | ✓ | ✓ | 仅影响本地显示和传播 |
| `suspend` | ✓ | ✓ | 本地封禁，远端账号仍可在原实例使用 |

#### 8.3.2 暂停操作的深层差异

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

#### 8.3.3 嘟文处理差异

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

#### 8.3.4 通知差异

**警告通知**: `app/models/admin/base_action.rb:62-64`

```ruby
def warnable?
  send_email_notification? && target_account.local?  # 仅本地账号能收到通知
end
```
- 本地账号: 发送邮件和站内通知
- 远端账号: 无法发送通知（没有本地 User 记录）

### 8.4 处理边界总结

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

## 九、端到端处理时间线

### 9.1 场景设定

**角色**:
- 用户 A: 本地用户 (@alice@local.example)
- 用户 B: 远端用户 (@bob@remote.example)
- 管理员 M: 本地实例管理员 (@admin@local.example)

**事件**: 用户 A 举报了用户 B 的一条回复性嘟文，管理员 M 处理举报并暂停账号 B。

---

### 9.2 详细时间线

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
  3. ` `action: :resolve`, `target: Report#R1`
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

### 9.3 时间线可视化

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

### 9.4 关键时间点的数据变化

| 时间点 | Report#R1 | ActionLog | ReportNote | Account#B |
|-------|-----------|-----------|------------|-----------|
| T1 | `action_taken_at: nil` | 空 | 空 | 正常 |
| T3 | `action_taken_at: nil` | 空 | N1 已创建 | 正常 |
| T4 | `action_taken_at: 现在` | 3 条记录 | N1 | `suspended_at: 现在` |
| T5 | 不变 | 不变 | 不变 | 后台清理完成 |
| T6 | 不变 | 不变 | 不变 | 已暂停 |

### 9.5 远端账号 B 的实际状态

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
