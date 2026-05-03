# Mastodon 媒体处理流水线分析报告

## 1. 概述

本文档分析 Mastodon 社交媒体平台中媒体文件从上传、转码到远端缓存的完整处理流程，重点关注任务分派机制、格式转换策略和缓存失效处理。

## 2. 媒体上传流程

### 2.1 上传入口

媒体上传主要通过 `Api::V2::MediaController` 处理：

- **文件**: `app/controllers/api/v2/media_controller.rb`
- **核心方法**: `create`

```ruby
def create
  @media_attachment = current_account.media_attachments.create!(media_and_delay_params)
  render json: @media_attachment, serializer: REST::MediaAttachmentSerializer, status: status_from_media_processing
end

private

def media_and_delay_params
  { delay_processing: true }.merge(media_attachment_params)
end
```

### 2.2 媒体附件模型

核心模型为 `MediaAttachment`，定义于 `app/models/media_attachment.rb`。

#### 2.2.1 媒体类型枚举

```ruby
enum :type, { image: 0, gifv: 1, video: 2, unknown: 3, audio: 4 }
```

#### 2.2.2 处理状态枚举

```ruby
enum :processing, { queued: 0, in_progress: 1, complete: 2, failed: 3 }, prefix: true
```

#### 2.2.3 文件大小限制

- **图片**: 16MB (`IMAGE_LIMIT = 16.megabytes`)
- **视频/音频**: 99MB (`VIDEO_LIMIT = 99.megabytes`)

#### 2.2.4 支持的文件格式

- **图片格式**: `.jpg`, `.jpeg`, `.png`, `.gif`, `.webp`, `.heic`, `.heif`, `.avif`
- **视频格式**: `.webm`, `.mp4`, `.m4v`, `.mov`
- **音频格式**: `.ogg`, `.oga`, `.mp3`, `.wav`, `.flac`, `.opus`, `.aac`, `.m4a`, `.3gp`, `.wma`

### 2.3 上传验证流程

验证通过 `Attachmentable` 模块（`app/models/concerns/attachmentable.rb`）实现，在 `before_validate` 回调中执行：

1. **尺寸检查** (`check_image_dimension`):
   - GIF 最大尺寸: 1280x720px (921,600 像素)
   - 普通图片最大尺寸: 7680x4320px (33,177,600 像素)

2. **内容类型修正** (`set_file_content_type`):
   - 修正浏览器或 kt-paperclip 报告的错误 MIME 类型
   - 受影响类型: `audio/vorbis`, `audio/opus`, `video/ogg`, `video/webm`

3. **文件名混淆** (`obfuscate_file_name`):
   - 使用 `SecureRandom.hex(8)` 生成随机文件名
   - 保留原始文件扩展名

4. **扩展名标准化** (`set_file_extension`):
   - 根据 MIME 类型确定正确的文件扩展名
   - 特殊处理: `jpe`, `jfif` 统一为 `jpeg`

### 2.4 视频维度验证

在 `media_attachment.rb` 的 `check_video_dimensions` 方法中进行额外验证：

```ruby
def check_video_dimensions
  return unless (video? || gifv?) && file.queued_for_write[:original].present?

  movie = ffmpeg_data(file.queued_for_write[:original].path)
  
  # 验证视频流存在
  raise Mastodon::StreamValidationError, 'Video has no video stream' if movie.width.nil? || movie.frame_rate.nil?
  
  # 验证分辨率: 最大 3840x2160px (8,294,400 像素)
  raise Mastodon::DimensionsValidationError, "#{movie.width}x#{movie.height} videos are not supported" if movie.width * movie.height > MAX_VIDEO_MATRIX_LIMIT
  
  # 验证帧率: 最大 120fps
  raise Mastodon::DimensionsValidationError, "#{movie.frame_rate.floor}fps videos are not supported" if movie.frame_rate.floor > MAX_VIDEO_FRAME_RATE
end
```

## 3. 任务分派机制

### 3.1 延迟处理策略

Mastodon 采用同步+异步混合处理策略：

