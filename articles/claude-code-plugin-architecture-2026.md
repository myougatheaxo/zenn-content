---
title: "Claude Codeでプラグインアーキテクチャを設計する：動的ロード・フック・サンドボックス分離"
emoji: "🔌"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "architecture", "nodejs"]
published: true
published_at: "2026-03-16 13:00"
---

## はじめに

「機能を追加するたびにコアを変更したくない」——プラグインアーキテクチャでコア機能を変えずに拡張できるシステムをClaude Codeに設計させる。動的ロード・フックシステム・サンドボックス分離まで含む実装。

---

## CLAUDE.mdにプラグインアーキテクチャ設計ルールを書く

```markdown
## プラグインアーキテクチャ設計ルール

### プラグインインターフェース
- 各プラグインはPlugin<Config>インターフェースを実装
- フックシステム: beforeRequest, afterResponse, onError等を定義
- プラグインはコアAPIのサブセットのみ利用可（サンドボックス）

### ロードと検証
- プラグイン読み込み時にZodでconfigを検証
- 依存関係グラフでロード順を決定（依存のあるプラグインは後からロード）
- バージョン互換性チェック（semver）

### 分離と安全性
- プラグインはコアのDB/Redisに直接アクセス禁止
- コアが提供するContext API経由でのみアクセス可
- プラグインのエラーはcatchしてコアへの波及を防止
```

---

## プラグインアーキテクチャの生成

```
プラグインアーキテクチャを設計してください。

要件：
- Pluginインターフェース定義
- フックシステム（before/after/error）
- 動的プラグインロード
- コアAPIサンドボックス

生成ファイル: src/plugin/
```

---

## 生成されるプラグインアーキテクチャ実装

```typescript
// src/plugin/types.ts — プラグインインターフェース

export interface PluginContext {
  // コアAPIのサブセット（プラグインが利用可能なAPI）
  logger: Pick<Logger, 'info' | 'warn' | 'error'>;
  cache: {
    get(key: string): Promise<string | null>;
    set(key: string, value: string, ttlSeconds?: number): Promise<void>;
  };
  db: {
    query<T>(sql: string, params?: unknown[]): Promise<T[]>; // 読み取り専用クエリ
  };
  http: {
    fetch(url: string, options?: RequestInit): Promise<Response>; // 外部HTTP
  };
  config: Record<string, unknown>; // プラグイン固有config
}

export interface PluginHooks {
  // HTTPリクエスト/レスポンスフック
  beforeRequest?: (req: RequestContext) => Promise<RequestContext | null>; // nullで処理中断
  afterResponse?: (req: RequestContext, res: ResponseContext) => Promise<ResponseContext>;
  onError?: (req: RequestContext, error: Error) => Promise<void>;

  // ジョブフック（バックグラウンド処理）
  beforeJob?: (job: JobContext) => Promise<void>;
  afterJob?: (job: JobContext, result: unknown) => Promise<void>;

  // ライフサイクルフック
  onLoad?: (ctx: PluginContext) => Promise<void>;    // プラグイン初期化
  onUnload?: () => Promise<void>;  // プラグイン削除
}

export interface Plugin<Config = Record<string, unknown>> {
  name: string;
  version: string;
  description: string;
  dependencies?: string[];    // 依存プラグイン名
  configSchema?: ZodSchema<Config>;
  setup(ctx: PluginContext): PluginHooks;
}
```

