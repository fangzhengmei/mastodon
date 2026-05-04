# 帖子过滤器和内容警告实现分析

## 一、整体架构

Mastodon 的帖子隐藏和折叠功能由两个独立但互补的系统实现：

1. **帖子过滤器系统**：基于用户定义的关键词/规则自动过滤帖子
2. **内容警告系统**：基于帖子自带的 `spoiler_text` 字段实现折叠显示

---

## 二、服务端过滤流程（从产生到前端生效）

### 2.1 服务端数据模型

#### CustomFilter 模型 (`app/models/custom_filter.rb`)

这是过滤器的核心数据模型，定义了三种动作类型：

```ruby
# 数据库字段
#  action     :integer          default("warn"), not null

# 动作类型枚举
enum :action, { warn: 0, hide: 1, blur: 2 }, suffix: :action, validate: true

# 别名
alias_attribute :title, :phrase
alias_attribute :filter_action, :action

# 有效上下文
VALID_CONTEXTS = %w(
  home
  notifications
  public
  thread
  account
).freeze
```

**三种动作类型的含义**：
| 动作值 | 枚举名 | 服务端行为 |
|--------|--------|-----------|
| 0 | `warn` | 在 `filtered` 数组中标记，前端决定显示警告 |
| 1 | `hide` | 在 `filtered` 数组中标记，前端决定是否完全隐藏 |
| 2 | `blur` | 在 `filtered` 数组中标记，前端决定是否模糊媒体 |

**重要**：服务端只负责**标记**帖子是否匹配过滤器，**不执行**实际的隐藏/折叠/模糊操作。这些行为由前端根据 `filter_action` 决定。

### 2.2 缓存机制

#### 过滤器缓存 (`app/models/custom_filter.rb:70-93`)

为了提高性能，过滤器结果会被缓存：

```ruby
def self.cached_filters_for(account_id)
  active_filters = Rails.cache.fetch("filters:v3:#{account_id}") do
    filters_hash = {}

    # 1. 处理关键词过滤器
    scope = CustomFilterKeyword.left_outer_joins(:custom_filter)
      .merge(unexpired.where(account_id: account_id))

    scope.to_a.group_by(&:custom_filter).each do |filter, keywords|
      keywords.map!(&:to_regex)
      filters_hash[filter.id] = { keywords: Regexp.union(keywords), filter: filter }
    end.to_h

    # 2. 处理特定帖子过滤器
    scope = CustomFilterStatus.left_outer_joins(:custom_filter)
      .merge(unexpired.where(account_id: account_id))

    scope.to_a.group_by(&:custom_filter).each do |filter, statuses|
      filters_hash[filter.id] ||= { filter: filter }
      filters_hash[filter.id].merge!(status_ids: statuses.map(&:status_id))
    end

    filters_hash.values.map { |cache| [cache.delete(:filter), cache] }
  end.to_a

  active_filters.reject { |custom_filter, _| custom_filter.expired? }
end
```

**缓存结构**：
```ruby
[
  [filter_object, { keywords: regexp, status_ids: [1, 2, 3] }],
  [filter_object, { keywords: regexp }],
  [filter_object, { status_ids: [4, 5] }],
]
```

**缓存失效**：
- 当过滤器创建/更新/删除时，`invalidate_cache!` 会被调用
- 同时通过 Redis 发布 `filters_changed` 事件通知前端

### 2.3 过滤匹配逻辑

#### 应用过滤器 (`app/models/custom_filter.rb:95-106`)

```ruby
def self.apply_cached_filters(cached_filters, status)
  cached_filters.filter_map do |filter, rules|
    # 1. 关键词匹配
    match = rules[:keywords].match(status.proper.searchable_text) if rules[:keywords].present?
    keyword_matches = [match.to_s] unless match.nil?

    # 2. 特定帖子匹配
    status_matches = [status.id, status.reblog_of_id].compact & rules[:status_ids] if rules[:status_ids].present?

    # 3. 任一种匹配成功则返回结果
    next if keyword_matches.blank? && status_matches.blank?

    FilterResultPresenter.new(
      filter: filter,
      keyword_matches: keyword_matches,
      status_matches: status_matches
    )
  end
end
```

**匹配逻辑说明**：
- **关键词匹配**：使用正则表达式匹配 `status.searchable_text`（包含内容、CW文本、投票选项、媒体描述等）
- **特定帖子匹配**：检查 `status.id` 或 `status.reblog_of_id` 是否在过滤器的 `status_ids` 列表中
- **OR 逻辑**：任一种匹配成功即视为过滤器命中

### 2.4 批量处理和序列化

#### StatusRelationshipsPresenter (`app/presenters/status_relationships_presenter.rb`)

在 API 响应中，过滤器匹配是批量处理的：

```ruby
def initialize(statuses, current_account_id = nil, **options)
  # ...
  @filters_map = build_filters_map(
    statuses.flat_map { |s| [s, s.proper.quote&.quoted_status] }.compact.uniq,
    current_account_id
  ).merge(options[:filters_map] || {})
  # ...
end

private

def build_filters_map(statuses, current_account_id)
  active_filters = CustomFilter.cached_filters_for(current_account_id)

  @filters_map = statuses.each_with_object({}) do |status, h|
    filter_matches = CustomFilter.apply_cached_filters(active_filters, status)

    unless filter_matches.empty?
      h[status.id] = filter_matches
      h[status.reblog_of_id] = filter_matches if status.reblog?
    end
  end
end
```