#### 3.1.1 延迟处理条件

```ruby
def delay_processing?
  @delay_processing && larger_media_format?
end

def larger_media_format?
  video? || gifv? || audio?
end
```

- **图片**: 同步处理（即时生成缩略图）
- **视频/GIFV/音频**: 延迟处理（异步转码）

#### 3.1.2 Paperclip 集成

通过 `Paperclip::AttachmentExtensions` 模块（`lib/paperclip/attachment_extensions.rb`）实现延迟处理：

```ruby
def process_style?(style_name, style_args)
  if style_name == :original && instance.respond_to?(:delay_processing_for_attachment?) && instance.delay_processing_for_attachment?(name)
    false  # 跳过 original 样式的同步处理
  else
    style_args.empty? || style_args.include?(style_name)
  end
end
```

### 3.2 任务入队机制

#### 3.2.1 入队触发点

在 `MediaAttachment` 模型的 `after_commit` 回调中：

```ruby
after_commit :enqueue_processing, on: :create

def enqueue_processing
  PostProcessMediaWorker.perform_async(id) if delay_processing?
end
```

#### 3.2.2 处理状态管理

```ruby
def set_processing
  self.processing = delay_processing? ? :queued : :complete
end
```

### 3.3 任务执行器: PostProcessMediaWorker

**文件**: `app/workers/post_process_media_worker.rb`

#### 3.3.1 Worker 配置

```ruby
sidekiq_options retry: 1, dead: false
```

- **重试次数**: 仅重试 1 次
- **死信队列**: 不进入死信队列

#### 3.3.2 处理流程

```ruby
def perform(media_attachment_id)
  media_attachment = MediaAttachment.find(media_attachment_id)
  media_attachment.processing = :in_progress
  media_attachment.save

  # 保存原有元数据
  previous_meta = media_attachment.file_meta

  # 重新处理 original 样式
  media_attachment.file.reprocess!(:original)
  
  # 更新状态并合并元数据
  media_attachment.processing = :complete
  media_attachment.file_meta = previous_meta.merge(media_attachment.file_meta).with_indifferent_access.slice(*MediaAttachment::META_KEYS)
  media_attachment.save
end
```

#### 3.3.3 失败处理

```ruby
sidekiq_retries_exhausted do |msg|
  media_attachment_id = msg['args'].first
  
  ActiveRecord::Base.connection_pool.with_connection do
    media_attachment = MediaAttachment.find(media_attachment_id)
    media_attachment.processing = :failed
    media_attachment.save
  rescue ActiveRecord::RecordNotFound
    true
  end
  
  Sidekiq.logger.error("Processing media attachment #{media_attachment_id} failed with #{msg['error_message']}")
end
```

## 4. 格式转换流程

### 4.1 样式配置系统

`MediaAttachment` 模型根据文件类型定义不同的处理样式：

