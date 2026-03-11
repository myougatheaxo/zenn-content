---
title: "Claude Codeでイベントバージョニングを設計する：ドメインイベントのスキーマ進化・アップキャスト・後方互換"
emoji: "📋"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-21 19:00"
---

## はじめに

「イベントのフォーマットを変えたら過去のイベントが処理できなくなった」「EventStoringのイベントにフィールドを追加したらProjectionが壊れた」——イベントスキーマの安全な進化とアップキャスト（古いバージョンを新しいバージョンに変換）を設計をClaude Codeに生成させる。

---

## CLAUDE.mdにイベントバージョニング設計ルールを書く

```markdown
## イベントバージョニング設計ルール

### バージョン管理の原則
- 全ドメインイベントにバージョン番号を付与（v1, v2...）
- 後方互換変更はマイナー更新（v1.0 → v1.1）
- 破壊的変更は新バージョン（v1 → v2）
- 古いバージョンのイベントは処理し続けられる

### 安全な変更（後方互換）
- フィールドの追加（デフォルト値つき）
- フィールドの任意化
- フィールドの型拡張（string → string | null）

### 破壊的変更への対応
- アップキャスト: v1イベントをv2に変換してから処理
- アップキャスターのチェーン: v1→v2→v3と段階的に変換
- イベントストアに変換済みイベントを保存しない（ソースデータを汚染しない）
```

---

## イベントバージョニング実装の生成

```
イベントバージョニングシステムを設計してください。

要件：
- バージョン付きイベント型
- アップキャスターチェーン
- スキーマレジストリ
- Projectionの後方互換

生成ファイル: src/domain/events/versioning/
```

---

## 生成されるイベントバージョニング実装

```typescript
// src/domain/events/versioning/versionedEvent.ts — バージョン付きイベント

export interface VersionedEvent {
  readonly eventId: string;
  readonly eventType: string;
  readonly eventVersion: number;  // スキーマバージョン
  readonly occurredAt: Date;
  readonly aggregateId: string;
}

// OrderCreated v1（最初のバージョン）
export interface OrderCreatedV1 extends VersionedEvent {
  readonly eventType: 'OrderCreated';
  readonly eventVersion: 1;
  readonly orderId: string;
  readonly userId: string;
  readonly totalAmount: number;  // 単純なnumber（通貨を持たない）
}

// OrderCreated v2（通貨情報を追加）
export interface OrderCreatedV2 extends VersionedEvent {
  readonly eventType: 'OrderCreated';
  readonly eventVersion: 2;
  readonly orderId: string;
  readonly userId: string;
  readonly totalAmount: number;
  readonly currency: string;  // 追加
  readonly itemCount: number;  // 追加
}

// OrderCreated v3（ユーザーメールを追加 + amountをオブジェクトに変更）
export interface OrderCreatedV3 extends VersionedEvent {
  readonly eventType: 'OrderCreated';
  readonly eventVersion: 3;
  readonly orderId: string;
  readonly userId: string;
  readonly amount: { value: number; currency: string };  // 型変更
  readonly itemCount: number;
  readonly userEmail: string;  // 追加
}

// 最新バージョンの型エイリアス
export type OrderCreatedEvent = OrderCreatedV3;
```

```typescript
// src/domain/events/versioning/upcaster.ts — アップキャスターチェーン

export interface Upcaster<TFrom, TTo> {
  readonly fromVersion: number;
  readonly toVersion: number;
  upcast(event: TFrom): TTo;
}

// V1 → V2 アップキャスター
export const OrderCreatedV1ToV2: Upcaster<OrderCreatedV1, OrderCreatedV2> = {
  fromVersion: 1,
  toVersion: 2,
  upcast(event: OrderCreatedV1): OrderCreatedV2 {
    return {
      ...event,
      eventVersion: 2,
      currency: 'JPY',  // デフォルト値（v1は全てJPYだった）
      itemCount: 0,     // 不明なのでデフォルト値
    };
  },
};

// V2 → V3 アップキャスター
export const OrderCreatedV2ToV3: Upcaster<OrderCreatedV2, OrderCreatedV3> = {
  fromVersion: 2,
  toVersion: 3,
  upcast(event: OrderCreatedV2): OrderCreatedV3 {
    return {
      ...event,
      eventVersion: 3,
      amount: {
        value: event.totalAmount,
        currency: event.currency,
      },
      userEmail: '',  // 過去データは空文字（後で補完する場合は別のマイグレーション）
    };
  },
};

// アップキャスターレジストリ（全バージョンの変換を管理）
export class UpcasterRegistry {
  private readonly upcasters = new Map<string, Map<number, Upcaster<any, any>>>();

  register<TFrom, TTo>(
    eventType: string,
    upcaster: Upcaster<TFrom, TTo>
  ): void {
    if (!this.upcasters.has(eventType)) {
      this.upcasters.set(eventType, new Map());
    }
    this.upcasters.get(eventType)!.set(upcaster.fromVersion, upcaster);
  }

  // 任意のバージョンから最新バージョンへのアップキャストチェーンを実行
  upcast<T extends VersionedEvent>(event: VersionedEvent, targetVersion: number): T {
    const typeUpcasters = this.upcasters.get(event.eventType);
    if (!typeUpcasters) return event as T;

    let current: VersionedEvent = event;

    while (current.eventVersion < targetVersion) {
      const upcaster = typeUpcasters.get(current.eventVersion);
      if (!upcaster) {
        throw new Error(
          `No upcaster found for ${event.eventType} v${current.eventVersion} → v${current.eventVersion + 1}`
        );
      }
      current = upcaster.upcast(current);
    }

    return current as T;
  }
}

// レジストリの設定
export const upcasterRegistry = new UpcasterRegistry();
upcasterRegistry.register('OrderCreated', OrderCreatedV1ToV2);
upcasterRegistry.register('OrderCreated', OrderCreatedV2ToV3);
```

