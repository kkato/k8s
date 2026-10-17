# radipod

「安住紳一郎の日曜天国」を録音し、外出先のAntennaPodからストリーミング再生する。

- RSS: `https://radipod.kkato.app/feed.xml`
- 表示名は「radipod｜日曜天国」。専用カバーは `configmap.yaml` の `image_url` で指定する。
- 起動時と30分ごとに過去7放送日を確認し、放送済みの未録音回を取得する。
- 録音はPVC `radipod-data` に保存。Pod再作成でも残る。
- Cloudflare TunnelによるHTTPS配信。RSSと音声の両方にBasic認証が必要。

AntennaPodにRSSとユーザー名・パスワードを登録し、ダウンロードせずにストリーミング再生する。必要に応じてモバイル回線での再生も許可する。[認証の対応について](https://forum.antennapod.org/t/basic-authentication-support/7404)

## 管理

Argo CDが `resources/` を自動反映する。イメージは[radipod](https://github.com/kkato/radipod)のGitHub ActionsでGHCRへ作成し、DeploymentのタグをコミットSHAに固定する。

初回デプロイ前にSecret `radipod-http-auth` を作成する。キーは `RADIPOD_HTTP_USER` と `RADIPOD_HTTP_PASSWORD`。認証情報はGitに含めない。Secretを変更した場合もPodの再起動が必要。

```sh
kubectl -n radipod logs deployment/radipod --tail=100
kubectl -n radipod get pods,pvc,svc
```

録音対象・URLは `resources/configmap.yaml` を編集してpushし、Argo CDの同期後に `kubectl -n radipod rollout restart deployment/radipod` で反映する。

保存先はOpenEBSのノードローカルディスク。自動削除・バックアップは未設定なので、必要な音声は別途バックアップする。PVCはArgo CDの削除対象から除外しているが、PVCやNamespaceを手動削除すると録音を失う。
