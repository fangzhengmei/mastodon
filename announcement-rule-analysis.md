# 公告与规则协作机制分析

## 概述

本文档分析 Mastodon 系统中管理员发布公告（Announcement）和修改实例规则（Rule）后，API、前端 Banner、用户已读状态和审计日志的协作机制。

---

## 一、公告（Announcement）协作机制

### 1.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              管理员操作层                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │   创建公告   │  │   编辑公告   │  │   发布公告   │  │  取消发布    │       │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘       │
│         │                  │                  │                  │              │
│         └──────────────────┼──────────────────┼──────────────────┘              │
│                            ▼                  ▼                                 │
│              ┌─────────────────────────────────────────────┐                   │
│              │    Admin::AnnouncementsController            │                   │
│              │  (app/controllers/admin/announcements.rb)   │                   │
│              └──────────────────────┬──────────────────────┘                   │
│                                     │                                           │
│         ┌───────────────────────────┼───────────────────────────┐             │
│         ▼                           ▼                           ▼             │
│  ┌──────────────┐          ┌──────────────┐          ┌──────────────┐       │
│  │  数据持久化   │          │  异步 Worker  │          │  审计日志    │       │
│  │ Announcement │          │ Sidekiq Jobs │          │ log_action   │       │
│  └──────────────┘          └──────────────┘          └──────────────┘       │
└─────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              数据层                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │   Announcement  │  │AnnouncementMute │  │AnnouncementReaction│            │
│  │   (公告主表)     │  │  (用户已读状态)  │  │   (用户反应)      │              │
│  └────────┬────────┘  └────────┬────────┘  └─────────────────┘              │
│           │                     │                                              │
│           ▼                     ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                         PostgreSQL 数据库                                  │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API 层                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │           Api::V1::AnnouncementsController                                │  │
│  │           (app/controllers/api/v1/announcements_controller.rb)          │  │
│  └───────────────────────────┬─────────────────────────────────────────────┘  │
│                              │                                                  │
│          ┌───────────────────┼───────────────────┐                           │
│          ▼                   ▼                   ▼                           │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐                    │
│  │ GET /api/v1/ │   │ POST /api/v1/│   │  反应相关    │                    │
│  │announcements │   │announcements/ │   │ PUT/DELETE  │                    │
│  │   (获取列表)  │   │:id/dismiss   │   │  reactions   │                    │
│  └──────────────┘   │  (标记已读)   │   └──────────────┘                    │
│                     └──────────────┘                                          │
└─────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              前端展示层                                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                        Redux 状态管理                                      │  │
│  │  ┌──────────────────┐  ┌──────────────────┐                              │  │
│  │  │ actions/         │  │ reducers/         │                              │  │
│  │  │ announcements.js │  │ announcements.js  │                              │  │
│  │  └────────┬─────────┘  └────────┬─────────┘                              │  │
│  │           │                       │                                        │  │
│  │           ▼                       ▼                                        │  │
│  │  ┌──────────────────────────────────────────┐                            │  │
│  │  │         announcements State               │                            │  │
│  │  │  { items: [], isLoading: bool, show: bool}│                            │  │
│  │  └─────────────────────┬────────────────────┘                            │  │
│  │                        │                                                  │  │
│  │                        ▼                                                  │  │
│  │  ┌──────────────────────────────────────────┐                            │  │
│  │  │    HomeTimeline 组件 (首页时间线)         │                            │  │
│  │  │  ┌────────────────────────────────────┐  │                            │  │
│  │  │  │  ColumnHeader (带未读计数的按钮)     │  │                            │  │
│  │  │  │  ┌──────────────────────────────┐  │  │                            │  │
│  │  │  │  │ IconWithBadge (喇叭图标+角标)  │  │  │                            │  │
│  │  │  │  └──────────────────────────────┘  │  │                            │  │
│  │  │  └────────────────────────────────────┘  │                            │  │
│  │  │  ┌────────────────────────────────────┐  │                            │  │
│  │  │  │  Announcements (Banner 轮播组件)    │  │                            │  │
│  │  │  │  ┌──────────────────────────────┐  │  │                            │  │
│  │  │  │  │  Carousel (轮播容器)          │  │  │                            │  │
│  │  │  │  │  ┌────────────────────────┐  │  │  │                            │  │
│  │  │  │  │  │ Announcement (单条公告) │  │  │  │                            │  │
│  │  │  │  │  │ - 内容展示              │  │  │  │                            │  │
│  │  │  │  │  │ - 时间戳                │  │  │  │                            │  │
│  │  │  │  │  │ - ReactionsBar (反应栏) │  │  │  │                            │  │
│  │  │  │  │  │ - 未读指示器            │  │  │  │                            │  │
│  │  │  │  │  └────────────────────────┘  │  │  │                            │  │
│  │  │  │  └──────────────────────────────┘  │  │                            │  │
│  │  │  └────────────────────────────────────┘  │                            │  │
│  │  └──────────────────────────────────────────┘                            │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 管理员发布公告流程

#### 1.2.1 后台管理控制器

**文件**: `app/controllers/admin/announcements_controller.rb`

管理员操作公告的核心入口，支持以下操作：

| 操作 | HTTP 方法 | 路由 | 说明 |
|------|----------|------|------|
| 创建 | POST | `/admin/announcements` | 创建新公告，支持立即发布或定时发布 |
| 编辑 | PATCH/PUT | `/admin/announcements/:id` | 修改公告内容 |
| 发布 | POST | `/admin/announcements/:id/publish` | 将草稿状态公告标记为已发布 |
| 取消发布 | POST | `/admin/announcements/:id/unpublish` | 撤回已发布的公告 |
| 删除 | DELETE | `/admin/announcements/:id` | 删除公告 |

**关键代码逻辑**：

```ruby
# 创建公告
def create
  @announcement = Announcement.new(resource_params)
  
  if @announcement.save
    # 如果已发布，触发异步 Worker
    PublishScheduledAnnouncementWorker.perform_async(@announcement.id) if @announcement.published?
    # 记录审计日志
    log_action :create, @announcement
    redirect_to admin_announcements_path
  end
end

# 发布公告
def publish
  @announcement.publish!  # 更新 published: true, published_at: Time.now
  PublishScheduledAnnouncementWorker.perform_async(@announcement.id)
  log_action :update, @announcement
end

# 取消发布
def unpublish
  @announcement.unpublish!
  UnpublishAnnouncementWorker.perform_async(@announcement.id)
  log_action :update, @announcement
end
```

#### 1.2.2 数据模型

**文件**: `app/models/announcement.rb`

公告核心属性：

| 字段 | 类型 | 说明 |
|------|------|------|
| `text` | text | 公告内容（必填） |
| `published` | boolean | 是否已发布（默认 false） |
| `published_at` | datetime | 发布时间 |
| `scheduled_at` | datetime | 定时发布时间 |
| `starts_at` | datetime | 公告开始时间 |
| `ends_at` | datetime | 公告结束时间 |
| `all_day` | boolean | 是否全天事件 |
| `status_ids` | bigint[] | 关联的嘟文 ID 数组 |
| `notification_sent_at` | datetime | 通知发送时间 |

**关键方法**：

