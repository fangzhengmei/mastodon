# Mastodon 远端媒体缓存机制分析

## 一、远端实例媒体文件缓存机制

### 1.1 核心数据模型

媒体文件缓存的核心数据模型是 `MediaAttachment`，位于 `app/models/media_attachment.rb`。

#### 关键字段定义：
```ruby
# 远程媒体源 URL（为空表示本地媒体）
remote_url: string, default: ""

# Paperclip 附件字段（本地缓存文件）
file_file_name: string
file_file_size: integer
file_content_type: string
file_updated_at: datetime

# 缩略图相关字段
thumbnail_file_name: string
thumbnail_remote_url: string
```

#### 关键作用域：
```ruby
# 远程媒体（remote_url 不为空）
scope :remote, -> { where.not(remote_url: '') }

# 本地媒体（remote_url 为空）
scope :local, -> { where(remote_url: '') }

# 已缓存的远程媒体（既有 remote_url 又有本地文件）
scope :cached, -> { remote.where.not(file_file_name: nil) }

# 未关联任何状态的孤立媒体
scope :unattached, -> { where(status_id: nil, scheduled_status_id: nil) }

# 排除与本地用户有交互的媒体
scope :without_local_interaction, lambda {
  where.not(Favourite.joins(:account).merge(Account.local)...).
  where.not(Bookmark...).
  where.not(Status.local.in_reply_to...).
  where.not(Status.local.reblog_of...).
  where.not(Quote...)
}
```

### 1.2 远程文件下载机制

#### Remotable 模块
远程附件下载逻辑封装在 `app/models/concerns/remotable.rb` 中，通过 `remotable_attachment` 宏为模型添加远程下载能力。

**核心方法：**

```ruby
def remotable_attachment(attachment_name, limit, suppress_errors: true, download_on_assign: true, attribute_name: nil)
  # 定义 download_#{attachment_name}! 方法
  define_method(:"download_#{attachment_name}!") do |url = nil|
    # 1. 解析 URL 并验证协议
    parsed_url = Addressable::URI.parse(url).normalize
    return unless %w(http https).include?(parsed_url.scheme)
    
    # 2. 发起 HTTP 请求获取远程文件
    Request.new(:get, url).perform do |response|
      raise unless (200...300).cover?(response.code)
      
      # 3. 使用 ResponseWithLimit 限制下载大小
      public_send(:"#{attachment_name}=", ResponseWithLimit.new(response, limit))
    end
  end
  
  # 当设置 remote_url 属性时自动下载
  define_method(:"#{attribute_name}=") do |url|
    public_send(:"download_#{attachment_name}!", url) if download_on_assign
  end
end
```

**MediaAttachment 中的配置：**
```ruby
# 文件附件：99MB 限制，下载时不抛出错误，赋值时不自动下载
remotable_attachment :file, VIDEO_LIMIT, suppress_errors: false, download_on_assign: false, attribute_name: :remote_url

# 缩略图附件：16MB 限制，下载失败时静默处理
remotable_attachment :thumbnail, IMAGE_LIMIT, suppress_errors: true, download_on_assign: false
```

### 1.3 按需缓存触发机制

媒体文件采用**按需下载（Lazy Loading）**策略，当用户访问时才从远端实例下载。

#### MediaProxyController
位于 `app/controllers/media_proxy_controller.rb`：

```ruby
def show
  # 检查是否需要重新下载（本地文件为空但有 remote_url）
  if @media_attachment.needs_redownload? && !reject_media?
    # 使用 Redis 分布式锁防止并发下载
    with_redis_lock("media_download:#{params[:id]}") do
      @media_attachment.reload
      redownload! if @media_attachment.needs_redownload?
    end
  end
  
  # 发送文件或重定向到存储服务
  if requires_file_streaming?
    send_file(media_attachment_file.path, ...)
  else
    redirect_to media_attachment_file_path
  end
end

def redownload!
  @media_attachment.download_file!
  @media_attachment.download_thumbnail!
  @media_attachment.created_at = Time.now.utc  # 更新创建时间以延长缓存寿命
  @media_attachment.save!
end
```

**关键设计要点：**
1. **分布式锁**：使用 `with_redis_lock` 防止同一媒体被多个请求同时下载
2. **双重检查**：获取锁后再次 `reload` 检查，避免重复下载
3. **时间戳更新**：每次重新下载时更新 `created_at`，相当于"刷新"缓存寿命
4. **域名拒绝检查**：通过 `reject_media?` 检查是否被 `DomainBlock` 拒绝

### 1.4 后台下载机制

除了按需下载，还支持通过后台 Worker 异步下载：

**RedownloadMediaWorker** (`app/workers/redownload_media_worker.rb`)：
```ruby
class RedownloadMediaWorker
  include Sidekiq::Worker
  include ExponentialBackoff
  
  sidekiq_options queue: 'pull', retry: 3
  
  def perform(id)
    media_attachment = MediaAttachment.find(id)
    return if media_attachment.remote_url.blank?
    
    media_attachment.download_file!
    media_attachment.download_thumbnail!
    media_attachment.save
  end
end
```

---

## 二、清理任务的触发和执行机制

### 2.1 定时任务触发

#### Sidekiq Scheduler 配置
清理任务通过 Sidekiq Scheduler 定时触发，配置在 `config/sidekiq.yml`：

```yaml
:scheduler:
  :schedule:
    vacuum_scheduler:
      cron: '<%= Random.rand(0..59) %> <%= Random.rand(3..5) %> * * *'
      class: Scheduler::VacuumScheduler
      queue: scheduler
```

**调度特点：**
- **执行时间**：每天凌晨 3:00-5:59 之间的随机时间（避免所有实例同时执行）
- **队列**：`scheduler` 专用队列
- **锁机制**：`lock: :until_executed` 确保同一任务不会并发执行