**关键点**：
1. 不仅处理主帖子，还处理引用的帖子 (`quoted_status`)
2. 转发帖子 (`reblog`) 的过滤结果同时关联到原帖和转发帖

#### 序列化器链路

**帖子序列化器** (`app/serializers/rest/status_serializer.rb`)：

```ruby
# 帖子序列化器中的 filtered 属性
has_many :filtered, serializer: REST::FilterResultSerializer, if: :current_user?

def filtered
  if relationships
    # 优先使用批量处理的结果（性能优化）
    relationships.filters_map[object.id] || []
  else
    # 单个帖子时实时计算
    current_user.account.status_matches_filters(object)
  end
end
```

**FilterResult 序列化器** (`app/serializers/rest/filter_result_serializer.rb`)：

```ruby
class REST::FilterResultSerializer < ActiveModel::Serializer
  belongs_to :filter, serializer: REST::FilterSerializer
  has_many :keyword_matches
  has_many :status_matches

  def status_matches
    object.status_matches&.map(&:to_s)
  end
end
```

**Filter 序列化器** (`app/serializers/rest/filter_serializer.rb`)：

```ruby
class REST::FilterSerializer < ActiveModel::Serializer
  attributes :id, :title, :context, :expires_at, :filter_action
  has_many :keywords, serializer: REST::FilterKeywordSerializer, if: :rules_requested?
  has_many :statuses, serializer: REST::FilterStatusSerializer, if: :rules_requested?
end
```

**重要发现**：
- `filter_action` **总是**被序列化输出
- `keywords` 和 `statuses` 只有在 `rules_requested?` 为 true 时才输出（即主动调用 `/api/v2/filters` 时）

### 2.5 API 响应结构

服务端返回的帖子数据中 `filtered` 字段结构：

```json
{
  "id": "12345",
  "content": "...",
  "filtered": [
    {
      "filter": {
        "id": "678",
        "title": "关键词过滤器",
        "context": ["home", "notifications"],
        "expires_at": "2026-06-01T00:00:00.000Z",
        "filter_action": "warn"  // 或 "hide" / "blur"
      },
      "keyword_matches": ["匹配到的关键词"],
      "status_matches": []
    }
  ]
}
```

**注意**：这里的 `filter` 对象包含完整的 `filter_action` 字段，这是前端决定如何处理的关键依据。

---

## 三、服务端到前端的完整数据流

### 3.1 前端数据接收和处理链路

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      服务端输出 → 前端处理 完整链路                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  【服务端】                                                                   │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ REST::StatusSerializer#filtered                                       │  │
│  │ → 包含 filter 对象，其中有 filter_action: "warn" | "hide" | "blur"  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                              │                                               │
│                              ▼ HTTP Response                                 │
│  {                                                                          │
│    "id": "12345",                                                           │
│    "filtered": [{                                                           │
│      "filter": {                                                            │
│        "id": "678",                                                         │
│        "filter_action": "warn"  // ✅ 完整信息                             │
│      },                                                                      │
│      "keyword_matches": ["..."]                                             │
│    }]                                                                        │
│  }                                                                          │
│                              │                                               │
│                              ▼                                               │
│  【前端 importFetchedStatuses】                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ app/javascript/mastodon/actions/importer/index.js:58-100             │  │
│  │                                                                         │  │
│  │ 阶段 A: 提取完整的 filter 对象                                          │  │
│  │ ─────────────────────────────────────                                  │  │
│  │ if (status.filtered) {                                                 │  │
│  │   status.filtered.forEach(result =>                                    │  │
│  │     pushUnique(filters, result.filter)  // ✅ 包含 filter_action      │  │
│  │   )                                                                     │  │
│  │ }                                                                       │  │
│  │                                                                         │  │
│  │ 阶段 B: 规范化帖子（filter 被替换为 ID）                                 │  │
│  │ ──────────────────────────────────────────                             │  │
│  │ pushUnique(normalStatuses, normalizeStatus(status, ...))              │  │
│  │                                                                         │  │
│  │ 阶段 C: 分发动作                                                        │  │
│  │ ────────────────────                                                   │  │
│  │ dispatch(importFilters(filters));        // → state.filters          │  │
│  │ dispatch(importStatuses(normalStatuses)); // → state.statuses[id]     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                              │                                               │
│                              ▼                                               │
│  【前端 normalizeStatus】                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ app/javascript/mastodon/actions/importer/normalizer.js:15-21,66-68  │  │
│  │                                                                         │  │
│  │ export function normalizeFilterResult(result) {                        │  │
│  │   const normalResult = { ...result };                                  │  │
│  │   normalResult.filter = normalResult.filter.id;  // ⚠️ 只保留 ID！     │  │
│  │   return normalResult;                                                  │  │
│  │ }                                                                       │  │
│  │                                                                         │  │
│  │ // 在 normalizeStatus 中:                                              │  │
│  │ if (status.filtered) {                                                 │  │
│  │   normalStatus.filtered = status.filtered.map(normalizeFilterResult); │  │
│  │ }                                                                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                              │                                               │
│                              ▼                                               │
│  【Redux State 最终结构】                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                                                                         │  │
│  │ state.filters (Map)                                                    │  │
│  │ ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │ │ "678": {                                                          │  │  │
│  │ │   id: "678",                                                      │  │  │
│  │ │   title: "关键词过滤器",                                           │  │  │
│  │ │   filter_action: "warn",  // ✅ 完整信息，来自 importFilters     │  │  │
│  │ │   context: ["home", "notifications"],                             │  │  │
│  │ │   ...                                                              │  │  │
│  │ │ }                                                                  │  │  │
│  │ └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                         │  │
│  │ state.statuses[id].filtered (List)                                     │  │
│  │ ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │ │ [{                                                                 │  │  │
│  │ │   filter: "678",  // ⚠️ 只是字符串 ID！                           │  │  │
│  │ │   keyword_matches: ["..."],                                        │  │  │
│  │ │   status_matches: []                                                │  │  │
│  │ │ }]                                                                  │  │  │
│  │ └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                         │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                              │                                               │
│                              ▼                                               │
│  【选择器结合两者】                                                           │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ app/javascript/mastodon/selectors/index.js                           │  │
│  │                                                                         │  │
│  │ // 1. 从 status.filtered 获取匹配结果（filter 只是 ID）               │  │
│  │ let filterResults = statusReblog?.get('filtered') || ...             │  │
│  │                                                                         │  │
│  │ // 2. 从 state.filters 获取完整的 filter 对象                         │  │
│  │ // filters = getFilters(state, { contextType })                      │  │
│  │                                                                         │  │
│  │ // 3. 结合两者判断动作类型                                             │  │
│  │ if (filterResults.some((result) =>                                    │  │
│  │     filters.getIn([result.get('filter'), 'filter_action']) === 'hide' │  │
│  │ )) {                                                                    │  │
│  │   // 完全隐藏                                                          │  │
│  │   return { status: null, loadingState: 'filtered' }                  │  │
│  │ }                                                                       │  │
│  │                                                                         │  │
│  │ // 同理检查 'blur' 和 'warn'                                           │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 过滤器数据的两种来源

