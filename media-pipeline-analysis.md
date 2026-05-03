# Mastodon 媒体处理流水线分析报告

## 概述

本文档详细分析了 Mastodon 平台中媒体文件从上传、转码到远端缓存的完整处理流程，重点关注任务分派机制、格式转换策略和缓存失效处理。

---

## 1. 媒体上传流程

### 1.1 上传入口

媒体上传的主要入口是 `Api::V1::MediaController`，该控制器提供了完整的 CRUD 操作：

```ruby
# app/controllers/api/v1/media_controller.rb
def create
  @media_attachment = current_account.media_attachments.create!(media_attachment_params)
  render json: @media_attachment, serializer: REST::MediaAttachmentSerializer
end
```

**关键点**：
- 需要 OAuth 授权 (`write:media` 权限)
- 支持文件上传、缩略图、描述和焦点点参数
- 处理两种特殊错误：文件类型无法验证 (422) 和处理错误 (500)

### 1.2 媒体附件模型

`MediaAttachment` 模型是媒体处理的核心，定义了媒体的属性、样式和处理流程：

**支持的媒体类型**：
- `image`: 图片 (JPEG, PNG, GIF, WebP, HEIC, HEIF, AVIF)
- `gifv`: 转换为视频的 GIF 动画
- `video`: 视频 (WebM, MP4, M4V, MOV)
- `audio`: 音频 (OGG, MP3, WAV, FLAC, AAC 等)
- `unknown`: 未知类型

**文件大小限制**：
- 图片: 16MB
- 视频/音频: 99MB

**视频参数限制**：
- 最大分辨率: 3840x2160px (8,294,400 像素)
- 最大帧率: 120fps
- 最大帧数: 36,000 帧 (约 5 分钟 @ 120fps)

### 1.3 上传处理流程

1. **验证阶段** (`Attachmentable` 模块)：
   - 检查图片尺寸
   - 修正文件内容类型 (处理浏览器/工具报告错误的 MIME 类型)
   - 混淆文件名 (使用 `SecureRandom.hex(8)`)
   - 设置正确的文件扩展名

2. **类型检测**：
   ```ruby
   # app/models/media_attachment.rb:356-366
   def set_type_and_extension
     self.type = begin
       if VIDEO_MIME_TYPES.include?(file_content_type)
         :video
       elsif AUDIO_MIME_TYPES.include?(file_content_type)
         :audio
       else
         :image
       end
     end
   end
   ```

3. **视频维度验证**：
   - 检查视频流是否存在
   - 验证分辨率不超过限制
   - 验证帧率不超过限制

---

## 2. 任务分派机制

### 2.1 延迟处理决策

Mastodon 对不同类型的媒体采用不同的处理策略：

```ruby
# app/models/media_attachment.rb:289-291
def delay_processing_for_attachment?(attachment_name)
  delay_processing? && attachment_name == :file
end

def delay_processing?
  @delay_processing && larger_media_format?
end

def larger_media_format?
  video? || gifv? || audio?
end
```

**策略总结**：
- **图片**: 同步处理 (即时完成)
- **视频/GIF/音频**: 异步处理 (延迟到后台任务)

### 2.2 任务队列架构

使用 **Sidekiq** 作为任务队列系统，主要涉及以下 Worker：

#### 2.2.1 PostProcessMediaWorker (媒体后处理)

```ruby
# app/workers/post_process_media_worker.rb
def perform(media_attachment_id)
  media_attachment = MediaAttachment.find(media_attachment_id)
  media_attachment.processing = :in_progress
  media_attachment.save

  # 保存元数据以避免被覆盖
  previous_meta = media_attachment.file_meta

  media_attachment.file.reprocess!(:original)
  media_attachment.processing = :complete
  # 合并新旧元数据
  media_attachment.file_meta = previous_meta.merge(media_attachment.file_meta)...
  media_attachment.save
end
```

**任务特性**：
- 重试次数: 1 次
- 失败策略: 标记为 `failed` 状态，不进入死信队列
- 状态流转: `queued` → `in_progress` → `complete` / `failed`

#### 2.2.2 RedownloadMediaWorker (远程媒体重新下载)

```ruby
# app/workers/redownload_media_worker.rb
def perform(id)
  media_attachment = MediaAttachment.find(id)
  return if media_attachment.remote_url.blank?

  media_attachment.download_file!
  media_attachment.download_thumbnail!
  media_attachment.save
end
```