#### VacuumScheduler
位于 `app/workers/scheduler/vacuum_scheduler.rb`：

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
      statuses_vacuum,           # 清理过期状态
      media_attachments_vacuum,  # 清理过期媒体缓存 ← 重点
      preview_cards_vacuum,      # 清理预览卡片
      backups_vacuum,            # 清理过期备份
      access_tokens_vacuum,      # 清理过期令牌
      feeds_vacuum,              # 清理过期订阅源
      imports_vacuum,            # 清理过期导入
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

### 2.2 清理执行流程

#### MediaAttachmentsVacuum
核心清理逻辑位于 `app/lib/vacuum/media_attachments_vacuum.rb`：

```ruby
class Vacuum::MediaAttachmentsVacuum
  TTL = 1.day.freeze  # 孤立记录的生存时间
  
  def initialize(retention_period)
    @retention_period = retention_period
  end
  
  def perform
    vacuum_orphaned_records!  # 清理孤立记录
    vacuum_cached_files! if retention_period?  # 清理过期缓存（如果配置了保留期）
  end
  
  private
  
  def vacuum_cached_files!
    # 批量处理超过保留期的远程缓存媒体
    media_attachments_past_retention_period.find_in_batches do |media_attachments|
      AttachmentBatch.new(MediaAttachment, media_attachments).clear
    rescue => e
      Rails.logger.error("Skipping batch while removing cached media attachments due to error: #{e}")
    end
  end
  
  def vacuum_orphaned_records!
    # 清理创建时间超过 1 天的未关联媒体
    orphaned_media_attachments.find_in_batches do |media_attachments|
      AttachmentBatch.new(MediaAttachment, media_attachments).delete
    end
  end
  
  # 超过保留期的缓存媒体查询条件
  def media_attachments_past_retention_period
    MediaAttachment
      .remote           # 远程媒体
      .cached           # 已缓存到本地
      .created_before(@retention_period.ago)  # 创建时间早于保留期
      .updated_before(@retention_period.ago)  # 更新时间早于保留期
  end
  
  # 孤立记录查询条件
  def orphaned_media_attachments
    MediaAttachment
      .unattached       # 未关联任何状态
      .created_before(TTL.ago)  # 创建时间超过 1 天
  end
end
```

**清理策略区分：**
| 方法 | 目标 | 操作 |
|------|------|------|
| `vacuum_cached_files!` | 超过保留期的远程缓存媒体 | `clear` - 删除文件，保留数据库记录 |
| `vacuum_orphaned_records!` | 孤立的未关联媒体 | `delete` - 删除文件和数据库记录 |

### 2.3 批量删除实现

#### AttachmentBatch
批量删除逻辑位于 `app/lib/attachment_batch.rb`，支持多种存储后端：

```ruby
class AttachmentBatch
  LIMIT = ENV.fetch('S3_BATCH_DELETE_LIMIT', 1000).to_i  # S3 批量删除限制
  MAX_RETRY = ENV.fetch('S3_BATCH_DELETE_RETRY', 3).to_i  # 重试次数
  
  # Paperclip 自动管理的可空字段
  NULLABLE_ATTRIBUTES = %w(file_name content_type file_size fingerprint created_at updated_at).freeze
  
  # 彻底删除：文件 + 数据库记录
  def delete
    remove_files
    batch.delete_all
  end
  
  # 仅清除缓存：删除文件，保留记录（清空 Paperclip 字段）
  def clear
    remove_files
    batch.update_all(nullified_attributes)  # 将 file_file_name 等字段设为 NULL
  end
  
  private
  
  def remove_files
    records.each do |record|
      @attachment_names.each do |attachment_name|
        attachment = record.public_send(attachment_name)
        styles = BASE_STYLES | attachment.styles.keys  # original + 自定义样式
        
        styles.each do |style|
          case @storage_mode
          when :filesystem
            # 本地文件系统：直接删除文件和空目录
            FileUtils.remove_file(path, true)
            FileUtils.rmdir(File.dirname(path), parents: true)
          when :s3
            # S3：收集 key 后批量删除
            keys << attachment.style_name_as_path(style)
          when :fog, :azure
            # 其他云存储：逐个删除
            attachment.destroy
          end
        end
      end
    end
    
    # S3 批量删除优化
    if storage_mode == :s3
      keys.each_slice(LIMIT) do |keys_slice|
        bucket.delete_objects(delete: { objects: keys_slice.map { |k| { key: k } }, quiet: true })
      end
    end
  end
  
  # 构建需要清空的字段哈希
  def nullified_attributes
    @attachment_names.flat_map { |name| 
      NULLABLE_ATTRIBUTES.map { |attr| "#{name}_#{attr}" } & klass.column_names 
    }.index_with(nil)
    # => { "file_file_name" => nil, "file_file_size" => nil, ... }
  end
end
```

### 2.4 手动清理命令

除了定时任务，还可以通过 CLI 手动触发清理：

#### tootctl media remove
位于 `lib/mastodon/cli/media.rb`：

```ruby
option :days, type: :numeric, default: 7, aliases: [:d]
option :prune_profiles, type: :boolean, default: false
option :remove_headers, type: :boolean, default: false
option :include_follows, type: :boolean, default: false
option :keep_interacted, type: :boolean, default: false
option :dry_run, type: :boolean, default: false

def remove
  time_ago = options[:days].days.ago
  
  # 清理头像/头图
  if options[:prune_profiles] || options[:remove_headers]
    # 处理远程账户的头像和头图...
  end
  
  # 清理媒体附件
  unless options[:prune_profiles] || options[:remove_headers]
    attachment_scope = MediaAttachment.cached.remote.where(created_at: ..time_ago)
    
    # 可选：保留与本地用户有交互的媒体
    attachment_scope = attachment_scope.without_local_interaction if options[:keep_interacted]
    
    parallelize_with_progress(attachment_scope) do |media_attachment|
      # 逐个删除文件
      media_attachment.file.destroy
      media_attachment.thumbnail.destroy
      media_attachment.save
    end
  end
end
```

