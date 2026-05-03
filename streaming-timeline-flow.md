# Mastodon Streaming API 实时 Timeline 更新链路分析

## 1. 整体架构概览

Mastodon 的实时更新系统采用以下分层架构：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           前端 (React/JavaScript)                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  stream.js - WebSocket 连接管理，订阅/取消订阅，消息分发                 │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ WebSocket (ws:// or wss://)
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Streaming API 服务 (Node.js)                            │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  index.js - WebSocket 服务器，订阅管理，消息转发                         │ │
│  │  redis.js - Redis 连接池管理                                              │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ Redis Pub/Sub
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Redis 消息中间件                                    │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  Channel: timeline:{account_id}, timeline:public, timeline:hashtag:tag │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ redis.publish
┌─────────────────────────────────────────────────────────────────────────────┐
│                      后端业务层 (Ruby on Rails)                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  FanOutOnWriteService - 写时扩散服务                                      │ │
│  │  FeedManager - Feed 管理器                                                │ │
│  │  PushUpdateWorker - 推送更新 Worker                                       │ │
│  │  FeedInsertWorker - Feed 插入 Worker                                      │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 2. 详细数据链路

### 2.1 链路总览

当用户发布一条新帖子时，完整的实时推送链路如下：

```
1. 用户发布帖子
        │
        ▼
2. Status 模型保存到数据库
        │
        ▼
3. FanOutOnWriteService.call(status) 被调用
        │
        ├───┬────────────────────────────────────────────┐
        │   │                                            │
        ▼   ▼                                            ▼
4a. 分发到本地接收者            4b. 分发到公共流
    (关注者、列表等)              (公共 timeline、标签流)
        │                                            │
        ▼                                            ▼
5a. FeedInsertWorker 入队              5b. 直接 redis.publish()
        │                                            │
        ▼                                            │
6a. FeedManager.push_to_home()                       │
        │                                            │
        ▼                                            │
7a. 检查用户是否在线 (subscribed: 键存在)             │
        │                                            │
        ▼ 是                                         │
8a. PushUpdateWorker 入队                            │
        │                                            │
        ▼                                            │
9a. redis.publish(timeline_id, message) ◄───────────┘
        │
        ▼
10. Node.js Streaming 服务收到消息
        │
        ▼
11. 通过 WebSocket 推送到前端
        │
        ▼
12. 前端更新 Timeline UI
```

### 2.2 关键代码位置与说明

#### 2.2.1 前端连接层

**文件**: `app/javascript/mastodon/stream.js`

前端使用 `@gamestdio/websocket` 库管理 WebSocket 连接，支持：

1. **共享连接**: 多个订阅复用同一个 WebSocket 连接
2. **自动重连**: 连接断开后自动尝试重连
3. **订阅管理**: 维护订阅计数器，当最后一个订阅取消时才真正取消 WebSocket 订阅

关键代码：

```javascript
// 连接 WebSocket
const ws = new WebSocketClient(`${streamingAPIBaseURL}/api/v1/streaming/?${params.join('&')}`, accessToken);

// 发送订阅消息
sharedConnection.send(JSON.stringify({ type: 'subscribe', stream: channelName, ...params }));

// 接收消息并分发给对应的订阅者
ws.onmessage = e => received(JSON.parse(e.data));
```

#### 2.2.2 Streaming API 服务层

**文件**: `streaming/index.js` (Node.js)

这是一个独立的 Node.js 服务，使用 `ws` 库作为 WebSocket 服务器，使用 `ioredis` 进行 Redis 订阅。

核心组件：

1. **WebSocket 服务器** (`WebSocketServer`): 处理前端的 WebSocket 连接升级
2. **Redis 订阅客户端**: 订阅 Redis 通道，接收后端推送的消息
3. **订阅管理器** (`subs` 对象): 维护通道到回调函数的映射

关键流程：

```javascript
// 1. 处理 WebSocket 连接升级
server.on('upgrade', async function handleUpgrade(request, socket, head) {
  // 认证、权限检查...
  wss.handleUpgrade(request, socket, head, function done(ws) {
    wss.emit('connection', ws, request, wsLogger);
  });
});

// 2. 连接建立后，订阅系统通道
function onConnection(ws, req, log) {
  // 订阅系统通道（用于 token 失效、过滤器变更等）
  subscribeWebsocketToSystemChannel(session);
  
  // 如果 URL 中指定了 stream 参数，立即订阅
  if (query && query.stream) {
    subscribeWebsocketToChannel(session, firstParam(query.stream), query);
  }
}

// 3. 接收前端的订阅/取消订阅消息
ws.on('message', (data, isBinary) => {
  const { type, stream, ...params } = json;
  if (type === 'subscribe') {
    subscribeWebsocketToChannel(session, firstParam(stream), params);
  } else if (type === 'unsubscribe') {
    unsubscribeWebsocketFromChannel(session, firstParam(stream), params);
  }
});

// 4. 订阅 Redis 通道
const subscribe = (channel, callback) => {
  subs[channel] = subs[channel] || [];
  
  // 如果是第一个订阅者，实际订阅 Redis 通道
  if (subs[channel].length === 0) {
    redisSubscribeClient.subscribe(redisNamespaced(channel), ...);
  }
  
  subs[channel].push(callback);
};

// 5. 处理 Redis 消息，转发给 WebSocket 客户端
const onRedisMessage = (channel, message) => {
  const key = redisUnnamespaced(channel);
  const callbacks = subs[key];
  
  const json = parseJSON(message, null);
  callbacks.forEach(callback => callback(json));
};

// 6. 发送消息到 WebSocket 客户端
const streamToWs = (req, ws, streamName) => (event, payload) => {
  const message = JSON.stringify({ stream: streamName, event, payload });
  ws.send(message, ...);
};
```

**心跳机制**:
- Streaming 服务每 6 分钟向 Redis 写入 `subscribed:{channel}` 键，过期时间 18 分钟
- 后端通过检查这个键是否存在来判断用户是否在线

```javascript
const subscriptionHeartbeat = channels => {
  const tellSubscribed = () => {
    channels.forEach(channel => 
      redisClient.set(redisNamespaced(`subscribed:${channel}`), '1', 'EX', interval * 3)
    );
  };
  // 每 6 分钟执行一次
  const heartbeat = setInterval(tellSubscribed, interval * 1000);
};
```

#### 2.2.3 后端业务层

**A. FanOutOnWriteService** (`app/services/fan_out_on_write_service.rb`)

这是整个实时推送的入口服务，当帖子创建或更新时被调用。

```ruby
def call(status, options = {})
  @status    = status
  @account   = status.account
  @options   = options

  check_race_condition!
  warm_payload_cache!  # 预热缓存，避免重复渲染

  fan_out_to_local_recipients!    # 分发给本地接收者
  fan_out_to_public_recipients!   # 分发给标签关注者
  fan_out_to_public_streams!       # 推送到公共流
end
```

**B. 公共流推送**

对于公开可见的帖子，直接推送到对应的 Redis 通道：

```ruby
def broadcast_to_public_streams!
  return if @status.reply? && @status.in_reply_to_account_id != @account.id

  redis.publish('timeline:public', anonymous_payload)
  redis.publish(@status.local? ? 'timeline:public:local' : 'timeline:public:remote', anonymous_payload)

  if @status.with_media?
    redis.publish('timeline:public:media', anonymous_payload)
    redis.publish(@status.local? ? 'timeline:public:local:media' : 'timeline:public:remote:media', anonymous_payload)
  end
end

def broadcast_to_hashtag_streams!
  @status.tags.map(&:name).each do |hashtag|
    redis.publish("timeline:hashtag:#{hashtag.downcase}", anonymous_payload)
    redis.publish("timeline:hashtag:#{hashtag.downcase}:local", anonymous_payload) if @status.local?
  end
end
```

**C. 本地接收者分发**

对于关注者、列表等，通过异步 Worker 处理：

```ruby
def fan_out_to_local_recipients!
  deliver_to_self!  # 推送给自己

  # 分发给所有关注者
  def deliver_to_all_followers!
    @account.followers_for_local_distribution.select(:id).reorder(nil).find_in_batches do |followers|
      FeedInsertWorker.push_bulk(followers) do |follower|
        [@status.id, follower.id, 'home', { 'update' => update? }]
      end
    end
  end

  # 分发给列表
  def deliver_to_lists!
    @account.lists_for_local_distribution.select(:id).reorder(nil).find_in_batches do |lists|
      FeedInsertWorker.push_bulk(lists) do |list|
        [@status.id, list.id, 'list', { 'update' => update? }]
      end
    end
  end
end
```

**D. FeedInsertWorker** (`app/workers/feed_insert_worker.rb`)

处理单个关注者的 Feed 插入：

```ruby
def perform(status_id, id, type = 'home', options = {})
  @type      = type.to_sym
  @status    = Status.find(status_id)
  @options   = options.symbolize_keys

  case @type
  when :home, :tags
    @follower = Account.find(id)
  when :list
    @list     = List.find(id)
    @follower = @list.account
  end

  check_and_insert
end

def check_and_insert
  filter_result = feed_filter

  if filter_result
    perform_unpush if update?  # 如果需要过滤且是更新操作，移除旧的
  else
    perform_push               # 插入到 Feed
  end
end

def perform_push
  case @type
  when :home, :tags
    FeedManager.instance.push_to_home(@follower, @status, update: update?)
  when :list
    FeedManager.instance.push_to_list(@list, @status, update: update?)
  end
end
```

**E. FeedManager** (`app/lib/feed_manager.rb`)

管理 Feed 的增删和实时推送：

```ruby
def push_to_home(account, status, update: false)
  # 只有最近登录的用户才需要更新
  return false unless account.user&.signed_in_recently?
  
  # 将 status ID 添加到 Redis ZSet (Feed 存储)
  return false unless add_to_feed(:home, account.id, status, aggregate_reblogs: account.user&.aggregates_reblogs?)

  trim(:home, account.id)  # 裁剪 Feed 到最大长度
  
  # 关键：如果用户在线（订阅了 Streaming API），推送实时更新
  PushUpdateWorker.perform_async(account.id, status.id, "timeline:#{account.id}", { 'update' => update }) if push_update_required?("timeline:#{account.id}")
  true
end

# 检查用户是否在线（通过 Streaming 服务的心跳键）
def push_update_required?(timeline_key)
  redis.exists?("subscribed:#{timeline_key}")
end
```

**F. PushUpdateWorker** (`app/workers/push_update_worker.rb`)

实际执行 Redis 发布的 Worker：

```ruby
class PushUpdateWorker
  include Sidekiq::Worker
  include Redisable

  def perform(account_id, status_id, timeline_id = nil, options = {})
    @status      = Status.find(status_id)
    @account_id  = account_id
    @timeline_id = timeline_id || "timeline:#{account_id}"
    @options     = options.symbolize_keys

    render_payload!   # 渲染 Status 为 JSON
    publish!          # 发布到 Redis
  end

  private

  def render_payload!
    @payload = StatusCacheHydrator.new(@status).hydrate(@account_id)
  end

  def message
    JSON.generate({
      event: update? ? :'status.update' : :update,
      payload: @payload,
    }.as_json)
  end

  def publish!
    redis.publish(@timeline_id, message)  # 关键：发布到 Redis 通道
  end
end
```

## 3. Redis 通道命名规范

| 通道名称模式 | 用途 | 示例 |
|-------------|------|------|
| `timeline:{account_id}` | 用户个人 Home Timeline | `timeline:123` |
| `timeline:{account_id}:notifications` | 用户通知流 | `timeline:123:notifications` |
| `timeline:public` | 全局公共 Timeline | `timeline:public` |
| `timeline:public:local` | 本实例公共 Timeline | `timeline:public:local` |
| `timeline:public:remote` | 远程实例公共 Timeline | `timeline:public:remote` |
| `timeline:public:media` | 全局媒体 Timeline | `timeline:public:media` |
| `timeline:public:local:media` | 本实例媒体 Timeline | `timeline:public:local:media` |
| `timeline:public:remote:media` | 远程实例媒体 Timeline | `timeline:public:remote:media` |
| `timeline:hashtag:{tag}` | 标签 Timeline | `timeline:hashtag:ruby` |
| `timeline:hashtag:{tag}:local` | 本实例标签 Timeline | `timeline:hashtag:ruby:local` |
| `timeline:list:{list_id}` | 列表 Timeline | `timeline:list:456` |
| `timeline:direct:{account_id}` | 私信 Timeline | `timeline:direct:123` |
| `timeline:system:{account_id}` | 系统消息通道 | `timeline:system:123` |
| `timeline:access_token:{token_id}` | Token 控制通道 | `timeline:access_token:789` |
| `subscribed:{channel}` | 在线状态心跳键 | `subscribed:timeline:123` |

## 4. 消息格式

### 4.1 从 Rails 到 Redis 的消息格式

```json
{
  "event": "update",
  "payload": "{\"id\":\"123456\",\"content\":\"<p>Hello World</p>\",...}"
}
```

- `event`: 事件类型，如 `update`、`delete`、`notification`、`status.update`
- `payload`: JSON 序列化后的实体数据

### 4.2 从 Node.js 到前端的消息格式

```json
{
  "stream": ["user"],
  "event": "update",
  "payload": "{\"id\":\"123456\",\"content\":\"<p>Hello World</p>\",...}"
}
```

- `stream`: 数组，标识消息来自哪个流，如 `["user"]`、`["hashtag", "ruby"]`、`["list", "456"]`
- `event`: 事件类型
- `payload`: JSON 序列化后的实体数据

## 5. 关键设计决策

### 5.1 写时扩散 (Fan-out on Write) vs 读时聚合 (Fan-in on Read)

Mastodon 采用**写时扩散**策略：

**优点**:
- 读取速度快（直接从 Redis ZSet 读取）
- 实时性好（写入时立即推送）

**缺点**:
- 写入开销大（大 V 有百万粉丝时，写入会很慢）
- 存储开销大（每个用户的 Feed 都需要存储）

**优化**:
- 使用 Sidekiq 异步处理 (`FeedInsertWorker`)
- 批量处理 (`find_in_batches` + `push_bulk`)
- 只给最近登录的用户更新 (`signed_in_recently?`)
- 只给在线用户推送实时更新 (`push_update_required?`)

### 5.2 在线状态判断

通过 Redis 键的存在性判断用户是否在线：

1. **Streaming 服务**: 每 6 分钟写入 `subscribed:{channel}`，过期时间 18 分钟
2. **Rails 后端**: 检查 `redis.exists?("subscribed:#{timeline_key}")` 来判断是否需要推送

这种设计避免了 Rails 直接维护 WebSocket 连接状态，实现了服务解耦。

### 5.3 共享 WebSocket 连接

前端使用**单个 WebSocket 连接**支持多个订阅：

1. 连接建立后，前端可以通过发送 `{ type: 'subscribe', stream: '...' }` 消息订阅多个流
2. Node.js 服务维护 `subs` 映射，将 Redis 消息分发给对应的订阅者
3. 消息中的 `stream` 字段告诉前端这条消息属于哪个订阅

**优势**:
- 减少 TCP 连接开销
- 减少 TLS 握手开销
- 简化连接管理

### 5.4 两级缓存

1. **帖子内容缓存**: `Rails.cache.write("fan-out/#{@status.id}", rendered_status)`
2. **订阅者过滤缓存**: Streaming 服务会缓存用户的过滤器、屏蔽列表等

## 6. 支持的事件类型

| 事件类型 | 说明 | 触发场景 |
|---------|------|---------|
| `update` | 新帖子 | 帖子创建 |
| `status.update` | 帖子更新 | 帖子编辑 |
| `delete` | 帖子删除 | 帖子删除 |
| `notification` | 新通知 | 提到、点赞、转发等 |
| `conversation` | 新私信会话 | 收到私信 |
| `filters_changed` | 过滤器变更 | 用户修改过滤规则 |
| `announcement` | 新公告 | 管理员发布公告 |
| `announcement.delete` | 公告删除 | 管理员删除公告 |
| `announcement.reaction` | 公告反应 | 用户对公告添加反应 |
| `kill` | 连接关闭 | Token 失效等系统事件 |

## 7. 数据流时序图

```
┌──────┐     ┌──────┐     ┌─────────────┐     ┌───────┐     ┌─────────┐     ┌──────┐
│ 前端 │     │ WS   │     │  Streaming  │     │ Redis │     │  Rails  │     │  DB  │
└──┬───┘     └──┬───┘     └──────┬──────┘     └───┬───┘     └────┬────┘     └──┬───┘
   │            │                 │                 │               │             │
   │  1. 建立 WebSocket 连接      │                 │               │             │
   │───────────>│                 │                 │               │             │
   │            │  upgrade()      │                 │               │             │
   │            │────────────────>│                 │               │             │
   │            │                 │  认证 + 订阅系统通道           │             │
   │            │                 │────────────────>│               │             │
   │            │                 │ SUBSCRIBE      │               │             │
   │            │                 │ timeline:system:123            │             │
   │            │                 │                 │               │             │
   │  2. 订阅 Home Timeline       │                 │               │             │
   │───────────────────────────────────────────────────────────────>│             │
   │ {type:'subscribe',stream:'user'}             │               │             │
   │            │                 │                 │               │             │
   │            │                 │ SUBSCRIBE      │               │             │
   │            │                 │ timeline:123   │               │             │
   │            │                 │────────────────>│               │             │
   │            │                 │ SET subscribed:timeline:123   │             │
   │            │                 │────────────────>│               │             │
   │            │                 │                 │               │             │
   │  3. 用户发布帖子             │                 │               │             │
   │───────────────────────────────────────────────────────────────>│             │
   │            │                 │                 │               │  INSERT     │
   │            │                 │                 │               │────────────>│
   │            │                 │                 │               │             │
   │            │                 │                 │               │ FanOutOnWriteService
   │            │                 │                 │               │             │
   │            │                 │                 │               │  检查在线状态
   │            │                 │                 │   EXISTS      │             │
   │            │                 │                 │<──────────────│             │
   │            │                 │                 │   (返回 true) │             │
   │            │                 │                 │               │             │
   │            │                 │                 │               │ PUBLISH     │
   │            │                 │                 │<──────────────│             │
   │            │                 │                 │ timeline:123  │             │
   │            │                 │                 │ {event:'update',payload:'...'}
   │            │                 │                 │               │             │
   │            │                 │  收到消息       │               │             │
   │            │                 │<────────────────│               │             │
   │            │                 │                 │               │             │
   │  4. 前端收到更新             │                 │               │             │
   │<───────────│                 │                 │               │             │
   │ {stream:['user'],event:'update',payload:'...'}│               │             │
   │            │                 │                 │               │             │
   │  5. 心跳维持                │                 │               │             │
   │            │                 │  每 6 分钟      │               │             │
   │            │                 │  SET subscribed:timeline:123   │             │
   │            │                 │────────────────>│               │             │
   │            │                 │  EX 18 分钟     │               │             │
   │            │                 │                 │               │             │
┌──┴───┐     ┌──┴───┐     ┌──────┴──────┐     ┌───┴───┐     ┌────┴────┐     ┌──┴───┐
│ 前端 │     │ WS   │     │  Streaming  │     │ Redis │     │  Rails  │     │  DB  │
└──────┘     └──────┘     └─────────────┘     └───────┘     └─────────┘     └──────┘
```

## 8. 关键文件索引

| 文件路径 | 职责 |
|---------|------|
| `app/javascript/mastodon/stream.js` | 前端 WebSocket 连接管理 |
| `streaming/index.js` | Node.js Streaming API 服务主入口 |
| `streaming/redis.js` | Node.js Redis 连接配置 |
| `app/services/fan_out_on_write_service.rb` | 写时扩散服务，帖子分发入口 |
| `app/workers/feed_insert_worker.rb` | Feed 插入异步 Worker |
| `app/workers/push_update_worker.rb` | 实时推送异步 Worker |
| `app/lib/feed_manager.rb` | Feed 管理器，处理增删和推送判断 |
| `app/controllers/api/v1/streaming_controller.rb` | Rails 端 Streaming API 控制器（仅重定向） |

## 9. 总结

Mastodon 的 Streaming API 实时推送链路是一个**经典的 Pub/Sub 架构**：

1. **生产端 (Rails)**: 帖子创建时，`FanOutOnWriteService` 负责将事件分发到各个 Redis 通道
2. **消息中间件 (Redis)**: 使用 Redis 的 Pub/Sub 机制实现跨服务通信
3. **消费端 (Node.js)**: 独立的 Streaming 服务订阅 Redis 通道，通过 WebSocket 推送到前端
4. **客户端 (JavaScript)**: 维护 WebSocket 连接，接收实时更新并更新 UI

**核心优化点**:
- **异步处理**: 使用 Sidekiq 处理耗时的粉丝分发
- **在线判断**: 通过 Redis 心跳键只给在线用户推送
- **连接复用**: 单个 WebSocket 连接支持多个订阅
- **批量操作**: 使用 `find_in_batches` 和 `push_bulk` 提高效率
- **分层过滤**: Rails 端做主要过滤，Streaming 服务做补充过滤
