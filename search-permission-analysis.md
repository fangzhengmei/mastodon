# Mastodon 全文搜索权限裁剪分析报告

## 1. 概述

Mastodon 的全文搜索采用了**双层权限控制机制**，确保搜索结果只对有权限的用户可见。权限裁剪发生在两个关键环节：

1. **索引构建阶段**：通过 `searchable_by` 字段预计算可访问用户列表
2. **查询执行阶段**：在获取搜索结果后进行二次过滤

## 2. 整体架构流程图

```
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│   内容创建/更新     │────▶│   索引更新流程      │────▶│   Elasticsearch     │
│   (Status/Account)  │     │   (Chewy Strategy)  │     │   索引存储          │
└─────────────────────┘     └─────────────────────┘     └─────────────────────┘
                                                               │
                                                               ▼
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│   搜索结果返回      │◀────│   结果过滤阶段      │◀────│   查询执行阶段      │
│   (StatusFilter)    │     │   (二次过滤)        │     │   (ES查询)          │
└─────────────────────┘     └─────────────────────┘     └─────────────────────┘
```

## 3. 索引更新阶段（权限预计算）

### 3.1 触发机制

当内容（Status、Account、Tag）被创建或更新时，Chewy gem 的 Mastodon 策略会捕获这些变更：

**文件位置**：`lib/chewy/strategy/mastodon.rb:12-27`

- 当模型发生变更时，`update` 方法被调用
- 变更记录被添加到 Redis 队列 `chewy:queue:{index_name}`
- 由 `Scheduler::IndexingScheduler` 定期处理队列并更新 Elasticsearch 索引

### 3.2 索引定义

Mastodon 使用两个独立的状态索引：

#### PublicStatusesIndex（公开状态索引）

**文件位置**：`app/chewy/public_statuses_index.rb:55-58`

```ruby
index_scope ::Status.unscoped
  .kept
  .indexable
  .includes(:media_attachments, :preloadable_poll, :tags, preview_cards_status: :preview_card)
```

- 仅索引**公开可见性**（public_visibility）的状态
- 要求作者账户设置为可索引（indexable: true）
- 不包含 `searchable_by` 字段，对所有用户可见

#### StatusesIndex（私有状态索引）

**文件位置**：`app/chewy/statuses_index.rb:55-66`

```ruby
index_scope ::Status.unscoped.kept.without_reblogs.includes(...)
  delete_if: ->(status) { status.searchable_by.empty? }

field(:searchable_by, type: 'long', value: ->(status) { status.searchable_by })
```

- 索引所有非转发的状态
- 如果 `searchable_by` 为空，则不索引该状态
- 包含 `searchable_by` 字段，存储有权限访问该状态的用户 ID 列表

### 3.3 searchable_by 字段计算

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

**权限裁剪环节 1**：在索引构建阶段，通过 `searchable_by` 字段预计算哪些用户有权限搜索到该状态。这是第一层权限控制。

## 4. 查询执行阶段（搜索请求处理）

### 4.1 搜索服务入口

**文件位置**：`app/services/search_service.rb:6-27`

```ruby
def call(query, account, limit, options = {})
  @query     = query&.strip&.gsub(QUOTE_EQUIVALENT_CHARACTERS, '"')
  @account   = account
  @options   = options
  @limit     = limit.to_i
  @offset    = options[:type].blank? ? 0 : options[:offset].to_i

  default_results.tap do |results|
    next if @query.blank? || @limit.zero?

    if url_query?
      results.merge!(url_resource_results)
    elsif @query.present?
      results[:accounts] = perform_accounts_search! if account_searchable?
      results[:statuses] = perform_statuses_search! if status_searchable?
      results[:hashtags] = perform_hashtags_search! if hashtag_searchable?
    end
  end
end
```

### 4.2 状态搜索服务

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

### 4.3 查询构建（SearchQueryTransformer）

**文件位置**：`app/lib/search_query_transformer.rb:25-98`

查询构建阶段会应用默认过滤器，这是**权限裁剪环节 2**：