**命令选项说明：**

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--days N` / `-d N` | 删除 N 天前的媒体 | 7 |
| `--prune-profiles` | 仅清理头像和头图 | false |
| `--remove-headers` | 仅清理头图 | false |
| `--include-follows` | 同时清理有关注关系的账户媒体 | false |
| `--keep-interacted` | 保留与本地用户有交互的媒体 | false |
| `--dry-run` | 试运行，不实际删除 | false |

**交互保留逻辑 (`without_local_interaction`)：**
```ruby
scope :without_local_interaction, lambda {
  where.not(Favourite.joins(:account).merge(Account.local)...)  # 未被收藏
    .where.not(Bookmark...)                                        # 未被书签
    .where.not(Status.local.where(in_reply_to_id: ...)...)        # 未被回复
    .where.not(Status.local.where(reblog_of_id: ...)...)          # 未被转发
    .where.not(Quote...)                                            # 未被引用
}
```

---

## 三、存储配额限制机制

### 3.1 核心结论：无磁盘容量硬限额

**重要说明**：Mastodon **没有内置的按磁盘容量的硬限额机制**。

经过全面代码分析，确认以下事实：

| 检查项 | 结果 | 说明 |
|--------|------|------|
| 按 GB/MB 限制 | ❌ 不存在 | 没有任何配置项或代码逻辑基于磁盘容量阈值触发清理 |
| 存储百分比限制 | ❌ 不存在 | 没有检查磁盘使用率（如 80%、90%）的逻辑 |
| `quota` 关键词 | ⚠️ 仅翻译服务 | 所有 `QuotaExceededError` 相关代码均为 DeepL/LibreTranslate 翻译 API 配额，与媒体存储无关 |
| `disk.*limit` | ❌ 不存在 | 没有磁盘容量限制相关的配置或代码 |

**唯一的配额机制**：基于**时间保留期**（天数）的软限制，而非磁盘容量的硬限制。

---

### 3.2 时间保留期机制（唯一的配额方式）

Mastodon 仅提供基于**保留天数**的媒体清理机制，通过 `media_cache_retention_period` 设置控制。

#### 默认设置
位于 `config/settings.yml`：
```yaml
defaults: &defaults
  backups_retention_period: 7  # 备份默认保留 7 天
  # 注意：media_cache_retention_period 没有默认值！
  # 不设置或设为 0/负数 = 不限制，不清理
```

#### 设置模型
位于 `app/models/form/admin_settings.rb`：
```ruby
# 可配置的设置键
KEYS = %i(
  media_cache_retention_period    # 媒体缓存保留期（天数）
  content_cache_retention_period  # 内容缓存保留期（危险区域）
  backups_retention_period         # 备份保留期
  min_age
).freeze

# 整数类型设置
INTEGER_KEYS = %i(
  media_cache_retention_period
  content_cache_retention_period
  backups_retention_period
  min_age
).freeze
```

#### 保留期策略
位于 `app/models/content_retention_policy.rb`：
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
  
  # 关键：仅当值为正整数时才返回时间间隔
  def retention_period(value)
    value.days if value.is_a?(Integer) && value.positive?
  end
end
```

**保留期生效条件：**
| 设置值 | 行为 |
|--------|------|
| 正整数（如 `7`） | 生效，清理超过 N 天的媒体 |
| `0` | 返回 `nil`，**不清理** |
| 负数 | 返回 `nil`，**不清理** |
| `nil`（未设置） | 返回 `nil`，**不清理** |
| 空字符串 | 返回 `nil`，**不清理** |

---

### 3.3 定时清理的触发条件与边界

#### 触发条件
定时清理由 `MediaAttachmentsVacuum` 执行，触发条件如下：

```ruby
# app/lib/vacuum/media_attachments_vacuum.rb

# 执行入口
def perform
  vacuum_orphaned_records!  # 无条件执行：清理孤立记录
  vacuum_cached_files! if retention_period?  # 仅当保留期设置为正整数时执行
end

# 保留期检查
def retention_period?
  @retention_period.present?  # 即：Setting.media_cache_retention_period 为正整数
end
```

#### 清理范围边界（定时任务）

**过期缓存媒体查询条件：**
```ruby
def media_attachments_past_retention_period
  MediaAttachment
    .remote           # 条件 1: 远程媒体（remote_url 不为空）
    .cached           # 条件 2: 已缓存到本地（file_file_name 不为空）
    .created_before(@retention_period.ago)  # 条件 3: created_at < N 天前
    .updated_before(@retention_period.ago)  # 条件 4: updated_at < N 天前
end
```

**边界条件详解：**

| 条件 | 说明 | 边界行为 |
|------|------|----------|
| `.remote` | 仅清理**远程**媒体 | **本地媒体永远不会被清理**（本地用户上传的媒体） |
| `.cached` | 仅清理**已缓存**的远程媒体 | 有 `remote_url` 但从未被下载的记录不参与清理 |
| `.created_before(N.days.ago)` | 创建时间早于 N 天前 | 精确的时间边界：`created_at < N.days.ago` |
| `.updated_before(N.days.ago)` | 更新时间早于 N 天前 | **双重保险**，防止意外清理 |

**孤立记录清理（无条件执行）：**
```ruby
def orphaned_media_attachments
  MediaAttachment
    .unattached       # 未关联任何状态（status_id 为空）
    .created_before(TTL.ago)  # TTL = 1.day
end
```
- **触发条件**：无配置依赖，每次 VacuumScheduler 执行都会运行
- **TTL**：硬编码为 1 天（`TTL = 1.day.freeze`）
- **操作类型**：`delete`（删除文件 + 删除数据库记录）

---

### 3.4 手动清理的触发条件与边界

