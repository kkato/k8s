# radipod

「安住紳一郎の日曜天国」を録音し、自宅Wi-FiからAntennaPodへ配信する。

- RSS: `http://192.168.10.112:30880/feed.xml`
- 起動時と30分ごとに過去7放送日を確認し、放送済みの未録音回を取得する。
- 録音はPVC `radipod-data` に保存。Pod再作成でも残る。
- NodePortによるLAN内配信。外部向けIngressは作成しない。自宅でダウンロードして通勤中にオフライン再生する。

## 管理

Argo CDが `resources/` を自動反映する。イメージは[radipod](https://github.com/kkato/radipod)のGitHub ActionsでGHCRへ作成し、DeploymentのタグをコミットSHAに固定する。

```sh
kubectl -n radipod logs deployment/radipod --tail=100
kubectl -n radipod get pods,pvc,svc
```

録音対象・URLは `resources/configmap.yaml` を編集してpushし、Argo CDの同期後に `kubectl -n radipod rollout restart deployment/radipod` で反映する。

保存先はOpenEBSのノードローカルディスク。自動削除・バックアップは未設定なので、必要な音声は別途バックアップする。PVCはArgo CDの削除対象から除外しているが、PVCやNamespaceを手動削除すると録音を失う。