前端有两种方式获取过滤器数据：

| 来源 | 触发时机 | 动作类型 | 数据完整性 |
|------|---------|---------|-----------|
| **A. 帖子数据附带** | 加载时间线/帖子时 | `FILTERS_IMPORT` | 部分（无 keywords/statuses，但有 filter_action） |
| **B. 主动 API 调用** | 打开过滤器管理界面时 | `FILTERS_FETCH_SUCCESS` | 完整（有所有字段） |

**Reducer 处理差异** (`app/javascript/mastodon/reducers/filters.js`)：

```javascript
case FILTERS_CREATE_SUCCESS:
  return normalizeFilter(state, action.filter);
case FILTERS_FETCH_SUCCESS:
  return normalizeFilters(ImmutableMap(), action.filters);  // ⚠️ 完全替换！
case FILTERS_IMPORT:
  return normalizeFilters(state, action.filters);  // ⚠️ 增量合并！
```

**重要**：
- `FILTERS_FETCH_SUCCESS` 会**完全替换** `state.filters`
- `FILTERS_IMPORT` 会**增量合并**，使用 `mergeWith` 策略

```javascript
// normalizeFilter 中的合并策略
return state.update(filter.id, ImmutableMap(), (old) => (
  old.mergeWith(
    ((old_value, new_value) => (new_value === undefined ? old_value : new_value)),
    normalizedFilter
  )
));
```

这意味着：
- 如果 `new_value` 是 `undefined`，保留旧值
- 这确保了从帖子数据附带的部分过滤器信息（无 keywords）不会覆盖已有的完整信息

### 3.3 完整的数据流时序

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   服务端      │     │   前端 API    │     │  Redux State │
└──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │
       │  GET /api/v1/timelines/home            │
       │◄───────────────────────────────────────│
       │                    │                    │
       │  Response with posts                   │
       │  每个帖子包含 filtered 数组             │
       │  filtered[].filter.filter_action       │
       ────────────────────────────────────────►│
                            │                    │
                            ▼                    │
                    importFetchedStatuses()     │
                            │                    │
              ┌─────────────┴─────────────┐    │
              │                           │    │
              ▼                           ▼    │
       提取 filter 对象          规范化帖子     │
       (完整信息)              (filter→ID)     │
              │                           │    │
              ▼                           ▼    │
       dispatch(importFilters)    dispatch(importStatuses)
              │                           │    │
              ▼                           ▼    │
       ┌─────────────────────────────────────────┐
       │           Redux State 更新               │
       │  state.filters[id].filter_action = "warn"│
       │  state.statuses[id].filtered[0].filter = "id"│
       └─────────────────────────────────────────┘
                            │
                            ▼
                    选择器应用过滤逻辑
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
    从 status.filtered 获取      从 state.filters 获取
    filter: "678" (ID)           完整 filter 对象
              │                           │
              └─────────────┬─────────────┘
                            │
                            ▼
              filters.getIn(["678", "filter_action"])
                            │
                            ▼
              根据 "warn" / "hide" / "blur" 决定行为
```

---

## 四、动作类型边界和不一致问题

### 4.1 服务端定义的三种动作

服务端 `app/models/custom_filter.rb:38` 明确定义了三种动作：

```ruby
enum :action, { warn: 0, hide: 1, blur: 2 }, suffix: :action, validate: true
```

| 动作 | 数据库值 | 服务端序列化输出 |
|------|---------|-----------------|
| `warn` | 0 | `"filter_action": "warn"` |
| `hide` | 1 | `"filter_action": "hide"` |
| `blur` | 2 | `"filter_action": "blur"` |

### 4.2 前端类型定义的不一致

**问题 1：API 类型定义不完整**

`app/javascript/mastodon/api_types/statuses.ts:68-76`：

```typescript
export interface ApiFilterJSON {
  id: string;
  title: string;
  context: FilterContext;
  expires_at: string;
  filter_action: 'warn' | 'hide';  // ❌ 缺少 'blur'！
  keywords?: unknown[];
  statuses?: unknown[];
}
```

**问题 2：实际代码使用了 'blur'**

`app/javascript/mastodon/selectors/index.js:63,68`：

```javascript
// 检查 blur 动作 - 模糊媒体
let mediaFilters = filterResults.filter(result => 
    filters.getIn([result.get('filter'), 'filter_action']) === 'blur');

