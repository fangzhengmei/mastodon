# Mastodon Locale 多语言机制分析

## 概述

Mastodon 采用前后端分离的多语言架构：
- **后端 (Rails)**: 使用 I18n gem，YAML 格式翻译文件
- **前端 (React)**: 使用 react-intl，JSON 格式翻译文件

---

## 一、后端 Locale 机制

### 1.1 配置层

#### 1.1.1 可用语言与默认语言

**文件**: `config/initializers/i18n.rb`

```ruby
config.i18n.available_locales = [
  :af, :an, :ar, :ast, :be, :bg, :bn, :br, :bs, :ca,
  :ckb, :co, :cs, :cy, :da, :de, :el, :en, :'en-GB', :eo,
  # ... 约 100+ 种语言
  :'zh-CN', :'zh-HK', :'zh-TW',
]

config.i18n.default_locale = ENV['DEFAULT_LOCALE']&.to_sym || :en
```

**关键点**:
- 支持超过 100 种语言/地区变体
- 默认语言可通过 `DEFAULT_LOCALE` 环境变量配置
- 语言代码遵循 BCP 47 规范

#### 1.1.2 翻译文件结构

**位置**: `config/locales/*.yml`

```yaml
en:
  about:
    title: About
    contact_missing: Not set
  accounts:
    followers:
      one: Follower
      other: Followers
  user_mailer:
    welcome:
      title: Welcome, %{name}!
```

- 使用嵌套键结构
- 支持复数形式 (`one`/`other`)
- 支持变量插值 (`%{name}`)

---

### 1.2 Locale 选择机制

#### 1.2.1 核心实现

**文件**: `app/controllers/concerns/localized.rb`

```ruby
module Localized
  extend ActiveSupport::Concern

  included do
    around_action :set_locale
  end

  def set_locale(&block)
    I18n.with_locale(requested_locale || I18n.default_locale, &block)
  end

  private

  def requested_locale
    requested_locale_name   = available_locale_or_nil(params[:lang])
    requested_locale_name ||= available_locale_or_nil(current_user.locale) if respond_to?(:user_signed_in?) && user_signed_in?
    requested_locale_name ||= http_accept_language unless ENV['FORCE_DEFAULT_LOCALE'] == 'true'
    requested_locale_name
  end

  def http_accept_language
    HttpAcceptLanguage::Parser.new(request.headers.fetch('Accept-Language'))
      .language_region_compatible_from(I18n.available_locales)
  end
end
```

**Locale 优先级顺序** (从高到低):

| 优先级 | 来源 | 示例 | 说明 |
|--------|------|------|------|
| 1 | URL 参数 `params[:lang]` | `?lang=zh-CN` | 临时切换语言 |
| 2 | 用户设置 `current_user.locale` | 用户表的 `locale` 字段 | 已登录用户的持久化设置 |
| 3 | HTTP Header `Accept-Language` | `zh-CN,zh;q=0.9,en;q=0.8` | 浏览器语言偏好 |
| 4 | 默认 `I18n.default_locale` | `:en` | 系统默认 |

**强制默认语言**:
- 设置 `FORCE_DEFAULT_LOCALE=true` 可忽略 `Accept-Language` Header

#### 1.2.2 后端读取 Locale 的时机

**每个 HTTP 请求都会执行以下流程**:

```
请求进入控制器
    │
    ▼
around_action :set_locale 触发
    │
    ▼
调用 requested_locale() 确定语言
    ├── 检查 params[:lang]
    ├── 检查 current_user.locale (已登录)
    ├── 检查 Accept-Language Header
    └── 回退到 default_locale
    │
    ▼
I18n.with_locale(locale) 包裹整个请求处理
    │
    ▼
请求处理完成，locale 自动重置
```

**关键点**:
- `I18n.locale` 是**线程局部变量**
- 每个请求独立设置，互不干扰
- `around_action` 确保请求结束后自动恢复

#### 1.2.3 控制器集成

**文件**: `app/controllers/application_controller.rb`

```ruby
class ApplicationController < ActionController::Base
  include Localized  # 引入 locale 机制
  # ...
end
```

所有继承自 `ApplicationController` 的控制器都会自动应用 locale 选择逻辑。

---

### 1.3 用户语言设置的存储与修改

#### 1.3.1 存储位置

**文件**: `app/models/user.rb` (Schema 注释)

```ruby
# Table name: users
#
#  id                        :bigint(8)        not null, primary key
#  locale                    :string           # <-- 用户语言设置
#  # ... 其他字段
```

**Normalization**:

```ruby
# app/models/user.rb:126
normalizes :locale, with: ->(locale) { I18n.available_locales.exclude?(locale.to_sym) ? nil : locale }
```

- 自动验证 locale 是否在可用列表中
- 无效值会被转换为 `nil`

#### 1.3.2 修改语言设置的流程

**文件**: `app/controllers/settings/preferences/base_controller.rb`

```ruby
class Settings::Preferences::BaseController < Settings::BaseController
  def update
    if current_user.update(user_params)
      I18n.locale = current_user.locale  # 立即更新当前请求的 locale
      redirect_to after_update_redirect_path, notice: I18n.t('generic.changes_saved_msg')
    else
      render :show
    end
  end

  private

  def user_params
    params.expect(user: [:locale, :time_zone, chosen_languages: [], settings_attributes: UserSettings.keys])
  end
end
```

**修改语言设置的完整链路**:

```
1. 用户在设置页面选择语言 (appearance 页面)
        │
        ▼
2. 表单提交到 POST /settings/preferences/appearance
        │
        ▼
3. Settings::Preferences::BaseController#update 处理
        │
        ├── current_user.update(user_params)
        │       └── 保存 users.locale 字段到数据库
        │
        ├── I18n.locale = current_user.locale
        │       └── 更新当前线程的 locale（用于显示成功消息）
        │
        └── redirect_to after_update_redirect_path
                │
                ▼
4. 重定向触发新的 GET 请求
        │
        ▼
5. 新请求的 Localized concern 读取 current_user.locale
        │
        ▼
6. 渲染页面时设置 <html lang="new_locale">
```

#### 1.3.3 设置页面视图

**文件**: `app/views/settings/preferences/appearance/show.html.haml`

```haml
= simple_form_for current_user, url: settings_preferences_appearance_path do |f|
  .fields-row
    .fields-group
      = f.input :locale,
                collection: ui_languages,
                hint: false,
                include_blank: false,
                label_method: ->(locale) { native_locale_name(locale) },
                selected: I18n.locale,  # 当前选中的语言
                wrapper: :with_label
```

---

### 1.4 后端设置 HTML lang 属性

#### 1.4.1 Helper 方法

**文件**: `app/helpers/application_helper.rb:156`

```ruby
def html_attributes
  base = {
    lang: I18n.locale,           # <-- 关键：设置语言
    class: html_classes,
    'data-contrast': contrast.parameterize,
    'data-color-scheme': page_color_scheme.parameterize,
    # ...
  }
end
```

#### 1.4.2 Layout 中使用

**文件**: `app/views/layouts/application.html.haml`

```haml
%html{ html_attributes }
  %head
    = vite_preload_file_tag "mastodon/locales/#{I18n.locale}.json"  # 预加载前端语言包
    # ...
```

**关键点**:
- `<html lang="...">` 由后端根据 `I18n.locale` 渲染
- 同时预加载对应的前端语言包 JSON 文件

---

### 1.5 Rails View 中的翻译使用

#### 1.5.1 基础用法

```erb
<%= t 'about.title' %>
<%= t 'user_mailer.welcome.title', name: @resource.account.username %>
<%= t('.title') %>  # 相对键，基于控制器/视图路径
```