#### 4.1.1 图片样式

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
}.freeze
```

#### 4.1.2 可转换图片样式 (HEIC/HEIF/AVIF)

```ruby
IMAGE_CONVERTED_STYLES = {
  original: {
    format: 'jpeg',
    content_type: 'image/jpeg',
  }.merge(IMAGE_STYLES[:original]).freeze,

  small: {
    format: 'jpeg',
  }.merge(IMAGE_STYLES[:small]).freeze,
}.freeze
```

#### 4.1.3 视频样式

```ruby
VIDEO_STYLES = {
  small: {
    convert_options: {
      output: {
        'loglevel' => 'fatal',
        :vf => 'scale=\'min(640\, iw):min(640\, ih)\':force_original_aspect_ratio=decrease',
      }.freeze,
    }.freeze,
    format: 'png',
    time: 0,
    file_geometry_parser: FastGeometryParser,
    blurhash: BLURHASH_OPTIONS,
  }.freeze,

  original: VIDEO_FORMAT.merge(passthrough_options: VIDEO_PASSTHROUGH_OPTIONS).freeze,
}.freeze
```

#### 4.1.4 音频样式

```ruby
AUDIO_STYLES = {
  original: {
    format: 'mp3',
    content_type: 'audio/mpeg',
    convert_options: {
      output: {
        'loglevel' => 'fatal',
        'q:a' => 2,
      }.freeze,
    }.freeze,
  }.freeze,
}.freeze
```

### 4.2 样式选择逻辑

```ruby
def self.file_styles(attachment)
  if attachment.instance.file_content_type == 'image/gif' || VIDEO_CONVERTIBLE_MIME_TYPES.include?(attachment.instance.file_content_type)
    VIDEO_CONVERTED_STYLES  # GIF 转换为 MP4 (GIFV)
  elsif IMAGE_CONVERTIBLE_MIME_TYPES.include?(attachment.instance.file_content_type)
    IMAGE_CONVERTED_STYLES   # HEIC/HEIF/AVIF 转换为 JPEG
  elsif IMAGE_MIME_TYPES.include?(attachment.instance.file_content_type)
    IMAGE_STYLES              # 普通图片
  elsif VIDEO_MIME_TYPES.include?(attachment.instance.file_content_type)
    VIDEO_STYLES              # 视频
  else
    AUDIO_STYLES              # 音频
  end
end
```

### 4.3 处理器选择逻辑

```ruby
def self.file_processors(instance)
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

### 4.4 核心转码器: Paperclip::Transcoder

**文件**: `lib/paperclip/transcoder.rb`

#### 4.4.1 视频格式配置

```ruby
VIDEO_FORMAT = {
  format: 'mp4',
  content_type: 'video/mp4',
  vfr_frame_rate_threshold: MAX_VIDEO_FRAME_RATE, # 120fps
  convert_options: {
    output: {
      'loglevel' => 'fatal',
      'preset' => 'veryfast',
      'movflags' => 'faststart',      # 元数据前移，支持流式播放
      'pix_fmt' => 'yuv420p',         # 跨浏览器兼容的颜色空间
      'vf' => 'crop=floor(iw/2)*2:floor(ih/2)*2',  # 确保宽高为偶数
      'c:v' => 'h264',                 # 视频编码器
      'c:a' => 'aac',                   # 音频编码器
      'b:a' => '192k',                  # 音频比特率
      'map_metadata' => '-1',           # 移除元数据
      'frames:v' => MAX_VIDEO_FRAMES,   # 最大帧数 (36,000)
    }.freeze,
  }.freeze,
}.freeze
```

#### 4.4.2 直通 (Passthrough) 选项

当源视频已符合要求时，跳过重新编码：

```ruby
VIDEO_PASSTHROUGH_OPTIONS = {
  video_codecs: ['h264'].freeze,
  audio_codecs: ['aac', nil].freeze,
  colorspaces: ['yuv420p', 'yuvj420p'].freeze,
  options: {
    format: 'mp4',
    convert_options: {
      output: {
        'loglevel' => 'fatal',
        'map_metadata' => '-1',
        'movflags' => 'faststart',
        'c:v' => 'copy',    # 直接复制视频流
        'c:a' => 'copy',    # 直接复制音频流
      }.freeze,
    }.freeze,
  }.freeze,
}.freeze
```

#### 4.4.3 直通条件判断

```ruby
def eligible_to_passthrough?(metadata)
  @passthrough_options && 
    @passthrough_options[:video_codecs].include?(metadata.video_codec) && 
    @passthrough_options[:audio_codecs].include?(metadata.audio_codec) && 
    @passthrough_options[:colorspaces].include?(metadata.colorspace)
end
```

#### 4.4.4 动态比特率计算

当需要重新编码时，根据视频尺寸和时长计算合适的比特率：

