---
title: "Claude Codeでデータエクスポートを設計する：CSV・Excel生成のベストプラクティス"
emoji: "📊"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "csv", "データ処理"]
published: true
---

## はじめに

「管理画面からCSVをダウンロードしたい」——実装してみると大量データでメモリを食い尽くしたり、文字化けが起きたり、タイムアウトしたりする。Claude Codeに正しいデータエクスポート設計を生成させる。

---

## CLAUDE.mdにデータエクスポートルールを書く

```markdown
## データエクスポートルール

### 大量データの処理
- 10万件以上のエクスポートはバックグラウンドジョブで処理（同期処理禁止）
- 1万件未満: 同期で処理してOK（タイムアウトに注意）
- ストリーミング出力を使う（全データをメモリに乗せない）

### CSV仕様
- 文字コード: UTF-8 BOM付き（Excelで文字化けしない）
- 改行コード: CRLF（Windowsのデフォルト）
- クォート: 全フィールドをダブルクォートで囲む
- ヘッダー行: 必ず含める（日本語ヘッダー対応）
- 日付フォーマット: YYYY-MM-DD HH:mm:ss（タイムゾーン注意）

### セキュリティ
- CSVインジェクション防止: =, +, -, @ で始まる値はシングルクォートを先頭に追加
- エクスポートは認証必須
- ダウンロードURLは一時的（15分で失効）

### パフォーマンス
- バッチサイズ: 1000件ずつDBから取得（全件SELECT禁止）
- 大容量ファイルは圧縮（gzip）してから配信
```

---

## ストリーミングCSVエクスポートの生成

```
大量データをストリーミングでCSV出力するAPIを生成してください。

要件：
- エンドポイント: GET /api/exports/users
- 最大10万件のユーザーデータをCSVで出力
- メモリに全件乗せない（cursor-basedでバッチ取得 + Transform streamで出力）
- UTF-8 BOM付き
- CSVインジェクション防止
- 認証必須

生成するファイル:
- src/utils/csvBuilder.ts（CSV生成ユーティリティ）
- src/routes/export.ts（エクスポートエンドポイント）
```

---

## 生成されるストリーミングCSV

```typescript
// src/utils/csvBuilder.ts
export function sanitizeCsvValue(value: unknown): string {
  const str = value === null || value === undefined ? '' : String(value);

  // CSVインジェクション防止: 危険な文字で始まる場合はシングルクォートを先頭に
  const dangerous = ['=', '+', '-', '@', '\t', '\r'];
  const sanitized = dangerous.some(c => str.startsWith(c))
    ? `'${str}`
    : str;

  // ダブルクォートでエスケープ
  return `"${sanitized.replace(/"/g, '""')}"`;
}

export function buildCsvRow(values: unknown[]): string {
  return values.map(sanitizeCsvValue).join(',') + '\r\n';
}

// UTF-8 BOM
export const CSV_BOM = '\uFEFF';
```

```typescript
// src/routes/export.ts
import { Transform } from 'stream';
import { buildCsvRow, CSV_BOM } from '../utils/csvBuilder';

router.get('/exports/users', authenticate, async (req, res) => {
  res.setHeader('Content-Type', 'text/csv; charset=utf-8');
  res.setHeader('Content-Disposition', 'attachment; filename="users.csv"');

  // BOM
  res.write(CSV_BOM);

  // ヘッダー行
  res.write(buildCsvRow(['ID', '名前', 'メールアドレス', '登録日', 'ロール']));

  // cursor-basedでバッチ取得してストリーミング出力
  let cursor: string | undefined;
  const batchSize = 1000;

  while (true) {
    const users = await prisma.user.findMany({
      take: batchSize,
      ...(cursor && { skip: 1, cursor: { id: cursor } }),
      orderBy: { id: 'asc' },
      select: { id: true, name: true, email: true, createdAt: true, role: true },
    });

    if (users.length === 0) break;

    for (const user of users) {
      if (!res.writable) break;
      res.write(buildCsvRow([
        user.id,
        user.name,
        user.email,
        user.createdAt.toLocaleString('ja-JP', { timeZone: 'Asia/Tokyo' }),
        user.role,
      ]));
    }

    cursor = users[users.length - 1].id;
    if (users.length < batchSize) break;
  }

  res.end();
});
```

---

## 大量データはバックグラウンドジョブで処理

```
10万件以上のエクスポートをバックグラウンドジョブで処理する仕組みを生成してください。

フロー:
1. POST /api/exports → ジョブをBullMQに登録 → jobIdを返す
2. ワーカーがCSVを生成してS3/ローカルに保存
3. GET /api/exports/:jobId → { status, downloadUrl? } を返す
4. ダウンロードURLは15分で失効（署名付きURL）

ジョブ進捗: job.updateProgress(処理済み件数 / 総件数 * 100)
```

---

## Excelファイルの生成

```
exceljs を使ったExcel（.xlsx）ファイルの生成を実装してください。

要件：
- ヘッダー行: 太字・背景色（青）
- 列幅: 内容に合わせて自動調整
- セルのスタイル: 日付列はYYYY/MM/DD形式
- フリーズペイン: 1行目（ヘッダー）を固定
- データ: ユーザーリスト（最大1000件）
- Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet

保存先: src/utils/excelBuilder.ts
```

```typescript
// src/utils/excelBuilder.ts（生成例）
import ExcelJS from 'exceljs';

export async function generateUsersExcel(users: User[]): Promise<Buffer> {
  const workbook = new ExcelJS.Workbook();
  const sheet = workbook.addWorksheet('ユーザー一覧');

  // ヘッダー設定
  sheet.columns = [
    { header: 'ID', key: 'id', width: 36 },
    { header: '名前', key: 'name', width: 20 },
    { header: 'メールアドレス', key: 'email', width: 30 },
    { header: '登録日', key: 'createdAt', width: 20, style: { numFmt: 'YYYY/MM/DD' } },
  ];

  // ヘッダー行スタイル
  const headerRow = sheet.getRow(1);
  headerRow.font = { bold: true, color: { argb: 'FFFFFFFF' } };
  headerRow.fill = { type: 'pattern', pattern: 'solid', fgColor: { argb: 'FF0070C0' } };

  // 1行目固定
  sheet.views = [{ state: 'frozen', ySplit: 1 }];

  // データ追加
  users.forEach(user => sheet.addRow(user));

  return workbook.xlsx.writeBuffer() as Promise<Buffer>;
}
```

---

## まとめ

Claude Codeでデータエクスポートを設計する：

1. **CLAUDE.md** に件数別の処理方式・CSV仕様・セキュリティルールを明記
2. **ストリーミング出力** でメモリを節約（全件SELECTしない）
3. **CSVインジェクション防止** でセキュリティを確保
4. **大量データはBullMQ** でバックグラウンドジョブに移す

---

*データエクスポートのセキュリティとパフォーマンス問題を自動検出するスキルは **Security Pack（¥1,480）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
