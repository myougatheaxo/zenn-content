---
title: "Claude Codeで国際化（i18n）を設計する：next-intl・型安全翻訳・RTL対応"
emoji: "🌍"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "nextjs", "i18n"]
published: true
---

## はじめに

i18nは「後付けで対応する」と大規模リファクタリングが必要になる。文字列のハードコード、日付フォーマット、RTL（右から左）レイアウト——Claude Codeに最初から正しい設計を生成させる。

---

## CLAUDE.mdにi18nルールを書く

```markdown
## 国際化（i18n）設計ルール

### 必須事項
- UIに表示する全文字列をi18nキーで管理（ハードコード禁止）
- 日付・時刻: Intl.DateTimeFormat または date-fns/locale を使う
- 数値・通貨: Intl.NumberFormat を使う
- 複数形: ICU Message Format使用（1件 vs 2件以上の区別）

### ファイル構成
- locales/en/common.json, locales/ja/common.json など
- 名前空間で分割（common, auth, dashboard等）
- キー名はcamelCase（例: 'welcomeMessage', 'errorNotFound'）

### 型安全
- TypeScriptでキーを型安全に（next-intlはuseTranslations()がキーを補完）
- 存在しないキーはコンパイルエラー

### RTL対応
- cssのmargin-left/rightではなくmargin-inline-start/endを使う
- アラビア語、ヘブライ語向けにdir="rtl"をhtmlタグに設定
```

---

## next-intl i18n設計の生成

```
Next.js App Router向けのi18n設計をnext-intlで生成してください。

言語: 日本語（デフォルト）+ 英語 + アラビア語（RTL）

要件：
- URLパスでロケール切り替え（/ja/、/en/、/ar/）
- 型安全な翻訳キー
- ICU Message Formatで複数形対応
- 日付・数値・通貨のローカライズ
- RTLレイアウト対応

生成ファイル:
- messages/ja.json, messages/en.json
- src/i18n/request.ts
```

---

## 生成されるi18n設計

```json
// messages/ja.json
{
  "common": {
    "loading": "読み込み中...",
    "error": "エラーが発生しました",
    "retry": "再試行",
    "cancel": "キャンセル",
    "save": "保存"
  },
  "auth": {
    "login": "ログイン",
    "logout": "ログアウト",
    "welcomeBack": "{name}さん、おかえりなさい",
    "errors": {
      "invalidCredentials": "メールアドレスまたはパスワードが正しくありません",
      "accountLocked": "アカウントがロックされています。{minutes}分後に再試行してください"
    }
  },
  "dashboard": {
    "orderCount": "{count, plural, =0 {注文なし} =1 {注文1件} other {注文{count}件}}",
    "lastLogin": "最終ログイン: {date}"
  }
}
```

```typescript
// src/i18n/request.ts
import { getRequestConfig } from 'next-intl/server';

export default getRequestConfig(async ({ locale }) => {
  const supportedLocales = ['ja', 'en', 'ar'];

  if (!supportedLocales.includes(locale)) {
    return { locale: 'ja', messages: (await import(`../../messages/ja.json`)).default };
  }

  return {
    messages: (await import(`../../messages/${locale}.json`)).default,
    timeZone: locale === 'ja' ? 'Asia/Tokyo' : 'UTC',
    now: new Date(),
  };
});
```

```typescript
// next.config.ts
import createNextIntlPlugin from 'next-intl/plugin';

const withNextIntl = createNextIntlPlugin('./src/i18n/request.ts');

export default withNextIntl({
  // ...
});
```

```typescript
// src/middleware.ts (ロケールルーティング)
import createMiddleware from 'next-intl/middleware';

export default createMiddleware({
  locales: ['ja', 'en', 'ar'],
  defaultLocale: 'ja',
  localeDetection: true, // Accept-Languageヘッダーで自動検出
});

export const config = {
  matcher: ['/((?!api|_next|.*\\..*).*)'],
};
```

---

## 型安全なコンポーネント

```typescript
// app/[locale]/dashboard/page.tsx
import { useTranslations, useFormatter } from 'next-intl';

export default function DashboardPage({ orderCount }: { orderCount: number }) {
  const t = useTranslations('dashboard');
  const format = useFormatter();

  return (
    <div>
      {/* 複数形対応 */}
      <p>{t('orderCount', { count: orderCount })}</p>

      {/* 日付のローカライズ */}
      <p>
        {t('lastLogin', {
          date: format.dateTime(new Date(), {
            dateStyle: 'long',
            timeStyle: 'short',
          }),
        })}
      </p>

      {/* 通貨のローカライズ */}
      <p>{format.number(1234567, { style: 'currency', currency: 'JPY' })}</p>
    </div>
  );
}
```

---

## RTL対応CSS

```css
/* RTL/LTR両対応のCSS */
.sidebar {
  /* margin-left の代わりに */
  margin-inline-start: 1rem;

  /* border-right の代わりに */
  border-inline-end: 1px solid #e5e7eb;

  /* text-align: left の代わりに */
  text-align: start;
}

/* Arabic/Hebrew向けにHTMLタグにdir="rtl"を設定 */
/* src/app/[locale]/layout.tsx */
```

```typescript
// src/app/[locale]/layout.tsx
const RTL_LOCALES = ['ar', 'he'];

export default function LocaleLayout({ children, params: { locale } }) {
  const dir = RTL_LOCALES.includes(locale) ? 'rtl' : 'ltr';

  return (
    <html lang={locale} dir={dir}>
      <body>{children}</body>
    </html>
  );
}
```

---

## まとめ

Claude Codeでi18nを設計する：

1. **CLAUDE.md** に全文字列キー管理・Intl API使用・ICU複数形を明記
2. **next-intl** で型安全なi18n（存在しないキーはコンパイルエラー）
3. **ICU Message Format** で複数形・変数を正確に処理
4. **margin-inline-start/end** でRTLを考慮したCSS

---

*i18n設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
