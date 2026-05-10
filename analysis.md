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

## 二、管理员操作对账号限制的影响

### 2.1 管理员操作类型
**账号操作**: `app/models/admin/account_action.rb`

**操作类型** (TYPES):
| 操作类型 | 说明 | 适用范围 |
|---------|------|---------|
| `none` | 仅发送警告，不采取限制 | 仅本地账号 |
| `disable` | 禁用用户账号 | 仅本地账号 |
| `sensitive` | 标记账号为敏感内容 | 本地+远端 |
| `silence` | 静音账号 | 本地+远端 |
| `suspend` | 暂停/封禁账号 | 本地+远端 |

**适用范围差异**: `app/models/admin/account_action.rb:22-28`
```ruby
def self.types_for_account(account)
  if account.local?
    TYPES  # 全部 5 种操作
  else
    TYPES - %w(none disable)  # 远端账号不能使用 none 和 disable
  end
end
```

### 2.2 操作执行流程
**基类**: `app/models/admin/base_action.rb`

**执行步骤** (`process_action!`):
1. **执行限制操作**: `handle_type!` - 根据类型执行具体限制
2. **创建警告记录**: `process_strike!` - 创建 `AccountWarning` 记录
3. **记录审计日志**: `create_log!` / 各操作中调用 `log_action`
4. **处理关联举报**: `process_reports!` - 关闭相关未处理举报
5. **发送通知**: `process_notification!` - 向用户发送邮件/站内通知（仅本地）
6. **后台任务**: `process_queue!` - 如封禁则启动后台任务

### 2.3 具体限制操作

#### 2.3.1 禁用账号 (disable)
- **位置**: `app/models/admin/account_action.rb:85-89`
- **影响**: 禁用用户的登录能力
- **审计**: `log_action(:disable, target_account.user)`

#### 2.3.2 标记为敏感 (sensitive)
- **位置**: `app/models/admin/account_action.rb:91-95`
- **方法**: `target_account.sensitize!` - `app/models/concerns/account/sensitizes.rb:14-16`
- **影响**: 设置 `sensitized_at` 时间戳
- **审计**: `log_action(:sensitive, target_account)`

#### 2.3.3 静音 (silence)
- **位置**: `app/models/admin/account_action.rb:97-101`
- **方法**: `target_account.silence!` - `app/models/concerns/account/silences.rb:15-17`
- **影响**: 设置 `silenced_at` 时间戳
- **审计**: `log_action(:silence, target_account)`

#### 2.3.4 暂停/封禁 (suspend)
- **位置**: `app/models/admin/account_action.rb:103-107`
- **方法**: `target_account.suspend!(origin: :local)` - `app/models/concerns/account/suspensions.rb:29-39`
- **影响**:
  - 创建删除请求
  - 设置 `suspended_at` 和 `suspension_origin`
  - 阻塞邮箱（可选）
  - 本地账号：强制断开所有流式连接
- **审计**: `log_action(:suspend, target_account)`
- **后台处理**: `Admin::SuspensionWorker`

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

### 2.4 嘟文/集合操作
**嘟文操作**: `app/models/admin/moderation_action.rb`

**操作类型**:
| 操作类型 | 说明 |
|---------|------|
| `delete` | 删除嘟文/集合 |
| `mark_as_sensitive` | 标记为敏感内容 |

**删除操作流程** (`handle_delete!`):
1. 删除嘟文: `delete_statuses!` - 调用 `status.discard_with_reblogs`
2. 删除集合: `delete_collections!`
3. 关闭举报: `resolve_report!`
4. 创建警告: `process_strike!`
5. 远端账号: 创建墓碑记录 `create_tombstones!`
6. 通知: `process_notification!`
7. 后台清理: `RemovalWorker`

**标记敏感流程** (`handle_mark_as_sensitive!`):
- 本地账号: 使用 `UpdateStatusService` 更新
- 远端账号: 直接更新 `sensitive: true`（仅本地标记，不影响原实例）

