# Mastodon 投票功能实现分析

## 一、整体架构概览

Mastodon 的投票功能采用典型的三层架构：后端模型存储 + API 服务层 + 前端 React 组件 + WebSocket 实时推送。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              前端层 (React + Redux)                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐     ┌──────────────┐     ┌─────────────────────────────┐ │
│  │ Poll.tsx     │────▶│  Redux Store │◀────│  streaming.js (WebSocket)  │ │
│  │ (渲染组件)    │     │  polls.ts    │     │  (实时订阅更新)             │ │
│  └──────────────┘     └──────────────┘     └─────────────────────────────┘ │
│         ▲                    │                                         ▲      │
│         │                    ▼                                         │      │
│         │           ┌─────────────────┐                                │      │
│         │           │ actions/polls.ts│                                │      │
│         │           │ (投票/刷新操作)  │                                │      │
│         │           └─────────────────┘                                │      │
│         │                    │                                         │      │
│         │                    ▼                                         │      │
│         │           ┌─────────────────┐                                │      │
│         │           │ api/polls.ts    │                                │      │
│         │           │ (API 调用封装)   │                                │      │
│         │           └─────────────────┘                                │      │
└─────────┼────────────────────┼─────────────────────────────────────────┼──────┘
          │                    │                                         │
          │                    ▼                                         │
          │    ┌─────────────────────────────────────────────────────┐  │
          └────│                  API 层 (Rails)                      │──┘
               ├─────────────────────────────────────────────────────┤
               │  GET  /api/v1/polls/:id     (获取投票详情)          │
               │  POST /api/v1/polls/:id/votes (提交投票)            │
               └─────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          后端业务逻辑层 (Rails)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────────┐    ┌──────────────────┐    ┌─────────────────────┐  │
│  │ VoteService      │    │ Poll Model       │    │ PollVote Model      │  │
│  │ (投票业务逻辑)    │───▶│ (投票主体)        │◀───│ (单条投票记录)      │  │
│  └──────────────────┘    └──────────────────┘    └─────────────────────┘  │
│           │                                                            │      │
│           │                        ┌──────────────────┐              │      │
│           │                        │  Validators      │              │      │
│           │                        │  - VoteValidator │              │      │
│           │                        │  - PollOptions   │              │      │
│           │                        └──────────────────┘              │      │
│           │                                                            │      │
│           ▼                                                            ▼      │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                         后台任务层 (Sidekiq)                             ││
│  ├─────────────────────────────────────────────────────────────────────────┤│
│  │  - PollExpirationNotifyWorker: 投票到期通知 + 结果广播                   ││
│  │  - DistributePollUpdateWorker: ActivityPub 跨实例广播投票更新           ││
│  │  - LocalNotificationWorker: 本地通知推送                                 ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、数据模型层

### 2.1 Poll 模型（投票主体）

**文件位置**: `app/models/poll.rb`

#### 数据库字段结构

| 字段名 | 类型 | 说明 |
|--------|------|------|
| `id` | bigint | 主键 |
| `account_id` | bigint | 创建者账户 ID (外键) |
| `status_id` | bigint | 关联嘟文 ID (外键) |
| `options` | string[] | 投票选项数组 (PostgreSQL Array) |
| `cached_tallies` | bigint[] | 缓存的各选项得票数数组 |
| `votes_count` | bigint | 总投票次数 (多选时可能大于投票人数) |
| `voters_count` | bigint | 投票人数 |
| `multiple` | boolean | 是否允许多选 |
| `hide_totals` | boolean | 是否隐藏票数 (匿名投票核心字段) |
| `expires_at` | datetime | 截止时间 |
| `last_fetched_at` | datetime | 最后获取时间 (用于远程实例投票) |
| `lock_version` | integer | 乐观锁版本号 |

#### 核心方法分析