```ruby
unless eligible_to_passthrough?(metadata)
  size_limit_in_bits = MediaAttachment::VIDEO_LIMIT * 8  # 99MB * 8 = 792Mb
  desired_bitrate = (metadata.width * metadata.height * 30 * BITS_PER_PIXEL).floor  # BITS_PER_PIXEL = 0.11
  duration = [metadata.duration, 1].max
  maximum_bitrate = (size_limit_in_bits / duration).floor - 192_000  # 预留音频空间
  bitrate = [desired_bitrate, maximum_bitrate].min

  @output_options['b:v']     = bitrate
  @output_options['maxrate'] = bitrate + 192_000
  @output_options['bufsize'] = bitrate * 5
end
```

#### 4.4.5 高帧率视频处理

```ruby
def high_vfr?(metadata)
  @vfr_threshold && metadata.r_frame_rate && metadata.r_frame_rate > @vfr_threshold
end

# 处理逻辑
@output_options['fps_mode'] = 'vfr' if high_vfr?(metadata)
```

#### 4.4.6 GIFV 类型检测

```ruby
def update_attachment_type(metadata)
  @attachment.instance.type = MediaAttachment.types[:gifv] unless metadata.audio_codec
end
```

**策略**: 无音频流的视频被标记为 `gifv` 类型。

#### 4.4.7 FFmpeg 命令构建

```ruby
def prepare_command(destination)
  command_arguments  = ['-nostdin']
  interpolations     = {}
  interpolation_keys = 0

  # 输入选项
  @input_options.each_pair do |key, value|
    interpolation_key = interpolation_keys
    command_arguments << "-#{key} :#{interpolation_key}"
    interpolations[interpolation_key] = value
    interpolation_keys += 1
  end

  command_arguments << '-i :source'
  interpolations[:source] = @file.path

  # 输出选项
  @output_options.each_pair do |key, value|
    interpolation_key = interpolation_keys
    command_arguments << "-#{key} :#{interpolation_key}"
    interpolations[interpolation_key] = value
    interpolation_keys += 1
  end

  command_arguments << '-y :destination'
  interpolations[:destination] = destination.path

  [command_arguments, interpolations]
end
```

## 5. 远端缓存机制

### 5.1 远程媒体标识

```ruby
scope :local, -> { where(remote_url: '') }
scope :remote, -> { where.not(remote_url: '') }
scope :cached, -> { remote.where.not(file_file_name: nil) }
```

- **local**: `remote_url` 为空，表示本地上传的媒体
- **remote**: `remote_url` 非空，表示来自其他实例的媒体
- **cached**: 远程媒体且已下载到本地

### 5.2 远程附件模块: Remotable

**文件**: `app/models/concerns/remotable.rb`

#### 5.2.1 核心方法定义

```ruby
def self.remotable_attachment(attachment_name, limit, suppress_errors: true, download_on_assign: true, attribute_name: nil)
  attribute_name ||= :"#{attachment_name}_remote_url"

  # 下载方法
  define_method(:"download_#{attachment_name}!") do |url = nil|
    url ||= self[attribute_name]
    return if url.blank?

    begin
      parsed_url = Addressable::URI.parse(url).normalize
    rescue Addressable::URI::InvalidURIError
      return
    end

    return if !%w(http https).include?(parsed_url.scheme) || parsed_url.host.blank?

    begin
      Request.new(:get, url).perform do |response|
        raise Mastodon::UnexpectedResponseError, response unless (200...300).cover?(response.code)
        public_send(:"#{attachment_name}=", ResponseWithLimit.new(response, limit))
      end
    rescue Mastodon::UnexpectedResponseError, *Mastodon::HTTP_CONNECTION_ERRORS => e
      Rails.logger.debug { "Error fetching remote #{attachment_name}: #{e}" }
      public_send(:"#{attachment_name}=", nil) if public_send(:"#{attachment_name}_file_name").present?
      raise e unless suppress_errors
    rescue Paperclip::Errors::NotIdentifiedByImageMagickError, ... => e
      Rails.logger.debug { "Error fetching remote #{attachment_name}: #{e}" }
      public_send(:"#{attachment_name}=", nil) if public_send(:"#{attachment_name}_file_name").present?
    end
  end

  # 赋值时自动下载
  define_method(:"#{attribute_name}=") do |url|
    return if self[attribute_name] == url && public_send(:"#{attachment_name}_file_name").present?
    self[attribute_name] = url if has_attribute?(attribute_name)
    public_send(:"download_#{attachment_name}!", url) if download_on_assign
  end
end
```