// 从结果中排除 blur
filterResults = filterResults.filter(result => 
    filters.has(result.get('filter')) && 
    filters.getIn([result.get('filter'), 'filter_action']) !== 'blur');
```

**问题 3：Reducer 实际上正确保存了 filter_action**

`app/javascript/mastodon/reducers/filters.js:6-14`：

```javascript
const normalizeFilter = (state, filter) => {
  const normalizedFilter = fromJS({
    id: filter.id,
    title: filter.title,
    context: filter.context,
    filter_action: filter.filter_action,  // ✅ 正确保存，包括 'blur'
    keywords: filter.keywords,
    expires_at: filter.expires_at ? Date.parse(filter.expires_at) : null,
  });
  // ...
};
```

### 4.3 三种动作的边界和行为

#### 行为矩阵（前端实现）

| 动作类型 | 上下文条件 | 前端行为 | 设置的状态字段 |
|---------|-----------|---------|---------------|
| **hide** | `warnInsteadOfHide = false` | 完全隐藏帖子 | `status = null`, `loadingState = 'filtered'` |
| **hide** | `warnInsteadOfHide = true` | 显示警告横幅（同 warn） | `matched_filters = [title]` |
| **warn** | 所有上下文 | 显示警告横幅，可展开 | `matched_filters = [title]` |
| **blur** | 所有上下文 | 正文正常显示，仅媒体模糊 | `matched_media_filters = [title]` |

#### 上下文类型对 hide 行为的影响

`warnInsteadOfHide` 参数决定了 `hide` 动作是否真正隐藏：

```javascript
// app/javascript/mastodon/selectors/index.js:17
(_, { contextType }) => ['detailed', 'bookmarks', 'favourites', 'search'].includes(contextType),
```

| contextType | warnInsteadOfHide | hide 动作行为 |
|-------------|------------------|--------------|
| `home` | false | 完全隐藏 |
| `notifications` | false | 完全隐藏 |
| `public` | false | 完全隐藏 |
| `list:*` | false | 完全隐藏 |
| `detailed` (详情页) | true | 显示警告 |
| `bookmarks` | true | 显示警告 |
| `favourites` | true | 显示警告 |
| `search` | true | 显示警告 |

#### 选择器中的处理逻辑

`app/javascript/mastodon/selectors/index.js`：

```javascript
function getStatusResultFunction(statusBase, statusReblog, ..., filters, warnInsteadOfHide) {
  // ...

  if ((accountReblog || accountBase).get('id') !== me && filters) {
    let filterResults = statusReblog?.get('filtered') || statusBase.get('filtered') || ImmutableList();
    
    // 1. 处理 hide 动作
    if (!warnInsteadOfHide && filterResults.some((result) => 
        filters.getIn([result.get('filter'), 'filter_action']) === 'hide')) {
      return {
        status: null,
        loadingState: 'filtered',  // 完全隐藏
      }
    }

    // 2. 处理 blur 动作（独立处理）
    let mediaFilters = filterResults.filter(result => 
        filters.getIn([result.get('filter'), 'filter_action']) === 'blur');
    if (!mediaFilters.isEmpty()) {
      mediaFiltered = mediaFilters.map(result => 
          filters.getIn([result.get('filter'), 'title']));
    }

    // 3. 处理 warn 动作（同时处理 warnInsteadOfHide=true 时的 hide）
    filterResults = filterResults.filter(result => 
        filters.has(result.get('filter')) && 
        filters.getIn([result.get('filter'), 'filter_action']) !== 'blur');
    if (!filterResults.isEmpty()) {
      filtered = filterResults.map(result => 
          filters.getIn([result.get('filter'), 'title']));
    }
  }

  return {
    status: statusBase.withMutations(map => {
      map.set('matched_filters', filtered);        // warn / hide(在详情页)
      map.set('matched_media_filters', mediaFiltered); // blur
    }),
    // ...
  };
}
```

**关键边界逻辑**：

1. **hide vs warn 的边界**：
   - `hide` 在时间线中导致 `status = null`
   - `hide` 在详情页中被当作 `warn` 处理
   - `warn` 在所有场景都显示警告横幅

2. **blur 与其他动作的边界**：
   - `blur` 被**单独处理**，设置 `matched_media_filters`
   - 其他动作（`warn`/`hide`）设置 `matched_filters`
   - 一个帖子可以**同时**被多个过滤器匹配，包括不同的 `filter_action`
   - 如果一个帖子同时匹配 `hide` 和 `blur` 过滤器：
     - 在时间线中：`hide` 优先，帖子完全隐藏
     - 在详情页中：两者都生效，显示警告横幅 + 媒体模糊

3. **多过滤器匹配的优先级**：
   ```javascript
   // hide 有最高优先级（在时间线中）
   if (!warnInsteadOfHide && filterResults.some(/* hide */)) {
     return { status: null, ... }  // 直接返回，不处理其他
   }
   
   // 然后处理 blur
   // 然后处理 warn/hide(详情页)
   ```

### 4.4 媒体模糊的实际应用

媒体模糊在 `Status` 组件中的使用 (`app/javascript/mastodon/components/status.jsx`)：

```javascript
// 默认媒体可见性计算
export const defaultMediaVisibility = (status) => {
  // ...
  return !status.get('matched_media_filters') && 
         (displayMedia !== 'hide_all' && !status.get('sensitive') || displayMedia === 'show_all');
};

