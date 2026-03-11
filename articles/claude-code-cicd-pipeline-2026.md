---
title: "Claude CodeでCI/CDパイプラインを設計する：GitHub Actions完全ガイド"
emoji: "🚀"
type: "tech"
topics: ["claudecode", "githubactions", "cicd", "nodejs", "devops"]
published: true
---

## はじめに: Claude CodeにCI/CD設計を任せる

CI/CDパイプラインの構築は、プロジェクトごとにゼロから考えると時間がかかる。
ワークフローファイルの文法、ジョブの依存関係、シークレットの管理——覚えることが多い。

Claude Codeは**CLAUDE.mdにルールを書いておくと、そのルールに従ったGitHub Actionsワークフローを自動生成する**。
一度ルールを定義すれば、新しいプロジェクトでも同じ品質のCI/CDをプロンプト一発で生成できる。

この記事では：
1. CLAUDE.mdへのCI/CDルール記述
2. Node.jsアプリのCIワークフロー生成
3. PRへの自動レビューコメント設定
4. 本番デプロイパイプラインのBlue-Green設計

を実際のプロンプト付きで解説する。

---

## 1. CLAUDE.mdにCI/CDルールを書く

まずプロジェクトルートの`CLAUDE.md`にCI/CDの制約を定義する。
Claude Codeはこのファイルをコンテキストとして読み込み、生成するコードに反映する。

```markdown
# CI/CD Rules

## ブランチ戦略
- mainへの直接pushは禁止。必ずPR経由
- 全PRにCIパスが必須（ci-required status check）
- feature/* → develop → main の3層構造

## パイプラインステージ（順序厳守）
1. lint       — ESLint + Prettier check
2. test       — Jest（カバレッジ80%以上必須）
3. build      — tsc + webpack/vite
4. deploy     — mainマージ時のみ、Blue-Greenで本番反映

## シークレット管理
- APIキーは全てGitHub Secretsに格納（ハードコード禁止）
- 環境変数名は SCREAMING_SNAKE_CASE
- .env.example は公開可、.env は .gitignore 必須

## デプロイ要件
- ヘルスチェックエンドポイント: GET /health → 200
- ロールバック: デプロイ失敗時は前バージョンに自動切り替え
- 通知: デプロイ成功/失敗をSlackに送信
```

このルールを書いた上で、以下のプロンプトを実行する。

---

## 2. GitHub Actionsワークフローの生成プロンプト

### Node.jsアプリのCIワークフロー

```
Node.jsアプリ（npm workspace、TypeScript）のCI用GitHub Actionsワークフローを
CLAUDE.mdのルールに従って生成してください。

要件：
- Node.js 20.x
- ステージ: lint → test（カバレッジレポート） → build
- PRへのpushとopenで起動
- キャッシュ: npm依存をnode_modules/.cacheでキャッシュ
- テスト結果: GitHub Actions Summaryに表示
- カバレッジが80%未満なら失敗
```

生成されるワークフロー（`.github/workflows/ci.yml`）の例：

```yaml
name: CI

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
      - run: npm ci
      - run: npm run lint
      - run: npm run format:check

  test:
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
      - run: npm ci
      - name: Run tests with coverage
        run: npm test -- --coverage --coverageReporters=json-summary
      - name: Coverage gate (80%)
        run: |
          COVERAGE=$(node -e "
            const s = require('./coverage/coverage-summary.json');
            console.log(s.total.lines.pct);
          ")
          echo "Line coverage: ${COVERAGE}%"
          node -e "if (${COVERAGE} < 80) { console.error('Coverage below 80%'); process.exit(1); }"
      - name: Post coverage to summary
        run: |
          echo "## Test Coverage" >> $GITHUB_STEP_SUMMARY
          echo "Line coverage: ${COVERAGE}%" >> $GITHUB_STEP_SUMMARY

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
```

---

## 3. プルリクエスト自動レビューの設定プロンプト

### PRコメントにテストカバレッジを表示

```
PRがオープン・更新されたときに、テストカバレッジの差分を
PRコメントとして自動投稿するGitHub Actionsジョブを追加してください。

要件：
- ベースブランチのカバレッジと比較して増減を表示
- カバレッジが下がった場合は ⚠️ アイコン付きで警告
- 既存コメントがあれば更新（新規追加ではなく）
- GitHub Token（secrets.GITHUB_TOKEN）を使用
```

生成されるジョブ例：

```yaml
  coverage-comment:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Get base coverage
        run: |
          git checkout ${{ github.base_ref }}
          npm ci --silent
          npm test -- --coverage --coverageReporters=json-summary 2>/dev/null || true
          BASE=$(node -e "
            try {
              const s = require('./coverage/coverage-summary.json');
              console.log(s.total.lines.pct);
            } catch { console.log('N/A'); }
          ")
          echo "BASE_COVERAGE=${BASE}" >> $GITHUB_ENV

      - name: Get PR coverage
        run: |
          git checkout ${{ github.head_ref }}
          npm ci --silent
          npm test -- --coverage --coverageReporters=json-summary
          PR=$(node -e "
            const s = require('./coverage/coverage-summary.json');
            console.log(s.total.lines.pct);
          ")
          echo "PR_COVERAGE=${PR}" >> $GITHUB_ENV

      - name: Post coverage comment
        uses: actions/github-script@v7
        with:
          script: |
            const base = parseFloat(process.env.BASE_COVERAGE) || 0;
            const pr = parseFloat(process.env.PR_COVERAGE) || 0;
            const diff = (pr - base).toFixed(1);
            const icon = pr < base ? '⚠️' : '✅';

            const body = `## ${icon} Test Coverage Report
            | | Coverage |
            |---|---|
            | Base (${context.payload.pull_request.base.ref}) | ${base}% |
            | This PR | ${pr}% |
            | Diff | ${diff > 0 ? '+' : ''}${diff}% |`;

            const comments = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            });

            const existing = comments.data.find(c =>
              c.body.includes('Test Coverage Report')
            );

            if (existing) {
              await github.rest.issues.updateComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: existing.id,
                body,
              });
            } else {
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.issue.number,
                body,
              });
            }
```