**任务特性**：
- 队列: `pull`
- 重试次数: 3 次 (指数退避)
- 用于从远程实例重新下载媒体文件

#### 2.2.3 CacheBusterWorker (缓存失效)

```ruby
# app/workers/cache_buster_worker.rb
def perform(path)
  cache_buster.bust(full_asset_url(path))
end
```

**任务特性**：
- 队列: `pull`
- 用于通知 CDN/缓存服务器失效特定资源

### 2.3 处理状态枚举

```ruby
enum :processing, { queued: 0, in_progress: 1, complete: 2, failed: 3 }, prefix: true
```

**状态流转**：
1. `queued`: 等待处理 (仅适用于大型媒体)
2. `in_progress`: 正在处理中
3. `complete`: 处理完成
4. `failed`: 处理失败

---

## 3. 格式转换（转码）流程

### 3.1 处理器选择机制

根据媒体类型自动选择不同的处理器：

```ruby
# app/models/media_attachment.rb:337-347
def file_processors(instance)
  if instance.file_content_type == 'image/gif'
    [:gif_transcoder, :blurhash_transcoder]
  elsif VIDEO_MIME_TYPES.include?(instance.file_content_type)
    [:transcoder, :blurhash_transcoder, :type_corrector]
  elsif AUDIO_MIME_TYPES.include?(instance.file_content_type)
    [:image_extractor, :transcoder, :type_corrector]
  else
    [:lazy_thumbnail, :blurhash_transcoder, :type_corrector]
  end
end
```

### 3.2 样式配置

根据媒体类型定义不同的输出样式：

#### 3.2.1 图片样式

```ruby
IMAGE_STYLES = {
  original: {
    pixels: 8_294_400, # 3840x2160px
    file_geometry_parser: FastGeometryParser,
  }.freeze,
  small: {
    pixels: 230_400, # 640x360px
    file_geometry_parser: FastGeometryParser,
    blurhash: BLURHASH_OPTIONS,
  }.freeze,
}
```

**可转换图片类型** (HEIC/HEIF/AVIF):
- 转换为 JPEG 格式
- 保持 `original` 和 `small` 两种尺寸

#### 3.2.2 视频样式

```ruby
VIDEO_FORMAT = {
  format: 'mp4',
  content_type: 'video/mp4',
  vfr_frame_rate_threshold: MAX_VIDEO_FRAME_RATE, # 120fps
  convert_options: {
    output: {
      'loglevel' => 'fatal',
      'preset' => 'veryfast',
      'movflags' => 'faststart', # 元数据前置，支持渐进式播放
      'pix_fmt' => 'yuv420p',    # 跨浏览器兼容的色彩空间
      'vf' => 'crop=floor(iw/2)*2:floor(ih/2)*2', # H.264 要求宽高为偶数
      'c:v' => 'h264',
      'c:a' => 'aac',
      'b:a' => '192k',
      'map_metadata' => '-1',     # 移除元数据
      'frames:v' => MAX_VIDEO_FRAMES, # 36,000 帧
    }.freeze,
  }.freeze,
}.freeze
```

#### 3.2.3 透传模式 (Passthrough)

对于符合特定条件的视频，直接复制流而不重新编码：

```ruby
VIDEO_PASSTHROUGH_OPTIONS = {
  video_codecs: ['h264'].freeze,
  audio_codecs: ['aac', nil].freeze,
  colorspaces: ['yuv420p', 'yuvj420p'].freeze,
  options: {
    format: 'mp4',
    convert_options: {
      output: {
        'c:v' => 'copy',  # 直接复制视频流
        'c:a' => 'copy',  # 直接复制音频流
        # ...
      }.freeze,
    }.freeze,
  }.freeze,
}.freeze
```

**透传条件**：
- 视频编码: H.264
- 音频编码: AAC 或无音频
- 色彩空间: yuv420p 或 yuvj420p

**优势**：
- 处理速度快 (无需重新编码)
- 保留原始质量
- 节省 CPU 资源

#### 3.2.4 音频样式

```ruby
AUDIO_STYLES = {
  original: {
    format: 'mp3',
    content_type: 'audio/mpeg',
    convert_options: {
      output: {
        'loglevel' => 'fatal',
        'q:a' => 2,  # VBR 质量级别 2 (约 170-210kbps)
      }.freeze,
    }.freeze,
  }.freeze,
}
```

### 3.3 转码处理器详解

#### 3.3.1 Transcoder (主转码器)

**位置**: `lib/paperclip/transcoder.rb`