```ruby
# 发布公告
def publish!
  update!(published: true, published_at: Time.now.utc, scheduled_at: nil)
end

# 取消发布
def unpublish!
  update!(published: false, scheduled_at: nil)
end

# 检查是否已发送通知
def notification_sent?
  notification_sent_at.present?
end
```

#### 1.2.3 异步推送机制

公告发布后通过 Sidekiq Worker 进行实时推送：

**PublishScheduledAnnouncementWorker** (`app/workers/publish_scheduled_announcement_worker.rb`)

```ruby
def perform(announcement_id)
  @announcement = Announcement.find(announcement_id)
  
  # 解析公告内容中的嘟文引用
  refresh_status_ids!
  
  # 确保公告已发布
  @announcement.publish! unless @announcement.published?
  
  # 序列化公告数据
  payload = InlineRenderer.render(@announcement, nil, :announcement)
  payload = { event: :announcement, payload: payload }.to_json
  
  # 向所有活跃用户的 Redis 频道推送
  FeedManager.instance.with_active_accounts do |account|
    redis.publish("timeline:#{account.id}", payload) if redis.exists?("subscribed:timeline:#{account.id}")
  end
end
```

**UnpublishAnnouncementWorker** (`app/workers/unpublish_announcement_worker.rb`)

```ruby
def perform(announcement_id)
  # 推送删除事件
  payload = { event: :'announcement.delete', payload: announcement_id.to_s }.to_json
  
  FeedManager.instance.with_active_accounts do |account|
    redis.publish("timeline:#{account.id}", payload) if redis.exists?("subscribed:timeline:#{account.id}")
  end
end
```

**推送机制说明**：
- 使用 Redis Pub/Sub 进行实时推送
- 只推送给当前在线（订阅了 timeline 频道）的用户
- 推送事件类型：
  - `:announcement` - 新公告或更新公告
  - `:'announcement.delete'` - 公告被删除/取消发布

### 1.3 API 层设计

#### 1.3.1 API 控制器

**文件**: `app/controllers/api/v1/announcements_controller.rb`

| 端点 | HTTP 方法 | 权限 | 说明 |
|------|----------|------|------|
| `/api/v1/announcements` | GET | **需登录** (`require_user!`) | 获取已发布公告列表（含个性化已读状态） |
| `/api/v1/announcements/:id/dismiss` | POST | 需登录 + `write:accounts` | 标记公告为已读 |

**权限说明**：
- 公告 API 所有端点都需要登录（`before_action :require_user!`）
- `dismiss` 端点还需要额外的 OAuth scope: `write:accounts`
- 未登录访问会返回 422 错误：`"This method requires an authenticated user"`

**关键实现**：

```ruby
# 获取公告列表
def index
  # 只返回已发布的公告，按时间排序
  @announcements = Announcement.published.chronological
  render json: @announcements, each_serializer: REST::AnnouncementSerializer
end

# 标记已读
def dismiss
  # 通过创建 AnnouncementMute 记录表示已读
  AnnouncementMute.find_or_create_by!(account: current_account, announcement: @announcement)
  render_empty
end
```

#### 1.3.2 序列化器

**文件**: `app/serializers/rest/announcement_serializer.rb`

API 响应包含的字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 公告 ID |
| `content` | string | 处理后的 HTML 内容（链接自动转换） |
| `starts_at` | datetime | 开始时间 |
| `ends_at` | datetime | 结束时间 |
| `all_day` | boolean | 是否全天 |
| `published_at` | datetime | 发布时间 |
| `updated_at` | datetime | 更新时间 |
| `read` | boolean | **当前用户是否已读**（仅登录用户可见） |
| `mentions` | array | @提及的用户 |
| `statuses` | array | 关联的嘟文 |
| `tags` | array | 话题标签 |
| `emojis` | array | 自定义表情 |
| `reactions` | array | 用户反应 |

**已读状态判断逻辑**：

```ruby
def read
  # 检查当前用户是否有 AnnouncementMute 记录
  object.announcement_mutes.exists?(account: current_user.account)
end
```

### 1.4 用户已读状态管理

#### 1.4.1 数据模型

**文件**: `app/models/announcement_mute.rb`

```ruby
class AnnouncementMute < ApplicationRecord
  belongs_to :account
  belongs_to :announcement, inverse_of: :announcement_mutes
  
  # 确保每个用户对每条公告只有一条已读记录
  validates :account_id, uniqueness: { scope: :announcement_id }
end
```

**数据库表结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint | 主键 |
| `account_id` | bigint | 用户账户 ID（外键） |
| `announcement_id` | bigint | 公告 ID（外键） |
| `created_at` | datetime | 创建时间 |
| `updated_at` | datetime | 更新时间 |

#### 1.4.2 已读标记流程

```
用户查看公告
     │
     ▼
┌─────────────────────────────────────┐
│  前端 Announcement 组件检测          │
│  active=true && read=false          │
└───────────────────┬─────────────────┘
                    │
                    ▼
┌─────────────────────────────────────┐
│  调用 dismissAnnouncement(id)       │
│  (Redux Action)                      │
└───────────────────┬─────────────────┘
                    │
                    ▼
┌─────────────────────────────────────┐
│  POST /api/v1/announcements/:id/    │
│  dismiss                             │
└───────────────────┬─────────────────┘
                    │
                    ▼
┌─────────────────────────────────────┐
│  AnnouncementMute.find_or_create_by!│
│  (account: current_account,          │
│   announcement: @announcement)       │
└───────────────────┬─────────────────┘
                    │
                    ▼
┌─────────────────────────────────────┐
│  前端 Reducer 更新 state:           │
│  { id: announcementId, read: true } │
└─────────────────────────────────────┘
```

#### 1.4.3 前端已读处理逻辑

**文件**: `app/javascript/mastodon/features/home_timeline/components/announcements/announcement.tsx`

```typescript
export const Announcement: FC<AnnouncementProps> = ({
  announcement,
  active,
}) => {
  const { read, id } = announcement;
  const dispatch = useAppDispatch();

  // 当公告变为 active（用户正在查看）且未读时，自动标记已读
  useEffect(() => {
    if (active && !read) {
      dispatch(dismissAnnouncement(id));
    }
  }, [active, id, dispatch, read]);

  // 视觉上的已读状态延迟更新（公告移出视野后才更新样式）
  const [isVisuallyRead, setIsVisuallyRead] = useState(read);
  const [previousActive, setPreviousActive] = useState(active);
  
  if (active !== previousActive) {
    setPreviousActive(active);
    // 只有当公告从 active 变为 inactive 时，才更新视觉状态
    if (!active && isVisuallyRead !== read) {
      setIsVisuallyRead(read);
    }
  }

  return (
    <AnimateEmojiProvider>
      {/* ... 公告内容 ... */}
      {/* 未读指示器 */}
      {!isVisuallyRead && <span className='announcements__unread' />}
    </AnimateEmojiProvider>
  );
};
```

**设计要点**：
1. **自动标记已读**：当用户正在查看某条公告（`active=true`）时自动调用 API 标记已读
2. **视觉延迟更新**：未读指示器的样式变化延迟到公告移出视野后，避免用户看到"闪烁"效果
3. **Carousel 轮播**：多条公告通过轮播组件展示，`active` 状态由轮播组件控制

### 1.5 前端 Banner 展示

#### 1.5.1 组件层级结构

