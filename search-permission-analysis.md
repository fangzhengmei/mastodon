# Mastodon 全文搜索权限裁剪分析报告

## 1. 概述

Mastodon 的全文搜索采用了**多层权限控制机制**，确保搜索结果只对有权限的用户可见。权限控制发生在多个关键环节：

1. **登录态门禁**：未登录用户无法进入状态全文搜索链路
2. **索引构建阶段**：通过 `searchable_by` 字段预计算可访问用户列表
3. **查询执行阶段**：根据 `in:` 参数选择不同索引并应用权限过滤
4. **结果过滤阶段**：对返回结果进行二次精细权限检查

## 2. 整体架构流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         搜索请求入口 (API Controller)                    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  登录态门禁: @account.present? (SearchService#status_searchable?) │    │
│  │  - 未登录用户: 直接跳过状态搜索，返回空数组                         │    │
│  │  - 已登录用户: 进入完整搜索链路                                   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        查询参数解析 (SearchQueryParser)                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  in: 参数解析 (SearchQueryTransformer#indexes)                    │    │
│  │  - in:public  → 只搜索 PublicStatusesIndex                       │    │
│  │  - in:library → 只搜索 StatusesIndex                             │    │
│  │  - 无参数     → 同时搜索两个索引                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        索引构建阶段 (权限预计算)                         │
│  ┌──────────────────────┐    ┌──────────────────────────────────┐      │
│  │ PublicStatusesIndex  │    │         StatusesIndex             │      │
│  │ - 公开可见性状态      │    │ - 所有非转发状态                   │      │
│  │ - 作者可索引          │    │ - 含 searchable_by 字段           │      │
│  │ - 无 searchable_by    │    │ - 预计算可访问用户ID列表           │      │
│  └──────────────────────┘    └──────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        查询执行阶段 (ES查询过滤)                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  default_filter 权限过滤 (SearchQueryTransformer)                │    │
│  │  - PublicStatusesIndex: 无条件可见                               │    │
│  │  - StatusesIndex: searchable_by 包含当前用户ID                   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        结果过滤阶段 (二次检查)                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  StatusFilter + StatusPolicy 精细检查                             │    │
│  │  - 作者不可用: 过滤                                             │    │
│  │  - 可见性级别检查: 私信/私有/公开                                │    │
│  │  - 拉黑/静音关系检查                                            │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

## 3. 登录态门禁：未登录用户无法进入状态搜索链路

### 3.1 门禁位置与实现

状态全文搜索有**两层登录态检查**，确保未登录用户无法进入该搜索链路：

#### 第一层：API 控制器层

**文件位置**：`app/controllers/api/v2/search_controller.rb:9-15`

```ruby
before_action -> { authorize_if_got_token! :read, :'read:search' }
before_action :validate_search_params!

with_options unless: :user_signed_in? do
  before_action :query_pagination_error, if: :pagination_requested?
  before_action :remote_resolve_error, if: :remote_resolve_requested?
end
```

虽然控制器允许未登录用户进行基础搜索（需 `authorize_if_got_token!`），但关键限制在服务层。

#### 第二层：SearchService 服务层（关键门禁）

**文件位置**：`app/services/search_service.rb:86-88`

```ruby
def status_searchable?
  Chewy.enabled? && status_search? && @account.present?
end
```

这是**关键的登录态门禁**。让我们查看 `perform_statuses_search!` 的调用逻辑：

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

### 3.2 为什么未登录用户不会进入状态搜索链路

`status_searchable?` 方法的三个条件：

| 条件 | 说明 | 未登录时的值 |
|------|------|-------------|
| `Chewy.enabled?` | Elasticsearch 是否启用 | 可能为 true |
| `status_search?` | 搜索类型是否包含 statuses | 可能为 true |
| `@account.present?` | 当前是否有登录账户 | **false** |

**关键结论**：
- 当 `@account.nil?`（未登录）时，`status_searchable?` 返回 `false`
- 因此 `perform_statuses_search!` 永远不会被调用
- `results[:statuses]` 保持为默认值 `[]`（空数组）
- 未登录用户的状态搜索结果始终为空

### 3.3 门禁的设计意图

这个门禁设计有以下考虑：

1. **隐私保护**：状态搜索可能包含用户的私人互动（如点赞、收藏的状态）
2. **避免滥用**：未登录用户无法大规模搜索状态内容
3. **性能优化**：减少无效的 Elasticsearch 查询
4. **与索引设计匹配**：`StatusesIndex` 的 `searchable_by` 字段需要用户 ID 进行过滤

## 4. 索引更新阶段（权限预计算）

### 4.1 触发机制

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

### 4.2 双索引设计

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
- 设计目标：对所有用户（包括未登录）可搜索的公开内容

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

### 4.3 searchable_by 字段计算

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

## 5. in:public 与 in:library 查询参数分析

### 5.1 索引选择逻辑

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

### 5.2 in:public 与 in:library 的详细对比

| 特性 | in:public | in:library | 默认（无参数） |
|------|-----------|------------|----------------|
| 搜索索引 | PublicStatusesIndex | StatusesIndex | 两个索引 |
| 内容范围 | 公开可见性状态 | 用户互动过的状态 | 全部 |
| 索引条件 | public_visibility + 作者 indexable | 非转发 + searchable_by 非空 | - |
| 权限过滤方式 | 无（索引时已筛选） | searchable_by 匹配 | 组合过滤 |
| 典型用例 | 搜索公开推文 | 搜索自己点赞/收藏过的内容 | 综合搜索 |

### 5.3 不同参数下的权限裁剪环节

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
| 1. 登录态门禁 | 生效 | 仍需 `@account.present?` 才能进入状态搜索链路 |
| 2. 索引构建阶段 | 已预筛选 | PublicStatusesIndex 只包含公开可见性状态 |
| 3. 查询执行阶段 | 简化 | 只命中 `_index == PublicStatusesIndex` 分支，无额外权限过滤 |
| 4. 结果过滤阶段 | **完整生效** | StatusFilter + StatusPolicy 仍会检查：<br>- 作者是否拉黑当前用户<br>- 作者是否拉黑当前用户域名<br>- 当前用户是否拉黑/静音作者 |

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
| 1. 登录态门禁 | 生效 | 必须登录才能使用 |
| 2. 索引构建阶段 | 生效 | `searchable_by` 预计算可访问用户列表 |
| 3. 查询执行阶段 | **关键过滤** | ES 查询时要求 `searchable_by` 包含当前用户 ID |
| 4. 结果过滤阶段 | **完整生效** | 额外检查动态关系（拉黑、静音等） |

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

## 6. 查询执行阶段（搜索请求处理）

### 6.1 搜索服务入口

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

### 6.2 状态搜索服务

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

### 6.3 查询构建与权限过滤

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

## 7. 结果过滤阶段（二次过滤）

### 7.1 StatusFilter 过滤器

**文件位置**：`app/services/statuses_search_service.rb:35`

```ruby
results.reject { |status| StatusFilter.new(status, @account).filtered? }
```

**权限裁剪环节 3（结果过滤阶段）**：对 Elasticsearch 返回的结果进行二次精细过滤。

### 7.2 StatusFilter 详细实现

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

### 7.3 StatusPolicy 权限检查

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

## 8. 权限裁剪环节完整总结

### 8.1 整体环节表

| 环节 | 阶段 | 实现位置 | 检查内容 | 未登录状态 | in:public | in:library |
|------|------|----------|----------|------------|-----------|------------|
| 0 | 登录态门禁 | `SearchService#status_searchable?` | `@account.present?` | ❌ 不进入链路 | ✅ 需登录 | ✅ 需登录 |
| 1 | 索引构建 | `Status#searchable_by` | 预计算可访问用户列表 | N/A | N/A（公开索引无此字段） | ✅ 生效 |
| 2 | 查询执行 | `SearchQueryTransformer#indexes` | 根据 `in:` 参数选择索引 | N/A | `[PublicStatusesIndex]` | `[StatusesIndex]` |
| 3 | 查询执行 | `SearchQueryTransformer#default_filter` | ES查询时的权限过滤 | N/A | 简化（只检查索引名） | ✅ `searchable_by` 匹配 |
| 4 | 结果过滤 | `StatusFilter` + `StatusPolicy` | 二次检查：可见性、拉黑/静音 | N/A | ✅ 完整检查 | ✅ 完整检查 |

### 8.2 不同场景的权限裁剪流程

#### 场景 A：未登录用户搜索状态

```
用户请求 → API Controller → SearchService#status_searchable?
                                    ↓
                           @account.present? == false
                                    ↓
                           perform_statuses_search! 不执行
                                    ↓
                           results[:statuses] = []
```

**结果**：始终返回空数组

#### 场景 B：已登录用户 + in:public

```
用户请求 → 登录态门禁通过 → 解析 in:public
                                    ↓
                          indexes = [PublicStatusesIndex]
                                    ↓
                          default_filter: _index 匹配即可
                                    ↓
                          ES 返回公开状态
                                    ↓
                          StatusFilter 二次检查
                                    ↓
                          过滤掉被拉黑/静音的内容
```

#### 场景 C：已登录用户 + in:library

```
用户请求 → 登录态门禁通过 → 解析 in:library
                                    ↓
                          indexes = [StatusesIndex]
                                    ↓
                          default_filter: searchable_by 包含当前用户ID
                                    ↓
                          ES 返回用户互动过的状态
                                    ↓
                          StatusFilter 二次检查
                                    ↓
                          过滤掉被拉黑/静音的内容
```

#### 场景 D：已登录用户 + 默认查询

```
用户请求 → 登录态门禁通过 → 无 in: 参数
                                    ↓
                    indexes = [PublicStatusesIndex, StatusesIndex]
                                    ↓
                    default_filter: 任一索引条件满足即可
                                    ↓
                    ES 返回两个索引的合并结果
                                    ↓
                    StatusFilter 二次检查
                                    ↓
                    返回最终结果
```

## 9. 技术设计特点

### 9.1 多层过滤的优势

1. **性能优化**：
   - 登录态门禁：在最早阶段拦截无效请求
   - 索引阶段预计算权限，减少查询时的计算量
   - ES 查询阶段过滤掉大部分无权限的内容
   - 结果阶段只对少量候选结果进行精细检查

2. **安全性保障**：
   - 未登录用户完全无法进入状态搜索链路
   - `in:public` 和 `in:library` 提供明确的内容隔离
   - 即使索引权限计算有误，结果阶段的二次过滤仍能保障安全
   - 动态变化的关系（如拉黑、关注）在结果阶段实时检查

3. **灵活性**：
   - `searchable_by` 字段可以覆盖复杂的权限场景
   - `in:` 参数允许用户精确控制搜索范围
   - `StatusPolicy` 可以实现精细化的权限规则

### 9.2 潜在注意事项

1. **索引更新延迟**：
   - 权限关系变更（如关注、拉黑）不会立即反映在 `searchable_by` 字段中
   - 需要等待状态重新索引才能更新权限信息
   - 但二次过滤可以弥补这一延迟

2. **索引膨胀**：
   - 对于热门状态，`searchable_by` 可能包含大量用户 ID
   - 但 Mastodon 限制了只索引本地用户，减轻了这个问题

3. **登录态门禁的严格性**：
   - 即使使用 `in:public` 只搜索公开内容，也需要登录
   - 这是设计决策，可能限制了未登录用户的搜索体验
   - 但增强了隐私保护和防止滥用

## 10. 关键代码位置汇总

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| 登录态门禁 | `app/services/search_service.rb` | 86-88 |
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