**核心功能**：
1. **元数据提取**：使用 `VideoMetadataExtractor` 分析输入文件
2. **类型更新**：根据音频流存在与否，将无音频视频标记为 `gifv`
3. **透传检查**：判断是否可以直接复制流
4. **动态码率计算**：

```ruby
# 计算目标码率
size_limit_in_bits = MediaAttachment::VIDEO_LIMIT * 8  # 99MB * 8
desired_bitrate = (metadata.width * metadata.height * 30 * BITS_PER_PIXEL).floor
# BITS_PER_PIXEL = 0.11 (H.264 High Profile)
duration = [metadata.duration, 1].max
maximum_bitrate = (size_limit_in_bits / duration).floor - 192_000 # 预留音频空间
bitrate = [desired_bitrate, maximum_bitrate].min
```

5. **可变帧率处理**：
   - 当输入帧率超过 120fps 时，使用 `fps_mode: 'vfr'` 保持可变帧率

6. **FFmpeg 命令构建**：
   - 使用 `Terrapin::CommandLine` 安全执行 FFmpeg 命令
   - 支持输入选项、输出选项的灵活配置

#### 3.3.2 GifTranscoder (GIF 转码器)

**位置**: `lib/paperclip/gif_transcoder.rb`

**核心功能**：
1. **动画检测**：使用自定义的 `GifReader` 类检测 GIF 是否为动画
2. **转换逻辑**：
   - 静态 GIF: 直接返回原文件
   - 动画 GIF: 转换为 MP4 视频 (gifv 格式)

**GifReader 工作原理**：
- 解析 GIF 文件头 (GIF87a/GIF89a)
- 计算图像块数量
- 超过 1 帧则判定为动画

#### 3.3.3 其他处理器

- **BlurhashTranscoder**: 生成 Blurhash 字符串 (用于占位图)
- **TypeCorrector**: 修正媒体类型
- **ImageExtractor**: 从音频提取封面图
- **LazyThumbnail**: 延迟生成缩略图

### 3.4 元数据处理

```ruby
# app/models/media_attachment.rb:384-398
def set_meta
  file.instance_write :meta, populate_meta
end

def populate_meta
  meta = (file.instance_read(:meta) || {}).with_indifferent_access.slice(*META_KEYS)
  
  file.queued_for_write.each do |style, file|
    meta[style] = style == :small || image? ? image_geometry(file) : video_metadata(file)
  end
  
  meta[:small] = image_geometry(thumbnail.queued_for_write[:original]) if thumbnail.queued_for_write.key?(:original)
  
  meta
end
```

**收集的元数据**：
- `focus`: 用户指定的焦点点
- `colors`: 主要颜色
- `original`: 原始版本的尺寸/元数据
- `small`: 缩略图版本的尺寸/元数据

**图片元数据**：
- `width`: 宽度
- `height`: 高度
- `size`: "宽x高" 格式
- `aspect`: 宽高比

**视频元数据**：
- `width`: 宽度
- `height`: 高度
- `frame_rate`: 帧率
- `duration`: 时长 (秒)
- `bitrate`: 码率

---

## 4. 远端缓存和缓存失效策略

### 4.1 远程附件机制

**Remotable 模块** (`app/models/concerns/remotable.rb`) 提供了远程媒体的下载和缓存能力：

```ruby
def remotable_attachment(attachment_name, limit, suppress_errors: true, download_on_assign: true, attribute_name: nil)
  # 定义 download_#{attachment_name}! 方法
  define_method(:"download_#{attachment_name}!") do |url = nil|
    # ...
    Request.new(:get, url).perform do |response|
      raise Mastodon::UnexpectedResponseError, response unless (200...300).cover?(response.code)
      public_send(:"#{attachment_name}=", ResponseWithLimit.new(response, limit))
    end
    # ...
  end
end
```

**MediaAttachment 中的应用**：
```ruby
remotable_attachment :file, VIDEO_LIMIT, suppress_errors: false, download_on_assign: false, attribute_name: :remote_url
remotable_attachment :thumbnail, IMAGE_LIMIT, suppress_errors: true, download_on_assign: false
```

**关键特性**：
- `download_on_assign: false`: 赋值时不自动下载，需手动触发
- `suppress_errors`: 控制是否静默失败
- 限制下载大小 (视频 99MB，图片 16MB)

### 4.2 作用域定义

