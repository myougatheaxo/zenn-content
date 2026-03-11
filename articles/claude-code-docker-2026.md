---
title: "Claude CodeでDockerを設計する：マルチステージビルドと本番最適化"
emoji: "🐳"
type: "tech"
topics: ["claudecode", "docker", "nodejs", "typescript", "devops"]
published: true
---

## はじめに

Dockerイメージが1GB超になる、rootユーザーで動かしてしまう、ビルドキャッシュが効かない——Claude Codeに本番品質のDockerfile設計を生成させる。

---

## CLAUDE.mdにDockerルールを書く

```markdown
## Docker設計ルール

### セキュリティ（必須）
- 本番イメージはrootユーザー禁止（専用ユーザーを作成）
- ベースイメージはslimまたはalpine（ツール最小限）
- 秘密情報をDockerfileにハードコード禁止（env変数またはSecrets）
- .dockerignoreで node_modules, .env, .git を除外

### マルチステージビルド
- ビルドステージ: devDependencies込みでビルド
- 本番ステージ: productionの依存のみ、ビルド成果物のみコピー
- 最終イメージにソースコードを含めない

### パフォーマンス
- package.jsonのCOPYとnpm installは最初に（キャッシュ効率化）
- アプリコードは最後にCOPY（キャッシュ無効化を最小限に）
- node:20-slim (debian) or node:20-alpine を使う
```

---

## Dockerfile設計の生成

```
Node.js TypeScriptアプリの本番Dockerfileを設計してください。

要件：
- マルチステージビルド（ビルド/本番）
- 非rootユーザーで実行
- キャッシュ効率の最適化
- ヘルスチェック設定
- .dockerignoreも生成

生成ファイル: Dockerfile, .dockerignore
```

---

## 生成されるDockerfile

```dockerfile
# Dockerfile

# ===== Build Stage =====
FROM node:20-slim AS builder

WORKDIR /app

# package.jsonを先にコピー（キャッシュ効率化）
COPY package.json package-lock.json ./
RUN npm ci --ignore-scripts

# ソースコードをコピーしてビルド
COPY tsconfig.json ./
COPY src ./src
COPY prisma ./prisma

# Prismaクライアント生成 + TypeScriptビルド
RUN npx prisma generate
RUN npm run build

# ===== Production Stage =====
FROM node:20-slim AS production

# セキュリティ: 非rootユーザーを作成
RUN groupadd --gid 1001 nodejs \
 && useradd --uid 1001 --gid nodejs --shell /bin/bash --create-home appuser

WORKDIR /app

# 本番依存のみインストール
COPY package.json package-lock.json ./
RUN npm ci --omit=dev --ignore-scripts \
 && npm cache clean --force

# ビルド成果物のみコピー（ソースコード不要）
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma

# 所有権を変更してからユーザー切り替え
RUN chown -R appuser:nodejs /app
USER appuser

EXPOSE 3000

# ヘルスチェック
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1))"

CMD ["node", "dist/main.js"]
```

```plaintext
# .dockerignore
node_modules
dist
.env
.env.*
.git
.gitignore
*.log
coverage
.nyc_output
.github
docs
*.md
```

---

## Docker Compose（開発環境）

```yaml
# docker-compose.yml
version: '3.9'

services:
  app:
    build:
      context: .
      target: builder   # 開発はbuilderステージを使う
    volumes:
      - ./src:/app/src   # ホットリロード用
    env_file: .env
    ports:
      - "3000:3000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: npm run dev

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
  redis_data:
```

---

## GitHub Actions CI でのビルド最適化

```yaml
# .github/workflows/docker.yml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: myapp:${{ github.sha }}
    cache-from: type=gha      # GitHub Actionsキャッシュを利用
    cache-to: type=gha,mode=max
```

---

## まとめ

Claude CodeでDockerを設計する：

1. **CLAUDE.md** に非root必須・マルチステージ必須・ハードコード禁止を明記
2. **マルチステージ** でビルドツールを本番イメージから除外
3. **package.jsonを先にCOPY** でnpm installをキャッシュ
4. **HEALTHCHECK** でコンテナの正常性をDocker/K8sが監視できる

---

*DockerfileのセキュリティレビューはCode Review Pack（¥980）の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
