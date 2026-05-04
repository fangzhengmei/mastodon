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

#### 1.2.2 控制器集成

**文件**: `app/controllers/application_controller.rb`

```ruby
class ApplicationController < ActionController::Base
  include Localized  # 引入 locale 机制
  # ...
end
```

所有继承自 `ApplicationController` 的控制器都会自动应用 locale 选择逻辑。

---

### 1.3 Rails View 中的翻译使用

#### 1.3.1 基础用法

```erb
<%= t 'about.title' %>
<%= t 'user_mailer.welcome.title', name: @resource.account.username %>
<%= t('.title') %>  # 相对键，基于控制器/视图路径
```

#### 1.3.2 实际示例

**文件**: `app/views/user_mailer/welcome.text.erb`

```erb
<%= t 'user_mailer.welcome.title', name: @resource.account.username %>
<%= t 'user_mailer.welcome.explanation' %>

<%= t('user_mailer.welcome.checklist_title') %>
===
1. <%= t('user_mailer.welcome.edit_profile_title') %>
   <%= t('user_mailer.welcome.edit_profile_step') %>
```

#### 1.3.3 复数形式

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

### 1.4 邮件模板的多语言处理

#### 1.4.1 核心模式

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

#### 1.4.2 邮件发送时的 Locale 选择

```ruby
# 场景1: 发送给特定用户 - 使用用户设置的 locale
I18n.with_locale(@resource.locale.presence || I18n.default_locale) do
  mail subject: I18n.t('devise.mailer.confirmation_instructions.subject')
end

# 场景2: use_current_locale=true 时优先使用当前请求的 locale
# 用于密码重置等场景，可能用户在未登录状态下操作
```

---

### 1.5 日期格式本地化

#### 1.5.1 Rails I18n 日期格式化

使用 `l()` (localize) 方法：

```ruby
l(@appeal.created_at)                    # 默认格式
l(@appeal.created_at, format: :short)   # 短格式
l(@appeal.created_at, format: :long)    # 长格式
```

#### 1.5.2 配置文件中的格式定义

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

### 1.6 服务端 API 返回消息

#### 1.6.1 两种处理方式

**方式1: 使用 I18n.t() 翻译 (推荐)**

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

**方式2: 硬编码英文消息 (不推荐)**

**文件**: `app/controllers/api/base_controller.rb`

```ruby
render json: { error: 'This method requires an authenticated user' }, status: 401
render json: { error: 'Your login is currently disabled' }, status: 403
render json: { error: 'Your login is missing a confirmed e-mail address' }, status: 403
render json: { error: Rack::Utils::HTTP_STATUS_CODES[code] }, status: code
```

#### 1.6.2 问题分析

**不一致性**:
- 部分错误消息使用 I18n 翻译
- 部分错误消息硬编码英文
- 部分使用 `Rack::Utils::HTTP_STATUS_CODES` (英文状态码描述)

**影响**:
- API 调用方收到的错误消息语言不一致
- 前端可能需要自己处理翻译逻辑

---

### 1.7 后台任务中的 Locale 处理

#### 1.7.1 Web Push 通知

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

#### 1.7.2 通用模式

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