```ruby
# app/models/media_attachment.rb:212-227
scope :attached, -> { where.not(status_id: nil).or(where.not(scheduled_status_id: nil)) }
scope :cached, -> { remote.where.not(file_file_name: nil) }
scope :created_before, ->(value) { where(arel_table[:created_at].lt(value)) }
scope :local, -> { where(remote_url: '') }
scope :ordered, -> { order(id: :asc) }
scope :remote, -> { where.not(remote_url: '') }
scope :unattached, -> { where(status_id: nil, scheduled_status_id: nil) }
scope :updated_before, ->(value) { where(arel_table[:updated_at].lt(value)) }
```

**关键作用域说明**：
- `remote`: 有 `remote_url` 的媒体 (来自其他实例)
- `cached`: 远程且已下载到本地的媒体
- `local`: 本地上传的媒体
- `unattached`: 未关联到任何状态的媒体

### 4.3 定时清理机制

#### 4.3.1 Vacuum Scheduler

```ruby
# app/workers/scheduler/vacuum_scheduler.rb
def perform
  vacuum_operations.each do |operation|
    operation.perform
  end
end

private

def vacuum_operations
  [
    statuses_vacuum,
    media_attachments_vacuum,  # 媒体附件清理
    preview_cards_vacuum,
    backups_vacuum,
    access_tokens_vacuum,
    feeds_vacuum,
    imports_vacuum,
  ]
end

def media_attachments_vacuum
  Vacuum::MediaAttachmentsVacuum.new(content_retention_policy.media_cache_retention_period)
end
```

**调度特性**：
- 锁机制: `lock: :until_executed` 确保单实例执行
- 锁 TTL: 1 天
- 重试: 0 次

#### 4.3.2 MediaAttachmentsVacuum

```ruby
# app/lib/vacuum/media_attachments_vacuum.rb
TTL = 1.day.freeze

def perform
  vacuum_orphaned_records!
  vacuum_cached_files! if retention_period?
end

private

def vacuum_cached_files!
  media_attachments_past_retention_period.find_in_batches do |media_attachments|
    AttachmentBatch.new(MediaAttachment, media_attachments).clear
  end
end

def vacuum_orphaned_records!
  orphaned_media_attachments.find_in_batches do |media_attachments|
    AttachmentBatch.new(MediaAttachment, media_attachments).delete
  end
end

def media_attachments_past_retention_period
  MediaAttachment
    .remote
    .cached
    .created_before(@retention_period.ago)
    .updated_before(@retention_period.ago)
end

def orphaned_media_attachments
  MediaAttachment
    .unattached
    .created_before(TTL.ago)  # 1 天前
end
```

**清理策略**：

1. **孤立记录清理** (`orphaned_media_attachments`):
   - 条件: 未关联到任何状态 (`unattached`) + 创建超过 1 天
   - 操作: 完全删除记录 (`delete`)

2. **缓存文件清理** (`media_attachments_past_retention_period`):
   - 条件: 远程媒体 + 已缓存 + 创建和更新都超过保留期
   - 操作: 清除文件但保留记录 (`clear`)

#### 4.3.3 内容保留策略

```ruby
# app/models/content_retention_policy.rb
def media_cache_retention_period
  retention_period Setting.media_cache_retention_period
end

def content_cache_retention_period
  retention_period Setting.content_cache_retention_period
end

private

def retention_period(value)
  value.days if value.is_a?(Integer) && value.positive?
end
```

**配置说明**：
- `media_cache_retention_period`: 媒体缓存保留天数
- `content_cache_retention_period`: 内容缓存保留天数
- 当值为 0 或负数时，不启用自动清理

### 4.4 主动缓存失效

当媒体附件被删除时，主动通知缓存服务器：

```ruby
# app/models/media_attachment.rb:296-297
before_destroy :prepare_cache_bust!, prepend: true
after_destroy :bust_cache!
```

#### 4.4.1 prepare_cache_bust! (删除前准备)

```ruby
def prepare_cache_bust!
  return unless Rails.configuration.x.cache_buster.enabled

  @paths_to_cache_bust = MediaAttachment.attachment_definitions.keys.flat_map do |attachment_name|
    attachment = public_send(attachment_name)
    next if attachment.blank?

    styles = DEFAULT_STYLES | attachment.styles.keys
    styles.map { |style| attachment.url(style) }
  end.compact
end
```

**收集的 URL**：
- `:file` 附件的所有样式 URL
- `:thumbnail` 附件的所有样式 URL
- 包含 `:original` 和其他定义的样式