---

## 4. 本番デプロイパイプラインの生成プロンプト

### mainマージで自動デプロイ（Blue-Greenデプロイ）

```
mainブランチへのマージで本番デプロイするGitHub Actionsワークフローを
Blue-Greenデプロイ方式で生成してください。

構成：
- Node.jsアプリをDockerコンテナでデプロイ
- Blue環境（現行）とGreen環境（新バージョン）を切り替え
- ヘルスチェック（GET /health）でGreen環境を確認後に切り替え
- ヘルスチェック失敗時は自動ロールバック
- デプロイ結果（成功/失敗）をSlackに通知
```

生成されるワークフロー（`.github/workflows/deploy.yml`）の例：

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: |
          docker build -t myapp:${{ github.sha }} .
          docker tag myapp:${{ github.sha }} myapp:latest

      - name: Push to registry
        env:
          REGISTRY_TOKEN: ${{ secrets.REGISTRY_TOKEN }}
        run: |
          echo "$REGISTRY_TOKEN" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker push ghcr.io/${{ github.repository }}/myapp:${{ github.sha }}

      - name: Deploy to Green environment
        env:
          DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
          DEPLOY_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
        run: |
          echo "$DEPLOY_KEY" > /tmp/deploy_key && chmod 600 /tmp/deploy_key
          ssh -i /tmp/deploy_key -o StrictHostKeyChecking=no ubuntu@${DEPLOY_HOST} "
            docker pull ghcr.io/${{ github.repository }}/myapp:${{ github.sha }}
            docker stop myapp-green 2>/dev/null || true
            docker rm myapp-green 2>/dev/null || true
            docker run -d --name myapp-green \
              -p 3001:3000 \
              -e NODE_ENV=production \
              ghcr.io/${{ github.repository }}/myapp:${{ github.sha }}
          "

      - name: Health check Green
        env:
          DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
        run: |
          for i in {1..10}; do
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://${DEPLOY_HOST}:3001/health)
            if [ "$STATUS" = "200" ]; then
              echo "Health check passed"
              exit 0
            fi
            echo "Attempt $i: status=$STATUS, retrying..."
            sleep 5
          done
          echo "Health check failed after 10 attempts"
          exit 1

      - name: Switch traffic to Green
        if: success()
        env:
          DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
          DEPLOY_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
        run: |
          ssh -i /tmp/deploy_key ubuntu@${DEPLOY_HOST} "
            # Nginx upstream切り替え
            sed -i 's/3000/3001/' /etc/nginx/conf.d/app.conf
            nginx -s reload
            # Blue環境を停止
            docker stop myapp-blue 2>/dev/null || true
            # GreenをBlueとして再タグ
            docker rename myapp-green myapp-blue
          "

      - name: Rollback on failure
        if: failure()
        env:
          DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
          DEPLOY_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
        run: |
          ssh -i /tmp/deploy_key ubuntu@${DEPLOY_HOST} "
            docker stop myapp-green 2>/dev/null || true
            docker rm myapp-green 2>/dev/null || true
            echo 'Rolled back to previous version'
          "

      - name: Notify Slack
        if: always()
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
        run: |
          STATUS="${{ job.status }}"
          COLOR=$([ "$STATUS" = "success" ] && echo "good" || echo "danger")
          ICON=$([ "$STATUS" = "success" ] && echo ":rocket:" || echo ":x:")
          curl -s -X POST "$SLACK_WEBHOOK" \
            -H "Content-Type: application/json" \
            -d "{
              \"attachments\": [{
                \"color\": \"${COLOR}\",
                \"text\": \"${ICON} Deploy ${STATUS}: \`${{ github.sha }}\` by ${{ github.actor }}\"
              }]
            }"
```

---

## まとめ

Claude Codeを使ったCI/CD設計のポイント：

| ステップ | 内容 |
|---------|------|
| CLAUDE.md定義 | ブランチ戦略・ステージ順序・シークレット管理ルールを明文化 |
| CIワークフロー | lint→test（カバレッジ）→buildの3ステージ。キャッシュで高速化 |
| PRレビュー自動化 | カバレッジ差分コメント。既存コメントを更新するのがポイント |
| デプロイ | Blue-Greenで安全なゼロダウンタイムデプロイ。ヘルスチェック必須 |

CLAUDE.mdにルールを書いておくことで、Claude Codeは**プロジェクト固有の制約を理解した上でワークフローを生成する**。
汎用テンプレートではなく、自分のプロジェクトに合ったCIが生成される点が大きい。

---

**Code Review Pack**（¥980）の `/code-review` スキルを使えば、生成したCI設定ファイル自体の漏れ・セキュリティリスクも自動検出できます。

👉 https://prompt-works.jp