#### 1.5.2 实际示例

**文件**: `app/views/user_mailer/welcome.text.erb`

```erb
<%= t 'user_mailer.welcome.title', name: @resource.account.username %>
<%= t 'user_mailer.welcome.explanation' %>

<%= t('user_mailer.welcome.checklist_title') %>
===
1. <%= t('user_mailer.welcome.edit_profile_title') %>
   <%= t('user_mailer.welcome.edit_profile_step') %>
```

#### 1.5.3 复数形式

```yaml
en:
  accounts:
    followers:
      one: Follower
      other: Followers
```

```erb
<%= t 'accounts.followers', count: 1 %>  => "Follower"
<%= t 'accounts.followers', count: 5 %>  => "Followers"
```

---

### 1.6 邮件模板的多语言处理

#### 1.6.1 核心模式

**文件**: `app/mailers/application_mailer.rb`

```ruby
class ApplicationMailer < ActionMailer::Base
  protected

  def locale_for_account(account, &block)
    I18n.with_locale(account.user_locale || I18n.default_locale, &block)
  end
end
```

**文件**: `app/mailers/user_mailer.rb`

```ruby
class UserMailer < Devise::Mailer
  def welcome(user)
    @resource = user
    @suggestions = AccountSuggestions.new(@resource.account).get(5)
    # ...

    I18n.with_locale(locale) do
      mail subject: default_i18n_subject
    end
  end

  private

  def locale(use_current_locale: false)
    @resource.locale.presence || 
      (use_current_locale && I18n.locale) || 
      I18n.default_locale
  end
end
```

#### 1.6.2 邮件发送时的 Locale 选择

```ruby
# 场景1: 发送给特定用户 - 使用用户设置的 locale
I18n.with_locale(@resource.locale.presence || I18n.default_locale) do
  mail subject: I18n.t('devise.mailer.confirmation_instructions.subject')
end

# 场景2: use_current_locale=true 时优先使用当前请求的 locale
# 用于密码重置等场景，可能用户在未登录状态下操作
```

#### 1.6.3 邮件 Layout

**文件**: `app/views/layouts/mailer.html.haml`

```haml
%html{ lang: I18n.locale, dir: locale_direction }
```

邮件模板也会设置正确的 `lang` 属性。

---

### 1.7 日期格式本地化

#### 1.7.1 Rails I18n 日期格式化

使用 `l()` (localize) 方法：

```ruby
l(@appeal.created_at)                    # 默认格式
l(@appeal.created_at, format: :short)   # 短格式
l(@appeal.created_at, format: :long)    # 长格式
```

#### 1.7.2 配置文件中的格式定义

`config/locales/en.yml` 中定义：

```yaml
en:
  time:
    formats:
      default: "%a, %d %b %Y %H:%M:%S %z"
      short: "%d %b %H:%M"
      long: "%B %d, %Y %H:%M"
  date:
    formats:
      default: "%Y-%m-%d"
      short: "%b %d"
      long: "%B %d, %Y"
```

---

### 1.8 服务端 API 返回消息

#### 1.8.1 后端负责翻译的内容

**场景 1: 使用 I18n.t() 的业务错误**

**文件**: `app/controllers/api/v1/statuses_controller.rb`

```ruby
render json: { error: I18n.t('statuses.errors.in_reply_not_found') }, status: 404
render json: { error: I18n.t('statuses.errors.quoted_status_not_found') }, status: 404
```

**文件**: `app/controllers/api/v1/statuses/translations_controller.rb`

```ruby
render json: { error: I18n.t('translation.errors.quota_exceeded') }, status: 503
render json: { error: I18n.t('translation.errors.too_many_requests') }, status: 503
```

**文件**: `app/controllers/api/v1/accounts_controller.rb`

```ruby
render json: { error: I18n.t('accounts.self_follow_error') }, status: 403
```

**文件**: `app/controllers/api/v1/invites_controller.rb`

```ruby
render json: { error: I18n.t('invites.invalid') }, status: 401
```

---

**场景 2: Web Push 通知标题（后端必须翻译）**

**文件**: `app/serializers/web/notification_serializer.rb`

```ruby
class Web::NotificationSerializer < ActiveModel::Serializer
  attributes :access_token, :preferred_locale, :notification_id,
             :notification_type, :icon, :title, :body

  def preferred_locale
    current_push_subscription.user&.locale || I18n.default_locale
  end

  def title
    I18n.t("notification_mailer.#{object.type}.subject", 
           name: object.from_account.display_name.presence || object.from_account.username)
  end

  def body
    str = strip_tags(object.target_status&.spoiler_text.presence || 
                     object.target_status&.text || 
                     object.from_account.note)
    truncate(HTMLEntities.new.decode(str.to_str), length: 140, escape: false)
  end
end
```

**关键点**:
- Web Push 通知的 `title` 由后端翻译
- 因为通知直接发送到浏览器/设备，前端无法干预
- `preferred_locale` 也被返回，供前端参考

---

**场景 3: 通知 Fallback 文案**

**文件**: `app/serializers/concerns/notification_fallback_concern.rb`

```ruby
module NotificationFallbackConcern
  def fallback
    {
      title: fallback_title,
      summary: fallback_summary,
      description: nil,
    }
  end

  def fallback_title
    case object.type
    when :severed_relationships
      I18n.t(
        'notification_fallbacks.severed_relationships.title',
        name: object.account_relationship_severance_event.target_name
      )
    when :moderation_warning
      I18n.t('notification_fallbacks.moderation_warning.title')
    # ... 其他类型
    end
  end
end
```

---

#### 1.8.2 后端**不应该**翻译的内容（但目前有些实现不一致）

**场景 1: 通用认证/授权错误（硬编码英文）**

**文件**: `app/controllers/api/base_controller.rb`

```ruby
render json: { error: 'This method requires an authenticated user' }, status: 401
render json: { error: 'Your login is currently disabled' }, status: 403
render json: { error: 'Your login is missing a confirmed e-mail address' }, status: 403
render json: { error: 'Your login is currently pending approval' }, status: 403
render json: { error: Rack::Utils::HTTP_STATUS_CODES[code] }, status: code
```

**问题分析**:
- 这些是通用的、低层级的错误消息
- 目前硬编码英文，或使用 Rack 的英文状态码
- 前端收到后无法翻译

**设计决策**:
- **方案 A**: 后端全部翻译（当前部分实现）
- **方案 B**: 返回错误码，前端翻译（更清晰的边界）

---

#### 1.8.3 API 数据中的时间戳

**统一做法**: 返回 ISO 格式字符串，前端自行格式化

```ruby
# app/serializers/rest/notification_serializer.rb
attributes :id, :type, :created_at, :group_key

# created_at 是 Ruby DateTime 对象，被序列化为 ISO 8601 字符串
# 例如: "2024-01-15T10:30:00Z"
```

**前端处理**:

```typescript
// app/javascript/mastodon/utils/time.ts
export function formatTime({ timestamp, intl, ... }) {
  // timestamp 是 ISO 字符串或毫秒数
  // 使用 intl.formatDate() 或 formatMessage() 格式化
}
```

---

#### 1.8.4 API 返回文案责任边界总结

