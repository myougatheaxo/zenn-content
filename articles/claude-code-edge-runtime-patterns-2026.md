---
title: "Edge Functions実践：Cloudflare Workers・Vercel Edgeの設計パターン2026"
emoji: "⚡"
type: "tech"
topics:
  - claudecode
  - cloudflare
  - edge
  - typescript
published: true
---

## Edge Functionsを選ぶべき場面

Edge Functionsは、ユーザーに最も近いロケーションでコードを実行する仕組みだ。従来のサーバーレスと違い、コールドスタートがほぼゼロで、グローバルに低レイテンシを実現できる。

しかしEdge Runtimeには制約がある。Node.js APIの多くが使えない。`fs`・`child_process` などは動作しない。SQLiteのような従来DBへの直接接続もできない。

**Edgeが向いているケース**:
- 認証チェック・リダイレクト（ユーザーに近い処理）
- A/Bテスト・フィーチャーフラグ
- キャッシュレイヤー・CDN前段処理
- 軽量なAPI（設定・メタデータ取得）

**通常のサーバーが向いているケース**:
- DB直接アクセス（大量クエリ・トランザクション）
- ファイル処理・大量計算
- Node.js固有ライブラリ利用

## Cloudflare Workers + Hono：型安全なルーティング

```typescript
// src/index.ts - Cloudflare Workers
import { Hono } from "hono";
import { cors } from "hono/cors";
import { jwt } from "hono/jwt";

interface Env {
  KV: KVNamespace;
  DB: D1Database;
  API_SECRET: string;
}

const app = new Hono<{ Bindings: Env }>();

app.use("*", cors({ origin: ["https://myapp.com"] }));
app.use("/api/*", jwt({ secret: (c) => c.env.API_SECRET }));

// KVキャッシュを使ったAPIルート
app.get("/api/config/:key", async (c) => {
  const key = c.req.param("key");

  // KVからキャッシュを確認
  const cached = await c.env.KV.get(`config:${key}`, "json");
  if (cached) {
    return c.json({ data: cached, source: "cache" });
  }

  // D1からデータ取得
  const result = await c.env.DB.prepare(
    "SELECT value FROM configs WHERE key = ?"
  ).bind(key).first<{ value: string }>();

  if (!result) {
    return c.json({ error: "Not found" }, 404);
  }

  const data = JSON.parse(result.value);
  // KVに1時間キャッシュ
  await c.env.KV.put(`config:${key}`, JSON.stringify(data), {
    expirationTtl: 3600,
  });

  return c.json({ data, source: "db" });
});

export default app;
```

## Vercel Edge Middleware：認証とリダイレクト

```typescript
// middleware.ts (Next.js App Router)
import { NextRequest, NextResponse } from "next/server";
import { jwtVerify } from "jose";

const PUBLIC_PATHS = ["/", "/login", "/signup", "/api/auth"];

export async function middleware(req: NextRequest) {
  const { pathname } = req.nextUrl;

  if (PUBLIC_PATHS.some((p) => pathname.startsWith(p))) {
    return NextResponse.next();
  }

  const token = req.cookies.get("auth-token")?.value;
  if (!token) {
    const loginUrl = new URL("/login", req.url);
    loginUrl.searchParams.set("redirect", pathname);
    return NextResponse.redirect(loginUrl);
  }

  try {
    const secret = new TextEncoder().encode(process.env.JWT_SECRET!);
    const { payload } = await jwtVerify(token, secret);

    if (pathname.startsWith("/admin") && payload.role !== "admin") {
      return NextResponse.redirect(new URL("/403", req.url));
    }

    const response = NextResponse.next();
    response.headers.set("x-user-id", String(payload.sub));
    response.headers.set("x-user-role", String(payload.role));
    return response;
  } catch {
    const response = NextResponse.redirect(new URL("/login", req.url));
    response.cookies.delete("auth-token");
    return response;
  }
}

export const config = {
  matcher: [
    "/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)",
  ],
};
```

## A/Bテストをエッジで実装する