```
HomeTimeline (首页时间线)
│
├── ColumnHeader (列头部)
│   └── IconWithBadge (公告按钮，显示未读计数)
│
└── Announcements (公告 Banner)
    ├── Carousel (轮播容器)
    │   └── Announcement (单条公告)
    │       ├── 标题 + 时间戳
    │       ├── EmojiHTML (内容渲染)
    │       ├── ReactionsBar (反应栏)
    │       └── .announcements__unread (未读指示器)
    │
    └── 装饰元素 (Mastodon 吉祥物图片)
```

#### 1.5.2 首页时间线集成

**文件**: `app/javascript/mastodon/features/home_timeline/index.jsx`

```javascript
class HomeTimeline extends PureComponent {
  // 组件挂载后 700ms 获取公告
  componentDidMount() {
    setTimeout(() => this.props.dispatch(fetchAnnouncements()), 700);
  }

  render() {
    const { hasAnnouncements, unreadAnnouncements, showAnnouncements } = this.props;

    // 公告按钮（显示在列头部）
    let announcementsButton;
    if (hasAnnouncements) {
      announcementsButton = (
        <button
          className={classNames('column-header__button', { 'active': showAnnouncements })}
          onClick={this.handleToggleAnnouncementsClick}
        >
          {/* 喇叭图标 + 未读计数角标 */}
          <IconWithBadge id='bullhorn' icon={CampaignIcon} count={unreadAnnouncements} />
        </button>
      );
    }

    return (
      <Column>
        <ColumnHeader
          extraButton={announcementsButton}
          {/* 当有公告且用户点击展开时显示 Banner */}
          appendContent={hasAnnouncements && showAnnouncements && <Announcements />}
        >
          {/* ... */}
        </ColumnHeader>
        {/* ... */}
      </Column>
    );
  }
}

// Redux 状态映射
const mapStateToProps = state => ({
  hasAnnouncements: !state.getIn(['announcements', 'items']).isEmpty(),
  // 计算未读公告数量
  unreadAnnouncements: state.getIn(['announcements', 'items']).count(item => !item.get('read')),
  showAnnouncements: state.getIn(['announcements', 'show']),
});
```

#### 1.5.3 公告 Banner 组件

**文件**: `app/javascript/mastodon/features/home_timeline/components/announcements/index.tsx`

```typescript
export const Announcements: FC = () => {
  // 从 Redux 获取公告列表（反转顺序，最新的在前）
  const announcements = useAppSelector(announcementSelector);
  const emojis = useCustomEmojis();

  // 轮播项渲染函数
  const renderSlide: RenderSlideFn = useCallback(
    (item, active) => (
      <Announcement
        announcement={item.announcement}
        active={active}  // 传递 active 状态给子组件
        key={item.id}
      />
    ),
    [],
  );

  if (announcements.length === 0) {
    return null;
  }

  return (
    <div className='announcements__root'>
      {/* 吉祥物装饰图片 */}
      <img className='announcements__mastodon' src={mascot} alt='' />
      
      <CustomEmojiProvider emojis={emojis}>
        {/* 轮播组件 */}
        <Carousel
          classNamePrefix='announcements'
          renderItem={renderSlide}
          items={announcements}
        />
      </CustomEmojiProvider>
    </div>
  );
};
```

#### 1.5.4 Redux 状态管理

**Actions** (`app/javascript/mastodon/actions/announcements.js`):

| Action Type | 触发时机 | 说明 |
|-------------|----------|------|
| `ANNOUNCEMENTS_FETCH_REQUEST` | 开始获取公告 | 设置加载状态 |
| `ANNOUNCEMENTS_FETCH_SUCCESS` | 获取成功 | 更新公告列表 |
| `ANNOUNCEMENTS_FETCH_FAIL` | 获取失败 | 清除加载状态 |
| `ANNOUNCEMENTS_DISMISS_SUCCESS` | 标记已读成功 | 更新单条公告的 read 状态 |
| `ANNOUNCEMENTS_TOGGLE_SHOW` | 用户点击展开/收起 | 切换 Banner 显示状态 |
| `ANNOUNCEMENTS_UPDATE` | WebSocket 推送 | 实时更新公告 |
| `ANNOUNCEMENTS_DELETE` | WebSocket 推送删除 | 移除公告 |

**Reducer** (`app/javascript/mastodon/reducers/announcements.js`):

```javascript
const initialState = ImmutableMap({
  items: ImmutableList(),      // 公告列表
  isLoading: false,            // 加载状态
  show: false,                 // 是否展开显示 Banner
});
```

### 1.6 审计日志

#### 1.6.1 日志记录机制

**文件**: `app/controllers/concerns/accountable_concern.rb`

```ruby
module AccountableConcern
  extend ActiveSupport::Concern

  # 记录管理员操作日志
  def log_action(action, target)
    current_account
      .action_logs
      .create(action:, target:)
  end
end
```

#### 1.6.2 公告相关审计事件

在 `Admin::AnnouncementsController` 中记录的操作：

| 操作 | Action 类型 | Target 类型 | 触发时机 |
|------|------------|-------------|----------|
| 创建公告 | `:create` | `Announcement` | 创建成功后 |
| 更新公告 | `:update` | `Announcement` | 修改成功后 |
| 发布公告 | `:update` | `Announcement` | 发布操作 |
| 取消发布 | `:update` | `Announcement` | 取消发布操作 |
| 删除公告 | `:destroy` | `Announcement` | 删除操作 |

**示例代码**：

```ruby
def create
  if @announcement.save
    log_action :create, @announcement  # 记录创建日志
    # ...
  end
end

def publish
  @announcement.publish!
  log_action :update, @announcement  # 记录发布操作
  # ...
end

def destroy
  @announcement.destroy!
  log_action :destroy, @announcement  # 记录删除操作
  # ...
end
```

#### 1.6.3 审计日志查看

**文件**: `app/controllers/admin/action_logs_controller.rb`

管理员可以在后台查看审计日志：

```ruby
module Admin
  class ActionLogsController < BaseController
    def index
      authorize :audit_log, :index?
      # 获取可审计的账户列表（用于筛选）
      @auditable_accounts = Account.auditable.select(:id, :username).order(username: :asc)
      # 使用过滤器获取操作日志
      @action_logs = Admin::ActionLogFilter.new(filter_params).results.page(params[:page])
    end
  end
end
```

---

