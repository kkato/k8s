# Miniflux

`rss.kkato.app` で運用する個人用 RSS リーダー。

- image: [`ghcr.io/miniflux/miniflux`](https://github.com/miniflux/v2) 2.3.3
- DB: CloudNative-PG（`miniflux-db`、1 インスタンス）。接続情報は CNPG が自動生成する `miniflux-db-app` Secret を使う
- 公開: Cloudflare Tunnel Ingress
- ログイン: パスキー（WebAuthn）有効。ログイン後に設定画面から登録する

## デプロイ前の手動準備

管理者アカウントの Secret を作成する（Git には入れない）。

```bash
kubectl create namespace miniflux
kubectl create secret generic miniflux-admin -n miniflux \
  --from-literal=username=admin \
  --from-literal=password="$(openssl rand -base64 24)"
```

パスワードは `kubectl get secret -n miniflux miniflux-admin -o jsonpath='{.data.password}' | base64 -d` で確認できる。

## デプロイ後の確認

```bash
kubectl get cluster -n miniflux miniflux-db
kubectl get pods -n miniflux
```

ブラウザで `https://rss.kkato.app` にアクセスしてログインする。