```typescript
const AB_TESTS: Record<string, { variants: string[]; paths: string[] }> = {
  "checkout-flow": {
    variants: ["control", "simplified"],
    paths: ["/checkout"],
  },
};

function getVariant(testId: string, userId: string): string {
  const test = AB_TESTS[testId];
  if (!test) return "control";
  const hash = Array.from(`${testId}:${userId}`).reduce(
    (acc, char) => (acc * 31 + char.charCodeAt(0)) & 0x7fffffff,
    0
  );
  return test.variants[hash % test.variants.length];
}

export async function middleware(req: NextRequest) {
  const response = NextResponse.next();
  const userId = req.cookies.get("user-id")?.value || crypto.randomUUID();

  for (const [testId, test] of Object.entries(AB_TESTS)) {
    if (test.paths.some((p) => req.nextUrl.pathname === p)) {
      const variant = getVariant(testId, userId);
      response.headers.set(`x-ab-${testId}`, variant);
    }
  }

  if (!req.cookies.get("user-id")) {
    response.cookies.set("user-id", userId, {
      maxAge: 60 * 60 * 24 * 365,
      httpOnly: true,
      sameSite: "lax",
    });
  }

  return response;
}
```

## Cloudflare Durable Objects：エッジでのステート管理

```typescript
export class DocumentRoom {
  private storage: DurableObjectStorage;
  private sessions: Set<WebSocket> = new Set();

  constructor(state: DurableObjectState) {
    this.storage = state.storage;
  }

  async fetch(request: Request): Promise<Response> {
    if (request.headers.get("Upgrade") === "websocket") {
      return this.handleWebSocket();
    }
    const doc = (await this.storage.get("document")) ?? "";
    return Response.json({ document: doc });
  }

  private async handleWebSocket(): Promise<Response> {
    const pair = new WebSocketPair();
    const [client, server] = Object.values(pair);
    server.accept();
    this.sessions.add(server);

    const currentDoc = (await this.storage.get("document")) ?? "";
    server.send(JSON.stringify({ type: "init", content: currentDoc }));

    server.addEventListener("message", async (event) => {
      const data = JSON.parse(event.data as string);
      if (data.type === "update") {
        await this.storage.put("document", data.content);
        for (const session of this.sessions) {
          if (session !== server && session.readyState === 1) {
            session.send(JSON.stringify({ type: "update", content: data.content }));
          }
        }
      }
    });

    server.addEventListener("close", () => this.sessions.delete(server));
    return new Response(null, { status: 101, webSocket: client });
  }
}
```

## エッジでのLLMストリーミング呼び出し

```typescript
// Vercel Edge RouteからClaudeをストリーミング呼び出し
import Anthropic from "@anthropic-ai/sdk";

export const runtime = "edge";

const client = new Anthropic();

export async function POST(req: Request) {
  const { messages } = await req.json();

  const stream = await client.messages.create({
    model: "claude-haiku-4-5",  // エッジでは軽量モデルを優先
    max_tokens: 1024,
    messages,
    stream: true,
  });

  const readableStream = new ReadableStream({
    async start(controller) {
      for await (const event of stream) {
        if (
          event.type === "content_block_delta" &&
          event.delta.type === "text_delta"
        ) {
          controller.enqueue(new TextEncoder().encode(event.delta.text));
        }
      }
      controller.close();
    },
  });

  return new Response(readableStream, {
    headers: { "Content-Type": "text/plain; charset=utf-8" },
  });
}
```

## wrangler.toml：Cloudflare Workers設定

```toml
# wrangler.toml
name = "my-edge-api"
main = "src/index.ts"
compatibility_date = "2026-01-01"
compatibility_flags = ["nodejs_compat"]

[[kv_namespaces]]
binding = "KV"
id = "xxxx"

[[d1_databases]]
binding = "DB"
database_name = "my-db"
database_id = "yyyy"

[vars]
ENVIRONMENT = "production"

[[routes]]
pattern = "api.myapp.com/*"
zone_name = "myapp.com"
```

Edge Functionsの設計の核心は「Edgeに持ち込む処理を最小化し、Nodeが必要な処理はオリジンサーバーに委ねる」ことだ。認証・キャッシュ・A/Bテストという薄いレイヤーに集中することで、エッジの低レイテンシ特性を最大限に活かせる。

---

この記事の内容は、Claude Code完全攻略ガイド（全7章）の一部。CLAUDE.md設計、Hooks実践、MCPセットアップ、マルチエージェント構成まで全7章にまとめた完全版はnoteで公開している。

https://note.com/myougatheaxo/n/ncf4f814d6b68

*みょうが (@myougaTheAxo) ― ウーパールーパーのVTuber。AIツールの実践的な使い方を発信中。*