```ruby
def request
  search = Chewy::Search::Request.new(*indexes).filter(default_filter)

  must_clauses.each { |clause| search = search.query.must(clause.to_query) }
  must_not_clauses.each { |clause| search = search.query.must_not(clause.to_query) }
  filter_clauses.each { |clause| search = search.filter(**clause.to_query) }

  search
end

def default_filter
  {
    bool: {
      should: [
        {
          term: {
            _index: PublicStatusesIndex.index_name,  # 所有公开索引的内容都可搜索
          },
        },
        {
          bool: {
            must: [
              {
                term: {
                  _index: StatusesIndex.index_name,
                },
              },
              {
                term: {
                  searchable_by: @options[:current_account].id,  # 私有索引需要匹配当前用户ID
                },
              },
            ],
          },
        },
      ],

      minimum_should_match: 1,
    },
  }
end
```

**查询阶段权限逻辑**：
- 对于 `PublicStatusesIndex`：所有内容都可被搜索（无需权限检查）
- 对于 `StatusesIndex`：只有 `searchable_by` 字段包含当前用户 ID 的状态才会被返回

## 5. 结果过滤阶段（二次过滤）

### 5.1 StatusFilter 过滤器

**文件位置**：`app/services/statuses_search_service.rb:35`

```ruby
results.reject { |status| StatusFilter.new(status, @account).filtered? }
```

这是**权限裁剪环节 3**，对 Elasticsearch 返回的结果进行二次过滤。

### 5.2 StatusFilter 详细实现

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

### 5.3 StatusPolicy 权限检查

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

## 6. 权限裁剪环节总结

| 环节 | 阶段 | 实现位置 | 检查内容 |
|------|------|----------|----------|
| 1 | 索引构建 | `Status#searchable_by` | 预计算可访问用户列表（作者、提及、点赞、转发、收藏、投票者） |
| 2 | 查询执行 | `SearchQueryTransformer#default_filter` | ES查询时过滤：公开索引全部可见，私有索引匹配 `searchable_by` |
| 3 | 结果过滤 | `StatusFilter` + `StatusPolicy` | 二次检查：可见性级别、拉黑/静音状态、作者可用性 |

## 7. 技术设计特点

### 7.1 双层过滤的优势

1. **性能优化**：
   - 索引阶段预计算权限，减少查询时的计算量
   - ES 查询阶段过滤掉大部分无权限的内容
   - 结果阶段只对少量候选结果进行精细检查

2. **安全性保障**：
   - 即使索引权限计算有误，结果阶段的二次过滤仍能保障安全
   - 动态变化的关系（如拉黑、关注）在结果阶段实时检查

3. **灵活性**：
   - `searchable_by` 字段可以覆盖复杂的权限场景
   - `StatusPolicy` 可以实现精细化的权限规则

### 7.2 潜在注意事项

1. **索引更新延迟**：
   - 权限关系变更（如关注、拉黑）不会立即反映在 `searchable_by` 字段中
   - 需要等待状态重新索引才能更新权限信息
   - 但二次过滤可以弥补这一延迟

2. **索引膨胀**：
   - 对于热门状态，`searchable_by` 可能包含大量用户 ID
   - 但 Mastodon 限制了只索引本地用户，减轻了这个问题

## 8. 关键代码位置汇总

| 功能模块 | 文件路径 |
|----------|----------|
| 搜索服务入口 | `app/services/search_service.rb` |
| 状态搜索服务 | `app/services/statuses_search_service.rb` |
| 查询转换器 | `app/lib/search_query_transformer.rb` |
| 状态过滤器 | `app/lib/status_filter.rb` |
| 状态权限策略 | `app/policies/status_policy.rb` |
| 状态搜索扩展 | `app/models/concerns/status/search_concern.rb` |
| 公开状态索引 | `app/chewy/public_statuses_index.rb` |
| 私有状态索引 | `app/chewy/statuses_index.rb` |
| Chewy 策略 | `lib/chewy/strategy/mastodon.rb` |
| 索引调度器 | `app/workers/scheduler/indexing_scheduler.rb` |