#### 4.4.2 bust_cache! (删除后执行)

```ruby
def bust_cache!
  return unless Rails.configuration.x.cache_buster.enabled

  CacheBusterWorker.push_bulk(@paths_to_cache_bust) { |path| [path] }
end
```

**使用批量推送** (`push_bulk`) 提高效率。

### 4.5 CacheBuster 实现

```ruby
# app/lib/cache_buster.rb
class CacheBuster
  def initialize(options = {})
    @secret_header = options[:secret_header]
    @secret = options[:secret]
    @http_method = options[:http_method] || 'GET'
  end

  def bust(url)
    site = Addressable::URI.parse(url).normalized_site

    request_pool.with(site) do |http_client|
      build_request(url, http_client).perform
    end
  end

  private

  def build_request(url, http_client)
    request = Request.new(@http_method.downcase.to_sym, url, http_client: http_client)
    request.add_headers(@secret_header => @secret) if @secret_header.present? && @secret && !@secret.empty?
    request
  end
end
```

**配置选项**：
- `secret_header`: 用于验证的请求头名称
- `secret`: 验证密钥
- `http_method`: HTTP 方法 (默认 GET)

**工作原理**：
1. 解析 URL 获取站点
2. 使用连接池复用 HTTP 连接
3. 发送请求到缓存服务器 (如 CDN)
4. 可选添加验证头

### 4.6 智能缓存保留

**无本地交互的远程媒体** 可能被优先清理：

```ruby
scope :without_local_interaction, lambda {
  where.not(Favourite.joins(:account).merge(Account.local).where(...).arel.exists)
    .where.not(Bookmark.where(...).arel.exists)
    .where.not(Status.local.where(Status.arel_table[:in_reply_to_id].eq(...)).arel.exists)
    .where.not(Status.local.where(Status.arel_table[:reblog_of_id].eq(...)).arel.exists)
    .where.not(Quote.joins(:status).merge(Status.local).where(...).arel.exists)
    .where.not(Quote.joins(:quoted_status).merge(Status.local).where(...).arel.exists)
}
```

**"无本地交互" 定义**：
- 没有本地用户收藏
- 没有本地用户书签
- 没有本地用户回复
- 没有本地用户转发
- 没有本地用户引用

### 4.7 父级缓存更新

当媒体附件更新时，清除相关状态的缓存：

```ruby
# app/models/media_attachment.rb:300
after_commit :reset_parent_cache, on: :update

def reset_parent_cache
  Rails.cache.delete("v3:statuses/#{status_id}") if status_id.present?
end
```

---

## 5. 完整流程图

### 5.1 本地上传媒体流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           本地上传媒体处理流程                                 │
└─────────────────────────────────────────────────────────────────────────────┘

用户上传文件
     │
     ▼
┌─────────────────┐
│ MediaController │
│   #create       │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────┐
│  MediaAttachment.create!    │
└─────────────┬───────────────┘
              │
              ▼
    ┌─────────────────────┐
    │  before_file_validate │
    │  (Attachmentable)    │
    └──────────┬──────────┘
               │
               ▼
    ┌──────────────────────┐
    │ 1. 检查图片尺寸       │
    │ 2. 修正内容类型       │
    │ 3. 混淆文件名         │
    │ 4. 设置扩展名         │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │ set_type_and_extension│
    │ (判断: 图片/视频/音频) │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │ check_video_dimensions│
    │ (视频参数验证)         │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │   set_processing     │
    │ 图片: complete       │
    │ 视频/音频: queued    │
    └──────────┬───────────┘
               │
               ▼
    ┌─────────────────────────────┐
    │      文件保存到存储          │
    │ (本地文件系统 / S3 / etc)   │
    └──────────┬──────────────────┘
               │
               ├──────────────────┐
               │ 图片(同步)       │ 视频/音频(异步)
               ▼                  ▼
    ┌──────────────────┐   ┌──────────────────────┐
    │  Paperclip 处理   │   │ after_commit 回调    │
    │  (生成缩略图等)   │   │ enqueue_processing   │
    └────────┬─────────┘   └──────────┬───────────┘
             │                          │
             ▼                          ▼
    ┌──────────────────┐   ┌──────────────────────────┐
    │  set_meta        │   │ PostProcessMediaWorker    │
    │  (保存元数据)     │   │   perform_async           │
    └────────┬─────────┘   └──────────┬───────────────┘
             │                          │
             ▼                          ▼
    ┌──────────────────┐   ┌──────────────────────────┐
    │ 返回 200 OK      │   │ 1. 状态: in_progress     │
    │ (处理完成)        │   │ 2. reprocess!(:original) │
    └──────────────────┘   │ 3. 状态: complete        │
                           │ 4. 合并/保存元数据        │
                           └──────────┬───────────────┘
                                      │
                                      ▼
                           ┌──────────────────┐
                           │ API 返回 206      │
                           │ (Partial Content) │
                           └──────────────────┘
