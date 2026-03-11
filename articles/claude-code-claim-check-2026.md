---
title: "Claude CodeでClaim-Checkパターンを設計する：大きなメッセージの分離保存・参照渡し・ストレージ最適化"
emoji: "🎫"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-17 17:00"
---

## はじめに

「メッセージブローカーに10MBのPDFを乗せたらキューが詰まった」——Claim-Checkパターンで大きなペイロードをS3等のストレージに保存し、メッセージには参照（クレームチェック）だけを乗せる設計をClaude Codeに生成させる。

---

## CLAUDE.mdにClaim-Check設計ルールを書く

```markdown
## Claim-Checkパターン設計ルール

### 分離基準
- メッセージペイロードが1KB超え: Claim-Checkを使用
- テキスト/JSONは直接送信可（小さい場合）
- 画像・PDF・動画・大きなJSONは必ずClaim-Check

### ストレージ
- 保存先: S3互換ストレージ（MinIO/AWS S3）
- キー形式: `claim-check/{serviceName}/{date}/{claimId}.{ext}`
- TTL: 72時間（処理後にConsumerが削除）

### セキュリティ
- 事前署名URLで一時的なアクセス許可（15分）
- メッセージには claimId のみ（URLは含めない）
- ConsumerがclaimIdを使ってURLを取得
```

---

## Claim-Check実装の生成

```
Claim-Checkパターンを設計してください。

要件：
- ペイロードのS3自動アップロード
- claimId経由での取得
- 事前署名URL
- 処理後の自動クリーンアップ

生成ファイル: src/messaging/claimCheck/
```

---

## 生成されるClaim-Check実装

```typescript
// src/messaging/claimCheck/claimCheckStore.ts — Claim-Checkストア

import { S3Client, PutObjectCommand, GetObjectCommand, DeleteObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

export interface ClaimCheckOptions {
  ttlHours?: number;        // ストレージTTL（デフォルト: 72時間）
  presignedUrlTtlSec?: number; // 事前署名URLの有効期限（デフォルト: 900秒 = 15分）
}

export class ClaimCheckStore {
  private readonly s3 = new S3Client({ region: process.env.AWS_REGION });
  private readonly bucket = process.env.CLAIM_CHECK_BUCKET!;

  constructor(private readonly opts: ClaimCheckOptions = {}) {}

  // 大きなペイロードをストレージに保存してclaimIdを返す
  async store(
    payload: Buffer | string | object,
    options: {
      serviceName: string;
      contentType?: string;
      extension?: string;
    }
  ): Promise<{ claimId: string; sizeBytes: number }> {
    const claimId = ulid();
    const ext = options.extension ?? 'json';
    const date = new Date().toISOString().slice(0, 10);
    const key = `claim-check/${options.serviceName}/${date}/${claimId}.${ext}`;

    const body =
      payload instanceof Buffer ? payload :
      typeof payload === 'string' ? Buffer.from(payload, 'utf-8') :
      Buffer.from(JSON.stringify(payload), 'utf-8');

    await this.s3.send(new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      Body: body,
      ContentType: options.contentType ?? 'application/json',
      // S3 Object Expirationで自動削除
      Expires: new Date(Date.now() + (this.opts.ttlHours ?? 72) * 3600_000),
    }));

    // claimId → S3キーのマッピングをRedisに保存
    await redis.set(
      `claim-check:key:${claimId}`,
      key,
      { EX: (this.opts.ttlHours ?? 72) * 3600 }
    );

    logger.debug({ claimId, key, sizeBytes: body.length }, 'Claim check stored');

    return { claimId, sizeBytes: body.length };
  }

  // claimIdから事前署名URLを取得
  async getSignedUrl(claimId: string): Promise<string> {
    const key = await redis.get(`claim-check:key:${claimId}`);
    if (!key) throw new NotFoundError(`Claim check ${claimId} not found or expired`);

    return getSignedUrl(
      this.s3,
      new GetObjectCommand({ Bucket: this.bucket, Key: key }),
      { expiresIn: this.opts.presignedUrlTtlSec ?? 900 }
    );
  }

  // ペイロードを直接取得（Consumer側）
  async retrieve<T = unknown>(claimId: string): Promise<T> {
    const key = await redis.get(`claim-check:key:${claimId}`);
    if (!key) throw new NotFoundError(`Claim check ${claimId} not found or expired`);

    const response = await this.s3.send(new GetObjectCommand({ Bucket: this.bucket, Key: key }));
    const body = await response.Body?.transformToString('utf-8');

    if (!body) throw new Error(`Empty claim check body for ${claimId}`);

    return JSON.parse(body) as T;
  }

  // 処理後にストレージから削除（クリーンアップ）
  async delete(claimId: string): Promise<void> {
    const key = await redis.get(`claim-check:key:${claimId}`);
    if (!key) return;

    await this.s3.send(new DeleteObjectCommand({ Bucket: this.bucket, Key: key }));
    await redis.del(`claim-check:key:${claimId}`);

    logger.debug({ claimId }, 'Claim check deleted');
  }
}
```

