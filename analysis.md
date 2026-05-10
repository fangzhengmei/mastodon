# Mastodon Hashtag 趋势链路分析

## 目录
1. [Hashtag 提取机制](#1-hashtag-提取机制)
2. [Hashtag 计数与存储](#2-hashtag-计数与存储)
3. [趋势计算算法](#3-趋势计算算法)
4. [从计算到探索页的展示链路](#4-从计算到探索页的展示链路)
5. [实例审核策略对公开发现能力的影响](#5-实例审核策略对公开发现能力的影响)

---

## 1. Hashtag 提取机制

### 1.1 正则表达式匹配

Hashtag 的提取使用精心设计的正则表达式，定义在 `app/models/tag.rb:37-46`：

```ruby
HASHTAG_SEPARATORS = "_\u00B7\u30FB\u200c"
HASHTAG_FIRST_SEQUENCE_CHUNK_ONE = "[[:word:]_][[:word:]#{HASHTAG_SEPARATORS}]*[[:alpha:]#{HASHTAG_SEPARATORS}]"
HASHTAG_FIRST_SEQUENCE_CHUNK_TWO = "[[:word:]#{HASHTAG_SEPARATORS}]*[[:word:]_]"
HASHTAG_FIRST_SEQUENCE = "(#{HASHTAG_FIRST_SEQUENCE_CHUNK_ONE}#{HASHTAG_FIRST_SEQUENCE_CHUNK_TWO})"
HASHTAG_LAST_SEQUENCE = '([[:word:]_]*[[:alpha:]][[:word:]_]*)'
HASHTAG_NAME_PAT = "#{HASHTAG_FIRST_SEQUENCE}|#{HASHTAG_LAST_SEQUENCE}"
HASHTAG_RE = /(?<=^|[[:space:]])[#＃](#{HASHTAG_NAME_PAT})/
```

**设计要点：**
- 支持全角 `＃` 和半角 `#` 符号
- 支持多语言字符（通过 `[[:alpha:]]` 和 `[[:word:]]` POSIX 字符类）
- 允许下划线 `_`、中间点 `·`、日语间隔号 `・` 和零宽非连接符 `\u200c`
- 要求至少包含一个字母字符，防止纯数字 hashtag

### 1.2 提取流程

提取逻辑在 `app/lib/extractor.rb:56-87` 的 `extract_hashtags_with_indices` 方法中：

```ruby
def extract_hashtags_with_indices(text, _options = {})
  return [] unless text&.index(/[#＃]/)
  
  text.scan(Tag::HASHTAG_RE) do |hash_text, _|
    # 处理 URL 重叠问题
    if after.start_with?('://')
      hash_text.match(/(.+)(https?\Z)/) do |matched|
        hash_text     = matched[1]
        end_position -= matched[2].codepoint_length
      end
    end
  end
end
```

**关键处理：**
- 快速检查：先检查文本中是否存在 `#` 或 `＃`，无则直接返回
- URL 重叠处理：如 `#examplehttp://example.com`，会正确提取 `#example`

### 1.3 发帖时的提取

在 `app/services/post_status_service.rb:162-164` 的 `postprocess_status!` 方法中：

```ruby
def postprocess_status!
  process_hashtags_service.call(@status)
  Trends.tags.register(@status)
  # ...
end
```

`ProcessHashtagsService` (`app/services/process_hashtags_service.rb:4-13`) 处理本地和远程帖子的差异：

```ruby
def call(status, raw_tags = [])
  @raw_tags = status.local? ? Extractor.extract_hashtags(status.text) : raw_tags
  assign_tags!  # Tag.find_or_create_by_names(@raw_tags)
  update_featured_tags!
end
```

---

## 2. Hashtag 计数与存储

### 2.1 实时注册

当帖子发布后，`Trends::Tags.register` 方法 (`app/models/trends/tags.rb:33-39`) 被调用：

```ruby
def register(status, at_time = Time.now.utc)
  return unless !status.reblog? && status.public_visibility? && !status.account.silenced?

  status.tags.each do |tag|
    add(tag, status.account_id, at_time) if tag.usable?
  end
end

def add(tag, account_id, at_time = Time.now.utc)
  tag.history.add(account_id, at_time)
  record_used_id(tag.id, at_time)
end
```

**过滤条件：**
- 不是转发 (`!status.reblog?`)
- 公开可见 (`status.public_visibility?`)
- 发帖账号未被静音 (`!status.account.silenced?`)
- Tag 可用 (`tag.usable?`)

### 2.2 计数存储机制

使用 Redis 进行高效计数，核心类 `Trends::History::Day` (`app/models/trends/history.rb:22-70`)：

```ruby
class Day
  EXPIRE_AFTER = 14.days.seconds

  def add(value)
    with_redis do |redis|
      redis.pipelined do |pipeline|
        pipeline.incrby(key_for(:uses), 1)
        pipeline.pfadd(key_for(:accounts), value)
        pipeline.expire(key_for(:uses), EXPIRE_AFTER)
        pipeline.expire(key_for(:accounts), EXPIRE_AFTER)
      end
    end
  end

  def key_prefix
    "activity:#{@prefix}:#{@id}:#{day.to_i}"
  end
end
```

**技术特点：**

| 计数器 | 数据结构 | 用途 | 键格式 |
|--------|----------|------|--------|
| uses | String (INCR) | 总使用次数 | `activity:tags:{tag_id}:{day_ts}` |
| accounts | HyperLogLog | 去重账户数 | `activity:tags:{tag_id}:{day_ts}:accounts` |

**HyperLogLog 优势：**
- 常数级内存占用（约 12KB 每个 key）
- 近似计数，标准误差约 0.81%
- 非常适合统计独立用户数这种大规模数据

**保留期限：** 14 天

### 2.3 每日使用记录

在 `Trends::Base` (`app/models/trends/base.rb:50-53`) 中：

```ruby
def record_used_id(id, at_time = Time.now.utc)
  redis.sadd(used_key(at_time), id)
  redis.expire(used_key(at_time), 1.day.seconds)
end
```

使用 Redis Set 记录当天使用过的 tag ID，便于后续批量计算趋势。

---

## 3. 趋势计算算法

### 3.1 定时刷新

在 `config/sidekiq.yml:19-26` 中配置：

```yaml
trends_refresh_scheduler:
  every: '5m'
  class: Scheduler::Trends::RefreshScheduler
  queue: scheduler
```

每 5 分钟执行一次，调用 `Trends.refresh!`。

### 3.2 核心算法

`Trends::Tags#calculate_scores` 方法 (`app/models/trends/tags.rb:94-127`) 是趋势计算的核心：

```ruby
def calculate_scores(tags, at_time)
  items = tags.map do |tag|
    expected  = tag.history.get(at_time - 1.day).accounts.to_f
    expected  = 1.0 if expected.zero?
    observed  = tag.history.get(at_time).accounts.to_f
    max_time  = tag.max_score_at
    max_score = tag.max_score
    max_score = 0 if max_time.nil? || max_time < (at_time - options[:max_score_cooldown])

    score = if expected > observed || observed < options[:threshold]
              0
            else
              ((observed - expected)**2) / expected
            end

    if score > max_score
      max_score = score
      max_time  = at_time
      tag.update_columns(max_score: max_score, max_score_at: max_time)
    end

    decaying_score = max_score * (0.5**((at_time.to_f - max_time.to_f) / options[:max_score_halflife].to_f))
    [decaying_score, tag]
  end
  # ...
end
```

### 3.3 算法详细解析

#### 3.3.1 默认参数 (`app/models/trends/tags.rb:8-14`)

```ruby
self.default_options = {
  threshold: 5,           # 最小独立账户数阈值
  review_threshold: 3,    # 需要审核的排名阈值
  max_score_cooldown: 2.days.freeze,   # 峰值冷却期
  max_score_halflife: 4.hours.freeze,  # 分数半衰期
  decay_threshold: 1,     # 最低保留分数
}
```

#### 3.3.2 评分公式

**卡方统计量（Chi-squared statistic）：**

```
score = (observed - expected)² / expected
```

其中：
- `observed` = 当天使用该 hashtag 的独立账户数
- `expected` = 前一天使用该 hashtag 的独立账户数（如为 0 则设为 1）

**零分条件：**
- `expected > observed`：使用量下降
- `observed < 5`：独立账户数不足 5

#### 3.3.3 峰值衰减机制

使用指数衰减公式：

```
decaying_score = max_score * (0.5 ^ (elapsed_time / halflife))
```

其中：
- `elapsed_time` = 当前时间 - 峰值时间
- `halflife` = 4 小时

**示例：**
- 初始峰值分数 = 100
- 4 小时后 → 50
- 8 小时后 → 25
- 24 小时后 → 约 1.56
- 32 小时后 → 约 0.78（低于 decay_threshold=1，被移除）

#### 3.3.4 冷却期机制

如果 `max_score_at` 超过 2 天，则 `max_score` 重置为 0。这样可以防止一次性热门话题永远占据趋势榜。

### 3.4 刷新流程

`Trends::Tags#refresh` 方法 (`app/models/trends/tags.rb:50-66`)：

```ruby
def refresh(at_time = Time.now.utc)
  # 1. 重新计算之前已经在趋势榜上的 tags
  Tag.where(id: TagTrend.select(:tag_id)).find_in_batches(batch_size: BATCH_SIZE) do |tags|
    calculate_scores(tags, at_time)
  end

  # 2. 计算当天使用过的 tags（可能有重复，但不影响）
  Tag.where(id: recently_used_ids(at_time)).find_in_batches(batch_size: BATCH_SIZE) do |tags|
    calculate_scores(tags, at_time)
  end

  # 3. 重新计算排名
  TagTrend.recalculate_ordered_rank
end
```

---

## 4. 从计算到探索页的展示链路

### 4.1 数据层：TagTrend 模型

`app/models/tag_trend.rb`：

```ruby
class TagTrend < ApplicationRecord
  include RankedTrend
  belongs_to :tag

  scope :allowed, -> { where(allowed: true) }
  scope :not_allowed, -> { where(allowed: false) }
end
```

**字段：**
| 字段 | 类型 | 说明 |
|------|------|------|
| tag_id | bigint | 关联 tag |
| score | float | 衰减后的分数 |
| rank | integer | 排名 |
| language | string | 语言（目前为空字符串） |
| allowed | boolean | 是否通过审核允许展示 |

### 4.2 API 层

`app/controllers/api/v1/trends/tags_controller.rb`：

```ruby
def set_tags
  @tags = if enabled?
            tags_from_trends.offset(offset_param).limit(limit_param(DEFAULT_TAGS_LIMIT))
          else
            []
          end
end

def tags_from_trends
  scope = Trends.tags.query.allowed.in_locale(content_locale)
  scope = scope.filtered_for(current_account) if user_signed_in?
  scope
end
```

**关键过滤：**
- `enabled?`：检查 `Setting.trends` 是否开启
- `.allowed`：只返回 `allowed = true` 的趋势
- `.in_locale`：按内容语言优先排序

### 4.3 查询构建器

`Trends::Tags::Query` (`app/models/trends/tags.rb:16-31`)：

```ruby
class Query < Trends::Query
  def to_arel
    scope = Tag.joins(:trend).reorder(score: :desc)
    scope = scope.reorder(language_order_clause, score: :desc) if preferred_languages.present?
    scope = scope.merge(TagTrend.allowed) if @allowed
    scope
  end
end
```

`Trends::Query` (`app/models/trends/query.rb:97-113`) 的语言偏好处理：

```ruby
def language_order_clause
  Arel::Nodes::Case.new.when(language_is_preferred).then(1).else(0).desc
end

def preferred_languages
  if @account&.chosen_languages.present?
    @account.chosen_languages
  else
    @locale
  end
end
```

### 4.4 序列化层

`app/serializers/rest/tag_serializer.rb`：

```ruby
attributes :id, :name, :url, :history
attribute :following, if: :current_user?
attribute :featuring, if: :current_user?
```

返回数据包含历史数据（7 天的 account 和 uses 统计）。

### 4.5 前端层

#### 4.5.1 API 调用

`app/javascript/mastodon/actions/trends.js:21-28`：

```javascript
export const fetchTrendingHashtags = () => (dispatch) => {
  api().get('/api/v1/trends/tags')
    .then(({ data }) => dispatch(fetchTrendingHashtagsSuccess(data)))
};
```

#### 4.5.2 探索页组件

`app/javascript/mastodon/features/explore/index.tsx`：

- 路由 `/explore/tags` 对应 `Tags` 组件

`app/javascript/mastodon/features/explore/tags.jsx`：

```javascript
componentDidMount() {
  dispatch(fetchTrendingHashtags());
}

render() {
  hashtags.map(hashtag => (
    <Hashtag key={hashtag.get('name')} hashtag={hashtag} />
  ))
}
```

### 4.6 完整链路图

```
用户发帖
    ↓
PostStatusService#call
    ↓
postprocess_status!
    ├─→ ProcessHashtagsService (提取 tag 并关联)
    └─→ Trends.tags.register (记录历史数据)
            ↓
    tag.history.add(account_id)
            ↓
    Redis: INCR uses, PFADD accounts
            ↓
================ 每 5 分钟 ================
            ↓
Trends::RefreshScheduler
    ↓
Trends.tags.refresh
    ├─→ calculate_scores (卡方统计 + 指数衰减)
    └─→ TagTrend.recalculate_ordered_rank
            ↓
    写入/更新 tag_trends 表
            ↓
================ 用户访问 ================
            ↓
GET /api/v1/trends/tags
    ↓
Trends::Tags::Query
    ├─→ .allowed (过滤未审核)
    └─→ .in_locale (语言偏好)
            ↓
Tag.joins(:trend).order(score: :desc)
            ↓
REST::TagSerializer (含 7 天历史)
            ↓
前端 fetchTrendingHashtags
    ↓
Explore/Tags 组件渲染
```

---

## 5. 实例审核策略对公开发现能力的影响

### 5.1 Tag 的三级审核标志

`app/models/tag.rb` 定义了三个**独立的**标志位，各自控制不同的发现渠道，**互不影响**：

| 标志位 | 默认值 | 控制范围 | 影响 |
|--------|--------|----------|------|
| `usable` | `true` | 发帖验证 + 趋势计数 | 禁止使用该 hashtag 发帖，且不计入趋势统计 |
| `listable` | `true` | 搜索功能（数据库搜索 + ES 索引） | 不出现在搜索结果中，但**不影响趋势** |
| `trendable` | 依赖 `Setting.trendable_by_default` | 趋势展示 | 决定能否在探索页的趋势榜中显示 |

```ruby
scope :usable, -> { where(usable: [true, nil]) }
scope :listable, -> { where(listable: [true, nil]) }
scope :trendable, -> { 
  Setting.trendable_by_default ? where(trendable: [true, nil]) : where(trendable: true) 
}
```

**关键设计原则：**
- 三个标志位是**完全独立**的
- 每个标志位控制一个独立的发现渠道
- 不存在级联影响（如 `listable` 不影响 `trendable`）

### 5.2 默认设置

`config/settings.yml:24-26`：

```yaml
trends: true                    # 趋势功能总开关
trendable_by_default: false     # 新 hashtag 默认不允许进入趋势
disallowed_hashtags:            # 空格分隔的禁止 hashtag 列表
```

**`trendable_by_default: false` 的含义：**
- 新创建的 hashtag `trendable = nil`
- 查询时 `where(trendable: true)`（不包含 nil）
- 必须经过管理员审核设置 `trendable = true` 才能上趋势榜

### 5.3 Usable 标志的影响

#### 发帖时验证

`app/validators/disallowed_hashtags_validator.rb`：

```ruby
def validate(status)
  return unless status.local? && !status.reblog?
  disallowed_hashtags = Tag.matching_name(Extractor.extract_hashtags(status.text))
                          .reject(&:usable?)
  status.errors.add(:text, ...) unless disallowed_hashtags.empty?
end
```

本地用户发帖时如果包含 `usable = false` 的 hashtag，帖子会被拒绝。

#### 趋势注册时过滤

`app/models/trends/tags.rb:37`：

```ruby
add(tag, status.account_id, at_time) if tag.usable?
```

即使绕过验证（如远程帖子），不可用的 tag 也不会被计入趋势。

### 5.4 Listable 标志的影响

**重要澄清：`listable` 主要影响搜索发现能力，不影响趋势展示！但在 Elasticsearch 搜索路径下存在例外情况。**

#### 5.4.1 数据库搜索路径

`app/models/tag.rb:130-141`：

```ruby
def search_for(term, limit = 5, offset = 0, options = {})
  options.reverse_merge!({ exclude_unlistable: true, exclude_unreviewed: false })
  query = Tag.matches_name(stripped_term)
  query = query.merge(Tag.listable) if options[:exclude_unlistable]
  # ...
end
```

当 ES 不可用时，直接使用数据库搜索，`listable = false` 的 tag **不会出现在搜索结果中**。

#### 5.4.2 Elasticsearch 索引范围（主路径）

`app/chewy/tags_index.rb:37`：

```ruby
index_scope ::Tag.listable
```

Elasticsearch 索引只包含 `listable` 为 true 或 nil 的 tag。因此：

- `listable = false` 的 tag **不会被索引**到 ES 中
- ES 搜索时无法通过模糊匹配找到这些 tag

#### 5.4.3 ES 搜索的例外：Exact Match 兜底逻辑

**这是最重要的例外情况！**

`app/services/tag_search_service.rb:39-51` 的 `ensure_exact_match` 方法：

```ruby
def ensure_exact_match(results)
  return results unless @offset.nil? || @offset.zero?

  normalized_query = Tag.normalize_value_for(:name, @query)
  exact_match = results.find { |tag| tag.name.downcase == normalized_query }
  exact_match ||= Tag.find_normalized(normalized_query)  # ⚠️ 无任何过滤！
  unless exact_match.nil?
    results.delete(exact_match)
    results = [exact_match] + results
  end
  results
end
```

`Tag.find_normalized` 方法（`tag.rb:143-145`）：

```ruby
def find_normalized(name)
  matching_name(name).first  # 只做名称匹配，不检查 listable、usable、trendable
end
```

**逻辑分析：**

1. 当用户搜索时，如果 ES 结果中没有 exact match
2. 系统会调用 `Tag.find_normalized` **直接从数据库**精确查找
3. 这个查找**完全没有过滤** —— 不检查 `listable`，也不检查 `usable`
4. 如果找到了，会把它**插入到结果最前面**

**触发条件：**
- `Chewy.enabled? = true`（启用了 ES）
- 用户进行的是**精确搜索**（名字完全匹配）
- 搜索结果是**第一页**（`offset = 0` 或 `nil`）

**实际影响：**
- 一个 `listable = false` 的 tag，虽然不在 ES 索引中
- 但如果用户**精确搜索它的名字**，仍然会被找到
- 并且会**排到结果第一位**

#### 5.4.4 Listable 不影响趋势

在以下链路中，**无任何 `listable` 检查：

| 链路 | 检查的标志位 | 代码位置 |
|------|---------------|-----------|
| 发帖验证 | `usable` | `disallowed_hashtags_validator.rb:7` |
| 趋势注册 | `usable` | `trends/tags.rb:37` |
| 趋势计算 | 无 | `trends/tags.rb:50-66` |
| 趋势展示 | `trendable` (通过 `allowed`) | `trends/tags.rb:16-31` |

**关键结论**：一个 `listable = false` 但 `trendable = true` 的 tag，**完全可以正常进入趋势榜并在探索页展示**。对于搜索发现：
- **数据库搜索路径**：无法被搜索到
- **ES 搜索路径（模糊匹配）**：无法被搜索到
- **ES 搜索路径（精确匹配 + 第一页）**：**可以被搜索到**（例外情况）

### 5.5 Trendable 标志的影响

#### 趋势榜过滤

`app/models/trends/tags.rb:125`：

```ruby
TagTrend.upsert_all(
  to_insert.map { |(score, tag)| 
    { tag_id: tag.id, score: score, allowed: tag.trendable? || false } 
  }, 
  unique_by: %w(tag_id language)
)
```

`allowed` 字段直接来源于 `tag.trendable?`。

API 查询时 `app/controllers/api/v1/trends/tags_controller.rb:34`：

```ruby
scope = Trends.tags.query.allowed.in_locale(content_locale)
```

`.allowed` 对应 `TagTrend.allowed` scope：

```ruby
scope :allowed, -> { where(allowed: true) }
```

### 5.6 审核流程

#### 5.6.1 触发审核通知

`config/sidekiq.yml:23-26`：

```yaml
trends_review_notifications_scheduler:
  every: '6h'
  class: Scheduler::Trends::ReviewNotificationsScheduler
  queue: scheduler
```

每 6 小时检查一次需要审核的内容。

#### 5.6.2 审核请求逻辑

`app/models/trends/tags.rb:68-80`：

```ruby
def request_review
  score_at_threshold = TagTrend.allowed.by_rank.ranked_below(options[:review_threshold]).first&.score || 0
  tag_trends = TagTrend.not_allowed.includes(:tag)

  tag_trends.filter_map do |trend|
    tag = trend.tag
    if trend.score > score_at_threshold && !tag.trendable? && tag.requires_review_notification?
      tag.touch(:requested_review_at)
      tag
    end
  end
end
```

**触发条件：**
1. 分数超过当前排名第 3 位的分数（`review_threshold: 3`）
2. `tag.trendable?` 为 false
3. 尚未请求过审核（`requires_review_notification?`）

`app/models/trends.rb:28-40`：

```ruby
def self.request_review!
  return if skip_review? || !enabled?
  # ...
  User.those_who_can(:manage_taxonomies).includes(:account).find_each do |user|
    AdminMailer.with(recipient: user.account)
      .new_trends(links_requiring_review, tags_requiring_review, statuses_requiring_review)
      .deliver_later! if user.allows_trends_review_emails?
  end
end
```

#### 5.6.3 跳过审核模式

如果 `Setting.trendable_by_default = true`，则 `Trends.skip_review?` 返回 `true`，不会发送审核邮件。

### 5.7 管理后台审核

在 `config/routes/api.rb:303-319` 中定义了管理员审核路由：

```ruby
namespace :admin do
  namespace :trends do
    concern :approvable do
      member do
        post :approve
        post :reject
      end
    end
    with_options only: [:index], concerns: :approvable do
      resources :tags
      resources :links
      resources :statuses
    end
  end
end
```

`app/models/trends/tag_filter.rb:49-59` 支持按审核状态过滤：

```ruby
def status_scope(value)
  case value.to_s
  when 'approved'
    Tag.trendable
  when 'rejected'
    Tag.not_trendable
  when 'pending_review'
    Tag.pending_review
  end
end
```

### 5.8 审核策略对发现能力的综合影响

#### 5.8.1 完整场景矩阵（按代码事实）

| 场景 | 标志位组合 | 发帖使用 | 搜索发现 | 趋势榜单 |
|------|------------|-----------|-----------|-----------|
| **场景 1** | `usable=true, listable=true, trendable=true` | ✅ 允许 | ✅ 可见 | ✅ 正常展示 |
| **场景 2** | `usable=true, listable=true, trendable=false` | ✅ 允许 | ✅ 可见 | ❌ 需审核（`allowed=false`） |
| **场景 3** | `usable=true, listable=false, trendable=true` | ✅ 允许 | ❌ 搜索隐藏 | ✅ **正常上趋势** ⭐ |
| **场景 4** | `usable=true, listable=false, trendable=false` | ✅ 允许 | ❌ 搜索隐藏 | ❌ 需审核 |
| **场景 5a** | `usable=false, listable=true, trendable=任意` | ❌ 禁止 | ✅ 仍可见 | ❌ 不计入趋势 |
| **场景 5b** | `usable=false, listable=false, trendable=任意` | ❌ 禁止 | ❌ 搜索隐藏 | ❌ 不计入趋势 |

#### 5.8.2 关键发现（与代码一致）

**场景 3 是最重要的发现：**
- 一个 tag 被设置为 `listable = false`，但 `trendable = true`
- **用户无法通过搜索找到它**（ES 不索引、数据库搜索过滤）
- **但它可以正常出现在趋势榜上**（趋势链路无 listable 检查）

**场景 5a/5b 的关键发现：**
- `usable = false` 只影响"能否发帖使用"和"能否计入趋势"
- **搜索可见性完全由 `listable` 决定，与 `usable` 无关**
- 即使一个 tag 被禁用（`usable=false`），只要 `listable=true`，用户仍然可以搜索到它

**代码依据：**

| 链路 | 检查的标志位 | 代码位置 | 结论 |
|------|---------------|-----------|------|
| 发帖验证 | `usable` | `disallowed_hashtags_validator.rb:7` | `reject(&:usable?)` |
| 趋势注册 | `usable` | `trends/tags.rb:37` | `if tag.usable?` |
| 趋势计算 | 无任何检查 | `trends/tags.rb:50-66` | 直接从 `recently_used_ids` 获取 |
| 趋势展示 | `trendable` | `trends/tags.rb:125` | `allowed: tag.trendable?` |
| 搜索（DB） | `listable` | `tag.rb:135` | `query.merge(Tag.listable)` — **无 usable 检查** |
| 搜索（ES） | `listable` | `tags_index.rb:37` | `index_scope ::Tag.listable` — **无 usable 检查** |

#### 5.8.3 三条链路的独立性总结

**三条链路完全解耦，互不影响：**

```
usable  ──→  发帖验证 + 趋势计数

listable ──→  搜索功能（含 Elasticsearch 索引）

trendable ──→ 趋势展示（通过 TagTrend.allowed）
```

**结论：**
- 管理员可以单独控制"能否发帖使用"、"能否被搜索"、"能否上趋势"
- `listable = false` **绝不会**阻止 hashtag 上趋势榜（场景 3 完全合法）
- 只有 `trendable = false` 会阻止趋势展示

### 5.9 实例级别的发现控制

除了 tag 级别的标志，还有实例级别的设置：

`config/settings.yml:15-18`：

```yaml
local_live_feed_access: 'public'
remote_live_feed_access: 'public'
local_topic_feed_access: 'public'   # 本地 hashtag 时间线
remote_topic_feed_access: 'public'  # 远程 hashtag 时间线
```

这些设置控制 hashtag 时间线对访客/未登录用户的可见性。

---

## 总结

Mastodon 的 hashtag 趋势系统设计体现了以下特点：

1. **技术先进性**：使用 Redis HyperLogLog 进行高效去重计数，使用卡方统计量检测异常增长，使用指数衰减处理热度消退

2. **审核灵活性**：三级标志位（usable、listable、trendable）提供精细的内容控制粒度，**三条链路完全解耦**：
   - `usable` 控制发帖和趋势计数，**不影响搜索可见性**
   - `listable` 仅控制搜索（含 ES 索引），**不影响趋势**
   - `trendable` 仅控制趋势展示，**不影响搜索**
   
   ⚠️ **两个重要澄清**：
   - 设置 `listable = false` 不会阻止 hashtag 上趋势榜（场景 3 可证明）
   - 设置 `usable = false` 不会阻止 hashtag 被搜索到（场景 5a 可证明）

3. **隐私保护**：默认 `trendable_by_default = false`，需要人工审核才能上趋势榜，防止算法滥用

4. **去中心化**：趋势计算是实例本地的，每个实例有自己的趋势榜，不受其他实例影响

5. **多语言友好**：正则表达式支持 Unicode 字母字符，查询支持语言偏好排序
