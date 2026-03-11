---
title: "Claude Codeでワークフローエンジンを設計する：ステートマシン・承認フロー・タイムアウト"
emoji: "⚙️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "redis"]
published: true
published_at: "2026-03-12 17:00"
---

## はじめに

「注文→審査→支払い→発送」の複雑なワークフロー——XStateとDBで永続化されたステートマシンを実装し、承認フロー・タイムアウト・並列ステップをClaude Codeに設計させる。

---

## CLAUDE.mdにワークフローエンジン設計ルールを書く

```markdown
## ワークフローエンジン設計ルール

### ステートマシン設計
- 状態遷移はXStateで定義（コード外で変更可能）
- 全状態遷移をDBに永続化（クラッシュしても再開可能）
- イベントドリブン（外部トリガーで遷移）

### 承認フロー
- 承認待ち状態のタイムアウト: 48時間
- タイムアウト時は自動エスカレーション
- 承認履歴を全て記録（監査証跡）

### エラーハンドリング
- ステップ失敗は自動リトライ（最大3回）
- 最終失敗は手動介入キューへ
- デッドレター状態（manual_review）で人間が確認
```

---

## ワークフローエンジンの生成

```
承認フロー付きワークフローエンジンを設計してください。

要件：
- XStateベースのステートマシン
- DB永続化（再開可能）
- 承認待ちタイムアウト
- 並列ステップ実行
- 監査ログ

生成ファイル: src/workflow/
```

---

## 生成されるワークフロー実装

```typescript
// src/workflow/orderWorkflow.ts
import { createMachine, assign, interpret } from 'xstate';

// 注文ワークフローのステートマシン定義
export const orderWorkflowMachine = createMachine({
  id: 'orderWorkflow',
  initial: 'submitted',
  context: {
    orderId: '',
    approvals: [] as string[],
    rejectionReason: '',
    retryCount: 0,
  },
  states: {
    submitted: {
      on: {
        AUTO_APPROVE:  'payment_pending',
        NEEDS_REVIEW:  'pending_approval',
      },
      // 注文額に応じて自動承認 or 審査振り分け
      entry: 'routeOrder',
    },

    pending_approval: {
      after: {
        // 48時間でタイムアウト → エスカレーション
        172_800_000: { target: 'escalated' },
      },
      on: {
        APPROVE:  { target: 'payment_pending', actions: 'recordApproval' },
        REJECT:   { target: 'rejected',        actions: 'recordRejection' },
      },
    },

    escalated: {
      on: {
        APPROVE:  { target: 'payment_pending', actions: 'recordApproval' },
        REJECT:   { target: 'rejected',        actions: 'recordRejection' },
      },
      entry: 'notifyEscalation',
    },

    payment_pending: {
      invoke: {
        id: 'processPayment',
        src: 'processPaymentService',
        onDone:  { target: 'fulfillment' },
        onError: [
          { cond: 'canRetry', target: 'payment_pending', actions: 'incrementRetry' },
          { target: 'payment_failed' },
        ],
      },
    },

    fulfillment: {
      type: 'parallel',
      states: {
        inventory: {
          initial: 'reserving',
          states: {
            reserving: {
              invoke: { src: 'reserveInventoryService', onDone: 'reserved', onError: 'failed' },
            },
            reserved: { type: 'final' },
            failed:   { type: 'final' },
          },
        },
        shipping: {
          initial: 'creating_label',
          states: {
            creating_label: {
              invoke: { src: 'createShippingLabelService', onDone: 'label_created', onError: 'failed' },
            },
            label_created: { type: 'final' },
            failed:        { type: 'final' },
          },
        },
      },
      onDone: 'shipped', // 両並列ステップ完了後
    },

    shipped: { type: 'final' },
    rejected: { type: 'final' },
    payment_failed: { type: 'final' },
    manual_review: { type: 'final' }, // デッドレター
  },
}, {
  guards: {
    canRetry: (ctx) => ctx.retryCount < 3,
  },
  actions: {
    recordApproval: assign({ approvals: (ctx, e) => [...ctx.approvals, e.approverId] }),
    recordRejection: assign({ rejectionReason: (_, e) => e.reason }),
    incrementRetry: assign({ retryCount: (ctx) => ctx.retryCount + 1 }),
    notifyEscalation: () => {}, // 実装は外部から注入
    routeOrder: () => {},
  },
});
```