// 渲染时传递给媒体组件
<MediaGallery
  // ...
  matchedFilters={status.get('matched_media_filters')}
/>
```

**关键点**：
- `matched_media_filters` 存在时，`defaultMediaVisibility` 返回 `false`
- 媒体组件会显示模糊效果，用户点击后显示
- 这与 `sensitive` 标记的行为类似，但触发原因不同

### 4.5 不一致问题的影响和修复建议

#### 问题总结

| 层级 | 内容 | 状态 |
|------|------|------|
| 服务端枚举 | `'warn' \| 'hide' \| 'blur'` | ✅ 完整 |
| 服务端序列化 | `filter_action` 输出三种值 | ✅ 完整 |
| 前端 Reducer | 保存三种 `filter_action` | ✅ 完整 |
| 前端选择器 | 处理三种 `filter_action` | ✅ 完整 |
| 前端 TypeScript 类型 | `'warn' \| 'hide'` | ❌ 缺少 `'blur'` |

#### 影响

| 问题 | 影响 | 严重程度 |
|------|------|---------|
| TypeScript 类型缺少 `'blur'` | 类型检查时可能报错，但运行时正常 | 低（代码实际正常工作） |
| 类型与实现不同步 | 维护者可能困惑，新增功能时可能遗漏 | 中 |

#### 建议修复

更新 `app/javascript/mastodon/api_types/statuses.ts:73`：

```typescript
// 修复前
filter_action: 'warn' | 'hide';

// 修复后
filter_action: 'warn' | 'hide' | 'blur';
```

---

## 五、帖子过滤器系统（前端）

### 5.1 数据模型

过滤器的数据结构在 `app/javascript/mastodon/api_types/statuses.ts` 中定义：

```typescript
// 注意：服务端支持 'warn' | 'hide' | 'blur' 三种
// 但当前类型定义只包含 'warn' | 'hide'（存在不一致）
filter_action: 'warn' | 'hide';
```

### 5.2 Redux 状态管理

#### Actions (`app/javascript/mastodon/actions/filters.js`)

主要动作类型：
- `FILTERS_FETCH_SUCCESS`：从 API 获取过滤器列表
- `FILTERS_CREATE_SUCCESS`：创建新过滤器
- `FILTERS_IMPORT`：从帖子数据中导入过滤器（增量）

关键 API 调用：
```javascript
// 获取过滤器列表
api().get('/api/v2/filters')

// 创建过滤器
api().post('/api/v2/filters', params)

