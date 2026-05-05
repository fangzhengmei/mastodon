# Mastodon Filter v2 多客户端分发与同步机制

> **重要说明**：本文档基于 Mastodon 主仓库代码分析。该仓库**不包含原生移动端应用代码**（iOS/Android/React Native），仅包含：
> - React Web 前端 (`app/javascript/mastodon/`)
> - PWA 支持（Service Worker + Web Push 通知）
> 
> 对于原生移动端客户端（如官方 iOS/Android 应用或第三方客户端），请参考 [第二部分：第三方客户端开发指南](#第二部分第三方客户端开发指南)。

---

## 目录

### 第一部分：项目现有实现分析
1. [Filter v2 核心数据模型](#1-filter-v2-核心数据模型)
2. [服务端 API 实现](#2-服务端-api-实现)
3. [服务端强制执行逻辑](#3-服务端强制执行逻辑)
4. [实时同步机制](#4-实时同步机制)
5. [Web 前端实现](#5-web-前端实现)
6. [PWA 与移动端浏览器支持](#6-pwa-与移动端浏览器支持)
7. [v1 与 v2 兼容实现](#7-v1-与-v2-兼容实现)

### 第二部分：第三方客户端开发指南
8. [API 使用建议](#8-api-使用建议)
9. [缓存策略设计](#9-缓存策略设计)
10. [实时同步实现](#10-实时同步实现)
11. [过滤行为处理](#11-过滤行为处理)
12. [移动端特定考虑](#12-移动端特定考虑)
13. [v1/v2 兼容策略](#13-v1v2-兼容策略)

---

## 第一部分：项目现有实现分析

本部分分析基于 Mastodon 主仓库的实际代码实现。

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
```

### 1.2 源码位置与实现

| 组件 | 文件路径 | 关键代码 |
|------|----------|----------|
| 过滤器模型 | `app/models/custom_filter.rb` | 完整模型定义 |
| 关键词模型 | `app/models/custom_filter_keyword.rb` | 关键词规则 |
| 状态过滤模型 | `app/models/custom_filter_status.rb` | 特定状态过滤 |
| 缓存管理 | `app/models/concerns/custom_filter_cache.rb` | 级联缓存失效 |

**核心模型实现** (`app/models/custom_filter.rb:17-43`):

```ruby
class CustomFilter < ApplicationRecord
  self.ignored_columns += %w(whole_word irreversible)

  alias_attribute :title, :phrase
  alias_attribute :filter_action, :action

  VALID_CONTEXTS = %w(home notifications public thread account).freeze
  EXPIRATION_DURATIONS = [30.minutes, 1.hour, 6.hours, 12.hours, 1.day, 1.week].freeze

  enum :action, { warn: 0, hide: 1, blur: 2 }, suffix: :action, validate: true

  belongs_to :account
  has_many :keywords, class_name: 'CustomFilterKeyword', inverse_of: :custom_filter, dependent: :destroy
  has_many :statuses, class_name: 'CustomFilterStatus', inverse_of: :custom_filter, dependent: :destroy
  accepts_nested_attributes_for :keywords, reject_if: :all_blank, allow_destroy: true
end
```

**关键词正则转换** (`app/models/custom_filter_keyword.rb:26-32`):

```ruby
def to_regex
  if whole_word?
    /(?mix:#{to_regex_sb}#{Regexp.escape(keyword)}#{to_regex_eb})/
  else
    /#{Regexp.escape(keyword)}/i
  end
end
```

---

## 2. 服务端 API 实现

### 2.1 V2 API 控制器

**文件**: `app/controllers/api/v2/filters_controller.rb`

```ruby
class Api::V2::FiltersController < Api::BaseController
  before_action -> { doorkeeper_authorize! :read, :'read:filters' }, only: [:index, :show]
  before_action -> { doorkeeper_authorize! :write, :'write:filters' }, except: [:index, :show]
  before_action :require_user!
  before_action :set_filters, only: :index
  before_action :set_filter, only: [:show, :update, :destroy]

  # 获取所有过滤器及规则
  def index
    render json: @filters, each_serializer: REST::FilterSerializer, rules_requested: true
  end

  # 获取单个过滤器
  def show
    render json: @filter, serializer: REST::FilterSerializer, rules_requested: true
  end

  # 创建过滤器（支持多关键词）
  def create
    @filter = current_account.custom_filters.create!(resource_params)
    render json: @filter, serializer: REST::FilterSerializer, rules_requested: true
  end

  # 更新过滤器
  def update
    @filter.update!(resource_params)
    render json: @filter, serializer: REST::FilterSerializer, rules_requested: true
  end

  # 删除过滤器
  def destroy
    @filter.destroy!
    render_empty
  end

  private

  def set_filters
    @filters = current_account.custom_filters.includes(:keywords, :statuses)
  end

  def resource_params
    params.permit(
      :title,
      :expires_in,
      :filter_action,
      context: [],
      keywords_attributes: [:id, :keyword, :whole_word, :_destroy]
    )
  end
end
```

### 2.2 V2 序列化器

**文件**: `app/serializers/rest/filter_serializer.rb`

```ruby
class REST::FilterSerializer < ActiveModel::Serializer
  attributes :id, :title, :context, :expires_at, :filter_action
  has_many :keywords, serializer: REST::FilterKeywordSerializer, if: :rules_requested?
  has_many :statuses, serializer: REST::FilterStatusSerializer, if: :rules_requested?

  def id
    object.id.to_s
  end

  def rules_requested?
    instance_options[:rules_requested]
  end
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
    { "id": "1", "keyword": "crypto scam", "whole_word": false },
    { "id": "2", "keyword": "free money", "whole_word": true }
  ],
  "statuses": []
}
```

---

## 3. 服务端强制执行逻辑

### 3.1 服务端缓存结构

**文件**: `app/models/custom_filter.rb:70-93`

```ruby
# 获取缓存的过滤器 (v3 版本)
def self.cached_filters_for(account_id)
  active_filters = Rails.cache.fetch("filters:v3:#{account_id}") do
    filters_hash = {}

    # 1. 构建关键词过滤器 - 预编译为正则表达式
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

### 3.2 过滤应用逻辑

**文件**: `app/models/custom_filter.rb:95-106`

```ruby
# 应用过滤器到状态
def self.apply_cached_filters(cached_filters, status)
  cached_filters.filter_map do |filter, rules|
    # 1. 关键词匹配 - 匹配状态的可搜索文本
    match = rules[:keywords].match(status.proper.searchable_text) if rules[:keywords].present?
    keyword_matches = [match.to_s] unless match.nil?

    # 2. 状态 ID 匹配 - 匹配特定状态
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

### 3.3 状态序列化时的过滤计算

**文件**: `app/serializers/rest/status_serializer.rb:147-153`

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

**Account 层面接口** (`app/models/concerns/account/interactions.rb:211-214`):

```ruby
def status_matches_filters(status)
  active_filters = CustomFilter.cached_filters_for(id)
  CustomFilter.apply_cached_filters(active_filters, status)
end
```

### 3.4 Streaming 服务中的过滤

**文件**: `streaming/index.js:768-895`

Streaming 服务在两种情况下执行过滤：

**情况 1: 负载已包含 filtered 属性（Rails 侧已计算）**
```javascript
if (Object.hasOwn(payload, "filtered")) {
  transmit(event, payload);  // 直接透传
  return;
}
```

**情况 2: 需要在 Streaming 侧计算（公共时间线等）**
```javascript
// 1. 从数据库查询并构建过滤器缓存
if (!req.cachedFilters) {
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

// 2. 构建可搜索文本 (CW + 内容 + 投票选项 + 媒体描述)
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

// 3. 应用过滤
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

// 4. 附加过滤结果到响应
transmit(event, {
  ...payload,
  filtered: filter_results
});
```

---

## 4. 实时同步机制

### 4.1 同步架构

```
┌─────────────────┐     Redis Pub/Sub      ┌─────────────────┐
│   Rails 服务    │ ──────────────────────> │  Streaming 服务 │
│  (过滤器变更)   │  channels:              │  (实时推送)     │
│                 │  - timeline:{accountId} │                 │
│                 │  - timeline:system:{id} │                 │
└─────────────────┘                         └────────┬────────┘
                                                      │
                                                      ▼ WebSocket/EventSource
┌─────────────────────────────────────────────────────────────────┐
│                        Web 前端 / PWA                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  接收事件     │───>│  清空 Redux   │───>│  重新获取过滤器   │  │
│  │filters_changed│    │   缓存       │    │  GET /api/v2/filters││
│  └──────────────┘    └──────────────┘    └──────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 服务端缓存失效

**文件**: `app/models/custom_filter.rb:52-120`

```ruby
class CustomFilter < ApplicationRecord
  # 变更前标记
  before_save :prepare_cache_invalidation!
  before_destroy :prepare_cache_invalidation!

  # 提交后失效缓存并推送事件
  after_commit :invalidate_cache!

  def prepare_cache_invalidation!
    @should_invalidate_cache = true
  end

  def invalidate_cache!
    return unless @should_invalidate_cache
    
    @should_invalidate_cache = false
    
    # 1. 清除 Rails 缓存
    Rails.cache.delete("filters:v3:#{account_id}")
    
    # 2. 向 Redis 发布事件
    redis.publish("timeline:#{account_id}", { event: :filters_changed }.to_json)
    redis.publish("timeline:system:#{account_id}", { event: :filters_changed }.to_json)
  end
end
```

### 4.3 级联缓存失效

**文件**: `app/models/concerns/custom_filter_cache.rb`

```ruby
module CustomFilterCache
  extend ActiveSupport::Concern

  included do
    after_commit :invalidate_cache!
    before_destroy :prepare_cache_invalidation!
    before_save :prepare_cache_invalidation!

    # 委托给关联的过滤器
    delegate(
      :invalidate_cache!,
      :prepare_cache_invalidation!,
      to: :custom_filter
    )
  end
end
```

**说明**: 当 `CustomFilterKeyword` 或 `CustomFilterStatus` 变更时，会自动触发所属 `CustomFilter` 的缓存失效。

### 4.4 Streaming 服务事件处理

**文件**: `streaming/index.js:499-517`

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
```

**订阅系统频道** (`streaming/index.js:1281-1308`):

```javascript
const subscribeWebsocketToSystemChannel = ({ websocket, request, subscriptions }) => {
  const accessTokenChannelId = `timeline:access_token:${request.accessTokenId}`;
  const systemChannelId = `timeline:system:${request.accountId}`;
  
  const listener = createSystemMessageListener(request, {
    onKill() {
      websocket.close();
    },
  });

  // 监听两个频道的 filters_changed 事件
  subscribe(accessTokenChannelId, listener);
  subscribe(systemChannelId, listener);
};
```

### 4.5 Web 前端事件监听

**文件**: `app/javascript/mastodon/stream.js:211-220`

```javascript
// 已知事件类型列表 - 包含 filters_changed
const KNOWN_EVENT_TYPES = [
  'update',
  'delete',
  'notification',
  'conversation',
  'filters_changed',  // 过滤器变更事件
  'announcement',
  'announcement.delete',
  'announcement.reaction',
];
```

---

## 5. Web 前端实现

### 5.1 Redux 状态管理

**文件**: `app/javascript/mastodon/reducers/filters.js`

```javascript
import { Map as ImmutableMap, is, fromJS } from 'immutable';
import { FILTERS_FETCH_SUCCESS, FILTERS_CREATE_SUCCESS } from '../actions/filters';
import { FILTERS_IMPORT } from '../actions/importer';

// 规范化单个过滤器
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
  if (is(state.get(filter.id), normalizedFilter)) {
    return state;
  } else {
    return state.update(filter.id, ImmutableMap(), (old) => (
      old.mergeWith(
        ((old_value, new_value) => (new_value === undefined ? old_value : new_value)),
        normalizedFilter
      )
    ));
  }
};

// 批量规范化
const normalizeFilters = (state, filters) => {
  filters.forEach(filter => {
    state = normalizeFilter(state, filter);
  });
  return state;
};

// Reducer
export default function filters(state = ImmutableMap(), action) {
  switch(action.type) {
  case FILTERS_CREATE_SUCCESS:
    return normalizeFilter(state, action.filter);
  case FILTERS_FETCH_SUCCESS:
    return normalizeFilters(ImmutableMap(), action.filters);
  case FILTERS_IMPORT:
    return normalizeFilters(state, action.filters);
  default:
    return state;
  }
}
```

### 5.2 API 调用 Actions

**文件**: `app/javascript/mastodon/actions/filters.js`

```javascript
import api from '../api';

export const FILTERS_FETCH_REQUEST = 'FILTERS_FETCH_REQUEST';
export const FILTERS_FETCH_SUCCESS = 'FILTERS_FETCH_SUCCESS';
export const FILTERS_FETCH_FAIL    = 'FILTERS_FETCH_FAIL';

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
  dispatch({ type: FILTERS_CREATE_REQUEST });
  api().post('/api/v2/filters', params).then(response => {
    dispatch({ type: FILTERS_CREATE_SUCCESS, filter: response.data });
    if (onSuccess) onSuccess(response.data);
  });
};
```

### 5.3 过滤器选择器

**文件**: `app/javascript/mastodon/selectors/filters.ts`

```typescript
import { createSelector } from '@reduxjs/toolkit';
import type { RootState } from 'mastodon/store';
import { toServerSideType } from 'mastodon/utils/filters';

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

  // 从状态中获取服务端预计算的过滤结果
  const filtered = state.statuses.getIn([id, 'filtered']);
  
  // 检查是否有 filter_action 为 'hide' 的过滤结果
  return filtered?.some(
    (result) =>
      filters.getIn([result.get('filter'), 'filter_action']) === 'hide',
  );
};
```

### 5.4 上下文类型映射

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

### 5.5 从时间线数据增量更新过滤器

**文件**: `app/javascript/mastodon/actions/importer/index.js`

```javascript
import { importFilters, FILTERS_IMPORT } from './index';

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
    
    // 导入到 Redux 状态 - 增量更新
    dispatch(importFilters(filters));
  };
}
```

**关键点**:
- Web 前端**不独立执行过滤匹配**
- 依赖服务端返回的 `status.filtered` 数组
- Redux 中存储的过滤器列表主要用于：
  1. 过滤器管理 UI 展示
  2. 配合 `filtered` 数组判断 `filter_action`

---

## 6. PWA 与移动端浏览器支持

> **说明**: Mastodon 主仓库不包含原生 iOS/Android 应用代码，但提供了 PWA (Progressive Web App) 支持，可在移动端浏览器中获得类似原生应用的体验。

### 6.1 Service Worker 实现

**文件**: `app/javascript/mastodon/service_worker/sw.js`

```javascript
import { ExpirationPlugin } from 'workbox-expiration';
import { registerRoute } from 'workbox-routing';
import { CacheFirst } from 'workbox-strategies';
import { handleNotificationClick, handlePush } from './web_push_notifications';

const CACHE_NAME_PREFIX = 'mastodon-';

// 缓存国际化文件 (30天)
registerRoute(
  /intl\/.*\.js$/,
  new CacheFirst({
    cacheName: `${CACHE_NAME_PREFIX}locales`,
    plugins: [
      new ExpirationPlugin({
        maxAgeSeconds: 30 * 24 * 60 * 60,
        maxEntries: 5,
      }),
    ],
  }),
);

// 缓存字体 (30天)
registerRoute(
  ({ request }) => request.destination === 'font',
  new CacheFirst({
    cacheName: `${CACHE_NAME_PREFIX}fonts`,
    plugins: [
      new ExpirationPlugin({
        maxAgeSeconds: 30 * 24 * 60 * 60,
        maxEntries: 5,
      }),
    ],
  }),
);

// 缓存图片 (7天)
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: `m${CACHE_NAME_PREFIX}media`,
    plugins: [
      new ExpirationPlugin({
        maxAgeSeconds: 7 * 24 * 60 * 60,
        maxEntries: 256,
      }),
    ],
  }),
);

// 登出时清除缓存
self.addEventListener('fetch', function(event) {
  const url = new URL(event.request.url);

  if (url.pathname === '/auth/sign_out') {
    const asyncResponse = fetch(event.request);
    const asyncCache = caches.open(`${CACHE_NAME_PREFIX}web`);

    event.respondWith(asyncResponse.then(response => {
      if (response.ok || response.type === 'opaqueredirect') {
        return Promise.all([
          asyncCache.then(cache => cache.delete('/')),
          indexedDB.deleteDatabase('mastodon'),  // 清除 IndexedDB
        ]).then(() => response);
      }
      return response;
    }));
  }
});

// Web Push 通知
self.addEventListener('push', handlePush);
self.addEventListener('notificationclick', handleNotificationClick);
```

### 6.2 Web Push 通知注册

**文件**: `app/javascript/mastodon/actions/push_notifications/registerer.js`

```javascript
import api from '../../api';
import { me } from '../../initial_state';
import { pushNotificationsSetting } from '../../settings';

// 检查浏览器支持
const supportsPushNotifications = (
  'serviceWorker' in navigator && 
  'PushManager' in window && 
  'getKey' in PushSubscription.prototype
);

// 注册推送通知
export function register() {
  return (dispatch, getState) => {
    dispatch(setBrowserSupport(supportsPushNotifications));

    if (supportsPushNotifications) {
      getRegistration()
        .then(getPushSubscription)
        .then(({ registration, subscription }) => {
          if (subscription !== null) {
            // 检查现有订阅是否有效
            // ...
          }
          // 订阅或重新订阅
          return subscribe(registration).then(
            subscription => sendSubscriptionToBackend(subscription));
        })
        .then(subscription => {
          dispatch(setSubscription(subscription));
        })
        .catch(error => {
          dispatch(clearSubscription());
        });
    }
  };
}

// 发送订阅到后端
const sendSubscriptionToBackend = (subscription) => {
  const params = { subscription: { ...subscription.toJSON(), standard: true } };

  if (me) {
    const data = pushNotificationsSetting.get(me);
    if (data) {
      params.data = data;
    }
  }

  return api().post('/api/web/push_subscriptions', params).then(response => response.data);
};
```

### 6.3 localStorage 工具

**文件**: `app/javascript/mastodon/hooks/useStorage.ts`

```typescript
interface StorageOptions {
  type?: 'local' | 'session';
  prefix?: string;
}

export function useStorage({
  type = 'local',
  prefix = '',
}: StorageOptions = {}) {
  const storageType = type === 'local' ? 'localStorage' : 'sessionStorage';
  
  const getItem = useCallback(
    (key: string) => {
      try {
        return window[storageType].getItem(prefix ? `${prefix};${key}` : key);
      } catch {
        return null;
      }
    },
    [storageType, prefix],
  );

  const setItem = useCallback(
    (key: string, value: string) => {
      try {
        window[storageType].setItem(prefix ? `${prefix};${key}` : key, value);
      } catch {}
    },
    [storageType, prefix],
  );

  const removeItem = useCallback(
    (key: string) => {
      try {
        window[storageType].removeItem(prefix ? `${prefix};${key}` : key);
      } catch {}
    },
    [storageType, prefix],
  );

  return { isAvailable, getItem, setItem, removeItem };
}
```

### 6.4 IndexedDB 使用 (Emoji 缓存示例)

**文件**: `app/javascript/mastodon/features/emoji/database.ts`

```typescript
// 使用 idb 库操作 IndexedDB
import { openEmojiDB } from './db-schema';

// 缓存自定义表情
export async function putCustomEmojiData({
  emojis,
  clear = false,
}: {
  emojis: ApiCustomEmojiJSON[];
  clear?: boolean;
}) {
  const db = await loadDB();
  const trx = db.transaction('custom', 'readwrite');

  if (clear) {
    await trx.store.clear();
  }

  await Promise.all(
    emojis.map((emoji) => trx.store.put(transformCustomEmojiData(emoji))),
  );
  await trx.done;
}

// 从缓存读取
export async function loadCacheValue(key: CacheKey) {
  const db = await loadDB();
  const value = await db.get('etags', key);
  return value;
}
```

### 6.5 PWA 中过滤器缓存现状分析

**当前实现的限制**:

| 功能 | 实现状态 | 说明 |
|------|---------|------|
| Redux 内存缓存 | ✅ 已实现 | `filters` reducer |
| localStorage 持久化 | ❌ 未实现 | 刷新页面后丢失 |
| IndexedDB 持久化 | ❌ 未实现 | 仅用于 emoji 缓存 |
| 后台同步 | ❌ 未实现 | 无 Background Sync |
| 离线过滤 | ❌ 不支持 | 依赖服务端 `filtered` 数组 |

**对移动端 PWA 的影响**:
1. 每次刷新页面都需要重新调用 `GET /api/v2/filters`
2. 离线状态下无法访问过滤器列表
3. 离线状态下无法正确应用过滤（依赖服务端计算）

---

## 7. v1 与 v2 兼容实现

### 7.1 版本差异总览

| 特性 | v1 (已废弃) | v2 |
|------|------------|-----|
| 废弃状态 | `deprecate_api '2022-11-14'` | 当前版本 |
| 数据模型 | 单关键词 = 一个 Filter | 过滤器组 + N 关键词 |
| 过滤动作 | `irreversible` (布尔) | `filter_action` (warn/hide/blur) |
| 状态过滤 | 不支持 | 支持 |

### 7.2 v1 API 控制器

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

**文件**: `app/models/custom_filter.rb:62-68`

```ruby
class CustomFilter < ApplicationRecord
  # v1 irreversible 兼容性
  
  # 设置时：true → hide, false → warn
  def irreversible=(value)
    self.action = ActiveModel::Type::Boolean.new.cast(value) ? :hide : :warn
  end

  # 读取时：hide → true, warn → false
  def irreversible?
    hide_action?
  end
end
```

**关键词别名** (`app/models/custom_filter_keyword.rb:24`):

```ruby
alias_attribute :phrase, :keyword  # v1: phrase → v2: keyword
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

**v1 响应** (`GET /api/v1/filters`):

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
    "context": ["home", "notifications"],  // 与上面相同的上下文
    "whole_word": true,
    "expires_at": "2024-12-31T23:59:59Z",
    "irreversible": true
  }
]
```

**v2 响应** (`GET /api/v2/filters`):

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

### 7.6 v1 的局限性

使用 v1 API 存在以下限制：

| 限制 | 说明 |
|------|------|
| 多关键词限制 | 无法通过 v1 API 修改包含多个关键词的过滤器组属性 |
| 无状态过滤 | v1 不支持 `CustomFilterStatus` |
| 无 blur 动作 | 只有 `irreversible` (映射到 hide/warn)，不支持 blur |
| ID 混淆 | v1 返回的是关键词 ID，不是过滤器组 ID |
| 废弃警告 | 响应包含 `Deprecation` HTTP 头 |

---

## 第二部分：第三方客户端开发指南

本部分为第三方客户端开发者提供建议，包括：
- 原生 iOS/Android 应用
- 跨平台应用 (React Native, Flutter)
- 第三方 Web 客户端

---

## 8. API 使用建议

### 8.1 端点选择

**推荐**: 优先使用 v2 API

| 场景 | 推荐端点 | 说明 |
|------|---------|------|
| 新开发客户端 | `GET /api/v2/filters` | 完整功能支持 |
| 维护现有 v1 客户端 | `GET /api/v1/filters` | 兼容层会处理 |
| 检测服务器版本 | `GET /api/v2/instance` | 检查 v2 API 是否可用 |

### 8.2 核心 API 调用流程

```
应用启动时:
1. 检查 API 版本支持
   └── 优先尝试 v2，失败则降级 v1

2. 获取过滤器列表
   └── GET /api/v2/filters
   └── 缓存到本地存储

3. 建立流式连接
   └── WebSocket: /api/v1/streaming/
   └── 订阅: user, user:notification
   └── 监听: filters_changed 事件

运行时:
4. 加载时间线
   └── 检查每个 status.filtered 数组
   └── 根据 filter_action 决定 UI 行为

5. 收到 filters_changed
   └── 清空本地缓存
   └── 重新 GET /api/v2/filters
   └── 重新应用过滤到当前显示的内容
```

### 8.3 TypeScript 类型定义

```typescript
// 核心类型定义

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
  content: string;
  // ... 其他字段
  filtered?: FilterResult[];  // 关键：服务端计算的过滤结果
}
```

### 8.4 v1 类型定义 (兼容用)

```typescript
interface V1Filter {
  id: string;           // 关键词 ID，不是过滤器组 ID
  phrase: string;       // 关键词
  context: string[];
  whole_word: boolean;
  expires_at: string | null;
  irreversible: boolean; // true=hide, false=warn
}
```

---

## 9. 缓存策略设计

### 9.1 推荐缓存层级

```
┌─────────────────────────────────────────────────────────────┐
│                      缓存层级架构                              │
├─────────────────────────────────────────────────────────────┤
│  L1: 内存缓存                                                 │
│  ├── Redux / MobX / ViewModel 状态                          │
│  └── 用于 UI 实时访问和过滤决策                              │
├─────────────────────────────────────────────────────────────┤
│  L2: 持久化存储                                               │
│  ├── iOS: CoreData / Realm / UserDefaults                   │
│  ├── Android: Room / SQLite / SharedPreferences             │
│  ├── React Native: AsyncStorage / Realm / WatermelonDB      │
│  └── Web: IndexedDB / localStorage                          │
├─────────────────────────────────────────────────────────────┤
│  L3: 服务端                                                   │
│  └── 始终作为数据真实来源                                     │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 缓存数据结构

**CoreData / Room 实体设计**:

```swift
// Swift (CoreData)
@objc(FilterEntity)
class FilterEntity: NSManagedObject {
    @NSManaged var id: String
    @NSManaged var title: String
    @NSManaged var context: [String]  // 存储为 JSON 或 Transformable
    @NSManaged var expiresAt: Date?
    @NSManaged var filterAction: String  // "warn", "hide", "blur"
    @NSManaged var keywords: Set<KeywordEntity>
    @NSManaged var lastFetchedAt: Date  // 缓存时间戳
}

@objc(KeywordEntity)
class KeywordEntity: NSManagedObject {
    @NSManaged var id: String
    @NSManaged var keyword: String
    @NSManaged var wholeWord: Bool
    @NSManaged var filter: FilterEntity
}
```

```kotlin
// Kotlin (Room)
@Entity(tableName = "filters")
data class FilterEntity(
    @PrimaryKey val id: String,
    val title: String,
    val context: String,  // JSON 数组字符串
    val expiresAt: Long?,  // 时间戳
    val filterAction: String,
    val lastFetchedAt: Long
)

@Entity(
    tableName = "filter_keywords",
    foreignKeys = [ForeignKey(
        entity = FilterEntity::class,
        parentColumns = ["id"],
        childColumns = ["filterId"],
        onDelete = CASCADE
    )]
)
data class FilterKeywordEntity(
    @PrimaryKey val id: String,
    val filterId: String,
    val keyword: String,
    val wholeWord: Boolean
)
```

### 9.3 缓存更新策略

| 触发时机 | 操作 |
|---------|------|
| 应用启动 | 调用 `GET /api/v2/filters` 并更新缓存 |
| 收到 `filters_changed` 事件 | 清空缓存，重新获取 |
| 用户修改过滤器 | 立即同步到服务端，成功后更新缓存 |
| 缓存过期 (如 24 小时) | 后台刷新 |

### 9.4 缓存过期处理

```typescript
// 示例：缓存有效性检查
function isCacheValid(lastFetchedAt: Date, maxAge: number = 24 * 60 * 60 * 1000): boolean {
  const now = new Date();
  return (now.getTime() - lastFetchedAt.getTime()) < maxAge;
}

// 过滤器过期检查
function isFilterExpired(filter: Filter): boolean {
  if (!filter.expires_at) return false;
  return new Date(filter.expires_at) < new Date();
}
```

---

## 10. 实时同步实现

### 10.1 WebSocket 连接管理

**事件监听**:

```typescript
interface StreamEvent {
  stream: string[];
  event: string;
  payload?: string;
}

class FilterSyncManager {
  private filters: Map<string, Filter> = new Map();
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;

  // 连接流式 API
  connect(accessToken: string, streamingUrl: string) {
    const url = `${streamingUrl}/api/v1/streaming/?access_token=${accessToken}&stream=user`;
    this.ws = new WebSocket(url);

    this.ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.handleStreamEvent(data);
    };

    this.ws.onclose = () => {
      this.handleDisconnect();
    };
  }

  // 处理流式事件
  private handleStreamEvent(data: StreamEvent) {
    switch (data.event) {
      case 'filters_changed':
        this.handleFiltersChanged();
        break;
      case 'update':
        // 新状态，检查 data.payload.filtered
        this.handleStatusUpdate(JSON.parse(data.payload!));
        break;
      // ... 其他事件
    }
  }

  // 处理过滤器变更
  private async handleFiltersChanged() {
    console.log('Filters changed, refreshing...');
    
    // 1. 通知 UI 显示加载状态
    this.onFiltersUpdating?.();
    
    // 2. 重新获取过滤器
    try {
      const filters = await this.fetchFilters();
      this.filters = new Map(filters.map(f => [f.id, f]));
      
      // 3. 持久化到本地存储
      await this.persistFilters(filters);
      
      // 4. 通知 UI 更新
      this.onFiltersChanged?.(filters);
      
      // 5. 重新应用过滤到当前时间线
      this.reapplyFiltersToCurrentTimeline();
    } catch (error) {
      console.error('Failed to refresh filters:', error);
    }
  }
}
```

### 10.2 重连策略

```typescript
private handleDisconnect() {
  if (this.reconnectAttempts < this.maxReconnectAttempts) {
    const delay = Math.pow(2, this.reconnectAttempts) * 1000;  // 指数退避
    this.reconnectAttempts++;
    
    setTimeout(() => {
      this.connect(this.accessToken!, this.streamingUrl!);
    }, delay);
  }
}
```

### 10.3 多设备同步场景

```
场景：用户在设备 A 上修改过滤器

1. 设备 A:
   └── PATCH /api/v2/filters/:id
   └── 成功后更新本地缓存

2. 服务端:
   └── 更新数据库
   └── 失效 Rails.cache("filters:v3:{account_id}")
   └── Redis.publish("timeline:{account_id}", {event: "filters_changed"})
   └── Redis.publish("timeline:system:{account_id}", {event: "filters_changed"})

3. Streaming 服务:
   └── 收到 filters_changed 事件
   └── 清空 req.cachedFilters
   └── 向所有已连接的 WebSocket 客户端推送 filters_changed

4. 设备 B (已连接):
   └── 收到 WebSocket 消息: {event: "filters_changed"}
   └── 清空本地缓存
   └── GET /api/v2/filters
   └── 更新 UI

5. 设备 C (后台/未连接):
   └── 下次启动或恢复连接时
   └── GET /api/v2/filters 获取最新状态
```

---

## 11. 过滤行为处理

### 11.1 核心原则

**重要**: 客户端**不应该**独立执行过滤匹配逻辑。

| 职责 | 服务端 | 客户端 |
|------|--------|--------|
| 计算 `status.filtered` 数组 | ✅ 负责 | ❌ 不负责 |
| 决定过滤匹配 | ✅ 负责 | ❌ 不负责 |
| 解释 `filter_action` | ❌ 不负责 | ✅ 负责 |
| UI 表现 (隐藏/警告/模糊) | ❌ 不负责 | ✅ 负责 |

### 11.2 上下文映射

客户端需要将 UI 上下文映射到服务端上下文:

```typescript
type ClientContext = 
  | 'home' 
  | 'notifications' 
  | 'public' 
  | 'thread' 
  | 'account'
  | 'detailed'      // 详情页
  | 'bookmarks'     // 书签
  | 'favourites'    // 收藏
  | `list:${string}` // 列表
  | string;          // 其他

function toServerSideContext(clientContext: ClientContext): FilterContext {
  switch (clientContext) {
    case 'home':
    case 'notifications':
    case 'public':
    case 'thread':
    case 'account':
      return clientContext as FilterContext;
    case 'detailed':
      return 'thread';
    case 'bookmarks':
    case 'favourites':
      return 'home';
    default:
      if (clientContext.startsWith('list:')) {
        return 'home';
      }
      return 'public';
  }
}
```

### 11.3 过滤行为判断

```typescript
interface FilterBehavior {
  shouldHide: boolean;      // 完全隐藏
  shouldWarn: boolean;      // 显示警告
  shouldBlur: boolean;      // 模糊显示
  activeFilters: FilterResult[];
}

function determineFilterBehavior(
  status: Status,
  filtersCache: Map<string, Filter>,
  context: ClientContext
): FilterBehavior {
  const result: FilterBehavior = {
    shouldHide: false,
    shouldWarn: false,
    shouldBlur: false,
    activeFilters: [],
  };

  // 1. 没有过滤结果，直接返回
  if (!status.filtered || status.filtered.length === 0) {
    return result;
  }

  const now = new Date();
  const serverContext = toServerSideContext(context);

  // 2. 检查每个过滤结果
  for (const filterResult of status.filtered) {
    const filter = filtersCache.get(filterResult.filter.id);
    
    // 3. 验证过滤器在当前上下文中是否有效
    if (!filter) continue;
    if (!filter.context.includes(serverContext)) continue;
    if (filter.expires_at && new Date(filter.expires_at) < now) continue;

    // 4. 收集有效过滤结果
    result.activeFilters.push(filterResult);

    // 5. 确定行为 (hide 优先级最高)
    switch (filter.filter_action) {
      case 'hide':
        result.shouldHide = true;
        break;
      case 'warn':
        if (!result.shouldHide) result.shouldWarn = true;
        break;
      case 'blur':
        if (!result.shouldHide && !result.shouldWarn) result.shouldBlur = true;
        break;
    }
  }

  // 6. 互斥处理
  if (result.shouldHide) {
    result.shouldWarn = false;
    result.shouldBlur = false;
  }

  return result;
}
```

### 11.4 UI 表现建议

| filter_action | 推荐 UI 行为 |
|---------------|-------------|
| `hide` | 从时间线中完全移除状态，不显示任何痕迹 |
| `warn` | 显示警告横幅，用户可点击展开查看内容 |
| `blur` | 模糊显示媒体内容，文字可能显示警告或模糊 |

**iOS/SwiftUI 示例**:

```swift
struct StatusRow: View {
    let status: Status
    let filters: [Filter]
    let context: ClientContext
    
    var body: some View {
        let behavior = determineFilterBehavior(status: status, filters: filters, context: context)
        
        Group {
            if behavior.shouldHide {
                // 不渲染任何内容
                EmptyView()
            } else if behavior.shouldWarn {
                // 警告横幅
                VStack(alignment: .leading) {
                    HStack {
                        Image(systemName: "exclamationmark.triangle")
                            .foregroundColor(.orange)
                        Text("包含过滤内容")
                            .font(.subheadline)
                            .foregroundColor(.secondary)
                    }
                    .onTapGesture {
                        // 展开显示内容
                    }
                    
                    // 可选：模糊显示预览
                    StatusContent(status: status)
                        .blur(radius: 8)
                }
            } else if behavior.shouldBlur {
                // 模糊显示
                StatusContent(status: status)
                    .blur(radius: 10)
                    .onTapGesture {
                        // 点击取消模糊
                    }
            } else {
                // 正常显示
                StatusContent(status: status)
            }
        }
    }
}
```

---

## 12. 移动端特定考虑

### 12.1 后台刷新

**iOS**: 使用 Background App Refresh

```swift
// AppDelegate 或 SceneDelegate
func application(_ application: UIApplication, 
                 didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    
    // 注册后台刷新
    application.setMinimumBackgroundFetchInterval(UIApplication.backgroundFetchIntervalMinimum)
    
    return true
}

func application(_ application: UIApplication, 
                 performFetchWithCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void) {
    
    // 后台刷新过滤器
    filterManager.refreshFilters { result in
        switch result {
        case .newData:
            completionHandler(.newData)
        case .noData:
            completionHandler(.noData)
        case .failed:
            completionHandler(.failed)
        }
    }
}
```

**Android**: 使用 WorkManager

```kotlin
// 定义刷新任务
class FilterRefreshWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            filterRepository.refreshFilters()
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }
    
    companion object {
        fun schedulePeriodic(context: Context) {
            val request = PeriodicWorkRequestBuilder<FilterRefreshWorker>(
                repeatInterval = 12,
                repeatIntervalTimeUnit = TimeUnit.HOURS
            ).build()
            
            WorkManager.getInstance(context)
                .enqueueUniquePeriodicWork(
                    "FilterRefresh",
                    ExistingPeriodicWorkPolicy.UPDATE,
                    request
                )
        }
    }
}
```

### 12.2 推送通知过滤

**重要**: 推送通知在到达客户端之前，服务端已应用过滤。

但客户端仍需处理：
1. 通知点击时的过滤检查
2. 本地通知的过滤

```typescript
// 处理通知点击
function handleNotificationClick(notification: PushNotification) {
  const statusId = notification.data.statusId;
  
  // 导航到状态前，检查过滤状态
  fetchStatus(statusId).then(status => {
    if (status.filtered) {
      const behavior = determineFilterBehavior(
        status, 
        filtersCache, 
        'notifications'
      );
      
      if (behavior.shouldHide) {
        // 显示提示：该状态已被过滤
        showToast('此状态已被您的过滤器隐藏');
        return;
      }
    }
    
    // 正常导航
    navigateToStatus(statusId);
  });
}
```

### 12.3 网络优化

**移动端网络条件较差，建议**:

1. **使用 ETag 缓存**:
```typescript
// 请求时发送 If-None-Match
async function fetchFilters(etag?: string): Promise<Filter[]> {
  const headers: Record<string, string> = {};
  if (etag) {
    headers['If-None-Match'] = etag;
  }
  
  const response = await api.get('/api/v2/filters', { headers });
  
  if (response.status === 304) {
    // 缓存未修改，使用本地缓存
    return getCachedFilters();
  }
  
  // 保存新的 ETag
  saveETag(response.headers.etag);
  return response.data;
}
```

2. **批量请求**: 合并相关 API 调用
3. **响应缓存**: 使用 HTTP 缓存或本地持久化

### 12.4 低内存处理

**移动端内存有限，建议**:

1. **过滤器数据保持轻量**: 只缓存必要字段
2. **收到内存警告时清理**:
```swift
// iOS
func applicationDidReceiveMemoryWarning(_ application: UIApplication) {
    // 清理内存缓存，但保留持久化存储
    filterManager.clearMemoryCache()
}
```

```kotlin
// Android
override fun onTrimMemory(level: Int) {
    super.onTrimMemory(level)
    if (level >= TRIM_MEMORY_RUNNING_LOW) {
        filterManager.clearMemoryCache()
    }
}
```

---

## 13. v1/v2 兼容策略

### 13.1 版本检测

```typescript
async function detectAPIVersion(): Promise<'v2' | 'v1'> {
  try {
    // 尝试 v2 API
    const response = await api.get('/api/v2/filters');
    return 'v2';
  } catch (error) {
    // v2 失败，尝试 v1
    try {
      await api.get('/api/v1/filters');
      return 'v1';
    } catch {
      throw new Error('Filter API not available');
    }
  }
}
```

### 13.2 v1 到 v2 数据转换

```typescript
function convertV1ToV2(v1Filters: V1Filter[]): Filter[] {
  // v1 每个关键词都是独立的"过滤器"
  // 转换策略：将相同 context 和 irreversible 的关键词合并
  
  const groups = new Map<string, {
    context: string[];
    filterAction: 'warn' | 'hide';
    keywords: { keyword: string; wholeWord: boolean }[];
  }>();

  v1Filters.forEach(v1 => {
    const key = JSON.stringify({
      context: v1.context.sort(),
      irreversible: v1.irreversible
    });

    if (!groups.has(key)) {
      groups.set(key, {
        context: v1.context,
        filterAction: v1.irreversible ? 'hide' : 'warn',
        keywords: []
      });
    }

    groups.get(key)!.keywords.push({
      keyword: v1.phrase,
      wholeWord: v1.whole_word
    });
  });

  return Array.from(groups.entries()).map(([key, group], index) => ({
    id: `v1-converted-${index}`,
    title: group.keywords[0]?.keyword || 'Imported Filter',
    context: group.context as FilterContext[],
    expires_at: v1Filters.find(f => 
      JSON.stringify({ context: f.context.sort(), irreversible: f.irreversible }) === key
    )?.expires_at || null,
    filter_action: group.filterAction,
    keywords: group.keywords.map((kw, i) => ({
      id: `v1-kw-${index}-${i}`,
      keyword: kw.keyword,
      whole_word: kw.wholeWord
    })),
    statuses: []
  }));
}
```

### 13.3 统一接口封装

```typescript
interface FilterAPI {
  getFilters(): Promise<Filter[]>;
  createFilter(params: CreateFilterParams): Promise<Filter>;
  updateFilter(id: string, params: UpdateFilterParams): Promise<Filter>;
  deleteFilter(id: string): Promise<void>;
}

// V2 实现
class V2FilterAPI implements FilterAPI {
  async getFilters(): Promise<Filter[]> {
    const response = await api.get('/api/v2/filters');
    return response.data;
  }
  // ... 其他方法
}

// V1 兼容实现
class V1FilterAPI implements FilterAPI {
  async getFilters(): Promise<Filter[]> {
    const response = await api.get('/api/v1/filters');
    return convertV1ToV2(response.data);
  }

  async createFilter(params: CreateFilterParams): Promise<Filter> {
    // v1 只能创建单关键词过滤器
    if (params.keywords.length > 1) {
      throw new Error('V1 API does not support multiple keywords');
    }

    const v1Params = {
      phrase: params.keywords[0].keyword,
      whole_word: params.keywords[0].whole_word,
      context: params.context,
      irreversible: params.filter_action === 'hide',
      expires_in: params.expires_in
    };

    const response = await api.post('/api/v1/filters', v1Params);
    return convertV1ToV2([response.data])[0];
  }
  // ... 其他方法
}

// 工厂函数
function createFilterAPI(version: 'v1' | 'v2'): FilterAPI {
  return version === 'v2' ? new V2FilterAPI() : new V1FilterAPI();
}
```

---

## 附录

### A. 关键文件索引

| 组件 | 文件路径 |
|------|----------|
| V2 API 控制器 | `app/controllers/api/v2/filters_controller.rb` |
| V1 API 控制器 | `app/controllers/api/v1/filters_controller.rb` |
| 过滤器模型 | `app/models/custom_filter.rb` |
| 关键词模型 | `app/models/custom_filter_keyword.rb` |
| 状态过滤模型 | `app/models/custom_filter_status.rb` |
| V2 序列化器 | `app/serializers/rest/filter_serializer.rb` |
| V1 序列化器 | `app/serializers/rest/v1/filter_serializer.rb` |
| 前端过滤器 Actions | `app/javascript/mastodon/actions/filters.js` |
| 前端过滤器 Reducer | `app/javascript/mastodon/reducers/filters.js` |
| 前端过滤器选择器 | `app/javascript/mastodon/selectors/filters.ts` |
| Service Worker | `app/javascript/mastodon/service_worker/sw.js` |
| 推送通知注册 | `app/javascript/mastodon/actions/push_notifications/registerer.js` |
| Streaming 服务 | `streaming/index.js` |

### B. API 端点速查

**V2 过滤器管理**:
- `GET /api/v2/filters` - 列出所有过滤器
- `GET /api/v2/filters/:id` - 获取单个过滤器
- `POST /api/v2/filters` - 创建过滤器
- `PUT /api/v2/filters/:id` - 更新过滤器
- `DELETE /api/v2/filters/:id` - 删除过滤器

**V2 关键词管理**:
- `POST /api/v2/filters/:filter_id/keywords` - 添加关键词
- `PUT /api/v2/filters/:filter_id/keywords/:id` - 更新关键词
- `DELETE /api/v2/filters/:filter