```typescript
// src/workflow/workflowService.ts — DB永続化 + 再開

export class WorkflowService {
  // ワークフロー開始
  async startOrderWorkflow(orderId: string): Promise<string> {
    const machine = orderWorkflowMachine.withContext({
      ...orderWorkflowMachine.initialState.context,
      orderId,
    });

    const initialState = machine.initialState;

    // 初期状態をDBに保存
    const instance = await prisma.workflowInstance.create({
      data: {
        type: 'order',
        entityId: orderId,
        state: initialState.value as string,
        context: initialState.context,
        status: 'running',
      },
    });

    // 初期遷移を実行（routeOrderアクション）
    await this.processEntryActions(instance.id, initialState);

    return instance.id;
  }

  // ワークフローを再開（クラッシュリカバリ）
  async resumeWorkflow(instanceId: string): Promise<void> {
    const instance = await prisma.workflowInstance.findUniqueOrThrow({
      where: { id: instanceId },
    });

    // DBの状態からステートマシンを復元
    const restoredState = State.create({
      value: instance.state,
      context: instance.context,
    });

    const service = interpret(orderWorkflowMachine.withConfig({
      services: this.getServiceImplementations(),
      actions: this.getActionImplementations(instanceId),
    }));

    service.start(restoredState); // DBの状態から再開
  }

  // イベントを送信（外部からのトリガー）
  async sendEvent(instanceId: string, event: { type: string; [key: string]: any }): Promise<void> {
    const instance = await prisma.workflowInstance.findUniqueOrThrow({
      where: { id: instanceId, status: 'running' },
    });

    const currentState = State.create({ value: instance.state, context: instance.context });
    const service = interpret(orderWorkflowMachine);
    service.start(currentState);

    const nextState = service.send(event);

    // 状態遷移をDBに保存
    await prisma.$transaction([
      prisma.workflowInstance.update({
        where: { id: instanceId },
        data: {
          state: nextState.value as string,
          context: nextState.context,
          status: nextState.done ? 'completed' : 'running',
          completedAt: nextState.done ? new Date() : null,
        },
      }),
      // 遷移履歴（監査ログ）
      prisma.workflowTransition.create({
        data: {
          instanceId,
          fromState: instance.state,
          toState: nextState.value as string,
          event: event.type,
          actorId: event.actorId,
          timestamp: new Date(),
        },
      }),
    ]);

    service.stop();
  }
}
```

---

## APIと利用例

```typescript
// 注文承認APIと使用例

// 承認
router.post('/orders/:orderId/approve', requireAuth, requireRole('approver'), async (req, res) => {
  const { orderId } = req.params;
  const instance = await prisma.workflowInstance.findFirst({ where: { entityId: orderId } });

  await workflowService.sendEvent(instance!.id, {
    type: 'APPROVE',
    approverId: req.user.id,
  });

  res.json({ success: true });
});

// ワークフロー状態確認
router.get('/orders/:orderId/workflow', requireAuth, async (req, res) => {
  const instance = await prisma.workflowInstance.findFirst({
    where: { entityId: req.params.orderId },
    include: { transitions: { orderBy: { timestamp: 'desc' }, take: 10 } },
  });

  res.json({
    currentState: instance?.state,
    history: instance?.transitions,
    completedAt: instance?.completedAt,
  });
});
```

---

## まとめ

Claude Codeでワークフローエンジンを設計する：

1. **CLAUDE.md** にXState定義・DB永続化・48時間タイムアウト・監査証跡を明記
2. **XStateのstateをDBに保存** してクラッシュしても前の状態から再開可能
3. **parallel state** でInventory + Shippingを並列実行（両方完了で次のステップへ）
4. **遷移履歴テーブル** で全アクション・承認者・日時を監査証跡として保存

---

*ワークフロー設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
