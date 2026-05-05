# Mastodon Filter v2 多客户端分发与同步机制

本文档详细分析 Mastodon Filter v2 在 Web、移动端和第三方客户端上的 API 分发、同步机制，客户端缓存与服务端强制执行的边界，以及 v1 与 v2 的兼容路径。

---

## 目录

1. [Filter v2 核心数据模型](#1-filter-v2-核心数据模型)
2. [API 分发机制](#2-api-分发机制)
3. [实时同步机制](#3-实时同步机制)
4. [客户端缓存策略](#4-客户端缓存策略)
5. [服务端强制执行逻辑](#5-服务端强制执行逻辑)
6. [客户端与服务端边界划分](#6-客户端与服务端边界划分)
7. [v1 与 v2 兼容路径](#7-v1-与-v2-兼容路径)
8. [第三方客户端开发指南](#8-第三方客户端开发指南)

---

## 1. Filter v2 核心数据模型

### 1.1 实体关系

```
CustomFilter (过滤器组)
├── id: 过滤器唯一标识
├── title: 过滤器名称 (原 v1 的 phrase)
├── context: 应用上下文数组 ['home', 'notifications', 'public', 'thread', 'account']
├── expires_at: 过期时间
├── filter_action: 过滤行为 (warn: 警告 | hide: 隐藏 | blur: 模糊)
├── keywords: CustomFilterKeyword[] (多关键词支持)
└── statuses: CustomFilterStatus[] (特定状态过滤)

CustomFilterKeyword (关键词规则)
├── keyword: 关键词文本
└── whole_word: 是否整词匹配

CustomFilterStatus (特定状态过滤)
└── status_id: 要过滤的状态 ID
```

### 1.2 数据模型源码位置

| 组件 | 文件路径 |
|------|----------|
| 过滤器模型 | `app/models/custom_filter.rb` |
| 关键词模型 | `app/models/custom_filter_keyword.rb` |
| 状态过滤模型 | `app/models/custom_filter_status.rb` |
| 缓存管理 | `app/models/concerns/custom_filter_cache.rb` |

---

## 2. API 分发机制

### 2.1 V2 API 端点

**基础路径**: `/api/v2/filters`

| 方法 | 端点 | 描述 | 权限 |
|------|------|------|------|
| GET | `/api/v2/filters` | 获取所有过滤器 | `read:filters` |
| GET | `/api/v2/filters/:id` | 获取单个过滤器 | `read:filters` |
| POST | `/api/v2/filters` | 创建过滤器 | `write:filters` |
| PUT/PATCH | `/api/v2/filters/:id` | 更新过滤器 | `write:filters` |
| DELETE | `/api/v2/filters/:id` | 删除过滤器 | `write:filters` |

### 2.2 API 控制器实现

**V2 控制器**: `app/controllers/api/v2/filters_controller.rb`

```ruby
# 索引 - 返回所有过滤器及规则
def index
  render json: @filters, each_serializer: REST::FilterSerializer, rules_requested: true
end

# 创建 - 支持批量关键词
def create
  @filter = current_account.custom_filters.create!(resource_params)
  render json: @filter, serializer: REST::FilterSerializer, rules_requested: true
end

# 参数结构
def resource_params
  params.permit(
    :title,           # 过滤器名称
    :expires_in,      # 过期时间 (秒)
    :filter_action,   # warn | hide | blur
    context: [],      # 应用上下文
    keywords_attributes: [:id, :keyword, :whole_word, :_destroy]
  )
end
```

### 2.3 V2 序列化器

**文件**: `app/serializers/rest/filter_serializer.rb`

```ruby
class REST::FilterSerializer < ActiveModel::Serializer
  attributes :id, :title, :context, :expires_at, :filter_action
  has_many :keywords, serializer: REST::FilterKeywordSerializer, if: :rules_requested?
  has_many :statuses, serializer: REST::FilterStatusSerializer, if: :rules_requested?
end
```

**API 响应示例**:

```json
{
  "id": "12345",
  "title": "Spam Keywords",
  "context": ["home", "notifications"],
  "expires_at": "2024-12-31T23:59:59Z",
  "filter_action": "hide",
  "keywords": [
    {
      "id": "1",
      "keyword": "crypto scam",
      "whole_word": false
    },
    {
      "id": "2",
      "keyword": "free money",
      "whole_word": true
    }
  ],
  "statuses": [
    {
      "id": "1",
      "status_id": "987654321"
    }
  ]
}
```

### 2.4 Web 前端 API 调用

**文件**: `app/javascript/mastodon/actions/filters.js`

```javascript
// 获取所有过滤器
export const fetchFilters = () => (dispatch) => {
  dispatch({ type: FILTERS_FETCH_REQUEST });

  api()
    .get('/api/v2/filters')
    .then(({ data }) => dispatch({
      type: FILTERS_FETCH_SUCCESS,
      filters: data,
    }))
    .catch(err => dispatch({
      type: FILTERS_FETCH_FAIL,
      err,
    }));
};

// 创建过滤器
export const createFilter = (params, onSuccess, onFail) => (dispatch) => {
  dispatch(createFilterRequest());
  api().post('/api/v2/filters', params).then(response => {
    dispatch(createFilterSuccess(response.data));
    if (onSuccess) onSuccess(response.data);
  });
};
```

---

## 3. 实时同步机制

### 3.1 同步架构概览

```
┌─────────────────┐     Redis Pub/Sub      ┌─────────────────┐
│   Rails 服务    │ ──────────────────────> │  Streaming 服务 │
│  (过滤器变更)   │  channel:               │  (实时推送)     │
│                 │  - timeline:{accountId} │                 │
│                 │  - timeline:system:{id} │                 │
└─────────────────┘                         └────────┬────────┘
                                                      │
                                                      ▼ WebSocket/EventSource
┌─────────────────────────────────────────────────────────────────┐
│                        客户端 (Web/移动端/第三方)                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  接收事件     │───>│  清空本地缓存 │───>│  重新获取过滤器   │  │
│  │filters_changed│    │              │    │  GET /api/v2/filters││
│  └──────────────┘    └──────────────┘    └──────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 服务端缓存失效机制

**文件**: `app/models/custom_filter.rb`

```ruby
# 缓存键
Rails.cache.fetch("filters:v3:#{account_id}")

# 变更前标记
before_save :prepare_cache_invalidation!
before_destroy :prepare_cache_invalidation!

# 提交后失效缓存并推送事件
after_commit :invalidate_cache!

def invalidate_cache!
  return unless @should_invalidate_cache
  
  @should_invalidate_cache = false
  
  # 1. 清除 Rails 缓存
  Rails.cache.delete("filters:v3:#{account_id}")
  
  # 2. 向 Redis 发布事件
  redis.publish("timeline:#{account_id}", { event: :filters_changed }.to_json)
  redis.publish("timeline:system:#{account_id}", { event: :filters_changed }.to_json)
end
```

### 3.3 Streaming 服务事件处理

**文件**: `streaming/index.js`

```javascript
// 系统消息监听器
const createSystemMessageListener = (req, eventHandlers) => {
  return message => {
    const { event } = message;
    
    if (event === 'filters_changed') {
      req.log.debug(`Invalidating filters cache for ${req.accountId}`);
      // 清空 Streaming 服务本地的过滤器缓存
      req.cachedFilters = null;
    }
  };
};

// 订阅系统频道
const subscribeWebsocketToSystemChannel = ({ websocket, request, subscriptions }) => {
  const accessTokenChannelId = `timeline:access_token:${request.accessTokenId}`;
  const systemChannelId = `timeline:system:${request.accountId}`;
  
  // 监听这两个频道的 filters_changed 事件
  subscribe(accessTokenChannelId, listener);
  subscribe(systemChannelId, listener);
};
```

### 3.4 前端事件处理

**文件**: `app/javascript/mastodon/stream.js`

```javascript
// 已知事件类型列表
const KNOWN_EVENT_TYPES = [
  'update',
  'delete',
  'notification',
  'conversation',
  'filters_changed',  // 过滤器变更事件
  'announcement',
  // ...
];
```

**客户端处理建议**:

```javascript
// 第三方客户端应监听 filters_changed 事件
function handleStreamEvent(event) {
  switch (event.type) {
    case 'filters_changed':
      // 1. 清空本地过滤器缓存
      clearLocalFiltersCache();
      // 2. 重新从服务器获取最新过滤器
      fetchAndUpdateFilters();
      // 3. 重新应用过滤到当前时间线
      reapplyFiltersToCurrentTimeline();
      break;
    // ... 其他事件
  }
}
```

### 3.5 级联缓存失效

**文件**: `app/models/concerns/custom_filter_cache.rb`

```ruby
module CustomFilterCache
  extend ActiveSupport::Concern

  included do
    after_commit :invalidate_cache!
    before_destroy :prepare_cache_invalidation!
    before_save :prepare_cache_invalidation!

    delegate(
      :invalidate_cache!,
      :prepare_cache_invalidation!,
      to: :custom_filter  // 委托给关联的过滤器
    )
  end
end
```

**说明**: 当 `CustomFilterKeyword` 或 `CustomFilterStatus` 变更时，会自动触发所属 `CustomFilter` 的缓存失效。

---

## 4. 客户端缓存策略

### 4.1 Web 前端状态管理

**Redux 状态结构**: `app/javascript/mastodon/reducers/filters.js`

```javascript
// 状态结构 (ImmutableMap)
{
  "filter_id_1": {
    id: "filter_id_1",
    title: "Filter Title",
    context: ["home", "notifications"],
    filter_action: "hide",
    keywords: [
      { id: "kw1", keyword: "spam", whole_word: false }
    ],
    expires_at: 1735689599000  // Unix 时间戳 (ms)
  }
}

// 规范化函数
const normalizeFilter = (state, filter) => {
  const normalizedFilter = fromJS({
    id: filter.id,
    title: filter.title,
    context: filter.context,
    filter_action: filter.filter_action,
    keywords: filter.keywords,
    expires_at: filter.expires_at ? Date.parse(filter.expires_at) : null,
  });

  // 智能合并：不覆盖已存在的关键词
  return state.update(filter.id, ImmutableMap(), (old) => (
    old.mergeWith(
      ((old_value, new_value) => (new_value === undefined ? old_value : new_value)),
      normalizedFilter
    )
  ));
};
```

### 4.2 过滤器选择器

**文件**: `app/javascript/mastodon/selectors/filters.ts`

```typescript
// 根据上下文类型获取有效过滤器
export const getFilters = createSelector(
  [
    (state: RootState) => state.filters,
    (_, { contextType }: { contextType: string }) => contextType,
  ],
  (filters, contextType) => {
    if (!contextType) return null;

    const now = new Date();
    const serverSideType = toServerSideType(contextType);

    // 过滤条件：
    // 1. 上下文匹配
    // 2. 未过期
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

// 检查状态是否应该被隐藏
export const getStatusHidden = (
  state: RootState,
  { id, contextType }: { id: string; contextType: string },
) => {
  const filters = getFilters(state, { contextType });
  if (filters === null) return false;

  // 从状态中获取预计算的过滤结果
  const filtered = state.statuses.getIn([id, 'filtered']);
  return filtered?.some(
    (result) =>
      filters.getIn([result.get('filter'), 'filter_action']) === 'hide',
  );
};
```

### 4.3 上下文类型映射

**文件**: `app/javascript/mastodon/utils/filters.ts`

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
      return 'thread';           // 详情页 → thread
    case 'bookmarks':
    case 'favourites':
      return 'home';             // 收藏/书签 → home
    default:
      if (columnType.includes('list:')) {
        return 'home';           // 列表 → home
      } else {
        return 'public';         // 其他 → public
      }
  }
};
```

### 4.4 流式数据中的过滤器同步

**文件**: `app/javascript/mastodon/actions/importer/index.js`

```javascript
// 从状态数据中提取过滤器信息
export function importFetchedStatuses(statuses, options = {}) {
  return (dispatch, getState) => {
    const filters = [];

    function processStatus(status) {
      // 从 status.filtered 中提取过滤器
      if (status.filtered) {
        status.filtered.forEach(result => pushUnique(filters, result.filter));
      }
      
      if (status.reblog?.id) {
        processStatus(status.reblog);
      }
    }

    statuses.forEach(processStatus);
    
    // 导入到 Redux 状态
    dispatch(importFilters(filters));
  };
}

// Reducer 处理
case FILTERS_IMPORT:
  return normalizeFilters(state, action.filters);
```

### 4.5 客户端缓存最佳实践

| 缓存层级 | 存储位置 | 刷新时机 | 适用场景 |
|---------|---------|---------|---------|
| 内存缓存 | Redux Store | 每次 `fetchFilters`、`filters_changed` 事件 | Web 前端实时过滤 |
| 本地存储 | localStorage/IndexedDB | 应用启动时、`filters_changed` 后 | 移动端/第三方客户端离线使用 |
| 会话缓存 | Streaming 服务内存 | `filters_changed` 事件 | 实时流过滤 |

---

## 5. 服务端强制执行逻辑

### 5.1 服务端缓存结构

**文件**: `app/models/custom_filter.rb`

```ruby
# 获取缓存的过滤器 (v3 版本)
def self.cached_filters_for(account_id)
  active_filters = Rails.cache.fetch("filters:v3:#{account_id}") do
    filters_hash = {}

    # 1. 构建关键词过滤器
    scope = CustomFilterKeyword.left_outer_joins(:custom_filter)
                                .merge(unexpired.where(account_id: account_id))
    
    scope.to_a.group_by(&:custom_filter).each do |filter, keywords|
      keywords.map!(&:to_regex)  # 转换为正则表达式
      filters_hash[filter.id] = { 
        keywords: Regexp.union(keywords), 
        filter: filter 
      }
    end

    # 2. 构建状态过滤器
    scope = CustomFilterStatus.left_outer_joins(:custom_filter)
                               .merge(unexpired.where(account_id: account_id))
    
    scope.to_a.group_by(&:custom_filter).each do |filter, statuses|
      filters_hash[filter.id] ||= { filter: filter }
      filters_hash[filter.id].merge!(status_ids: statuses.map(&:status_id))
    end

    filters_hash.values.map { |cache| [cache.delete(:filter), cache] }
  end

  # 二次检查过期
  active_filters.reject { |custom_filter, _| custom_filter.expired? }
end
```

### 5.2 过滤应用逻辑

```ruby
# 应用过滤器到状态
def self.apply_cached_filters(cached_filters, status)
  cached_filters.filter_map do |filter, rules|
    # 1. 关键词匹配
    match = rules[:keywords].match(status.proper.searchable_text) if rules[:keywords].present?
    keyword_matches = [match.to_s] unless match.nil?

    # 2. 状态 ID 匹配
    status_matches = [status.id, status.reblog_of_id].compact & rules[:status_ids] if rules[:status_ids].present?

    # 3. 返回过滤结果
    next if keyword_matches.blank? && status_matches.blank?

    FilterResultPresenter.new(
      filter: filter,
      keyword_matches: keyword_matches,
      status_matches: status_matches
    )
  end
end
```

### 5.3 Account 层面的过滤接口

**文件**: `app/models/concerns/account/interactions.rb`

```ruby
def status_matches_filters(status)
  active_filters = CustomFilter.cached_filters_for(id)
  CustomFilter.apply_cached_filters(active_filters, status)
end
```

### 5.4 状态序列化时的过滤计算

**文件**: `app/serializers/rest/status_serializer.rb`

```ruby
class REST::StatusSerializer < ActiveModel::Serializer
  # ...
  
  has_many :filtered, serializer: REST::FilterResultSerializer, if: :current_user?

  def filtered
    if relationships
      # 使用预计算的过滤结果 (批量查询优化)
      relationships.filters_map[object.id] || []
    else
      # 实时计算
      current_user.account.status_matches_filters(object)
    end
  end
end
```

### 5.5 FilterResult 实体

**文件**: `app/presenters/filter_result_presenter.rb`

```ruby
class FilterResultPresenter < ActiveModelSerializers::Model
  attributes :filter, :keyword_matches, :status_matches
end
```

**API 响应中的 `filtered` 字段**:

```json
{
  "id": "123456",
  "content": "...",
  "filtered": [
    {
      "filter": {
        "id": "filter_1",
        "title": "Spam Filter",
        "context": ["home"],
        "expires_at": null,
        "filter_action": "hide"
      },
      "keyword_matches": ["crypto scam"],
      "status_matches": null
    }
  ]
}
```

### 5.6 Streaming 服务中的过滤

**文件**: `streaming/index.js` (关键逻辑)

```javascript
// Streaming 服务的过滤分两种情况：

// 情况 1: 负载已包含 filtered 属性 (Rails 侧已计算)
if (Object.hasOwn(payload, "filtered")) {
  transmit(event, payload);  // 直接透传
  return;
}

// 情况 2: 需要在 Streaming 侧计算 (公共时间线等)
if (!req.cachedFilters) {
  // 从数据库查询并构建过滤器缓存
  const filterRows = values[...].rows;
  req.cachedFilters = filterRows.reduce((cache, filter) => {
    // 构建关键词正则表达式
    cache[filter.id].regexp = new RegExp(
      keywords.map(([keyword, whole_word]) => {
        let expr = keyword.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
        if (whole_word) {
          if (/^[\w]/.test(expr)) expr = `\\b${expr}`;
          if (/[\w]$/.test(expr)) expr = `${expr}\\b`;
        }
        return expr;
      }).join('|'), 'i'
    );
    return cache;
  }, {});
}

// 应用过滤
if (req.cachedFilters) {
  // 构建可搜索文本
  const searchableContent = ([
    status.spoiler_text || '', 
    status.content
  ]).concat(
    (status.poll && status.poll.options) ? 
      status.poll.options.map(option => option.title) : []
  ).concat(
    status.media_attachments.map(att => att.description)
  ).join('\n\n');
  
  const searchableTextContent = JSDOM.fragment(searchableContent).textContent;
  
  // 匹配正则
  const filter_results = Object.values(req.cachedFilters).reduce((results, cachedFilter) => {
    if (cachedFilter.expires_at !== null && cachedFilter.expires_at < now) {
      return results;  // 已过期
    }
    
    const keyword_matches = searchableTextContent.match(cachedFilter.regexp);
    if (keyword_matches) {
      results.push({
        filter: cachedFilter.filter,
        keyword_matches,
        status_matches: null
      });
    }
    return results;
  }, []);
  
  transmit(event, {
    ...payload,
    filtered: filter_results  // 添加过滤结果
  });
}
```

### 5.7 通知过滤

**文件**: `app/services/notify_service.rb` (隐含逻辑)

通知的过滤在两个层面执行：
1. **创建时**: 检查是否匹配过滤器，决定是否创建通知
2. **传递时**: Streaming 服务和序列化器再次应用过滤

---

## 6. 客户端与服务端边界划分

### 6.1 职责划分总览

```
┌────────────────────────────────────────────────────────────────────┐
│                         服务端 (强制执行)                            │
├────────────────────────────────────────────────────────────────────┤
│  ✅ 决定状态是否匹配过滤器规则                                        │
│  ✅ 计算 filtered 数组并附加到响应                                    │
│  ✅ 维护过滤器的缓存和失效机制                                        │
│  ✅ 通过 WebSocket 推送 filters_changed 事件                         │
│  ✅ 对公共时间线在 Streaming 侧执行过滤计算                           │
│  ✅ 处理 hide 动作的不可逆过滤 (通知清理等)                           │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ filtered[] 数组
┌────────────────────────────────────────────────────────────────────┐
│                         客户端 (表现层)                              │
├────────────────────────────────────────────────────────────────────┤
│  ✅ 缓存过滤器列表用于 UI 展示                                        │
│  ✅ 根据 filtered[].filter.filter_action 决定 UI 行为               │
│  ✅ 处理警告对话框、模糊显示等交互                                     │
│  ✅ 监听 filters_changed 事件并刷新缓存                               │
│  ❌ 不独立执行过滤匹配逻辑 (依赖服务端的 filtered 数组)               │
│  ❌ 不修改过滤器的 filter_action 含义                                 │
└────────────────────────────────────────────────────────────────────┘
```

### 6.2 详细边界表

| 功能 | 服务端 | 客户端 | 说明 |
|------|--------|--------|------|
| **过滤规则匹配** | ✅ 负责 | ❌ 不负责 | 服务端计算 `filtered` 数组 |
| **过滤结果传递** | ✅ 提供 | ✅ 使用 | 服务端在响应中附加 `filtered` |
| **filter_action 语义** | ✅ 定义 | ✅ 遵循 | hide=隐藏, warn=警告, blur=模糊 |
| **不可逆过滤** | ✅ 执行 | ❌ 不参与 | hide 动作可能触发通知清理等 |
| **缓存维护** | ✅ 服务端缓存 | ✅ 本地缓存 | 双重缓存，通过事件同步 |
| **变更推送** | ✅ 发起 | ✅ 接收 | `filters_changed` 事件 |

### 6.3 filter_action 的语义

| 值 | 服务端行为 | 客户端行为 |
|----|-----------|-----------|
| `warn` | 标记过滤结果 | 显示警告/折叠内容，用户可展开 |
| `hide` | 标记过滤结果，可能触发清理逻辑 | 完全隐藏内容 |
| `blur` | 标记过滤结果 | 模糊显示敏感媒体/内容 |

**重要**: 服务端不直接执行隐藏操作，只是提供 `filter_action` 建议。客户端根据此建议决定 UI 表现。

### 6.4 为什么客户端不独立过滤？

**设计决策原因**:

1. **一致性保证**: 所有客户端看到相同的过滤结果
2. **计算效率**: 服务端可以批量计算、缓存优化
3. **数据完整性**: 服务端能访问完整的搜索文本 (包括 CW、媒体描述等)
4. **安全性**: 防止客户端绕过过滤规则
5. **未来兼容**: 服务端过滤逻辑可独立升级

### 6.5 客户端的"轻量"过滤

客户端确实存在一些"轻量"过滤逻辑，但这些是**基于服务端结果的二次处理**:

```typescript
// 从 selectors/filters.ts 可以看到
export const getStatusHidden = (state, { id, contextType }) => {
  const filters = getFilters(state, { contextType });
  
  // 使用服务端提供的 filtered 数组
  const filtered = state.statuses.getIn([id, 'filtered']);
  
  // 检查是否有 filter_action 为 'hide' 的过滤结果
  return filtered?.some(
    (result) =>
      filters.getIn([result.get('filter'), 'filter_action']) === 'hide',
  );
};
```

**关键点**: 客户端只是**解释**服务端的过滤结果，而不是**重新计算**过滤匹配。

---

## 7. v1 与 v2 兼容路径

### 7.1 版本差异总览

| 特性 | v1 (已废弃) | v2 |
|------|------------|-----|
| **废弃状态** | 活跃废弃 (deprecate_api '2022-11-14') | 当前版本 |
| **数据模型** | 单关键词 = 一个 Filter | 过滤器组 = Filter + N 关键词 |
| **API 端点** | `/api/v1/filters` | `/api/v2/filters` |
| **序列化器** | `REST::V1::FilterSerializer` | `REST::FilterSerializer` |
| **过滤动作** | `irreversible` (布尔) | `filter_action` (枚举: warn/hide/blur) |
| **状态过滤** | 不支持 | 支持 (`CustomFilterStatus`) |

### 7.2 v1 API 控制器实现

**文件**: `app/controllers/api/v1/filters_controller.rb`

```ruby
class Api::V1::FiltersController < Api::BaseController
  include DeprecationConcern
  deprecate_api '2022-11-14'  # 标记废弃

  # v1 的 "索引" 实际返回的是关键词列表
  def set_filters
    @filters = CustomFilterKeyword.includes(:custom_filter)
                                   .where(custom_filter: { account: current_account })
  end

  # v1 的 "show" 返回单个关键词
  def set_filter
    @filter = CustomFilterKeyword.includes(:custom_filter)
                                  .where(custom_filter: { account: current_account })
                                  .find(params[:id])
  end

  # v1 创建：创建一个过滤器组 + 一个关键词
  def create
    ApplicationRecord.transaction do
      filter_category = current_account.custom_filters.create!(filter_params)
      @filter = filter_category.keywords.create!(keyword_params)
    end
    render json: @filter, serializer: REST::V1::FilterSerializer
  end

  # v1 更新：有局限性
  def update
    ApplicationRecord.transaction do
      @filter.update!(keyword_params)
      @filter.custom_filter.assign_attributes(filter_params)
      
      # 限制：如果过滤器组有多个关键词，不能通过 v1 API 修改组属性
      raise Mastodon::ValidationError, 
        I18n.t('filters.errors.deprecated_api_multiple_keywords') 
        if @filter.custom_filter.changed? && @filter.custom_filter.keywords.many?

      @filter.custom_filter.save!
    end
  end

  # 参数映射
  def resource_params
    params.permit(:phrase, :expires_in, :irreversible, :whole_word, context: [])
  end

  def filter_params
    resource_params.slice(:phrase, :expires_in, :irreversible, :context)
  end

  def keyword_params
    resource_params.slice(:phrase, :whole_word)
  end
end
```

### 7.3 模型层面的兼容

**文件**: `app/models/custom_filter.rb`

```ruby
class CustomFilter < ApplicationRecord
  # 忽略旧字段 (已迁移)
  self.ignored_columns += %w(whole_word irreversible)

  # 别名兼容
  alias_attribute :title, :phrase           # v1: phrase → v2: title
  alias_attribute :filter_action, :action   # v2: filter_action → 内部: action

  # 枚举定义
  enum :action, { warn: 0, hide: 1, blur: 2 }, suffix: :action, validate: true

  # v1 irreversible 兼容性
  def irreversible=(value)
    # true → hide, false → warn
    self.action = ActiveModel::Type::Boolean.new.cast(value) ? :hide : :warn
  end

  def irreversible?
    hide_action?
  end
end
```

**CustomFilterKeyword 兼容**:

```ruby
class CustomFilterKeyword < ApplicationRecord
  alias_attribute :phrase, :keyword  # v1: phrase → v2: keyword
end
```

### 7.4 v1 序列化器

**文件**: `app/serializers/rest/v1/filter_serializer.rb`

```ruby
class REST::V1::FilterSerializer < ActiveModel::Serializer
  # v1 视角：以关键词为中心
  attributes :id, :phrase, :context, :whole_word, :expires_at, :irreversible

  delegate :context, :expires_at, to: :custom_filter

  def id
    object.id.to_s  # 返回关键词的 ID，不是过滤器组的 ID
  end

  def phrase
    object.keyword  # 关键词文本
  end

  def irreversible
    custom_filter.irreversible?  # 从过滤器组获取
  end

  private

  def custom_filter
    object.custom_filter  # 关联的过滤器组
  end
end
```

### 7.5 v1 vs v2 响应对比

**v1 响应 (GET /api/v1/filters)**:

```json
[
  {
    "id": "keyword_id_1",
    "phrase": "spam",
    "context": ["home", "notifications"],
    "whole_word": false,
    "expires_at": "2024-12-31T23:59:59Z",
    "irreversible": true
  },
  {
    "id": "keyword_id_2",
    "phrase": "scam",
    "context": ["home", "notifications"],  // 与上面相同
    "whole_word": true,
    "expires_at": "2024-12-31T23:59:59Z",
    "irreversible": true
  }
]
```

**v2 响应 (GET /api/v2/filters)**:

```json
[
  {
    "id": "filter_group_id",
    "title": "Anti-Spam",
    "context": ["home", "notifications"],
    "expires_at": "2024-12-31T23:59:59Z",
    "filter_action": "hide",
    "keywords": [
      { "id": "keyword_id_1", "keyword": "spam", "whole_word": false },
      { "id": "keyword_id_2", "keyword": "scam", "whole_word": true }
    ],
    "statuses": []
  }
]
```

### 7.6 迁移路径

**对于服务端**:
- 双 API 并存，v1 内部映射到 v2 数据模型
- v1 API 返回 `Deprecation` 警告头
- 新功能只在 v2 实现 (如状态过滤、blur 动作)

**对于客户端**:

| 客户端类型 | 推荐策略 |
|-----------|---------|
| **新开发客户端** | 直接使用 v2 API |
| **现有 v1 客户端** | 继续使用 v1 API (兼容层会处理)，但应尽快迁移 |
| **跨版本客户端** | 检测服务器支持，优先 v2，降级 v1 |

### 7.7 v1 的局限性

使用 v1 API 存在以下限制：

1. **多关键词限制**: 无法通过 v1 API 修改包含多个关键词的过滤器组属性
2. **无状态过滤**: v1 不支持 `CustomFilterStatus` (特定状态过滤)
3. **无 blur 动作**: v1 只有 `irreversible` (映射到 hide/warn)，不支持 blur
4. **ID 混淆**: v1 返回的是关键词 ID，不是过滤器组 ID
5. **废弃警告**: 响应包含 `Deprecation` HTTP 头

---

## 8. 第三方客户端开发指南

### 8.1 API 使用建议

**推荐流程**:

```
1. 应用启动时:
   └── GET /api/v2/filters → 缓存到本地存储

2. 建立 WebSocket/EventSource 连接:
   └── 订阅 timeline 和 system 频道
   └── 监听 filters_changed 事件

3. 收到 filters_changed 时:
   ├── 清空本地缓存
   ├── 重新 GET /api/v2/filters
   └── 重新应用过滤到当前显示的时间线

4. 处理时间线/通知数据时:
   └── 检查 status.filtered 数组
   └── 根据 filter_result.filter.filter_action 决定 UI 行为
```

### 8.2 关键类型定义

```typescript
// Filter v2 核心类型

interface Filter {
  id: string;
  title: string;
  context: FilterContext[];
  expires_at: string | null;
  filter_action: 'warn' | 'hide' | 'blur';
  keywords: FilterKeyword[];
  statuses: FilterStatus[];
}

type FilterContext = 'home' | 'notifications' | 'public' | 'thread' | 'account';

interface FilterKeyword {
  id: string;
  keyword: string;
  whole_word: boolean;
}

interface FilterStatus {
  id: string;
  status_id: string;
}

interface FilterResult {
  filter: Filter;
  keyword_matches: string[] | null;
  status_matches: string[] | null;
}

interface Status {
  id: string;
  // ... 其他字段
  filtered?: FilterResult[];  // 关键：服务端计算的过滤结果
}
```

### 8.3 上下文映射

客户端需要将 UI 上下文映射到服务端上下文:

```typescript
function toServerSideContext(clientContext: string): FilterContext {
  switch (clientContext) {
    case 'home':
    case 'notifications':
    case 'public':
    case 'thread':
    case 'account':
      return clientContext as FilterContext;
    case 'detailed':
    case 'thread-view':
      return 'thread';
    case 'bookmarks':
    case 'favourites':
    case 'list':
      return 'home';
    default:
      return 'public';
  }
}
```

### 8.4 过滤器生效检查

```typescript
function isFilterActiveInContext(
  filter: Filter, 
  context: FilterContext, 
  now: Date = new Date()
): boolean {
  // 1. 检查上下文匹配
  if (!filter.context.includes(context)) {
    return false;
  }
  
  // 2. 检查是否过期
  if (filter.expires_at) {
    const expireDate = new Date(filter.expires_at);
    if (expireDate <= now) {
      return false;
    }
  }
  
  return true;
}
```

### 8.5 状态过滤行为处理

```typescript
function determineFilterBehavior(
  status: Status,
  filters: Map<string, Filter>,
  context: FilterContext
): { 
  shouldHide: boolean; 
  shouldWarn: boolean; 
  shouldBlur: boolean;
  activeFilters: FilterResult[];
} {
  const result = {
    shouldHide: false,
    shouldWarn: false,
    shouldBlur: false,
    activeFilters: [] as FilterResult[],
  };

  if (!status.filtered || status.filtered.length === 0) {
    return result;
  }

  const now = new Date();

  for (const filterResult of status.filtered) {
    const filter = filterResult.filter;
    
    // 检查过滤器在当前上下文中是否生效
    if (!isFilterActiveInContext(filter, context, now)) {
      continue;
    }

    result.activeFilters.push(filterResult);

    // 根据 filter_action 决定行为
    switch (filter.filter_action) {
      case 'hide':
        result.shouldHide = true;
        break;
      case 'warn':
        result.shouldWarn = true;
        break;
      case 'blur':
        result.shouldBlur = true;
        break;
    }
  }

  // 优先级: hide > warn/blur
  if (result.shouldHide) {
    result.shouldWarn = false;
    result.shouldBlur = false;
  }

  return result;
}
```

### 8.6 WebSocket 事件处理

```typescript
interface StreamEvent {
  stream: string[];
  event: string;
  payload?: string;
}

class FilterSyncManager {
  private filters: Map<string, Filter> = new Map();
  private onFiltersChanged?: () => void;

  constructor(onFiltersChanged?: () => void) {
    this.onFiltersChanged = onFiltersChanged;
  }

  // 处理 WebSocket 消息
  handleStreamEvent(event: StreamEvent): void {
    if (event.event === 'filters_changed') {
      this.handleFiltersChanged();
    }
  }

  private async handleFiltersChanged(): Promise<void> {
    // 1. 触发 UI 更新前的准备
    // 2. 重新获取过滤器
    await this.fetchFilters();
    // 3. 通知 UI 更新
    this.onFiltersChanged?.();
  }

  // 从服务器获取最新过滤器
  async fetchFilters(): Promise<void> {
    const response = await fetch('/api/v2/filters', {
      headers: {
        'Authorization': `Bearer ${this.accessToken}`,
      },
    });
    
    const filters: Filter[] = await response.json();
    this.filters = new Map(filters.map(f => [f.id, f]));
    
    // 持久化到本地存储
    this.persistToStorage();
  }

  // 获取过滤器
  getFilter(id: string): Filter | undefined {
    return this.filters.get(id);
  }

  // 获取指定上下文的有效过滤器
  getActiveFilters(context: FilterContext): Filter[] {
    const now = new Date();
    return Array.from(this.filters.values()).filter(f => 
      isFilterActiveInContext(f, context, now)
    );
  }
}
```

### 8.7 错误处理和降级策略

```typescript
class FilterManager {
  // 获取过滤器时的降级策略
  async getFiltersWithFallback(): Promise<Filter[]> {
    try {
      // 尝试 v2 API
      return await this.fetchV2Filters();
    } catch (v2Error) {
      // v2 失败，尝试 v1
      try {
        const v1Filters = await this.fetchV1Filters();
        return this.convertV1ToV2(v1Filters);
      } catch (v1Error) {
        // 都失败，使用本地缓存
        return this.getCachedFilters() || [];
      }
    }
  }

  // v1 到 v2 的转换 (用于降级)
  private convertV1ToV2(v1Filters: V1Filter[]): Filter[] {
    // 注意：v1 每个关键词都是独立的"过滤器"
    // 转换时可以将它们分组或保持独立
    
    return v1Filters.map((v1, index) => ({
      id: `v1-converted-${v1.id}`,
      title: v1.phrase,  // v1 没有 title，用 phrase 代替
      context: v1.context,
      expires_at: v1.expires_at,
      filter_action: v1.irreversible ? 'hide' : 'warn',
      keywords: [{
        id: v1.id,
        keyword: v1.phrase,
        whole_word: v1.whole_word,
      }],
      statuses: [],  // v1 不支持状态过滤
    }));
  }
}
```

### 8.8 测试建议

**测试场景**:

1. **基本过滤**:
   - 创建包含多个关键词的过滤器
   - 发布包含关键词的状态
   - 验证时间线中状态的 `filtered` 数组

2. **不同 filter_action**:
   - 测试 `warn`：状态应显示警告但可展开
   - 测试 `hide`：状态应完全隐藏
   - 测试 `blur`：媒体应模糊显示

3. **上下文过滤**:
   - 过滤器只在 `notifications` 上下文生效
   - 验证主页不应用此过滤

4. **实时同步**:
   - 客户端 A 创建过滤器
   - 客户端 B 应收到 `filters_changed` 事件
   - 客户端 B 应刷新过滤器列表

5. **过期处理**:
   - 创建有过期时间的过滤器
   - 等待过期后验证过滤器不再生效

---

## 附录

### A. 关键文件索引

| 功能 | 文件路径 |
|------|----------|
| V2 API 控制器 | `app/controllers/api/v2/filters_controller.rb` |
| V1 API 控制器 | `app/controllers/api/v1/filters_controller.rb` |
| 过滤器模型 | `app/models/custom_filter.rb` |
| 关键词模型 | `app/models/custom_filter_keyword.rb` |
| 状态过滤模型 | `app/models/custom_filter_status.rb` |
| 缓存管理 | `app/models/concerns/custom_filter_cache.rb` |
| V2 序列化器 | `app/serializers/rest/filter_serializer.rb` |
| V1 序列化器 | `app/serializers/rest/v1/filter_serializer.rb` |
| 过滤结果呈现器 | `app/presenters/filter_result_presenter.rb` |
| 前端过滤器 Actions | `app/javascript/mastodon/actions/filters.js` |
| 前端过滤器 Reducer | `app/javascript/mastodon/reducers/filters.js` |
| 前端过滤器选择器 | `app/javascript/mastodon/selectors/filters.ts` |
| 前端工具函数 | `app/javascript/mastodon/utils/filters.ts` |
| 流式连接管理 | `app/javascript/mastodon/stream.js` |
| Streaming 服务 | `streaming/index.js` |
| Account 交互模块 | `app/models/concerns/account/interactions.rb` |

### B. API 端点速查

**V2 过滤器管理**:
- `GET /api/v2/filters` - 列出所有过滤器
- `GET /api/v2/filters/:id` - 获取单个过滤器
- `POST /api/v2/filters` - 创建过滤器
- `PUT /api/v2/filters/:id` - 更新过滤器
- `DELETE /api/v2/filters/:id` - 删除过滤器

**V2 关键词管理** (嵌套在过滤器下):
- `POST /api/v2/filters/:filter_id/keywords` - 添加关键词
- `PUT /api/v2/filters/:filter_id/keywords/:id` - 更新关键词
- `DELETE /api/v2/filters/:filter_id/keywords/:id` - 删除关键词

**V2 状态过滤管理**:
- `POST /api/v2/filters/:filter_id/statuses` - 添加状态过滤
- `DELETE /api/v2/filters/:filter_id/statuses/:id` - 移除状态过滤

**V1 (已废弃)**:
- `GET /api/v1/filters`
- `GET /api/v1/filters/:id`
- `POST /api/v1/filters`
- `PUT /api/v1/filters/:id`
- `DELETE /api/v1/filters/:id`

### C. 变更历史

| 版本 | 变更内容 |
|------|---------|
| v1 引入 | 初始版本，单关键词模型 |
| v2 引入 | 过滤器组模型，多关键词支持，状态过滤，filter_action 枚举 |
| 2022-11-14 | v1 API 标记废弃 |

---

**文档版本**: 1.0  
**基于代码版本**: Mastodon (当前仓库状态)  
**生成日期**: 2026-05-05