| 内容类型 | 负责方 | 说明 |
|----------|--------|------|
| **业务逻辑错误** | **后端** | 使用 `I18n.t()` 翻译，如 `statuses.errors.*` |
| **Web Push 通知标题** | **后端** | 直接发送到设备，前端无法干预 |
| **通知 Fallback 文案** | **后端** | 用于旧版客户端不支持的通知类型 |
| **通用认证错误** | **混合** | 目前硬编码英文，建议统一策略 |
| **HTTP 状态码描述** | **后端（英文）** | 使用 `Rack::Utils::HTTP_STATUS_CODES` |
| **时间戳** | **前端** | 返回 ISO 字符串，前端用 Intl API 格式化 |
| **通知类型** | **前端** | 返回 `type: "favourite"` 字符串，前端翻译 |
| **UI 文案** | **前端** | 所有按钮、标签、提示等 |

---

### 1.9 后台任务中的 Locale 处理

#### 1.9.1 Web Push 通知

**文件**: `app/workers/web/push_notification_worker.rb`

```ruby
def push_notification_json
  I18n.with_locale(@subscription.locale.presence || I18n.default_locale) do
    serialized_notification.to_json
  end
end
```

**关键点**:
- Sidekiq 后台任务不继承请求上下文
- 需要显式使用 `I18n.with_locale` 设置语言
- locale 存储在 `Web::PushSubscription` 模型中

#### 1.9.2 通用模式

```ruby
# 后台任务中处理多语言的标准模式
def perform(user_id, *args)
  user = User.find(user_id)
  
  I18n.with_locale(user.locale || I18n.default_locale) do
    # 执行需要翻译的逻辑
    send_email(user)
    generate_notification(user)
  end
end
```

---

## 二、前端 Locale 机制

### 2.1 配置层

#### 2.1.1 翻译文件结构

**位置**: `app/javascript/mastodon/locales/*.json`

```json
{
  "about.blocks": "被限制的服务器",
  "about.contact": "联系方式：",
  "account.badges.admin": "管理员",
  "account.familiar_followers_many": "被 {name1}、{name2}、及 {othersCount, plural, other {其他你认识的 # 人}} 关注",
  "relative_time.today": "今天",
  "relative_time.full.days": "{number, plural, one {# day} other {# days}} ago"
}
```

**特点**:
- 使用 ICU Message Format 语法
- 支持变量插值 `{name}`
- 支持复数形式 `{count, plural, one {...} other {...}}`
- 支持选择格式

#### 2.1.2 与后端翻译文件的差异

| 特性 | 后端 YAML | 前端 JSON |
|------|-----------|-----------|
| 格式 | YAML | JSON |
| 变量语法 | `%{name}` | `{name}` |
| 复数形式 | `one`/`other` 键 | `{count, plural, one {...} other {...}}` |
| 选择格式 | 不支持 | 支持 `select` |

---

### 2.2 Locale 加载机制

#### 2.2.1 前端读取 Locale 的时机

**前端只在两个时机读取 locale**:

1. **页面加载时** - 从 `<html lang="...">` 读取
2. **IntlProvider 挂载时** - 仅执行一次

**文件**: `app/javascript/mastodon/locales/load_locale.ts`

```typescript
import { Semaphore } from 'async-mutex';
import type { LocaleData } from './global_locale';
import { isLocaleLoaded, setLocale } from './global_locale';

const localeLoadingSemaphore = new Semaphore(1);

const localeFiles = import.meta.glob<{ default: LocaleData['messages'] }>([
  './*.json',
]);

export async function loadLocale() {
  // 从 <html> 标签读取 lang 属性
  const locale = document.querySelector<HTMLElement>('html')?.lang || 'en';

  await localeLoadingSemaphore.runExclusive(async () => {
    if (isLocaleLoaded()) return;  // 已加载则跳过

    const localeFile = Object.hasOwn(localeFiles, `./${locale}.json`)
      ? localeFiles[`./${locale}.json`]
      : localeFiles['./en.json'];

    if (!localeFile) throw new Error('Could not load the locale JSON file');

    const { default: localeData } = await localeFile();
    setLocale({ messages: localeData, locale });
  });
}
```

#### 2.2.2 加载流程

```
页面加载
    │
    ▼
React 应用挂载
    │
    ▼
IntlProvider 组件挂载
    │
    ▼
useEffect 触发 (空依赖数组，仅执行一次)
    │
    ▼
调用 loadLocale()
    │
    ├── 读取 <html lang="...">
    │
    ├── 动态 import 对应语言包 .json
    │
    └── setLocale() 存储到全局
    │
    ▼
Locale 加载完成，渲染子组件
```

#### 2.2.3 全局存储

**文件**: `app/javascript/mastodon/locales/global_locale.ts`

```typescript
export interface LocaleData {
  locale: string;
  messages: Record<string, string>;
}

let loadedLocale: LocaleData | undefined;

export function setLocale(locale: LocaleData) {
  loadedLocale = locale;
}

export function getLocale(): LocaleData {
  if (!loadedLocale) {
    if (isDevelopment()) {
      throw new Error('getLocale() called before any locale has been set');
    } else {
      return { locale: 'unknown', messages: {} };
    }
  }
  return loadedLocale;
}

export function isLocaleLoaded() {
  return !!loadedLocale;
}
```

#### 2.2.4 前端不支持运行时切换的原因

**文件**: `app/javascript/mastodon/locales/intl_provider.tsx`

```typescript
export const IntlProvider: React.FC = ({ children, ...props }) => {
  const [localeLoaded, setLocaleLoaded] = useState(false);

  useEffect(() => {
    async function loadLocaleData() {
      if (!isLocaleLoaded()) {
        await loadLocale();
      }
      setLocaleLoaded(true);
    }
    void loadLocaleData();
  }, []);  // ⚠️ 空依赖数组，只执行一次！

  if (!localeLoaded) return null;

  const { locale, messages } = getLocale();

  return (
    <BaseIntlProvider
      locale={locale}
      messages={messages}
      onError={onProviderError}
      {...props}
    >
      {children}
    </BaseIntlProvider>
  );
};
```

**关键点**:
- `useEffect` 的依赖数组是空的 `[]`
- 只在组件**首次挂载**时执行一次
- 即使 `<html lang>` 后续变化，也不会重新加载
- 必须**刷新页面**才能切换语言

---

### 2.3 React Intl 集成

#### 2.3.1 IntlProvider 封装

**文件**: `app/javascript/mastodon/locales/intl_provider.tsx`

```typescript
import { useEffect, useState } from 'react';
import { IntlProvider as BaseIntlProvider } from 'react-intl';
import { isProduction } from 'mastodon/utils/environment';
import { getLocale, isLocaleLoaded } from './global_locale';
import { loadLocale } from './load_locale';

function onProviderError(error: unknown) {
  if (isProduction()) return;
  
  // 处理浏览器缺少 Intl 数据的情况
  if (error instanceof Error && /MISSING_DATA/.exec(error.message)) {
    console.warn(error.message);
  }
  console.error(error);
}

export const IntlProvider: React.FC = ({ children, ...props }) => {
  const [localeLoaded, setLocaleLoaded] = useState(false);

  useEffect(() => {
    async function loadLocaleData() {
      if (!isLocaleLoaded()) {
        await loadLocale();
      }
      setLocaleLoaded(true);
    }
    void loadLocaleData();
  }, []);

  if (!localeLoaded) return null;  // 加载中显示空白

  const { locale, messages } = getLocale();

  return (
    <BaseIntlProvider
      locale={locale}
      messages={messages}
      onError={onProviderError}
      {...props}
    >
      {children}
    </BaseIntlProvider>
  );
};
```

#### 2.3.2 组件树中的位置

**文件**: `app/javascript/mastodon/containers/mastodon.jsx`