### 5.3 媒体附件中的远程配置

```ruby
remotable_attachment :file, VIDEO_LIMIT, suppress_errors: false, download_on_assign: false, attribute_name: :remote_url
remotable_attachment :thumbnail, IMAGE_LIMIT, suppress_errors: true, download_on_assign: false
```

**配置说明**:
- `download_on_assign: false`: 赋值时不自动下载，需要手动触发
- `suppress_errors: false` (file): 下载失败时抛出异常
- `suppress_errors: true` (thumbnail): 下载失败时静默处理

### 5.4 重新下载 Worker: RedownloadMediaWorker

**文件**: `app/workers/redownload_media_worker.rb`

```ruby
class RedownloadMediaWorker
  include Sidekiq::Worker
  include ExponentialBackoff
  include JsonLdHelper

  sidekiq_options queue: 'pull', retry: 3

  def perform(id)
    media_attachment = MediaAttachment.find(id)
    return if media_attachment.remote_url.blank?

    media_attachment.download_file!
    media_attachment.download_thumbnail!
    media_attachment.save
  rescue ActiveRecord::RecordNotFound
    # Do nothing
  rescue Mastodon::UnexpectedResponseError => e
    response = e.response
    raise(e) unless response_error_unsalvageable?(response)
  end
end
```

**特性**:
- **队列**: `pull` 队列
- **重试**: 3 次，使用指数退避策略
- **处理内容**: 同时下载文件和缩略图

### 5.5 缓存清理机制

#### 5.5.1 清理服务: MediaAttachmentsVacuum

**文件**: `app/lib/vacuum/media_attachments_vacuum.rb`

```ruby
class Vacuum::MediaAttachmentsVacuum
  TTL = 1.day.freeze

  def initialize(retention_period)
    @retention_period = retention_period
  end

  def perform
    vacuum_orphaned_records!
    vacuum_cached_files! if retention_period?
  end

  private

  def vacuum_cached_files!
    media_attachments_past_retention_period.find_in_batches do |media_attachments|
      AttachmentBatch.new(MediaAttachment, media_attachments).clear
    rescue => e
      Rails.logger.error("Skipping batch while removing cached media attachments due to error: #{e}")
    end
  end

  def vacuum_orphaned_records!
    orphaned_media_attachments.find_in_batches do |media_attachments|
      AttachmentBatch.new(MediaAttachment, media_attachments).delete
    rescue => e
      Rails.logger.error("Skipping batch while removing orphaned media attachments due to error: #{e}")
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
      .created_before(TTL.ago)
  end
end
```

#### 5.5.2 清理策略

| 类型 | 条件 | 操作 |
|------|------|------|
| 过期缓存 | 远程 + 已缓存 + 创建时间早于保留期 + 更新时间早于保留期 | 清除文件 (clear) |
| 孤立记录 | 未关联任何状态 + 创建时间早于 1 天前 | 删除记录 (delete) |

#### 5.5.3 无本地交互的远程媒体

```ruby
scope :without_local_interaction, lambda {
  where.not(Favourite.joins(:account).merge(Account.local).where(...).select(1).arel.exists)
    .where.not(Bookmark.where(...).select(1).arel.exists)
    .where.not(Status.local.where(Status.arel_table[:in_reply_to_id].eq(...)).select(1).arel.exists)
    .where.not(Status.local.where(Status.arel_table[:reblog_of_id].eq(...)).select(1).arel.exists)
    .where.not(Quote.joins(:status).merge(Status.local).where(...).select(1).arel.exists)
    .where.not(Quote.joins(:quoted_status).merge(Status.local).where(...).select(1).arel.exists)
}
```

**用途**: 用于识别可以安全清理的远程媒体（本地用户未交互过）。

## 6. 缓存失效处理策略

### 6.1 缓存失效触发点

