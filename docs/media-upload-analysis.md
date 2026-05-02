# Mastodon 媒体上传完整链路分析

## 目录

1. [概述](#概述)
2. [客户端上传流程](#客户端上传流程)
3. [服务端转码处理](#服务端转码处理)
4. [存储系统实现](#存储系统实现)
5. [远端实例缓存机制](#远端实例缓存机制)
6. [缓存生命周期管理](#缓存生命周期管理)
7. [总结](#总结)

---

## 概述

Mastodon 的媒体处理系统是一个复杂的多阶段流程，涵盖了从客户端上传、服务端转码、分布式存储，到联邦网络中的缓存同步与生命周期管理的完整链路。本文档基于代码分析，详细解析这一流程的每个环节。

---

## 客户端上传流程

### API 版本差异

Mastodon 提供了两个版本的媒体上传 API，它们在处理方式上有重要差异：

#### API v1 (`/api/v1/media`)

**文件位置**: `app/controllers/api/v1/media_controller.rb`

**核心特性**:
- 同步处理：上传后立即进行转码处理
- 响应状态码：
  - `200 OK` - 处理完成
  - `206 Partial Content` - 仍在处理中（`not_processed?`）
- 适用场景：简单图片上传，即时可用

**关键代码**:
```ruby
# app/controllers/api/v1/media_controller.rb:13-21
def create
  @media_attachment = current_account.media_attachments.create!(media_attachment_params)
  render json: @media_attachment, serializer: REST::MediaAttachmentSerializer
end

def status_code_for_media_attachment
  @media_attachment.not_processed? ? 206 : 200
end
```

#### API v2 (`/api/v2/media`)

**文件位置**: `app/controllers/api/v2/media_controller.rb`

**核心特性**:
- 异步处理：使用 `delay_processing: true` 标志
- 响应状态码：
  - `202 Accepted` - 已接受，正在处理中
  - `200 OK` - 处理完成
- 适用场景：大文件（视频、音频）上传，后台处理

**关键代码**:
```ruby
# app/controllers/api/v2/media_controller.rb:4-22
def create
  @media_attachment = current_account.media_attachments.create!(media_and_delay_params)
  render json: @media_attachment, serializer: REST::MediaAttachmentSerializer, status: status_from_media_processing
end

private

def media_and_delay_params
  { delay_processing: true }.merge(media_attachment_params)
end

def status_from_media_processing
  @media_attachment.not_processed? ? 202 : 200
end
```

### 上传参数

两种 API 都接受以下参数：

| 参数 | 类型 | 说明 |
|------|------|------|
| `file` | File | **必填** 媒体文件本体 |
| `thumbnail` | File | 可选缩略图（用于音频/视频） |
| `description` | String | 媒体描述（无障碍文本，最大 1500 字符） |
| `focus` | String | 焦点坐标，格式为 `"x,y"`，用于智能裁剪 |

### 媒体类型枚举

**文件位置**: `app/models/media_attachment.rb:38-39`

```ruby
enum :type, { image: 0, gifv: 1, video: 2, unknown: 3, audio: 4 }
enum :processing, { queued: 0, in_progress: 1, complete: 2, failed: 3 }, prefix: true
```

### 文件大小限制

**文件位置**: `app/models/media_attachment.rb:46-51`

| 媒体类型 | 大小限制 | 说明 |
|---------|---------|------|
| 图片 | 16 MB | `IMAGE_LIMIT` |
| 视频/音频/GIFV | 99 MB | `VIDEO_LIMIT` |
| 视频分辨率 | 3840×2160 px | `MAX_VIDEO_MATRIX_LIMIT` (8,294,400 像素) |
| 视频帧率 | 120 fps | `MAX_VIDEO_FRAME_RATE` |
| 视频帧数 | 36,000 帧 | 约 5 分钟 @ 120fps |

### 支持的文件格式

**文件位置**: `app/models/media_attachment.rb:53-68`

**图片格式**:
- `IMAGE_MIME_TYPES`: `image/jpeg`, `image/png`, `image/gif`, `image/heic`, `image/heif`, `image/webp`, `image/avif`
- `IMAGE_FILE_EXTENSIONS`: `.jpg`, `.jpeg`, `.png`, `.gif`, `.webp`, `.heic`, `.heif`, `.avif`

**视频格式**:
- `VIDEO_MIME_TYPES`: `video/webm`, `video/mp4`, `video/quicktime`, `video/ogg`
- `VIDEO_FILE_EXTENSIONS`: `.webm`, `.mp4`, `.m4v`, `.mov`

**音频格式**:
- `AUDIO_MIME_TYPES`: 多种音频格式（wav, ogg, mp3, flac, aac, m4a 等）
- `AUDIO_FILE_EXTENSIONS`: `.ogg`, `.oga`, `.mp3`, `.wav`, `.flac`, `.opus`, `.aac`, `.m4a`, `.3gp`, `.wma`

---

## 服务端转码处理

### 处理架构

Mastodon 使用 **Paperclip**（现 kt-paperclip）作为文件处理框架，通过自定义处理器（Processor）实现不同媒体类型的转码逻辑。

### 延迟处理机制

**文件位置**: `app/models/media_attachment.rb:283-396`

当使用 API v2 或上传视频/音频等大文件时，系统采用延迟处理模式：

```ruby
attr_writer :delay_processing

def delay_processing?
  @delay_processing && larger_media_format?
end

def larger_media_format?
  video? || gifv? || audio?
end

def set_processing
  self.processing = delay_processing? ? :queued : :complete
end

after_commit :enqueue_processing, on: :create

def enqueue_processing
  PostProcessMediaWorker.perform_async(id) if delay_processing?
end
```

### 异步处理 Worker

**文件位置**: `app/workers/post_process_media_worker.rb`

```ruby
class PostProcessMediaWorker
  include Sidekiq::Worker
  
  sidekiq_options retry: 1, dead: false

  def perform(media_attachment_id)
    media_attachment = MediaAttachment.find(media_attachment_id)
    media_attachment.processing = :in_progress
    media_attachment.save

    # 保存原有元数据，因为 paperclip-av-transcoder 会覆盖
    previous_meta = media_attachment.file_meta

    media_attachment.file.reprocess!(:original)
    media_attachment.processing = :complete
    media_attachment.file_meta = previous_meta.merge(media_attachment.file_meta).with_indifferent_access.slice(*MediaAttachment::META_KEYS)
    media_attachment.save
  end
end
```

**处理失败处理**:
```ruby
sidekiq_retries_exhausted do |msg|
  media_attachment_id = msg['args'].first
  # 设置 processing = :failed
  media_attachment.processing = :failed
  media_attachment.save
end
```

### 媒体类型处理器链

**文件位置**: `app/models/media_attachment.rb:323-347`

根据媒体类型的不同，系统使用不同的处理器链（Processor Chain）：

```ruby
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

### 处理样式定义

**文件位置**: `app/models/media_attachment.rb:75-174`

不同媒体类型有不同的输出样式（Styles）：

```ruby
# 图片样式
IMAGE_STYLES = {
  original: { pixels: 8_294_400, ... },  # 3840x2160px
  small: { pixels: 230_400, blurhash: ... }  # 640x360px
}

# 视频格式定义
VIDEO_FORMAT = {
  format: 'mp4',
  content_type: 'video/mp4',
  convert_options: {
    output: {
      'preset' => 'veryfast',
      'movflags' => 'faststart',      # 元数据前置，支持流式播放
      'pix_fmt' => 'yuv420p',         # 跨浏览器兼容色彩空间
      'c:v' => 'h264',                 # H.264 视频编码
      'c:a' => 'aac',                  # AAC 音频编码
      'b:a' => '192k',                 # 音频比特率
    }
  }
}

# 音频样式
AUDIO_STYLES = {
  original: {
    format: 'mp3',
    content_type: 'audio/mpeg',
    convert_options: { output: { 'q:a' => 2 } }  # VBR 质量
  }
}
```

### 核心处理器详解

#### 1. GIF 转码器 (`GifTranscoder`)

**文件位置**: `lib/paperclip/gif_transcoder.rb`

**功能**: 
- 检测 GIF 是否为动画（多帧）
- 只有动画 GIF 才会被转码为视频
- 静态 GIF 保持原样

**关键逻辑**:
```ruby
class GifReader
  def self.animated?(path)
    new(path).animated  # 检查是否有超过 1 帧
  end
end

class GifTranscoder < Paperclip::Processor
  def make
    return File.open(@file.path) unless needs_convert?
    
    # 调用视频转码器转换为 MP4
    final_file = Paperclip::Transcoder.make(file, options, attachment)
    
    # 更新元数据：类型变为 gifv
    attachment.instance.type = MediaAttachment.types[:gifv]
    attachment.instance.file_content_type = 'video/mp4'
  end
  
  def needs_convert?
    GifReader.animated?(file.path)  # 只处理动画 GIF
  end
end
```

**注意**: 静态 GIF 不会被转码，保持为 `image` 类型；动画 GIF 会被转为 `gifv` 类型（实质上是 MP4 视频）。

#### 2. 视频转码器 (`Transcoder`)

**文件位置**: `lib/paperclip/transcoder.rb`

**功能**:
- 使用 FFmpeg 进行视频转码
- 支持"透传"（Passthrough）模式：符合条件的视频直接复用流，不重新编码
- 自动调整比特率以适应文件大小限制

**关键特性**:

**透传条件**:
```ruby
def eligible_to_passthrough?(metadata)
  @passthrough_options && 
    @passthrough_options[:video_codecs].include?(metadata.video_codec) &&  # 必须是 h264
    @passthrough_options[:audio_codecs].include?(metadata.audio_codec) &&  # aac 或无音频
    @passthrough_options[:colorspaces].include?(metadata.colorspace)        # yuv420p
end
```

**透传配置** (`app/models/media_attachment.rb:119-135`):
```ruby
VIDEO_PASSTHROUGH_OPTIONS = {
  video_codecs: ['h264'].freeze,
  audio_codecs: ['aac', nil].freeze,    # nil 表示无音频
  colorspaces: ['yuv420p', 'yuvj420p'].freeze,
  options: {
    format: 'mp4',
    convert_options: {
      output: {
        'c:v' => 'copy',    # 视频流直接复制
        'c:a' => 'copy',    # 音频流直接复制
      }
    }
  }
}
```

**智能比特率计算**:
```ruby
unless eligible_to_passthrough?(metadata)
  # BITS_PER_PIXEL = 0.11 (H.264 High 配置的经验值)
  desired_bitrate = (metadata.width * metadata.height * 30 * BITS_PER_PIXEL).floor
  
  # 确保不超过文件大小限制
  size_limit_in_bits = MediaAttachment::VIDEO_LIMIT * 8
  duration = [metadata.duration, 1].max
  maximum_bitrate = (size_limit_in_bits / duration).floor - 192_000  # 预留音频空间
  
  bitrate = [desired_bitrate, maximum_bitrate].min
end
```

**类型修正**:
```ruby
def update_attachment_type(metadata)
  # 如果没有音频流，标记为 gifv 而非 video
  @attachment.instance.type = MediaAttachment.types[:gifv] unless metadata.audio_codec
end
```

#### 3. 图片提取器 (`ImageExtractor`)

**文件位置**: `lib/paperclip/image_extractor.rb`

**功能**: 为音频文件提取封面图（从视频流的第一帧）

```ruby
class ImageExtractor < Paperclip::Processor
  def make
    return @file unless options[:style] == :original
    
    # 使用 FFmpeg 提取第一帧作为 PNG
    image = extract_image_from_file!
    
    unless image.nil?
      attachment.instance.thumbnail = image if image.size.positive?
    end
  end
  
  def extract_image_from_file!
    # ffmpeg -i source -loglevel fatal -y destination.png
    command = Terrapin::CommandLine.new(
      Rails.configuration.x.ffmpeg_binary, 
      '-i :source -loglevel :loglevel -y :destination'
    )
    command.run(source: @file.path, destination: dst.path, loglevel: 'fatal')
  end
end
```

#### 4. 延迟缩略图处理器 (`LazyThumbnail`)

**文件位置**: `lib/paperclip/lazy_thumbnail.rb`

**功能**: 智能调整图片尺寸，只在需要时进行转换

```ruby
def needs_convert?
  needs_different_geometry? ||   # 尺寸不同
    needs_different_format? ||    # 格式不同
    needs_metadata_stripping?     # 本地图片需要剥离元数据
end

def needs_metadata_stripping?
  @attachment.instance.respond_to?(:local?) && @attachment.instance.local?
  # 注意：远程缓存的媒体不会剥离元数据，保持原样
end
```

#### 5. 类型修正器 (`TypeCorrector`)

**文件位置**: `lib/paperclip/type_corrector.rb`

**功能**: 修正文件扩展名和 MIME 类型

```ruby
def make
  return @file unless options[:format]
  
  target_extension = ".#{options[:format]}"
  extension = File.extname(attachment.instance_read(:file_name))
  
  # 仅在 original 样式且扩展名不符时修正
  return @file unless options[:style] == :original && target_extension && extension != target_extension
  
  attachment.instance_write(:content_type, options[:content_type] || ...)
  attachment.instance_write(:file_name, File.basename(..., '.*') + target_extension)
end
```

#### 6. Blurhash 编码器 (`BlurhashTranscoder`)

**文件位置**: `lib/paperclip/blurhash_transcoder.rb`

**功能**: 生成 Blurhash 字符串（用于加载时的占位图）

```ruby
class BlurhashTranscoder < Paperclip::Processor
  def make
    return @file unless options[:style] == :small || options[:blurhash]
    
    width, height, data = blurhash_params
    # Blurhash.encode 生成模糊哈希字符串
    attachment.instance.blurhash = Blurhash.encode(
      width, height, data, 
      **(options[:blurhash] || {})  # 默认 x_comp: 4, y_comp: 4
    )
  end
  
  def blurhash_params
    # 使用 libvips 缩小图片后提取像素数据
    image = Vips::Image.thumbnail(@file.path, 100)
    [image.width, image.height, image.colourspace(:srgb).extract_band(0, n: 3).to_a.flatten]
  end
end
```

#### 7. 颜色提取器 (`ColorExtractor`)

**文件位置**: `lib/paperclip/color_extractor.rb`

**功能**: 从图片中提取主色调（背景色、前景色、强调色）

```ruby
def make
  # 使用 libvips 生成颜色直方图
  background_palette, foreground_palette = palettes_from_libvips
  
  # 计算对比度，选择合适的颜色组合
  # 要求：背景与前景对比度 >= 3.0 (W3C 标准)
  #       背景与强调色对比度 >= 2.0
  
  meta = {
    colors: {
      background: '#xxxxxx',
      foreground: '#xxxxxx',
      accent: '#xxxxxx',
    },
  }
  
  attachment.instance.file.instance_write(:meta, ...)
end
```

### 处理样式选择逻辑

**文件位置**: `app/models/media_attachment.rb:323-335`

```ruby
def file_styles(attachment)
  if attachment.instance.file_content_type == 'image/gif' || VIDEO_CONVERTIBLE_MIME_TYPES.include?(...)
    VIDEO_CONVERTED_STYLES    # 需要转码为 MP4
  elsif IMAGE_CONVERTIBLE_MIME_TYPES.include?(...)  # heic, heif, avif
    IMAGE_CONVERTED_STYLES    # 转换为 JPEG
  elsif IMAGE_MIME_TYPES.include?(...)
    IMAGE_STYLES               # 常规图片处理
  elsif VIDEO_MIME_TYPES.include?(...)
    VIDEO_STYLES               # 视频处理（可能透传）
  else
    AUDIO_STYLES               # 音频转 MP3
  end
end
```

**可转换 MIME 类型**:
```ruby
IMAGE_CONVERTIBLE_MIME_TYPES = %w(image/heic image/heif image/avif).freeze
VIDEO_CONVERTIBLE_MIME_TYPES = %w(video/webm video/quicktime).freeze
```

### 元数据提取

**文件位置**: `app/models/media_attachment.rb:384-425`

处理完成后，系统会提取并存储媒体元数据：

```ruby
def populate_meta
  meta = (file.instance_read(:meta) || {}).with_indifferent_access.slice(*META_KEYS)
  
  file.queued_for_write.each do |style, file|
    meta[style] = style == :small || image? ? 
      image_geometry(file) :    # 图片：宽、高、尺寸、比例
      video_metadata(file)       # 视频：宽、高、帧率、时长、比特率
  end
  
  # 缩略图元数据
  meta[:small] = image_geometry(thumbnail.queued_for_write[:original]) if thumbnail.queued_for_write.key?(:original)
  
  meta
end
```

**元数据键**:
```ruby
META_KEYS = %i(
  focus     # 焦点坐标
  colors    # 提取的颜色
  original  # 原始尺寸信息
  small     # 缩略图尺寸信息
).freeze
```

---

## 存储系统实现

Mastodon 支持多种存储后端，通过 Paperclip 抽象层统一管理。

### 支持的存储类型

**文件位置**: `config/initializers/paperclip.rb`

| 存储类型 | 环境变量开关 | 说明 |
|---------|------------|------|
| **本地文件系统** | 默认 | 存储在 `public/system` 目录 |
| **AWS S3** | `S3_ENABLED=true` | 亚马逊 S3 或兼容服务 |
| **OpenStack Swift** | `SWIFT_ENABLED=true` | OpenStack 对象存储 |
| **Azure Blob** | `AZURE_ENABLED=true` | 微软 Azure 存储 |

### 路径结构

**文件位置**: `config/initializers/paperclip.rb:6-30`

```ruby
PATH = ':prefix_url:class/:attachment/:id_partition/:style/:filename'

# 路径插值：本地媒体 vs 远程缓存
Paperclip.interpolates :prefix_path do |attachment, _style|
  if attachment.storage_schema_version >= 1 && attachment.instance.respond_to?(:local?) && !attachment.instance.local?
    "cache#{File::SEPARATOR}"   # 远程缓存：cache/ 前缀
  else
    ''                            # 本地上传：无前缀
  end
end
```

**实际路径示例**:

| 媒体类型 | 路径 |
|---------|------|
| 本地上传图片 | `media_attachments/files/000/001/234/original/abc.jpg` |
| 远程缓存图片 | `cache/media_attachments/files/000/001/567/original/def.jpg` |

**注意**: `id_partition` 是 Paperclip 的分片机制，将 ID `1234` 转换为 `000/001/234`，避免单目录文件过多。

### S3 配置详解

**文件位置**: `config/initializers/paperclip.rb:38-117`

```ruby
if ENV['S3_ENABLED'] == 'true'
  require 'aws-sdk-s3'
  
  s3_region   = ENV.fetch('S3_REGION')   { 'us-east-1' }
  s3_protocol = ENV.fetch('S3_PROTOCOL') { 'https' }
  s3_hostname = ENV.fetch('S3_HOSTNAME') { "s3-#{s3_region}.amazonaws.com" }
  
  # 可选：路径前缀
  Paperclip::Attachment.default_options[:path] = ENV.fetch('S3_KEY_PREFIX') + "/#{PATH}" if ENV.key?('S3_KEY_PREFIX')
  
  Paperclip::Attachment.default_options.merge!(
    storage: :s3,
    s3_protocol: s3_protocol,
    s3_host_name: s3_hostname,
    
    s3_headers: {
      'X-Amz-Multipart-Threshold' => ENV.fetch('S3_MULTIPART_THRESHOLD') { 15.megabytes }.to_i,
      'Cache-Control' => 'public, max-age=315576000, immutable',  # 10 年缓存
    },
    
    s3_permissions: ENV.fetch('S3_PERMISSION') { 'public-read' },
    s3_region: s3_region,
    
    s3_credentials: {
      bucket: ENV['S3_BUCKET'],
      access_key_id: ENV['AWS_ACCESS_KEY_ID'],
      secret_access_key: ENV['AWS_SECRET_ACCESS_KEY'],
    },
    
    s3_options: {
      signature_version: ENV.fetch('S3_SIGNATURE_VERSION') { 'v4' },
      http_open_timeout: 5,
      http_read_timeout: 5,
      retry_limit: 0,
    }
  )
  
  # 自定义 Endpoint（用于 MinIO、Backblaze 等 S3 兼容服务）
  if ENV.key?('S3_ENDPOINT')
    Paperclip::Attachment.default_options[:s3_options].merge!(
      endpoint: ENV['S3_ENDPOINT'],
      force_path_style: ENV['S3_OVERRIDE_PATH_STYLE'] != 'true'
    )
    Paperclip::Attachment.default_options[:url] = ':s3_path_url'
  end
  
  # CDN 别名（CloudFront 或自定义域名）
  if ENV.key?('S3_ALIAS_HOST') || ENV.key?('S3_CLOUDFRONT_HOST')
    Paperclip::Attachment.default_options.merge!(
      url: ':s3_alias_url',
      s3_host_alias: ENV['S3_ALIAS_HOST'] || ENV['S3_CLOUDFRONT_HOST']
    )
  end
  
  # 可选：存储类别（STANDARD, IA, GLACIER 等）
  Paperclip::Attachment.default_options[:s3_headers]['X-Amz-Storage-Class'] = ENV['S3_STORAGE_CLASS'] if ENV.key?('S3_STORAGE_CLASS')
end
```

### S3 兼容扩展

**文件位置**: `config/initializers/paperclip.rb:98-117`

为了兼容某些不完全符合 S3 规范的服务，Mastodon 扩展了 S3 存储模块：

```ruby
module Paperclip
  module Storage
    module S3Extensions
      def copy_to_local_file(style, local_dest_path)
        options = {}
        # 强制单请求下载（某些 S3 兼容服务不支持分块）
        options[:mode] = 'single_request' if ENV['S3_FORCE_SINGLE_REQUEST'] == 'true'
        # 禁用校验和模式（某些服务不支持）
        options[:checksum_mode] = 'DISABLED' unless ENV['S3_ENABLE_CHECKSUM_MODE'] == 'true'
        
        s3_object(style).download_file(local_dest_path, options)
      end
    end
  end
end

Paperclip::Storage::S3.prepend(Paperclip::Storage::S3Extensions)
```

### 本地文件系统配置

**文件位置**: `config/initializers/paperclip.rb:162-169`

```ruby
else
  # 存储根路径
  Rails.configuration.x.file_storage_root_path = ENV.fetch(
    'PAPERCLIP_ROOT_PATH', 
    File.join(':rails_root', 'public', 'system')
  )
  
  Paperclip::Attachment.default_options.merge!(
    storage: :filesystem,
    path: File.join(
      Rails.configuration.x.file_storage_root_path, 
      ':prefix_path:class', ':attachment', ':id_partition', ':style', ':filename'
    ),
    url: ENV.fetch('PAPERCLIP_ROOT_URL', '/system') + "/#{PATH}"
  )
end
```

---

## 远端实例缓存机制

在联邦网络（Fediverse）中，Mastodon 实例需要缓存来自其他实例的媒体，以提升用户体验并减轻源实例压力。

### 远程媒体识别

**文件位置**: `app/models/media_attachment.rb:231-233`

```ruby
def local?
  remote_url.blank?  # remote_url 非空表示远程媒体
end
```

**数据库字段**:
```ruby
# 来自 Schema Information
remote_url: string           # default(""), not null
file_file_name: string       # 本地缓存的文件名
```

### 远程媒体作用域

**文件位置**: `app/models/media_attachment.rb:212-227`

```ruby
scope :attached, -> { where.not(status_id: nil).or(where.not(scheduled_status_id: nil)) }
scope :cached, -> { remote.where.not(file_file_name: nil) }     # 已缓存的远程媒体
scope :remote, -> { where.not(remote_url: '') }                   # 所有远程媒体
scope :local, -> { where(remote_url: '') }                         # 本地上传媒体
scope :unattached, -> { where(status_id: nil, scheduled_status_id: nil) }  # 未关联的媒体

# 关键：无本地交互的远程媒体（用于清理判断）
scope :without_local_interaction, lambda {
  where.not(
    Favourite.joins(:account).merge(Account.local)
      .where(Favourite.arel_table[:status_id].eq(MediaAttachment.arel_table[:status_id]))
      .select(1).arel.exists
  )
  .where.not(
    Bookmark.where(Bookmark.arel_table[:status_id].eq(MediaAttachment.arel_table[:status_id]))
      .select(1).arel.exists
  )
  .where.not(
    Status.local.where(Status.arel_table[:in_reply_to_id].eq(MediaAttachment.arel_table[:status_id]))
      .select(1).arel.exists
  )
  .where.not(
    Status.local.where(Status.arel_table[:reblog_of_id].eq(MediaAttachment.arel_table[:status_id]))
      .select(1).arel.exists
  )
  .where.not(
    Quote.joins(:status).merge(Status.local)
      .where(Quote.arel_table[:quoted_status_id].eq(MediaAttachment.arel_table[:status_id]))
      .select(1).arel.exists
  )
  .where.not(
    Quote.joins(:quoted_status).merge(Status.local)
      .where(Quote.arel_table[:status_id].eq(MediaAttachment.arel_table[:status_id]))
      .select(1).arel.exists
  )
}
```

**`without_local_interaction` 含义**: 排除满足以下任一条件的媒体：
1. 被本地用户收藏（Favourite）
2. 被本地用户书签（Bookmark）
3. 被本地用户回复
4. 被本地用户转嘟（Reblog）
5. 被本地用户引用（Quote）

### 远程附件处理机制

**文件位置**: `app/models/concerns/remotable.rb`

`Remotable` 是一个通用的关注点（Concern），为模型添加远程附件下载能力：

```ruby
module Remotable
  class_methods do
    def remotable_attachment(attachment_name, limit, suppress_errors: true, download_on_assign: true, attribute_name: nil)
      attribute_name ||= :"#{attachment_name}_remote_url"
      
      # 定义下载方法：download_file! / download_thumbnail!
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
            
            # 下载并赋值给 attachment
            # ResponseWithLimit 限制下载大小，防止 OOM
            public_send(:"#{attachment_name}=", ResponseWithLimit.new(response, limit))
          end
        rescue Mastodon::UnexpectedResponseError, *Mastodon::HTTP_CONNECTION_ERRORS => e
          public_send(:"#{attachment_name}=", nil) if public_send(:"#{attachment_name}_file_name").present?
          raise e unless suppress_errors
        rescue Paperclip::Errors::NotIdentifiedByImageMagickError, ... => e
          # 格式错误等，静默处理
          public_send(:"#{attachment_name}=", nil) if ...
        end
      end
      
      # 当设置 remote_url 时自动下载
      define_method(:"#{attribute_name}=") do |url|
        return if self[attribute_name] == url && public_send(:"#{attachment_name}_file_name").present?
        
        self[attribute_name] = url if has_attribute?(attribute_name)
        public_send(:"download_#{attachment_name}!", url) if download_on_assign
      end
    end
  end
end
```

### MediaAttachment 中的远程配置

**文件位置**: `app/models/media_attachment.rb:196, 205`

```ruby
# 主文件：不静默错误，赋值时不自动下载
remotable_attachment :file, VIDEO_LIMIT, suppress_errors: false, download_on_assign: false, attribute_name: :remote_url

# 缩略图：静默错误，赋值时不自动下载
remotable_attachment :thumbnail, IMAGE_LIMIT, suppress_errors: true, download_on_assign: false
```

**注意**: `download_on_assign: false` 意味着设置 `remote_url` 时不会立即下载，需要显式调用 `download_file!`。

### ActivityPub 解析流程

**文件位置**: `app/lib/activitypub/parser/media_attachment_parser.rb`

当从其他实例接收 ActivityPub 消息时，媒体附件被解析为：

```ruby
class ActivityPub::Parser::MediaAttachmentParser
  def remote_url
    url = Addressable::URI.parse(url_to_href(@json['url']))&.normalize&.to_s
    url unless unsupported_uri_scheme?(url)
  end
  
  def thumbnail_remote_url
    url = Addressable::URI.parse(@json['icon'].is_a?(Hash) ? @json['icon']['url'] : @json['icon'])&.normalize&.to_s
    url unless unsupported_uri_scheme?(url)
  end
  
  # 还包括：description, focus, blurhash, file_content_type
end
```

### 远程媒体下载触发

**文件位置**: `app/workers/redownload_media_worker.rb`

远程媒体的下载是异步执行的：

```ruby
class RedownloadMediaWorker
  include Sidekiq::Worker
  include ExponentialBackoff
  
  sidekiq_options queue: 'pull', retry: 3
  
  def perform(id)
    media_attachment = MediaAttachment.find(id)
    
    return if media_attachment.remote_url.blank?
    
    # 下载主文件和缩略图
    media_attachment.download_file!
    media_attachment.download_thumbnail!
    media_attachment.save
  rescue ActiveRecord::RecordNotFound
    # 记录已删除，忽略
  rescue Mastodon::UnexpectedResponseError => e
    response = e.response
    raise(e) unless response_error_unsalvageable?(response)
    # 404、410 等错误视为永久失败，不再重试
  end
end
```

**重新下载判断**:
```ruby
# app/models/media_attachment.rb:239-241
def needs_redownload?
  file.blank? && remote_url.present?  # 有远程 URL 但无本地缓存
end
```

---

## 缓存生命周期管理

远程媒体缓存不会永久保留，Mastodon 有一套完整的生命周期管理机制。

### 内容保留策略

**文件位置**: `app/models/content_retention_policy.rb`

```ruby
class ContentRetentionPolicy
  def self.current
    new
  end
  
  def media_cache_retention_period
    retention_period Setting.media_cache_retention_period
  end
  
  def content_cache_retention_period
    retention_period Setting.content_cache_retention_period
  end
  
  def backups_retention_period
    retention_period Setting.backups_retention_period
  end
  
  private
  
  def retention_period(value)
    value.days if value.is_a?(Integer) && value.positive?
    # nil 或 0 表示不限制（永久保留）
  end
end
```

**管理设置**:
- `media_cache_retention_period`: 媒体缓存保留天数（默认可能为 7 天或由管理员配置）
- 设置为 0 或未设置：不清理，永久保留

### 定期清理调度

**文件位置**: `app/workers/scheduler/vacuum_scheduler.rb`

```ruby
class Scheduler::VacuumScheduler
  include Sidekiq::Worker
  
  sidekiq_options retry: 0, lock: :until_executed, lock_ttl: 1.day.to_i
  
  def perform
    vacuum_operations.each do |operation|
      operation.perform
    rescue => e
      Rails.logger.error("Error while running #{operation.class.name}: #{e}")
    end
  end
  
  private
  
  def vacuum_operations
    [
      statuses_vacuum,
      media_attachments_vacuum,    # 媒体附件清理
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
  
  def content_retention_policy
    ContentRetentionPolicy.current
  end
end
```

### 媒体清理实现

**文件位置**: `app/lib/vacuum/media_attachments_vacuum.rb`

```ruby
class Vacuum::MediaAttachmentsVacuum
  TTL = 1.day.freeze  # 孤儿记录的存活时间
  
  def initialize(retention_period)
    @retention_period = retention_period
  end
  
  def perform
    vacuum_orphaned_records!      # 清理孤儿记录
    vacuum_cached_files! if retention_period?  # 清理过期缓存（如果配置了保留期）
  end
  
  private
  
  # 清理缓存的远程媒体文件
  def vacuum_cached_files!
    media_attachments_past_retention_period.find_in_batches do |media_attachments|
      # 清除文件但保留数据库记录（保留 remote_url，可重新下载）
      AttachmentBatch.new(MediaAttachment, media_attachments).clear
    rescue => e
      Rails.logger.error("Skipping batch while removing cached media attachments due to error: #{e}")
    end
  end
  
  # 清理孤儿记录（未关联到任何 status，且超过 1 天）
  def vacuum_orphaned_records!
    orphaned_media_attachments.find_in_batches do |media_attachments|
      # 删除文件和数据库记录
      AttachmentBatch.new(MediaAttachment, media_attachments).delete
    rescue => e
      Rails.logger.error("Skipping batch while removing orphaned media attachments due to error: #{e}")
    end
  end
  
  # 过期缓存查询条件
  def media_attachments_past_retention_period
    MediaAttachment
      .remote                            # 远程媒体
      .cached                            # 已下载到本地
      .created_before(@retention_period.ago)    # 创建时间早于保留期前
      .updated_before(@retention_period.ago)    # 更新时间早于保留期前
      # 注意：不包含 without_local_interaction！这意味着有本地交互的也会被清理？
      # 但实际上，有本地交互的 status 通常不会被清理，其 media_attachments 也会保留
  end
  
  # 孤儿记录查询条件
  def orphaned_media_attachments
    MediaAttachment
      .unattached                         # 未关联到 status 或 scheduled_status
      .created_before(TTL.ago)            # 创建超过 1 天
  end
  
  def retention_period?
    @retention_period.present?
  end
end
```

### 批量删除实现

**文件位置**: `app/lib/attachment_batch.rb`

```ruby
class AttachmentBatch
  LIMIT = ENV.fetch('S3_BATCH_DELETE_LIMIT', 1000).to_i  # S3 批量删除限制
  MAX_RETRY = ENV.fetch('S3_BATCH_DELETE_RETRY', 3).to_i
  
  NULLABLE_ATTRIBUTES = %w(
    file_name content_type file_size fingerprint created_at updated_at
  ).freeze
  
  # 删除文件 + 删除数据库记录
  def delete
    remove_files
    batch.delete_all
  end
  
  # 仅清除文件，保留数据库记录（设置属性为 nil）
  def clear
    remove_files
    batch.update_all(nullified_attributes)
  end
  
  private
  
  def remove_files
    keys = []
    
    records.each do |record|
      @attachment_names.each do |attachment_name|
        attachment = record.public_send(attachment_name)
        styles = BASE_STYLES | attachment.styles.keys  # :original + 自定义样式
        
        next if attachment.blank?
        
        styles.each do |style|
          case @storage_mode
          when :filesystem
            # 本地文件：直接删除
            FileUtils.remove_file(path, true)
            # 尝试删除空目录
            FileUtils.rmdir(File.dirname(path), parents: true)
            
          when :s3
            # S3：收集 key，批量删除
            keys << attachment.style_name_as_path(style)
            
          when :fog
            # Swift：逐个删除
            attachment.send(:directory).files.new(key: path).destroy
            
          when :azure
            # Azure：调用 destroy
            attachment.destroy
          end
        end
      end
    end
    
    # S3 批量删除
    return unless storage_mode == :s3
    
    keys.each_slice(LIMIT) do |keys_slice|
      bucket.delete_objects(delete: {
        objects: keys_slice.map { |key| { key: key } },
        quiet: true,
      })
    end
  end
  
  # clear 操作时要置空的属性
  def nullified_attributes
    @attachment_names.flat_map { |attachment_name| 
      NULLABLE_ATTRIBUTES.map { |attribute| "#{attachment_name}_#{attribute}" } & klass.column_names 
    }.index_with(nil)
    # 结果示例: { "file_file_name" => nil, "file_content_type" => nil, ... }
  end
end
```

### 缓存清除前后的状态对比

| 阶段 | remote_url | file_file_name | 状态 |
|------|-----------|----------------|------|
| 刚解析（未下载） | `https://other.instance/media/xxx` | `nil` | 远程，未缓存 |
| 下载完成后 | `https://other.instance/media/xxx` | `abcdef12345.jpg` | 远程，已缓存 |
| 缓存清理后 | `https://other.instance/media/xxx` | `nil` | 远程，缓存已过期（可重新下载） |

### 缓存过期后的重新访问

当用户访问已过期缓存的媒体时：
1. URL 仍然指向本地实例的 cache/ 路径
2. 如果文件不存在，取决于存储配置：
   - 本地文件系统：可能返回 404 或触发后端逻辑
   - CDN/S3：如果 CDN 有缓存，可能仍能访问；否则 404

实际上，Mastodon 的设计是：
- `clear` 操作仅删除文件，保留 `remote_url`
- 当 status 被重新获取或用户再次访问时，可能触发 `RedownloadMediaWorker` 重新下载

---

## 总结

### 完整链路流程图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           本地媒体上传链路                                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  客户端                                                                                 │
│     │                                                                                   │
│     │ POST /api/v1/media 或 /api/v2/media                                             │
│     │  - file: 媒体文件                                                                 │
│     │  - description: 描述                                                              │
│     │  - focus: 焦点坐标                                                                │
│     ▼                                                                                   │
│  ┌─────────────────────┐                                                                │
│  │ Api::V1::MediaController │ 或 Api::V2::MediaController                             │
│  │ - v1: 同步处理，返回 200/206                                                        │
│  │ - v2: 延迟处理，返回 202 Accepted                                                    │
│  └───────────┬─────────┘                                                                │
│              │                                                                           │
│              ▼                                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐                │
│  │ MediaAttachment.create!                                              │                │
│  │ - 验证文件类型、大小                                                  │                │
│  │ - 检查视频分辨率、帧率限制                                            │                │
│  │ - 设置 processing 状态:                                               │                │
│  │   - v1/图片: :complete (同步处理)                                    │                │
│  │   - v2/视频: :queued (延迟处理)                                      │                │
│  └───────────┬────────────────────────────────────────────────────────┘                │
│              │                                                                           │
│              ├────────────────── 延迟处理 ──────────────────┐                          │
│              │                                                 │                          │
│              ▼                                                 ▼                          │
│  ┌─────────────────────┐                    ┌───────────────────────────────────────┐ │
│  │ Paperclip 同步处理   │                    │ PostProcessMediaWorker (Sidekiq)      │ │
│  │ - LazyThumbnail      │                    │ - processing: :in_progress → :complete │ │
│  │ - BlurhashTranscoder │                    │ - 调用 file.reprocess!(:original)     │ │
│  │ - ColorExtractor     │                    └───────────────────┬───────────────────┘ │
│  │ - TypeCorrector      │                                        │                       │
│  └───────────┬─────────┘                                        ▼                       │
│              │                                    ┌─────────────────────────────────┐   │
│              │                                    │ 根据媒体类型选择处理器链          │   │
│              │                                    │ - GIF: gif_transcoder + ...    │   │
│              │                                    │ - 视频: transcoder + ...        │   │
│              │                                    │ - 音频: image_extractor + ...   │   │
│              │                                    └───────────────┬─────────────────┘   │
│              │                                                    │                       │
│              └──────────────────────────┬───────────────────────┘                       │
│                                         │                                                       │
│                                         ▼                                                       │
│                              ┌─────────────────────┐                                            │
│                              │ Paperclip 存储       │                                            │
│                              │ - 路径: :class/:attachment/... │                               │
│                              │ - 本地: public/system/...        │                               │
│                              │ - S3: bucket/:prefix_url/...    │                               │
│                              └───────────┬─────────┘                                            │
│                                          │                                                          │
│                                          ▼                                                          │
│                              ┌──────────────────────────────────────┐                           │
│                              │ 元数据写入 (after_post_process)       │                           │
│                              │ - file_meta: 尺寸、颜色、焦点等       │                           │
│                              │ - blurhash: 模糊哈希字符串            │                           │
│                              └──────────────────────────────────────┘                           │
│                                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           远程媒体缓存链路                                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  远程实例                                                                               │
│     │                                                                                   │
│     │ ActivityPub: Create/Announce 活动                                                │
│     │ 包含 attachment: { type: "Image", url: "https://..." }                         │
│     ▼                                                                                   │
│  ┌────────────────────────────────────────────────────────────────────────┐           │
│  │ ActivityPub::Parser::MediaAttachmentParser                              │           │
│  │ - 解析 remote_url, thumbnail_remote_url                                  │           │
│  │ - 解析 description, focus, blurhash                                      │           │
│  └───────────┬────────────────────────────────────────────────────────────┘           │
│              │                                                                           │
│              ▼                                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐                │
│  │ MediaAttachment 创建                                                  │                │
│  │ - remote_url = "https://other.instance/media/xxx"                   │                │
│  │ - file_file_name = nil (尚未下载)                                     │                │
│  │ - 标记为 "远程" 媒体                                                  │                │
│  └───────────┬────────────────────────────────────────────────────────┘                │
│              │                                                                           │
│              ▼                                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐                │
│  │ RedownloadMediaWorker (Sidekiq, queue: :pull)                       │                │
│  │ - 调用 download_file!                                                 │                │
│  │ - 使用 Request.get 获取远程文件                                       │                │
│  │ - ResponseWithLimit 限制下载大小                                      │                │
│  │ - 通过 remotable_attachment 机制处理                                  │                │
│  └───────────┬────────────────────────────────────────────────────────┘                │
│              │                                                                           │
│              ▼                                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐                │
│  │ 存储到本地缓存                                                        │                │
│  │ - 路径前缀: cache/ (通过 :prefix_path 插值)                         │                │
│  │ - 本地: public/system/cache/media_attachments/...                    │                │
│  │ - S3: bucket/cache/media_attachments/...                             │                │
│  │ - file_file_name = "abc123.jpg" (已缓存)                             │                │
│  └────────────────────────────────────────────────────────────────────┘                │
│                                                                                           │
└──────────────────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           缓存生命周期管理                                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  Scheduler::VacuumScheduler (定期执行)                                                 │
│     │                                                                                   │
│     ▼                                                                                   │
│  ┌────────────────────────────────────────────────────────────────────┐                │
│  │ Vacuum::MediaAttachmentsVacuum                                       │                │
│  │ 配置: ContentRetentionPolicy.current.media_cache_retention_period   │                │
│  │      (从 Setting.media_cache_retention_period 读取，单位：天)       │                │
│  └───────────┬────────────────────────────────────────────────────────┘                │
│              │                                                                           │
│              ├──────────────────────────────────────────────────────────┐              │
│              │                                                             │              │
│              ▼                                                             ▼              │
│  ┌─────────────────────────┐                         ┌──────────────────────────────┐  │
│  │ vacuum_orphaned_records!│                         │ vacuum_cached_files!          │  │
│  │ - 目标: 孤儿媒体附件      │                         │ - 目标: 过期的远程缓存        │  │
│  │ - 条件: unattached       │                         │ - 条件: remote + cached       │  │
│  │         created_before(1.day.ago) │                │         created_before(retention.ago) │
│  │                         │                         │         updated_before(retention.ago) │
│  │ - 操作: AttachmentBatch.delete │                   │ - 操作: AttachmentBatch.clear │  │
│  │   → 删除文件 + 删除记录  │                         │   → 仅删除文件，保留记录      │  │
│  └─────────────────────────┘                         │   → file_file_name 设为 nil   │  │
│                                                        │   → remote_url 保留（可重下） │  │
│                                                        └───────────────┬──────────────┘  │
│                                                                        │                  │
│                                                                        ▼                  │
│                                                          ┌──────────────────────────────┐  │
│                                                          │ AttachmentBatch 批量处理     │  │
│                                                          │                              │  │
│                                                          │ 存储模式:                     │  │
│                                                          │ - :filesystem                │  │
│                                                          │   FileUtils.remove_file      │  │
│                                                          │                              │  │
│                                                          │ - :s3                        │  │
│                                                          │   bucket.delete_objects      │  │
│                                                          │   (批量删除，1000 个/批)     │  │
│                                                          │                              │  │
│                                                          │ - :fog (Swift)               │  │
│                                                          │ - :azure                     │  │
│                                                          └──────────────────────────────┘  │
│                                                                                             │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

### 不同媒体类型处理差异总结

| 特性 | 图片 (JPG/PNG/WebP) | 静态 GIF | 动画 GIF | 视频 (MP4/WebM) | 音频 (MP3/Ogg) |
|------|---------------------|---------|----------|-----------------|----------------|
| **类型枚举** | `image` | `image` | `gifv` | `video` 或 `gifv` | `audio` |
| **处理器链** | lazy_thumbnail, blurhash_transcoder, type_corrector | 同图片 | gif_transcoder, blurhash_transcoder | transcoder, blurhash_transcoder, type_corrector | image_extractor, transcoder, type_corrector |
| **输出格式** | 保持原格式 或 JPEG | GIF | MP4 | MP4 (可能透传) | MP3 |
| **缩略图** | `:small` 样式 | 同图片 | 从视频提取 | 从视频提取第一帧 | 从音频封面/第一帧提取 |
| **延迟处理** | 否 (API v2 除外) | 否 | 是 | 是 | 是 |
| **文件限制** | 16 MB | 16 MB | 99 MB | 99 MB | 99 MB |

### 关键设计要点

1. **API v1 与 v2 的差异**:
   - v1: 同步处理，适合小图片
   - v2: 异步处理（`delay_processing: true`），适合大文件，返回 202 Accepted

2. **视频透传优化**:
   - 符合条件的 H.264/AAC/MP4 视频可直接透传，不重新编码
   - 条件：视频编码 H.264，音频编码 AAC（或无音频），色彩空间 YUV420P

3. **远程与本地路径区分**:
   - 本地媒体：无前缀
   - 远程缓存：`cache/` 前缀（通过 `:prefix_path` 插值实现）

4. **缓存清理策略**:
   - `clear`: 仅删除文件，保留数据库记录（`remote_url` 保留，可重新下载）
   - `delete`: 删除文件 + 删除数据库记录（用于孤儿记录）
   - 有本地交互的媒体（收藏、回复、转嘟等）其关联的 status 不会被轻易清理

5. **存储抽象**:
   - 通过 Paperclip 统一支持 S3、Swift、Azure、本地文件系统
   - S3 支持批量删除（1000 个/批）和 CDN 别名配置

6. **安全性考虑**:
   - `ResponseWithLimit` 限制下载大小，防止超大文件导致 OOM
   - 本地图片会剥离元数据（`needs_metadata_stripping?`），远程缓存保持原样
   - 视频尺寸、帧率、帧数限制，防止 DoS 攻击

### 相关文件位置速查

| 功能 | 文件路径 |
|------|---------|
| API 控制器 | `app/controllers/api/v1/media_controller.rb` |
| | `app/controllers/api/v2/media_controller.rb` |
| 媒体模型 | `app/models/media_attachment.rb` |
| 后处理 Worker | `app/workers/post_process_media_worker.rb` |
| 远程下载 Worker | `app/workers/redownload_media_worker.rb` |
| 远程附件机制 | `app/models/concerns/remotable.rb` |
| 视频转码器 | `lib/paperclip/transcoder.rb` |
| GIF 转码器 | `lib/paperclip/gif_transcoder.rb` |
| 图片提取器 | `lib/paperclip/image_extractor.rb` |
| 缩略图处理器 | `lib/paperclip/lazy_thumbnail.rb` |
| Blurhash 编码器 | `lib/paperclip/blurhash_transcoder.rb` |
| 颜色提取器 | `lib/paperclip/color_extractor.rb` |
| 类型修正器 | `lib/paperclip/type_corrector.rb` |
| 存储配置 | `config/initializers/paperclip.rb` |
| 定期清理调度 | `app/workers/scheduler/vacuum_scheduler.rb` |
| 媒体清理逻辑 | `app/lib/vacuum/media_attachments_vacuum.rb` |
| 批量删除实现 | `app/lib/attachment_batch.rb` |
| 保留策略 | `app/models/content_retention_policy.rb` |
| ActivityPub 解析 | `app/lib/activitypub/parser/media_attachment_parser.rb` |
