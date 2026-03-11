---
title: "Claude CodeでHelmチャートを設計する：values.yaml・Secrets・staging/prod差分管理"
emoji: "⚓"
type: "tech"
topics: ["claudecode", "kubernetes", "devops", "helm", "typescript"]
published: true
published_at: "2026-03-13 08:00"
---

## はじめに

Kubernetesマニフェストのコピペ地獄を終わらせる——Helmチャートでstaging/prod差分を`values.yaml`で管理し、外部シークレットを安全に注入する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにHelm設計ルールを書く

```markdown
## Helmチャート設計ルール

### チャート構成
- templates/: deployment, service, ingress, hpa, configmap
- values.yaml: デフォルト値（開発用）
- values.staging.yaml: ステージング上書き
- values.production.yaml: 本番上書き

### シークレット管理
- Secretリソースはチャートに含めない
- External Secrets Operator + AWS SSM/Secrets Managerで外部注入
- .env ファイルや生の値をvalues.yamlに書かない

### イメージタグ
- latest禁止（SHA256ダイジェストかセマンティックバージョン）
- CIでimage.tagを--set image.tag=$(git rev-parse --short HEAD)で注入
```

---

## Helmチャートの生成

```
Node.jsアプリのHelmチャートを設計してください。

要件：
- Deployment + Service + Ingress
- HPA（水平スケーリング）
- External Secrets統合
- staging/prod値分離
- Helmテスト

生成ファイル: helm/myapp/
```

---

## 生成されるHelmチャート

```yaml
# helm/myapp/Chart.yaml

apiVersion: v2
name: myapp
description: Node.js application Helm chart
type: application
version: 1.2.0
appVersion: "0.0.0"  # CIで上書き

dependencies:
  - name: redis
    version: "19.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

```yaml
# helm/myapp/values.yaml — デフォルト値

replicaCount: 2

image:
  repository: ghcr.io/myorg/myapp
  pullPolicy: IfNotPresent
  tag: ""  # CIで--set image.tag=<SHA>に上書き

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - myapp.example.com

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 60
  targetMemoryUtilizationPercentage: 80

env:
  NODE_ENV: development
  LOG_LEVEL: debug
  PORT: "3000"

externalSecrets:
  enabled: true
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  secrets:
    - name: myapp-secrets
      remoteRefs:
        - remoteKey: myapp/development/database-url
          property: DATABASE_URL
        - remoteKey: myapp/development/jwt-secret
          property: JWT_SECRET

redis:
  enabled: true
  auth:
    enabled: true
    existingSecret: myapp-redis-secret
  master:
    resources:
      requests: { cpu: 50m, memory: 64Mi }
      limits: { cpu: 200m, memory: 128Mi }

healthCheck:
  livenessPath: /health/live
  readinessPath: /health/ready
  initialDelaySeconds: 15
  periodSeconds: 10

podDisruptionBudget:
  enabled: true
  minAvailable: 1
```

```yaml
# helm/myapp/values.production.yaml — 本番上書き

replicaCount: 5

image:
  pullPolicy: Always

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2000m
    memory: 1Gi

autoscaling:
  minReplicas: 5
  maxReplicas: 50

env:
  NODE_ENV: production
  LOG_LEVEL: warn

externalSecrets:
  secrets:
    - name: myapp-secrets
      remoteRefs:
        - remoteKey: myapp/production/database-url
          property: DATABASE_URL
        - remoteKey: myapp/production/jwt-secret
          property: JWT_SECRET
        - remoteKey: myapp/production/redis-url
          property: REDIS_URL

podDisruptionBudget:
  minAvailable: 2

redis:
  enabled: false  # 本番はElastiCacheを使用
```

```yaml
# helm/myapp/templates/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # 無停止デプロイ
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          envFrom:
            - secretRef:
                name: myapp-secrets  # External Secrets Operatorが作成
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: {{ .Values.healthCheck.livenessPath }}
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: {{ .Values.healthCheck.initialDelaySeconds }}
            periodSeconds: {{ .Values.healthCheck.periodSeconds }}
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: {{ .Values.healthCheck.readinessPath }}
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]  # LB drain待機
```

```yaml
# helm/myapp/templates/externalsecret.yaml

{{- if .Values.externalSecrets.enabled }}
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "myapp.fullname" . }}-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: {{ .Values.externalSecrets.secretStoreRef.name }}
    kind: {{ .Values.externalSecrets.secretStoreRef.kind }}
  target:
    name: myapp-secrets
    creationPolicy: Owner
  data:
    {{- range .Values.externalSecrets.secrets }}
    {{- range .remoteRefs }}
    - secretKey: {{ .property }}
      remoteRef:
        key: {{ .remoteKey }}
    {{- end }}
    {{- end }}
{{- end }}
```

```bash
# CI/CDデプロイコマンド

# ステージングへデプロイ
helm upgrade --install myapp ./helm/myapp \
  --namespace staging \
  --create-namespace \
  --values helm/myapp/values.yaml \
  --values helm/myapp/values.staging.yaml \
  --set image.tag=$(git rev-parse --short HEAD) \
  --wait \
  --timeout 5m

# 本番へデプロイ
helm upgrade --install myapp ./helm/myapp \
  --namespace production \
  --values helm/myapp/values.yaml \
  --values helm/myapp/values.production.yaml \
  --set image.tag=${IMAGE_TAG} \
  --atomic \  # 失敗時に自動ロールバック
  --wait \
  --timeout 10m
```

---

## まとめ

Claude CodeでHelmチャートを設計する：

1. **CLAUDE.md** にlatest禁止・Secrets外部注入・values差分管理・SHA256ダイジストタグを明記
2. **values.production.yaml** でstaging→prod差分（レプリカ数・リソース・外部サービス切替）を管理
3. **External Secrets Operator** でAWS SSM/Secrets ManagerからK8s Secretへ自動同期（値はGitに含めない）
4. **--atomic フラグ** でデプロイ失敗時に前バージョンへ自動ロールバック

---

*インフラ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
