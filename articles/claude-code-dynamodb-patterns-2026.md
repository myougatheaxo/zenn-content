---
title: "Claude CodeでDynamoDB Single-Table設計を実装する：GSI・条件付き書き込み・Streams"
emoji: "🗃️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "aws", "dynamodb"]
published: true
published_at: "2026-03-14 12:00"
---

## はじめに

「とりあえずRDS」でなくDynamoDBを選ぶ場面——Single-Table Designで全エンティティを1テーブルに入れ、GSIで多様なクエリパターンを実現する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにDynamoDB設計ルールを書く

```markdown
## DynamoDB Single-Table設計ルール

### キー設計
- PK: エンティティタイプ + ID（例: USER#123, ORDER#456）
- SK: バリエーション or 関連エンティティ（例: PROFILE, ORDER#789）
- GSI1: 逆引き・日付順・ステータス別クエリ用
- GSI2: テナント別・カテゴリ別等

### アクセスパターン先行設計
- 必要なクエリを列挙してからキー設計を決める
- JOIN禁止: 1クエリで必要なデータを取得できる設計
- ホットパーティション回避: ランダムサフィックスを付与

### 書き込み
- 条件付き書き込み: VersionAttrで楽観ロック
- バッチ書き込み: 25件/リクエスト上限
- トランザクション: TransactWrite（最大25操作）
```

---

## DynamoDB実装の生成

```
DynamoDB Single-Table Designを設計してください。

要件：
- ユーザー・注文・商品の1テーブル設計
- GSIによる多様なクエリ
- 楽観ロック
- DynamoDB Streams連携

生成ファイル: src/db/dynamodb/
```

---

## 生成されるDynamoDB実装

```typescript
// src/db/dynamodb/client.ts — DynamoDBクライアント設定

import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, TranslateConfig } from '@aws-sdk/lib-dynamodb';

const client = new DynamoDBClient({
  region: process.env.AWS_REGION ?? 'ap-northeast-1',
  ...(process.env.DYNAMODB_ENDPOINT && {
    endpoint: process.env.DYNAMODB_ENDPOINT, // ローカル開発用 (DynamoDB Local)
    credentials: { accessKeyId: 'local', secretAccessKey: 'local' },
  }),
});

const translateConfig: TranslateConfig = {
  marshallOptions: {
    convertEmptyValues: false,
    removeUndefinedValues: true,
    convertClassInstanceToMap: false,
  },
};

export const docClient = DynamoDBDocumentClient.from(client, translateConfig);
export const TABLE_NAME = process.env.DYNAMODB_TABLE ?? 'myapp';
```

```typescript
// src/db/dynamodb/keys.ts — キー設計（Single-Table）

// アクセスパターン:
// 1. ユーザー取得:         PK=USER#{id}, SK=PROFILE
// 2. ユーザーの注文一覧:   PK=USER#{id}, SK begins_with ORDER#
// 3. 注文詳細:             PK=ORDER#{id}, SK=DETAILS
// 4. 日付別注文:           GSI1PK=ORDER_DATE#{yyyy-mm}, GSI1SK={createdAt}#{id}
// 5. ステータス別注文:     GSI1PK=STATUS#{status}, GSI1SK={createdAt}#{id}

export const Keys = {
  user: {
    profile: (userId: string) => ({
      PK: `USER#${userId}`,
      SK: 'PROFILE',
    }),
    orderPrefix: (userId: string) => ({
      PK: `USER#${userId}`,
      SK: 'ORDER#',
    }),
  },
  order: {
    details: (orderId: string) => ({
      PK: `ORDER#${orderId}`,
      SK: 'DETAILS',
    }),
  },
  gsi1: {
    ordersByDate: (month: string) => ({
      GSI1PK: `ORDER_DATE#${month}`,
    }),
    ordersByStatus: (status: string) => ({
      GSI1PK: `STATUS#${status}`,
    }),
  },
};
```

```typescript
// src/db/dynamodb/repositories/userRepository.ts — ユーザーリポジトリ

import { GetCommand, PutCommand, QueryCommand, UpdateCommand } from '@aws-sdk/lib-dynamodb';

export class UserRepository {
  async getUser(userId: string): Promise<User | null> {
    const result = await docClient.send(new GetCommand({
      TableName: TABLE_NAME,
      Key: Keys.user.profile(userId),
    }));

    return result.Item ? this.deserialize(result.Item) : null;
  }