## 二、规则（Rule）协作机制

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              管理员操作层                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │   创建规则   │  │   编辑规则   │  │   调整顺序   │  │   删除规则   │       │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘       │
│         │                  │                  │                  │              │
│         └──────────────────┼──────────────────┼──────────────────┘              │
│                            ▼                  ▼                                 │
│              ┌─────────────────────────────────────────────┐                   │
│              │       Admin::RulesController                 │                   │
│              │   (app/controllers/admin/rules_controller.rb)│                   │
│              └──────────────────────┬──────────────────────┘                   │
│                                     │                                           │
│         ┌───────────────────────────┴───────────────────────────┐             │
│         │                                                           │             │
│         ▼                                                           ▼             │
│  ┌──────────────┐                                          ┌──────────────┐       │
│  │  数据持久化   │                                          │  软删除机制   │       │
│  │    Rule      │                                          │  (Discard)   │       │
│  │  + 翻译表    │                                          └──────────────┘       │
│  └──────────────┘                                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              数据层                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                         PostgreSQL 数据库                                  │  │
│  │                                                                           │  │
│  │  ┌──────────────┐          ┌──────────────────┐                          │  │
│  │  │    rules     │          │  rule_translations │                          │  │
│  │  │   (规则主表)  │◄────────┤    (翻译表)        │                          │  │
│  │  ├──────────────┤   1:N    ├──────────────────┤                          │  │
│  │  │ id           │          │ id               │                          │  │
│  │  │ text         │          │ rule_id (FK)     │                          │  │
│  │  │ hint         │          │ language         │                          │  │
│  │  │ priority     │          │ text             │                          │  │
│  │  │ deleted_at   │          │ hint             │                          │  │
│  │  │ created_at   │          │ created_at       │                          │  │
│  │  │ updated_at   │          │ updated_at       │                          │  │
│  │  └──────────────┘          └──────────────────┘                          │  │
│  │                                                                           │  │
│  │  软删除说明：                                                              │  │
│  │  - 使用 discard gem 进行软删除                                            │  │
│  │  - 删除时设置 deleted_at 字段                                             │  │
│  │  - 查询时使用 kept scope 过滤已删除项                                     │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API 层                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │       Api::V1::Instances::RulesController                                │  │
│  │  (app/controllers/api/v1/instances/rules_controller.rb)                 │  │
│  └───────────────────────────┬─────────────────────────────────────────────┘  │
│                              │                                                  │
│                              ▼                                                  │
│              ┌──────────────────────────────┐                                 │
│              │  GET /api/v1/instance/rules  │                                 │
│              │    (获取实例规则列表)          │                                 │
│              │  - 支持缓存                    │                                 │
│              │  - 无需登录（公开访问）         │                                 │
│              └──────────────────────────────┘                                 │
│                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                    REST::RuleSerializer                                   │  │
│  │         (app/serializers/rest/rule_serializer.rb)                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐ │  │
│  │  │  序列化字段：                                                          │ │  │
│  │  │  - id: string                                                         │ │  │
│  │  │  - text: string          (规则文本)                                    │ │  │
│  │  │  - hint: string          (提示说明)                                    │ │  │
│  │  │  - translations: object  (多语言翻译)                                  │ │  │
│  │  │    {                                                                   │ │  │
│  │  │      "en": { text: "...", hint: "..." },                             │ │  │
│  │  │      "zh": { text: "...", hint: "..." }                              │ │  │
│  │  │    }                                                                   │ │  │
│  │  └─────────────────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              前端展示层                                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                    RulesSection 组件                                      │  │
│  │        (app/javascript/mastodon/features/about/components/rules.tsx)    │  │
│  │                                                                           │  │
│  │  显示位置：关于页面 (/about)                                               │  │
│  │                                                                           │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐ │  │
│  │  │                           组件结构                                    │ │  │
│  │  │                                                                      │ │  │
│  │  │  ┌─────────────────────────────────────────────────────────────┐   │ │  │
│  │  │  │ Section (标题: "Server rules")                                │   │ │  │
│  │  │  │                                                               │   │ │  │
│  │  │  │  ┌───────────────────────────────────────────────────────┐  │   │ │  │
│  │  │  │  │ ol.rules-list (有序列表)                                │  │   │ │  │
│  │  │  │  │                                                         │  │   │ │  │
│  │  │  │  │  ┌─────────────────────────────────────────────────┐  │  │   │ │  │
│  │  │  │  │  │ li (每条规则)                                      │  │  │   │ │  │
│  │  │  │  │  │ ├── .rules-list__text (规则文本)                  │  │  │   │ │  │
│  │  │  │  │  │ └── .rules-list__hint (提示说明，可选)            │  │  │   │ │  │
│  │  │  │  │  └─────────────────────────────────────────────────┘  │  │   │ │  │
│  │  │  │  └───────────────────────────────────────────────────────┘  │   │ │  │
│  │  │  │                                                               │   │ │  │
│  │  │  │  ┌───────────────────────────────────────────────────────┐  │   │ │  │
│  │  │  │  │ .rules-languages (语言选择器，多语言时显示)            │  │   │ │  │
│  │  │  │  │                                                         │  │   │ │  │
│  │  │  │  │  Label: "Language"                                      │  │   │ │  │
│  │  │  │  │  Select (下拉选择)                                       │  │   │ │  │
│  │  │  │  │  ├── "Default" (默认语言)                               │  │   │ │  │
│  │  │  │  │  ├── "English"                                          │  │   │ │  │
│  │  │  │  │  ├── "中文"                                             │  │   │ │  │
│  │  │  │  │  └── ...                                                │  │   │ │  │
│  │  │  │  └───────────────────────────────────────────────────────┘  │   │ │  │
│  │  │  └─────────────────────────────────────────────────────────────┘   │ │  │
│  │  └─────────────────────────────────────────────────────────────────────┘ │  │
│  │                                                                           │  │
│  │  多语言支持逻辑：                                                          │  │
│  │  1. 从 Redux state 获取规则和翻译数据                                      │  │
│  │  2. 根据用户选择的语言自动切换显示内容                                      │  │
│  │  3. 语言匹配优先级：精确匹配 → 语言前缀匹配 → 默认                          │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 管理员管理规则流程

#### 2.2.1 后台管理控制器

**文件**: `app/controllers/admin/rules_controller.rb`

管理员操作规则的核心入口：

| 操作 | HTTP 方法 | 路由 | 说明 |
|------|----------|------|------|
| 列表 | GET | `/admin/rules` | 查看所有规则（含排序） |
| 创建 | POST | `/admin/rules` | 创建新规则，支持多语言翻译 |
| 编辑 | PATCH/PUT | `/admin/rules/:id` | 修改规则内容和翻译 |
| 上移 | POST | `/admin/rules/:id/move_up` | 调整规则优先级（上移） |
| 下移 | POST | `/admin/rules/:id/move_down` | 调整规则优先级（下移） |
| 删除 | DELETE | `/admin/rules/:id` | **软删除**规则 |

**关键代码逻辑**：

```ruby
# 创建规则
def create
  @rule = Rule.new(resource_params)
  
  if @rule.save
    redirect_to admin_rules_path
  else
    render :new
  end
end

# 更新规则（支持嵌套翻译属性）
def update
  if @rule.update(resource_params)
    redirect_to admin_rules_path
  else
    render :edit
  end
end

# 调整顺序（上移）
def move_up
  @rule.move!(-1)  # priority -1
  redirect_to admin_rules_path
end

# 调整顺序（下移）
def move_down
  @rule.move!(+1)  # priority +1
  redirect_to admin_rules_path
end

# 软删除
def destroy
  @rule.discard  # 设置 deleted_at = Time.now
  redirect_to admin_rules_path
end

# 允许的参数（含翻译的嵌套属性）
def resource_params
  params
    .expect(rule: [:text, :hint, :priority, 
                   translations_attributes: [[:id, :language, :text, :hint, :_destroy]]])
end
```

#### 2.2.2 数据模型

**文件**: `app/models/rule.rb`

规则核心属性：

| 字段 | 类型 | 说明 |
|------|------|------|
| `text` | text | 规则文本（必填，最大 300 字符） |
| `hint` | text | 规则详细说明/提示（可选） |
| `priority` | integer | 排序优先级（默认 0，越小越靠前） |
| `deleted_at` | datetime | 软删除时间戳 |

