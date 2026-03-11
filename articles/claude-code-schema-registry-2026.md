---
title: "Claude Codeでイベントスキーマレジストリを設計する：JSON Schema検証・バージョニング・後方互換"
emoji: "📋"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-16 17:00"
---

## はじめに

「イベントのペイロード構造を変えたら下流のサービスが壊れた」——スキーマレジストリでイベントのスキーマをバージョン管理し、後方互換性を保ちながら進化させる設計をClaude Codeに生成させる。

---

## CLAUDE.mdにスキーマレジストリ設計ルールを書く

```markdown
## イベントスキーマレジストリ設計ルール

### スキーマバージョニング
- セマンティックバージョニング（1.0.0, 1.1.0, 2.0.0）
- MINOR（1.x.0）: 後方互換の追加（フィールド追加、optional化）
- MAJOR（2.0.0）: 破壊的変更（フィールド削除、型変更）

### 後方互換ルール
- フィールド追加は常に任意（required追加は禁止）
- フィールド削除はMajorバージョンで行い、deprecated期間を設ける
- 型変更（string→number等）は禁止

### バリデーション
- 送信前にスキーマ検証（無効なイベントを発行しない）
- 受信時も検証（バージョン不一致を検出）
- 検証エラーはDead Letter Queueへ
```

---

## スキーマレジストリの生成

```
イベントスキーマレジストリを設計してください。

要件：
- JSON Schema定義とバージョン管理
- 送受信時のスキーマ検証
- 後方互換チェック（CI/CD連携）
- スキーマ進化のマイグレーション

生成ファイル: src/events/schema/
```

---

## 生成されるスキーマレジストリ実装

```typescript
// src/events/schema/registry.ts — スキーマレジストリ

import Ajv, { JSONSchemaType } from 'ajv';
import addFormats from 'ajv-formats';

const ajv = new Ajv({ allErrors: true });
addFormats(ajv);

export interface SchemaVersion {
  version: string;          // semver: '1.0.0'
  schema: object;           // JSON Schema
  deprecated?: boolean;
  deprecatedReason?: string;
  registeredAt: Date;
}

export class SchemaRegistry {
  private readonly schemas = new Map<string, SchemaVersion[]>();

  register(eventType: string, version: string, schema: object): void {
    const versions = this.schemas.get(eventType) ?? [];

    // バージョン重複チェック
    if (versions.some(v => v.version === version)) {
      throw new Error(`Schema version ${version} already registered for ${eventType}`);
    }

    // 後方互換チェック（MINOR変更）
    if (versions.length > 0) {
      const latest = versions[versions.length - 1];
      const [latestMajor] = latest.version.split('.').map(Number);
      const [newMajor] = version.split('.').map(Number);

      if (newMajor === latestMajor) {
        this.checkBackwardCompatibility(latest.schema, schema, eventType, version);
      }
    }

    versions.push({ version, schema, registeredAt: new Date() });
    this.schemas.set(eventType, versions);
  }

  // 後方互換性検証（MINORバージョン変更時）
  private checkBackwardCompatibility(oldSchema: any, newSchema: any, eventType: string, newVersion: string): void {
    const oldRequired = new Set(oldSchema.required ?? []);
    const newRequired = new Set(newSchema.required ?? []);

    // 新しい必須フィールドの追加は禁止
    for (const field of newRequired) {
      if (!oldRequired.has(field)) {
        throw new SchemaCompatibilityError(
          `Cannot add required field '${field}' in ${eventType}@${newVersion} (backward-incompatible)`
        );
      }
    }

    // フィールドの削除は禁止（MINORの場合）
    const oldFields = new Set(Object.keys(oldSchema.properties ?? {}));
    const newFields = new Set(Object.keys(newSchema.properties ?? {}));
    for (const field of oldFields) {
      if (!newFields.has(field)) {
        throw new SchemaCompatibilityError(
          `Cannot remove field '${field}' in ${eventType}@${newVersion} without major version bump`
        );
      }
    }
  }

  validate(eventType: string, version: string, data: unknown): { valid: boolean; errors?: string[] } {
    const schemaVersion = this.getSchema(eventType, version);
    if (!schemaVersion) {
      return { valid: false, errors: [`Unknown schema: ${eventType}@${version}`] };
    }

    const validate = ajv.compile(schemaVersion.schema);
    const valid = validate(data);

    return {
      valid,
      errors: valid ? undefined : validate.errors?.map(e => `${e.instancePath} ${e.message}`) ?? [],
    };
  }

  getSchema(eventType: string, version?: string): SchemaVersion | undefined {
    const versions = this.schemas.get(eventType);
    if (!versions || versions.length === 0) return undefined;
    if (!version) return versions[versions.length - 1]; // 最新
    return versions.find(v => v.version === version);
  }

  // バージョン互換性チェック（受信側が古いバージョンでも読めるか）
  isCompatible(eventType: string, producerVersion: string, consumerVersion: string): boolean {
    const [pMajor] = producerVersion.split('.').map(Number);
    const [cMajor] = consumerVersion.split('.').map(Number);
    return pMajor === cMajor; // Majorが同じなら互換
  }
}

export const schemaRegistry = new SchemaRegistry();
```

