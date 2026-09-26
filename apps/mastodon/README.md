# Mastodon

`social.kkato.app` で運用する個人用 Mastodon サーバー。

- chart: [`mastodon/helm-charts`](https://github.com/mastodon/helm-charts) 1.2.2（appVersion v4.7.2）
- DB: CloudNative-PG（`mastodon-db`）
- メディア: Cloudflare R2（S3 互換）
- 登録: クローズ（シングルユーザー）
- 全文検索: 有効化済み（自前 Elasticsearch, 3ノードクラスタ）

## v4.6.6 → v4.7.2 の更新

[v4.7.0](https://github.com/mastodon/mastodon/releases/tag/v4.7.0) と
[v4.7.2](https://github.com/mastodon/mastodon/releases/tag/v4.7.2) の更新手順に従う。
mainへのpushでArgoCDの自動syncが始まるため、事前にDBバックアップを取得する。
4.7では大規模なDBマイグレーションがあり、データ量によっては長時間かかる。

既存DBの更新では `mastodon.hooks.dbPrepare.enabled: false` とし、
ArgoCDが `pre-install` hookの `db:prepare` を再実行することを防ぐ。
`dbMigrate.enabled: true` により、以下の順序で実行される:

1. `mastodon-db-pre-migrate`: 更新前のマイグレーション（`SKIP_POST_DEPLOYMENT_MIGRATIONS=true`）
2. web / streaming / Sidekiqをv4.7.2へ更新
3. `mastodon-db-migrate`: 更新後のマイグレーション

sync完了後はArgoCDの `Synced` / `Healthy` と、各Deploymentのイメージを確認する。
hook Jobは成功すると削除される。新規環境への初回導入時のみ `dbPrepare.enabled: true` に戻す。

## ディレクトリ構成

```
apps/mastodon/
├── values.yaml          # Helm values（ArgoCD ref: values で参照）
└── resources/           # 素マニフェスト（ArgoCD directory Application）
    ├── cluster.yaml     # CNPG Cluster（PostgreSQL）
    ├── redis.yaml       # Redis Deployment + Service + PVC
    ├── elasticsearch.yaml  # Elasticsearch StatefulSet(3 replica) + Service + Headless Service（全文検索用）
    ├── hpa.yaml         # HorizontalPodAutoscaler（web / streaming / sidekiq-default）
    ├── vpa.yaml         # VerticalPodAutoscaler - updateMode: Off（推奨値算出のみ）
    ├── external-secret-db.yaml       # mastodon-db Secret（DB 認証情報）
    ├── external-secret-secrets.yaml  # mastodon-secrets Secret（app secrets）
    └── external-secret-s3.yaml       # mastodon-s3 Secret（R2 認証情報）
```

## デプロイ前の手動準備

### 1. Cloudflare R2 バケット作成

Cloudflare ダッシュボードで以下を行う:

1. R2 バケットを作成（例: `mastodon-media`）
2. 「R2 API トークン」を発行（S3 互換 API 用）
3. （任意）メディア用カスタムドメインを割り当て（例: `media.social.kkato.app`）

### 2. values.yaml の R2 設定を更新

`apps/mastodon/values.yaml` の以下を実際の値に変更する:

```yaml
mastodon:
  s3:
    bucket: mastodon-media  # 実際のバケット名
    endpoint: https://<account-id>.r2.cloudflarestorage.com  # R2 エンドポイント
    aliasHost: ""  # カスタムドメインを設定する場合は記入
```

### 3. GCP Secret Manager にシークレットを登録

```bash
# 1. DB パスワード（任意の強パスワード）
gcloud secrets create mastodon-db-password --data-file=- <<< "<your-db-password>"

# 2. SECRET_KEY_BASE
SECRET_KEY_BASE=$(docker run --rm ghcr.io/mastodon/mastodon:v4.7.2 bin/rails secret)
gcloud secrets create mastodon-secret-key-base --data-file=- <<< "$SECRET_KEY_BASE"

# 3. VAPID 鍵ペア
VAPID=$(docker run --rm ghcr.io/mastodon/mastodon:v4.7.2 bin/rails mastodon:webpush:generate_vapid_key)
VAPID_PRIVATE=$(echo "$VAPID" | grep VAPID_PRIVATE_KEY | cut -d= -f2)
VAPID_PUBLIC=$(echo "$VAPID" | grep VAPID_PUBLIC_KEY | cut -d= -f2)
gcloud secrets create mastodon-vapid-private-key --data-file=- <<< "$VAPID_PRIVATE"
gcloud secrets create mastodon-vapid-public-key --data-file=- <<< "$VAPID_PUBLIC"

# 4. ActiveRecord Encryption 鍵
ARE=$(docker run --rm ghcr.io/mastodon/mastodon:v4.7.2 bin/rails db:encryption:init)
ARE_PRIMARY=$(echo "$ARE" | grep primary_key | awk '{print $2}')
ARE_DETER=$(echo "$ARE" | grep deterministic_key | awk '{print $2}')
ARE_SALT=$(echo "$ARE" | grep key_derivation_salt | awk '{print $2}')
gcloud secrets create mastodon-are-primary-key --data-file=- <<< "$ARE_PRIMARY"
gcloud secrets create mastodon-are-deterministic-key --data-file=- <<< "$ARE_DETER"
gcloud secrets create mastodon-are-derivation-salt --data-file=- <<< "$ARE_SALT"

# 5. Cloudflare R2 認証情報
gcloud secrets create mastodon-r2-access-key-id --data-file=- <<< "<r2-access-key-id>"
gcloud secrets create mastodon-r2-secret-access-key --data-file=- <<< "<r2-secret-access-key>"
```

### 4. Cloudflare Tunnel にルーティングを追加

既存トンネル（`kkato.app`）のダッシュボードで `social.kkato.app` を追加する。

## デプロイ後の確認

```bash
# ArgoCD の sync 状態
kubectl get applications -n argocd

# Pod の起動確認
kubectl get pods -n mastodon

# CNPG クラスタの状態
kubectl get cluster -n mastodon mastodon-db

# 管理者アカウントの初期パスワードをログで確認
kubectl logs -n mastodon job/mastodon-create-admin
```

ブラウザで `https://social.kkato.app` にアクセスしてログインする。

## オートスケール（HPA / VPA）

### 構成

| workload | HPA | VPA |
|---|---|---|
| `mastodon-web` | CPU 70%、min 3 / max 6 | Off（推奨値算出のみ） |
| `mastodon-streaming` | CPU 70%、min 3 / max 6 | Off（推奨値算出のみ） |
| `mastodon-sidekiq-default` | CPU 75%、min 3 / max 8 | Off（推奨値算出のみ） |
| `mastodon-sidekiq-scheduler` | 対象外（常に 1 固定） | Off（推奨値算出のみ） |

VPA の `updateMode: "Off"` は Pod を自動リサイズしない。`kubectl describe vpa` で推奨値を確認し、`values.yaml` の resources を手動で調整する運用。

### 確認コマンド

```bash
# HPA の状態（TARGETS に実 % が表示されていれば metrics-server 正常）
kubectl get hpa -n mastodon

# VPA の推奨値を確認（recommender が数分後に算出）
kubectl describe vpa -n mastodon

# Pod のリソース使用量
kubectl top pods -n mastodon
```

### VPA 推奨値を反映する手順

1. `kubectl describe vpa mastodon-web -n mastodon` で `target` の cpu / memory を確認
2. `apps/mastodon/values.yaml` の `mastodon.web.resources.requests` を推奨値に更新して push
3. ArgoCD が自動 sync し、Deployment がローリングアップデートされる

### 前提 addon

- **metrics-server**（namespace: `kube-system`）: Metrics API を提供。HPA / VPA 両方の前提
- **vertical-pod-autoscaler**（namespace: `vpa`）: recommender のみ起動（updater / admissionController は無効）

## SMTP の追加（後から設定する場合）

`apps/mastodon/values.yaml` に以下を追加して push する:

```yaml
mastodon:
  smtp:
    server: smtp.example.com
    port: 587
    existingSecret: mastodon-smtp  # keys: login, password
    from_address: notifications@social.kkato.app
```

GCP Secret Manager にも `mastodon-smtp-login` / `mastodon-smtp-password` を登録し、
`resources/external-secret-secrets.yaml`（または新規ファイル）に ExternalSecret を追加する。

## 全文検索（Elasticsearch）

自前の Elasticsearch 3ノードクラスタ（`resources/elasticsearch.yaml`）を `mastodon` namespace 内にデプロイし、
`values.yaml` の `elasticsearch.enabled: true` / `hostname: mastodon-es` / `port: 9200` / `preset: small_cluster` で接続している。

Mastodon本体は複数ホスト指定に対応していないため、`mastodon-es` Service（非headless）が3 Pod へロードバランスする単一エンドポイントとして機能する。
discovery 用に `mastodon-es-headless`（clusterIP: None）を別途用意し、StatefulSet の各 Pod (`mastodon-es-0/1/2`) が専用の PVC を持つ。

### インデックスの作成・再作成

初回デプロイ後、またはスキーマ変更後は以下でインデックスを作成する:

```bash
kubectl exec -n mastodon deploy/mastodon-web -- bin/tootctl search deploy
```

### 状態確認

```bash
# Pod / PVC の起動確認
kubectl get pods -n mastodon -l app=mastodon-es
kubectl get pvc -n mastodon -l app=mastodon-es

# クラスタの状態確認（3ノード認識・status: green を期待）
kubectl exec -n mastodon mastodon-es-0 -- curl -s localhost:9200/_cluster/health

# インデックス状態確認
kubectl exec -n mastodon deploy/mastodon-web -- bin/tootctl search deploy --only=accounts,statuses,tags --only-if-required
```