  async createUser(user: CreateUserInput): Promise<User> {
    const now = new Date().toISOString();
    const item = {
      ...Keys.user.profile(user.id),
      GSI1PK: `TENANT#${user.tenantId}`,
      GSI1SK: `USER#${user.email}`,  // テナント内でメール検索
      entityType: 'USER',
      id: user.id,
      email: user.email,
      name: user.name,
      tenantId: user.tenantId,
      version: 1,
      createdAt: now,
      updatedAt: now,
    };

    await docClient.send(new PutCommand({
      TableName: TABLE_NAME,
      Item: item,
      // 重複作成防止（PK/SKが存在しないこと）
      ConditionExpression: 'attribute_not_exists(PK)',
    }));

    return this.deserialize(item);
  }

  async updateUser(userId: string, updates: Partial<User>, expectedVersion: number): Promise<User> {
    const result = await docClient.send(new UpdateCommand({
      TableName: TABLE_NAME,
      Key: Keys.user.profile(userId),
      UpdateExpression: 'SET #name = :name, updatedAt = :updatedAt, version = :newVersion',
      // 楽観ロック: バージョンが一致する場合のみ更新
      ConditionExpression: 'version = :expectedVersion',
      ExpressionAttributeNames: { '#name': 'name' },
      ExpressionAttributeValues: {
        ':name': updates.name,
        ':updatedAt': new Date().toISOString(),
        ':newVersion': expectedVersion + 1,
        ':expectedVersion': expectedVersion,
      },
      ReturnValues: 'ALL_NEW',
    }));

    if (!result.Attributes) throw new Error('Update failed');
    return this.deserialize(result.Attributes);
  }

  // ユーザーの注文一覧をPKで取得
  async getUserOrders(userId: string, limit = 20): Promise<Order[]> {
    const result = await docClient.send(new QueryCommand({
      TableName: TABLE_NAME,
      KeyConditionExpression: 'PK = :pk AND begins_with(SK, :skPrefix)',
      ExpressionAttributeValues: {
        ':pk': `USER#${userId}`,
        ':skPrefix': 'ORDER#',
      },
      ScanIndexForward: false, // 新しい順
      Limit: limit,
    }));

    return (result.Items ?? []).map(item => this.deserializeOrder(item));
  }

  private deserialize(item: Record<string, unknown>): User {
    return {
      id: item.id as string,
      email: item.email as string,
      name: item.name as string,
      tenantId: item.tenantId as string,
      version: item.version as number,
      createdAt: item.createdAt as string,
      updatedAt: item.updatedAt as string,
    };
  }
}
```

```typescript
// src/db/dynamodb/streams/handler.ts — DynamoDB Streams処理

import { DynamoDBStreamHandler } from 'aws-lambda';
import { unmarshall } from '@aws-sdk/util-dynamodb';

export const handler: DynamoDBStreamHandler = async (event) => {
  for (const record of event.Records) {
    if (!record.dynamodb) continue;

    const eventType = record.eventName; // INSERT | MODIFY | REMOVE
    const newImage = record.dynamodb.NewImage
      ? unmarshall(record.dynamodb.NewImage as any)
      : null;
    const oldImage = record.dynamodb.OldImage
      ? unmarshall(record.dynamodb.OldImage as any)
      : null;

    const entityType = newImage?.entityType ?? oldImage?.entityType;

    switch (entityType) {
      case 'ORDER':
        await handleOrderChange(eventType!, newImage, oldImage);
        break;
      case 'USER':
        await handleUserChange(eventType!, newImage, oldImage);
        break;
    }
  }
};

async function handleOrderChange(
  eventType: string,
  newOrder: Record<string, unknown> | null,
  oldOrder: Record<string, unknown> | null
): Promise<void> {
  if (eventType === 'INSERT') {
    // 新規注文: Elasticsearchにインデックス追加
    await searchIndex.index('orders', newOrder!);
    // 在庫引き当て
    await inventoryService.reserve(newOrder!.id as string);
  } else if (eventType === 'MODIFY') {
    const statusChanged = oldOrder?.status !== newOrder?.status;
    if (statusChanged && newOrder?.status === 'SHIPPED') {
      // 発送完了通知
      await notificationService.send(newOrder!.userId as string, 'ORDER_SHIPPED');
    }
  }
}
```

---

## まとめ

Claude CodeでDynamoDB Single-Table設計を実装する：

1. **CLAUDE.md** にアクセスパターン先行設計・PK/SKにエンティティタイプ付与・GSIで多パターン対応を明記
2. **Keys.user.orderPrefix** でbegins_withクエリを使い、ユーザーの全注文を1クエリで取得
3. **楽観ロック** `ConditionExpression: 'version = :expectedVersion'` で同時更新による競合を防止
4. **DynamoDB Streams** でINSERT/MODIFY/REMOVEイベントをトリガーに検索インデックス更新・通知送信

---

*DB設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