---

## 三、审计记录机制

### 3.1 审计记录模型
**模型**: `app/models/admin/action_log.rb`

**存储表**: `admin_action_logs`

**关键字段**:
| 字段 | 说明 |
|------|------|
| `account_id` | 执行操作的管理员账号 |
| `action` | 操作类型（字符串） |
| `target_type` | 目标对象类型（多态） |
| `target_id` | 目标对象 ID |
| `human_identifier` | 人类可读的目标标识 |
| `permalink` | 目标对象链接 |
| `route_param` | 路由参数 |
| `created_at` | 操作时间 |

### 3.2 记录创建方式
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

### 3.3 举报相关的审计记录
**举报历史查询**: `app/models/report.rb:138-162`
```ruby
def history
  subquery = [
    Admin::ActionLog.where(target_type: 'Report', target_id: id),           # 举报本身的操作
    Admin::ActionLog.where(target_type: 'Account', target_id: target_account_id),  # 目标账号的操作
    Admin::ActionLog.where(target_type: 'Status', target_id: status_ids),    # 相关嘟文的操作
    Admin::ActionLog.where(target_type: 'AccountWarning', target_id: AccountWarning.where(report_id: id)),  # 警告记录
  ].reduce { |union, query| Arel::Nodes::UnionAll.new(union, query) }
  
  Admin::ActionLog.latest.from(Arel::Nodes::As.new(subquery, Admin::ActionLog.arel_table))
end
```

**管理员控制器中的记录点**: `app/controllers/admin/reports_controller.rb`
- `log_action :assigned_to_self, @report` - 分配给自己
- `log_action :unassigned, @report` - 取消分配
- `log_action :reopen, @report` - 重新打开
- `log_action :resolve, @report` - 标记解决

**账号操作中的记录点**: `app/models/admin/account_action.rb`
- `log_action(:disable, target_account.user)`
- `log_action(:sensitive, target_account)`
- `log_action(:silence, target_account)`
- `log_action(:suspend, target_account)`
- `log_action(:resolve, report)` - 解决关联举报

**嘟文操作中的记录点**: `app/models/admin/moderation_action.rb`
- `log_action(:destroy, status)` - 删除嘟文
- `log_action(:destroy, collection)` - 删除集合
- `log_action(:update, status)` - 更新嘟文（标记敏感）
- `log_action(:resolve, report)` - 解决关联举报

---

## 四、本地审核与远端账号处理的边界

### 4.1 本地/远端账号判断标准
**判断逻辑**: `app/models/account.rb:208-214`
```ruby
def local?
  domain.nil?  # 无域名 → 本地账号
end

def remote?
  !domain.nil? # 有域名 → 远端账号
end
```

### 4.2 举报阶段的差异

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

### 4.3 审核操作的边界

#### 4.3.1 可用操作类型差异
| 操作类型 | 本地账号 | 远端账号 | 说明 |
|---------|---------|---------|------|
| `none` | ✓ | ✗ | 远端账号无法仅发送警告 |
| `disable` | ✓ | ✗ | 远端账号无本地 User 记录 |
| `sensitive` | ✓ | ✓ | 仅影响本地显示 |
| `silence` | ✓ | ✓ | 仅影响本地显示和传播 |
| `suspend` | ✓ | ✓ | 本地封禁，远端账号仍可在原实例使用 |

#### 4.3.2 暂停操作的深层差异
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

#### 4.3.3 嘟文处理差异
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

#### 4.3.4 通知差异
**警告通知**: `app/models/admin/base_action.rb:62-64`
```ruby
def warnable?
  send_email_notification? && target_account.local?  # 仅本地账号能收到通知
end
```
- 本地账号: 发送邮件和站内通知
- 远端账号: 无法发送通知（没有本地 User 记录）

### 4.4 处理边界总结

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