**关联关系**：

```ruby
class Rule < ApplicationRecord
  include Discard::Model  # 软删除
  self.discard_column = :deleted_at
  
  # 多语言翻译（一对多）
  has_many :translations, 
           -> { order(language: :asc) }, 
           inverse_of: :rule, 
           class_name: 'RuleTranslation', 
           dependent: :destroy
  
  # 接受翻译的嵌套属性
  accepts_nested_attributes_for :translations, 
                                reject_if: ->(attributes) { attributes['text'].blank? }, 
                                allow_destroy: true
end
```

**排序机制**：

```ruby
def move!(offset)
  rules = Rule.ordered.to_a  # 获取当前排序的所有规则
  position = rules.index(self)
  
  rules.delete_at(position)
  rules.insert(position + offset, self)  # 插入到新位置
  
  transaction do
    # 重新计算所有规则的 priority
    rules.each.with_index do |rule, index|
      rule.update!(priority: index)
    end
  end
end

# 默认排序 scope
scope :ordered, -> { kept.order(priority: :asc, id: :asc) }
```

#### 2.2.3 翻译模型

规则支持多语言翻译，通过 `RuleTranslation` 模型实现：

| 字段 | 类型 | 说明 |
|------|------|------|
| `rule_id` | bigint | 关联的规则 ID |
| `language` | string | 语言代码（如 'en', 'zh'） |
| `text` | text | 翻译后的规则文本 |
| `hint` | text | 翻译后的提示说明 |

**获取指定语言翻译**：

```ruby
def translation_for(locale)
  @cached_translations ||= {}
  @cached_translations[locale] ||= 
    translations.for_locale(locale).by_language_length.first || 
    translations.build(language: locale, text: text, hint: hint)
end
```

### 2.3 API 层设计

#### 2.3.1 API 控制器

**文件**: `app/controllers/api/v1/instances/rules_controller.rb`

| 端点 | HTTP 方法 | 权限 | 说明 |
|------|----------|------|------|
| `/api/v1/instance/rules` | GET | **公开** | 获取实例规则列表 |

**关键实现**：

```ruby
class Api::V1::Instances::RulesController < Api::V1::Instances::BaseController
  skip_around_action :set_locale  # 不使用会话 locale
  
  # 重写 current_user：有限联邦模式下才读取 session
  def current_user
    super if limited_federation_mode?
  end

  def index
    cache_even_if_authenticated!  # 支持缓存
    render json: @rules, each_serializer: REST::RuleSerializer
  end

  private

  def set_rules
    # 获取所有未删除的规则，按 priority 排序
    @rules = Rule.ordered.includes(:translations)
  end
end
```

**设计要点**：
1. **公开访问**：无需登录即可访问，因为规则是实例的公开信息
2. **缓存支持**：使用 `cache_even_if_authenticated!` 提高性能
3. **排序保证**：使用 `ordered` scope 确保按管理员设置的顺序返回

#### 2.3.2 序列化器

**文件**: `app/serializers/rest/rule_serializer.rb`

```ruby
class REST::RuleSerializer < ActiveModel::Serializer
  attributes :id, :text, :hint, :translations

  def id
    object.id.to_s
  end

  # 将翻译转换为 Hash 格式
  def translations
    object.translations.to_h do |translation|
      [translation.language, { text: translation.text, hint: translation.hint }]
    end
  end
end
```

**API 响应示例**：

```json
[
  {
    "id": "1",
    "text": "不得发布违法内容",
    "hint": "包括但不限于色情、暴力、欺诈等内容",
    "translations": {
      "en": {
        "text": "No illegal content",
        "hint": "Including but not limited to porn, violence, fraud"
      },
      "ja": {
        "text": "違法コンテンツの投稿禁止",
        "hint": "ポルノ、暴力、詐欺などを含みます"
      }
    }
  },
  {
    "id": "2",
    "text": "尊重他人，禁止骚扰",
    "hint": "",
    "translations": {}
  }
]
```

### 2.4 前端展示

#### 2.4.1 规则展示组件

**文件**: `app/javascript/mastodon/features/about/components/rules.tsx`

组件功能：
1. 从 Redux state 获取规则数据
2. 支持多语言切换
3. 有序列表展示规则

**类型定义**：

```typescript
interface BaseRule {
  text: string;
  hint: string;
}

interface Rule extends BaseRule {
  id: string;
  translations?: Record<string, BaseRule>;
}
```

**语言选择逻辑**：

```typescript
// 获取默认选中的语言
function getDefaultSelectedLocale(
  currentUiLocale: string,  // 当前 UI 语言
  localeOptions: SelectItem[],
) {
  // 1. 精确匹配（如 'zh-CN'）
  const preciseMatch = localeOptions.find(
    (option) => option.value === currentUiLocale,
  );
  if (preciseMatch) return preciseMatch.value;

  // 2. 语言前缀匹配（如 'zh'）
  const partialLocale = currentUiLocale.split('-')[0];
  const partialMatch = localeOptions.find(
    (option) => option.value.split('-')[0] === partialLocale,
  );

  // 3. 使用默认语言
  return partialMatch?.value ?? 'default';
}
```

**规则选择器（根据语言返回对应文本）**：

```typescript
const rulesSelector = createSelector(
  [selectRules, (_state, locale: string) => locale],
  (rules, locale): Rule[] => {
    return rules.map((rule) => {
      const translations = rule.translations;
      if (!translations) return rule;

      // 1. 先尝试语言前缀匹配
      const partialLocale = locale.split('-')[0];
      if (partialLocale && translations[partialLocale]) {
        rule.text = translations[partialLocale].text;
        rule.hint = translations[partialLocale].hint;
      }

      // 2. 再尝试精确匹配（覆盖前缀匹配结果）
      if (translations[locale]) {
        rule.text = translations[locale].text;
        rule.hint = translations[locale].hint;
      }

      return rule;
    });
  },
);
```

#### 2.4.2 组件渲染

```typescript
export const RulesSection: FC<RulesSectionProps> = ({ isLoading = false }) => {
  const intl = useIntl();
  // 获取可用语言选项
  const localeOptions = useAppSelector((state) =>
    localeOptionsSelector(state, intl),
  );
  // 当前选中的语言
  const [selectedLocale, setSelectedLocale] = useState(() =>
    getDefaultSelectedLocale(intl.locale, localeOptions),
  );
  // 根据选中语言获取规则列表
  const rules = useAppSelector((state) => rulesSelector(state, selectedLocale));

  // 加载状态
  if (isLoading) {
    return <Section title={intl.formatMessage(messages.rules)} />;
  }

  // 无规则时显示提示
  if (rules.length === 0) {
    return (
      <Section title={intl.formatMessage(messages.rules)}>
        <p><FormattedMessage id='about.not_available' ... /></p>
      </Section>
    );
  }

  return (
    <Section title={intl.formatMessage(messages.rules)}>
      {/* 规则列表 */}
      <ol className='rules-list'>
        {rules.map((rule) => (
          <li key={rule.id}>
            <div className='rules-list__text'>{rule.text}</div>
            {!!rule.hint && <div className='rules-list__hint'>{rule.hint}</div>}
          </li>
        ))}
      </ol>

      {/* 多语言选择器（有多种翻译时显示） */}
      {localeOptions.length > 1 && (
        <div className='rules-languages'>
          <label htmlFor='language-select'>
            <FormattedMessage id='about.language_label' defaultMessage='Language' />
          </label>
          <Select onChange={handleLocaleChange} id='language-select'>
            {localeOptions.map((option) => (
              <option
                key={option.value}
                value={option.value}
                selected={option.value === selectedLocale}
              >
                {option.text}
              </option>
            ))}
          </Select>
        </div>
      )}
    </Section>
  );
};
```