#### 2.2.1 核心实现

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
  const locale = document.querySelector<HTMLElement>('html')?.lang || 'en';

  await localeLoadingSemaphore.runExclusive(async () => {
    if (isLocaleLoaded()) return;

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
1. 从 <html lang="..."> 获取当前语言
        ↓
2. 使用 Semaphore 确保单例加载 (防止并发问题)
        ↓
3. 检查是否已加载
        ↓
4. 使用 import.meta.glob 动态导入对应 JSON
        ↓
5. 不存在则回退到 en.json
        ↓
6. 调用 setLocale() 存储到全局
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

```
<Mastodon>
  <IntlProvider>  ← 异步加载 locale，加载完成后渲染子组件
    <App>
      <ComponentA />
      <ComponentB />
    </App>
  </IntlProvider>
</Mastodon>
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

## 三、前后端 Locale 边界分析

### 3.1 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                        用户浏览器                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  <html lang="zh-CN">                                 │   │
│  │    ┌─────────────────────────────────────────────┐  │   │
│  │    │  React App (IntlProvider)                   │  │   │
│  │    │  - 从 <html lang> 读取 locale              │  │   │
│  │    │  - 动态加载 locales/*.json                  │  │   │
│  │    │  - 使用 react-intl 格式化                   │  │   │
│  │    └─────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTP 请求
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      Rails 后端                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  ApplicationController (include Localized)          │   │
│  │    - params[:lang]                                   │   │
│  │    - current_user.locale                             │   │
│  │    - Accept-Language Header                          │   │
│  │    - I18n.default_locale                             │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  渲染 HTML 时设置:                                    │   │
│  │  <html lang="<%= I18n.locale %>">                   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

### 3.2 前后端 Locale 源对比

| 维度 | 后端 Rails | 前端 React |
|------|-----------|------------|
| **Locale 来源** | 1. params[:lang]<br>2. current_user.locale<br>3. Accept-Language<br>4. default_locale | 仅从 `<html lang="...">` 读取 |
| **翻译文件** | `config/locales/*.yml` | `app/javascript/mastodon/locales/*.json` |
| **翻译引擎** | I18n gem | react-intl (基于 Intl API) |
| **变量语法** | `%{name}` | `{name}` |
| **复数形式** | YAML 层级结构 | ICU MessageFormat |
| **日期格式化** | `l()` 方法 + 配置文件 | `intl.formatDate()` (浏览器 Intl API) |

---

### 3.3 前后端职责边界

#### 3.3.1 后端负责

| 领域 | 具体内容 |
|------|----------|
| **Locale 决策** | 决定用户使用哪种语言 (优先级规则) |
| **HTML 渲染** | 设置 `<html lang="...">` 属性 |
| **Rails Views** | 服务端渲染页面的文案翻译 |
| **邮件模板** | 所有邮件的主题和内容 |
| **部分 API 错误** | 使用 `I18n.t()` 的错误消息 |
| **后台任务** | Sidekiq job 中的通知和邮件 |

#### 3.3.2 前端负责

| 领域 | 具体内容 |
|------|----------|
| **React 组件** | 所有 SPA 界面的文案 |
| **动态内容** | 用户交互、表单验证提示 |
| **时间格式化** | 相对时间、日期显示 |
| **数字格式化** | 数字、百分比等 |
| **客户端通知** | Web Push 通知的显示文案 |
| **emoji 选择器** | Emoji 分类名称 |

---

### 3.4 前后端同步机制

#### 3.4.1 语言同步

**后端 → 前端** 的语言传递：

1. **渲染 HTML 时设置 lang 属性**
   ```erb
   <!-- app/views/layouts/application.html.haml -->
   %html{ lang: I18n.locale }
   ```

2. **前端读取该属性**
   ```typescript
   const locale = document.querySelector<HTMLElement>('html')?.lang || 'en';
   ```

**单向数据流**:
```
用户请求 → 后端确定 locale → 渲染 <html lang="..."> → 前端读取
```

#### 3.4.2 翻译键不一致问题

**问题**: 前后端使用不同的翻译文件，键可能不对应。

**示例**:
```yaml
# 后端 config/locales/en.yml
en:
  accounts:
    self_follow_error: Following your own account is not allowed
```

```json
// 前端 app/javascript/mastodon/locales/en.json
{
  "account.block": "Block @{name}",
  // 没有 "accounts.self_follow_error" 这个键
}
```

**影响**:
- 后端通过 API 返回的翻译消息，前端无法再次翻译
- 前端显示时直接使用后端返回的字符串

#### 3.4.3 硬编码消息的问题

**文件**: `app/controllers/api/base_controller.rb`

```ruby
render json: { error: 'This method requires an authenticated user' }, status: 401
render json: { error: 'Your login is currently disabled' }, status: 403
```

**问题**:
- 这些消息始终是英文
- 前端收到后无法翻译
- 非英语用户看到英文错误消息

**可能的改进方案**:

```ruby
# 方案1: 返回错误码，前端翻译
render json: { error_code: 'authentication_required' }, status: 401

# 方案2: 使用 I18n 翻译
render json: { error: I18n.t('errors.authentication_required') }, status: 401
```

---

### 3.5 数据流转中的 Locale

#### 3.5.1 用户登录场景

```
1. 用户访问登录页面
   │
   ▼
2. 后端通过 Accept-Language 确定 locale (假设是 zh-CN)
   │
   ▼
3. 渲染页面: <html lang="zh-CN">
   │
   ▼
4. 前端读取 lang="zh-CN"，加载 zh-CN.json
   │
   ▼
5. 用户输入凭据，提交表单
   │
   ▼
6. 后端验证，发现用户设置的 locale 是 ja
   │
   ▼
7. 更新 I18n.locale = :ja，重定向到首页
   │
   ▼
8. 渲染首页: <html lang="ja">
   │
   ▼
9. 前端检测到 lang 变化，重新加载 ja.json (需刷新页面)
```

**注意**: 前端 locale 切换需要页面刷新，因为 `IntlProvider` 只在挂载时加载一次。

#### 3.5.2 API 请求场景

```
前端 React App
    │
    │ 1. 调用 API (无 locale 参数)
    ▼
后端 API Controller
    │
    │ 2. Localized concern 确定 locale
    │    - current_user.locale (已登录)
    │    - 或 Accept-Language Header
    │
    ▼
    │ 3. 执行业务逻辑
    │    - 使用 I18n.t() 翻译错误消息
    │    - 使用 l() 格式化日期
    │
    ▼
    │ 4. 返回响应
    │    - 已翻译的 error 消息
    │    - ISO 格式的时间戳 (前端自己格式化)
    │
    ▼
前端 React App
    │
    │ 5. 处理响应
    │    - 直接显示 error 消息 (后端已翻译)
    │    - 使用 intl.formatDate() 格式化时间戳
    │    - 使用 formatMessage() 翻译 UI 文案
```

---

## 四、关键文件索引

### 4.1 后端文件

| 文件路径 | 职责 |
|----------|------|
| `config/initializers/i18n.rb` | 可用语言列表、默认语言配置 |
| `config/locales/*.yml` | 后端翻译文件 (YAML 格式) |
| `app/controllers/concerns/localized.rb` | Locale 选择核心逻辑 |
| `app/controllers/application_controller.rb` | 引入 Localized 模块 |
| `app/mailers/application_mailer.rb` | 邮件发送的 locale 辅助方法 |
| `app/mailers/user_mailer.rb` | 用户邮件的 locale 处理 |
| `app/workers/web/push_notification_worker.rb` | 后台任务的 locale 处理 |
| `app/models/concerns/user/has_settings.rb` | 用户语言设置相关方法 |

### 4.2 前端文件

| 文件路径 | 职责 |
|----------|------|
| `app/javascript/mastodon/locales/*.json` | 前端翻译文件 (JSON 格式) |
| `app/javascript/mastodon/locales/global_locale.ts` | 全局 locale 存储 |
| `app/javascript/mastodon/locales/load_locale.ts` | 动态加载 locale bundle |
| `app/javascript/mastodon/locales/intl_provider.tsx` | React IntlProvider 封装 |
| `app/javascript/mastodon/locales/index.ts` | 导出统一接口 |
| `app/javascript/mastodon/utils/time.ts` | 时间格式化工具 |
| `app/javascript/mastodon/features/emoji/locale.ts` | Emoji 选择器 locale |

---

## 五、总结与建议

### 5.1 当前架构的优点

1. **清晰的分离**: 前后端各自管理自己的翻译，互不干扰
2. **灵活的后端优先级**: 支持 URL 参数、用户设置、浏览器偏好多层级
3. **异步加载**: 前端按需加载语言包，减少初始包大小
4. **标准技术**: 使用 I18n (Rails 标准) 和 react-intl (React 生态标准)

### 5.2 存在的问题

1. **翻译文件重复**: 前后端各有一套翻译，需要同步维护
2. **API 错误消息不一致**: 部分使用 I18n，部分硬编码英文
3. **无错误码机制**: 前端无法根据错误类型自己翻译
4. **前端语言切换需刷新**: `<html lang>` 变化后，IntlProvider 不会自动重新加载

### 5.3 可能的改进方向

#### 短期改进
- 统一 API 错误消息使用 `I18n.t()` 翻译
- 对于硬编码的通用错误，考虑添加到翻译文件

#### 中期改进
- 引入错误码机制，前端根据错误码翻译
- 或者确保所有 API 错误消息都经过后端翻译

#### 长期考虑
- 探索翻译文件共享机制 (单一数据源)
- 或建立同步流程 (如从 YAML 生成 JSON)
- 考虑前端支持运行时语言切换 (无需刷新)

---

*分析基于 Mastodon 代码库版本: 2024年*