```typescript
export default class Mastodon extends PureComponent {
  render () {
    return (
      <IdentityContext.Provider value={this.identity}>
        <IntlProvider>
          <ReduxProvider store={store}>
            <ErrorBoundary>
              <Router>
                <Route path='/' component={UI} />
              </Router>
            </ErrorBoundary>
          </ReduxProvider>
        </IntlProvider>
      </IdentityContext.Provider>
    );
  }
}
```

---

### 2.4 组件中的翻译使用

#### 2.4.1 useIntl Hook (推荐)

```typescript
import { useIntl, defineMessages } from 'react-intl';

const messages = defineMessages({
  unfollow: {
    id: 'account.unfollow',
    defaultMessage: 'Unfollow @{name}',
  },
});

function UnfollowButton({ account }) {
  const intl = useIntl();
  
  return (
    <button>
      {intl.formatMessage(messages.unfollow, { name: account.acct })}
    </button>
  );
}
```

#### 2.4.2 FormattedMessage 组件

```typescript
import { FormattedMessage } from 'react-intl';

function WelcomeMessage({ name }) {
  return (
    <h1>
      <FormattedMessage
        id="welcome.title"
        defaultMessage="Welcome, {name}!"
        values={{ name }}
      />
    </h1>
  );
}
```

#### 2.4.3 defineMessages 的作用

```typescript
// defineMessages 用于静态分析 (babel-plugin-react-intl)
// 便于提取翻译键，不影响运行时
const messages = defineMessages({
  title: {
    id: 'component.title',
    defaultMessage: 'Title',  // 默认消息，作为 fallback
  },
});
```

---

### 2.5 日期和时间格式化

#### 2.5.1 相对时间格式化

**文件**: `app/javascript/mastodon/utils/time.ts`

```typescript
import type { IntlShape } from 'react-intl';
import { defineMessages } from 'react-intl';

const timeMessages = defineMessages({
  today: { id: 'relative_time.today', defaultMessage: 'today' },
  just_now: { id: 'relative_time.just_now', defaultMessage: 'now' },
  seconds: { id: 'relative_time.seconds', defaultMessage: '{number}s' },
  seconds_full: {
    id: 'relative_time.full.seconds',
    defaultMessage: '{number, plural, one {# second} other {# seconds}} ago',
  },
  minutes: { id: 'relative_time.minutes', defaultMessage: '{number}m' },
  minutes_full: {
    id: 'relative_time.full.minutes',
    defaultMessage: '{number, plural, one {# minute} other {# minutes}} ago',
  },
  // ... hours, days 类似
});

export function formatRelativePastTime({
  value,
  unit,
  intl,
  short = false,
}: {
  value: number;
  unit: TimeUnit;
  intl: Pick<IntlShape, 'formatMessage'>;
  short?: boolean;
}) {
  const absValue = Math.abs(value);
  if (unit === 'day') {
    return intl.formatMessage(
      short ? timeMessages.days : timeMessages.days_full,
      { number: absValue },
    );
  }
  // ... 其他单位
}
```

#### 2.5.2 绝对日期格式化

```typescript
export function formatAbsoluteTime({
  timestamp,
  intl,
  now = Date.now(),
}: {
  timestamp: number;
  intl: Pick<IntlShape, 'formatDate'>;
  now?: number;
}) {
  return intl.formatDate(timestamp, {
    month: 'short',
    day: 'numeric',
    year: isSameYear(timestamp, now) ? undefined : 'numeric',
  });
}
```

#### 2.5.3 时间格式化策略

```
时间戳
    ↓
判断时间差
    ├── 未来时间 → 使用 "remaining" 格式 (e.g., "5 seconds left")
    │
    └── 过去时间
            ├── < 10 秒 → "just now"
            ├── < 1 分钟 → "{n} seconds ago"
            ├── < 1 小时 → "{n} minutes ago"
            ├── < 1 天 → "{n} hours ago"
            ├── < 7 天 → "{n} days ago"
            └── >= 7 天 → 绝对日期 (e.g., "Jan 15" 或 "Jan 15, 2024")
```

---

### 2.6 前端入口点

**文件**: `app/javascript/mastodon/main.tsx`

```typescript
import { createRoot } from 'react-dom/client';
import Mastodon from 'mastodon/containers/mastodon';

function main() {
  return ready(async () => {
    const mountNode = document.getElementById('mastodon');
    const props = JSON.parse(
      mountNode.getAttribute('data-props') ?? '{}',
    );

    const { initializeEmoji } = await import('./features/emoji/index');
    await initializeEmoji();  // Emoji 也有自己的 locale 处理

    const root = createRoot(mountNode);
    root.render(<Mastodon {...props} />);
  });
}
```

**注意**: `IntlProvider` 在 `Mastodon` 组件内部，locale 加载是异步的。

---

## 三、用户修改语言后的完整生效链路

### 3.1 完整时序图

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  用户    │     │  前端    │     │  后端    │     │  数据库  │
└────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │                │
     │ 1. 在设置页面选择新语言          │                │
     │───────────────────────────────>│                │
     │                │                │                │
     │ 2. 提交表单    │                │                │
     │────────────────>│                │                │
     │                │                │                │
     │                │ POST /settings/preferences/appearance
     │                │───────────────>│                │
     │                │                │                │
     │                │                │ 3. 验证 params[:user][:locale]
     │                │                │                │
     │                │                │ 4. 更新 users.locale 字段
     │                │                │───────────────>│
     │                │                │                │
     │                │                │ 5. I18n.locale = new_locale
     │                │                │                │
     │                │ 6. 302 重定向 │                │
     │                │<───────────────│                │
     │                │                │                │
     │ 7. 浏览器跟随重定向              │                │
     │<───────────────│                │                │
     │                │                │                │
     │ 8. GET 新页面  │                │                │
     │────────────────>│                │                │
     │                │                │                │
     │                │                │ 9. Localized concern 读取
     │                │                │    current_user.locale
     │                │                │                │
     │                │                │ 10. 渲染 <html lang="new_locale">
     │                │                │    预加载 locales/new_locale.json
     │                │<───────────────│                │
     │                │                │                │
     │ 11. 新页面加载 │                │                │
     │<───────────────│                │                │
     │                │                │                │
     │                │ 12. React 应用挂载
     │                │                │                │
     │                │ 13. IntlProvider 读取 <html lang>
     │                │                │                │
     │                │ 14. 动态加载 new_locale.json
     │                │                │                │
     │                │ 15. 渲染组件（使用新语言）
     │                │                │                │
┌────┴─────┐     ┌────┴─────┐     ┌────┴─────┐     ┌────┴─────┐
│  用户    │     │  前端    │     │  后端    │     │  数据库  │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
```

### 3.2 关键步骤详解

#### 步骤 1-4: 后端处理更新

**文件**: `app/controllers/settings/preferences/base_controller.rb`

```ruby
def update
  if current_user.update(user_params)  # 保存到数据库
    I18n.locale = current_user.locale   # 更新当前请求的 locale
    redirect_to after_update_redirect_path, 
                notice: I18n.t('generic.changes_saved_msg')  # 使用新语言显示成功消息
  else
    render :show
  end
end
```

**关键点**:
- `current_user.update()` 保存到 `users.locale` 字段
- `I18n.locale = ...` 仅影响**当前请求**
- 重定向会触发**新的请求**，新请求会重新读取 locale

#### 步骤 5-8: 重定向与新请求

```ruby
# 新请求进入时
# app/controllers/concerns/localized.rb

def set_locale(&block)
  I18n.with_locale(requested_locale || I18n.default_locale, &block)
end