```

### 5.2 远程媒体缓存流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           远程媒体缓存流程                                     │
└─────────────────────────────────────────────────────────────────────────────┘

从联邦网络接收媒体 (ActivityPub)
     │
     ▼
┌─────────────────────────────────┐
│  创建 MediaAttachment 记录      │
│  - remote_url = 远程 URL        │
│  - 不立即下载 (download_on_assign: false) │
└─────────────┬───────────────────┘
              │
              ├──────────────────────────────┐
              │ 需要访问时                    │ 定时清理检查
              ▼                               ▼
┌─────────────────────────┐         ┌──────────────────────────┐
│ RedownloadMediaWorker   │         │ MediaAttachmentsVacuum   │
│ 或手动触发 download_*!  │         │   perform                  │
└──────────┬──────────────┘         └──────────┬───────────────┘
           │                                     │
           ▼                                     ▼
┌─────────────────────────┐         ┌──────────────────────────┐
│ 1. 验证 URL (http/https)│         │ 检查保留期:               │
│ 2. 发送 GET 请求         │         │ created_before +         │
│ 3. 检查响应状态 (2xx)    │         │ updated_before           │
│ 4. 限制下载大小          │         └──────────┬───────────────┘
└──────────┬──────────────┘                     │
           │                                     ▼
           ▼                          ┌──────────────────────────┐
┌─────────────────────────┐           │ 超过保留期?              │
│ 保存到本地存储           │           │  + 远程 + 已缓存         │
│ (与本地上传相同处理)     │           └──────────┬───────────────┘
└──────────┬──────────────┘                     │
           │                              ┌──────┴──────┐
           ▼                              │             │
    ┌──────────────┐                    是             否
    │ 标记为 cached │                      │             │
    └──────────────┘                      ▼             │
                                 ┌──────────────────┐   │
                                 │ AttachmentBatch  │   │
                                 │    .clear        │   │
                                 └────────┬─────────┘   │
                                          │             │
                                          ▼             │
                                 ┌──────────────────┐   │
                                 │ 删除本地文件      │   │
                                 │ 保留数据库记录    │   │
                                 │ (可重新下载)      │   │
                                 └──────────────────┘   │
                                                        │
                                                        ▼
                                                 ┌──────────────┐
                                                 │ 保留缓存文件  │
                                                 └──────────────┘
```

### 5.3 缓存失效流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           缓存失效流程                                         │
└─────────────────────────────────────────────────────────────────────────────┘

触发条件:
1. 媒体附件被删除
2. 媒体附件被更新
3. 关联状态被修改

     │
     ▼ (删除场景)
┌─────────────────────────────────┐
│ before_destroy :prepare_cache_bust!│
│ (prepend: true - 最先执行)      │
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│ 1. 检查 cache_buster 是否启用    │
│ 2. 遍历所有附件 (file, thumbnail)│
│ 3. 收集所有样式的 URL            │
│    - :original                   │
│    - :small                      │
│    - 其他定义的样式               │
│ 4. 保存到 @paths_to_cache_bust   │
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│      Paperclip 删除文件          │
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│    after_destroy :bust_cache!   │
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│ CacheBusterWorker.push_bulk     │
│ (批量入队，提高效率)             │
└─────────────┬───────────────────┘
              │
              ▼
┌──────────────────────────────────────────┐
│         CacheBusterWorker#perform         │
└─────────────┬────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────────┐
│ CacheBuster.bust(full_asset_url(path))   │
└─────────────┬────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────────┐
│ 1. 解析 URL 获取 normalized_site          │
│ 2. 从连接池获取 HTTP 客户端                │
│ 3. 构建请求 (可配置方法和验证头)           │
│ 4. 发送请求到缓存服务器 (CDN)              │
└──────────────────────────────────────────┘


========== 更新场景 ==========

触发: after_commit :reset_parent_cache, on: :update

     │
     ▼
┌─────────────────────────────────┐
│ Rails.cache.delete              │
│ "v3:statuses/#{status_id}"      │
└─────────────────────────────────┘

