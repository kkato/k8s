# ArgoCD

`argocd.kkato.app`（Cloudflare Tunnel経由）で公開する。ログインは GitHub のみで、admin のパスワードログインは無効にしている。

## デプロイ前の手動準備

### GitHub OAuth App の作成

GitHub の Settings → Developer settings → OAuth Apps で作成する。

| 項目 | 値 |
|---|---|
| Homepage URL | `https://argocd.kkato.app` |
| Authorization callback URL | `https://argocd.kkato.app/api/dex/callback` |

### Client ID / Secret の登録

```bash
kubectl -n argocd create secret generic argocd-github-oauth \
  --from-literal=clientID=<Client ID> \
  --from-literal=clientSecret=<Client Secret>
kubectl -n argocd label secret argocd-github-oauth app.kubernetes.io/part-of=argocd
```

ラベルが無いと ArgoCD から読めない。

## ログインできなくなったとき

admin を一時的に有効に戻してログインする。ArgoCD の selfHeal で次の sync 時に元に戻る。

```bash
kubectl -n argocd patch cm argocd-cm -p '{"data":{"admin.enabled":"true"}}'
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```