#### 触发条件
手动清理通过 `tootctl media remove` 命令触发，完全独立于定时任务的配置。

```bash
# 基本用法
tootctl media remove --days 7

# 带选项
tootctl media remove --days 30 --keep-interacted --dry-run
```

#### 清理范围边界（手动命令）

**媒体附件查询条件：**
```ruby
# lib/mastodon/cli/media.rb

def remove
  time_ago = options[:days].days.ago  # 默认 7 天
  
  # ...
  
  # 媒体附件清理范围
  unless options[:prune_profiles] || options[:remove_headers]
    attachment_scope = MediaAttachment.cached.remote.where(created_at: ..time_ago)
    #                            ↑        ↑              ↑
    #                         已缓存   远程媒体      仅 created_at 检查
    
    # 可选：排除与本地用户有交互的媒体
    attachment_scope = attachment_scope.without_local_interaction if options[:keep_interacted]
    
    # 逐个处理
    parallelize_with_progress(attachment_scope) do |media_attachment|
      # ...
    end
  end
end
```

**边界条件详解：**

| 条件 | 定时清理 | 手动清理 | 差异说明 |
|------|----------|----------|----------|
| 远程媒体检查 | `.remote` | `.remote` | 相同 |
| 已缓存检查 | `.cached` | `.cached` | 相同 |
| 时间检查 | `created_before` **AND** `updated_before` | `where(created_at: ..time_ago)` | **关键差异**：手动清理**不检查 `updated_at`** |
| 交互媒体 | 一律清理 | 可选 `--keep-interacted` 保留 | 手动清理支持精细化控制 |

**`--keep-interacted` 的排除逻辑：**
```ruby
# app/models/media_attachment.rb

scope :without_local_interaction, lambda {
  # 以下任意情况为真，则**排除**该媒体（即保留）
  where.not(Favourite.joins(:account).merge(Account.local)...)  # 被本地用户收藏
    .where.not(Bookmark...)                                        # 被本地用户书签
    .where.not(Status.local.where(in_reply_to_id: ...)...)        # 被本地用户回复
    .where.not(Status.local.where(reblog_of_id: ...)...)          # 被本地用户转发
    .where.not(Quote...)                                            # 被本地用户引用
}
```

---

### 3.5 定时清理 vs 手动清理 完整对比

| 对比维度 | 定时清理 (`MediaAttachmentsVacuum`) | 手动清理 (`tootctl media remove`) |
|----------|--------------------------------------|------------------------------------|
| **触发方式** | Sidekiq Scheduler 定时执行 | 管理员手动执行命令 |
| **调度时间** | 每天凌晨 3:00-5:59 随机时间 | 即时执行 |
| **配置依赖** | 依赖 `Setting.media_cache_retention_period` 为正整数 | 不依赖系统设置，使用 `--days` 参数 |
| **时间条件** | `created_at < N.days.ago` **AND** `updated_at < N.days.ago` | 仅 `created_at < N.days.ago` |
| **双重检查** | ✅ 有（created_at + updated_at） | ❌ 无（仅 created_at） |
| **交互媒体保留** | ❌ 不支持，一律清理 | ✅ 支持 `--keep-interacted` 选项 |
| **操作方式** | 批量 `update_all`（高效） | 逐个 `destroy` + `save`（较慢） |
| **并行处理** | ❌ 无 | ✅ 支持 `--concurrency N`（默认 5） |
| **试运行** | ❌ 不支持 | ✅ 支持 `--dry-run` |
| **并发控制** | ✅ `lock: :until_executed` 防止并发 | ❌ 无内置并发控制 |
| **孤立记录清理** | ✅ 每次执行都会清理 1 天前的孤立记录 | ❌ 不清理孤立记录（需单独命令） |

---

---

## 3.9 缓存失效边界：最终统一口径

### 3.9.1 重下载后的时间戳变化（最终确认）

**重下载触发场景**：
- 用户访问过期媒体（`needs_redownload?` = true，即 `file.blank? && remote_url.present?`）
- 或通过 `RedownloadMediaWorker` 后台任务

**重下载核心代码**（MediaProxyController#redownload!）：
```ruby
# app/controllers/media_proxy_controller.rb

def redownload!
  @media_attachment.download_file!
  @media_attachment.download_thumbnail!
  @media_attachment.created_at = Time.now.utc  # 显式设置 created_at
  @media_attachment.save!                        # Rails 自动更新 updated_at
end
```

**时间戳变化分析**：

| 字段 | 变化方式 | 重下载后的值 | 代码依据 |
|------|----------|-------------|----------|
| `created_at` | **显式设置** | 当前时间 `Time.now.utc` | 第 650 行：`created_at = Time.now.utc` |
| `updated_at` | **Rails 自动更新** | 当前时间 | Rails `save!` 触发 `timestamps` 自动更新 |

**最终结论**：重下载后，`created_at` 和 `updated_at` **都被刷新为当前时间**。

---

### 3.9.2 时间比较方式的精确分析

#### 定时清理的时间比较

**定时清理的 Scope 定义**（app/models/media_attachment.rb）：
```ruby
scope :created_before, ->(value) { where(arel_table[:created_at].lt(value)) }  # created_at < value
scope :updated_before, ->(value) { where(arel_table[:updated_at].lt(value)) }  # updated_at < value
```

**定时清理的查询组合**（app/lib/vacuum/media_attachments_vacuum.rb）：
```ruby
def media_attachments_past_retention_period
  MediaAttachment
    .remote
    .cached
    .created_before(@retention_period.ago)  # created_at < N.days.ago
    .updated_before(@retention_period.ago)  # updated_at < N.days.ago
end
```

**定时清理时间比较特性**：
- 使用 `.lt(value)` = **`<`**（**不包含边界**）
- 两个条件是 **AND** 关系，必须同时满足
- 重下载后，两个时间戳都被刷新，所以都不会满足 `< N.days.ago`

