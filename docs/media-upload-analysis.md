# Mastodon 媒体上传完整链路深度分析

## 目录

1. [概述](#概述)
2. [API v1 与 v2 的精确差异](#api-v1-与-v2-的精确差异)
3. [延迟处理机制深度解析](#延迟处理机制深度解析)
4. [不同媒体类型的处理差异](#不同媒体类型的处理差异)
5. [存储系统实现](#存储系统实现)
6. [远程媒体缓存完整生命周期](#远程媒体缓存完整生命周期)
7. [缓存保留策略的设计取舍](#缓存保留策略的设计取舍)
8. [总结](#总结)

---

## 概述

本文档是对 Mastodon 媒体处理系统的深度源码分析，纠正了之前的一些误解，特别是关于 API v1/v2 的状态码行为、延迟处理机制的精确触发条件，以及远程缓存的完整生命周期和保留策略的设计取舍。

---

## API v1 与 v2 的精确差异

### 之前的误解

❌ **错误理解**:
- API v1 可能返回 206 Partial Content
- API v2 对所有媒体类型都延迟处理

### 精确分析

#### 核心代码位置

**状态码决定逻辑**:
```ruby
# app/controllers/api/v1/media_controller.rb:39-41
def status_code_for_media_attachment
  @media_attachment.not_processed? ? 206 : 200
end

# app/controllers/api/v2/media_controller.rb:20-22
def status_from_media_processing
  @media_attachment.not_processed? ? 202 : 200
end
```

**`not_processed?` 的定义**:
```ruby
# app/models/media_attachment.rb:235-237
def not_processed?
  processing.present? && !processing_complete?
end

# app/models/media_attachment.rb:39
enum :processing, { queued: 0, in_progress: 1, complete: 2, failed: 3 }, prefix: true
```

**`processing` 状态的设置**:
```ruby
# app/models/media_attachment.rb:368-370
def set_processing
  self.processing = delay_processing? ? :queued : :complete
end
```

**`delay_processing?` 的完整条件**:
```ruby
# app/models/media_attachment.rb:283-291
attr_writer :delay_processing

def delay_processing?
  @delay_processing && larger_media_format?
end

def larger_media_format?
  video? || gifv? || audio?
end
```

#### 完整真相

| API 版本 | 媒体类型 | `delay_processing` 设置 | `larger_media_format?` | `delay_processing?` | `processing` 状态 | `not_processed?` | 返回状态码 |
|---------|---------|------------------------|-----------------------|--------------------|------------------|-----------------|-----------|
| **v1** | 任意 | `nil` (未设置) | 任意 | ❌ `false` | `:complete` | ❌ `false` | **200** |
| **v2** | 图片/静态 GIF | `true` | ❌ `false` | ❌ `false` | `:complete` | ❌ `false` | **200** |
| **v2** | 视频/音频/动画 GIF | `true` | ✅ `true` | ✅ `true` | `:queued` | ✅ `true` | **202** |

#### 关键结论

1. **API v1 永远返回 200**，永远同步处理
   - 不设置 `delay_processing` 属性
   - `delay_processing?` 永远为 `false`
   - `processing = :complete`
   - `not_processed?` 永远为 `false`

2. **API v2 只对视频/音频/动画 GIF 延迟处理**
   - 设置 `delay_processing: true`
   - 但还需要 `larger_media_format?` 为 `true`
   - 图片上传时 v2 行为与 v1 完全相同（同步处理，返回 200）
   - 只有视频/音频/动画 GIF 才会返回 202 Accepted

3. **状态码含义差异**:
   - v1: 206 理论上存在，但实际上永远不会触发（v1 不延迟处理）
   - v2: 202 表示已接受，正在后台处理（仅适用于大媒体）

#### API v2 的实现

```ruby
# app/controllers/api/v2/media_controller.rb:4-22
class Api::V2::MediaController < Api::V1::MediaController
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
end
```

---

## GIF 上传时序深度解析

### 关键发现

这是整个媒体处理系统中最复杂、最容易误解的部分。核心问题是**时序不一致**：

1. `set_type_and_extension` 在 Paperclip 处理前执行，基于 MIME 类型设置 `type`
2. `process_style?` 在 Paperclip 处理时执行，依赖 `type` 字段
3. `GifTranscoder` 在 Paperclip 处理中执行，可能改变 `type` 字段
4. `set_processing` 在 Paperclip 处理后执行，再次检查 `type` 字段

### MIME 类型分类的"分裂"

首先理解一个关键的"分裂"设计：

```ruby
# app/models/media_attachment.rb:64-68
IMAGE_MIME_TYPES             = %w(image/jpeg image/png image/gif image/heic image/heif image/webp image/avif).freeze
VIDEO_MIME_TYPES             = %w(video/webm video/mp4 video/quicktime video/ogg).freeze
VIDEO_CONVERTIBLE_MIME_TYPES = %w(video/webm video/quicktime).freeze
```

**`image/gif` 属于 `IMAGE_MIME_TYPES`，不属于 `VIDEO_MIME_TYPES`**

但 `file_styles` 和 `file_processors` 有单独的检查：

```ruby
# app/models/media_attachment.rb:323-347
def file_styles(attachment)
  if attachment.instance.file_content_type == 'image/gif' || VIDEO_CONVERTIBLE_MIME_TYPES.include?(...)
    VIDEO_CONVERTED_STYLES    # 视频风格：转换为 MP4
  elsif IMAGE_MIME_TYPES.include?(...)
    IMAGE_STYLES               # 图片风格：保持原格式
  end
end

def file_processors(instance)
  if instance.file_content_type == 'image/gif'
    [:gif_transcoder, :blurhash_transcoder]  # 视频处理器
  elsif IMAGE_MIME_TYPES.include?(instance.file_content_type)
    [:lazy_thumbnail, :blurhash_transcoder, :type_corrector]  # 图片处理器
  end
end
```

**这个"分裂"是时序问题的根源。**

### 完整执行时序

让我们追踪 `image/gif` 上传的完整执行顺序：

#### 阶段 1: `before_file_validate` - `set_type_and_extension`

```ruby
# app/models/media_attachment.rb:356-366
def set_type_and_extension
  self.type = begin
    if VIDEO_MIME_TYPES.include?(file_content_type)
      :video
    elsif AUDIO_MIME_TYPES.include?(file_content_type)
      :audio
    else
      :image    # image/gif 进入这里！
    end
  end
end
```

**结果**: `type = :image`（因为 `image/gif` 属于 `IMAGE_MIME_TYPES`，不属于 `VIDEO_MIME_TYPES`）

#### 阶段 2: Paperclip 处理 - `process_style?`

Paperclip 决定是否处理某个样式：

```ruby
# lib/paperclip/attachment_extensions.rb:37-47
def process_style?(style_name, style_args)
  if style_name == :original && instance.respond_to?(:delay_processing_for_attachment?) && instance.delay_processing_for_attachment?(name)
    false  # 跳过
  else
    style_args.empty? || style_args.include?(style_name)
  end
end
```

`delay_processing_for_attachment?` 调用：

```ruby
# app/models/media_attachment.rb:289-291
def delay_processing_for_attachment?(attachment_name)
  delay_processing? && attachment_name == :file
end

# app/models/media_attachment.rb:285-287
def delay_processing?
  @delay_processing && larger_media_format?
end

# app/models/media_attachment.rb:251-253
def larger_media_format?
  video? || gifv? || audio?  # 当前 type = :image，所以返回 false！
end
```

**关键**: 此时 `GifTranscoder` 还没执行，`type = :image`，所以 `larger_media_format? = false`

**结果**:
- `delay_processing? = @delay_processing && false = false`（即使 API v2 设置了 `@delay_processing = true`）
- `process_style?(:original, ...) = true` → **`:original` 样式被处理，不跳过！**

#### 阶段 3: `GifTranscoder` 执行

```ruby
# lib/paperclip/gif_transcoder.rb:105-118
class GifTranscoder < Paperclip::Processor
  def make
    return File.open(@file.path) unless needs_convert?  # 检查是否为动画
    
    # 动画 GIF：转换为 MP4
    final_file = Paperclip::Transcoder.make(file, options, attachment)
    
    if options[:style] == :original
      # 关键：改变 type 字段！
      attachment.instance.type = MediaAttachment.types[:gifv]
    end
  end
end

# lib/paperclip/gif_transcoder.rb:122-124
def needs_convert?
  GifReader.animated?(file.path)  # 检查帧数 > 1
end
```

**结果**:
- **静态 GIF**（1 帧）：`needs_convert? = false`，直接返回，`type` 保持 `:image`
- **动画 GIF**（多帧）：`needs_convert? = true`，转换为 MP4，`type` 被改为 `:gifv`

#### 阶段 4: `before_create` - `set_processing`

```ruby
# app/models/media_attachment.rb:368-370
def set_processing
  self.processing = delay_processing? ? :queued : :complete
end

# app/models/media_attachment.rb:285-287
def delay_processing?
  @delay_processing && larger_media_format?
end

# app/models/media_attachment.rb:251-253
def larger_media_format?
  video? || gifv? || audio?  # 现在检查 type 字段！
end
```

**注意**: 这里再次调用 `delay_processing?`，此时 `GifTranscoder` 已经执行过了。

#### 阶段 5: `after_commit` - `enqueue_processing`

```ruby
# app/models/media_attachment.rb:420-422
def enqueue_processing
  PostProcessMediaWorker.perform_async(id) if delay_processing?
end
```

### 三种场景的精确结果

#### 场景 1: 静态 GIF（所有 API）

```
阶段 1: set_type_and_extension
  └── file_content_type = 'image/gif'
  └── type = :image（因为 image/gif 属于 IMAGE_MIME_TYPES）

阶段 2: process_style?(:original)
  └── larger_media_format? = video? || gifv? || audio? = false
  └── delay_processing? = false（无论 API v1 还是 v2）
  └── process_style? = true → :original 被处理

阶段 3: GifTranscoder
  └── needs_convert? = GifReader.animated?(path) = false（1 帧）
  └── type 保持 :image

阶段 4: set_processing
  └── larger_media_format? = false
  └── delay_processing? = false
  └── processing = :complete

阶段 5: enqueue_processing
  └── 不执行

结果:
  ├── type: image
  ├── processing: complete
  ├── not_processed?: false
  └── 返回状态码: 200（API v1 和 v2）
```

#### 场景 2: 动画 GIF（API v1）

```
阶段 1: set_type_and_extension
  └── type = :image

阶段 2: process_style?(:original)
  └── larger_media_format? = false
  └── delay_processing? = false
  └── process_style? = true → :original 被处理

阶段 3: GifTranscoder
  └── needs_convert? = true（多帧）
  └── 转换为 MP4
  └── type 被改为 :gifv

阶段 4: set_processing
  └── larger_media_format? = video? || gifv? || audio? = true
  └── @delay_processing = nil（API v1 未设置）
  └── delay_processing? = nil && true = false
  └── processing = :complete

阶段 5: enqueue_processing
  └── 不执行

结果:
  ├── type: gifv
  ├── processing: complete
  ├── not_processed?: false
  └── 返回状态码: 200
```

#### 场景 3: 动画 GIF（API v2）

```
阶段 1: set_type_and_extension
  └── type = :image

阶段 2: process_style?(:original)
  └── larger_media_format? = false（type = :image）
  └── delay_processing? = true && false = false
  └── process_style? = true → :original 被处理（同步！）

阶段 3: GifTranscoder
  └── needs_convert? = true
  └── 转换为 MP4
  └── type 被改为 :gifv

阶段 4: set_processing
  └── larger_media_format? = true（type = :gifv）
  └── @delay_processing = true（API v2 设置）
  └── delay_processing? = true && true = true
  └── processing = :queued

阶段 5: enqueue_processing
  └── delay_processing? = true
  └── PostProcessMediaWorker.perform_async(id)
  └── 会再次处理 :original！

结果:
  ├── type: gifv
  ├── processing: queued
  ├── not_processed?: true
  ├── 返回状态码: 202
  └── ⚠️ :original 被处理两次！
```

### 关键发现总结

#### 1. API 行为精确表

| 场景 | API 版本 | 静态/动画 | `type`（最终） | `larger_media_format?`（阶段 2） | `delay_processing?`（阶段 2） | `:original` 是否跳过 | `processing` | `not_processed?` | 返回状态码 |
|------|---------|----------|----------------|-----------------------------------|--------------------------------|----------------------|-------------|-----------------|-----------|
| 静态 GIF | v1 | 静态 | `image` | `false` | `false` | ❌ 不跳过 | `complete` | `false` | **200** |
| 静态 GIF | v2 | 静态 | `image` | `false` | `false` | ❌ 不跳过 | `complete` | `false` | **200** |
| 动画 GIF | v1 | 动画 | `gifv` | `false`（阶段 2） | `false`（阶段 2） | ❌ 不跳过 | `complete` | `false` | **200** |
| 动画 GIF | v2 | 动画 | `gifv` | `false`（阶段 2） | `false`（阶段 2） | ❌ 不跳过 | `queued` | `true` | **202** |

#### 2. 时序不一致问题

这是一个"设计上的时序不一致"：

```
阶段 2（process_style?）:
  type = :image → larger_media_format? = false → :original 不跳过

阶段 4（set_processing）:
  type = :gifv（动画 GIF）→ larger_media_format? = true → processing = :queued
```

**结果**：
- API v2 的动画 GIF 会被**处理两次**
- 第一次：同步处理（阶段 2-3）
- 第二次：`PostProcessMediaWorker` 异步处理

#### 3. 与视频的对比

对于真正的视频（`video/mp4`、`video/webm` 等）：

```
阶段 1: set_type_and_extension
  └── file_content_type = 'video/mp4' 属于 VIDEO_MIME_TYPES
  └── type = :video

阶段 2: process_style?(:original)
  └── larger_media_format? = video? = true
  └── API v2: @delay_processing = true → delay_processing? = true
  └── process_style? = false → :original 被跳过！

阶段 4: set_processing
  └── type = :video → larger_media_format? = true
  └── processing = :queued

结果:
  └── :original 只被处理一次（在 PostProcessMediaWorker 中）
```

**视频的时序是一致的**，因为 `type` 在阶段 1 就被设置为 `:video`，不会改变。

### 为什么存在这个时序问题？

这可能是历史遗留问题：

1. **最初的设计**：GIF 被当作图片处理，没有延迟机制
2. **后来的优化**：动画 GIF 应该被当作视频处理，添加了 `gifv` 类型
3. **延迟处理的引入**：API v2 引入了延迟处理，但 `type` 的改变发生在处理过程中

**核心问题**：
- `process_style?` 依赖 `type` 字段
- 但 `type` 字段在 `GifTranscoder` 中才被改变
- 而 `GifTranscoder` 是在 Paperclip 处理过程中执行的

### 流程图总结

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      GIF 上传完整时序流程图                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  上传 image/gif                                                                │
│       │                                                                       │
│       ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 1: before_file_validate                                         │   │
│  │ set_type_and_extension                                                │   │
│  │                                                                       │   │
│  │ file_content_type = 'image/gif'                                      │   │
│  │ IMAGE_MIME_TYPES.include?('image/gif') = true                       │   │
│  │ VIDEO_MIME_TYPES.include?('image/gif') = false                       │   │
│  │                                                                       │   │
│  │ 结果: type = :image ◄── 关键！                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                       │
│       ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 2: Paperclip 处理 - process_style?(:original)                  │   │
│  │                                                                       │   │
│  │ delay_processing_for_attachment?(:file)                              │   │
│  │   └── delay_processing? = @delay_processing && larger_media_format? │   │
│  │                                                                       │   │
│  │ larger_media_format? = video? || gifv? || audio?                    │   │
│  │                    = false （因为 type = :image）                     │   │
│  │                                                                       │   │
│  │ 结果:                                                                 │   │
│  │ - 静态 GIF: delay_processing? = false                                │   │
│  │ - 动画 GIF: delay_processing? = false（即使 API v2）                │   │
│  │ - process_style? = true → :original 被处理！◄── 关键！               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                       │
│       ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 3: GifTranscoder 执行                                            │   │
│  │                                                                       │   │
│  │ needs_convert? = GifReader.animated?(file.path)                      │   │
│  │                                                                       │   │
│  │     ┌─────────────────┬─────────────────┐                            │   │
│  │     │                 │                 │                            │   │
│  │     ▼                 ▼                 │                            │   │
│  │  静态 GIF（1 帧）  动画 GIF（多帧）     │                            │   │
│  │  needs_convert?=   needs_convert?=     │                            │   │
│  │  false              true                 │                            │   │
│  │     │                 │                 │                            │   │
│  │     │                 ▼                 │                            │   │
│  │     │           转换为 MP4              │                            │   │
│  │     │           type = :gifv ◄── 改变！ │                            │   │
│  │     │                 │                 │                            │   │
│  │     └───────┬─────────┘                 │                            │   │
│  │             │                           │                            │   │
│  │  结果: type = :image（静态）或 :gifv（动画）                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                       │
│       ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 4: before_create - set_processing                               │   │
│  │                                                                       │   │
│  │ self.processing = delay_processing? ? :queued : :complete           │   │
│  │                                                                       │   │
│  │ 注意：这里再次调用 delay_processing?！                                │   │
│  │                                                                       │   │
│  │ ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │ │ 静态 GIF（所有 API）:                                              │ │   │
│  │ │ type = :image → larger_media_format? = false                     │ │   │
│  │ │ delay_processing? = false → processing = :complete               │ │   │
│  │ └─────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                       │   │
│  │ ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │ │ 动画 GIF（API v1）:                                                │ │   │
│  │ │ type = :gifv → larger_media_format? = true                       │ │   │
│  │ │ @delay_processing = nil → delay_processing? = false              │ │   │
│  │ │ processing = :complete                                             │ │   │
│  │ └─────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                       │   │
│  │ ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │ │ 动画 GIF（API v2）:                                                │ │   │
│  │ │ type = :gifv → larger_media_format? = true                       │ │   │
│  │ │ @delay_processing = true → delay_processing? = true              │ │   │
│  │ │ processing = :queued ◄── 与阶段 2 不一致！                         │ │   │
│  │ └─────────────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                       │
│       ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 5: after_commit - enqueue_processing                           │   │
│  │                                                                       │   │
│  │ PostProcessMediaWorker.perform_async(id) if delay_processing?       │   │
│  │                                                                       │   │
│  │ - 静态 GIF: 不执行                                                    │   │
│  │ - 动画 GIF（API v1）: 不执行                                          │   │
│  │ - 动画 GIF（API v2）: delay_processing? = true → 执行！             │   │
│  │   └── PostProcessMediaWorker 会再次处理 :original                    │   │
│  │   └── ⚠️ :original 被处理两次！                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 延迟处理机制深度解析

### 之前的误解

❌ **错误理解**:
- 延迟处理会跳过所有样式的处理
- 延迟处理期间什么都不做

### 精确分析

#### 核心代码：Paperclip 猴子补丁

```ruby
# lib/paperclip/attachment_extensions.rb:37-47
# We overwrite this method to support delayed processing in
# Sidekiq. Since we process the original file to reduce disk
# usage, and we still want to generate thumbnails straight
# away, it's the only style we need to exclude
def process_style?(style_name, style_args)
  if style_name == :original && instance.respond_to?(:delay_processing_for_attachment?) && instance.delay_processing_for_attachment?(name)
    false  # 跳过 :original
  else
    style_args.empty? || style_args.include?(style_name)
  end
end
```

**关键注释翻译**:
> "我们重写这个方法以支持 Sidekiq 中的延迟处理。由于我们处理 original 文件是为了减少磁盘使用，而我们仍然希望立即生成缩略图，所以这是我们唯一需要排除的样式。"

#### 延迟处理的精确含义

| 样式 | 是否延迟处理 | 原因 |
|------|-------------|------|
| **`:original`** | ✅ **延迟** | 可能是大文件（视频转码、音频转码），耗时久 |
| **`:small`** | ❌ **立即处理** | 缩略图很小，生成快，UI 需要立即显示 |

#### 延迟处理的触发条件

```ruby
# app/models/media_attachment.rb:289-291
def delay_processing_for_attachment?(attachment_name)
  delay_processing? && attachment_name == :file
end
```

**完整条件链**:
```
delay_processing_for_attachment?(:file)
  └── delay_processing?
        ├── @delay_processing == true  (来自 API v2 的参数)
        └── larger_media_format?
              └── video? || gifv? || audio?
```

#### 异步处理 Worker

```ruby
# app/workers/post_process_media_worker.rb:22-37
def perform(media_attachment_id)
  media_attachment = MediaAttachment.find(media_attachment_id)
  media_attachment.processing = :in_progress
  media_attachment.save

  # 保存原有元数据，因为 paperclip-av-transcoder 会覆盖
  previous_meta = media_attachment.file_meta

  # 只重处理 :original 样式！
  media_attachment.file.reprocess!(:original)
  
  media_attachment.processing = :complete
  media_attachment.file_meta = previous_meta.merge(media_attachment.file_meta).with_indifferent_access.slice(*MediaAttachment::META_KEYS)
  media_attachment.save
end
```

#### 处理流程图

```
API v2 上传视频 (large_media_format? = true)
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  MediaAttachment.create!                                      │
│                                                               │
│  1. before_create: set_processing                             │
│     processing = :queued (因为 delay_processing? = true)    │
│                                                               │
│  2. Paperclip 处理                                            │
│     process_style?(:original, ...)                           │
│       └── style_name == :original && delay_processing? = true │
│       └── 返回 false → 跳过 :original                         │
│                                                               │
│     process_style?(:small, ...)                              │
│       └── style_name != :original                             │
│       └── 返回 true → 立即处理 :small (缩略图)               │
│                                                               │
│  3. after_commit: enqueue_processing                          │
│     PostProcessMediaWorker.perform_async(id)                 │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  客户端收到响应                                                │
│  - status: 202 Accepted                                       │
│  - processing: "queued"                                       │
│  - 但缩略图 URL 已经可用！（:small 已处理）                   │
└─────────────────────────────────────────────────────────────┘
         │
         ▼ (稍后，Sidekiq 异步执行)
┌─────────────────────────────────────────────────────────────┐
│  PostProcessMediaWorker#perform                              │
│                                                               │
│  1. processing = :in_progress                                 │
│  2. file.reprocess!(:original)  ◄─── 只处理 :original       │
│  3. processing = :complete                                    │
│  4. 保存元数据                                                 │
└─────────────────────────────────────────────────────────────┘
```

#### 设计意图

这个设计非常巧妙：

1. **用户体验优先**: 缩略图（`:small`）立即生成，用户在 UI 上能看到预览
2. **后台处理重任务**: 大文件转码（`:original`）放到后台，不阻塞请求
3. **状态可追踪**: `processing` 字段记录状态（queued → in_progress → complete/failed）

---

## 不同媒体类型的处理差异

### 处理器链选择逻辑

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

### 各类型处理器详解

#### 1. 图片 (JPG/PNG/WebP/HEIC/AVIF)

**处理器链**: `lazy_thumbnail` → `blurhash_transcoder` → `type_corrector`

**样式选择**:
```ruby
# app/models/media_attachment.rb:323-335
def file_styles(attachment)
  if IMAGE_CONVERTIBLE_MIME_TYPES.include?(...)  # heic, heif, avif
    IMAGE_CONVERTED_STYLES    # 转换为 JPEG
  elsif IMAGE_MIME_TYPES.include?(...)
    IMAGE_STYLES               # 保持原格式
  end
end
```

**IMAGE_CONVERTED_STYLES**:
```ruby
IMAGE_CONVERTED_STYLES = {
  original: { format: 'jpeg', content_type: 'image/jpeg', ... },
  small: { format: 'jpeg', ... }
}
```

**各处理器作用**:

| 处理器 | 作用 |
|--------|------|
| `lazy_thumbnail` | 智能调整尺寸，只在需要时转换；剥离元数据（仅本地上传） |
| `blurhash_transcoder` | 生成 Blurhash 字符串（用于加载占位图） |
| `type_corrector` | 修正文件扩展名（如 heic → jpg） |

#### 2. 静态 GIF

**处理器链**: 同图片 → `lazy_thumbnail` + `blurhash_transcoder` + `type_corrector`

**关键**: `GifTranscoder` 只在动画 GIF 时触发

```ruby
# lib/paperclip/gif_transcoder.rb:122-124
def needs_convert?
  GifReader.animated?(file.path)  # 检查是否有超过 1 帧
end
```

**GifReader 检测逻辑**:
```ruby
# lib/paperclip/gif_transcoder.rb:15-75
class GifReader
  def initialize(path, max_frames = 2)
    # 解析 GIF 文件结构，统计帧数
    # 只需要检测到第 2 帧就知道是动画
  end
  
  def self.animated?(path)
    new(path).animated  # @nb_frames > 1
  end
end
```

#### 3. 动画 GIF

**处理器链**: `gif_transcoder` → `blurhash_transcoder`

**样式**: `VIDEO_CONVERTED_STYLES`（转换为视频）

**转换过程**:
```ruby
# lib/paperclip/gif_transcoder.rb:105-118
class GifTranscoder < Paperclip::Processor
  def make
    return File.open(@file.path) unless needs_convert?  # 静态 GIF 跳过
    
    # 调用视频转码器转换为 MP4
    final_file = Paperclip::Transcoder.make(file, options, attachment)
    
    # 更新类型为 gifv
    if options[:style] == :original
      attachment.instance.file_file_name = "#{File.basename(..., '.*')}.mp4"
      attachment.instance.file_content_type = 'video/mp4'
      attachment.instance.type = MediaAttachment.types[:gifv]
    end
  end
end
```

**结果**: 动画 GIF 被转换为无声 MP4，类型标记为 `gifv`

#### 4. 视频 (MP4/WebM/MOV)

**处理器链**: `transcoder` → `blurhash_transcoder` → `type_corrector`

**样式选择**:
```ruby
def file_styles(attachment)
  if VIDEO_CONVERTIBLE_MIME_TYPES.include?(...)  # webm, quicktime
    VIDEO_CONVERTED_STYLES    # 强制转码
  elsif VIDEO_MIME_TYPES.include?(...)
    VIDEO_STYLES               # 可能透传
  end
end
```

**透传机制（关键优化）**:

```ruby
# lib/paperclip/transcoder.rb:114-116
def eligible_to_passthrough?(metadata)
  @passthrough_options && 
    @passthrough_options[:video_codecs].include?(metadata.video_codec) && 
    @passthrough_options[:audio_codecs].include?(metadata.audio_codec) && 
    @passthrough_options[:colorspaces].include?(metadata.colorspace)
end
```

**透传条件配置**:
```ruby
# app/models/media_attachment.rb:119-135
VIDEO_PASSTHROUGH_OPTIONS = {
  video_codecs: ['h264'].freeze,           # 必须是 H.264
  audio_codecs: ['aac', nil].freeze,        # AAC 或无音频
  colorspaces: ['yuv420p', 'yuvj420p'].freeze,
  options: {
    format: 'mp4',
    convert_options: {
      output: {
        'c:v' => 'copy',    # 视频流直接复制，不重新编码
        'c:a' => 'copy',    # 音频流直接复制，不重新编码
      }
    }
  }
}
```

**透传 vs 转码对比**:

| 条件 | 透传 | 转码 |
|------|------|------|
| 视频编码 | H.264 | 其他（VP8, VP9, 等） |
| 音频编码 | AAC 或无 | 其他（MP3, Vorbis, 等） |
| 色彩空间 | YUV420P / YUVJ420P | 其他 |
| 处理方式 | 直接复制流 | FFmpeg 重新编码 |
| 速度 | 极快 | 较慢（取决于长度） |
| CPU 占用 | 极低 | 高 |

**智能比特率计算**:
```ruby
# lib/paperclip/transcoder.rb:43-55
unless eligible_to_passthrough?(metadata)
  # BITS_PER_PIXEL = 0.11 (H.264 High 经验值)
  desired_bitrate = (metadata.width * metadata.height * 30 * BITS_PER_PIXEL).floor
  
  # 确保不超过文件大小限制（99MB）
  size_limit_in_bits = MediaAttachment::VIDEO_LIMIT * 8
  duration = [metadata.duration, 1].max
  maximum_bitrate = (size_limit_in_bits / duration).floor - 192_000  # 预留音频空间
  
  bitrate = [desired_bitrate, maximum_bitrate].min
  
  @output_options['b:v'] = bitrate
  @output_options['maxrate'] = bitrate + 192_000
  @output_options['bufsize'] = bitrate * 5
end
```

**类型修正**:
```ruby
# lib/paperclip/transcoder.rb:118-120
def update_attachment_type(metadata)
  # 无音频流的视频标记为 gifv
  @attachment.instance.type = MediaAttachment.types[:gifv] unless metadata.audio_codec
end
```

#### 5. 音频 (MP3/Ogg/FLAC/AAC/WAV)

**处理器链**: `image_extractor` → `transcoder` → `type_corrector`

**样式**: `AUDIO_STYLES`（转换为 MP3）

**特殊处理：封面图提取**:
```ruby
# lib/paperclip/image_extractor.rb:6-49
class ImageExtractor < Paperclip::Processor
  def make
    return @file unless options[:style] == :original
    
    # 从音频/视频中提取封面图
    image = extract_image_from_file!
    
    unless image.nil?
      begin
        # 保存为缩略图
        attachment.instance.thumbnail = image if image.size.positive?
      ensure
        # 清理临时文件
        image.close(true)
      end
    end
  end
  
  def extract_image_from_file!
    # 使用 FFmpeg 提取第一帧
    # ffmpeg -i source -loglevel fatal -y destination.png
    command = Terrapin::CommandLine.new(
      Rails.configuration.x.ffmpeg_binary,
      '-i :source -loglevel :loglevel -y :destination'
    )
  end
end
```

**音频转码配置**:
```ruby
# app/models/media_attachment.rb:154-165
AUDIO_STYLES = {
  original: {
    format: 'mp3',
    content_type: 'audio/mpeg',
    convert_options: {
      output: {
        'q:a' => 2,  # VBR 质量 (0-9，2 是高质量)
      }
    }
  }
}
```

### 媒体类型状态流转图

```
上传文件
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  检测 MIME 类型                                                   │
│  (before_file_validate: set_type_and_extension)                  │
└─────────────────────────────────────────────────────────────────┘
    │
    ├── image/gif ──────────────────────────────────────────────┐
    │                                                            │
    │    ▼                                                       │
    │  ┌─────────────────────┐                                  │
    │  │ GifReader.animated? │                                  │
    │  └───────────┬─────────┘                                  │
    │              │                                            │
    │     ┌────────┴────────┐                                   │
    │     │                 │                                   │
    │     ▼                 ▼                                   │
    │  静态 GIF          动画 GIF                                │
    │  type: image       type: (处理后变为 gifv)               │
    │  不转码             转码为 MP4                             │
    │                    处理器: gif_transcoder                 │
    │                                                            │
    ├── image/* ────────────────────────────────────────────────┤
    │                                                            │
    │    ▼                                                       │
    │  图片 (JPG/PNG/WebP/HEIC/AVIF)                           │
    │  type: image                                               │
    │  处理器: lazy_thumbnail → blurhash_transcoder → ...      │
    │  HEIC/AVIF 转换为 JPEG                                     │
    │                                                            │
    ├── video/* ────────────────────────────────────────────────┤
    │                                                            │
    │    ▼                                                       │
    │  视频检测                                                   │
    │  (check_video_dimensions 验证分辨率、帧率)                  │
    │                                                            │
    │  ┌─────────────────────────────────────────────────┐     │
    │  │ eligible_to_passthrough?                         │     │
    │  │ (H.264 + AAC + YUV420P)                          │     │
    │  └───────────────────────┬─────────────────────────┘     │
    │                          │                               │
    │            ┌─────────────┴─────────────┐                 │
    │            │                           │                 │
    │            ▼                           ▼                 │
    │        透传模式                      转码模式             │
    │        c:v copy, c:a copy          FFmpeg 重新编码       │
    │        极快，低 CPU                 较慢，高 CPU          │
    │                                                            │
    │  处理器: transcoder                                        │
    │                                                            │
    │  类型检测:                                                  │
    │  ┌─────────────────────────────────────────────────┐     │
    │  │ metadata.audio_codec.present?                    │     │
    │  └───────────────────────┬─────────────────────────┘     │
    │                          │                               │
    │            ┌─────────────┴─────────────┐                 │
    │            │                           │                 │
    │            ▼                           ▼                 │
    │        有音频                       无音频                │
    │        type: video               type: gifv              │
    │                                                            │
    └── audio/* ────────────────────────────────────────────────┤
                                                                 │
         ▼                                                       │
      音频文件                                                    │
      type: audio                                                │
                                                                 │
      处理器: image_extractor → transcoder → type_corrector     │
         │              │              │                         │
         │              │              └── 修正扩展名           │
         │              │                                         │
         │              └── 转码为 MP3 (q:a=2, VBR 高质量)     │
         │                                                        │
         └── 提取封面图 → 保存为 thumbnail                       │
                                                                 │
                                                                 │
  延迟处理？                                                       │
  ┌──────────────────────────────────────────────────────────┐  │
  │ delay_processing? = @delay_processing && larger_media_format? │
  │                                                              │  │
  │ larger_media_format? = video? || gifv? || audio?          │  │
  │                                                              │  │
  │ 结果:                                                         │  │
  │ - 图片/静态 GIF: delay_processing? = false → 同步处理       │  │
  │ - 视频/音频/动画 GIF:                                        │  │
  │   - API v1: @delay_processing = nil → delay_processing? = false │
  │   - API v2: @delay_processing = true → delay_processing? = true │
  └──────────────────────────────────────────────────────────┘  │
                                                                 │
                                                                 │
  最终类型总结:                                                  │
  ┌────────────┬──────────────────────────────────────────────┐ │
  │    type    │                   来源                        │ │
  ├────────────┼──────────────────────────────────────────────┤ │
  │   image    │ 静态图片 (JPG/PNG/WebP/HEIC/AVIF) + 静态 GIF │ │
  │   gifv     │ 动画 GIF + 无音频的视频                       │ │
  │   video    │ 有音频的视频                                   │ │
  │   audio    │ 音频文件                                       │ │
  │  unknown   │ 特殊情况（待处理）                             │ │
  └────────────┴──────────────────────────────────────────────┘ │
                                                                 │
┌────────────────────────────────────────────────────────────────┤
│                        缩略图处理 (:small)                       │
│                                                                 │
│  注意：:small 样式永远不会延迟！                                 │
│  即使 delay_processing? = true，:small 也会立即处理            │
│                                                                 │
│  各类型的 :small 样式处理:                                       │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ 图片    │ lazy_thumbnail → 生成缩小版图片                 │ │
│  │ GIF     │ 同图片                                          │ │
│  │ 视频    │ transcoder → 提取第 0 帧 → PNG 图片            │ │
│  │         │ time: 0, format: 'png'                         │ │
│  │ 音频    │ 依赖 image_extractor 提取的封面图               │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                 │
│  这样设计的目的：                                                │
│  1. UI 需要立即显示缩略图预览                                   │
│  2. :small 处理很快，不会阻塞请求                               │
│  3. :original 可能是大文件转码，放到后台处理                   │
└────────────────────────────────────────────────────────────────┘
```

---

## 存储系统实现

Mastodon 使用 Paperclip（kt-paperclip）作为文件存储抽象层，支持多种存储后端。

### 支持的存储类型

| 存储类型 | 环境变量开关 | 典型使用场景 |
|---------|-------------|-------------|
| **本地文件系统** | 默认 | 开发、小型单用户实例 |
| **AWS S3** | `S3_ENABLED=true` | 生产环境、大规模部署 |
| **S3 兼容服务** | `S3_ENABLED=true` + `S3_ENDPOINT` | MinIO, Backblaze B2, 等 |
| **OpenStack Swift** | `SWIFT_ENABLED=true` | OpenStack 云环境 |
| **Azure Blob** | `AZURE_ENABLED=true` | 微软 Azure 云 |

### 路径结构

**核心插值逻辑**:
```ruby
# config/initializers/paperclip.rb:6-30
PATH = ':prefix_url:class/:attachment/:id_partition/:style/:filename'

# 关键：本地 vs 远程的路径区分
Paperclip.interpolates :prefix_path do |attachment, _style|
  if attachment.storage_schema_version >= 1 && attachment.instance.respond_to?(:local?) && !attachment.instance.local?
    "cache#{File::SEPARATOR}"   # 远程缓存：cache/ 前缀
  else
    ''                           # 本地上传：无前缀
  end
end

Paperclip.interpolates :prefix_url do |attachment, _style|
  if attachment.storage_schema_version >= 1 && attachment.instance.respond_to?(:local?) && !attachment.instance.local?
    'cache/'   # URL 中的前缀
  else
    ''
  end
end
```

**实际路径示例**:

| 媒体类型 | 本地文件系统路径 | URL 路径 |
|---------|-----------------|---------|
| **本地上传图片** | `public/system/media_attachments/files/000/123/456/original/abc.jpg` | `/system/media_attachments/files/000/123/456/original/abc.jpg` |
| **远程缓存图片** | `public/system/cache/media_attachments/files/000/789/012/original/def.jpg` | `/system/cache/media_attachments/files/000/789/012/original/def.jpg` |

**`id_partition` 解释**:
- Paperclip 的分片机制，避免单目录文件过多
- 将数字 ID `123456` 转换为 `000/123/456`
- 每层 3 位数字，从右向左分组

### S3 配置详解

```ruby
# config/initializers/paperclip.rb:38-117
if ENV['S3_ENABLED'] == 'true'
  require 'aws-sdk-s3'
  
  # 基础配置
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
  
  # S3 兼容服务（MinIO 等）
  if ENV.key?('S3_ENDPOINT')
    Paperclip::Attachment.default_options[:s3_options].merge!(
      endpoint: ENV['S3_ENDPOINT'],
      force_path_style: ENV['S3_OVERRIDE_PATH_STYLE'] != 'true'
    )
    Paperclip::Attachment.default_options[:url] = ':s3_path_url'
  end
  
  # CDN / 自定义域名
  if ENV.key?('S3_ALIAS_HOST') || ENV.key?('S3_CLOUDFRONT_HOST')
    Paperclip::Attachment.default_options.merge!(
      url: ':s3_alias_url',
      s3_host_alias: ENV['S3_ALIAS_HOST'] || ENV['S3_CLOUDFRONT_HOST']
    )
  end
  
  # 存储类别
  Paperclip::Attachment.default_options[:s3_headers]['X-Amz-Storage-Class'] = ENV['S3_STORAGE_CLASS'] if ENV.key?('S3_STORAGE_CLASS')
  
  # S3 兼容扩展（解决部分服务的兼容性问题）
  module Paperclip
    module Storage
      module S3Extensions
        def copy_to_local_file(style, local_dest_path)
          options = {}
          options[:mode] = 'single_request' if ENV['S3_FORCE_SINGLE_REQUEST'] == 'true'
          options[:checksum_mode] = 'DISABLED' unless ENV['S3_ENABLE_CHECKSUM_MODE'] == 'true'
          s3_object(style).download_file(local_dest_path, options)
        end
      end
    end
  end
  
  Paperclip::Storage::S3.prepend(Paperclip::Storage::S3Extensions)
end
```

**S3 环境变量完整列表**:

| 环境变量 | 必需 | 默认值 | 说明 |
|---------|------|--------|------|
| `S3_ENABLED` | 是 | - | 设为 `true` 启用 S3 |
| `S3_BUCKET` | 是 | - | Bucket 名称 |
| `AWS_ACCESS_KEY_ID` | 是 | - | Access Key |
| `AWS_SECRET_ACCESS_KEY` | 是 | - | Secret Key |
| `S3_REGION` | 否 | `us-east-1` | AWS 区域 |
| `S3_ENDPOINT` | 否 | - | 自定义端点（用于 MinIO 等） |
| `S3_HOSTNAME` | 否 | `s3-{region}.amazonaws.com` | S3 主机名 |
| `S3_PROTOCOL` | 否 | `https` | 协议 |
| `S3_KEY_PREFIX` | 否 | - | 路径前缀 |
| `S3_ALIAS_HOST` / `S3_CLOUDFRONT_HOST` | 否 | - | CDN 域名 |
| `S3_PERMISSION` | 否 | `public-read` | 对象权限 |
| `S3_STORAGE_CLASS` | 否 | - | 存储类别（STANDARD, IA, GLACIER） |
| `S3_MULTIPART_THRESHOLD` | 否 | `15.megabytes` | 分块上传阈值 |
| `S3_SIGNATURE_VERSION` | 否 | `v4` | 签名版本 |
| `S3_FORCE_SINGLE_REQUEST` | 否 | - | 单请求下载（兼容某些服务） |

### 本地文件系统配置

```ruby
# config/initializers/paperclip.rb:162-169
else
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

**本地存储环境变量**:

| 环境变量 | 默认值 | 说明 |
|---------|--------|------|
| `PAPERCLIP_ROOT_PATH` | `:rails_root/public/system` | 存储根路径 |
| `PAPERCLIP_ROOT_URL` | `/system` | URL 前缀 |

---

## 远程媒体缓存完整生命周期

这是 Mastodon 联邦网络中最复杂也最容易被误解的部分。让我们深度解析。

### 核心概念

| 概念 | 定义 | 数据库字段 |
|------|------|-----------|
| **本地媒体** | 本实例用户上传的媒体 | `remote_url = ""` |
| **远程媒体** | 来自其他实例的媒体 | `remote_url = "https://other.instance/..."` |
| **已缓存** | 远程媒体已下载到本地 | `file_file_name` 非空 |
| **需重下** | 远程媒体缓存已过期/被清理 | `file_file_name` 为空但 `remote_url` 存在 |

**关键方法**:
```ruby
# app/models/media_attachment.rb:231-241
def local?
  remote_url.blank?
end

def needs_redownload?
  file.blank? && remote_url.present?
end
```

**作用域**:
```ruby
# app/models/media_attachment.rb:212-217
scope :attached, -> { where.not(status_id: nil).or(where.not(scheduled_status_id: nil)) }
scope :cached, -> { remote.where.not(file_file_name: nil) }   # 已缓存的远程媒体
scope :remote, -> { where.not(remote_url: '') }                # 所有远程媒体
scope :local, -> { where(remote_url: '') }                      # 本地上传
scope :unattached, -> { where(status_id: nil, scheduled_status_id: nil) }  # 孤儿
```

### 阶段 1：初始下载

#### 触发时机：ActivityPub Create

当从其他实例接收新帖子时：

```ruby
# app/lib/activitypub/activity/create.rb:300-325
def process_attachments
  # ...
  
  # 1. 创建 MediaAttachment 记录
  media_attachment = MediaAttachment.create(
    account: @account,
    remote_url: media_attachment_parser.remote_url,           # 保存远程 URL
    thumbnail_remote_url: media_attachment_parser.thumbnail_remote_url,
    description: media_attachment_parser.description,
    focus: media_attachment_parser.focus,
    blurhash: media_attachment_parser.blurhash
  )
  
  # 2. 检查是否跳过下载
  next if unsupported_media_type?(media_attachment_parser.file_content_type) || skip_download?
  
  # 3. 同步下载！
  media_attachment.download_file!
  media_attachment.download_thumbnail!
  media_attachment.save
  
  # 4. 失败处理
rescue Mastodon::UnexpectedResponseError, *Mastodon::HTTP_CONNECTION_ERRORS
  # 网络错误 → 稍后重试
  RedownloadMediaWorker.perform_in(rand(PROCESSING_DELAY), media_attachment.id)
rescue Seahorse::Client::NetworkingError => e
  # S3 存储错误 → 立即重试
  RedownloadMediaWorker.perform_async(media_attachment.id)
end
```

#### 跳过下载的条件

```ruby
# app/lib/activitypub/activity/create.rb:435-439
def skip_download?
  return @skip_download if defined?(@skip_download)
  
  # 如果来源域名被配置为"拒绝媒体"，则跳过下载
  @skip_download ||= DomainBlock.reject_media?(@account.domain)
end
```

#### `download_file!` 的实现

通过 `Remotable` 关注点：

```ruby
# app/models/concerns/remotable.rb:10-49
def download_file!(url = nil)
  url ||= self[:remote_url]
  return if url.blank?
  
  # 解析并验证 URL
  begin
    parsed_url = Addressable::URI.parse(url).normalize
  rescue Addressable::URI::InvalidURIError
    return
  end
  
  return if !%w(http https).include?(parsed_url.scheme) || parsed_url.host.blank?
  
  # 发起 HTTP 请求下载
  begin
    Request.new(:get, url).perform do |response|
      raise Mastodon::UnexpectedResponseError, response unless (200...300).cover?(response.code)
      
      # ResponseWithLimit 限制下载大小，防止 OOM
      # limit = VIDEO_LIMIT = 99.megabytes
      self.file = ResponseWithLimit.new(response, limit)
    end
  rescue Mastodon::UnexpectedResponseError, *Mastodon::HTTP_CONNECTION_ERRORS => e
    # 下载失败，清除现有文件
    self.file = nil if file_file_name.present?
    raise e unless suppress_errors
  rescue Paperclip::Errors::NotIdentifiedByImageMagickError, ... => e
    # 格式错误等，静默处理
    self.file = nil if file_file_name.present?
  end
end
```

### 阶段 2：缓存过期与清理

#### 清理的两个层级

Mastodon 有两个独立的清理机制，用途完全不同：

| 层级 | 触发配置 | 清理对象 | 操作 | 可恢复？ |
|------|---------|---------|------|---------|
| **缓存清理** | `media_cache_retention_period` | 媒体文件 | 删除文件，保留记录 | ✅ 可重新下载 |
| **记录删除** | `content_cache_retention_period` | Status 记录 | 删除记录 + 级联删除媒体 | ❌ 不可恢复 |

#### 层级 1：媒体缓存清理 (`MediaAttachmentsVacuum`)

**触发**:
```ruby
# app/workers/scheduler/vacuum_scheduler.rb:34-36
def media_attachments_vacuum
  Vacuum::MediaAttachmentsVacuum.new(content_retention_policy.media_cache_retention_period)
end

# app/models/content_retention_policy.rb:8-10
def media_cache_retention_period
  retention_period Setting.media_cache_retention_period
end

def retention_period(value)
  value.days if value.is_a?(Integer) && value.positive?
  # nil 或 0 表示不清理
end
```

**清理逻辑**:
```ruby
# app/lib/vacuum/media_attachments_vacuum.rb:10-49
def perform
  vacuum_orphaned_records!      # 清理孤儿记录
  vacuum_cached_files! if retention_period?  # 清理过期缓存（如果配置了保留期）
end

def vacuum_cached_files!
  media_attachments_past_retention_period.find_in_batches do |media_attachments|
    # clear = 删除文件，保留记录
    AttachmentBatch.new(MediaAttachment, media_attachments).clear
  end
end

def media_attachments_past_retention_period
  MediaAttachment
    .remote                            # 远程媒体
    .cached                            # 已缓存（有 file_file_name）
    .created_before(@retention_period.ago)    # 创建时间早于 N 天前
    .updated_before(@retention_period.ago)    # 更新时间早于 N 天前
end
```

**关键发现**: `without_local_interaction` **没有被使用**！

自动缓存清理只看时间，不考虑是否有本地交互（收藏、书签等）。

#### `AttachmentBatch#clear` 操作

```ruby
# app/lib/attachment_batch.rb:38-41
def clear
  remove_files              # 删除存储中的文件
  batch.update_all(nullified_attributes)  # 把数据库字段设为 nil
end

def nullified_attributes
  # 结果示例: { "file_file_name" => nil, "file_content_type" => nil, ... }
  @attachment_names.flat_map { |attachment_name| 
    NULLABLE_ATTRIBUTES.map { |attribute| "#{attachment_name}_#{attribute}" } & klass.column_names 
  }.index_with(nil)
end
```

**清理后的状态**:
| 字段 | 清理前 | 清理后 |
|------|--------|--------|
| `remote_url` | `"https://other.instance/..."` | **不变** |
| `file_file_name` | `"abc123.jpg"` | `nil` |
| `file_content_type` | `"image/jpeg"` | `nil` |
| `created_at` | `[下载时间]` | **不变** |
| `updated_at` | `[下载时间]` | **不变** |

**关键**：`remote_url` 保留，所以 `needs_redownload?` 变为 `true`，可以重新下载。

#### 层级 2：内容记录删除 (`StatusesVacuum`)

这是**真正的硬删除**，不可恢复。

**触发**:
```ruby
# app/workers/scheduler/vacuum_scheduler.rb:30-32
def statuses_vacuum
  Vacuum::StatusesVacuum.new(content_retention_policy.content_cache_retention_period)
end
```

**清理逻辑**:
```ruby
# app/lib/vacuum/statuses_vacuum.rb:16-42
def vacuum_statuses!
  statuses_scope.in_batches do |statuses|
    # 1. 清理关联（会话、搜索索引等）
    statuses.direct_visibility.includes(mentions: :account).find_each(&:unlink_from_conversations!)
    if Chewy.enabled?
      remove_from_index(statuses.ids, 'chewy:queue:StatusesIndex')
      remove_from_index(statuses.ids, 'chewy:queue:PublicStatusesIndex')
    end
    
    # 2. 删除 status 记录
    # 注意：外键会自动处理大部分关联记录
    # 但 media_attachments 会变成"孤儿"（status_id = nil）
    statuses.delete_all
  end
end

def statuses_scope
  Status.unscoped.kept
    .joins(:account).merge(Account.remote)           # 只清理远程账户的 status
    .where(statuses: { id: ...retention_period_as_id })  # 超过保留期
end
```

**级联效应**:

当 `status` 被删除后，其 `media_attachments` 的 `status_id` 变为 `nil`，变成"孤儿"。

然后 `MediaAttachmentsVacuum#vacuum_orphaned_records!` 会清理这些孤儿：

```ruby
# app/lib/vacuum/media_attachments_vacuum.rb:25-31
def vacuum_orphaned_records!
  orphaned_media_attachments.find_in_batches do |media_attachments|
    # delete = 删除文件 + 删除数据库记录
    AttachmentBatch.new(MediaAttachment, media_attachments).delete
  end
end

def orphaned_media_attachments
  MediaAttachment
    .unattached           # status_id = nil AND scheduled_status_id = nil
    .created_before(TTL.ago)  # TTL = 1.day
end
```

**`AttachmentBatch#delete` 操作**:
```ruby
# app/lib/attachment_batch.rb:33-36
def delete
  remove_files          # 删除文件
  batch.delete_all      # 删除数据库记录！
end
```

**结果**：记录完全消失，`remote_url` 也没了，**无法再重新下载**。

#### 设计警告

从 locale 文件可以看到 `content_cache_retention_period` 的强烈警告：

> "All posts from other servers (including boosts and replies) will be deleted after the specified number of days, **regardless of any local user interaction** with those posts. This includes posts where a local user has bookmarked or favorited them. Private mentions between users from different instances will also be lost and cannot be recovered. Use of this setting is intended for special-purpose instances and breaks many user expectations when implemented for general-purpose use."

**翻译**:
> "来自其他服务器的所有帖子（包括 boosts 和回复）将在指定天数后被删除，**无论本地用户是否与这些帖子有任何交互**。这包括本地用户已添加书签或收藏的帖子。不同实例用户之间的私人提及也将丢失且无法恢复。此设置专为特殊用途实例设计，在通用实例上使用会破坏许多用户预期。"

**关键设计取舍**:
- `media_cache_retention_period`: 相对安全，只删文件缓存，可重下
- `content_cache_retention_period`: 危险！删除记录，不可恢复，且忽略本地交互

### 阶段 3：重新下载

#### 触发时机：用户访问

当用户访问已过期缓存的媒体时：

```ruby
# app/controllers/media_proxy_controller.rb:11-44
before_action :set_media_attachment

def show
  # 检查是否需要重新下载
  if @media_attachment.needs_redownload? && !reject_media?
    # 使用 Redis 锁防止并发下载
    with_redis_lock("media_download:#{params[:id]}") do
      # 重新加载（双重检查，避免锁等待期间已被其他进程下载）
      @media_attachment.reload
      redownload! if @media_attachment.needs_redownload?
    end
  end
  
  # 重定向到文件或流式传输
  if requires_file_streaming?
    send_file(...)
  else
    redirect_to media_attachment_file_path, allow_other_host: true
  end
end

def redownload!
  @media_attachment.download_file!
  @media_attachment.download_thumbnail!
  @media_attachment.created_at = Time.now.utc  # 关键！更新 created_at
  @media_attachment.save!
end

def set_media_attachment
  @media_attachment = MediaAttachment.attached.find(params[:id])
  authorize @media_attachment, :download?
end
```

#### 关键点：时间戳重置

```ruby
@media_attachment.created_at = Time.now.utc
```

这意味着：
1. 重新下载后，`created_at` 被更新为当前时间
2. Rails 的 `save!` 会自动更新 `updated_at`
3. **下次清理检查时**：
   - `created_before(@retention_period.ago)` → false（太新）
   - `updated_before(@retention_period.ago)` → false（太新）
4. **不会被立即清理**

这实际上实现了一个 **LRU (Least Recently Used) 风格的缓存策略**：
- 热门媒体（频繁被访问）会不断更新时间戳，永远不会被清理
- 冷门媒体（长期无人访问）会被清理，节省存储空间
- 需要时可以重新下载

### 阶段 4：更新时的重新下载

当远程帖子的媒体 URL 变化时：

```ruby
# app/services/activitypub/process_status_update_service.rb:126-140
def download_media_files!
  @next_media_attachments.each do |media_attachment|
    next if media_attachment.skip_download
    
    # 只有当 URL 变化时才重新下载
    media_attachment.download_file! if media_attachment.remote_url_previously_changed?
    media_attachment.download_thumbnail! if media_attachment.thumbnail_remote_url_previously_changed?
    media_attachment.save
    
  rescue Mastodon::UnexpectedResponseError, *Mastodon::HTTP_CONNECTION_ERRORS
    # 失败则调度 worker 稍后重试
    RedownloadMediaWorker.perform_in(rand(PROCESSING_DELAY), media_attachment.id)
  end
end
```

### 远程缓存生命周期完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                           远程媒体缓存完整生命周期                                                 │
├─────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                   │
│  ┌──────────────────┐                                                                             │
│  │   远程实例        │                                                                             │
│  │   发布新帖子      │                                                                             │
│  └────────┬─────────┘                                                                             │
│           │                                                                                        │
│           │ ActivityPub: Create / Announce                                                        │
│           ▼                                                                                        │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────┐│
│  │ ActivityPub::Activity::Create#process_attachments                                            ││
│  │                                                                                                ││
│  │ 1. 创建 MediaAttachment 记录                                                                   ││
│  │    - remote_url = "https://other.instance/media/xxx"                                          ││
│  │    - file_file_name = nil                                                                      ││
│  │    - created_at = Time.now                                                                     ││
│  │    - updated_at = Time.now                                                                     ││
│  │                                                                                                ││
│  │ 2. 检查是否跳过下载                                                                             ││
│  │    - if DomainBlock.reject_media?(domain) → skip                                              ││
│  │    - if unsupported_media_type → skip                                                          ││
│  │                                                                                                ││
│  │ 3. 同步下载                                                                                     ││
│  │    - media_attachment.download_file!                                                           ││
│  │    - 通过 Remotable 机制发起 HTTP 请求                                                         ││
│  │    - ResponseWithLimit 限制大小（最大 99MB）                                                   ││
│  │                                                                                                ││
│  │ 4. 下载后的状态                                                                                 ││
│  │    - file_file_name = "abc123.jpg" (已缓存)                                                   ││
│  │    - scope: .remote.cached                                                                     ││
│  │    - needs_redownload? = false                                                                 ││
│  │                                                                                                ││
│  │ 5. 失败处理                                                                                     ││
│  │    - 网络错误 → RedownloadMediaWorker.perform_in(rand(delay), id)                            ││
│  │    - S3 错误 → RedownloadMediaWorker.perform_async(id)                                        ││
│  └─────────────────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                                   │
│           │                                                                                        │
│           ▼                                                                                        │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────┐│
│  │ 状态：缓存已就绪                                                                                ││
│  │ - remote_url: "https://other.instance/media/xxx" (保留)                                       ││
│  │ - file_file_name: "abc123.jpg"                                                                 ││
│  │ - created_at: [下载时间]                                                                        ││
│  │ - updated_at: [下载时间]                                                                        ││
│  │ - local? = false, needs_redownload? = false                                                    ││
│  └─────────────────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                                   │
│           │                                                                                        │
│           ├──────────────────────────────────────────────────────────────────────────────────────┤│
│           │                                                                                       ││
│           ▼                                                                                       ▼│
│  ┌──────────────────────────┐                                           ┌─────────────────────────┐│
│  │ 用户访问媒体              │                                           │ 时间流逝，超过保留期     ││
│  │                          │                                           │ media_cache_retention_period │
│  │ GET /media_proxy/:id     │                                           │                         ││
│  │                          │                                           │ VacuumScheduler 定期触发   ││
│  └────────────┬─────────────┘                                           └───────────┬─────────────┘│
│               │                                                                       │              │
│               │                                                                       │              │
│               │                                                               ┌───────▼──────┐       │
│               │                                                               │              │       │
│               │                                                               ▼              │       │
│               │                                                    ┌──────────────────────┐       │
│               │                                                    │ MediaAttachmentsVacuum│       │
│               │                                                    │                      │       │
│               │                                                    │ 检查条件：            │       │
│               │                                                    │ .remote               │       │
│               │                                                    │ .cached               │       │
│               │                                                    │ .created_before(retention.ago) │
│               │                                                    │ .updated_before(retention.ago) │
│               │                                                    │                      │       │
│               │                                                    │ 操作：AttachmentBatch.clear │
│               │                                                    │ - 删除文件            │       │
│               │                                                    │ - 保留记录            │       │
│               │                                                    │ - remote_url 不变     │       │
│               │                                                    └───────────┬──────────┘       │
│               │                                                                │                  │
│               │                                                                ▼                  │
│               │                                                    ┌──────────────────────┐       │
│               │                                                    │ 状态：缓存已清理      │       │
│               │                                                    │                      │       │
│               │                                                    │ remote_url: 保留      │       │
│               │                                                    │ file_file_name: nil   │       │
│               │                                                    │ created_at: 不变      │       │
│               │                                                    │ updated_at: 不变      │       │
│               │                                                    │ needs_redownload? = true │   │
│               │                                                    └───────────┬──────────┘       │
│               │                                                                │                  │
│               └────────────────────────────────────────────────────────────────┘                  │
│                                               │                                                        │
│                                               │ 用户再次访问                                            │
│                                               ▼                                                        │
│                                    ┌──────────────────────────┐                                          │
│                                    │ MediaProxyController#show │                                          │
│                                    │                          │                                          │
│                                    │ 1. needs_redownload?     │                                          │
│                                    │    = file.blank? &&      │                                          │
│                                    │      remote_url.present? │                                          │
│                                    │    = true                 │                                          │
│                                    │                          │                                          │
│                                    │ 2. with_redis_lock       │                                          │
│                                    │    (防止并发下载)         │                                          │
│                                    │                          │                                          │
│                                    │ 3. redownload!           │                                          │
│                                    │    - download_file!      │                                          │
│                                    │    - download_thumbnail! │                                          │
│                                    │    - created_at = now! ◄── 关键！重置时间戳                     │
│                                    │    - save!               │                                          │
│                                    │                          │                                          │
│                                    │ 结果：                    │                                          │
│                                    │ - file_file_name 恢复    │                                          │
│                                    │ - created_at 更新        │                                          │
│                                    │ - 不会被立即清理          │                                          │
│                                    └──────────────────────────┘                                          │
│                                                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 另一条路径：真正的删除（不可恢复）                                                              │  │
│  │                                                                                                │  │
│  │ 触发条件：content_cache_retention_period 已配置                                               │  │
│  │                                                                                                │  │
│  │ Vacuum::StatusesVacuum                                                                        │  │
│  │ - 选择远程账户的 status                                                                        │  │
│  │ - 超过保留期                                                                                   │  │
│  │ - **忽略本地交互！**（收藏、书签都没用）                                                      │  │
│  │                                                                                                │  │
│  │ 操作：statuses.delete_all                                                                      │  │
│  │ - 删除 status 记录                                                                             │  │
│  │ - media_attachments.status_id 变为 nil → 变成"孤儿"                                         │  │
│  │                                                                                                │  │
│  │ 然后：MediaAttachmentsVacuum#vacuum_orphaned_records!                                        │  │
│  │ - 选择 unattached (status_id = nil)                                                           │  │
│  │ - created_before(1.day.ago)                                                                   │  │
│  │                                                                                                │  │
│  │ 操作：AttachmentBatch.delete                                                                   │  │
│  │ - 删除文件                                                                                     │  │
│  │ - 删除数据库记录！                                                                             │  │
│  │ - remote_url 也没了                                                                            │  │
│  │                                                                                                │  │
│  │ 结果：**完全消失，无法恢复**                                                                   │  │
│  │                                                                                                │  │
│  │ ⚠️  警告：这是危险操作！                                                                       │  │
│  │ - 即使是本地用户收藏的帖子也会被删除                                                          │  │
│  │ - 不同实例用户之间的私人提及也会丢失                                                          │  │
│  │ - 专为特殊用途实例设计，通用实例不推荐使用                                                    │  │
│  └─────────────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                                           │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 缓存保留策略的设计取舍

### 两个独立的保留策略

Mastodon 有两套完全独立的保留策略，设计目标截然不同：

| 策略 | 配置项 | 清理对象 | 风险等级 | 设计目标 |
|------|--------|---------|---------|---------|
| **媒体缓存保留** | `media_cache_retention_period` | 媒体文件 | 🟢 低 | 节省存储空间，保持可恢复性 |
| **内容缓存保留** | `content_cache_retention_period` | Status 记录 | 🔴 高 | 极端隐私/存储场景，破坏性 |

### 媒体缓存保留策略设计分析

#### 核心设计：LRU 风格的缓存

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    媒体缓存策略的核心洞察                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  传统 LRU (Least Recently Used) 缓存：                                  │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐                           │
│  │ A │ B │ C │ D │ E │ F │ G │ H │ I │ J │  ← 缓存满了               │
│  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘                           │
│         │                                                                 │
│         └── 访问 B → 移到最右端                                          │
│                                                                         │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐                           │
│  │ A │ C │ D │ E │ F │ G │ H │ I │ J │ B │  ← B 现在"最热"           │
│  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘                           │
│                                                                         │
│  新元素 K 需要插入：                                                      │
│  - 淘汰最左端（最久未使用）的 A                                          │
│  - K 插入到最右端                                                         │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Mastodon 的"时间戳 LRU"：                                              │
│                                                                         │
│  不用链表，用时间戳：                                                    │
│  - 每次访问（重新下载）→ created_at = now                               │
│  - 清理时 → 删除 created_at < retention_period.ago 的                   │
│                                                                         │
│  效果等价：                                                              │
│  ┌──────────────────────────────────────────────────────────────┐     │
│  │ 时间轴                                                          │     │
│  │ ───────────────────────────────────────────────────────────▶ │     │
│  │                                                                  │     │
│  │ [保留期]                    [现在]                               │     │
│  │ <───────────────────────────▶                                   │     │
│  │                                                                  │     │
│  │ 媒体 A [created: 30天前] ──┐                                    │     │
│  │ 媒体 B [created: 20天前]   │  超过保留期 → 被清理              │     │
│  │ 媒体 C [created: 15天前] ──┘                                    │     │
│  │                                                                  │     │
│  │ 媒体 D [created: 10天前，3天前被重下 → updated: 3天前]          │     │
│  │ 媒体 E [created: 5天前]                                          │     │
│  │ 媒体 F [created: 1天前]   ──┐  在保留期内 → 保留               │     │
│  │ 媒体 G [created: 今天]    ──┘                                    │     │
│  └──────────────────────────────────────────────────────────────┘     │
│                                                                         │
│  关键：媒体 D 虽然创建于 10 天前，但 3 天前被访问过（重下），          │
│       所以不会被清理。                                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 策略的优点

| 优点 | 说明 |
|------|------|
| **自动冷热分离** | 热门媒体保持缓存，冷门媒体自动清理 |
| **可恢复性** | 只删文件，保留 `remote_url`，需要时可重下 |
| **存储效率** | 只保留实际需要的数据，不浪费空间 |
| **配置简单** | 只需配置天数，不用复杂的缓存大小计算 |

#### 策略的权衡

| 权衡 | 说明 |
|------|------|
| **清理时间不精确** | 基于定期任务（通常每天一次），不是精确到秒 |
| **首次访问延迟** | 缓存失效后首次访问需要等待重新下载 |
| **源实例依赖** | 如果源实例下线或删除了媒体，无法重下 |
| **无大小限制** | 只看时间，不看缓存总大小。热门媒体多的话可能用很多空间 |

#### 与 `without_local_interaction` 的关系

之前发现自动清理**不使用** `without_local_interaction`，但 CLI 有 `--keep-interacted` 选项。

**设计意图分析**:

```ruby
# lib/mastodon/cli/media.rb:67-87
def remove
  attachment_scope = MediaAttachment.cached.remote.where(created_at: ..time_ago)
  
  # 只有 --keep-interacted 选项时才过滤
  attachment_scope = attachment_scope.without_local_interaction if options[:keep_interacted]
  
  # ... 清理
end
```

**为什么自动清理不考虑交互？**

可能的设计原因：

1. **性能考虑**:
   ```ruby
   # without_local_interaction 包含 6 个 EXISTS 子查询！
   scope :without_local_interaction, lambda {
     where.not(Favourite...exists)      # 1. 收藏
       .where.not(Bookmark...exists)     # 2. 书签
       .where.not(Status.local...exists) # 3. 回复
       .where.not(Status.local...exists) # 4. 转嘟
       .where.not(Quote...exists)        # 5. 引用（作为引用者）
       .where.not(Quote...exists)        # 6. 引用（作为被引用者）
   }
   ```
   
   这些都是关联子查询，在大数据集上可能很慢。

2. **语义考虑**:
   - "交互过"只是表示用户曾经感兴趣，不代表永远需要
   - 如果用户真的想永久保存，可以下载到本地
   - 重新下载成本很低（源实例在线的话）

3. **CLI 选项的用途**:
   - `--keep-interacted` 是给管理员手动清理时用的
   - 让管理员可以选择更保守的清理策略
   - 例如：`tootctl media remove --days=7 --keep-interacted`

### 内容缓存保留策略设计分析

#### 这是一个"核选项"

从 locale 文件的强烈警告可以看出，这不是给普通实例用的：

> "Use of this setting is intended for **special-purpose instances** and breaks **many user expectations** when implemented for **general-purpose use**."

#### 策略行为

| 特性 | 行为 |
|------|------|
| **删除范围** | 所有远程帖子，**无论是否有本地交互** |
| **包含内容** | 原始帖子、转嘟、回复、私人提及 |
| **可恢复性** | ❌ 完全不可恢复 |
| **级联删除** | Status → MediaAttachment（变成孤儿后删除） |

#### 设计目标：极端隐私场景

这个策略可能针对以下特殊场景：

1. **高安全隐私实例**:
   - 不希望在服务器上永久存储任何来自外部的数据
   - 即使是用户收藏的内容也只保留有限时间
   - 合规要求：某些地区或行业可能要求定期清理数据

2. **临时/ disposable 实例**:
   - 用于短期活动或测试
   - 不期望永久保留任何数据
   - 定期自动清理降低运维成本

3. **极端存储限制**:
   - 存储空间极其有限
   - 愿意牺牲用户体验来节省空间
   - 接受数据丢失的风险

#### 为什么这是一个"核选项"

从设计上看，这个策略有几个"故意为之"的激进特性：

| 特性 | 设计意图 | 用户体验影响 |
|------|---------|-------------|
| **忽略本地交互** | 简单、可预测、无例外 | 收藏的帖子也会消失，违反直觉 |
| **级联删除媒体** | 彻底清理，不留痕迹 | 媒体完全无法恢复 |
| **无排除机制** | 配置简单，行为一致 | 管理员无法保护特定数据 |

#### 与媒体缓存策略的对比

| 维度 | 媒体缓存保留 | 内容缓存保留 |
|------|------------|------------|
| **设计哲学** | 缓存优化 | 数据清理 |
| **可恢复性** | ✅ 可重新下载 | ❌ 完全删除 |
| **用户体验** | 透明，仅首次访问慢 | 破坏性，数据消失 |
| **适用场景** | 所有实例 | 仅特殊用途实例 |
| **配置默认** | 建议启用 | 建议禁用 |

---

## 总结

### 关键修正

本文档纠正了之前的几个重要误解：

#### 1. API v1 与 v2 的精确行为

❌ **之前的错误理解**:
- API v1 可能返回 206
- API v2 对所有媒体类型延迟处理

✅ **精确理解**:
- **API v1**: 永远同步处理，永远返回 200（`delay_processing` 未设置）
- **API v2**: 只对视频/音频/动画 GIF 延迟处理（返回 202）；对图片仍同步处理（返回 200）
- **关键条件**: `delay_processing? = @delay_processing && larger_media_format?`

#### 2. 延迟处理的精确含义

❌ **之前的错误理解**:
- 延迟处理会跳过所有样式的处理

✅ **精确理解**:
- **只跳过 `:original` 样式**（大文件转码）
- **`:small` 样式永远立即处理**（缩略图，UI 需要预览）
- **设计意图**: 用户体验优先，后台处理重任务

#### 3. 远程缓存的完整生命周期

❌ **之前的错误理解**:
- 清理后无法恢复
- 自动清理考虑本地交互

✅ **精确理解**:
- **媒体缓存清理** (`media_cache_retention_period`): 只删文件，保留 `remote_url`，可重下
- **真正删除** (`content_cache_retention_period`): 删除记录，不可恢复，**忽略本地交互**
- **LRU 风格**: 用户访问时触发重新下载并更新 `created_at`，热门媒体自动"续期"

### 核心架构决策总结

#### 1. 上传处理策略

```
┌─────────────────────────────────────────────────────────────────┐
│                    上传处理决策树                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  上传请求                                                        │
│     │                                                           │
│     ├── API v1 ──────────────────────────────────────────┐     │
│     │    @delay_processing = nil                          │     │
│     │    delay_processing? = false                        │     │
│     │    processing = :complete                           │     │
│     │    所有样式同步处理                                  │     │
│     │    返回 200                                         │     │
│     └─────────────────────────────────────────────────────┘     │
│                                                                 │
│     └── API v2 ──────────────────────────────────────────┐     │
│          @delay_processing = true                         │     │
│          │                                                 │     │
│          ├── 图片/静态 GIF                                │     │
│          │   larger_media_format? = false                │     │
│          │   delay_processing? = false                    │     │
│          │   processing = :complete                       │     │
│          │   所有样式同步处理                              │     │
│          │   返回 200                                     │     │
│          │                                                 │     │
│          └── 视频/音频/动画 GIF                          │     │
│              larger_media_format? = true                 │     │
│              delay_processing? = true                    │     │
│              processing = :queued                        │     │
│              :original 延迟（PostProcessMediaWorker）    │     │
│              :small 立即处理（缩略图）                    │     │
│              返回 202                                     │     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 2. 缓存生命周期策略

```
┌─────────────────────────────────────────────────────────────────┐
│                    缓存生命周期决策树                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  远程媒体                                                         │
│     │                                                           │
│     ├── 初始获取（ActivityPub Create）                          │
│     │   ├── 创建记录，设置 remote_url                          │
│     │   ├── 同步下载（除非 skip_download?）                     │
│     │   └── 失败 → RedownloadMediaWorker 重试                 │
│     │                                                           │
│     ├── 缓存过期检查（定期调度）                                 │
│     │   │                                                       │
│     │   ├── media_cache_retention_period 已配置？              │
│     │   │   │                                                   │
│     │   │   ├── 是 → 检查时间戳                                │
│     │   │   │   ├── created_at < retention.ago?               │
│     │   │   │   ├── updated_at < retention.ago?               │
│     │   │   │   └── 都满足 → AttachmentBatch.clear            │
│     │   │   │           ├── 删除文件                          │
│     │   │   │           ├── 保留记录                          │
│     │   │   │           └── remote_url 保留 → 可重下         │
│     │   │   │                                                   │
│     │   │   └── 否 → 不清理                                   │
│     │   │                                                       │
│     │   └── content_cache_retention_period 已配置？            │
│     │       │                                                   │
│     │       ├── 是 → StatusesVacuum                           │
│     │       │   ├── 删除远程 status（忽略本地交互！）          │
│     │       │   ├── media_attachments 变成孤儿                │
│     │       │   └── 1 天后 → AttachmentBatch.delete          │
│     │       │           ├── 删除文件                          │
│     │       │           ├── 删除记录                          │
│     │       │           └── remote_url 丢失 → 不可恢复       │
│     │       │                                                   │
│     │       └── 否 → 不删除                                   │
│     │                                                           │
│     └── 用户访问（MediaProxyController）                        │
│         ├── needs_redownload? = file.blank? && remote_url?   │
│         ├── 是 → 重新下载                                      │
│         │   ├── download_file! + download_thumbnail!          │
│         │   ├── created_at = now! ◄── 关键：重置时间戳       │
│         │   └── 下次清理检查时：太新，不清理                  │
│         │                                                      │
│         └── 否 → 直接提供文件                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 3. 两级清理策略对比

| 维度 | 媒体缓存清理 | 内容记录删除 |
|------|------------|------------|
| **配置项** | `media_cache_retention_period` | `content_cache_retention_period` |
| **触发条件** | 远程 + 已缓存 + 时间过期 | 远程 status + 时间过期 |
| **操作** | `AttachmentBatch.clear` | `Status.delete_all` + 级联 |
| **文件处理** | 删除 | 删除 |
| **记录处理** | 保留（字段置空） | 删除 |
| **remote_url** | 保留 | 丢失 |
| **可恢复** | ✅ 可重新下载 | ❌ 不可恢复 |
| **考虑交互** | ❌ 忽略 | ❌ 忽略（更激进！） |
| **风险等级** | 🟢 低 | 🔴 高 |
| **推荐配置** | 建议启用（如 7 天） | 建议禁用（仅特殊实例） |

### 设计原则与权衡

Mastodon 的媒体处理系统体现了以下设计原则：

#### 1. 用户体验优先

- **缩略图立即生成**: `:small` 样式永不延迟，确保 UI 响应
- **缓存透明**: 媒体缓存清理对用户透明（除了首次访问稍慢）
- **渐进式加载**: Blurhash 提供占位图，优化感知性能

#### 2. 性能与资源优化

- **延迟处理大文件**: 视频转码放到后台，不阻塞请求
- **LRU 缓存策略**: 时间戳+访问驱动的缓存淘汰，自动冷热分离
- **视频透传优化**: 符合条件的 H.264/AAC 视频直接复用流，不重新编码

#### 3. 安全性与隐私

- **文件大小限制**: 图片 16MB，视频 99MB，防止 DoS
- **元数据剥离**: 本地上传图片自动剥离 EXIF 等元数据
- **两级清理**: 提供从"安全缓存"到"彻底删除"的选项，适应不同隐私需求

#### 4. 联邦网络特性

- **远程媒体缓存**: 提升用户体验，减轻源实例压力
- **可恢复性设计**: 保留 `remote_url`，缓存失效后可重下
- **域名级别控制**: `DomainBlock.reject_media?` 可跳过特定域名的媒体下载

### 操作建议

基于以上分析，给出以下建议：

#### 对于普通实例

1. **媒体缓存保留期**: 建议设置为 7-30 天
   - 平衡存储成本和用户体验
   - 热门媒体自动保持缓存
   - 冷门媒体可重新下载

2. **内容缓存保留期**: 建议**禁用**（设置为 0 或不设置）
   - 这个选项破坏性太强
   - 违反用户对"收藏"、"书签"的直觉预期
   - 私人提及也会丢失，影响通信历史

3. **使用 CLI 手动清理**:
   - `tootctl media remove --days=7` 安全清理
   - `tootctl media remove --days=7 --keep-interacted` 更保守，保留有交互的媒体

#### 对于特殊用途实例

如果确实需要 `content_cache_retention_period`，请确保：

1. **告知用户**: 明确说明数据不会永久保留
2. **提供导出功能**: 让用户可以导出自己的数据
3. **谨慎选择期限**: 过短的期限（如 1 天）会严重影响用户体验
4. **理解 trade-off**: 这是隐私/存储 vs 用户体验的极端选择

### 相关代码位置速查

| 功能 | 文件路径 |
|------|---------|
| API v1 控制器 | `app/controllers/api/v1/media_controller.rb` |
| API v2 控制器 | `app/controllers/api/v2/media_controller.rb` |
| 媒体模型 | `app/models/media_attachment.rb` |
| 延迟处理条件 | `app/models/media_attachment.rb:283-291` |
| Paperclip 处理拦截 | `lib/paperclip/attachment_extensions.rb:37-47` |
| 后处理 Worker | `app/workers/post_process_media_worker.rb` |
| 视频转码器 | `lib/paperclip/transcoder.rb` |
| GIF 转码器 | `lib/paperclip/gif_transcoder.rb` |
| 图片提取器 | `lib/paperclip/image_extractor.rb` |
| 存储配置 | `config/initializers/paperclip.rb` |
| 远程附件机制 | `app/models/concerns/remotable.rb` |
| 媒体代理控制器 | `app/controllers/media_proxy_controller.rb` |
| 媒体清理 | `app/lib/vacuum/media_attachments_vacuum.rb` |
| Status 清理 | `app/lib/vacuum/statuses_vacuum.rb` |
| 定期调度 | `app/workers/scheduler/vacuum_scheduler.rb` |
| 保留策略 | `app/models/content_retention_policy.rb` |
| 批量删除 | `app/lib/attachment_batch.rb` |

---

*本文档基于 Mastodon 源码深度分析，纠正了之前的多个理解偏差，特别是关于 API 行为、延迟处理机制和缓存生命周期的精确语义。*