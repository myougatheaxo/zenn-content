---
title: "Claude CodeでPDF生成を設計する：Puppeteer・HTML/CSSテンプレート・S3保存・非同期処理"
emoji: "📄"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "aws", "puppeteer"]
published: true
published_at: "2026-03-14 08:00"
---

## はじめに

請求書・レポート・証明書のPDF生成——HTMLテンプレートをPuppeteerでPDFに変換し、S3に保存するパターンをClaude Codeに設計させる。重い処理はバックグラウンドジョブで非同期化。

---

## CLAUDE.mdにPDF生成設計ルールを書く

```markdown
## PDF生成設計ルール

### 技術選定
- Puppeteer（Chrome headless）: HTML/CSS完全対応、日本語フォント対応
- Handlebars: HTMLテンプレートエンジン
- S3保存: 生成後即URLを返す（ファイルはサーバーに残さない）

### 非同期処理
- PDF生成はBullMQジョブキューで非同期実行
- APIは即座にjobIdを返す → クライアントはポーリング or WebhookでURL受取
- タイムアウト: 60秒（大きなPDFでも十分な余裕）

### セキュリティ
- HTMLテンプレートのインジェクション対策: Handlebarsの自動エスケープ
- 外部URL埋め込み禁止: --no-sandbox, --disable-web-security 禁止
- ユーザーIDでS3パスを分離: /pdfs/{userId}/{jobId}.pdf
```

---

## PDF生成システムの生成

```
Puppeteer + BullMQによるPDF生成システムを設計してください。

要件：
- HTMLテンプレート → PDF変換
- 非同期ジョブキュー
- S3保存 + 署名付きURL
- 日本語フォント対応

生成ファイル: src/pdf/
```

---

## 生成されるPDF生成実装

```typescript
// src/pdf/templates/invoice.hbs — 請求書HTMLテンプレート（Handlebars）

/*
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <style>
    @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+JP&display=swap');
    body { font-family: 'Noto Sans JP', sans-serif; padding: 40px; }
    .header { display: flex; justify-content: space-between; }
    .invoice-number { font-size: 24px; font-weight: bold; }
    table { width: 100%; border-collapse: collapse; margin-top: 20px; }
    th, td { border: 1px solid #ccc; padding: 8px 12px; text-align: left; }
    th { background: #f5f5f5; }
    .total { font-size: 20px; font-weight: bold; text-align: right; margin-top: 20px; }
  </style>
</head>
<body>
  <div class="header">
    <div>
      <div class="invoice-number">請求書 #{{invoiceNumber}}</div>
      <div>発行日: {{issueDate}}</div>
      <div>支払期限: {{dueDate}}</div>
    </div>
    <div>
      <strong>請求先:</strong><br>
      {{customer.company}}<br>
      {{customer.name}} 様
    </div>
  </div>

  <table>
    <thead>
      <tr><th>商品名</th><th>数量</th><th>単価</th><th>小計</th></tr>
    </thead>
    <tbody>
      {{#each items}}
      <tr>
        <td>{{this.name}}</td>
        <td>{{this.quantity}}</td>
        <td>¥{{formatNumber this.unitPrice}}</td>
        <td>¥{{formatNumber this.subtotal}}</td>
      </tr>
      {{/each}}
    </tbody>
  </table>

  <div class="total">
    合計（税込）: ¥{{formatNumber totalAmount}}
  </div>
</body>
</html>
*/
```

```typescript
// src/pdf/generator.ts — PDF生成コア

import puppeteer, { Browser } from 'puppeteer';
import Handlebars from 'handlebars';
import { readFile } from 'fs/promises';
import path from 'path';

// 数値フォーマットヘルパー
Handlebars.registerHelper('formatNumber', (value: number) =>
  value.toLocaleString('ja-JP')
);

// Puppeteerインスタンスは使い回す（起動コスト削減）
let browserInstance: Browser | null = null;
let browserUseCount = 0;
const MAX_BROWSER_USES = 100; // 100回使用後に再起動（メモリリーク対策）

async function getBrowser(): Promise<Browser> {
  if (!browserInstance || !browserInstance.connected || browserUseCount >= MAX_BROWSER_USES) {
    if (browserInstance) await browserInstance.close();
    browserInstance = await puppeteer.launch({
      headless: true,
      args: [
        '--no-sandbox',             // コンテナ内では必要
        '--disable-setuid-sandbox',
        '--disable-dev-shm-usage',  // /dev/shm が小さいDockerで必要
        '--disable-gpu',
        '--font-render-hinting=none',
      ],
    });
    browserUseCount = 0;
  }
  browserUseCount++;
  return browserInstance;
}

export async function generatePDF(
  templateName: string,
  data: Record<string, unknown>
): Promise<Buffer> {
  // テンプレート読み込み・コンパイル
  const templatePath = path.join(__dirname, 'templates', `${templateName}.hbs`);
  const templateSource = await readFile(templatePath, 'utf-8');
  const template = Handlebars.compile(templateSource);
  const html = template(data);

  const browser = await getBrowser();
  const page = await browser.newPage();

  try {
    // HTMLを設定（外部リソース読み込みを待機）
    await page.setContent(html, {
      waitUntil: 'networkidle0', // フォント等の読み込み完了まで待機
    });

    // 日本語フォントの描画完了を待機
    await page.evaluate(() => document.fonts.ready);

    const pdfBuffer = await page.pdf({
      format: 'A4',
      margin: { top: '20mm', right: '15mm', bottom: '20mm', left: '15mm' },
      printBackground: true,
    });

    return Buffer.from(pdfBuffer);
  } finally {
    await page.close(); // ページは必ずクローズ（ブラウザは使い回す）
  }
}
```