#### 手动清理的时间比较

**手动清理的查询**（lib/mastodon/cli/media.rb）：
```ruby
attachment_scope = MediaAttachment.cached.remote.where(created_at: ..time_ago)
```

**手动清理时间比较特性**：
- Ruby Range `..time_ago` 表示 `created_at <= time_ago`（**包含边界**）
- **不检查** `updated_at`
- 重下载后，`created_at` 被刷新，所以不会满足 `<= time_ago`（如果 time_ago 在刷新时间之前）

---

### 3.9.3 精确时间边界对比

**假设条件**：
- 保留期 N = 7 天
- 媒体在时间 **T** 被重下载

| 时间点 | created_at | updated_at | 定时清理条件判定 | 手动清理条件判定 |
|--------|------------|------------|------------------|------------------|
| 重下载时（T） | T | T | ❌ 否（`T < T-7d`？假） | ❌ 否（`T <= T-7d`？假） |
| T+6 天 23:59:59 | T | T | ❌ 否（`T < (T+6d23h59m59s)-7d = T-1s`？假） | ❌ 否（`T <= T-1s`？假） |
| T+7 天 00:00:00 | T | T | ❌ 否（`T < (T+7d)-7d = T`？假，`<` 不包含边界） | ✅ 是（`T <= T`？真，`<=` 包含边界） |
| T+7 天 00:00:01 | T | T | ✅ 是（`T < (T+7d1s)-7d = T+1s`？真） | ✅ 是（`T <= T+1s`？真） |

**关键发现**：
- **定时清理**：在 T+7 天整**不会**命中（因为 `< T` 不包含边界），在 T+7 天 1 秒**才会**命中
- **手动清理**：在 T+7 天整**就会**命中（因为 `<= T` 包含边界）

---

### 3.9.4 对两种清理方式的实际影响

#### 重下载对定时清理的影响

| 重下载前状态 | 重下载后状态 | 定时清理命中情况 |
|--------------|--------------|------------------|
| `created_at` = T-10d, `updated_at` = T-10d | `created_at` = T, `updated_at` = T | ❌ 续命成功，**不会**被清理 |
| `created_at` = T-10d, `updated_at` = T-8d | `created_at` = T, `updated_at` = T | ❌ 续命成功，**不会**被清理 |
| `created_at` = T-3d, `updated_at` = T-3d | `created_at` = T, `updated_at` = T | ❌ 续命成功，**不会**被清理 |

**结论**：重下载后，定时清理的**两个时间条件都被重置**，媒体获得完整的 N 天续命。

#### 重下载对手动清理的影响

| 重下载前状态 | 重下载后状态 | 手动清理命中情况（`--days 7`） |
|--------------|--------------|-------------------------------|
| `created_at` = T-10d | `created_at` = T | ❌ 续命成功，**不会**被清理 |
| `created_at` = T-3d | `created_at` = T | ❌ 续命成功，**不会**被清理 |

**结论**：重下载后，`created_at` 被重置，手动清理同样**续命成功**。

#### `--keep-interacted` 的特殊影响

手动清理的 `--keep-interacted` 选项：
```ruby
# app/models/media_attachment.rb

scope :without_local_interaction, lambda {
  where.not(Favourite.joins(:account).merge(Account.local)...)  # 未被本地用户收藏
    .where.not(Bookmark...)                                        # 未被本地用户书签
    .where.not(Status.local.where(in_reply_to_id: ...)...)        # 未被本地用户回复
    .where.not(Status.local.where(reblog_of_id: ...)...)          # 未被本地用户转发
    .where.not(Quote...)                                            # 未被本地用户引用
}
```

**影响**：
- 这是一个**额外的过滤条件**，与时间戳无关
- 即使时间戳满足清理条件，如果媒体与本地用户有交互，也会**被保留**
- **定时清理不支持**这个逻辑，定时清理一律清理过期媒体

---

### 3.9.5 最终结论汇总表

| 问题 | 最终答案 | 代码依据 |
|------|----------|----------|
| 重下载后 `created_at` 变化？ | ✅ 显式设置为当前时间 `Time.now.utc` | `app/controllers/media_proxy_controller.rb:650` |
| 重下载后 `updated_at` 变化？ | ✅ Rails `save!` 自动更新为当前时间 | Rails `timestamps` 标准行为 |
| 定时清理检查哪些字段？ | `created_at` **AND** `updated_at`（AND 关系） | `app/lib/vacuum/media_attachments_vacuum.rb:37-38` |
| 手动清理检查哪些字段？ | 仅 `created_at` | `lib/mastodon/cli/media.rb:682` |
| 定时清理时间比较方式？ | `.lt(value)` = **`<`**（不包含边界） | `app/models/media_attachment.rb:214,219` |
| 手动清理时间比较方式？ | `..time_ago` = **`<=`**（包含边界） | Ruby Range 语法 |
| 重下载后定时清理是否续命？ | ✅ 是，两个字段都被重置 | 两个时间戳都刷新 |
| 重下载后手动清理是否续命？ | ✅ 是，`created_at` 被重置 | `created_at` 刷新 |
| 哪个清理更早命中？ | 手动清理（N 天整就命中，`<=` 包含边界） | 时间比较方式差异 |
| 哪个清理更晚命中？ | 定时清理（N 天整 + 1 秒才命中，`<` 不包含边界） | 时间比较方式差异 |
| `--keep-interacted` 哪个支持？ | 仅手动清理，定时清理不支持 | `without_local_interaction` scope 仅在 CLI 中使用 |

---

