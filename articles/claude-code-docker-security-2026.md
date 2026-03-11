---
title: "Claude CodeでDockerセキュリティを強化する：コンテナハードニングの実践"
emoji: "🐳"
type: "tech"
topics: ["claudecode", "docker", "security", "nodejs", "コンテナ"]
published: true
---

## はじめに

Dockerfileをとりあえず動かすために書いたら、後でセキュリティ問題が発覚した——そういう経験は珍しくない。デフォルトのDockerfileには典型的な落とし穴が3つある。

**1. rootで実行している**
コンテナ内のプロセスがrootで動くと、コンテナブレイクアウト時に即ホストを掌握される。

**2. イメージが肥大化している**
ビルドツールや開発依存が本番イメージに混入すると、攻撃面が広がる。CVEの対象パッケージ数が増え、スキャンのノイズも増える。

**3. 機密情報が混入している**
`COPY . .` でAPIキーや `.env` をまるごとコピーしていたり、`ARG SECRET_KEY` をビルドログに残していたりするケースが多い。

Claude CodeにDockerセキュリティのルールを `CLAUDE.md` に書いておくと、Dockerfile生成・レビュー・CI設定まで一貫してセキュアな出力を得られる。

---

## CLAUDE.mdにDockerセキュリティルールを書く

プロジェクトルートの `CLAUDE.md` に以下を追加する。

```markdown
## Dockerセキュリティルール

### 必須要件
- **非rootユーザー必須**: 全てのコンテナは非rootユーザーで実行すること
  - `RUN useradd -r -u 1001 appuser && chown -R appuser:appuser /app`
  - `USER appuser`
- **multi-stage build必須**: ビルド成果物のみを本番イメージにコピーする
  - `FROM node:20-alpine AS builder` → `FROM gcr.io/distroless/nodejs20-debian12`
- **.dockerignore必須**: `.env`, `*.key`, `*.pem`, `node_modules`, `.git` を除外
- **distrolessイメージ推奨**: `gcr.io/distroless/nodejs20-debian12`（シェルなし）
- **固定バージョン**: `FROM node:20.12.0-alpine3.19`（latestタグ禁止）
- **シークレット禁止**: `ARG` / `ENV` でAPIキーを渡さない。実行時マウントを使う

### セキュリティチェックリスト（Dockerfile生成時に必ず確認）
- [ ] rootで実行していないか
- [ ] ビルド依存が本番イメージに含まれていないか
- [ ] .dockerignoreが存在するか
- [ ] ベースイメージのバージョンが固定されているか
- [ ] HEALTHCHECK が定義されているか
```

---

## セキュアなDockerfileの生成プロンプト

```
Node.js 20の本番コンテナ向けDockerfileを生成してください。

要件:
- multi-stage build（builderステージでnpm ci --production、本番はdistrolessまたはalpine）
- 非rootユーザー（uid 1001）で実行
- ビルド引数にシークレットを含めない
- HEALTHCHECK を含める
- .dockerignore も生成する（.env, node_modules, .git, *.key, *.pem を除外）

アプリ: Express.js API、ポート3000
```

生成される `Dockerfile` の例：

```dockerfile
# ---- ビルドステージ ----
FROM node:20.12.0-alpine3.19 AS builder

WORKDIR /app

# 依存のみ先にコピー（レイヤーキャッシュ最適化）
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

COPY . .

# ---- 本番ステージ ----
FROM gcr.io/distroless/nodejs20-debian12

WORKDIR /app

# 非rootユーザーで実行
USER nonroot:nonroot

# ビルドステージから成果物のみコピー
COPY --from=builder --chown=nonroot:nonroot /app/node_modules ./node_modules
COPY --from=builder --chown=nonroot:nonroot /app/src ./src
COPY --from=builder --chown=nonroot:nonroot /app/package.json .

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD ["/nodejs/bin/node", "-e", "require('http').get('http://localhost:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1))"]

CMD ["src/index.js"]
```

`.dockerignore`:

```
.env
.env.*
node_modules
.git
*.key
*.pem
*.log
.DS_Store
coverage/
.nyc_output/
```

