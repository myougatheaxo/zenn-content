---
title: "Claude CodeでArgoCD GitOpsを設計する：自動デプロイ・差分検知・ロールバック"
emoji: "🔃"
type: "tech"
topics: ["claudecode", "kubernetes", "devops", "argocd", "gitops"]
published: true
published_at: "2026-03-13 05:00"
---

## はじめに

「本番への変更は全てGitプッシュから」——ArgoCDでGitリポジトリの状態をKubernetesに自動同期し、手動kubectlを排除するGitOpsパイプラインをClaude Codeに設計させる。

---

## CLAUDE.mdにArgoCD GitOps設計ルールを書く

```markdown
## ArgoCD GitOps設計ルール

### リポジトリ構成
- アプリコード: アプリケーションリポジトリ
- K8s設定: 独立した設定リポジトリ（config repo）
- CIがimage tagを設定リポジトリに自動コミット

### 同期ポリシー
- 開発環境: 自動同期 + 自動ヒーリング
- ステージング: 自動同期（手動プロモーション後）
- 本番: 手動同期（デプロイ承認ゲート付き）

### 差分管理
- Argo CD Image Updater: 新しいイメージタグを自動検知
- ApplicationSet: 複数環境の設定を1リソースで管理
- SyncWave: 依存関係順でリソースをデプロイ
```

---

## ArgoCD GitOpsの生成

```
ArgoCDによるGitOpsデプロイパイプラインを設計してください。

要件：
- マルチ環境（dev/staging/prod）
- 自動イメージ更新
- SyncWaveによる順序制御
- ロールバック戦略
- Slack通知

生成ファイル: gitops/, .github/workflows/
```

---

## 生成されるArgoCD GitOps設定

```yaml
# gitops/argocd/applicationset.yaml — 3環境を1リソースで管理

apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
            cluster: https://kubernetes.default.svc
            namespace: dev
            autoSync: "true"
            selfHeal: "true"
          - env: staging
            cluster: https://staging-cluster.example.com
            namespace: staging
            autoSync: "true"
            selfHeal: "false"
          - env: prod
            cluster: https://prod-cluster.example.com
            namespace: production
            autoSync: "false"
            selfHeal: "false"

  template:
    metadata:
      name: "myapp-{{env}}"
      annotations:
        notifications.argoproj.io/subscribe.on-sync-succeeded.slack: deployments
        notifications.argoproj.io/subscribe.on-sync-failed.slack: alerts
        notifications.argoproj.io/subscribe.on-health-degraded.slack: alerts
    spec:
      project: myapp

      source:
        repoURL: https://github.com/myorg/myapp-config
        targetRevision: main
        path: "environments/{{env}}"

      destination:
        server: "{{cluster}}"
        namespace: "{{namespace}}"

      syncPolicy:
        automated:
          prune: "{{autoSync}}"
          selfHeal: "{{selfHeal}}"
        syncOptions:
          - CreateNamespace=true
          - PrunePropagationPolicy=foreground
          - RespectIgnoreDifferences=true

      ignoreDifferences:
        - group: apps
          kind: Deployment
          jsonPointers:
            - /spec/replicas  # HPAが管理するためIgnore
```

```yaml
# gitops/argocd/image-updater.yaml — 自動イメージ更新

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-dev
  namespace: argocd
  annotations:
    # 最新のイメージタグを自動検知してGitにコミット
    argocd-image-updater.argoproj.io/image-list: myapp=ghcr.io/myorg/myapp
    argocd-image-updater.argoproj.io/myapp.update-strategy: newest-build
    argocd-image-updater.argoproj.io/myapp.helm.image-name: image.repository
    argocd-image-updater.argoproj.io/myapp.helm.image-tag: image.tag
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
spec:
  # ...（ApplicationSetで管理）
```

```yaml
# gitops/environments/prod/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - external-secret.yaml

images:
  - name: ghcr.io/myorg/myapp
    newTag: v1.2.3  # Image Updaterが自動更新

patches:
  - path: hpa-patch.yaml
  - path: ingress-patch.yaml

configMapGenerator:
  - name: myapp-config
    literals:
      - NODE_ENV=production
      - LOG_LEVEL=warn
```

```yaml
# gitops/base/deployment.yaml (SyncWave付き)

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    argocd.argoproj.io/sync-wave: "2"  # DBマイグレーション後にデプロイ
spec:
  # ...
---
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # まずマイグレーションを実行
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: ghcr.io/myorg/myapp:latest
          command: ["node", "dist/migrate.js"]
          envFrom:
            - secretRef:
                name: myapp-secrets
      restartPolicy: Never
  backoffLimit: 3
```

```yaml
# .github/workflows/deploy.yml — CIからGitOps設定リポジトリを更新

name: Deploy

on:
  push:
    branches: [main]
    paths: ['src/**', 'Dockerfile', 'package*.json']

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}

    steps:
      - uses: actions/checkout@v4

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/myorg/myapp
          tags: |
            type=sha,prefix=sha-
            type=semver,pattern={{version}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  update-dev:
    needs: build-and-push
    runs-on: ubuntu-latest

    steps:
      - name: Checkout config repo
        uses: actions/checkout@v4
        with:
          repository: myorg/myapp-config
          token: ${{ secrets.CONFIG_REPO_PAT }}

      - name: Update dev image tag
        run: |
          cd environments/dev
          kustomize edit set image ghcr.io/myorg/myapp:${{ needs.build-and-push.outputs.image-tag }}
          git config user.email "ci@myorg.com"
          git config user.name "CI Bot"
          git commit -am "ci: update dev image to ${{ needs.build-and-push.outputs.image-tag }}"
          git push
```

```typescript
// src/deploy/promotionGate.ts — プロモーション承認ゲート（GitHub Actions経由）

// staging → prod プロモーション時の自動チェック
export async function checkPromotionReadiness(env: 'staging' | 'prod'): Promise<{
  ready: boolean;
  checks: Array<{ name: string; passed: boolean; detail: string }>;
}> {
  const checks = await Promise.all([
    checkHealthEndpoint(`https://${env}.myapp.example.com/health`),
    checkErrorRate(env, 0.01),     // エラー率 1% 未満
    checkP99Latency(env, 500),     // P99 500ms 未満
    checkRecentTests(),            // 直近のCIテスト全通過
  ]);

  return {
    ready: checks.every(c => c.passed),
    checks,
  };
}
```

---

## まとめ

Claude CodeでArgoCD GitOpsを設計する：

1. **CLAUDE.md** に設定リポジトリ分離・dev自動同期・prod手動承認・SyncWave順序制御を明記
2. **ApplicationSet** で dev/staging/prod の3環境を1リソースで管理し、設定の重複を排除
3. **Argo CD Image Updater** が新しいイメージを検知してGitにコミット → ArgoCD が自動同期
4. **SyncWave** でDBマイグレーション(wave=1)→アプリデプロイ(wave=2)の順序を保証

---

*デプロイ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
