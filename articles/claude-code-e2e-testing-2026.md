---
title: "Claude CodeでE2Eテストを設計する：PlaywrightでUIテストを自動化する"
emoji: "🎭"
type: "tech"
topics: ["claudecode", "playwright", "testing", "typescript", "e2eテスト"]
published: true
---

## はじめに

単体テスト・統合テストが通っても本番でUIが壊れることがある。PlaywrightでE2Eテストを書き、Claude Codeにテストシナリオを生成させる。

---

## CLAUDE.mdにE2Eテストルールを書く

```markdown
## E2Eテストルール

### ツール
- Playwright（クロスブラウザ対応、headless/headful両対応）
- テストファイル: e2e/ ディレクトリ（srcとは別）
- 設定ファイル: playwright.config.ts

### テスト設計原則
- ユーザーが実際に行う操作を記述する（「送信ボタンをクリック」）
- 内部実装の詳細に依存しない（class名・内部IDは使わない）
- セレクター優先順位: data-testid > role > text > CSS

### テストデータ管理
- テスト用DBをリセットしてから実行（beforeAll）
- テストデータはfixture/factoryで管理（ハードコード禁止）
- 本番データを使わない

### CI設定
- PRごとにE2Eを実行
- 失敗時はスクリーンショット・動画を保存
- Chromium/Firefox/WebKitの3ブラウザで実行
- 実行時間の目標: 5分以内（並列実行を活用）

### 禁止
- sleep/waitForTimeout の使用（waitForSelector/waitForResponse を使う）
- テスト間のデータ共有（テストは独立して実行できること）
- 本番APIエンドポイントへのリクエスト（モックを使う）
```

---

## Playwright設定ファイルの生成

```
Playwright設定ファイルを生成してください。

要件：
- ブラウザ: Chromium, Firefox, WebKit
- 並列実行: ワーカー数はCPUコア数の半分
- テスト前にNext.jsアプリを起動（dev server）
- 失敗時: スクリーンショット + 動画を保存
- タイムアウト: アクション10秒、テスト1分
- ベースURL: http://localhost:3000

保存先: playwright.config.ts
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 2 : undefined,
  reporter: [['html', { open: 'never' }], ['list']],

  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    actionTimeout: 10_000,
  },

  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
  ],

  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

---

## ログインフローのE2Eテスト生成

```
ユーザーログインのE2Eテストを生成してください。

テストシナリオ:
1. 正常ログイン: メール + パスワードで成功 → ダッシュボードにリダイレクト
2. 失敗ログイン: 間違ったパスワード → エラーメッセージ表示
3. バリデーション: メールアドレス形式が無効 → フォームエラー
4. パスワードリセット: 「パスワードを忘れた」フロー

セレクター: data-testid属性を使う
ログイン後の状態確認: roleベースのアサーション

保存先: e2e/auth/login.spec.ts
```

```typescript
// e2e/auth/login.spec.ts
import { test, expect } from '@playwright/test';

test.describe('ログインフロー', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
  });

  test('正しい認証情報でログインできる', async ({ page }) => {
    await page.getByTestId('email-input').fill('user@example.com');
    await page.getByTestId('password-input').fill('correct-password');
    await page.getByRole('button', { name: 'ログイン' }).click();

    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByRole('heading', { name: 'ダッシュボード' })).toBeVisible();
  });

  test('間違ったパスワードでエラーが表示される', async ({ page }) => {
    await page.getByTestId('email-input').fill('user@example.com');
    await page.getByTestId('password-input').fill('wrong-password');
    await page.getByRole('button', { name: 'ログイン' }).click();

    await expect(page.getByTestId('error-message')).toHaveText(
      'メールアドレスまたはパスワードが間違っています'
    );
    await expect(page).toHaveURL('/login');
  });

  test('無効なメールアドレス形式でバリデーションエラー', async ({ page }) => {
    await page.getByTestId('email-input').fill('invalid-email');
    await page.getByTestId('password-input').fill('any-password');
    await page.getByRole('button', { name: 'ログイン' }).click();

    await expect(page.getByTestId('email-error')).toBeVisible();
  });
});
```

---

## Page Object Modelの生成

```
ログインページのPage Object Modelを生成してください。

要件：
- テストコードからセレクターを隠蔽
- Playwright Page クラスを継承しない（composition パターン）
- 全ての操作をメソッドとして定義
- アサーションメソッドも含める

保存先: e2e/pages/LoginPage.ts
```

```typescript
// e2e/pages/LoginPage.ts
import { Page, Locator, expect } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByTestId('email-input');
    this.passwordInput = page.getByTestId('password-input');
    this.submitButton = page.getByRole('button', { name: 'ログイン' });
    this.errorMessage = page.getByTestId('error-message');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async assertError(message: string) {
    await expect(this.errorMessage).toHaveText(message);
  }
}
```

---

## まとめ

Claude CodeでE2Eテストを設計する：

1. **CLAUDE.md** にセレクター優先順位・テストデータ管理・禁止事項を明記
2. **Playwright設定** を3ブラウザ対応・並列実行で生成
3. **ユーザーシナリオ** ベースのテストを生成（実装詳細に依存しない）
4. **Page Object Model** でセレクターを一元管理

---

*E2Eテストの設計問題を自動検出するスキルは **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