### 2.5 审计日志

**注意**：与公告不同，规则的管理操作**默认没有记录审计日志**。

在 `Admin::RulesController` 中，没有调用 `log_action` 方法：

```ruby
def create
  @rule = Rule.new(resource_params)
  if @rule.save
    redirect_to admin_rules_path
    # ❌ 没有 log_action :create, @rule
  end
end

def update
  if @rule.update(resource_params)
    redirect_to admin_rules_path
    # ❌ 没有 log_action :update, @rule
  end
end

def destroy
  @rule.discard
  redirect_to admin_rules_path
  # ❌ 没有 log_action :destroy, @rule
end
```

**原因分析**：
1. 规则变更频率相对较低
2. 规则是实例级别的公开信息，变更影响不如公告直接
3. 如需添加审计日志，可参照公告控制器的实现方式添加 `log_action` 调用

---

## 三、公告与规则的对比分析

### 3.1 核心差异

| 维度 | 公告 (Announcement) | 规则 (Rule) |
|------|---------------------|-------------|
| **用途** | 临时通知、活动公告、重要提醒 | 实例行为准则、用户协议 |
| **时效性** | 通常有明确的开始/结束时间 | 长期有效，相对稳定 |
| **用户交互** | 支持已读标记、表情反应 | 仅展示，无交互 |
| **展示位置** | 首页时间线顶部 Banner | 关于页面 (/about) |
| **推送机制** | Redis Pub/Sub 实时推送 | 无推送，用户主动查看 |
| **已读状态** | 有 (AnnouncementMute) | 无 |
| **审计日志** | 完整记录 | **无默认记录** |
| **删除方式** | 物理删除 | 软删除 (Discard) |
| **多语言** | 不支持（内容直接写在 text 中） | 支持（RuleTranslation 模型） |

### 3.2 数据模型对比

```
公告模型关系：
┌─────────────────┐       ┌─────────────────────┐
│   Announcement  │◄──────│   AnnouncementMute  │
│  (公告主表)      │  1:N  │   (用户已读记录)    │
├─────────────────┤       ├─────────────────────┤
│ id              │       │ id                  │
│ text            │       │ account_id          │
│ published       │       │ announcement_id     │
│ published_at    │       │ created_at          │
│ starts_at       │       └─────────────────────┘
│ ends_at         │
│ status_ids      │       ┌─────────────────────┐
│ notification_   │◄──────│ AnnouncementReaction│
│ sent_at         │  1:N  │   (用户反应记录)    │
└─────────────────┘       ├─────────────────────┤
                          │ id                  │
                          │ account_id          │
                          │ announcement_id     │
                          │ name                │
                          │ custom_emoji_id     │
                          └─────────────────────┘

规则模型关系：
┌─────────────────┐       ┌─────────────────────┐
│      Rule       │◄──────│   RuleTranslation   │
│   (规则主表)     │  1:N  │    (翻译表)         │
├─────────────────┤       ├─────────────────────┤
│ id              │       │ id                  │
│ text            │       │ rule_id             │
│ hint            │       │ language            │
│ priority        │       │ text                │
│ deleted_at      │       │ hint                │
└─────────────────┘       └─────────────────────┘
```

### 3.3 前端展示对比

| 特性 | 公告 Banner | 规则列表 |
|------|------------|----------|
| **组件位置** | `home_timeline/components/announcements/` | `about/components/rules.tsx` |
| **触发方式** | 自动加载 + 手动展开 | 用户访问 /about 页面 |
| **展示形式** | Carousel 轮播 | 有序列表 |
| **状态管理** | Redux (announcements module) | Redux (server.rules) |
| **未读提示** | 有（图标角标 + 未读指示器） | 无 |
| **交互操作** | 标记已读、添加反应 | 切换语言 |

### 3.4 API 对比

| 特性 | 公告 API | 规则 API |
|------|----------|----------|
| **端点** | `/api/v1/announcements` | `/api/v1/instance/rules` |
| **权限要求** | 公开（已读状态需登录） | **完全公开** |
| **缓存** | 无特殊缓存 | `cache_even_if_authenticated!` |
| **响应字段** | 含 `read` 个性化字段 | 纯公共数据 |
| **更新频率** | 可能频繁变化 | 相对稳定 |

---

## 四、完整协作流程图

### 4.1 管理员发布公告完整流程

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           管理员发布公告时序图                                      │
└──────────────────────────────────────────────────────────────────────────────────┘

  管理员          Admin::AnnouncementsController    Announcement Model    Sidekiq
     │                       │                              │                  │
     │  POST /admin/announcements (with published: true)  │                  │
     │──────────────────────>│                              │                  │
     │                       │                              │                  │
     │                       │  Announcement.create!(...)   │                  │
     │                       │─────────────────────────────>│                  │
     │                       │                              │                  │
     │                       │  log_action(:create, @announcement)            │
     │                       │─────────────────────────────────────────────────>│ (ActionLog)
     │                       │                              │                  │
     │                       │  PublishScheduledAnnouncementWorker             │
     │                       │  .perform_async(@announcement.id)               │
     │                       │─────────────────────────────────────────────────>│
     │                       │                              │                  │
     │<──────────────────────│                              │                  │
     │   302 Redirect        │                              │                  │
     │                       │                              │                  │
     │                       │                              │     ┌────────────────────────┐
     │                       │                              │     │  Worker 异步执行         │
     │                       │                              │     │                        │
     │                       │                              │     │ 1. refresh_status_ids!  │
     │                       │                              │     │ 2. @announcement.publish!│
     │                       │                              │     │ 3. 序列化为 JSON         │
     │                       │                              │     │ 4. Redis Pub/Sub 推送   │
     │                       │                              │     │    timeline:{account_id} │
     │                       │                              │     └────────────────────────┘
     │                       │                              │                  │

──────────────────────────────────────────────────────────────────────────────────────

  在线用户            WebSocket Client              Redux Store        UI Component
     │                       │                         │                  │
     │                       │  Redis 推送消息          │                  │
     │                       │  { event: 'announcement',│                  │
     │                       │    payload: {...} }      │                  │
     │<──────────────────────│                         │                  │
     │                       │                         │                  │
     │                       │  dispatch(updateAnnouncements)              │
     │                       │────────────────────────>│                  │
     │                       │                         │                  │
     │                       │                         │  更新 state:     │
     │                       │                         │  items.push(...) │
     │                       │                         │                  │
     │                       │                         │  重新排序         │
     │                       │                         │                  │
     │                       │                         │  mapStateToProps │
     │                       │                         │<─────────────────│
     │                       │                         │                  │
     │                       │                         │  hasAnnouncements│
     │                       │                         │  unreadCount++   │
     │                       │                         │                  │
     │                       │                         │─────────────────>│
     │                       │                         │                  │
     │                       │                         │                  │  ColumnHeader
     │                       │                         │                  │  - 显示喇叭图标
     │                       │                         │                  │  - 角标显示未读数
     │                       │                         │                  │
     │                       │                         │                  │  (用户点击展开)
     │                       │                         │                  │
     │                       │                         │                  │  Announcements
     │                       │                         │                  │  - Carousel 轮播
     │                       │                         │                  │  - 显示新公告内容
     │                       │                         │                  │  - 未读指示器闪烁
