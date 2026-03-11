---
title: "Claude CodeでDockerコンテナセキュリティを設計する：イメージスキャン・非root・最小権限"
emoji: "🐳"
type: "tech"
topics: ["claudecode", "docker", "security", "devops", "typescript"]
published: true
published_at: "2026-03-13 15:00"
---

## はじめに

Dockerイメージに脆弱性が入り込むのを防ぐ——Trivy・Dockerの非rootユーザー・読み取り専用ファイルシステムでコンテナのセキュリティを強化する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにコンテナセキュリティ設計ルールを書く

```markdown
## Dockerコンテナセキュリティ設計ルール

### Dockerfile
- ベースイメージ: node:22-alpine（最小サイズ）
- 非rootユーザー: USER node（UID 1000）
- 読み取り専用ファイルシステム: --read-only起動
- Multi-stage build: 開発依存を本番に含めない

### イメージスキャン
- CIでTrivyスキャン: HIGH/CRITICAL脆弱性でブロック
- ベースイメージは週次で更新（自動PR）
- Dockerfileのベストプラクティス: Hadolintでリント

### Runtime設定
- Capabilities: --cap-drop ALL（全capabilities削除）
- no-new-privileges: --security-opt no-new-privileges
- Seccomp: Dockerデフォルトプロファイルを適用
```

---

## コンテナセキュリティの生成

```
Node.jsアプリのセキュアなDockerfile設定を設計してください。

要件：
- Multi-stage build
- 非rootユーザー
- 脆弱性スキャン（Trivy）
- CI/CDパイプライン統合
- Runtime最小権限

生成ファイル: Dockerfile, .github/workflows/
```

---

## 生成されるセキュアなDockerfile

```dockerfile
# Dockerfile — セキュアなNode.js本番イメージ

# ===========================
# Stage 1: Dependencies
# ===========================
FROM node:22-alpine AS deps
WORKDIR /app

# パッケージファイルのみコピー（レイヤーキャッシュ最適化）
COPY package*.json ./
RUN npm ci --only=production

# ===========================
# Stage 2: Builder
# ===========================
FROM node:22-alpine AS builder
WORKDIR /app

# 開発依存を含む全パッケージをインストール
COPY package*.json ./
RUN npm ci

# ソースをコピーしてビルド
COPY . .
RUN npm run build

# ===========================
# Stage 3: Production (最小イメージ)
# ===========================
FROM node:22-alpine AS production
WORKDIR /app

# セキュリティパッチ適用（alpineの場合）
RUN apk upgrade --no-cache

# 本番依存のみコピー（開発用パッケージを除外）
COPY --from=deps   /app/node_modules ./node_modules
COPY --from=builder /app/dist        ./dist
COPY --from=builder /app/package.json ./

# 非rootユーザー設定（UID:GID = 1000:1000）
# node:alpineにはデフォルトでnodeユーザーが存在
RUN chown -R node:node /app
USER node

# アプリが書き込む一時ディレクトリのみwritable
VOLUME ["/app/tmp"]

# ポート
EXPOSE 3000

# ヘルスチェック
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1))"

# エントリポイント
CMD ["node", "dist/server.js"]
```

```yaml
# .github/workflows/container-security.yml

name: Container Security Scan

on:
  push:
    paths:
      - 'Dockerfile'
      - 'package*.json'
      - 'src/**'

jobs:
  security-scan:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          exit-code: '1'              # HIGH/CRITICALで失敗
          ignore-unfixed: true        # fixが出ていない脆弱性は無視
          severity: 'HIGH,CRITICAL'

      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Lint Dockerfile with Hadolint
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile
          failure-threshold: warning
```

```yaml
# docker-compose.yml — 本番セキュリティ設定

version: '3.8'
services:
  app:
    image: myapp:latest
    user: "1000:1000"        # 非rootユーザー
    read_only: true          # 読み取り専用ファイルシステム
    tmpfs:
      - /tmp:size=100m       # tmpのみ書き込み可
      - /app/tmp:size=100m
    cap_drop:
      - ALL                   # 全capabilities削除
    cap_add:
      - NET_BIND_SERVICE      # ポート80/443バインドに必要（3000以上なら不要）
    security_opt:
      - no-new-privileges:true
    environment:
      NODE_ENV: production
    ports:
      - "3000:3000"
    restart: unless-stopped
```

```typescript
// src/security/secretsValidation.ts — 起動時のシークレット検証

// 必須シークレットが設定されているか起動時に確認
export function validateRequiredSecrets(): void {
  const required = [
    'DATABASE_URL',
    'REDIS_URL',
    'JWT_SECRET',
    'SESSION_SECRET',
  ];

  const missing = required.filter(key => !process.env[key]);

  if (missing.length > 0) {
    throw new Error(
      `Missing required environment variables: ${missing.join(', ')}\n` +
      'Container will not start without these secrets.'
    );
  }

  // JWT_SECRETが短すぎる場合も起動拒否
  if ((process.env.JWT_SECRET?.length ?? 0) < 32) {
    throw new Error('JWT_SECRET must be at least 32 characters');
  }

  logger.info('All required secrets validated');
}

// server.ts の起動時に呼び出す
validateRequiredSecrets();
const server = app.listen(PORT, () => console.log(`Server running on ${PORT}`));
```

---

## 週次ベースイメージ自動更新

```yaml
# .github/workflows/base-image-update.yml
name: Weekly Base Image Update

on:
  schedule:
    - cron: '0 9 * * MON'  # 毎週月曜午前9時

jobs:
  update-base-image:
    steps:
      - uses: actions/checkout@v4

      - name: Check for base image updates
        run: |
          # 現在のベースイメージのダイジェストを確認
          CURRENT=$(grep 'FROM node:22-alpine' Dockerfile | head -1)
          LATEST=$(docker pull node:22-alpine 2>&1 | grep Digest | awk '{print $2}')
          echo "LATEST_DIGEST=$LATEST" >> $GITHUB_ENV

      - name: Create PR if update available
        uses: peter-evans/create-pull-request@v5
        with:
          commit-message: 'chore: update node:22-alpine base image'
          title: 'Security: Update base image node:22-alpine'
          body: 'Weekly base image update with latest security patches.'
```

---

## まとめ

Claude CodeでDockerコンテナセキュリティを設計する：

1. **CLAUDE.md** にalpine使用・非rootユーザー・読み取り専用FS・CIでTrivyスキャンを明記
2. **Multi-stage build** で本番イメージから開発依存・ソースを除外（イメージサイズ最小化）
3. **cap_drop ALL** で全Linux Capabilitiesを削除（必要なものだけをcap_addで追加）
4. **週次ベースイメージ自動PR** でセキュリティパッチを継続的に適用

---

*コンテナセキュリティ設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
