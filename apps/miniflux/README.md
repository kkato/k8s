# Miniflux

`rss.kkato.app` で運用する個人用 RSS リーダー。

- image: [`ghcr.io/miniflux/miniflux`](https://github.com/miniflux/v2) 2.3.3
- DB: CloudNative-PG（`miniflux-db`、1 インスタンス）。接続情報は CNPG が自動生成する `miniflux-db-app` Secret を使う
- 公開: Cloudflare Tunnel Ingress
- ログイン: パスキー（WebAuthn）と Google ログインが有効。どちらもログイン後に設定画面から登録・紐付けする
  - Google アカウントでの新規ユーザー作成は無効。紐付け済みのユーザーだけがログインできる

## デプロイ前の手動準備

管理者アカウントの Secret を作成する（Git には入れない）。

```bash
kubectl create namespace miniflux
kubectl create secret generic miniflux-admin -n miniflux \
  --from-literal=username=admin \
  --from-literal=password="$(openssl rand -base64 24)"
```

パスワードは `kubectl get secret -n miniflux miniflux-admin -o jsonpath='{.data.password}' | base64 -d` で確認できる。

Google ログイン用の OAuth クライアントを [Google Cloud Console](https://console.cloud.google.com/apis/credentials) で作成し（種類: ウェブアプリケーション、承認済みのリダイレクト URI: `https://rss.kkato.app/oauth2/google/callback`）、Secret を作成する。

```bash
kubectl create secret generic miniflux-oauth2 -n miniflux \
  --from-literal=client-id=<クライアント ID> \
  --from-literal=client-secret=<クライアント シークレット>
```

## デプロイ後の確認

```bash
kubectl get cluster -n miniflux miniflux-db
kubectl get pods -n miniflux
```

ブラウザで `https://rss.kkato.app` にアクセスしてログインする。Google ログインを使うときは、パスワードでログインしたあと設定画面から Google アカウントを紐付ける。
