---
title: "Claude CodeでTDDを実践する：テスト先行開発のワークフロー"
emoji: "🧪"
type: "tech"
topics: ["claudecode", "tdd", "vitest", "pytest", "テスト"]
published: true
---

## はじめに

Claude Codeに「テスト付きでUserServiceを実装して」と頼むと、テストが実装に合わせて後付けで書かれることが多い。

これでは本来のTDD（テスト駆動開発）にならない。テストが要件を定義し、実装がテストを通すために書かれる流れを、Claude Codeと一緒に作るパターンを紹介する。

---

## TDDの3ステップをClaude Codeで実行する

### ステップ1: テストを先に書かせる

```
UserService.createUser()のテストを書いてください。
実装はまだ書かないでください。

要件：
- email（文字列）とname（文字列）を受け取る
- 作成したユーザーオブジェクトを返す（id付き）
- emailが無効な場合はValidationErrorをthrow
- emailが重複する場合はDuplicateErrorをthrow
- DBには直接繋がずmockを使う
```

**「実装はまだ書かないでください」が重要。**

### ステップ2: テストを通す実装を書かせる

```
このテストが全て通るUserService.createUser()を実装してください：

[前のステップで生成されたテストコード]
```

### ステップ3: テストが通ったままリファクタリング

```
テストは通っています。コードの可読性を上げるためにリファクタリングしてください。
動作は変えないでください。
```

---

## 「両方一緒に作って」がうまくいかない理由

「テスト付きでUserServiceを実装して」と頼むと：
- テストが実装に合わせて書かれる（本末転倒）
- エッジケースが漏れやすい
- 「テストが通るから正しい」という誤った安心感

要件 → テスト → 実装の順番が、思考の順序としても正しい。

---

## CLAUDE.mdにTDDルールを書く

```markdown
## テスト規約（TDD）
- テストは実装より先に書く
- テストファイル: `tests/` 以下（`src/` と同構成）
- テスト命名: `describe("クラス名", () => { describe("メソッド名", ...`
- パターン: Arrange-Act-Assert（1テストに1アサーション）
- 外部サービスは必ずmock（実際のDB・APIに繋がない）

## カバレッジ目標
- 全ての新機能: カバレッジ80%以上
- 重要サービス: 90%以上
- 未テストでのPRマージ不可
```

---

## テストファクトリーを生成させる

テストデータの準備を毎回書くのは辛い。ファクトリーを作らせる。

```
User型のテスト用ファクトリーを作成してください。

要件：
- 全ての必須フィールドにデフォルト値を設定
- 任意のフィールドをオーバーライドできる
- 呼び出すたびに一意のIDとemailを生成

型定義：
[User型の定義をここに貼る]
```

---

## カバレッジを上げる

既存コードのカバレッジが低い場合：

```
現在のカバレッジは68%です。カバーされていないブランチ：

[カバレッジレポートをここに貼る]

これらのブランチをカバーするテストを追加してください。
```

---

## /test-genスキルで自動生成

カスタムスキルがあれば：

```bash
/test-gen src/services/user.service.ts
```

生成されるテストの例：

```typescript
describe("UserService", () => {
  let service: UserService;
  let mockRepo: jest.Mocked<UserRepository>;

  beforeEach(() => {
    mockRepo = createMock<UserRepository>();
    service = new UserService(mockRepo);
  });

  describe("createUser", () => {
    it("有効なデータでユーザーを作成する", async () => {
      const input = { email: "test@example.com", name: "Test" };
      mockRepo.findByEmail.mockResolvedValue(null);
      mockRepo.create.mockResolvedValue({ id: "1", ...input });

      const result = await service.createUser(input);

      expect(result.id).toBeDefined();
      expect(result.email).toBe(input.email);
    });

    it("無効なemailでValidationErrorをthrow", async () => {
      await expect(
        service.createUser({ email: "invalid", name: "Test" })
      ).rejects.toThrow(ValidationError);
    });
  });
});
```

---

## CI統合

```yaml
# .github/workflows/test.yml
- name: テスト実行（カバレッジ付き）
  run: npm test -- --coverage

- name: カバレッジ閾値チェック
  run: npx vitest run --coverage.threshold.lines=80
```

カバレッジが80%を下回るとPRをブロックする。

---

## まとめ

Claude CodeでのTDD：

1. **テスト先行**（「実装はまだ書かないで」）
2. **要件をテストで表現**（ドキュメント代わりになる）
3. **実装はテストを通すための最小コード**
4. **リファクタリングは後から**

テストが先にあることで、Claude Codeが生成する実装の品質も上がる。

---

*`/test-gen` スキルは **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