```typescript
// src/pdf/jobs/pdfJob.ts — BullMQジョブ定義

import { Queue, Worker } from 'bullmq';
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

export const pdfQueue = new Queue('pdf-generation', {
  connection: redis,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 5000 },
    removeOnComplete: 100,
    removeOnFail: 1000,
  },
});

const s3 = new S3Client({ region: process.env.AWS_REGION });

// ワーカー
new Worker('pdf-generation', async (job) => {
  const { jobId, userId, templateName, data } = job.data;

  logger.info({ jobId }, 'Starting PDF generation');

  // PDF生成
  const pdfBuffer = await generatePDF(templateName, data);

  // S3にアップロード
  const s3Key = `pdfs/${userId}/${jobId}.pdf`;
  await s3.send(new PutObjectCommand({
    Bucket: process.env.PDF_BUCKET!,
    Key: s3Key,
    Body: pdfBuffer,
    ContentType: 'application/pdf',
    ContentDisposition: `attachment; filename="${templateName}-${jobId}.pdf"`,
    ServerSideEncryption: 'AES256',
  }));

  // 署名付きURL発行（7日間有効）
  const signedUrl = await getSignedUrl(
    s3,
    new GetObjectCommand({ Bucket: process.env.PDF_BUCKET!, Key: s3Key }),
    { expiresIn: 7 * 24 * 3600 }
  );

  // DBにURLを保存
  await prisma.pdfJob.update({
    where: { id: jobId },
    data: { status: 'COMPLETED', pdfUrl: signedUrl, completedAt: new Date() },
  });

  logger.info({ jobId, s3Key }, 'PDF generation completed');
  return { pdfUrl: signedUrl };
}, { connection: redis, concurrency: 3 }); // 同時3ジョブまで

// API エンドポイント
router.post('/invoices/:id/pdf', requireAuth, async (req, res) => {
  const invoice = await prisma.invoice.findUnique({ where: { id: req.params.id } });
  if (!invoice || invoice.userId !== req.user.id) return res.sendStatus(404);

  const jobId = crypto.randomUUID();
  await prisma.pdfJob.create({
    data: { id: jobId, userId: req.user.id, status: 'PENDING', invoiceId: invoice.id },
  });

  await pdfQueue.add('generate', {
    jobId,
    userId: req.user.id,
    templateName: 'invoice',
    data: await buildInvoiceData(invoice),
  });

  res.status(202).json({ jobId, statusUrl: `/api/pdf-jobs/${jobId}` });
});

// ステータスポーリングエンドポイント
router.get('/pdf-jobs/:jobId', requireAuth, async (req, res) => {
  const job = await prisma.pdfJob.findFirst({
    where: { id: req.params.jobId, userId: req.user.id },
  });
  if (!job) return res.sendStatus(404);
  res.json({ status: job.status, pdfUrl: job.pdfUrl });
});
```

---

## まとめ

Claude CodeでPDF生成を設計する：

1. **CLAUDE.md** にPuppeteer+Handlebars・BullMQ非同期・S3保存・60秒タイムアウトを明記
2. **ブラウザ使い回し** でPuppeteer起動コストをゼロに——100回使用後に自動再起動でメモリリーク対策
3. **BullMQで非同期化** APIは202を即返しjobIdを渡す——PDF生成が終わったらDBにURLを保存
4. **S3署名付きURL（7日間）** でPDFをプライベートに保存しつつ一時的に共有可能なURLを提供

---

*PDF設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
