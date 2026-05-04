# 帖子过滤器和内容警告实现分析

## 一、整体架构

Mastodon 的帖子隐藏和折叠功能由两个独立但互补的系统实现：

1. **帖子过滤器系统**：基于用户定义的关键词/规则自动过滤帖子
2. **内容警告系统**：基于帖子自带的 `spoiler_text` 字段实现折叠显示

---

## 二、帖子过滤器系统

### 2.1 数据模型

过滤器的数据结构在 `app/javascript/mastodon/api_types/statuses.ts` 中定义：

```typescript
filter_action: 'warn' | 'hide';
```

过滤器支持两种动作类型：
- `hide`：完全隐藏帖子
- `warn`：显示警告横幅，用户可选择显示
- `blur`：模糊媒体内容（在选择器逻辑中处理）

### 2.2 Redux 状态管理

#### Actions (`app/javascript/mastodon/actions/filters.js`)

主要动作类型：
- `FILTERS_FETCH_SUCCESS`：从 API 获取过滤器列表
- `FILTERS_CREATE_SUCCESS`：创建新过滤器
- `FILTERS_IMPORT`：导入过滤器

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

### 2.3 选择器逻辑

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

## 三、内容警告系统

### 3.1 组件结构

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

## 四、前端隐藏和折叠触发逻辑

### 4.1 帖子组件核心逻辑 (`app/javascript/mastodon/components/status.jsx`)

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

### 4.2 状态容器 (`app/javascript/mastodon/containers/status_container.jsx`)

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

### 4.3 通知中的过滤逻辑 (`app/javascript/mastodon/features/notifications_v2/components/notification_with_status.tsx`)

```tsx
const isFiltered = useAppSelector(
  (state) =>
    statusId &&
    getStatusHidden(state, { id: statusId, contextType: 'notifications' }),
);

// 如果被过滤，直接返回 null
if (!statusId || isFiltered) return null;
```

### 4.4 长内容折叠逻辑 (`app/javascript/mastodon/components/status_content.jsx`)

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

## 五、完整流程图

```
帖子加载
    │
    ├───► 服务器端过滤
    │        │
    │        └───► API 返回 status.filtered 数组
    │                    包含匹配的过滤器 ID 和 filter_action
    │
    └───► 客户端处理
             │
             ├───► 1. makeGetStatus 选择器处理
             │         │
             │         ├───► 检查 filter_action === 'hide'
             │         │         │
             │         │         ├───► warnInsteadOfHide = false（时间线）
             │         │         │         └───► 返回 status: null, loadingState: 'filtered'
             │         │         │
             │         │         └───► warnInsteadOfHide = true（详情页/收藏/搜索）
             │         │                   └───► 继续处理，不隐藏
             │         │
             │         ├───► 检查 filter_action === 'blur'
             │         │         └───► 设置 matched_media_filters
             │         │
             │         └───► 检查 filter_action === 'warn'
             │                   └───► 设置 matched_filters
             │
             └───► 2. Status 组件渲染
                       │
                       ├───► 检查 matched_filters
                       │         │
                       │         └───► 渲染 FilterWarning 组件
                       │                   点击 → 切换 showDespiteFilter 状态
                       │
                       ├───► 检查 spoiler_text
                       │         │
                       │         └───► 渲染 ContentWarning 组件
                       │                   点击 → dispatch(toggleStatusSpoilers)
                       │
                       └───► 计算 expanded 状态
                                 │
                                 └───► expanded = (!matched_filters || showDespiteFilter) 
                                              && (!status.hidden || !spoiler_text)
                                          │
                                          ├───► true → 渲染 StatusContent + 媒体
                                          │
                                          └───► false → 只显示警告横幅
```

---

## 六、关键文件索引

| 功能 | 文件路径 |
|------|----------|
| 过滤器 Reducer | `app/javascript/mastodon/reducers/filters.js` |
| 过滤器 Actions | `app/javascript/mastodon/actions/filters.js` |
| 过滤器选择器 | `app/javascript/mastodon/selectors/filters.ts` |
| 主选择器 | `app/javascript/mastodon/selectors/index.js` |
| 上下文映射 | `app/javascript/mastodon/utils/filters.ts` |
| 内容警告组件 | `app/javascript/mastodon/components/content_warning.tsx` |
| 状态横幅组件 | `app/javascript/mastodon/components/status_banner.tsx` |
| 帖子主组件 | `app/javascript/mastodon/components/status.jsx` |
| 帖子内容组件 | `app/javascript/mastodon/components/status_content.jsx` |
| 帖子容器 | `app/javascript/mastodon/containers/status_container.jsx` |
| API 类型定义 | `app/javascript/mastodon/api_types/statuses.ts` |

---

## 七、总结

### 7.1 两种系统的区别

| 特性 | 帖子过滤器 | 内容警告 |
|------|-----------|---------|
| 触发源 | 用户定义的规则/关键词 | 帖子作者设置的 `spoiler_text` |
| 数据来源 | `status.filtered` 数组 + 本地过滤器状态 | `status.spoiler_text` |
| 动作类型 | `hide` / `warn` / `blur` | 单一折叠/展开 |
| 上下文感知 | 是（不同列应用不同过滤器） | 否 |
| 可被覆盖 | 是（`showDespiteFilter`） | 是（`toggleStatusSpoilers`） |

### 7.2 隐藏 vs 折叠的决策逻辑

**完全隐藏**（`filter_action = 'hide'` + `warnInsteadOfHide = false`）：
- 时间线中匹配 hide 过滤器的帖子
- 通知中匹配 hide 过滤器的帖子
- 结果：`status = null`，不渲染任何内容

**折叠显示**：
- 有 `spoiler_text` 的帖子 → 显示内容警告
- 匹配 `warn` 过滤器的帖子 → 显示过滤器警告
- 匹配 `hide` 过滤器但在详情页/收藏/搜索 → 显示警告
- 结果：显示警告横幅，用户可点击展开

**长内容自动折叠**：
- 高度超过 706px 且没有 `spoiler_text`
- 结果：显示 "Read more" 按钮