distrolessにはシェルが存在しないため、`docker exec` でのデバッグ侵入が不可能になる。本番環境には最適なセキュリティプロファイルだ。

---

## docker-compose.yamlのセキュリティ設定プロンプト

```
docker-compose.yamlにセキュリティ設定を追加してください。

対象サービス: app (Node.js API), db (PostgreSQL), redis

要件:
- read_only: trueでファイルシステムを読み取り専用に
- no-new-privilegesでsetuidバイナリの悪用を防止
- メモリ・CPU制限を設定（app: 512MB/0.5CPU、db: 1GB/1CPU）
- 不要なcapabilityを全て削除（cap_drop: ALL）
- ネットワークを分離（frontendとbackendに分離）
- シークレットはdocker secretsまたは環境変数ファイル参照
```

生成される `docker-compose.yaml` のセキュリティ設定部分：

```yaml
services:
  app:
    build: .
    read_only: true
    tmpfs:
      - /tmp
      - /app/logs  # 書き込みが必要なディレクトリのみtmpfs
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # 必要なcapabilityのみ追加
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "0.5"
    env_file:
      - .env.production  # .gitignore済みの環境変数ファイル
    networks:
      - frontend
      - backend
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16.2-alpine3.19
    read_only: true
    tmpfs:
      - /tmp
      - /var/run/postgresql
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: "1.0"
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - db_data:/var/lib/postgresql/data  # データは名前付きボリューム
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # 外部からアクセス不可

secrets:
  db_password:
    file: ./secrets/db_password.txt  # .gitignore済み

volumes:
  db_data:
```

`read_only: true` と `tmpfs` の組み合わせで、ランタイム中のファイル改ざんを防止できる。`internal: true` のネットワークでDB/Redisをインターネットから完全に隔離する。

---

## コンテナイメージのスキャン設定プロンプト

```
GitHub ActionsでDockerイメージの脆弱性スキャンを設定してください。

要件:
- trivyとgrypeの両方でスキャン
- HIGH/CRITICALの脆弱性があればCIを失敗させる
- スキャン結果をGitHub Security AlertsのSARIF形式でアップロード
- PRのコメントにサマリーを表示
- スキャン対象: ビルドしたDockerイメージ
```

生成される `.github/workflows/container-scan.yml`：

```yaml
name: Container Security Scan

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  security-events: write
  pull-requests: write

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t app:${{ github.sha }} .

      # trivyスキャン（SARIF形式でGitHub Security Alertsに連携）
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: app:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: HIGH,CRITICAL
          exit-code: "1"  # HIGH/CRITICALで失敗

      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif

      # grypeスキャン（サマリーをPRコメントに表示）
      - name: Run Grype scanner
        uses: anchore/scan-action@v3
        id: grype
        with:
          image: app:${{ github.sha }}
          severity-cutoff: high
          output-format: table

      - name: Post scan summary to PR
        uses: actions/github-script@v7
        if: github.event_name == 'pull_request'
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Container Security Scan Results\n\`\`\`\n${{ steps.grype.outputs.table }}\n\`\`\``
            })
```

trivyはSARIF形式でGitHub Security Alertsに直接連携でき、grypeはテーブル形式のPRコメントに適している。両方を使うと見逃しが減る。

---

## まとめ

Claude CodeにDockerセキュリティルールを `CLAUDE.md` で渡すことで、以下が自動化できる：

1. **Dockerfile生成**: multi-stage build + distroless + 非rootユーザーがデフォルトになる
2. **docker-compose設定**: read_only / no-new-privileges / ネットワーク分離が標準になる
3. **CI脆弱性スキャン**: trivy + grype でPRごとにHIGH/CRITICAL脆弱性を検出する

CLAUDE.mdを一度書けば、チーム全員がセキュアな設定を生成できる。知識を個人に依存させず、ルールとしてリポジトリに残すのが重要だ。

---

*Security Pack（¥1,480）の `/security-check` スキルを使うと、Dockerfileの設定ミスをローカルで自動検出できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