def requested_locale
  requested_locale_name   = available_locale_or_nil(params[:lang])
  requested_locale_name ||= available_locale_or_nil(current_user.locale)  # ← 读取新设置
  requested_locale_name ||= http_accept_language
  requested_locale_name
end
```

#### 步骤 9-10: 渲染新页面

**文件**: `app/helpers/application_helper.rb`

```ruby
def html_attributes
  {
    lang: I18n.locale,  # ← 使用新的 locale
    # ...
  }
end
```

**文件**: `app/views/layouts/application.html.haml`

```haml
%html{ html_attributes }
  %head
    = vite_preload_file_tag "mastodon/locales/#{I18n.locale}.json"  # 预加载新语言包
```

#### 步骤 11-15: 前端加载新语言

```typescript
// app/javascript/mastodon/locales/load_locale.ts

export async function loadLocale() {
  // 此时 <html lang> 已经是新值
  const locale = document.querySelector<HTMLElement>('html')?.lang || 'en';
  
  // 动态加载对应语言包
  const localeFile = localeFiles[`./${locale}.json`] || localeFiles['./en.json'];
  const { default: localeData } = await localeFile();
  
  setLocale({ messages: localeData, locale });
}
```

---

### 3.3 为什么需要刷新页面？

| 层级 | 支持热切换吗？ | 原因 |
|------|---------------|------|
| **后端** | ✅ 支持 | 每个请求独立读取 `current_user.locale` |
| **HTML lang** | ✅ 支持 | 后端渲染时设置 |
| **前端语言包** | ❌ 不支持 | `IntlProvider` 只在挂载时加载一次 |
| **React 组件** | ❌ 不支持 | `useEffect([])` 只执行一次 |

**当前实现的限制**:

```typescript
// app/javascript/mastodon/locales/intl_provider.tsx

