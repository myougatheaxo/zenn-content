---
title: "Claude Codeでファイルアップロードを設計する：S3連携とセキュリティ対策"
emoji: "📤"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "s3", "ファイル処理"]
published: true
---

## はじめに

ファイルアップロードは実装するとセキュリティリスクが多い——ファイルサイズ制限・ファイル種別チェック・マルウェアスキャン・ストレージ管理。Claude Codeに安全な設計を生成させる。

---

## CLAUDE.mdにファイルアップロードルールを書く

```markdown
## ファイルアップロードルール

### セキュリティ（必須）
- ファイルサイズ制限: 画像10MB, ドキュメント25MB, 動画500MB
- ファイル種別: MIMEタイプとマジックバイトの両方を検証
- ファイル名: ランダムなUUIDに置換（元のファイル名を使わない）
- ウイルススキャン: ClamAVまたはAWSのS3 Malware Scanning
- アップロード先: S3（ローカルストレージ禁止）

### 許可するファイル種別
- 画像: image/jpeg, image/png, image/webp, image/gif
- ドキュメント: application/pdf
- スプレッドシート: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet

### アップロードフロー
1. クライアント → API: メタデータを送信 → 署名付きURLを取得
2. クライアント → S3: 署名付きURLで直接アップロード（サーバーを通さない）
3. S3 → API（Webhook）: アップロード完了を通知
4. API: ウイルススキャン → DB登録 → 公開URLを発行

### 公開URL
- 署名付きURL: 期限付き（1時間）
- 機密ファイルはpublic ACL禁止
```

---

## 署名付きURLによるアップロードの生成

```
S3の署名付きURLを使ったファイルアップロード設計を生成してください。

要件：
- フロー: API → 署名付きURL発行 → クライアントがS3に直接アップロード → DBに記録
- ファイルサイズ制限をURL発行時に設定（Content-Length-Range）
- MIMEタイプの検証
- ファイル名をUUIDに置換
- アップロード完了後のコールバック処理

生成するファイル:
- src/services/fileUploadService.ts
- src/routes/upload.ts
```

```typescript
// src/services/fileUploadService.ts
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { v4 as uuidv4 } from 'uuid';

const ALLOWED_MIME_TYPES = new Set([
  'image/jpeg', 'image/png', 'image/webp', 'image/gif',
  'application/pdf',
]);

const MAX_FILE_SIZES = {
  'image/jpeg': 10 * 1024 * 1024,
  'image/png': 10 * 1024 * 1024,
  'image/webp': 10 * 1024 * 1024,
  'image/gif': 10 * 1024 * 1024,
  'application/pdf': 25 * 1024 * 1024,
};

export class FileUploadService {
  private s3 = new S3Client({ region: process.env.AWS_REGION });
  private bucket = process.env.S3_BUCKET!;

  async createUploadUrl(
    userId: string,
    originalFilename: string,
    mimeType: string,
    fileSize: number
  ): Promise<{ uploadUrl: string; fileKey: string }> {
    // MIMEタイプ検証
    if (!ALLOWED_MIME_TYPES.has(mimeType)) {
      throw new BadRequestError(`Unsupported file type: ${mimeType}`);
    }

    // ファイルサイズ検証
    const maxSize = MAX_FILE_SIZES[mimeType as keyof typeof MAX_FILE_SIZES];
    if (fileSize > maxSize) {
      throw new BadRequestError(`File too large (max: ${maxSize / 1024 / 1024}MB)`);
    }

    // 元のファイル名を使わずUUIDで生成
    const ext = originalFilename.split('.').pop() ?? '';
    const fileKey = `uploads/${userId}/${uuidv4()}.${ext}`;

    const command = new PutObjectCommand({
      Bucket: this.bucket,
      Key: fileKey,
      ContentType: mimeType,
      // メタデータとして元のファイル名を保存（URL/パスには使わない）
      Metadata: { 'original-filename': encodeURIComponent(originalFilename) },
    });

    const uploadUrl = await getSignedUrl(this.s3, command, { expiresIn: 900 }); // 15分

    return { uploadUrl, fileKey };
  }

  async getDownloadUrl(fileKey: string): Promise<string> {
    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: fileKey,
    });
    return getSignedUrl(this.s3, command, { expiresIn: 3600 }); // 1時間
  }
}
```

---

## マジックバイトによるファイル種別検証

```
ファイルのマジックバイトを検証するユーティリティを生成してください。

要件：
- MIMEタイプだけでなく実際のバイト列を確認
- 検証するファイル種別: JPEG, PNG, GIF, WEBP, PDF
- ファイルの先頭バイトのみ読み取る（全ファイルを読まない）
- 不一致の場合は拒否（content-typeの偽装を防ぐ）

保存先: src/utils/magicBytes.ts
```

```typescript
// src/utils/magicBytes.ts
const MAGIC_BYTES: Record<string, number[][]> = {
  'image/jpeg': [[0xff, 0xd8, 0xff]],
  'image/png': [[0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a]],
  'image/gif': [[0x47, 0x49, 0x46, 0x38, 0x37], [0x47, 0x49, 0x46, 0x38, 0x39]],
  'image/webp': [[0x52, 0x49, 0x46, 0x46]], // RIFF (WebP)
  'application/pdf': [[0x25, 0x50, 0x44, 0x46]], // %PDF
};

export async function validateMagicBytes(
  buffer: Buffer,
  mimeType: string
): Promise<boolean> {
  const patterns = MAGIC_BYTES[mimeType];
  if (!patterns) return false;

  return patterns.some((pattern) =>
    pattern.every((byte, index) => buffer[index] === byte)
  );
}
```

---

## 画像の自動リサイズ

```
アップロードされた画像を複数サイズにリサイズするBullMQジョブを生成してください。

要件：
- トリガー: S3アップロード完了 → S3イベントでBullMQに追加
- サイズ: thumbnail(150x150), medium(800x600), original（そのまま）
- sharp を使ってリサイズ
- リサイズ後のパス: uploads/{userId}/{uuid}-{size}.webp（WebP変換）
- DB: 各サイズのURLをfileテーブルに保存

生成ファイル: src/workers/imageResizeWorker.ts
```

---

## まとめ

Claude Codeでファイルアップロードを設計する：

1. **CLAUDE.md** にセキュリティルール・許可種別・アップロードフローを明記
2. **署名付きURL** でサーバーを通さずS3に直接アップロード
3. **マジックバイト検証** でMIMEタイプ偽装を防ぐ
4. **BullMQで画像リサイズ** を非同期処理

---

*ファイルアップロードのセキュリティレビューは **Security Pack（¥1,480）** の `/security-check` で自動化できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