(清除关联状态的 API 响应缓存)
```

---

## 6. 关键设计决策分析

### 6.1 同步 vs 异步处理

**决策**：图片同步处理，视频/音频异步处理

**原因**：
- 图片处理相对快速 (缩略图生成、格式转换)
- 视频处理耗时较长 (编码、码率计算)
- 用户体验：图片可即时返回，视频返回 206 状态码告知处理中

**代码依据**：
```ruby
def delay_processing?
  @delay_processing && larger_media_format?
end
```

### 6.2 透传模式优化

**决策**：符合条件的视频直接复制流，不重新编码

**条件**：
- H.264 视频编码
- AAC 音频编码 (或无音频)
- yuv420p 色彩空间

**优势**：
- 处理速度提升显著
- 保留原始质量
- 降低 CPU 使用率
- 减少能源消耗

### 6.3 两级缓存失效

**主动失效**：
- 删除时通知 CDN
- 更新时清除父级缓存

**被动失效**：
- 定时清理过期缓存
- 基于保留期配置

**设计考虑**：
- 主动失效确保即时一致性
- 被动失效控制存储成本
- 结合使用平衡性能和成本

### 6.4 可重下载的远程缓存

**决策**：远程媒体缓存可清除，保留记录

**实现**：
- `AttachmentBatch#clear`: 清除文件，保留记录
- `remote_url` 字段保留源地址
- `RedownloadMediaWorker` 支持重新下载

**优势**：
- 节省存储空间
- 可按需重新获取
- 联邦网络特性支持

---

## 7. 相关文件索引

### 7.1 核心模型

| 文件路径 | 功能描述 |
|---------|---------|
| `app/models/media_attachment.rb` | 媒体附件核心模型，定义类型、样式、处理器 |
| `app/models/concerns/remotable.rb` | 远程附件下载机制 |
| `app/models/concerns/attachmentable.rb` | 附件验证、文件名处理 |
| `app/models/content_retention_policy.rb` | 内容保留期配置 |

### 7.2 控制器

| 文件路径 | 功能描述 |
|---------|---------|
| `app/controllers/api/v1/media_controller.rb` | 媒体上传 API 入口 |

### 7.3 处理器

| 文件路径 | 功能描述 |
|---------|---------|
| `lib/paperclip/transcoder.rb` | 主视频转码器 |
| `lib/paperclip/gif_transcoder.rb` | GIF 动画转视频 |
| `lib/paperclip/blurhash_transcoder.rb` | Blurhash 生成 |

### 7.4 后台任务

| 文件路径 | 功能描述 |
|---------|---------|
| `app/workers/post_process_media_worker.rb` | 媒体后处理 (转码) |
| `app/workers/redownload_media_worker.rb` | 远程媒体重新下载 |
| `app/workers/cache_buster_worker.rb` | 缓存失效通知 |
| `app/workers/scheduler/vacuum_scheduler.rb` | 定时清理调度 |

### 7.5 清理机制

| 文件路径 | 功能描述 |
|---------|---------|
| `app/lib/vacuum/media_attachments_vacuum.rb` | 媒体附件清理逻辑 |
| `app/lib/cache_buster.rb` | CDN 缓存失效客户端 |
| `app/controllers/concerns/cache_concern.rb` | 控制器缓存辅助 |

---

## 8. 配置要点

### 8.1 环境变量/设置

| 设置项 | 类型 | 描述 |
|-------|------|------|
| `Setting.media_cache_retention_period` | 整数 | 媒体缓存保留天数 |
| `Setting.content_cache_retention_period` | 整数 | 内容缓存保留天数 |
| `Rails.configuration.x.cache_buster.enabled` | 布尔 | 是否启用缓存失效 |
| `Rails.configuration.x.cache_buster.secret_header` | 字符串 | 缓存验证头名称 |
| `Rails.configuration.x.cache_buster.secret` | 字符串 | 缓存验证密钥 |
| `Rails.configuration.x.cache_buster.http_method` | 字符串 | 缓存失效 HTTP 方法 |
| `Rails.configuration.x.ffmpeg_binary` | 字符串 | FFmpeg 可执行文件路径 |

### 8.2 硬编码限制