```ruby
# 决定是否显示投票结果的关键逻辑
def show_totals_now?
  expired? || !hide_totals?
end
```
[app/models/poll.rb:121-123](app/models/poll.rb#L121-L123)

**匿名投票机制**:
- 当 `hide_totals=true` 且投票未过期时，前端不显示具体票数
- 只有当投票过期或 `hide_totals=false` 时才显示结果
- 这通过 `loaded_options` 方法动态控制 `votes_count` 的返回：

```ruby
def loaded_options
  options.map.with_index { |title, key| 
    Option.new(self, key.to_s, title, show_totals_now? ? (cached_tallies[key] || 0) : nil) 
  }
end
```
[app/models/poll.rb:51-53](app/models/poll.rb#L51-L53)

#### 投票选项验证

**文件位置**: `app/validators/poll_options_validator.rb`

```ruby
class PollOptionsValidator < ActiveModel::Validator
  MAX_OPTIONS      = 4    # 最多 4 个选项
  MAX_OPTION_CHARS = 50   # 每个选项最多 50 字符

  def validate(poll)
    # 至少 2 个选项
    poll.errors.add(:options, I18n.t('polls.errors.too_few_options')) unless poll.options.size > 1
    # 最多 4 个选项
    poll.errors.add(:options, I18n.t('polls.errors.too_many_options', max: MAX_OPTIONS)) if poll.options.size > MAX_OPTIONS
    # 选项字符数限制
    poll.errors.add(:options, I18n.t('polls.errors.over_character_limit', max: MAX_OPTION_CHARS)) if poll.options.any? { |option| option.each_grapheme_cluster.size > MAX_OPTION_CHARS }
    # 选项不能重复
    poll.errors.add(:options, I18n.t('polls.errors.duplicate_options')) unless poll.options.uniq.size == poll.options.size
  end
end
```
[app/validators/poll_options_validator.rb:3-12](app/validators/poll_options_validator.rb#L3-L12)

### 2.2 PollVote 模型（投票记录）

**文件位置**: `app/models/poll_vote.rb`

#### 数据库字段结构

| 字段名 | 类型 | 说明 |
|--------|------|------|
| `id` | bigint | 主键 |
| `account_id` | bigint | 投票者账户 ID |
| `poll_id` | bigint | 关联投票 ID |
| `choice` | integer | 选择的选项索引 (从 0 开始) |
| `uri` | string | ActivityPub URI (用于远程实例) |

#### 核心机制：计数器缓存更新

```ruby
after_create_commit :increment_counter_cache

def increment_counter_cache
  poll.cached_tallies[choice] = (poll.cached_tallies[choice] || 0) + 1
  poll.save
rescue ActiveRecord::StaleObjectError
  poll.reload
  retry
end
```
[app/models/poll_vote.rb:23-40](app/models/poll_vote.rb#L23-L40)

**乐观锁处理并发**:
- 使用 `lock_version` 字段实现乐观锁
- 当并发更新导致 `StaleObjectError` 时，自动重试
- 这保证了高并发投票场景下计数的准确性

---

## 三、投票业务逻辑层

### 3.1 VoteService 核心流程

**文件位置**: `app/services/vote_service.rb`

```ruby
def call(account, poll, choices)
  return if choices.empty?

  authorize_with account, poll, :vote?

  @account = account
  @poll    = poll
  @choices = choices
  @votes   = []

  # 分布式锁防止重复投票
  with_redis_lock("vote:#{@poll.id}:#{@account.id}") do
    already_voted = @poll.votes.exists?(account: @account)

    ApplicationRecord.transaction do
      @choices.each do |choice|
        @votes << @poll.votes.create!(account: @account, choice: Integer(choice))
      end
    end
  end

  increment_voters_count! unless already_voted

  ActivityTracker.increment('activity:interactions')

  # 根据投票是否为本地实例，执行不同的分发策略
  if @poll.account.local?
    distribute_poll!   # 本地投票：延迟广播更新
  else
    deliver_votes!      # 远程投票：立即投递到源实例
    queue_final_poll_check!
  end
end
```
[app/services/vote_service.rb:9-41](app/services/vote_service.rb#L9-L41)

#### 关键机制解析

| 机制 | 实现方式 | 说明 |
|------|----------|------|
| **防止重复投票** | Redis 分布式锁 | 锁 key: `vote:{poll_id}:{account_id}` |
| **事务保障** | `ApplicationRecord.transaction` | 多选时所有投票记录原子创建 |
| **投票人数计数** | `increment_voters_count!` | 只有首次投票才增加 `voters_count` |
| **匿名投票延迟广播** | `perform_in(3.minutes)` | 非匿名投票才广播，且延迟 3 分钟 |

#### 投票验证器

**文件位置**: `app/validators/vote_validator.rb`

```ruby
def validate(vote)
  # 1. 检查投票是否过期
  vote.errors.add(:base, I18n.t('polls.errors.expired')) if vote.poll_expired?
  # 2. 检查选项索引是否有效
  vote.errors.add(:base, I18n.t('polls.errors.invalid_choice')) if invalid_choice?(vote)
  # 3. 禁止给自己的投票投票
  vote.errors.add(:base, I18n.t('polls.errors.self_vote')) if self_vote?(vote)
  # 4. 检查重复投票
  vote.errors.add(:base, I18n.t('polls.errors.already_voted')) if additional_voting_not_allowed?(vote)
end
```
[app/validators/vote_validator.rb:4-10](app/validators/vote_validator.rb#L4-L10)

### 3.2 投票创建与更新

投票随嘟文一起创建，通过 `UpdateStatusService` 处理。

**文件位置**: `app/services/update_status_service.rb`

```ruby
def update_poll!
  previous_poll        = @status.preloadable_poll
  @previous_expires_at = previous_poll&.expires_at

  if @options[:poll].present?
    poll = previous_poll || @status.account.polls.new(status: @status, votes_count: 0)

    # 如果选项或多选属性改变，重置所有投票
    @poll_changed = true if @options[:poll][:options] != poll.options || ActiveModel::Type::Boolean.new.cast(@options[:poll][:multiple]) != poll.multiple

    poll.options     = @options[:poll][:options]
    poll.hide_totals = @options[:poll][:hide_totals] || false
    poll.multiple    = @options[:poll][:multiple] || false
    poll.expires_in  = @options[:poll][:expires_in]
    poll.reset_votes! if @poll_changed  # 重置投票数据
    poll.save!

    @status.poll_id = poll.id
  elsif previous_poll.present?
    previous_poll.destroy
    @poll_changed = true
    @status.poll_id = nil
  end

  @poll_changed = true if @previous_expires_at != @status.preloadable_poll&.expires_at
end
```
[app/services/update_status_service.rb:85-111](app/services/update_status_service.rb#L85-L111)

---

## 四、截止时间与过期机制

### 4.1 截止时间数据结构

Poll 模型使用 `expires_at` 字段存储绝对过期时间：

```ruby
# 验证：本地投票必须设置过期时间
validates :expires_at, presence: true, if: :local?
```
[app/models/poll.rb:41](app/models/poll.rb#L41)

前端创建投票时选择的是相对时间（秒数）：

| 选项值（秒） | 显示文本 |
|-------------|----------|
| 300 | 5 分钟 |
| 1800 | 30 分钟 |
| 3600 | 1 小时 |
| 21600 | 6 小时 |
| 43200 | 12 小时 |
| 86400 | 1 天 |
| 259200 | 3 天 |
| 604800 | 7 天 |

**文件位置**: `app/javascript/mastodon/features/compose/components/poll_form.jsx:141-150`

### 4.2 过期检测机制

后端检测：
```ruby
def show_totals_now?
  expired? || !hide_totals?
end
```
[app/models/poll.rb:121-123](app/models/poll.rb#L121-L123)

前端检测（双重保障）：
```typescript
const isPollExpired = (expiresAt: Model.Poll['expires_at']) =>
  new Date(expiresAt).getTime() < Date.now();

// 组件中使用
const expired = useMemo(() => {
  if (!poll) return false;
  return poll.expired || isPollExpired(poll.expires_at);
}, [poll]);
```
[app/javascript/mastodon/components/poll.tsx:38-65](app/javascript/mastodon/components/poll.tsx#L38-L65)

### 4.3 过期定时任务

**文件位置**: `app/workers/poll_expiration_notify_worker.rb`

```ruby
def perform(poll_id)
  @poll = Poll.find(poll_id)

  return if missing_expiration?
  requeue! && return if not_due_yet?  # 如果时间未到，重新入队

  notify_remote_voters_and_owner! if @poll.local?
  notify_local_voters!
end

def notify_remote_voters_and_owner!
  # 广播最终结果到 ActivityPub 网络
  ActivityPub::DistributePollUpdateWorker.perform_async(@poll.status.id)
  # 通知投票创建者
  LocalNotificationWorker.perform_async(@poll.account_id, @poll.id, 'Poll', 'poll')
end

def notify_local_voters!
  # 批量通知所有参与投票的本地用户
  @poll.voters.merge(Account.local).select(:id).find_in_batches do |accounts|
    LocalNotificationWorker.push_bulk(accounts) do |account|
      [account.id, @poll.id, 'Poll', 'poll']
    end
  end
end
```
[app/workers/poll_expiration_notify_worker.rb:8-51](app/workers/poll_expiration_notify_worker.rb#L8-L51)

#### 任务调度时机

| 场景 | 调度时机 | 代码位置 |
|------|----------|----------|
| 创建/更新投票 | `expires_at + 5.minutes` | `update_status_service.rb:154` |
| 对远程实例投票 | `expires_at + 5.minutes` | `vote_service.rb:54` |

---

## 五、匿名性与结果统计

### 5.1 匿名投票实现机制

核心字段是 `hide_totals`，其行为由 `show_totals_now?` 控制：

```ruby
def show_totals_now?
  expired? || !hide_totals?
end
```

**逻辑真值表**:

| hide_totals | 投票状态 | show_totals_now? | 结果可见性 |
|-------------|----------|-------------------|-----------|
| `false` | 进行中 | `true` | 实时可见 |
| `false` | 已过期 | `true` | 可见 |
| `true` | 进行中 | `false` | **隐藏** |
| `true` | 已过期 | `true` | 可见 |

### 5.2 匿名性的多层保障

#### 1. 序列化层过滤

**文件位置**: `app/serializers/rest/poll_serializer.rb`

```ruby
has_many :loaded_options, key: :options

# loaded_options 内部实现：
# 当 show_totals_now? 为 false 时，votes_count 传 nil
Option.new(self, key.to_s, title, show_totals_now? ? (cached_tallies[key] || 0) : nil)
```

#### 2. 广播延迟策略

**文件位置**: `app/services/vote_service.rb:45-49`

```ruby
def distribute_poll!
  return if @poll.hide_totals?  # 匿名投票不实时广播！

  ActivityPub::DistributePollUpdateWorker.perform_in(3.minutes, @poll.status.id)
end
```

**关键设计**:
- 匿名投票（`hide_totals=true`）在投票过程中**完全不触发** `DistributePollUpdateWorker`
- 只有当投票**过期后**，才通过 `PollExpirationNotifyWorker` 触发一次最终结果广播
- 非匿名投票也有 **3 分钟延迟**，防止通过时间差推断投票行为

### 5.3 结果统计数据结构

投票统计使用三种计数字段，各司其职：

```
┌─────────────────────────────────────────────────────────────────┐
│                        Poll 计数字段关系                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  cached_tallies: [12, 8, 15, 5]  ← 每个选项的具体票数          │
│         │                                                        │
│         │ sum()                                                  │
│         ▼                                                        │
│  votes_count: 40  ← 总投票次数（多选时 > voters_count）         │
│                                                                   │
│  voters_count: 28  ← 独立投票人数                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

#### 计数器更新时机

| 计数器 | 更新时机 | 代码位置 |
|--------|----------|----------|
| `cached_tallies[choice]` | 每条 PollVote 创建后 | `poll_vote.rb:34-40` |
| `votes_count` | `cached_tallies` 变更前计算 | `poll.rb:99-101` |
| `voters_count` | 用户首次投票时 | `vote_service.rb:71-79` |

#### 前端百分比计算

**文件位置**: `app/javascript/mastodon/components/poll.tsx:227-232`

```typescript
const percent = useMemo(() => {
  const pollVotesCount = poll.voters_count ?? poll.votes_count;
  return pollVotesCount === 0
    ? 0
    : (option.votes_count / pollVotesCount) * 100;
}, [option, poll]);
```

**优先级**: 优先使用 `voters_count`（投票人数）作为分母，如果为空则使用 `votes_count`（总投票次数）。

---

## 六、前端组件与状态管理

### 6.1 前端数据流架构

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         前端投票数据流程图                                      │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ┌─────────────────┐                                                          │
│  │ Poll.tsx 组件   │◀─────────────────────────────────────────────────────┐ │
│  │ (投票渲染层)     │                                                          │ │
│  └────────┬────────┘                                                          │ │
│           │                                                                    │ │
│           │ 1. useAppSelector 获取 poll                                       │ │
│           │    state.polls[pollId]                                            │ │
│           ▼                                                                    │ │
│  ┌─────────────────────────────────────────┐                                  │ │
│  │           Redux Store (polls 切片)       │                                  │ │
│  │  Record<string, Poll>                     │                                  │ │
│  │  {                                        │                                  │ │
│  │    "123": { id: "123", options: [...], } │                                  │ │
│  │  }                                        │                                  │ │
│  └─────────────────────────────────────────┘                                  │ │
│                    ▲                                                           │ │
│                    │                                                           │ │
│                    │ 3. importPolls action                                    │ │
│                    │                                                           │ │
│  ┌─────────────────────────────────────────┐     ┌────────────────────────┐ │ │
│  │      actions/polls.ts (异步 Thunk)       │     │  actions/importer/    │ │ │
│  │                                         │     │  polls.ts (同步 Action) │ │ │
│  │  • vote({ pollId, choices })            │────▶│                        │ │ │
│  │  • fetchPoll({ pollId })                 │     │  importPolls({ polls })│ │ │
│  │  • importFetchedPoll({ poll })          │     │  (type: 'poll/import'  │ │ │
│  └─────────────────────────────────────────┘     └────────────────────────┘ │ │
│                    │                                                           │ │
│                    │ 2. API 调用                                               │ │
│                    ▼                                                           │ │
│  ┌─────────────────────────────────────────┐                                  │ │
│  │           api/polls.ts                    │                                  │ │
│  │                                         │                                  │ │
│  │  • apiGetPoll(pollId)                    │───▶ GET /api/v1/polls/:id      │ │
│  │  • apiPollVote(pollId, choices)          │───▶ POST /api/v1/polls/:id/   │ │
│  │                                         │     │        votes                 │ │
│  └─────────────────────────────────────────┘                                  │ │
│                                                                                │ │
│  ┌────────────────────────────────────────────────────────────────────────┐  │ │
│  │                    实时更新: streaming.js + WebSocket                   │  │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │ │ │
│  │  │  订阅 channel (public/hashtag/list/user)                         │  │ │ │
│  │  │  ──────────────────────────────────────────────────────────────  │  │ │ │
│  │  │  收到 'update' 事件 → payload 是 status JSON                      │  │ │ │
│  │  │  status.poll → 触发 importPolls → 更新 Redux Store                │  │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘  │ │ │
│  └────────────────────────────────────────────────────────────────────────┘  │ │
│                                                                                │ │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Poll 组件核心逻辑

**文件位置**: `app/javascript/mastodon/components/poll.tsx`

#### 状态与交互管理

```typescript
// 内部状态
const [revealed, setRevealed] = useState(false);      // 用户点击"查看结果"
const [selected, setSelected] = useState<Record<string, boolean>>({});  // 选中的选项

// 派生状态
const showResults = poll.voted || revealed || expired;  // 是否显示结果
```

#### 结果显示决策树

```
showResults = 已投票 OR 用户主动查看 OR 已过期
      │
      ├── 已投票 (voted = true)
      │      └── 显示结果（查看自己投了什么）
      │
      ├── 主动查看 (revealed = true)
      │      └── 点击"See results"按钮触发
      │           注意：匿名投票进行中时，看到的可能不是最新数据
      │
      └── 已过期 (expired = true)
             └── 显示最终结果
```

#### 投票提交流程

```typescript
const handleVote = useCallback(() => {
  if (voteDisabled) return;

  if (identity.signedIn) {
    // 已登录用户：直接调用 vote action
    void dispatch(vote({ pollId, choices: Object.keys(selected) }));
  } else {
    // 未登录用户：打开交互模态框
    dispatch(openModal({
      modalType: 'INTERACTION',
      modalProps: { intent: 'vote', accountId: status.getIn(['account', 'id']), url: status.get('uri') },
    }));
  }
}, [voteDisabled, dispatch, identity, pollId, selected, status]);
```
[app/javascript/mastodon/components/poll.tsx:101-120](app/javascript/mastodon/components/poll.tsx#L101-L120)

#### 手动刷新机制

当 `showResults = true` 时，组件提供"Refresh"按钮：

```typescript
const handleRefresh = useCallback(() => {
  if (disabled) return;
  void dispatch(fetchPoll({ pollId }));  // 手动调用 API 刷新
}, [disabled, dispatch, pollId]);
```
[app/javascript/mastodon/components/poll.tsx:126-131](app/javascript/mastodon/components/poll.tsx#L126-L131)

### 6.3 Redux 状态管理

**文件位置**: `app/javascript/mastodon/reducers/polls.ts`

```typescript
const initialState: Record<string, Poll> = {};

export const pollsReducer: Reducer<PollsState> = (draft = initialState, action) => {
  if (importPolls.match(action)) {
    // 核心更新逻辑：全量替换指定 poll 的数据
    action.payload.polls.forEach((poll) => {
      draft[poll.id] = poll;
    });
  }
  // ... 翻译相关逻辑
  return draft;
};
```
[app/javascript/mastodon/reducers/polls.ts:36-50](app/javascript/mastodon/reducers/polls.ts#L36-L50)

---

## 七、实时更新机制详解

### 7.1 更新渠道概览

Mastodon 投票的实时更新通过**三种互补渠道**实现：

| 更新渠道 | 触发场景 | 实时性 | 覆盖范围 |
|----------|----------|--------|----------|
| **1. 投票后 API 响应** | 用户自己投票后 | 即时 | 仅当前用户 |
| **2. WebSocket 订阅** | 新嘟文/状态更新推送 | 近实时 | 所有订阅用户 |
| **3. 手动刷新按钮** | 用户主动点击 | 按需 | 主动触发 |

### 7.2 渠道一：投票后即时更新

**文件位置**: `app/javascript/mastodon/actions/polls.ts`

```typescript
export const vote = createDataLoadingThunk(
  'poll/vote',
  ({ pollId, choices }: { pollId: string; choices: string[] }) =>
    apiPollVote(pollId, choices),  // 1. POST 到 /api/v1/polls/:id/votes
  async (poll, { dispatch, discardLoadData }) => {
    await dispatch(importFetchedPoll({ poll }));  // 2. API 返回最新 poll 数据
    return discardLoadData;
  },
);
```
[app/javascript/mastodon/actions/polls.ts:24-32](app/javascript/mastodon/actions/polls.ts#L24-L32)

**流程**:
1. 用户点击"Vote"按钮
2. 调用 `POST /api/v1/polls/:id/votes`
3. 后端返回**最新的投票数据**（包含更新后的 `cached_tallies`）
4. 通过 `importFetchedPoll` 更新 Redux Store
5. 组件自动重新渲染显示最新结果

### 7.3 渠道二：WebSocket 订阅推送

**文件位置**: `app/javascript/mastodon/stream.js`

#### Streaming 连接机制

```typescript
// 支持的 channel 类型
const KNOWN_EVENT_TYPES = [
  'update',           // 新嘟文或嘟文更新
  'delete',
  'notification',
  'conversation',
  'filters_changed',
  'announcement',
  'announcement.delete',
  'announcement.reaction',
];

// WebSocket 消息处理
ws.onmessage = e => received(JSON.parse(e.data));
```
[app/javascript/mastodon/stream.js:211-275](app/javascript/mastodon/stream.js#L211-L275)

#### 推送触发时机

当后端执行 `DistributePollUpdateWorker` 时，会触发 `DistributionWorker`，最终通过 Streaming API 推送 `update` 事件：

**文件位置**: `app/workers/activitypub/distribute_poll_update_worker.rb`

```ruby
def perform(status_id)
  @status  = Status.find(status_id)
  @account = @status.account

  return unless @status.preloadable_poll

  # 推送给相关收件箱
  ActivityPub::DeliveryWorker.push_bulk(inboxes, limit: 1_000) do |inbox_url|
    [payload, @account.id, inbox_url]
  end

  relay! if relayable?  # 公开嘟文还会推送给中继
end

# 收集需要推送的实例 inboxes
def inboxes
  @inboxes = [@status.mentions, @status.reblogs, @status.preloadable_poll.votes].flat_map do |relation|
    relation.includes(:account).map do |record|
      record.account.preferred_inbox_url if !record.account.local? && record.account.activitypub?
    end
  end
  @inboxes.concat(@account.followers.inboxes) unless @status.direct_visibility?
  # ...
end
```
[app/workers/activitypub/distribute_poll_update_worker.rb:9-42](app/workers/activitypub/distribute_poll_update_worker.rb#L9-L42)

#### 前端接收与更新

当 WebSocket 收到 `update` 事件时：

1. Payload 是完整的 `status` JSON
2. `status.poll` 字段包含最新投票数据
3. 通过 `importStatuses` → `importPolls` 流程更新 Redux Store

**文件位置**: `app/javascript/mastodon/actions/importer/index.js`

```typescript
export const importFetchedStatus = createAppAsyncThunk(
  'status/importFetched',
  async (status: ApiStatusJSON, { dispatch, getState }) => {
    const { accounts, statuses, polls, filters, mentions } =
      normalizeStatus(status);
    // ...
    dispatch(importPolls({ polls }));  // 从 status 中提取 poll 并更新
    // ...
  },
);
```
[app/javascript/mastodon/actions/importer/index.js:96](app/javascript/mastodon/actions/importer/index.js#L96)

### 7.4 匿名投票的推送策略

这是整个投票系统设计中最精妙的部分：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    匿名投票 vs 非匿名投票的推送差异                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  非匿名投票 (hide_totals = false)                                            │
│  ────────────────────────────────────────────────────────────────────────   │
│                                                                               │
│  用户投票 ──▶ VoteService ──▶ distribute_poll! ──▶ 延迟 3 分钟广播         │
│  (每次投票)          │                │                                       │
│                      │                ▼                                       │
│                      │         DistributePollUpdateWorker                    │
│                      │                │                                       │
│                      │                ▼                                       │
│                      │         ActivityPub 推送 → WebSocket → 前端更新      │
│                      │                                                        │
│                      ▼                                                        │
│                 API 响应中也返回最新数据                                       │
│                 (当前用户立即可见)                                             │
│                                                                               │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                               │
│  匿名投票 (hide_totals = true)                                                │
│  ────────────────────────────────────────────────────────────────────────   │
│                                                                               │
│  用户投票 ──▶ VoteService ──▶ distribute_poll! 立即返回！                   │
│  (每次投票)          │              │                                          │
│                      │              ▼                                          │
│                      │         return if @poll.hide_totals?  ← 直接退出！   │
│                      │                                                         │
│                      │         ❌ 不触发 DistributePollUpdateWorker          │
│                      │         ❌ 不推送 ActivityPub                          │
│                      │         ❌ 不触发 WebSocket 推送                       │
│                      │                                                         │
│                      ▼                                                         │
│                 API 响应中的 poll                                              │
│                      │                                                         │
│                      ▼                                                         │
│                 loaded_options 的 votes_count = nil  ← 隐藏！                │
│                                                                               │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                               │
│  投票过期时 (两种投票类型统一行为)                                              │
│  ────────────────────────────────────────────────────────────────────────   │
│                                                                               │
│  PollExpirationNotifyWorker ──▶ DistributePollUpdateWorker ──▶ 最终广播     │
│         │                              │                                      │
│         │                              ▼                                      │
│         │                    所有用户收到最终结果推送                          │
│         │                                                                      │
│         ▼                                                                      │
│    本地投票者和创建者收到通知                                                    │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.5 手动刷新机制的必要性

**为什么匿名投票需要 Refresh 按钮？**

1. **匿名投票进行中无推送**：`hide_totals=true` 时不触发 `DistributePollUpdateWorker`
2. **WebSocket 无更新**：没有推送，其他用户看不到票数变化
3. **主动刷新**：用户点击"Refresh"调用 `fetchPoll`，但返回的数据中 `votes_count` 仍为 `nil`

**注意**: 即使手动刷新，匿名投票进行中时，`show_totals_now?` 仍返回 `false`，所以前端还是看不到具体票数。Refresh 按钮的主要作用是：
- 获取最新的 `expired` 状态（判断是否已结束）
- 获取可能已更新的其他元数据

---

## 八、ActivityPub 跨实例协作

### 8.1 跨实例投票架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ActivityPub 跨实例投票流程图                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────────────┐                           ┌──────────────────────┐ │
│  │    实例 A (源实例)    │                           │    实例 B (投票者)    │ │
│  │  @alice@mastodon.a   │                           │  @bob@mastodon.b     │ │
│  └──────────┬───────────┘                           └──────────┬───────────┘ │
│             │                                                   │              │
│             │  1. Alice 创建带投票的嘟文                        │              │
│             │     (Poll.expires_at, options, hide_totals)     │              │
│             │                                                   │              │
│             │  2. ActivityPub Create 活动 (Note with Question) │              │
│             │──────────────────────────────────────────────────▶│              │
│             │                                                   │              │
│             │                              3. Bob 在实例 B 看到嘟文和投票      │
│             │                                                   │              │
│             │                              4. Bob 点击投票                      │
│             │                                                   │              │
│             │◀──────────────────────────────────────────────────│              │
│             │  5. ActivityPub Create 活动 (Vote)               │              │
│             │     POST 到 Alice 的 inbox                        │              │
│             │                                                   │              │
│  ┌──────────▼───────────┐                           ┌──────────▼───────────┐ │
│  │  VoteService 处理    │                           │  deliver_votes!      │ │
│  │  - 创建 PollVote     │◀──────────────────────────│  - 构建 Vote JSON    │ │
│  │  - 更新 cached_tallies│                           │  - 投递到源实例 inbox │ │
│  │  - 验证投票有效性     │                           └──────────────────────┘ │
│  └──────────┬───────────┘                                                      │
│             │                                                                  │
│             │  6. 若为非匿名投票，延迟广播更新                                  │
│             │     DistributePollUpdateWorker                                  │
│             │                                                                  │
│             │──────────────────────────────────────────────────▶              │
│             │  7. ActivityPub Update 活动 (更新后的 Question)  │              │
│             │     推送给所有关注者和投票者                     │              │
│  ┌──────────▼───────────┐                           ┌──────────▼───────────┐ │
│  │  实例 A 刷新投票数据  │                           │  实例 B 收到更新      │ │
│  │  本地用户实时可见     │                           │  通过 WebSocket 刷新  │ │
│  └──────────────────────┘                           └──────────────────────┘ │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 投票序列化器

**文件位置**: `app/serializers/activitypub/vote_serializer.rb`

```ruby
# 将 PollVote 序列化为 ActivityPub Vote 对象
class ActivityPub::VoteSerializer < ActiveModel::Serializer
  attributes :id, :type, :actor, :object

  def type
    'Create'
  end

  def actor
    ActivityPub::TagManager.instance.uri_for(object.account)
  end

  def object
    {
      type: 'Note',       # 实际上是 Vote 类型的包装
      name: object.poll.options[object.choice],
      inReplyTo: object.poll.status.uri,
      to: [ActivityPub::TagManager::COLLECTIONS[:public]],
      cc: [object.poll.account.followers_url],
    }
  end
end
```

### 8.3 远程投票解析

**文件位置**: `app/lib/activitypub/parser/poll_parser.rb`

```ruby
# 解析来自远程实例的 Question 对象
module ActivityPub::Parser::PollParser
  def poll
    return nil if object['type'] != 'Question'

    {
      options:    object['oneOf'] || object['anyOf'],  # oneOf=单选, anyOf=多选
      multiple:   object['anyOf'].present?,             
      end_time:   object['endTime'],
      voters_count: object['votersCount'],
    }
  end
end
```

### 8.4 远程投票刷新机制

**文件位置**: `app/controllers/api/v1/polls_controller.rb:24-26`

```ruby
def refresh_poll
  # 如果是远程实例的投票且可能已过期，主动从源实例拉取
  ActivityPub::FetchRemotePollService.new.call(@poll, current_account) if user_signed_in? && @poll.possibly_stale?
end
```

`possibly_stale?` 判断逻辑：
```ruby
def possibly_stale?
  remote? && last_fetched_before_expiration? && time_passed_since_last_fetch?
end

def time_passed_since_last_fetch?
  last_fetched_at.nil? || last_fetched_at < MAKE_FETCH_HAPPEN.ago  # 1.minute
end
```
[app/models/poll.rb:55-57, 117-119](app/models/poll.rb#L55-L57)

---

## 九、关键设计决策总结

### 9.1 并发控制

| 问题 | 解决方案 | 代码位置 |
|------|----------|----------|
| 同一用户重复投票 | Redis 分布式锁 `vote:{poll_id}:{account_id}` | `vote_service.rb:21` |
| 高并发计数不准 | 乐观锁 + 自动重试 `lock_version` | `poll_vote.rb:37-39` |
| 数据库事务一致性 | `ApplicationRecord.transaction` | `vote_service.rb:24-28` |

### 9.2 匿名性保障

| 保障层级 | 实现方式 | 说明 |
|----------|----------|------|
| **数据层** | `hide_totals` 字段控制 `show_totals_now?` | 序列化时决定是否返回票数 |
| **传输层** | 匿名投票跳过 `distribute_poll!` | 不触发 ActivityPub 广播 |
| **时间层** | 非匿名投票也延迟 3 分钟广播 | 防止通过时间关联投票行为 |
| **过期层** | 过期后统一广播最终结果 | 只暴露最终状态，不暴露过程 |

### 9.3 实时性权衡

| 场景 | 更新方式 | 延迟 | 原因 |
|------|----------|------|------|
| 自己投票 | API 响应即时更新 | 0 | 需要立即反馈 |
| 非匿名投票他人更新 | WebSocket 推送 | ~3 分钟 + 网络延迟 | 延迟广播防推断 |
| 匿名投票进行中 | 无推送 | 不更新 | 保护隐私 |
| 投票过期 | WebSocket + 通知 | 即时 | 最终结果可公开 |

---

## 十、数据流时序图

### 10.1 用户投票完整流程

```
┌─────────┐     ┌──────────┐     ┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  用户   │     │ Poll.tsx │     │ Redux Store │     │ VoteService  │     │  PostgreSQL  │
└────┬────┘     └────┬─────┘     └──────┬──────┘     └──────┬───────┘     └──────┬──────┘
     │                │                   │                    │                    │
     │  点击投票按钮    │                   │                    │                    │
     │───────────────▶│                   │                    │                    │
     │                │                   │                    │                    │
     │                │ dispatch(vote())  │                    │                    │
     │                │──────────────────▶│                    │                    │
     │                │                   │                    │                    │
     │                │                   │ apiPollVote()      │                    │
     │                │                   │───────────────────▶│                    │
     │                │                   │                    │                    │
     │                │                   │                    │ Redis 分布式锁      │
     │                │                   │                    │───────────────────▶│
     │                │                   │                    │  获取锁成功         │
     │                │                   │                    │◀───────────────────│
     │                │                   │                    │                    │
     │                │                   │                    │  事务创建 PollVote   │
     │                │                   │                    │───────────────────▶│
     │                │                   │                    │                    │
     │                │                   │                    │  after_create_commit │
     │                │                   │                    │  increment_counter   │
     │                │                   │                    │  更新 cached_tallies │
     │                │                   │                    │───────────────────▶│
     │                │                   │                    │                    │
     │                │                   │                    │  increment_voters_count │
     │                │                   │                    │  (首次投票时)       │
     │                │                   │                    │───────────────────▶│
     │                │                   │                    │                    │
     │                │                   │                    │  非匿名投票: 入队广播 │
     │                │                   │                    │  perform_in(3.min)  │
     │                │                   │                    │                    │
     │                │                   │◀───────────────────│ 返回最新 poll 数据   │
     │                │                   │                    │                    │
     │                │  importPolls()    │                    │                    │
     │                │◀──────────────────│                    │                    │
     │                │                   │                    │                    │
     │  显示投票结果    │                   │                    │                    │
     │◀───────────────│                   │                    │                    │
     │                │                   │                    │                    │
```

### 10.2 投票过期通知流程

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                        Sidekiq 定时任务触发流程                                  │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  创建投票时:                                                                     │
│  PollExpirationNotifyWorker.perform_at(poll.expires_at + 5.minutes, poll.id)│
│                                                                                 │
│  ────────────────────────────────────────────────────────────────────────────  │
│                                                                                 │
│  时间到达 expires_at + 5.minutes:                                              │
│                                                                                 │
│  ┌───────────────────────┐                                                      │
│  │ PollExpirationNotify  │                                                      │
│  │ Worker                │                                                      │
│  └───────────┬───────────┘                                                      │
│              │                                                                  │
│              ├───▶ notify_remote_voters_and_owner! (仅本地投票)                │
│              │           │                                                      │
│              │           ├───▶ DistributePollUpdateWorker.perform_async        │
│              │           │         │                                            │
│              │           │         ▼                                            │
│              │           │    ActivityPub 广播到所有关注者/投票者                │
│              │           │    (包含最终投票结果)                                 │
│              │           │                                                      │
│              │           └───▶ LocalNotificationWorker (通知投票创建者)          │
│              │                                                                  │
│              └───▶ notify_local_voters!                                         │
│                          │                                                      │
│                          ▼                                                      │
│                    LocalNotificationWorker.push_bulk                           │
│                    (批量通知所有本地投票者)                                       │
│                                                                                 │
└────────────────────────────────────────────────────────────────────────────────┘
```

---

## 十一、文件索引

### 后端文件

| 文件路径 | 职责 |
|----------|------|
| `app/models/poll.rb` | Poll 模型、匿名判断、过期判断 |
| `app/models/poll_vote.rb` | PollVote 模型、计数器缓存更新 |
| `app/services/vote_service.rb` | 投票核心业务逻辑、分布式锁 |
| `app/services/update_status_service.rb` | 投票创建/更新、过期任务调度 |
| `app/workers/poll_expiration_notify_worker.rb` | 过期通知与最终结果广播 |
| `app/workers/activitypub/distribute_poll_update_worker.rb` | ActivityPub 投票更新广播 |
| `app/controllers/api/v1/polls_controller.rb` | 投票 API 端点 |
| `app/serializers/rest/poll_serializer.rb` | REST API 投票序列化 |
| `app/validators/vote_validator.rb` | 投票有效性验证 |
| `app/validators/poll_options_validator.rb` | 投票选项验证 |

### 前端文件

| 文件路径 | 职责 |
|----------|------|
| `app/javascript/mastodon/components/poll.tsx` | 投票渲染组件、交互逻辑 |
| `app/javascript/mastodon/features/compose/components/poll_form.jsx` | 投票创建表单 |
| `app/javascript/mastodon/actions/polls.ts` | 投票相关 Redux actions |
| `app/javascript/mastodon/reducers/polls.ts` | 投票 Redux reducer |
| `app/javascript/mastodon/api/polls.ts` | 投票 API 调用封装 |
| `app/javascript/mastodon/models/poll.ts` | 投票前端模型 |
| `app/javascript/mastodon/stream.js` | WebSocket 实时订阅 |

---

## 十二、设计亮点总结

1. **匿名投票的多层隐私保护**: 从数据层、传输层、时间层全方位保护投票隐私，只有最终结果公开

2. **分布式锁 + 乐观锁双重并发控制**: Redis 锁防重复投票，数据库乐观锁防计数冲突

3. **延迟广播防推断**: 即使非匿名投票也延迟 3 分钟广播，防止通过时间差关联用户行为

4. **过期时统一通知**: 无论匿名与否，过期时统一广播最终结果，保证所有用户看到一致的最终状态

5. **ActivityPub 原生兼容**: 使用 Question/Note/Vote 等标准类型，无缝融入联邦宇宙

6. **远程投票智能刷新**: `possibly_stale?` 机制自动检测远程投票是否需要从源实例刷新
