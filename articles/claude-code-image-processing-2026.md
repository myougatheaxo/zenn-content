---
title: "Claude Codeで画像処理パイプラインを設計する：Sharp・リサイズ・WebP変換・CDN配信"
emoji: "🖼️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "aws", "performance"]
published: true
published_at: "2026-03-12 16:00"
---

## はじめに

ユーザーがアップロードした画像を適切なサイズ・フォーマットに変換し、CDNから高速配信する——Sharp + S3 + CloudFrontの画像処理パイプラインをClaude Codeに設計させる。

---

## CLAUDE.mdに画像処理設計ルールを書く

```markdown
## 画像処理パイプライン設計ルール

### 変換仕様
- フォーマット: WebP優先（非対応ブラウザはJPEG）
- サイズバリエーション: thumb(100px)・small(300px)・medium(600px)・large(1200px)
- 品質: WebP 80%（容量削減）・JPEG 85%
- 非同期処理: アップロード後にキュー経由で変換

### セキュリティ
- アップロード前にMIMEタイプ検証（Content-Type偽装対策）
- ファイルのマジックバイト検証
- 最大ファイルサイズ: 10MB
- ウイルススキャン（ClamAV）

### CDN配信
- CloudFrontで配信（S3直接アクセス禁止）
- Cache-Control: max-age=31536000（1年）
- 署名付きURL: プライベート画像はCloudFront署名付きURL
```

---

## 画像処理パイプラインの生成

```
画像処理パイプラインを設計してください。

要件：
- アップロード → 非同期処理キュー
- Sharp（リサイズ・WebP変換）
- 複数サイズのバリエーション生成
- S3保存 + CloudFront配信
- ウイルススキャン

生成ファイル: src/images/
```

---

## 生成される画像処理実装

```typescript
// src/images/imageProcessor.ts
import sharp from 'sharp';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';

const s3 = new S3Client({ region: 'ap-northeast-1' });

type ImageSize = 'thumb' | 'small' | 'medium' | 'large' | 'original';

const SIZE_CONFIG: Record<Exclude<ImageSize, 'original'>, { width: number; height?: number }> = {
  thumb:  { width: 100, height: 100 }, // 正方形クロップ
  small:  { width: 300 },
  medium: { width: 600 },
  large:  { width: 1200 },
};

export class ImageProcessor {
  // 画像を全サイズに変換してS3にアップロード
  async processAndStore(params: {
    sourceKey: string;  // S3の元画像キー
    imageId: string;    // DBのimage.id
    userId: string;
  }): Promise<ProcessedImageResult> {
    // S3から元画像を取得
    const sourceStream = await this.downloadFromS3(params.sourceKey);

    // ウイルススキャン（バイトバッファで実行）
    const sourceBuffer = await streamToBuffer(sourceStream);
    const scanResult = await scanForViruses(sourceBuffer);
    if (!scanResult.clean) {
      await this.quarantineImage(params.sourceKey);
      throw new SecurityError(`Malware detected in image: ${scanResult.threat}`);
    }

    // Sharpインスタンスを作成（一度読み込んで複数サイズに変換）
    const sharpInstance = sharp(sourceBuffer);
    const metadata = await sharpInstance.metadata();

    const results: Record<ImageSize, string> = {} as any;

    // オリジナルも保存（そのまま）
    const originalKey = this.buildS3Key(params.imageId, 'original', metadata.format ?? 'jpeg');
    await this.uploadToS3(originalKey, sourceBuffer, metadata.format === 'png' ? 'image/png' : 'image/jpeg');
    results.original = this.buildCDNUrl(originalKey);

    // 各サイズのWebPを並列生成
    await Promise.all(
      (Object.entries(SIZE_CONFIG) as [Exclude<ImageSize, 'original'>, typeof SIZE_CONFIG[ImageSize]][]).map(
        async ([sizeName, config]) => {
          const resized = await this.resizeToWebP(sharpInstance.clone(), config, sizeName === 'thumb');
          const key = this.buildS3Key(params.imageId, sizeName, 'webp');

          await this.uploadToS3(key, resized, 'image/webp');
          results[sizeName] = this.buildCDNUrl(key);
        }
      )
    );

    // DBに処理済み画像情報を保存
    await prisma.image.update({
      where: { id: params.imageId },
      data: {
        status: 'processed',
        urls: results,
        width: metadata.width,
        height: metadata.height,
        originalFormat: metadata.format,
        processedAt: new Date(),
      },
    });

    // 元画像（アップロード用一時ファイル）を削除
    await this.deleteFromS3(params.sourceKey);

    return { imageId: params.imageId, urls: results };
  }

  private async resizeToWebP(
    instance: sharp.Sharp,
    config: { width: number; height?: number },
    crop: boolean
  ): Promise<Buffer> {
    return instance
      .resize(config.width, config.height, {
        fit: crop ? 'cover' : 'inside',    // thumb: トリミング、それ以外: 縦横比維持
        position: 'attention',             // 顔検出でクロップ位置を決定
        withoutEnlargement: true,          // 元より大きくしない
      })
      .webp({ quality: 80, effort: 4 })   // effort: 1-6（4は速度と品質のバランス）
      .toBuffer();
  }

  private buildS3Key(imageId: string, size: string, format: string): string {
    // images/{year}/{month}/{imageId}/{size}.webp
    const now = new Date();
    return `images/${now.getFullYear()}/${String(now.getMonth() + 1).padStart(2, '0')}/${imageId}/${size}.${format}`;
  }

  private buildCDNUrl(key: string): string {
    return `https://${process.env.CLOUDFRONT_DOMAIN}/${key}`;
  }

  private async uploadToS3(key: string, data: Buffer, contentType: string): Promise<void> {
    await s3.send(new PutObjectCommand({
      Bucket: process.env.IMAGES_BUCKET!,
      Key: key,
      Body: data,
      ContentType: contentType,
      CacheControl: 'public, max-age=31536000, immutable', // 1年キャッシュ
      Metadata: { 'processed-by': 'image-processor-v1' },
    }));
  }
}
```

```typescript
// src/images/uploadHandler.ts — アップロード + キュー投入

