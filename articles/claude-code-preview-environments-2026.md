---
title: "Claude Codeでプレビュー環境を設計する：PRごとの一時K8s Namespace・自動クリーンアップ"
emoji: "🔬"
type: "tech"
topics: ["claudecode", "kubernetes", "devops", "typescript", "github"]
published: true
published_at: "2026-03-14 17:00"
---

## はじめに

「このPRどう動くか見たい」——PRが作られると自動でKubernetesのNamespaceが作られ、マージ後に自動削除されるプレビュー環境をClaude Codeに設計させる。

---

## CLAUDE.mdにプレビュー環境設計ルールを書く

```markdown
## プレビュー環境設計ルール

### Namespace命名規則
- preview-pr-{PR番号}（例: preview-pr-123）
- PR番号はGitHub Actionsから取得
- 最大文字数: 63文字（K8sの制限）

### リソース制限（コスト管理）
- CPU: request 50m / limit 500m
- Memory: request 64Mi / limit 256Mi
- 最大生存期間: 7日（自動削除）

### クリーンアップ
- PRマージ/クローズ: GitHub Actions で即座に削除
- 孤立したNamespaceは毎晩CronJobで掃除
- DB: 本番DBには接続しない（専用SQLiteかin-memory）
```

---

## プレビュー環境の生成

```
PRごとのプレビュー環境自動構築を設計してください。

要件：
- GitHub Actions連携
- Kubernetes Namespace自動作成
- PR URL自動コメント
- マージ/クローズ時に自動削除
- リソース制限

生成ファイル: .github/workflows/, k8s/preview/
```

---

## 生成されるプレビュー環境実装

```yaml
# .github/workflows/preview-deploy.yml

name: Preview Environment

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches: [main]

env:
  PREVIEW_NAMESPACE: preview-pr-${{ github.event.pull_request.number }}
  IMAGE_TAG: pr-${{ github.event.pull_request.number }}-${{ github.sha }}

jobs:
  deploy-preview:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      contents: read
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-northeast-1

      - name: Build and push image
        run: |
          docker build -t ${{ secrets.ECR_REGISTRY }}/myapp:${{ env.IMAGE_TAG }} .
          docker push ${{ secrets.ECR_REGISTRY }}/myapp:${{ env.IMAGE_TAG }}

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubeconfig
        run: aws eks update-kubeconfig --name myapp-cluster

      - name: Create preview namespace
        run: |
          kubectl create namespace ${{ env.PREVIEW_NAMESPACE }} --dry-run=client -o yaml | kubectl apply -f -

          # ResourceQuota でコスト制限
          cat <<EOF | kubectl apply -f -
          apiVersion: v1
          kind: ResourceQuota
          metadata:
            name: preview-quota
            namespace: ${{ env.PREVIEW_NAMESPACE }}
          spec:
            hard:
              requests.cpu: "500m"
              requests.memory: "512Mi"
              limits.cpu: "2"
              limits.memory: "1Gi"
              pods: "10"
          EOF

      - name: Deploy to preview namespace
        run: |
          helm upgrade --install myapp ./helm/myapp \
            --namespace ${{ env.PREVIEW_NAMESPACE }} \
            --values helm/myapp/values.yaml \
            --values helm/myapp/values.preview.yaml \
            --set image.tag=${{ env.IMAGE_TAG }} \
            --set ingress.hosts[0].host=pr-${{ github.event.pull_request.number }}.preview.myapp.example.com \
            --set env.DATABASE_URL=sqlite:///tmp/preview.db \
            --set env.PR_NUMBER=${{ github.event.pull_request.number }} \
            --wait --timeout 3m

      - name: Get preview URL
        id: preview-url
        run: |
          echo "url=https://pr-${{ github.event.pull_request.number }}.preview.myapp.example.com" >> $GITHUB_OUTPUT

      - name: Comment preview URL on PR
        uses: actions/github-script@v7
        with:
          script: |
            const url = '${{ steps.preview-url.outputs.url }}';
            const body = `## Preview Environment

            **Preview URL**: ${url}

            | Item | Value |
            |------|-------|
            | Namespace | \`${{ env.PREVIEW_NAMESPACE }}\` |
            | Image Tag | \`${{ env.IMAGE_TAG }}\` |
            | Auto-cleanup | On PR close/merge |

            > Preview environments are automatically deleted when this PR is closed or merged.`;

            // 既存のコメントを更新、なければ作成
            const comments = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            });

            const existing = comments.data.find(c =>
              c.user.login === 'github-actions[bot]' && c.body.includes('Preview Environment')
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

  cleanup-preview:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest

    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-northeast-1

      - name: Configure kubeconfig
        run: aws eks update-kubeconfig --name myapp-cluster

      - name: Delete preview namespace
        run: |
          kubectl delete namespace ${{ env.PREVIEW_NAMESPACE }} --ignore-not-found=true
          echo "Preview namespace ${{ env.PREVIEW_NAMESPACE }} deleted"
```

```yaml
# k8s/preview/values.preview.yaml — プレビュー用Helm値

replicaCount: 1

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 500m
    memory: 256Mi

autoscaling:
  enabled: false  # プレビューはスケーリング不要

env:
  NODE_ENV: preview
  LOG_LEVEL: debug
  FEATURE_FLAGS: "all_enabled"  # プレビューでは全フィーチャーフラグON

redis:
  enabled: false  # in-memory fallback使用

ingress:
  annotations:
    # 7日後に自動削除するアノテーション
    preview.myapp.com/expires-at: ""  # CIが動的に設定

podAnnotations:
  preview.myapp.com/pr-number: ""
  preview.myapp.com/created-at: ""
```

```typescript
// scripts/cleanup-stale-previews.ts — 孤立Namespaceの定期クリーンアップ

import { KubeConfig, CoreV1Api } from '@kubernetes/client-node';

const PREVIEW_PREFIX = 'preview-pr-';
const MAX_AGE_DAYS = 7;

async function cleanupStalePreviewEnvironments(): Promise<void> {
  const kc = new KubeConfig();
  kc.loadFromDefault();
  const k8sApi = kc.makeApiClient(CoreV1Api);

  const { body: nsList } = await k8sApi.listNamespace();
  const previewNamespaces = nsList.items.filter(ns =>
    ns.metadata?.name?.startsWith(PREVIEW_PREFIX)
  );

  const now = new Date();
  let cleaned = 0;

  for (const ns of previewNamespaces) {
    const createdAt = new Date(ns.metadata!.creationTimestamp!);
    const ageDays = (now.getTime() - createdAt.getTime()) / (1000 * 60 * 60 * 24);

    if (ageDays > MAX_AGE_DAYS) {
      const name = ns.metadata!.name!;
      await k8sApi.deleteNamespace(name);
      logger.info({ name, ageDays: ageDays.toFixed(1) }, 'Stale preview namespace deleted');
      cleaned++;
    }
  }

  logger.info({ total: previewNamespaces.length, cleaned }, 'Preview cleanup completed');
}

cleanupStalePreviewEnvironments().catch(console.error);
```

---

## まとめ

Claude CodeでPRプレビュー環境を設計する：

1. **CLAUDE.md** にpreview-pr-{番号}命名・7日自動削除・CPU/メモリ制限・本番DB非接続を明記
2. **ResourceQuota** でNamespaceごとのリソース上限を設定——コスト爆発を防止
3. **PRコメントの更新** で既存コメントがある場合は新規作成せず更新（コメント氾濫防止）
4. **孤立Namespace掃除** CronJobで7日超えたプレビューを自動削除、GitHub Actionsが失敗した場合の保険

---

*デプロイ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
