---
title: "Claude CodeでファイルストレージをS3で設計する：署名付きURL・ウイルススキャン・CDN配信"
emoji: "📁"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "aws", "s3"]
published: true
---

## はじめに

ユーザーがアップロードしたファイルをサーバー経由で受け取るとメモリを使い切る。S3のPresigned URLを使ってクライアントから直接アップロードさせ、ウイルススキャン後にCDNで配信する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにファイルストレージ設計ルールを書く

```markdown
## ファイルストレージ設計ルール

### アップロード方式
- クライアント → S3直接アップロード（Presigned URL）
- サーバーをファイル転送経路に入れない（メモリ節約）
- アップロード完了後にWebhook/SQSでサーバーに通知

### セキュリティ
- アップロード前にファイルサイズ制限をPresigned URLに埋め込む
- 全ファイルにClamAVでウイルススキャン（アップロード後）
- スキャン済みフラグが立つまでパブリックアクセスを遮断
- ContentType検証（拡張子偽装防止）

### 配信
- S3バケットは非公開（CloudFront経由のみ）
- CloudFront署名付きURLまたは署名付きCookieでアクセス制御
- キャッシュTTL: 画像は1時間、ドキュメントは5分

### ストレージ管理
- tmpフォルダは7日でExpiry設定（未完了アップロードの掃除）
- ユーザー削除時は関連ファイルを非同期削除
- バージョニング有効化（誤削除対策）
```

---

## ファイルアップロードシステムの生成

```
S3を使ったファイルアップロードシステムを設計してください。

要件：
- Presigned URLでクライアントから直接S3にアップロード
- アップロード完了通知（S3イベント → SQS）
- ClamAVウイルススキャン
- CloudFront署名付きURLで配信

生成ファイル: src/storage/
```

---

## 生成されるファイルストレージ実装

```typescript
// src/storage/s3Service.ts
import { S3Client, PutObjectCommand, GetObjectCommand, DeleteObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { CloudFrontClient, CreateInvalidationCommand } from '@aws-sdk/client-cloudfront';

const s3 = new S3Client({ region: process.env.AWS_REGION ?? 'ap-northeast-1' });
const BUCKET = process.env.S3_BUCKET_NAME!;
const UPLOAD_BUCKET = process.env.S3_UPLOAD_BUCKET_NAME!; // 未スキャンファイル用

interface UploadRequest {
  userId: string;
  fileName: string;
  contentType: string;
  maxSizeBytes?: number;
}

interface PresignedUploadResult {
  uploadUrl: string;   // クライアントがPUTするURL
  fileKey: string;     // S3のキー
  expiresAt: Date;
}

// アップロード用Presigned URLを生成
export async function createPresignedUploadUrl(
  request: UploadRequest
): Promise<PresignedUploadResult> {
  const { userId, fileName, contentType, maxSizeBytes = 10 * 1024 * 1024 } = request;

  // ファイルタイプを検証（拡張子偽装防止）
  const allowedTypes = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf'];
  if (!allowedTypes.includes(contentType)) {
    throw new ValidationError(`File type ${contentType} is not allowed`);
  }

  // ユニークなキーを生成（ユーザーIDサブフォルダで分離）
  const extension = fileName.split('.').pop();
  const fileKey = `uploads/${userId}/${crypto.randomUUID()}.${extension}`;

  const command = new PutObjectCommand({
    Bucket: UPLOAD_BUCKET,
    Key: fileKey,
    ContentType: contentType,
    ContentLength: maxSizeBytes, // サイズ上限をPresigned URLに埋め込む
    Metadata: {
      'uploaded-by': userId,
      'original-name': encodeURIComponent(fileName),
      'scan-status': 'pending',
    },
  });

  // 15分間有効なPresigned URLを生成
  const uploadUrl = await getSignedUrl(s3, command, { expiresIn: 900 });

  // DBにアップロード予約を記録
  await prisma.fileUpload.create({
    data: {
      fileKey,
      userId,
      fileName,
      contentType,
      status: 'pending',
      expiresAt: new Date(Date.now() + 900_000),
    },
  });

  return {
    uploadUrl,
    fileKey,
    expiresAt: new Date(Date.now() + 900_000),
  };
}
```