### 3.9.6 关键代码位置索引

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| 重下载时间戳更新 | `app/controllers/media_proxy_controller.rb` | 第 647-652 行 |
| 定时清理 created_before scope | `app/models/media_attachment.rb` | 第 214 行 |
| 定时清理 updated_before scope | `app/models/media_attachment.rb` | 第 219 行 |
| 定时清理查询组合 | `app/lib/vacuum/media_attachments_vacuum.rb` | 第 33-39 行 |
| 手动清理查询 | `lib/mastodon/cli/media.rb` | 第 682 行 |
| 交互保留 scope | `app/models/media_attachment.rb` | 第 220-227 行 |

---

## 3.10 "访问即续命"机制的边界影响（已整合到 3.9）

*本小节内容已整合到上方"3.9 缓存失效边界：最终统一口径"中，以确保结论的一致性。*

---

### 3.7 管理界面配置

#### 内容保留设置页面
位于 `app/views/admin/settings/content_retention/show.html.haml`：

```haml
= simple_form_for @admin_settings, url: admin_settings_content_retention_path do |f|
  .fields-group
    = f.input :media_cache_retention_period,
              input_html: { pattern: '[0-9]+' },
              wrapper: :with_block_label
    = f.input :backups_retention_period,
              input_html: { pattern: '[0-9]+' },
              wrapper: :with_block_label

  %h2= t('admin.settings.content_retention.danger_zone')
  
  .fields-group
    = f.input :content_cache_retention_period,
              hint: false,
              input_html: { pattern: '[0-9]+' },
              warning_hint: t('simple_form.hints.form_admin_settings.content_cache_retention_period'),
              wrapper: :with_block_label
```

**设置分级：**
1. **常规设置**：`media_cache_retention_period`、`backups_retention_period`
2. **危险区域**：`content_cache_retention_period`（会删除旧的远程状态本身，不仅仅是媒体）

---

### 3.8 存储使用统计

#### CLI 查看使用情况
`tootctl media usage` 命令仅提供**统计**功能，不做任何限制：

```ruby
def usage
  print_table [
    %w(Object Total Local),
    *object_storage_summary,
  ]
end

def object_storage_summary
  [
    [:attachments, MediaAttachment.sum(combined_media_sum), MediaAttachment.where(account: Account.local).sum(combined_media_sum)],
    [:custom_emoji, CustomEmoji.sum(:image_file_size), CustomEmoji.local.sum(:image_file_size)],
    [:avatars, Account.sum(:avatar_file_size), Account.local.sum(:avatar_file_size)],
    [:headers, Account.sum(:header_file_size), Account.local.sum(:header_file_size)],
    [:preview_cards, PreviewCard.sum(:image_file_size), nil],
    [:backups, Backup.sum(:dump_file_size), nil],
  ]
end

def combined_media_sum
  MediaAttachment.combined_media_file_size  # file_size + thumbnail_size
end
```

**统计表格式：**
| Object | Total | Local |
|--------|-------|-------|
| Attachments | 总媒体大小 | 本地账户媒体大小 |
| Custom Emoji | 总表情大小 | 本地上传表情大小 |
| Avatars | 所有头像大小 | 本地账户头像大小 |
| Headers | 所有头图大小 | 本地账户头图大小 |
| Preview Cards | 预览卡片大小 | - |
| Backups | 备份大小 | - |

---

## 四、完整流程图

### 4.1 媒体缓存生命周期

```
┌─────────────────────────────────────────────────────────────────┐
│                      远程媒体缓存生命周期                          │
└─────────────────────────────────────────────────────────────────┘

  远程实例发布状态
       │
       ▼
┌──────────────┐
│ ActivityPub  │←────── 通过联合协议接收远程状态
│  接收状态    │        包含 media_attachments（仅 URL）
└──────────────┘
       │
       ▼
┌──────────────┐
│  创建        │←────── MediaAttachment.create(
│ MediaAttach- │           remote_url: "https://remote-instance/...",
│   ment       │           status_id: ...,
│              │           file_file_name: nil) ← 本地文件为空
└──────────────┘
       │
       │ 用户访问该媒体
       ▼
┌──────────────┐
│ MediaProxy   │←────── 检查 needs_redownload?
│ Controller   │        (file.blank? && remote_url.present?)
└──────────────┘
       │
       │ 需要下载
       ▼
┌──────────────┐
│  with_redis  │←────── 获取分布式锁防止并发下载
│    _lock     │
└──────────────┘
       │
       ▼
┌──────────────┐
│ Remotable    │←────── Request.new(:get, remote_url).perform
│ download_    │        ResponseWithLimit 限制大小
│   file!      │
└──────────────┘
       │
       ▼
┌──────────────┐
│  更新记录    │←────── file_file_name = "xxx.jpg"
│              │        created_at = Time.now  ← 刷新时间戳
└──────────────┘
       │
       │ 时间流逝...
       │
       ▼
┌──────────────┐
│ 定时任务触发  │←────── Sidekiq Scheduler 每天 3-5 点
│              │
│ Vacuum-      │
│ Scheduler    │
└──────────────┘
       │
       ▼
┌──────────────┐
│ MediaAttach- │←────── 查询条件：
│ mentsVacuum  │        remote + cached +
│              │        created_before(N.days.ago) +
│              │        updated_before(N.days.ago)
└──────────────┘
       │
       │ 超过保留期？
       │
       ├─ 是 ──▶ AttachmentBatch.clear ──▶ 删除文件，保留记录
       │
       └─ 否 ──▶ 保留缓存

  用户再次访问过期媒体
       │
       ▼
  ┌─────────┐
  │重新下载 │←────── 回到 MediaProxyController 流程
  │并续命  │        created_at 再次更新
  └─────────┘
```

### 4.2 清理任务触发链