useEffect(() => {
  // 空依赖数组 = 只执行一次
  // 即使 <html lang> 变化，也不会重新执行
}, []);
```

**如果要支持热切换，需要**:

1. 监听 `<html lang>` 属性变化
2. 或者通过 props/context 传递 locale
3. 重新加载语言包并更新 `IntlProvider`

---

## 四、前后端 Locale 边界与数据流

### 4.1 架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                           用户浏览器                                   │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  HTML 页面（后端渲染）                                         │   │
│  │  <html lang="zh-CN">                                          │   │
│  │  <link rel="preload" href="/locales/zh-CN.json">             │   │
│  │  <script id="initial-state">                                   │   │
│  │    { "meta": { "locale": "zh-CN", ... } }                     │   │
│  │  </script>                                                     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  React SPA（前端运行时）                                       │   │
│  │                                                               │   │
│  │  IntlProvider                                                 │   │
│  │  ├── 读取 <html lang="zh-CN">                                │   │
│  │  ├── 动态加载 zh-CN.json                                      │   │
│  │  └── 使用 react-intl 格式化所有 UI 文案                      │   │
│  │                                                               │   │
│  │  数据流向:                                                    │   │
│  │  API 返回 ISO 时间戳 ──► 前端用 intl.formatDate() 格式化    │   │
│  │  API 返回 type: "favourite" ──► 前端用 formatMessage() 翻译 │   │
│  └─────────────────────────────────────────────────────────────┘   │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ HTTP 请求
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           Rails 后端                                   │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  请求处理流程（每个请求独立）                                   │   │
│  │                                                               │   │
│  │  1. Localized concern 确定 locale                            │   │
│  │     ├── params[:lang]                                        │   │
│  │     ├── current_user.locale ←── 数据库持久化存储            │   │
│  │     ├── Accept-Language Header                               │   │
│  │     └── default_locale                                       │   │
│  │                                                               │   │
│  │  2. 执行业务逻辑                                               │   │
│  │     ├── 渲染 Rails Views: 使用 t() 翻译                      │   │
│  │     ├── 发送邮件: I18n.with_locale() 包裹                    │   │
│  │     ├── API 响应:                                             │   │
│  │         ├── 部分错误: I18n.t() 翻译                          │   │
│  │         ├── Web Push 标题: 必须后端翻译                       │   │
│  │         └── 数据: 返回原始值（时间戳、type 等）               │   │
│  │                                                               │   │
│  │  3. 渲染响应                                                   │   │
│  │     ├── HTML: 设置 <html lang="...">                         │   │
│  │     └── JSON: 返回原始数据或已翻译的错误消息                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 4.2 前后端 Locale 源对比

| 维度 | 后端 Rails | 前端 React |
|------|-----------|------------|
| **Locale 来源** | 1. params[:lang]<br>2. current_user.locale<br>3. Accept-Language<br>4. default_locale | 仅从 `<html lang="...">` 读取 |
| **读取时机** | **每个 HTTP 请求** | **页面加载时仅一次** |
| **持久化存储** | `users.locale` 数据库字段 | 无（依赖后端） |
| **翻译文件** | `config/locales/*.yml` | `app/javascript/mastodon/locales/*.json` |
| **翻译引擎** | I18n gem | react-intl (基于 Intl API) |
| **变量语法** | `%{name}` | `{name}` |
| **热切换支持** | ✅ 支持 | ❌ 需刷新页面 |

---

### 4.3 前后端职责边界

#### 4.3.1 后端负责

| 领域 | 具体内容 | 时机/位置 |
|------|----------|-----------|
| **Locale 决策** | 确定用户语言的优先级规则 | 每个请求的 `around_action` |
| **持久化存储** | 读取/写入 `users.locale` 字段 | Settings 控制器 |
| **HTML 渲染** | 设置 `<html lang="...">` | Layout 渲染时 |
| **语言包预加载** | `vite_preload_file_tag` | HTML head 中 |
| **Rails Views** | 服务端渲染页面的文案 | `.erb`/`.haml` 模板 |
| **邮件模板** | 所有邮件的主题和内容 | Mailers + Views |
| **Web Push 标题** | 推送通知的标题（必须后端翻译） | `Web::NotificationSerializer` |
| **通知 Fallback** | 旧版客户端不支持的通知类型 | `NotificationFallbackConcern` |
| **部分 API 错误** | 业务逻辑错误消息 | 控制器中 `I18n.t()` |
| **后台任务** | Sidekiq job 中的通知和邮件 | Worker 中 `I18n.with_locale` |

#### 4.3.2 前端负责

| 领域 | 具体内容 | 时机/位置 |
|------|----------|-----------|
| **语言包加载** | 从 `<html lang>` 读取，动态加载 JSON | `load_locale.ts` |
| **React 组件** | 所有 SPA 界面的文案 | `useIntl` / `FormattedMessage` |
| **时间格式化** | 相对时间、绝对日期显示 | `utils/time.ts` |
| **数字格式化** | 数字、百分比等 | `react-intl` |
| **通知类型翻译** | 根据 `type: "favourite"` 翻译显示文案 | 通知组件 |
| **表单验证** | 客户端验证提示 | 表单组件 |
| **Emoji 选择器** | Emoji 分类名称 | `features/emoji/locale.ts` |

---

### 4.4 API 响应内容的责任划分

#### 4.4.1 数据流向图

```
后端 API 响应
    │
    ├── 已翻译的字符串（后端负责）
    │       ├── Web Push 通知的 title
    │       ├── 通知 fallback 的 title/summary
    │       └── 部分业务错误消息（使用 I18n.t()）
    │
    ├── 原始数据（前端负责翻译/格式化）
    │       ├── ISO 时间戳 ──► 前端用 intl.formatDate()
    │       ├── type: "favourite" ──► 前端用 formatMessage()
    │       ├── count: 5 ──► 前端用复数形式翻译
    │       └── 所有 UI 交互相关的文案
    │
    └── 硬编码英文（问题点）
            ├── 通用认证错误
            └── HTTP 状态码描述
```

#### 4.4.2 详细对照表

| API 响应字段 | 示例值 | 负责方 | 处理方式 |
|--------------|--------|--------|----------|
| **error** (业务) | `"无法回复不存在的嘟文"` | **后端** | `I18n.t('statuses.errors.in_reply_not_found')` |
| **error** (认证) | `"This method requires authentication"` | **混合** | 目前硬编码英文 |
| **notification.title** | `"@alice 赞了你的嘟文"` | **后端** | Web Push 直接发送到设备 |
| **notification.fallback.title** | `"服务器关系已 severed"` | **后端** | 旧版客户端兼容 |
| **created_at** | `"2024-01-15T10:30:00Z"` | **前端** | `intl.formatDate()` 格式化 |
| **type** | `"favourite"`, `"reblog"` | **前端** | `formatMessage({id: 'notification.favourite'})` |
| **count** | `5` | **前端** | `{count, plural, one {...} other {...}}` |
| **locked** | `true` | **前端** | 决定显示"已锁定"图标和文案 |

---

### 4.5 前后端同步机制

#### 4.5.1 单向数据流

```
┌──────────────────────────────────────────────────────────────┐
│                        同步流程                                 │
└──────────────────────────────────────────────────────────────┘

用户修改语言设置
        │
        ▼
后端更新 users.locale 字段
        │
        ▼
后端渲染 HTML 时设置
        │
        ├── <html lang="new_locale">
        └── preload /locales/new_locale.json
        │
        ▼
前端加载时读取
        │
        ├── 从 <html lang> 读取语言代码
        └── 动态加载对应的 .json 语言包
        │
        ▼
使用新语言渲染组件
```

#### 4.5.2 关键同步点

| 同步点 | 后端 | 前端 |
|--------|------|------|
| **语言代码传递** | 渲染 `<html lang="...">` | 读取 `document.documentElement.lang` |
| **语言包预加载** | `vite_preload_file_tag` | `import.meta.glob` 动态导入 |
| **Initial State** | `meta.locale` 字段 | `initialState.meta.locale` |

#### 4.5.3 Initial State 中的 locale

**文件**: `app/serializers/initial_state_serializer.rb`

```ruby
def default_meta_store
  {
    # ...
    locale: I18n.locale,  # 也包含在 initial_state 中
    # ...
  }
end
```

**文件**: `app/javascript/mastodon/initial_state.ts`

```typescript
interface InitialStateMeta {
  // ...
  locale: string;  // 前端也能读取
  // ...
}

// 注意：前端目前不用这个来确定语言，而是用 <html lang>
// 但可以用来创建 Intl.DisplayNames 等
```

---

## 五、用户语言设置生效链路总结

### 5.1 完整链路速览

```
┌─────────────────────────────────────────────────────────────────┐
│                    用户修改语言设置                                │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    ▼                           ▼
            ┌───────────────┐           ┌───────────────┐
            │   后端生效    │           │   前端生效    │
            └───────┬───────┘           └───────┬───────┘
                    │                           │
                    ▼                           ▼
            每个请求读取              页面刷新后读取
            current_user.locale       <html lang="...">
                    │                           │
                    ▼                           ▼
            I18n.locale 设置            加载对应语言包
                    │                           │
                    ▼                           ▼
            所有 Rails Views             所有 React 组件
            t() 翻译生效                 formatMessage() 生效
```

### 5.2 生效时机对照表

| 场景 | 后端何时生效 | 前端何时生效 |
|------|-------------|-------------|
| **用户刚修改设置** | 重定向后的**第一个新请求** | **页面刷新后** |
| **后续 API 请求** | **每个请求**都读取 `current_user.locale` | 无需额外操作（语言包已加载） |
| **后台任务发送邮件** | Job 执行时 `I18n.with_locale(user.locale)` | 不涉及 |
| **Web Push 通知** | 序列化时 `I18n.t()` | 不涉及 |
| **新用户注册** | 注册后可设置，下次登录生效 | 页面刷新后 |

### 5.3 关键代码位置索引

#### 后端语言设置相关

| 文件路径 | 职责 |
|----------|------|
| `app/models/user.rb:126` | locale 字段 normalization |
| `app/controllers/concerns/localized.rb` | 每个请求的 locale 选择 |
| `app/controllers/settings/preferences/base_controller.rb:6-13` | 语言设置更新 |
| `app/helpers/application_helper.rb:156-161` | `html_attributes` 设置 lang |
| `app/views/layouts/application.html.haml:1,37` | 使用 `html_attributes` 和预加载语言包 |

#### 前端语言加载相关

| 文件路径 | 职责 |
|----------|------|
| `app/javascript/mastodon/locales/load_locale.ts:14` | 从 `<html lang>` 读取 |
| `app/javascript/mastodon/locales/intl_provider.tsx:33-42` | `useEffect([])` 只执行一次 |
| `app/javascript/mastodon/containers/mastodon.jsx:47` | `IntlProvider` 位置 |

#### API 响应翻译相关

| 文件路径 | 职责 |
|----------|------|
| `app/serializers/web/notification_serializer.rb:31-34` | Web Push 标题翻译 |
| `app/serializers/concerns/notification_fallback_concern.rb` | 通知 fallback 翻译 |
| `app/controllers/api/v1/statuses_controller.rb:160,168` | 业务错误翻译示例 |
| `app/controllers/api/base_controller.rb:54-73` | 硬编码英文错误示例 |

---

## 六、存在的问题与建议

### 6.1 当前问题

#### 问题 1: API 错误消息不一致

**现状**:
- 部分错误使用 `I18n.t()` 翻译
- 部分错误硬编码英文
- 部分使用 `Rack::Utils::HTTP_STATUS_CODES`（英文）

**影响**:
- 非英语用户看到混合语言的错误消息
- 前端无法统一处理

**建议**:
1. **短期**: 将所有硬编码英文错误改为 `I18n.t()`
2. **中期**: 考虑引入错误码机制，前端根据错误码翻译

```ruby
# 建议方案：返回错误码 + 已翻译的消息
render json: { 
  error_code: 'authentication_required',
  error: I18n.t('errors.authentication_required')
}, status: 401
```

---

#### 问题 2: 前端语言切换需刷新页面

**现状**:
- `IntlProvider` 的 `useEffect` 依赖数组为空
- 只在页面加载时执行一次
- 用户修改语言后必须刷新

**影响**:
- 用户体验不流畅
- 与现代 SPA 应用的预期不符

**建议**:
1. 监听 `<html lang>` 属性变化（使用 MutationObserver）
2. 或通过路由/状态管理传递 locale
3. 支持运行时重新加载语言包

```typescript
// 概念性改进方案
function IntlProvider({ children }) {
  const [localeData, setLocaleData] = useState(null);
  
  useEffect(() => {
    // 监听 html lang 变化
    const observer = new MutationObserver(() => {
      const newLocale = document.documentElement.lang;
      loadLocale(newLocale).then(setLocaleData);
    });
    
    observer.observe(document.documentElement, { 
      attributes: true, 
      attributeFilter: ['lang'] 
    });
    
    // 初始加载
    loadLocale(document.documentElement.lang).then(setLocaleData);
    
    return () => observer.disconnect();
  }, []);
  
  // ...
}
```

---

#### 问题 3: 翻译文件重复维护

**现状**:
- 后端: `config/locales/*.yml`
- 前端: `app/javascript/mastodon/locales/*.json`
- 两套文件，键可能不对应

**影响**:
- 添加新翻译时需要修改两处
- 可能出现前后端翻译不一致

**建议**:
1. **短期**: 建立清晰的命名规范和文档
2. **中期**: 考虑从单一数据源生成两套文件
3. **长期**: 探索同构翻译方案（但改动较大）

---

### 6.2 架构优点

1. **清晰的分离**: 前后端各自管理自己的翻译，互不干扰
2. **灵活的后端优先级**: 支持 URL 参数、用户设置、浏览器偏好多层级
3. **异步加载**: 前端按需加载语言包，减少初始包大小
4. **标准技术**: 使用 I18n (Rails 标准) 和 react-intl (React 生态标准)
5. **线程安全**: 后端使用 `I18n.with_locale` 确保线程安全

---

## 七、关键文件索引

### 7.1 后端文件

| 文件路径 | 职责 |
|----------|------|
| `config/initializers/i18n.rb` | 可用语言列表、默认语言配置 |
| `config/locales/*.yml` | 后端翻译文件 (YAML 格式) |
| `app/controllers/concerns/localized.rb` | Locale 选择核心逻辑 |
| `app/controllers/application_controller.rb` | 引入 Localized 模块 |
| `app/controllers/settings/preferences/base_controller.rb` | 语言设置更新 |
| `app/models/user.rb:126` | locale 字段 normalization |
| `app/helpers/application_helper.rb:156` | html_attributes 设置 lang |
| `app/mailers/application_mailer.rb` | 邮件发送的 locale 辅助方法 |
| `app/mailers/user_mailer.rb` | 用户邮件的 locale 处理 |
| `app/workers/web/push_notification_worker.rb` | 后台任务的 locale 处理 |
| `app/serializers/web/notification_serializer.rb` | Web Push 标题翻译 |
| `app/serializers/concerns/notification_fallback_concern.rb` | 通知 fallback 翻译 |
| `app/views/layouts/application.html.haml` | 使用 html_attributes 和预加载语言包 |

### 7.2 前端文件

| 文件路径 | 职责 |
|----------|------|
| `app/javascript/mastodon/locales/*.json` | 前端翻译文件 (JSON 格式) |
| `app/javascript/mastodon/locales/global_locale.ts` | 全局 locale 存储 |
| `app/javascript/mastodon/locales/load_locale.ts` | 从 <html lang> 读取，动态加载 |
| `app/javascript/mastodon/locales/intl_provider.tsx` | React IntlProvider 封装 |
| `app/javascript/mastodon/locales/index.ts` | 导出统一接口 |
| `app/javascript/mastodon/utils/time.ts` | 时间格式化工具 |
| `app/javascript/mastodon/features/emoji/locale.ts` | Emoji 选择器 locale |
| `app/javascript/mastodon/containers/mastodon.jsx` | IntlProvider 位置 |
| `app/javascript/mastodon/initial_state.ts` | initial_state.meta.locale |

---

---

## 八、前后端可用语言集合的对齐情况

### 8.1 可用语言数量对比

| 层级 | 数量 | 来源 |
|------|------|------|
| **后端 (Rails)** | 102 种 | `config/initializers/i18n.rb` 中的 `available_locales` |
| **前端 (React)** | 110+ 种 | `app/javascript/mastodon/locales/*.json` |

**关键发现**: 前端的可用语言数量比后端多约 8 种。

---

### 8.2 语言集合差异分析

#### 8.2.1 后端独有的语言

**后端有但前端没有的语言：无**

后端 `available_locales` 中的所有 102 种语言，前端都有对应的 `.json` 文件。

#### 8.2.2 前端独有的语言（后端没有）

**前端有但后端没有的语言（9 种）：**

| 语言代码 | 语言名称 | 说明 |
|----------|----------|------|
| `az` | 阿塞拜疆语 (Azerbaijani) | |
| `fil` | 菲律宾语/他加禄语 (Filipino/Tagalog) | |
| `lad` | 拉迪诺语 (Ladino) | 西班牙犹太人使用的语言 |
| `ne` | 尼泊尔语 (Nepali) | |
| `ry` | 鲁塞尼亚语 (Rusyn) | 喀尔巴阡鲁塞尼亚语 |
| `tai` | 傣语 (Tai) | |
| `tok` | 巴布亚皮钦语 (Tok Pisin) | 巴布亚新几内亚使用 |
| `tlh` | 克林贡语 (Klingon) | **人造语言**（《星际迷航》） |
| `uz` | 乌兹别克语 (Uzbek) | |

---

### 8.3 语言集合不一致的影响

#### 8.3.1 用户设置语言时的回退

**后端验证逻辑**:

```ruby
# app/models/user.rb:126
normalizes :locale, with: ->(locale) { 
  I18n.available_locales.exclude?(locale.to_sym) ? nil : locale 
}
```

**场景分析**：

假设用户的浏览器语言是 `fil`（菲律宾语），或者在请求中尝试设置 `?lang=fil`：

```
1. 用户请求 ?lang=fil 或浏览器 Accept-Language: fil
         │
         ▼
2. 后端 Localized concern 调用 available_locale_or_nil('fil')
         │
         ▼
3. 'fil'.to_sym = :fil 不在 I18n.available_locales 中
         │
         ▼
4. available_locale_or_nil 返回 nil
         │
         ▼
5. requested_locale 回退到下一个优先级：
   - 已登录用户：回退到 current_user.locale
   - 未登录用户：回退到 Accept-Language 的下一个选项
         │
         ▼
6. 如果所有选项都不匹配，最终回退到 I18n.default_locale (:en)
```

**具体影响的页面/响应**：

| 场景 | 后端行为 | 实际使用语言 |
|------|----------|-------------|
| 用户在设置页面尝试选择 `fil` | 下拉列表中**不显示**该选项 | 无法选择 |
| 用户手动构造 URL `?lang=fil` | 被 `available_locale_or_nil` 忽略 | 回退到下一个优先级 |
| 浏览器 Accept-Language 首选 `fil` | 无法匹配，尝试下一个语言 | 回退到下一个匹配项 |
| 用户表中 `locale` 字段是 `fil` | normalization 时被转为 `nil` | 使用 default_locale |

---

#### 8.3.2 浏览器语言协商的回退

**后端协商逻辑**:

```ruby
# app/controllers/concerns/localized.rb:23-25

def http_accept_language
  HttpAcceptLanguage::Parser
    .new(request.headers.fetch('Accept-Language'))
    .language_region_compatible_from(I18n.available_locales)
end
```

**`language_region_compatible_from` 的行为**：

这个方法会尝试：
1. 精确匹配（如 `zh-CN` → `zh-CN`）
2. 语言级别匹配（如 `zh-TW` → `zh-CN` 如果只有后者可用）
3. 回退到 `nil`

**示例场景**：

```
浏览器发送: Accept-Language: fil-PH, fil;q=0.9, en-US;q=0.8, en;q=0.7

后端处理:
1. 尝试 fil-PH → 不在 available_locales
2. 尝试 fil → 不在 available_locales  
3. 尝试 en-US → 不在 available_locales (后端只有 en, en-GB)
4. 尝试 en → 在 available_locales ✅

结果: 使用 :en
```

**对比：前端的行为**：

如果后端 somehow 允许 `fil` 通过，前端会：

```typescript
// app/javascript/mastodon/locales/load_locale.ts:25-27

const localeFile = Object.hasOwn(localeFiles, `./${locale}.json`)
  ? localeFiles[`./${locale}.json`]
  : localeFiles['./en.json'];
```

- 前端**有** `fil.json`，所以会成功加载
- 但后端没有 `fil` 的翻译

**这会导致什么问题？**

| 组件类型 | 语言 |
|----------|------|
| React 组件文案 | **菲律宾语** (前端 `fil.json`) |
| Rails Views (登录页面、设置页面) | **英语** (后端回退到 `:en`) |
| 邮件 | **英语** (后端回退到 `:en`) |
| Web Push 通知 | **英语** (后端回退到 `:en`) |
| API 错误消息 | **英语** (后端回退到 `:en`) |

**用户体验**: 混合语言界面，部分内容是菲律宾语，部分是英语。

---

#### 8.3.3 API 返回文案的回退

**场景**: 用户使用前端独有的语言

```
假设:
- 用户浏览器: Accept-Language: tlh (克林贡语)
- 但 tlh 不在后端 available_locales

后端行为:
1. http_accept_language 尝试匹配 tlh → 失败
2. 回退到 default_locale (:en)
3. 所有后端翻译使用英语

前端行为:
1. 从 <html lang="en"> 读取
2. 加载 en.json
3. 所有 React 组件使用英语

结果: 统一使用英语（因为后端回退到 en，前端跟随）
```

**另一个场景**: 后端 somehow 设置了前端独有的语言

```
假设:
- 后端 somehow 允许 fil 通过（比如直接设置 I18n.locale）
- 渲染 <html lang="fil">

后端行为:
1. 没有 fil.yml 翻译文件
2. I18n.t() 会使用英语 fallback（或抛出异常）

前端行为:
1. 从 <html lang="fil"> 读取
2. 加载 fil.json ✓（前端有这个文件）
3. React 组件使用菲律宾语

结果: 混合语言！
- 后端渲染的内容：英语
- 前端 React 组件：菲律宾语
```

---

### 8.4 回退机制总结

#### 8.4.1 回退流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    语言设置与回退流程                              │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    ▼                           ▼
            ┌───────────────┐           ┌───────────────┐
            │   后端验证    │           │   前端加载    │
            └───────┬───────┘           └───────┬───────┘
                    │                           │
                    ▼                           ▼
            语言在 available_locales?    语言有对应的 .json 文件?
                    │                           │
          ┌─────────┴─────────┐         ┌───────┴───────┐
          │                   │         │               │
          ▼                   ▼         ▼               ▼
        是 ✓                否 ✗      是 ✓            否 ✗
          │                   │         │               │
          ▼                   ▼         ▼               ▼
    使用该语言         回退到下一个   使用该语言     回退到 en.json
                    优先级或 default
```

#### 8.4.2 回退优先级详细表

| 层级 | 验证/加载逻辑 | 回退路径 |
|------|--------------|----------|
| **后端 - 用户设置** | `normalizes :locale` 检查 `available_locales` | 无效值转为 `nil`，下次请求使用默认 |
| **后端 - URL 参数** | `available_locale_or_nil(params[:lang])` | 无效则忽略，尝试下一个优先级 |
| **后端 - 用户 locale** | `available_locale_or_nil(current_user.locale)` | 无效则忽略，尝试 `Accept-Language` |
| **后端 - Accept-Language** | `language_region_compatible_from(available_locales)` | 无匹配则返回 `nil` |
| **后端 - 最终** | `requested_locale \|\| default_locale` | 回退到 `:en` |
| **前端 - 加载** | `Object.hasOwn(localeFiles, \`./${locale}.json\`)` | 回退到 `./en.json` |

---

### 8.5 影响范围对照表

| 场景 | 后端行为 | 前端行为 | 最终语言 |
|------|----------|----------|----------|
| **用户尝试选择前端独有语言** | 下拉列表不显示 | N/A | 无法选择 |
| **URL 参数 `?lang=fil`** | 被忽略，回退 | 跟随后端 | 回退到下一个优先级 |
| **Accept-Language 首选 `fil`** | 无法匹配，回退 | 跟随后端 | 回退到下一个匹配项 |
| **数据库 `locale` 是 `fil`** | normalization 转 `nil` | N/A | 使用 default_locale |
| **后端 somehow 允许 `fil`** | 使用英语 fallback | 加载 `fil.json` | **混合语言** ⚠️ |
| **前端独有语言，但后端回退到 `en`** | 使用英语 | 加载 `en.json` | **统一英语** |

---

### 8.6 为什么会有这种不一致？

#### 8.6.1 可能的原因

1. **贡献流程不同**
   - 前端翻译可能通过 Crowdin/Weblate 等平台
   - 后端翻译可能有不同的审核流程

2. **测试覆盖不同**
   - 前端独有的语言可能是"实验性"的
   - 后端可能更谨慎，只添加经过充分测试的语言

3. **人造语言**
   - `tlh` (克林贡语) 是人造语言
   - 这类语言通常只在前端添加，用于趣味/展示

#### 8.6.2 风险评估

**低风险场景**:
- 用户浏览器语言是前端独有的，但有其他可匹配的语言
- 例如：`Accept-Language: fil, en-US, en` → 回退到 `en`

**高风险场景**:
- 用户浏览器语言只有前端独有的语言
- 例如：`Accept-Language: fil` → 回退到 `en`（这是预期行为）

**潜在 Bug 场景**:
- 后端某处绕过了 `available_locales` 检查
- 直接设置 `I18n.locale = 'fil'`
- 这会导致：
  - 后端渲染的内容：英语（I18n fallback）
  - 前端 React 组件：菲律宾语
  - **混合语言界面**

---

### 8.7 建议

#### 8.7.1 短期建议

1. **保持现状**：当前设计是合理的
   - 后端严格验证 `available_locales`
   - 前端有额外的语言作为"友好 fallback"
   - 实际上用户不会遇到混合语言，因为后端会先回退

2. **文档化**：明确说明
   - 哪些语言只在前端可用
   - 回退机制的详细行为

#### 8.7.2 中期建议

1. **同步语言集合**
   - 考虑将前端独有的语言添加到后端 `available_locales`
   - 或者从前端移除这些语言（如果后端不会支持）

2. **添加一致性检查**
   - 在 CI 中添加检查，确保前后端语言集合一致
   - 或明确记录差异的原因

#### 8.7.3 长期建议

1. **统一翻译管理**
   - 使用统一的翻译管理平台
   - 确保前后端翻译同步更新

2. **语言启用机制**
   - 实现语言的"启用/禁用"机制
   - 区分：
     - `available_locales`: 代码层面支持
     - `enabled_locales`: 实例层面启用

---

## 九、总结

### 9.1 前后端语言机制核心差异

| 维度 | 后端 Rails | 前端 React |
|------|-----------|------------|
| **语言数量** | 102 种 | 110+ 种 |
| **语言验证** | 严格检查 `available_locales` | 宽松，不存在则回退到 `en` |
| **读取时机** | 每个 HTTP 请求 | 页面加载时仅一次 |
| **热切换** | ✅ 支持（每个请求重新读取） | ❌ 需刷新页面 |
| **回退机制** | 多层级优先级回退 | 仅回退到 `en.json` |

### 9.2 语言不一致的实际影响

**实际上，用户通常不会遇到混合语言**，因为：

```
用户请求 (fil)
     │
     ▼
后端验证：fil 不在 available_locales
     │
     ▼
回退到 default_locale (:en)
     │
     ▼
渲染 <html lang="en">
     │
     ▼
前端加载 en.json
     │
     ▼
统一使用英语 ✓
```

**只有当后端绕过 `available_locales` 检查时，才会出现混合语言**。

---

*分析基于 Mastodon 代码库版本: 2024年*