#### 6.1.1 媒体删除时的缓存失效

在 `MediaAttachment` 模型中定义：

```ruby
before_destroy :prepare_cache_bust!, prepend: true
after_destroy :bust_cache!
```

#### 6.1.2 媒体更新时的父缓存重置

```ruby
after_commit :reset_parent_cache, on: :update

def reset_parent_cache
  Rails.cache.delete("v3:statuses/#{status_id}") if status_id.present?
end
```

### 6.2 缓存键预收集

```ruby
def prepare_cache_bust!
  return unless Rails.configuration.x.cache_buster.enabled

  @paths_to_cache_bust = MediaAttachment.attachment_definitions.keys.flat_map do |attachment_name|
    attachment = public_send(attachment_name)
    next if attachment.blank?

    styles = DEFAULT_STYLES | attachment.styles.keys
    styles.map { |style| attachment.url(style) }
  end.compact
rescue => e
  Rails.logger.warn "Error #{e.class} busting cache: #{e.message}"
end
```

**设计要点**:
- 在文件删除**之前**收集 URL（删除后无法获取）
- 收集所有样式的 URL（original, small 等）
- 异常处理：不阻止媒体删除操作

### 6.3 缓存失效执行

```ruby
def bust_cache!
  return unless Rails.configuration.x.cache_buster.enabled

  CacheBusterWorker.push_bulk(@paths_to_cache_bust) { |path| [path] }
rescue => e
  Rails.logger.warn "Error #{e.class} busting cache: #{e.message}"
end
```

### 6.4 缓存失效 Worker: CacheBusterWorker

**文件**: `app/workers/cache_buster_worker.rb`

```ruby
class CacheBusterWorker
  include Sidekiq::Worker
  include RoutingHelper

  sidekiq_options queue: 'pull'

  def perform(path)
    cache_buster.bust(full_asset_url(path))
  end

  private

  def cache_buster
    CacheBuster.new(Rails.configuration.x.cache_buster)
  end
end
```

### 6.5 缓存失效器: CacheBuster

**文件**: `app/lib/cache_buster.rb`

```ruby
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

  def request_pool
    RequestPool.current
  end

  def build_request(url, http_client)
    request = Request.new(@http_method.downcase.to_sym, url, http_client: http_client)
    request.add_headers(@secret_header => @secret) if @secret_header.present? && @secret && !@secret.empty?
    request
  end
end
```

### 6.6 配置方式

**文件**: `config/application.rb`

```ruby
config.x.cache_buster = config_for(:cache_buster)
```

**预期配置文件**: `config/cache_buster.yml`

**配置选项**:
- `secret_header`: 认证头名称（可选）
- `secret`: 认证密钥（可选）
- `http_method`: HTTP 方法（默认 GET）

## 7. 完整流程图

### 7.1 本地上传媒体流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         本地媒体上传处理流程                                   │
└─────────────────────────────────────────────────────────────────────────────┘

1. 用户上传
   │
   ▼
2. Api::V2::MediaController#create
   ├── 创建 MediaAttachment 记录
   ├── 设置 delay_processing: true
   └── 验证通过后返回 202 Accepted (未处理) 或 200 OK (已处理)
   │
   ▼
3. Attachmentable 验证 (before_validate)
   ├── check_image_dimension: 验证尺寸限制
   ├── set_file_content_type: 修正 MIME 类型
   ├── obfuscate_file_name: 混淆文件名
   └── set_file_extension: 标准化扩展名
   │
   ▼
4. Paperclip 处理
   ├── 图片: 同步处理所有样式 (original + small)
   └── 视频/音频/GIF: 仅处理 small 样式 (缩略图)，original 样式延迟
   │
   ▼
5. after_commit: enqueue_processing
   └── PostProcessMediaWorker.perform_async(id)  [仅针对延迟处理的媒体]
   │
   ▼