| 限制项 | 值 | 说明 |
|-------|-----|------|
| 图片文件大小 | 16MB | `IMAGE_LIMIT` |
| 视频文件大小 | 99MB | `VIDEO_LIMIT` |
| 最大视频分辨率 | 3840x2160px | `MAX_VIDEO_MATRIX_LIMIT` |
| 最大帧率 | 120fps | `MAX_VIDEO_FRAME_RATE` |
| 最大帧数 | 36,000 | 约 5 分钟 @ 120fps |
| 孤立记录 TTL | 1 天 | `Vacuum::MediaAttachmentsVacuum::TTL` |
| 每像素比特数 | 0.11 | H.264 High 码率计算 |

---

## 9. 错误处理策略

### 9.1 上传阶段

| 错误类型 | HTTP 状态码 | 处理方式 |
|---------|------------|---------|
| Paperclip::Errors::NotIdentifiedByImageMagickError | 422 | "File type of uploaded media could not be verified" |
| Paperclip::Error | 500 | "Error processing thumbnail for uploaded media" |
| 视频无视频流 | 422 | `Mastodon::StreamValidationError` |
| 视频尺寸超限 | 422 | `Mastodon::DimensionsValidationError` |
| 视频帧率超限 | 422 | `Mastodon::DimensionsValidationError` |

### 9.2 后台处理阶段

| Worker | 重试次数 | 失败处理 |
|--------|---------|---------|
| PostProcessMediaWorker | 1 | 标记 `processing: :failed`，记录日志 |
| RedownloadMediaWorker | 3 (指数退避) | 可恢复错误继续重试，否则放弃 |
| CacheBusterWorker | 默认 | 依赖 Sidekiq 默认策略 |
| VacuumScheduler | 0 | 单实例执行，锁 TTL 1 天 |

### 9.3 远程下载阶段

| 错误类型 | 处理方式 |
|---------|---------|
| 非 2xx 响应 | 抛出 `UnexpectedResponseError` |
| 网络错误 | 可配置是否静默失败 |
| 文件类型错误 | 静默失败 (suppress_errors: true) |
| 尺寸超限 | 静默失败 |

---

## 10. 性能优化建议

### 10.1 转码优化

1. **优先使用透传模式**：
   - 确保上传视频符合 H.264/AAC/yuv420p 规范
   - 可显著减少处理时间和 CPU 消耗

2. **调整 FFmpeg 预设**：
   - 当前使用 `preset: veryfast`
   - 可根据服务器负载调整为 `fast` 或 `medium`

3. **硬件加速**：
   - 考虑配置 VA-API 或 NVENC 硬件编码
   - 需要修改转码器代码支持

### 10.2 缓存优化

1. **CDN 配置**：
   - 配置合理的缓存头 (Cache-Control)
   - 使用 CacheBuster 机制确保更新及时生效

2. **保留期策略**：
   - 根据存储空间和访问模式调整 `media_cache_retention_period`
   - 热门实例可适当延长保留期

3. **预取策略**：
   - 对于已知热门远程媒体，可主动触发 `RedownloadMediaWorker`

### 10.3 数据库优化

1. **索引优化**：
   - 确保 `remote_url`、`status_id`、`created_at`、`updated_at` 有索引
   - Vacuum 查询频繁使用这些字段

2. **批量操作**：
   - 清理使用 `find_in_batches` 避免内存压力
   - 缓存失效使用 `push_bulk` 提高入队效率

---

## 11. 安全考虑

### 11.1 输入验证

- **文件类型验证**：多层验证 (MIME 类型 + `file` 命令 + Paperclip)
- **尺寸限制**：防止 DoS 攻击 (超大图片/视频)
- **文件名混淆**：防止路径遍历和信息泄露

### 11.2 远程资源

- **URL 验证**：仅允许 http/https 协议
- **主机验证**：防止 SSRF 攻击
- **下载限制**：限制文件大小，防止磁盘耗尽

### 11.3 元数据处理

- **移除元数据**：`map_metadata: '-1'` 移除视频元数据
- **保护隐私**：防止 GPS 坐标等敏感信息泄露

---

## 总结

Mastodon 的媒体处理流水线设计体现了以下特点：

1. **分层处理**：同步处理快速任务，异步处理耗时任务
2. **智能转码**：支持透传模式，条件性重新编码
3. **两级缓存**：主动失效 + 定时清理，平衡一致性和成本
4. **联邦友好**：远程媒体可缓存、可清除、可重下载
5. **安全可靠**：多层验证、错误处理完善

该设计既考虑了用户体验 (快速响应、渐进式播放)，也考虑了运营成本 (存储优化、CPU 优化)，同时保持了联邦社交网络的特性 (远程媒体处理)。