```
┌────────────────────────────────────────────────────────────────────┐
│                         清理任务触发链                               │
└────────────────────────────────────────────────────────────────────┘

  config/sidekiq.yml
       │
       │ cron: '随机分钟 3-5点 * * *'
       ▼
┌─────────────────┐
│ Sidekiq Scheduler │
│ (sidekiq-cron)  │
└─────────────────┘
       │
       ▼
┌─────────────────┐     ┌─────────────────────────────┐
│ VacuumScheduler │────▶│ ContentRetentionPolicy      │
│                 │     │ .current                    │
│ #perform        │     │                             │
└─────────────────┘     │ .media_cache_retention_    │
       │                │   period = Setting.xxx     │
       │                └─────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│ MediaAttachmentsVacuum.new(retention_period)            │
│                                                           │
│ #perform                                                  │
│   ├── vacuum_orphaned_records!                           │
│   │       └── 清理 unattached + created_before(1.day.ago)│
│   │            └── AttachmentBatch.delete                │
│   │                 ├── 删除文件                          │
│   │                 └── 删除数据库记录                     │
│   │                                                        │
│   └── vacuum_cached_files! (if retention_period.present?)│
│           └── 清理 remote + cached +                      │
│               created_before(N.days.ago) +               │
│               updated_before(N.days.ago)                 │
│                └── AttachmentBatch.clear                  │
│                     ├── 删除文件                           │
│                     └── 更新记录（字段设为 NULL）           │
└─────────────────────────────────────────────────────────┘
```

---

## 五、关键设计总结

### 5.1 缓存策略特点

| 特性 | 实现方式 | 代码位置 |
|------|----------|----------|
| **按需下载** | MediaProxyController 检测 needs_redownload? 时触发 | `app/controllers/media_proxy_controller.rb:18-23` |
| **并发保护** | Redis 分布式锁 `with_redis_lock("media_download:#{id}")` | `app/controllers/media_proxy_controller.rb:19` |
| **访问续命** | 重新下载时更新 `created_at = Time.now.utc` | `app/controllers/media_proxy_controller.rb:42` |
| **双重时间检查** | `created_before` + `updated_before` | `app/lib/vacuum/media_attachments_vacuum.rb:34-39` |

### 5.2 清理策略特点

| 场景 | 触发方式 | 清理范围 | 操作类型 |
|------|----------|----------|----------|
| **日常维护** | Sidekiq Scheduler（每天 3-5 点） | 超过保留期的远程缓存 | `clear`（删文件保记录） |
| **孤立记录** | 同上 | 未关联状态超过 1 天 | `delete`（删文件删记录） |
| **手动清理** | `tootctl media remove` | 指定天数前的缓存 | 可配置保留交互媒体 |
| **特定域名** | `ClearDomainMediaService` | 特定域名的所有媒体 | 彻底清理 |

### 5.3 存储配额机制

```
┌────────────────────────────────────────────────────────────┐
│                    存储配额配置流程                          │
└────────────────────────────────────────────────────────────┘

  管理员设置
       │
       ▼
┌──────────────────┐
│  /admin/settings │
│  /content-       │←─── 表单输入 media_cache_retention_period
│  retention       │     (整数，表示天数)
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ Form::Admin-     │
│ Settings         │←─── 类型转换：空字符串 → nil
│ #save            │     正整数 → 存储到 settings 表
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ ContentRetention │
│ Policy           │←─── 读取 Setting.media_cache_retention_period
│                  │
│ #retention_      │     if value.is_a?(Integer) && value.positive?
│ period(value)    │       value.days  ← 返回时间间隔
│                  │     else
│                  │       nil         ← 返回 nil（不限制）
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ MediaAttachments │
│ Vacuum           │
│                  │←─── 检查 retention_period?
│ #perform         │
│                  │     if @retention_period.present?
│                  │       vacuum_cached_files!  ← 执行清理
│                  │     else
│                  │       skip                    ← 跳过清理
└──────────────────┘
```

### 5.4 关键代码位置索引

| 功能 | 文件路径 | 关键方法/行号 |
|------|----------|---------------|
| 媒体模型定义 | `app/models/media_attachment.rb` | 第 213-227 行（scopes） |
| 远程下载机制 | `app/models/concerns/remotable.rb` | 第 10-49 行 |
| 按需缓存触发 | `app/controllers/media_proxy_controller.rb` | 第 17-44 行 |
| 后台下载 Worker | `app/workers/redownload_media_worker.rb` | 第 10-24 行 |
| 定时清理调度 | `config/sidekiq.yml` | 第 31-34 行 |
| 清理调度器 | `app/workers/scheduler/vacuum_scheduler.rb` | 第 34-36 行 |
| 媒体清理逻辑 | `app/lib/vacuum/media_attachments_vacuum.rb` | 第 10-49 行 |
| 批量删除 | `app/lib/attachment_batch.rb` | 第 33-41 行（clear/delete） |
| 保留期策略 | `app/models/content_retention_policy.rb` | 第 8-24 行 |
| 设置表单 | `app/models/form/admin_settings.rb` | 第 34-36 行（KEYS） |
| CLI 命令 | `lib/mastodon/cli/media.rb` | 第 40-88 行（remove 命令） |

---

## 六、配置建议

### 6.1 典型配置场景

#### 场景 1：存储空间有限（推荐配置）
```yaml
# 仅保留最近 7 天的远程媒体缓存
media_cache_retention_period: 7
backups_retention_period: 7

# 不配置 content_cache_retention_period（保留所有远程状态）
```
**行为说明**：
- 定时任务每天凌晨 3-5 点清理超过 7 天的远程媒体缓存
- 热门媒体因"访问即续命"可能永远保留
- 本地媒体不受影响

#### 场景 2：非常有限的存储空间
```yaml
# 保留最近 3 天
media_cache_retention_period: 3
backups_retention_period: 3
```
**配合手动清理**：
```bash
# 每周手动清理一次，保留交互媒体
tootctl media remove --days 7 --keep-interacted
```