6. PostProcessMediaWorker 执行
   ├── 状态: queued → in_progress
   ├── file.reprocess!(:original)
   │   └── Paperclip::Transcoder (FFmpeg)
   │       ├── 检测元数据
   │       ├── 判断是否可直通 (passthrough)
   │       ├── 计算比特率
   │       └── 执行转码
   ├── 状态: in_progress → complete
   └── 保存元数据
   │
   ▼
7. 完成
   └── 媒体可访问
```

### 7.2 远程媒体缓存流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         远程媒体缓存处理流程                                   │
└─────────────────────────────────────────────────────────────────────────────┘

1. 接收联邦媒体 (ActivityPub)
   │
   ▼
2. 解析 MediaAttachment
   ├── 设置 remote_url
   └── download_on_assign: false (不自动下载)
   │
   ▼
3. 按需下载
   ├── 方式 A: RedownloadMediaWorker (后台任务)
   │   └── 通常由 ActivityPub 处理触发
   │
   └── 方式 B: 访问时触发
       └── 需要时调用 download_file!
   │
   ▼
4. Remotable#download_#{name}!
   ├── 解析 URL (HTTP/HTTPS 验证)
   ├── Request.get 获取响应
   ├── ResponseWithLimit 限制大小
   └── 赋值给 attachment 触发 Paperclip 处理
   │
   ▼
5. 缓存管理
   │
   ├── 读取: cached scope (remote + file_file_name NOT NULL)
   │
   └── 清理: Vacuum::MediaAttachmentsVacuum
       ├── 过期缓存: 超过保留期的远程缓存
       └── 孤立记录: 超过 1 天未关联的媒体
```

### 7.3 缓存失效流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           缓存失效处理流程                                     │
└─────────────────────────────────────────────────────────────────────────────┘

场景 A: 媒体删除

1. before_destroy :prepare_cache_bust!
   ├── 检查 cache_buster.enabled
   ├── 遍历所有附件 (file, thumbnail)
   ├── 收集所有样式的 URL
   │   ├── DEFAULT_STYLES = [:original]
   │   └── 合并 attachment.styles.keys
   └── 保存到 @paths_to_cache_bust
   │
   ▼
2. Paperclip 删除文件
   │
   ▼
3. after_destroy :bust_cache!
   └── CacheBusterWorker.push_bulk(@paths_to_cache_bust)
       │
       ▼
4. CacheBusterWorker#perform
   └── CacheBuster#bust(full_asset_url(path))
       │
       ▼
5. CacheBuster#bust
   ├── 解析 URL 获取 site
   ├── RequestPool 复用连接
   ├── 构建请求 (可配置 HTTP 方法和认证头)
   └── 发送请求通知 CDN/缓存服务器

场景 B: 媒体更新

1. after_commit :reset_parent_cache, on: :update
   └── Rails.cache.delete("v3:statuses/#{status_id}")