// 为特定过滤器添加状态
api().post(`/api/v2/filters/${params.filter_id}/statuses`, params)
```

#### Reducer (`app/javascript/mastodon/reducers/filters.js`)

状态规范化函数 `normalizeFilter` 处理过滤器数据：

```javascript
const normalizeFilter = (state, filter) => {
  const normalizedFilter = fromJS({
    id: filter.id,
    title: filter.title,
    context: filter.context,
    filter_action: filter.filter_action,
    keywords: filter.keywords,
    expires_at: filter.expires_at ? Date.parse(filter.expires_at) : null,
  });
  // ... 合并逻辑
};
```

### 5.3 选择器逻辑

#### 过滤器选择器 (`app/javascript/mastodon/selectors/filters.ts`)

**`getFilters`** - 根据上下文类型获取有效过滤器：

```typescript
export const getFilters = createSelector(
  [
    (state: RootState) => state.filters as Immutable.Map<string, Filter>,
    (_, { contextType }: { contextType: string }) => contextType,
  ],
  (filters, contextType) => {
    const now = new Date();
    const serverSideType = toServerSideType(contextType);

    return filters.filter((filter) => {
      const context = filter.get('context') as Immutable.List<string>;
      const expiration = filter.get('expires_at') as Date | null;
      return (
        context.includes(serverSideType) &&
        (expiration === null || expiration > now)
      );
    });
  },
);
```

**`getStatusHidden`** - 检查帖子是否应该被隐藏：

```typescript
export const getStatusHidden = (
  state: RootState,
  { id, contextType }: { id: string; contextType: string },
) => {
  const filters = getFilters(state, { contextType });
  if (filters === null) return false;

  const filtered = state.statuses.getIn([id, 'filtered']) as
    | Immutable.List<FilterResult>
    | undefined;
  return filtered?.some(
    (result) =>
      filters.getIn([result.get('filter'), 'filter_action']) === 'hide',
  );
};
```

#### 主选择器 (`app/javascript/mastodon/selectors/index.js`)

**`makeGetStatus`** - 核心帖子过滤逻辑：

```javascript
function getStatusResultFunction(
  statusBase,
  statusReblog,
  accountBase,
  accountReblog,
  filters,
  warnInsteadOfHide  // 关键参数：在详情页等场景不隐藏
) {
  // ... 基础检查

  let filtered = false;
  let mediaFiltered = false;
  
  // 只过滤非自己的帖子
  if ((accountReblog || accountBase).get('id') !== me && filters) {
    let filterResults = statusReblog?.get('filtered') || statusBase.get('filtered') || ImmutableList();
    
    // 1. 检查 hide 动作 - 完全隐藏
    if (!warnInsteadOfHide && filterResults.some((result) => 
        filters.getIn([result.get('filter'), 'filter_action']) === 'hide')) {
      return {
        status: null,
        loadingState: 'filtered',  // 标记为已过滤
      }
    }

    // 2. 检查 blur 动作 - 模糊媒体
    let mediaFilters = filterResults.filter(result => 
        filters.getIn([result.get('filter'), 'filter_action']) === 'blur');
    if (!mediaFilters.isEmpty()) {
      mediaFiltered = mediaFilters.map(result => 
          filters.getIn([result.get('filter'), 'title']));
    }

    // 3. 检查 warn 动作 - 显示警告
    filterResults = filterResults.filter(result => 
        filters.has(result.get('filter')) && 
        filters.getIn([result.get('filter'), 'filter_action']) !== 'blur');
    if (!filterResults.isEmpty()) {
      filtered = filterResults.map(result => 
          filters.getIn([result.get('filter'), 'title']));
    }
  }

  return {
    status: statusBase.withMutations(map => {
      map.set('reblog', statusReblog);
      map.set('account', accountBase);
      map.set('matched_filters', filtered);        // 匹配的过滤器标题
      map.set('matched_media_filters', mediaFiltered); // 匹配的媒体过滤器
    }),
    loadingState: statusBase.get('isLoading') ? 'loading' : 'complete'
  };
}
```

**上下文类型映射** (`app/javascript/mastodon/utils/filters.ts`)：

```typescript
export const toServerSideType = (columnType: string) => {
  switch (columnType) {
    case 'home':
    case 'notifications':
    case 'public':
    case 'thread':
    case 'account':
      return columnType;
    case 'detailed':
      return 'thread';
    case 'bookmarks':
    case 'favourites':
      return 'home';
    default:
      if (columnType.includes('list:')) {
        return 'home';
      } else {
        return 'public'; // community, account, hashtag
      }
  }
};
```

---

## 六、内容警告系统

### 6.1 组件结构

#### ContentWarning 组件 (`app/javascript/mastodon/components/content_warning.tsx`)

```tsx
export const ContentWarning: React.FC<{
  status: Status;
  expanded?: boolean;
  onClick?: () => void;
}> = ({ status, expanded, onClick }) => {
  // 检查是否有 spoiler_text
  const hasSpoiler = !!status.get('spoiler_text');
  if (!hasSpoiler) {
    return null;
  }

  // 获取警告文本（支持翻译）
  const text =
    status.getIn(['translation', 'spoilerHtml']) || status.get('spoilerHtml');
  if (typeof text !== 'string' || text.length === 0) {
    return null;
  }

  return (
    <StatusBanner
      expanded={expanded}
      onClick={onClick}
      variant={BannerVariant.Warning}  // 警告样式
    >
      <EmojiHTML
        as='span'
        htmlString={text}
        extraEmojis={status.get('emojis') as List<CustomEmoji>}
      />
    </StatusBanner>
  );
};
```

#### StatusBanner 组件 (`app/javascript/mastodon/components/status_banner.tsx`)

支持两种变体：
- `BannerVariant.Warning`：内容警告样式
- `BannerVariant.Filter`：过滤器警告样式

```tsx
export const StatusBanner: React.FC<{
  children: React.ReactNode;
  variant: BannerVariant;
  expanded?: boolean;
  onClick?: () => void;
}> = ({ children, variant, expanded, onClick }) => {
  // ... 点击转发逻辑

  return (
    <AnimateEmojiProvider
      className={
        variant === BannerVariant.Warning
          ? 'content-warning'
          : 'content-warning content-warning--filter'  // 过滤器有额外样式
      }
      onClick={forwardClick}
      onMouseUp={stopPropagation}
    >
      <p id={descriptionId}>{children}</p>

      <button
        ref={buttonRef}
        type='button'
        className='link-button'
        onClick={onClick}
        aria-describedby={descriptionId}
      >
        {expanded ? (
          <FormattedMessage
            id='content_warning.hide'
            defaultMessage='Hide post'
          />
        ) : variant === BannerVariant.Warning ? (
          <FormattedMessage
            id='content_warning.show_more'
            defaultMessage='Show more'
          />
        ) : (
          <FormattedMessage
            id='content_warning.show'
            defaultMessage='Show anyway'
          />
        )}
      </button>
    </AnimateEmojiProvider>
  );
};
```

---

## 七、前端隐藏和折叠触发逻辑

### 7.1 帖子组件核心逻辑 (`app/javascript/mastodon/components/status.jsx`)

#### 状态管理

```javascript
state = {
  showMedia: defaultMediaVisibility(this.props.status) && !(this.context?.hideMediaByDefault),
  showDespiteFilter: undefined,  // 关键：是否忽略过滤器显示
};
```

#### 展开状态计算

```javascript
// 在 render 方法中
const expanded = (!matchedFilters || this.state.showDespiteFilter) && 
                 (!status.get('hidden') || status.get('spoiler_text').length === 0);