```typescript
// src/messaging/claimCheck/claimCheckPublisher.ts — Publisher側

const store = new ClaimCheckStore({ ttlHours: 72, presignedUrlTtlSec: 900 });

export class DocumentProcessingPublisher {
  // 大きなドキュメントをClaim-Checkで送信
  async publishDocumentForProcessing(document: {
    userId: string;
    fileName: string;
    content: Buffer;
    contentType: string;
  }): Promise<void> {
    const extension = document.fileName.split('.').pop() ?? 'bin';

    // ペイロードをストレージに保存
    const { claimId, sizeBytes } = await store.store(document.content, {
      serviceName: 'document-processor',
      contentType: document.contentType,
      extension,
    });

    // キューにはclaimIdのみ送信（小さなメッセージ）
    await redis.xAdd('queue:document-processing', '*', {
      messageId: ulid(),
      userId: document.userId,
      fileName: document.fileName,
      contentType: document.contentType,
      claimId,            // ← 実際のデータへの参照
      originalSizeBytes: sizeBytes.toString(),
      publishedAt: new Date().toISOString(),
    });

    logger.info({ claimId, sizeBytes, fileName: document.fileName }, 'Document queued via claim check');
  }
}

// src/messaging/claimCheck/claimCheckConsumer.ts — Consumer側

export class DocumentProcessingConsumer {
  private readonly store = new ClaimCheckStore();

  async handleMessage(message: {
    userId: string;
    fileName: string;
    contentType: string;
    claimId: string;
  }): Promise<void> {
    // claimIdからペイロードを取得
    const content = await this.store.retrieve<Buffer>(message.claimId);

    logger.info({ claimId: message.claimId, fileName: message.fileName }, 'Claim check retrieved');

    // 実際の処理
    await this.processDocument({
      userId: message.userId,
      fileName: message.fileName,
      content,
      contentType: message.contentType,
    });

    // 処理完了後にストレージをクリーンアップ
    await this.store.delete(message.claimId);
    logger.info({ claimId: message.claimId }, 'Claim check cleaned up');
  }

  private async processDocument(doc: {
    userId: string;
    fileName: string;
    content: Buffer;
    contentType: string;
  }): Promise<void> {
    // OCR、変換、解析など
  }
}

// サイズに応じて自動でClaim-Checkを使うユーティリティ
export async function publishWithAutoClaimCheck<T extends object>(
  queueKey: string,
  payload: T,
  threshold = 1024 // 1KB超えたらClaim-Check
): Promise<void> {
  const serialized = JSON.stringify(payload);

  if (Buffer.byteLength(serialized) > threshold) {
    const { claimId } = await store.store(payload, { serviceName: 'auto' });
    await redis.xAdd(queueKey, '*', { claimId, useClaimCheck: 'true' });
  } else {
    await redis.xAdd(queueKey, '*', { payload: serialized, useClaimCheck: 'false' });
  }
}
```

---

## まとめ

Claude CodeでClaim-Checkパターンを設計する：

1. **CLAUDE.md** に1KB超えペイロードはClaim-Checkを使用・S3に保存してclaimIdのみをメッセージに含める・処理後にConsumerがストレージから削除を明記
2. **事前署名URL（15分TTL）** でclaimIdから一時的なダウンロードリンクを生成——S3バケットを直接公開せず、Consumer以外がアクセスできない
3. **Redisのclaimid→S3キーマッピング** でS3のキー体系をConsumerに隠蔽——S3のパス変更があってもRedisのTTL（72時間）が切れた後の清算のみで対応可能
4. **サイズ自動判定（`publishWithAutoClaimCheck`）** で閾値を超えた場合だけClaim-Checkを使用——小さなメッセージはオーバーヘッドなく直接送信

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