```

### 4.2 用户标记公告已读流程

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           用户标记公告已读时序图                                    │
└──────────────────────────────────────────────────────────────────────────────────┘

  用户          Announcement Component    Redux Action    API::V1::AnnouncementsController
   │                    │                      │                      │
   │  (轮播切换到该公告)│                      │                      │
   │<───────────────────│                      │                      │
   │                    │                      │                      │
   │                    │  useEffect 检测:     │                      │
   │                    │  active=true &&      │                      │
   │                    │  read=false          │                      │
   │                    │                      │                      │
   │                    │  dispatch(dismissAnnouncement(id))          │
   │                    │─────────────────────>│                      │
   │                    │                      │                      │
   │                    │                      │  POST /api/v1/       │
   │                    │                      │  announcements/:id/  │
   │                    │                      │  dismiss             │
   │                    │                      │─────────────────────>│
   │                    │                      │                      │
   │                    │                      │                      │  AnnouncementMute
   │                    │                      │                      │  .find_or_create_by!(
   │                    │                      │                      │    account: current_user,
   │                    │                      │                      │    announcement: @announcement
   │                    │                      │                      │  )
   │                    │                      │                      │
   │                    │                      │<─────────────────────│
   │                    │                      │   200 OK (empty)     │
   │                    │                      │                      │
   │                    │                      │  dispatch(dismissAnnouncementSuccess(id))
   │                    │<─────────────────────│                      │
   │                    │                      │                      │
   │                    │  Reducer 更新:       │                      │
   │                    │  { id: announcementId, read: true }         │
   │                    │─────────────────────────────────────────────>│ (Redux Store)
   │                    │                      │                      │
   │                    │  但视觉状态延迟更新:  │                      │
   │                    │  isVisuallyRead 保持 │                      │
   │                    │  false 直到公告移出   │                      │
   │                    │  视野                 │                      │
   │                    │                      │                      │
   │                    │  (公告轮播切换)       │                      │
   │                    │                      │                      │
   │                    │  active 变为 false    │                      │
   │                    │                      │                      │
   │                    │  setIsVisuallyRead(read) │                  │
   │                    │  .announcements__unread 指示器消失          │
   │                    │                      │                      │
```

### 4.3 管理员修改规则流程

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           管理员修改规则时序图                                      │
└──────────────────────────────────────────────────────────────────────────────────┘

  管理员          Admin::RulesController      Rule Model         前端用户
     │                       │                      │                  │
     │  PATCH /admin/rules/:id                     │                  │
     │  (with text, hint, translations)            │                  │
     │──────────────────────>│                      │                  │
     │                       │                      │                  │
     │                       │  @rule.update!(      │                  │
     │                       │    resource_params   │                  │
     │                       │  )                    │                  │
     │                       │─────────────────────>│                  │
     │                       │                      │                  │
     │                       │  更新主表 + 翻译表    │                  │
     │                       │                      │                  │
     │                       │  (注意：无 audit log) │                  │
     │                       │                      │                  │
     │<──────────────────────│                      │                  │
     │   302 Redirect        │                      │                  │
     │                       │                      │                  │
     │                       │                      │                  │
     │ (用户访问 /about 页面) │                      │                  │
     │────────────────────────────────────────────────────────────────>│
     │                       │                      │                  │
     │                       │                      │  GET /api/v1/    │
     │                       │                      │  instance/rules  │
     │                       │                      │<─────────────────│
     │                       │                      │                  │
     │                       │                      │  Rule.ordered.   │
     │                       │                      │  includes(:translations)
     │                       │                      │                  │
     │                       │                      │─────────────────>│
     │                       │                      │  200 OK          │
     │                       │                      │  (含 translations)│
     │                       │                      │                  │
     │                       │                      │  RulesSection    │
     │                       │                      │  组件渲染         │
     │                       │                      │  - 显示新文本     │
     │                       │                      │  - 语言选择器     │
     │                       │                      │    (如有翻译)     │
```

---

## 五、关键代码位置索引

### 5.1 公告相关文件

| 文件路径 | 功能描述 |
|----------|----------|
| `app/controllers/admin/announcements_controller.rb` | 后台公告管理控制器 |
| `app/controllers/api/v1/announcements_controller.rb` | 公告 API 控制器 |
| `app/models/announcement.rb` | 公告模型 |
| `app/models/announcement_mute.rb` | 公告已读状态模型 |
| `app/models/announcement_reaction.rb` | 公告反应模型 |
| `app/serializers/rest/announcement_serializer.rb` | 公告序列化器 |
| `app/workers/publish_scheduled_announcement_worker.rb` | 发布公告异步 Worker |
| `app/workers/unpublish_announcement_worker.rb` | 取消公告异步 Worker |
| `app/javascript/mastodon/actions/announcements.js` | 公告 Redux Actions |
| `app/javascript/mastodon/reducers/announcements.js` | 公告 Redux Reducer |
| `app/javascript/mastodon/features/home_timeline/index.jsx` | 首页时间线（公告入口） |
| `app/javascript/mastodon/features/home_timeline/components/announcements/` | 公告 Banner 组件目录 |
| `app/controllers/concerns/accountable_concern.rb` | 审计日志辅助方法 |

### 5.2 规则相关文件

| 文件路径 | 功能描述 |
|----------|----------|
| `app/controllers/admin/rules_controller.rb` | 后台规则管理控制器 |
| `app/controllers/api/v1/instances/rules_controller.rb` | 规则 API 控制器 |
| `app/models/rule.rb` | 规则模型 |
| `app/models/rule_translation.rb` | 规则翻译模型 |
| `app/serializers/rest/rule_serializer.rb` | 规则序列化器 |
| `app/javascript/mastodon/features/about/components/rules.tsx` | 规则展示组件 |

### 5.3 数据库迁移文件

| 文件路径 | 功能描述 |
|----------|----------|
| `db/migrate/20191218153258_create_announcements.rb` | 创建公告表 |
| `db/migrate/20210221045109_create_rules.rb` | 创建规则表 |
| `db/migrate/20210221045110_create_rule_translations.rb` | 创建规则翻译表 |
| `db/migrate/20191218153259_create_announcement_mutes.rb` | 创建公告已读表 |
| `db/migrate/20191218153300_create_announcement_reactions.rb` | 创建公告反应表 |

---

## 六、扩展与优化建议

### 6.1 规则审计日志

当前规则管理没有记录审计日志，如需添加可参考公告的实现：

```ruby
# 在 app/controllers/admin/rules_controller.rb 中添加

def create
  @rule = Rule.new(resource_params)
  
  if @rule.save
    log_action :create, @rule  # 添加此行
    redirect_to admin_rules_path
  else
    render :new
  end
end

def update
  if @rule.update(resource_params)
    log_action :update, @rule  # 添加此行
    redirect_to admin_rules_path
  else
    render :edit
  end