```

**展开条件解析**：
1. **过滤器条件**：`!matchedFilters || this.state.showDespiteFilter`
   - 没有匹配的过滤器，或者用户主动选择显示（`showDespiteFilter = true`）
   
2. **内容警告条件**：`!status.get('hidden') || status.get('spoiler_text').length === 0`
   - `status.hidden = false`（已展开），或者没有 spoiler_text

#### 渲染逻辑

```javascript
// 1. 显示过滤器警告
{matchedFilters && <FilterWarning 
    title={matchedFilters.join(', ')} 
    expanded={this.state.showDespiteFilter} 
    onClick={this.handleFilterToggle} 
/>}

// 2. 显示内容警告（如果没有过滤器或已选择显示）
{(!matchedFilters || this.state.showDespiteFilter) && 
    <ContentWarning 
        status={status} 
        expanded={expanded} 
        onClick={this.handleExpandedToggle} 
    />
}

// 3. 只有 expanded 为 true 时才显示内容
{expanded && (
  <>
    <StatusContent ... />
    {media}
    {hashtagBar}
    {children}
  </>
)}
```

#### 交互处理

**过滤器切换**：
```javascript
handleFilterToggle = () => {
  this.setState(state => ({ ...state, showDespiteFilter: !state.showDespiteFilter }));
};
```

**内容警告切换**：
```javascript
handleExpandedToggle = () => {
  this.props.onToggleHidden(this._properStatus());
};
```

**热键处理**（智能切换逻辑）：
```javascript
handleHotkeyToggleHidden = () => {
  const { onToggleHidden } = this.props;
  const status = this._properStatus();

  if (this.props.status.get('matched_filters')) {
    // 有匹配的过滤器时，需要考虑两种状态
    const expandedBecauseOfCW = !status.get('hidden') || status.get('spoiler_text').length === 0;
    const expandedBecauseOfFilter = this.state.showDespiteFilter;

    if (expandedBecauseOfFilter && !expandedBecauseOfCW) {
      // 过滤器已展开，但内容警告未展开 → 切换内容警告
      onToggleHidden(status);
    } else if (expandedBecauseOfFilter && expandedBecauseOfCW) {
      // 两者都已展开 → 先切换内容警告，再切换过滤器
      onToggleHidden(status);
      this.handleFilterToggle();
    } else {
      // 过滤器未展开 → 先切换过滤器
      this.handleFilterToggle();
    }
  } else {
    // 没有过滤器，直接切换内容警告
    onToggleHidden(status);
  }
};
```

### 7.2 状态容器 (`app/javascript/mastodon/containers/status_container.jsx`)

#### Dispatch 映射

```javascript
const mapDispatchToProps = (dispatch, { contextType }) => ({
  // 切换内容警告展开状态
  onToggleHidden (status) {
    dispatch(toggleStatusSpoilers(status.get('id')));
  },

  // 切换长内容折叠状态
  onToggleCollapsed (status, isCollapsed) {
    dispatch(toggleStatusCollapse(status.get('id'), isCollapsed));
  },
  
  // ... 其他方法
});
```

#### 上下文类型影响

`contextType` 参数决定了：
1. **过滤器选择**：通过 `toServerSideType` 映射确定哪些过滤器适用
2. **隐藏行为**：`makeGetStatus` 中的 `warnInsteadOfHide` 参数

```javascript
// 在 getStatusInputSelectors 中
(_, { contextType }) => ['detailed', 'bookmarks', 'favourites', 'search'].includes(contextType),
```

**关键**：在 `detailed`、`bookmarks`、`favourites`、`search` 这些上下文类型中，`warnInsteadOfHide = true`，意味着：
- `hide` 动作不会完全隐藏帖子
- 而是像 `warn` 一样显示警告横幅

### 7.3 通知中的过滤逻辑 (`app/javascript/mastodon/features/notifications_v2/components/notification_with_status.tsx`)

```tsx
const isFiltered = useAppSelector(
  (state) =>
    statusId &&
    getStatusHidden(state, { id: statusId, contextType: 'notifications' }),
);

// 如果被过滤，直接返回 null
if (!statusId || isFiltered) return null;
```

### 7.4 长内容折叠逻辑 (`app/javascript/mastodon/components/status_content.jsx`)

**自动折叠条件**：
```javascript
_updateStatusLinks () {
  const node = this.node;
  if (!node) return;

  const { status, onCollapsedToggle } = this.props;
  
  // 只有当 collapsed 状态未确定时才计算
  if (status.get('collapsed', null) === null && onCollapsedToggle) {
    const { collapsible, onClick } = this.props;
    const text = node.querySelector(':scope > .status__content__text');

    const collapsed =
        collapsible  // 允许折叠
        && onClick   // 有点击处理（可交互）
        && (node.clientHeight > MAX_HEIGHT || (text !== null && text.scrollWidth > text.clientWidth))
        // 高度超过 706px 或宽度溢出
        && status.get('spoiler_text').length === 0;  // 没有内容警告

    onCollapsedToggle(collapsed);
  }
}
```

**折叠常量**：
```javascript
const MAX_HEIGHT = 706; // 22px * 32 (+ 2px padding at the top)
```

---

## 八、关键文件索引

### 服务端文件

| 功能 | 文件路径 |
|------|----------|
| 过滤器模型 | `app/models/custom_filter.rb` |
| 过滤结果呈现 | `app/presenters/filter_result_presenter.rb` |
| 状态关系处理 | `app/presenters/status_relationships_presenter.rb` |
| 账户交互方法 | `app/models/concerns/account/interactions.rb` |
| 帖子序列化器 | `app/serializers/rest/status_serializer.rb` |
| 过滤结果序列化器 | `app/serializers/rest/filter_result_serializer.rb` |
| 过滤器序列化器 | `app/serializers/rest/filter_serializer.rb` |

### 前端文件

| 功能 | 文件路径 |
|------|----------|
| 过滤器 Reducer | `app/javascript/mastodon/reducers/filters.js` |
| 过滤器 Actions | `app/javascript/mastodon/actions/filters.js` |
| 过滤器选择器 | `app/javascript/mastodon/selectors/filters.ts` |
| 主选择器 | `app/javascript/mastodon/selectors/index.js` |
| 上下文映射 | `app/javascript/mastodon/utils/filters.ts` |
| 数据规范化 | `app/javascript/mastodon/actions/importer/normalizer.js` |
| 数据导入 | `app/javascript/mastodon/actions/importer/index.js` |
| 内容警告组件 | `app/javascript/mastodon/components/content_warning.tsx` |
| 状态横幅组件 | `app/javascript/mastodon/components/status_banner.tsx` |
| 帖子主组件 | `app/javascript/mastodon/components/status.jsx` |
| 帖子内容组件 | `app/javascript/mastodon/components/status_content.jsx` |
| 帖子容器 | `app/javascript/mastodon/containers/status_container.jsx` |
| API 类型定义 | `app/javascript/mastodon/api_types/statuses.ts` |

---

## 九、总结

### 9.1 服务端到前端的完整流程

```
用户创建过滤器
    │
    ▼