// アップロード前検証
export async function validateImageUpload(file: Express.Multer.File): Promise<void> {
  // 1. ファイルサイズ制限
  if (file.size > 10 * 1024 * 1024) {
    throw new ValidationError('File size exceeds 10MB limit');
  }

  // 2. MIMEタイプ検証（Content-Typeは信用しない）
  const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
  if (!ALLOWED_TYPES.includes(file.mimetype)) {
    throw new ValidationError(`File type ${file.mimetype} not allowed`);
  }

  // 3. マジックバイト検証（ファイルの先頭バイトを実際に確認）
  const magicBytes = file.buffer.slice(0, 8);
  if (!isValidImageMagicBytes(magicBytes)) {
    throw new SecurityError('Invalid file content (magic bytes mismatch)');
  }
}

function isValidImageMagicBytes(bytes: Buffer): boolean {
  // JPEG: FF D8 FF
  if (bytes[0] === 0xFF && bytes[1] === 0xD8 && bytes[2] === 0xFF) return true;
  // PNG: 89 50 4E 47
  if (bytes[0] === 0x89 && bytes[1] === 0x50 && bytes[2] === 0x4E && bytes[3] === 0x47) return true;
  // GIF: 47 49 46 38
  if (bytes[0] === 0x47 && bytes[1] === 0x49 && bytes[2] === 0x46 && bytes[3] === 0x38) return true;
  // WebP: 52 49 46 46 ... 57 45 42 50
  if (bytes[0] === 0x52 && bytes[1] === 0x49 && bytes[2] === 0x46 && bytes[3] === 0x46) return true;
  return false;
}

// アップロードAPIハンドラー
router.post('/images', requireAuth, upload.single('image'), async (req, res) => {
  await validateImageUpload(req.file!);

  // S3に一時保存（処理前）
  const tempKey = `temp/${req.user.id}/${Date.now()}-${req.file!.originalname}`;
  await s3.send(new PutObjectCommand({
    Bucket: process.env.IMAGES_BUCKET!,
    Key: tempKey,
    Body: req.file!.buffer,
    ContentType: req.file!.mimetype,
  }));

  // DBにレコード作成（status: pending）
  const image = await prisma.image.create({
    data: {
      userId: req.user.id,
      status: 'pending',
      originalFilename: req.file!.originalname,
      mimeType: req.file!.mimetype,
      sizeBytes: req.file!.size,
    },
  });

  // 処理キューに投入（非同期処理）
  await imageQueue.add('process', {
    sourceKey: tempKey,
    imageId: image.id,
    userId: req.user.id,
  });

  res.status(202).json({
    imageId: image.id,
    status: 'pending',
    message: 'Image is being processed',
  });
});
```

---

## まとめ

Claude Codeで画像処理パイプラインを設計する：

1. **CLAUDE.md** にWebP優先・マジックバイト検証・非同期キュー処理・CloudFront配信を明記
2. **Sharpのclone()** で一度読み込んで複数サイズを並列変換（I/O効率化）
3. **マジックバイト検証** でContent-Type偽装を防ぐ（セキュリティ必須）
4. **S3 CacheControl immutable** + **CloudFront 1年キャッシュ** でCDNヒット率最大化

---

*画像処理設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