```typescript
// src/events/schema/eventSchemas.ts — イベントスキーマ定義

// 注文完了イベント v1.0.0
schemaRegistry.register('order.completed', '1.0.0', {
  type: 'object',
  required: ['orderId', 'userId', 'totalAmount', 'completedAt'],
  properties: {
    orderId:     { type: 'string', format: 'uuid' },
    userId:      { type: 'string', format: 'uuid' },
    totalAmount: { type: 'number', minimum: 0 },
    completedAt: { type: 'string', format: 'date-time' },
  },
  additionalProperties: false,
});

// v1.1.0: itemsフィールドを追加（後方互換: optional）
schemaRegistry.register('order.completed', '1.1.0', {
  type: 'object',
  required: ['orderId', 'userId', 'totalAmount', 'completedAt'], // requiredは変更なし
  properties: {
    orderId:     { type: 'string', format: 'uuid' },
    userId:      { type: 'string', format: 'uuid' },
    totalAmount: { type: 'number', minimum: 0 },
    completedAt: { type: 'string', format: 'date-time' },
    items: {     // 追加: optional（後方互換）
      type: 'array',
      items: {
        type: 'object',
        required: ['productId', 'quantity'],
        properties: {
          productId: { type: 'string' },
          quantity:  { type: 'integer', minimum: 1 },
        },
      },
    },
  },
  additionalProperties: false,
});
```

```typescript
// src/events/schema/validatingPublisher.ts — スキーマ検証付きイベント発行

export class ValidatingEventPublisher {
  async publish(eventType: string, version: string, data: unknown): Promise<void> {
    const validation = schemaRegistry.validate(eventType, version, data);

    if (!validation.valid) {
      logger.error({ eventType, version, errors: validation.errors }, 'Schema validation failed');
      throw new SchemaValidationError(
        `Invalid event ${eventType}@${version}: ${validation.errors?.join(', ')}`
      );
    }

    const envelope = {
      id: ulid(),
      type: eventType,
      version,
      payload: data,
      publishedAt: new Date().toISOString(),
    };

    await redis.xAdd(`events:${eventType}`, '*', {
      envelope: JSON.stringify(envelope),
    });

    logger.info({ eventType, version, eventId: envelope.id }, 'Event published');
  }
}

// 受信時の検証（下流サービス）
export async function processEvent(rawMessage: string): Promise<void> {
  const envelope = JSON.parse(rawMessage);
  const { type, version, payload } = envelope;

  const validation = schemaRegistry.validate(type, version);
  if (!validation.valid) {
    logger.error({ type, version, errors: validation.errors }, 'Received invalid event');
    // DLQへ送信
    await redis.xAdd('events:dlq', '*', { message: rawMessage, reason: JSON.stringify(validation.errors) });
    return;
  }

  // バージョン互換チェック
  const consumerVersion = '1.0.0'; // このサービスが対応しているバージョン
  if (!schemaRegistry.isCompatible(type, version, consumerVersion)) {
    logger.warn({ type, producerVersion: version, consumerVersion }, 'Major version mismatch');
    // Major変更: 処理を保留（新しいデプロイを待つ）
    return;
  }

  // 正常処理
  await handleEvent(type, payload);
}
```

---

## まとめ

Claude Codeでイベントスキーマレジストリを設計する：

1. **CLAUDE.md** にMINOR変更での後方互換ルール（required追加禁止・フィールド削除禁止）・MAJOR変更は破壊的変更のみ・DLQへの無効イベント送信を明記
2. **後方互換チェック** をregister時に自動実行——「required追加」「フィールド削除」を試みるとコード時点でエラー
3. **バージョンエンベロープ** でメッセージにtype+versionを含める——受信側が自分の対応バージョンを確認してから処理
4. **Majorバージョン不一致** は処理を保留——新しいデプロイが完了するまでメッセージを消費しないことで古いコードでの誤処理を防止

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