CustomFilter (数据库)
  - action: :warn/:hide/:blur (枚举值 0/1/2)
  - context: ['home', 'notifications', ...]
    │
    ▼
缓存优化 (Rails.cache)
  - 关键词编译为 Regexp
  - 状态 ID 整理为数组
    │
    ▼
API 请求处理
  - StatusRelationshipsPresenter 批量处理
  - CustomFilter.apply_cached_filters 匹配
    │
    ▼
服务端序列化输出
  - filtered 数组
  - 每个元素包含完整的 filter 对象
  - filter.filter_action: "warn" | "hide" | "blur" (字符串)
    │
    ▼
前端 importFetchedStatuses
  ├─► 阶段 A: 提取完整 filter 对象 → importFilters → state.filters[id].filter_action
  └─► 阶段 B: 规范化帖子 → filter 替换为 ID → state.statuses[id].filtered[0].filter = "id"
    │
    ▼
选择器应用
  - 从 status.filtered 获取 filter ID
  - 从 state.filters 获取完整 filter_action
  - 根据 "warn" / "hide" / "blur" 决定行为
    │
    ▼
组件渲染
  - FilterWarning / ContentWarning
  - 内容展开/折叠
  - 媒体模糊
```

### 9.2 三种动作类型的行为边界

| 动作 | 时间线/通知 | 详情页/收藏/搜索 | 媒体显示 | 正文显示 |
|------|------------|-----------------|---------|---------|
| **hide** | 完全隐藏 (`status=null`) | 显示警告横幅 | - | - |
| **warn** | 显示警告 | 显示警告 | 正常 | 隐藏（可展开） |
| **blur** | 正常显示 | 正常显示 | 模糊 | 正常 |

### 9.3 类型不一致问题

**当前状态**：

| 层级 | 支持的 filter_action | 状态 |
|------|---------------------|------|
| 服务端枚举 | `warn` / `hide` / `blur` | ✅ 完整 |
| 服务端序列化 | `warn` / `hide` / `blur` | ✅ 完整 |
| 前端 Reducer | `warn` / `hide` / `blur` | ✅ 完整 |
| 前端选择器逻辑 | `warn` / `hide` / `blur` | ✅ 完整 |
| 前端 TypeScript 类型 | `warn` / `hide` | ❌ 缺少 `blur` |

**影响**：
- TypeScript 类型检查可能报错
- 代码维护时可能产生困惑

**建议修复**：
更新 `app/javascript/mastodon/api_types/statuses.ts`：
```typescript
// 从
filter_action: 'warn' | 'hide';
// 改为
filter_action: 'warn' | 'hide' | 'blur';
```

### 9.4 两种系统的区别

| 特性 | 帖子过滤器 | 内容警告 |
|------|-----------|---------|
| 触发源 | 用户定义的规则/关键词 | 帖子作者设置的 `spoiler_text` |
| 数据来源 | `status.filtered` 数组 + 本地过滤器状态 | `status.spoiler_text` |
| 动作类型 | `hide` / `warn` / `blur` | 单一折叠/展开 |
| 上下文感知 | 是（不同列应用不同过滤器） | 否 |
| 可被覆盖 | 是（`showDespiteFilter`） | 是（`toggleStatusSpoilers`） |

### 9.5 隐藏 vs 折叠的决策逻辑

**完全隐藏**（`filter_action = 'hide'` + `warnInsteadOfHide = false`）：
- 时间线中匹配 hide 过滤器的帖子
- 通知中匹配 hide 过滤器的帖子
- 结果：`status = null`，不渲染任何内容

**折叠显示**：
- 有 `spoiler_text` 的帖子 → 显示内容警告
- 匹配 `warn` 过滤器的帖子 → 显示过滤器警告
- 匹配 `hide` 过滤器但在详情页/收藏/搜索 → 显示警告
- 结果：显示警告横幅，用户可点击展开

**媒体模糊**：
- 匹配 `blur` 过滤器的帖子
- 结果：正文正常显示，媒体模糊显示

**长内容自动折叠**：
- 高度超过 706px 且没有 `spoiler_text`
- 结果：显示 "Read more" 按钮