```typescript
// src/domain/events/versioning/versionedEventStore.ts — バージョン対応イベントストア

const CURRENT_VERSION: Record<string, number> = {
  'OrderCreated': 3,
  'OrderSubmitted': 2,
  'OrderCompleted': 1,
};

export class VersionedEventStore {
  constructor(
    private readonly db: PrismaClient,
    private readonly registry: UpcasterRegistry
  ) {}

  // イベントを保存（常に最新バージョンで保存）
  async save(event: VersionedEvent): Promise<void> {
    const expectedVersion = CURRENT_VERSION[event.eventType] ?? 1;

    if (event.eventVersion !== expectedVersion) {
      throw new Error(
        `Event version mismatch: expected v${expectedVersion}, got v${event.eventVersion}`
      );
    }

    await this.db.domainEvent.create({
      data: {
        id: event.eventId,
        aggregateId: event.aggregateId,
        eventType: event.eventType,
        eventVersion: event.eventVersion,
        payload: JSON.stringify(event),
        occurredAt: event.occurredAt,
      },
    });
  }

  // イベントを読み込み（古いバージョンは自動アップキャスト）
  async loadEvents(aggregateId: string): Promise<VersionedEvent[]> {
    const rows = await this.db.domainEvent.findMany({
      where: { aggregateId },
      orderBy: { occurredAt: 'asc' },
    });

    return rows.map(row => {
      const event: VersionedEvent = JSON.parse(row.payload);
      const targetVersion = CURRENT_VERSION[event.eventType] ?? event.eventVersion;

      // 旧バージョンは自動アップキャスト
      if (event.eventVersion < targetVersion) {
        return this.registry.upcast(event, targetVersion);
      }

      return event;
    });
  }
}

// Projection（常に最新バージョンのイベントを受け取る）
export class OrderProjection {
  // V3形式のイベントだけを処理（アップキャスト済み）
  async onOrderCreated(event: OrderCreatedV3): Promise<void> {
    await this.db.orderReadModel.upsert({
      where: { orderId: event.orderId },
      create: {
        orderId: event.orderId,
        userId: event.userId,
        totalAmount: event.amount.value,
        currency: event.amount.currency,
        itemCount: event.itemCount,
        userEmail: event.userEmail,
        createdAt: event.occurredAt,
      },
      update: {},
    });
  }
}
```

---

## まとめ

Claude Codeでイベントバージョニングを設計する：

1. **CLAUDE.md** に全イベントにeventVersionフィールド必須・後方互換変更は同バージョン・破壊的変更は新バージョン+アップキャスター・Projectionは常に最新バージョンを受け取るを明記
2. **アップキャスターチェーン（V1→V2→V3）** で古いイベントを段階的に変換——`OrderCreatedV1`を`UpcasterRegistry.upcast(event, 3)`に渡すだけでV1→V2→V3と自動変換。イベントストアの生データは変えない
3. **`CURRENT_VERSION`レジストリ** で各イベントタイプの最新バージョンを管理——`OrderCreated: 3`と定義することで、保存時のバージョン検証と読み込み時のアップキャスト先が明確になる
4. **Projectionは最新バージョンだけを処理** ——`onOrderCreated(event: OrderCreatedV3)`と型付けすることで、Projectionにバージョン分岐が入らない。古いイベントの変換はEventStore層で完結

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