end

def destroy
  @rule.discard
  log_action :destroy, @rule  # 添加此行
  redirect_to admin_rules_path
end
```

### 6.2 公告多语言支持

当前公告不支持多语言，如需实现可参考规则的翻译模式：

1. 创建 `AnnouncementTranslation` 模型
2. 修改序列化器返回翻译数据
3. 前端组件根据用户语言选择显示对应文本

### 6.3 规则变更通知

规则是实例的重要准则，重大变更时可考虑：

1. 发布配套公告通知用户
2. 或在登录时强制显示规则变更确认
3. 记录用户对规则版本的确认状态

---

## 七、总结

### 7.1 公告协作机制总结

公告系统是一个**实时推送、用户交互、状态跟踪**完整的通知系统：

1. **管理端**：管理员创建 → 自动/手动发布 → 审计日志记录
2. **推送层**：Sidekiq Worker 异步处理 → Redis Pub/Sub 实时推送给在线用户
3. **API 层**：提供列表获取和已读标记接口，序列化时返回个性化已读状态
4. **数据层**：AnnouncementMute 表跟踪每个用户的已读状态
5. **前端层**：
   - Redux 管理公告列表、加载状态、展开状态
   - ColumnHeader 显示未读计数角标
   - Announcements 组件以轮播形式展示公告 Banner
   - 公告进入视野时自动标记已读，移出视野后才更新视觉状态

### 7.2 规则协作机制总结

规则系统是一个**相对静态、公开只读、多语言支持**的展示系统：

1. **管理端**：管理员 CRUD → 软删除 → 优先级排序（无审计日志）
2. **API 层**：公开访问端点，支持缓存，返回翻译数据
3. **数据层**：Rule 主表 + RuleTranslation 翻译表，Discard 软删除
4. **前端层**：
   - 展示在关于页面，用户主动访问
   - 支持多语言切换（精确匹配 → 前缀匹配 → 默认）
   - 有序列表形式展示，含可选的 hint 说明

### 7.3 设计思想对比

| 设计原则 | 公告系统 | 规则系统 |
|----------|----------|----------|
| **实时性** | 高 - Redis Pub/Sub 实时推送 | 低 - 用户主动访问获取 |
| **个性化** | 高 - 每个用户独立的已读状态 | 低 - 全局统一内容 |
| **交互性** | 高 - 已读标记、表情反应 | 低 - 仅展示，无交互 |
| **可追溯性** | 高 - 完整审计日志 | 低 - 无默认审计日志 |
| **国际化** | 不支持 | 完善支持（多语言翻译） |
| **数据保留** | 物理删除，不可恢复 | 软删除，可追溯 |

---

## 八、数据流总结

### 8.1 公告数据流

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        公告完整数据流                                          │
└─────────────────────────────────────────────────────────────────────────────┘

管理员操作
    │
    ▼
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│ Admin::       │────▶│ Announcement  │────▶│ ActionLog     │
│ Announcements │     │ (持久化)      │     │ (审计日志)    │
│ Controller    │     └───────┬───────┘     └───────────────┘
└───────────────┘             │
                              │
                              ▼
                    ┌───────────────┐
                    │  Sidekiq      │
                    │    Worker     │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ Redis Pub/Sub │
                    │  (实时推送)    │
                    └───────┬───────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
  在线用户A            在线用户B           在线用户C
        │                   │                   │
        ▼                   ▼                   ▼
┌─────────────────────────────────────────────────────────┐
│              WebSocket / Streaming API                   │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │   Redux Store           │
              │  ┌───────────────────┐  │
              │  │ announcements: {  │  │
              │  │   items: [...],   │  │
              │  │   show: false,    │  │
              │  │   isLoading: false│  │
              │  │ }                 │  │
              │  └───────────────────┘  │
              └─────────────┬───────────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │ ColumnHeader│  │ Announcements│  │ Announcement│
    │  (喇叭按钮)  │  │  (轮播容器)  │  │  (单条公告)  │
    │             │  │             │  │             │
    │ - 未读计数  │  │ - Carousel  │  │ - 内容展示  │
    │ - 展开/收起 │  │ - 动画效果  │  │ - 时间戳    │
    │             │  │             │  │ - 反应栏    │
    └─────────────┘  └─────────────┘  │ - 自动标记  │
                                        │   已读      │
                                        └─────────────┘
```

### 8.2 规则数据流

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        规则完整数据流                                          │
└─────────────────────────────────────────────────────────────────────────────┘

管理员操作
    │
    ▼
┌───────────────┐     ┌───────────────┐     ┌──────────────────┐
│ Admin::       │────▶│ Rule +        │────▶│ RuleTranslation  │
│ RulesController│     │ RuleTranslation│     │   (翻译表)       │
└───────────────┘     │   (主表+翻译)  │     └──────────────────┘
                      └───────┬───────┘
                              │
                              │ 注意：无审计日志
                              │
                              ▼
                    ┌───────────────┐
                    │   用户访问    │
                    │  /about 页面  │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ GET /api/v1/  │
                    │instance/rules │
                    │  (支持缓存)   │
                    └───────┬───────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │   Rule.ordered          │
              │  .includes(:translations)│
              │  (按 priority 排序)     │
              └─────────────┬───────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │   REST::RuleSerializer  │
              │  - id, text, hint       │
              │  - translations (Hash)  │
              └─────────────┬───────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │   Redux Store           │
              │  server.rules           │
              └─────────────┬───────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │   RulesSection 组件     │
              │                         │
              │  ┌───────────────────┐  │
              │  │ 1. 语言选择器     │  │
              │  │    (如有翻译)      │  │
              │  │                   │  │
              │  │ 2. 有序列表展示    │  │
              │  │    - 规则文本      │  │
              │  │    - 提示说明      │  │
              │  │      (可选)        │  │
              │  └───────────────────┘  │
              └─────────────────────────┘
```

---

## 九、关键设计决策分析

### 9.1 公告系统设计决策

| 决策点 | 设计选择 | 原因分析 |
|--------|----------|----------|
| **已读状态存储** | 独立的 `announcement_mutes` 表 | 每个用户每条公告独立状态，支持快速查询 |
| **实时推送** | Redis Pub/Sub | 低延迟，只推送给在线用户，不打扰离线用户 |
| **自动标记已读** | 公告进入视野时触发 | 减少用户操作，提升体验 |
| **视觉延迟更新** | 公告移出视野后才更新样式 | 避免用户看到"闪烁"效果，提升 UX |
| **删除方式** | 物理删除 | 公告具有时效性，过期后无需保留 |

### 9.2 规则系统设计决策

| 决策点 | 设计选择 | 原因分析 |
|--------|----------|----------|
| **删除方式** | 软删除 (Discard gem) | 规则是重要文档，需要保留历史追溯 |
| **多语言支持** | 独立 `rule_translations` 表 | 灵活支持任意语言，不影响主表结构 |
| **排序机制** | `priority` 字段 + 重计算所有规则 | 确保排序连续，避免 gap |
| **API 缓存** | `cache_even_if_authenticated!` | 规则变更频率低，缓存提升性能 |
| **无审计日志** | 默认不记录 | 规则变更频率低，影响相对较小 |

---

**文档生成时间**: 2026-05-04

**分析范围**: Mastodon 公告与规则功能的完整协作机制