```

## 8. 关键数据结构

### 8.1 MediaAttachment 数据表字段

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| type | integer | 媒体类型 (image/gifv/video/unknown/audio) |
| processing | integer | 处理状态 (queued/in_progress/complete/failed) |
| remote_url | string | 远程 URL，空表示本地媒体 |
| shortcode | string | 短码 (19 字符) |
| account_id | bigint | 所属账户 |
| status_id | bigint | 关联的状态 |
| scheduled_status_id | bigint | 关联的定时状态 |
| file_* | 多种 | Paperclip 文件字段 (file_name, content_type, file_size, updated_at, meta, storage_schema_version) |
| thumbnail_* | 多种 | 缩略图字段 (同 file) |
| blurhash | string | 模糊哈希值 |
| description | text | 媒体描述 |
| created_at | datetime | 创建时间 |
| updated_at | datetime | 更新时间 |

### 8.2 元数据结构 (file_meta)

```ruby
META_KEYS = %i(
  focus    # 焦点坐标 { x:, y: }
  colors   # 颜色信息
  original # 原始尺寸信息
  small    # 缩略图尺寸信息
).freeze
```

**图片元数据示例**:
```ruby
{
  width: 1920,
  height: 1080,
  size: "1920x1080",
  aspect: 1.7777777777777777
}
```

**视频元数据示例**:
```ruby
{
  width: 1920,
  height: 1080,
  frame_rate: 30.0,
  duration: 10.5,
  bitrate: 5000000
}
```

## 9. 关键设计决策分析

### 9.1 延迟处理策略

**决策**: 视频/音频/GIF 采用异步处理，图片同步处理

**理由**:
1. **用户体验**: 图片处理快，可即时返回；视频处理慢，异步避免用户等待
2. **资源隔离**: 耗时操作放入后台队列，不阻塞 Web 请求
3. **失败隔离**: 转码失败不影响上传流程，可单独处理

### 9.2 直通 (Passthrough) 优化

**决策**: 符合条件的视频直接复制流，不重新编码

**条件**:
- 视频编码: H.264
- 音频编码: AAC 或无音频
- 颜色空间: yuv420p 或 yuvj420p

**收益**:
- 减少 CPU 使用率
- 缩短处理时间
- 避免质量损失

### 9.3 动态比特率计算

**决策**: 根据视频尺寸和时长动态计算比特率

**公式**:
```
desired_bitrate = width * height * 30 * 0.11
maximum_bitrate = (99MB * 8 / duration) - 192k (音频预留)
bitrate = min(desired_bitrate, maximum_bitrate)
```

**收益**:
- 确保输出文件不超过 99MB 限制
- 根据内容复杂度分配合理比特率
- 平衡质量和文件大小

### 9.4 缓存失效两阶段提交

**决策**: before_destroy 收集 URL，after_destroy 执行失效

**理由**:
1. **数据一致性**: Paperclip 删除文件后无法获取 URL
2. **原子性**: 确保删除和失效操作关联
3. **容错**: 任何阶段失败都不阻止核心操作（媒体删除）

### 9.5 远程媒体按需缓存

**决策**: download_on_assign: false，不自动下载远程媒体

**理由**:
1. **节省存储空间**: 仅缓存实际需要的媒体
2. **减少网络流量**: 避免批量下载未访问的媒体
3. **联邦隐私**: 减少对其他实例的请求压力

## 10. 总结

Mastodon 的媒体处理流水线是一个设计精良的系统，具有以下特点：

### 10.1 架构优势

1. **分层设计**: 清晰分离上传、处理、缓存、失效各阶段
2. **异步优先**: 耗时操作全部后台化，保证 Web 响应速度
3. **智能优化**: 直通转码、动态比特率等策略平衡性能和质量
4. **容错机制**: 完善的错误处理和重试策略
5. **联邦友好**: 远程媒体按需缓存，减少跨实例流量

### 10.2 核心组件

| 组件 | 职责 | 文件位置 |
|------|------|-----------|
| MediaAttachment | 媒体模型、验证、状态管理 | app/models/media_attachment.rb |
| PostProcessMediaWorker | 异步转码任务 | app/workers/post_process_media_worker.rb |
| Paperclip::Transcoder | FFmpeg 转码封装 | lib/paperclip/transcoder.rb |
| Remotable | 远程附件下载 | app/models/concerns/remotable.rb |
| RedownloadMediaWorker | 远程媒体重新下载 | app/workers/redownload_media_worker.rb |
| CacheBuster | CDN 缓存失效 | app/lib/cache_buster.rb |
| MediaAttachmentsVacuum | 过期缓存清理 | app/lib/vacuum/media_attachments_vacuum.rb |

### 10.3 处理状态流转

```
┌─────────┐     ┌─────────────┐     ┌──────────┐
│ queued  │ ──▶ │ in_progress │ ──▶ │ complete │
└─────────┘     └─────────────┘     └──────────┘
                      │
                      ▼
                 ┌────────┐
                 │ failed │
                 └────────┘
```

- **queued**: 已入队等待处理
- **in_progress**: 正在转码处理
- **complete**: 处理完成
- **failed**: 处理失败（重试耗尽后）

---

*报告生成时间: 2026-05-03*
*分析代码版本: Mastodon (当前工作目录)*