```typescript
// src/storage/virusScanner.ts
// S3イベント → SQS → Lambda/Worker でウイルススキャン

import { exec } from 'child_process';
import { promisify } from 'util';
const execAsync = promisify(exec);

export async function scanFile(fileKey: string): Promise<'clean' | 'infected'> {
  // S3からダウンロード
  const command = new GetObjectCommand({ Bucket: UPLOAD_BUCKET, Key: fileKey });
  const response = await s3.send(command);
  const buffer = Buffer.from(await response.Body!.transformToByteArray());

  // 一時ファイルに書き込み
  const tmpPath = `/tmp/${crypto.randomUUID()}`;
  await fs.writeFile(tmpPath, buffer);

  try {
    // ClamAVでスキャン
    await execAsync(`clamscan --no-summary ${tmpPath}`);
    return 'clean';
  } catch (err: unknown) {
    // clamscan exit code 1 = infected
    if ((err as { code: number }).code === 1) {
      return 'infected';
    }
    throw err; // その他のエラーは再スロー
  } finally {
    await fs.unlink(tmpPath).catch(() => {}); // 一時ファイル削除
  }
}

// スキャン完了後の処理
export async function handleScanResult(fileKey: string, userId: string): Promise<void> {
  const result = await scanFile(fileKey);

  if (result === 'infected') {
    // 感染ファイルはS3から削除
    await s3.send(new DeleteObjectCommand({ Bucket: UPLOAD_BUCKET, Key: fileKey }));

    await prisma.fileUpload.update({
      where: { fileKey },
      data: { status: 'rejected', rejectedReason: 'virus_detected' },
    });

    logger.warn({ fileKey, userId }, 'Virus detected in uploaded file');
    return;
  }

  // クリーン: 本番バケットにコピー
  const productionKey = fileKey.replace('uploads/', 'files/');
  await s3.send(new CopyObjectCommand({
    CopySource: `${UPLOAD_BUCKET}/${fileKey}`,
    Bucket: BUCKET,
    Key: productionKey,
    MetadataDirective: 'REPLACE',
    Metadata: { 'scan-status': 'clean' },
  }));

  // アップロードバケットから削除
  await s3.send(new DeleteObjectCommand({ Bucket: UPLOAD_BUCKET, Key: fileKey }));

  await prisma.fileUpload.update({
    where: { fileKey },
    data: { status: 'ready', productionKey },
  });
}
```

---

## CloudFront署名付きURLでセキュア配信

```typescript
// src/storage/cloudFrontService.ts
import { getSignedUrl as getCFSignedUrl } from '@aws-sdk/cloudfront-signer';

export async function getSecureFileUrl(
  productionKey: string,
  userId: string,
  expiresInSeconds = 3600
): Promise<string> {
  // アクセス権限確認
  const upload = await prisma.fileUpload.findFirst({
    where: { productionKey, userId, status: 'ready' },
  });

  if (!upload) {
    throw new ForbiddenError('File not found or access denied');
  }

  // CloudFront署名付きURL生成（キーペアで署名）
  const cfUrl = `https://${process.env.CF_DOMAIN}/${productionKey}`;
  const expiresAt = Math.floor(Date.now() / 1000) + expiresInSeconds;

  return getCFSignedUrl({
    url: cfUrl,
    keyPairId: process.env.CF_KEY_PAIR_ID!,
    dateLessThan: new Date(expiresAt * 1000).toISOString(),
    privateKey: process.env.CF_PRIVATE_KEY!,
  });
}
```

---

## まとめ

Claude CodeでS3ファイルストレージを設計する：

1. **CLAUDE.md** にPresigned URL方式・ウイルススキャン必須・CloudFront配信を明記
2. **Presigned URL** でクライアントからS3に直接アップロード（サーバーのメモリを使わない）
3. **ClamAV** でウイルススキャン後にクリーンなファイルのみ本番バケットに移動
4. **CloudFront署名付きURL** でS3バケットを非公開のままアクセス制御

---

*ファイルストレージ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