```typescript
// src/plugin/pluginRegistry.ts — プラグインレジストリ

export class PluginRegistry {
  private readonly plugins = new Map<string, { plugin: Plugin; hooks: PluginHooks }>();

  async load(plugin: Plugin, config: Record<string, unknown> = {}): Promise<void> {
    if (this.plugins.has(plugin.name)) {
      throw new Error(`Plugin '${plugin.name}' is already loaded`);
    }

    // 依存関係チェック
    for (const dep of plugin.dependencies ?? []) {
      if (!this.plugins.has(dep)) {
        throw new Error(`Plugin '${plugin.name}' requires '${dep}' to be loaded first`);
      }
    }

    // configバリデーション
    const validatedConfig = plugin.configSchema
      ? plugin.configSchema.parse(config)
      : config;

    // コンテキスト作成（サンドボックス）
    const ctx = this.createContext(plugin.name, validatedConfig);

    // フック登録
    const hooks = plugin.setup(ctx);

    this.plugins.set(plugin.name, { plugin, hooks });

    // 初期化フック
    if (hooks.onLoad) {
      await hooks.onLoad(ctx);
      logger.info({ plugin: plugin.name }, 'Plugin loaded');
    }
  }

  async unload(pluginName: string): Promise<void> {
    const entry = this.plugins.get(pluginName);
    if (!entry) return;

    if (entry.hooks.onUnload) {
      await entry.hooks.onUnload();
    }

    this.plugins.delete(pluginName);
    logger.info({ plugin: pluginName }, 'Plugin unloaded');
  }

  private createContext(pluginName: string, config: Record<string, unknown>): PluginContext {
    const pluginLogger = logger.child({ plugin: pluginName });

    return {
      logger: pluginLogger,
      cache: {
        get: (key) => redis.get(`plugin:${pluginName}:${key}`),
        set: (key, value, ttl) => redis.set(`plugin:${pluginName}:${key}`, value, ttl ? { EX: ttl } : undefined),
      },
      db: {
        // 読み取り専用クエリのみ（INSERTやUPDATEは禁止）
        query: async (sql, params) => {
          if (/^\s*(INSERT|UPDATE|DELETE|DROP|TRUNCATE)/i.test(sql)) {
            throw new Error(`Plugin '${pluginName}' is not allowed to execute write queries`);
          }
          return prisma.$queryRawUnsafe(sql, ...(params ?? []));
        },
      },
      http: { fetch },
      config,
    };
  }

  // フックを全プラグインに対して実行
  async runHook<K extends keyof PluginHooks>(
    hookName: K,
    ...args: Parameters<NonNullable<PluginHooks[K]>>
  ): Promise<void> {
    for (const [name, { hooks }] of this.plugins) {
      const hook = hooks[hookName] as ((...args: unknown[]) => Promise<unknown>) | undefined;
      if (!hook) continue;

      try {
        await hook(...args);
      } catch (error) {
        // プラグインのエラーはコアに波及させない
        logger.error({ plugin: name, hookName, error }, 'Plugin hook error (ignored)');
      }
    }
  }
}
```

```typescript
// src/plugin/examples/auditLogPlugin.ts — 監査ログプラグイン

export const auditLogPlugin: Plugin<{ retentionDays: number }> = {
  name: 'audit-log',
  version: '1.0.0',
  description: '全APIリクエストの監査ログを記録する',
  configSchema: z.object({ retentionDays: z.number().default(90) }),

  setup(ctx) {
    return {
      afterResponse: async (req, res) => {
        // 書き込み操作のみ監査ログ記録
        if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(req.method)) {
          await ctx.cache.set(
            `audit:${req.requestId}`,
            JSON.stringify({ method: req.method, path: req.path, userId: req.userId, statusCode: res.statusCode, timestamp: Date.now() }),
            ctx.config.retentionDays * 86400
          );
        }
        return res; // レスポンスを変更しない
      },
    };
  },
};

// src/plugin/examples/rateLimitPlugin.ts — IPブラックリストプラグイン
export const ipBlockPlugin: Plugin<{ blockedIPs: string[] }> = {
  name: 'ip-block',
  version: '1.0.0',
  description: '特定IPからのリクエストをブロック',
  configSchema: z.object({ blockedIPs: z.array(z.string()) }),

  setup(ctx) {
    return {
      beforeRequest: async (req) => {
        if (ctx.config.blockedIPs.includes(req.ip)) {
          ctx.logger.warn({ ip: req.ip }, 'Blocked IP detected');
          return null; // nullを返すとリクエスト中断
        }
        return req;
      },
    };
  },
};

// プラグインのロードと使用
const registry = new PluginRegistry();
await registry.load(auditLogPlugin, { retentionDays: 30 });
await registry.load(ipBlockPlugin, { blockedIPs: ['192.168.0.1', '10.0.0.5'] });

// Expressミドルウェアでフックを実行
app.use(async (req, res, next) => {
  const reqCtx = createRequestContext(req);
  const modified = await registry.runHook('beforeRequest', reqCtx);
  if (!modified) return res.sendStatus(403); // プラグインが中断
  next();
});
```

---

## まとめ

Claude Codeでプラグインアーキテクチャを設計する：

1. **CLAUDE.md** にコアAPIサンドボックス（読み取りのみ）・フックシステム定義・プラグインエラーは飲み込んでコアへ波及禁止を明記
2. **PluginContext** でプラグインに提供するAPIを制限——DBへの書き込みやコアの直接参照を防ぎ、プラグインの暴走を防止
3. **依存関係グラフ** でロード順を制御——`dependencies`配列で「このプラグインが先にロードされていなければロード失敗」を保証
4. **beforeRequestがnullを返すと中断** というシンプルな規約——プラグインがリクエストをインターセプト・変換・中断できる

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