#### 场景 3：无限存储（不清理远程媒体）
```yaml
# 不配置或设为 0/负数
media_cache_retention_period: null  # 或 0
```
**行为说明**：
- `ContentRetentionPolicy#retention_period` 返回 `nil`
- `MediaAttachmentsVacuum#retention_period?` 返回 `false`
- `vacuum_cached_files!` 永远不会执行
- **仅孤立记录会被清理**（1 天前未关联状态的媒体）

#### 场景 4：混合策略（定时 + 手动）
```yaml
# 定时任务保留 30 天
media_cache_retention_period: 30
```
```bash
# 但每周手动清理超过 7 天且无交互的媒体
tootctl media remove --days 7 --keep-interacted --dry-run  # 先试运行
tootctl media remove --days 7 --keep-interacted            # 实际执行
```

---

### 6.2 监控与运维建议

#### 1. 存储使用监控
```bash
# 查看当前存储使用情况
tootctl media usage
```

**典型输出**：
```
Object          Total   Local
------------------------------
Attachments     2.5 GB  500 MB
Custom Emoji    10 MB   2 MB
Avatars         50 MB   10 MB
Headers         100 MB  20 MB
Preview Cards   200 MB  (not applicable)
Backups         1 GB    (not applicable)
```

#### 2. 试运行清理（推荐在实际清理前执行）
```bash
# 查看哪些媒体会被清理
tootctl media remove --days 7 --dry-run

# 带交互保留的试运行
tootctl media remove --days 30 --keep-interacted --dry-run
```

#### 3. 强制清理热门媒体
由于"访问即续命"机制，热门媒体可能永远不会被定时任务清理。如需强制清理：

```bash
# 方式 1：手动清理（不保留交互媒体）
tootctl media remove --days 7

# 方式 2：清理特定域名的媒体（如果某个远程实例占用过多空间）
# 注意：需要使用 ClearDomainMediaService 或手动 SQL
```

---

### 6.3 边界情况与注意事项

#### 1. `content_cache_retention_period` 是危险设置
- **作用**：会删除超过保留期的**远程状态本身**（不仅仅是媒体）
- **后果**：时间线中出现"此状态已删除"，无法恢复
- **建议**：除非极端存储压力，否则**不要设置**

#### 2. 本地媒体永远不会被清理
- 定时清理条件：`.remote`（remote_url 不为空）
- 手动清理条件：`.remote`
- **结论**：本地用户上传的媒体不受任何保留期限制

#### 3. 孤立记录的特殊处理
孤立记录（`unattached`，即 `status_id` 为空的媒体）：
- **触发条件**：无配置依赖，每次 VacuumScheduler 都会执行
- **TTL**：硬编码为 1 天（`TTL = 1.day.freeze`）
- **操作**：`delete`（删除文件 + 删除数据库记录）
- **无法配置**：这是硬编码的清理逻辑，无设置项

#### 4. 清理是不可逆的
- 删除的媒体文件无法恢复
- 依赖远程实例重新提供（如果远程实例仍可用且媒体未被删除）
- 建议：重要媒体应考虑备份或使用 `--keep-interacted`

#### 5. S3 批量删除配置
- 默认每批 1000 个对象
- 可通过环境变量调整：
  ```bash
  S3_BATCH_DELETE_LIMIT=500    # 每批 500 个
  S3_BATCH_DELETE_RETRY=5      # 重试 5 次
  ```

---

### 6.4 故障排查

#### 问题 1：设置了保留期但媒体未被清理
**可能原因**：
1. **保留期设置无效**：设为 0、负数或空字符串
2. **媒体被"续命"**：热门媒体被反复访问，`created_at` 持续更新
3. **媒体是本地的**：`remote_url` 为空，永远不会被清理
4. **媒体未被缓存**：有 `remote_url` 但从未被下载

**排查步骤**：
```ruby
# 检查设置值
Setting.media_cache_retention_period

# 检查哪些媒体会被清理（在 Rails console 中执行）
retention = 7.days
MediaAttachment
  .remote
  .cached
  .created_before(retention.ago)
  .updated_before(retention.ago)
  .count
```

#### 问题 2：手动清理与定时清理行为不一致
**原因**：两种清理的查询条件不同
- 定时：`created_before` + `updated_before`（双重检查）
- 手动：仅 `created_at` 检查

**如果需要一致的行为**：
- 建议优先使用定时任务，手动清理仅作为补充
- 或使用 `--keep-interacted` 获得更细粒度的控制

#### 问题 3：磁盘空间持续增长
**可能原因**：
1. **热门媒体过多**：被反复访问，永远不会被清理
2. **保留期设置过长**：如 `365` 天
3. **孤立记录清理失败**：检查 Sidekiq 日志

**解决建议**：
```bash
# 手动清理超过 7 天的所有媒体（包括有交互的）
tootctl media remove --days 7

# 或者更保守地：清理超过 30 天但保留有交互的
tootctl media remove --days 30 --keep-interacted
```

---

### 6.5 关键配置速查表

| 配置项 | 类型 | 默认值 | 有效值 | 说明 |
|--------|------|--------|--------|------|
| `media_cache_retention_period` | 整数 | 无（不配置） | 正整数 = 生效；0/负数/空 = 不清理 | 远程媒体缓存保留天数 |
| `content_cache_retention_period` | 整数 | 无（不配置） | 同上 | **危险**：远程状态本身保留天数 |
| `backups_retention_period` | 整数 | `7` | 同上 | 备份文件保留天数 |

| 命令 | 作用 | 等效定时行为 |
|------|------|--------------|
| `tootctl media remove --days 7` | 清理 7 天前的远程媒体 | 不等效（仅检查 `created_at`） |
| `tootctl media remove --days 7 --keep-interacted` | 清理 7 天前且无交互的媒体 | 不等效（定时不支持交互保留） |
| `tootctl media remove --days 7 --dry-run` | 试运行，统计但不删除 | 无 |
| `tootctl media usage` | 查看存储使用统计 | 无 |
