# 6. 運用管理（ログ、モニタリング）
最後に運用管理としてログ、モニタリングの操作について確認していきます。

**ゴール**

- OpenShift や 各 Pod のログを確認できること
- ノードや Pod のモニタリングが確認できること

**セクション**

- 6.1. ログの確認
- 6.2. モニタリング

<div style="page-break-before:always"></div>

## 6.1. ログの確認
ここでは Pod のログ確認を行います。4章で GUI から Pod のログを確認しましたが、ここでは CLI で確認します。

**Step 1** nginx にポート設定を追加します。

```
$ oc edit dc nginx

以下を追加
---------------------------------------------
・・・
spec:
・・・
  template:
・・・
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
        ports:                        #追加
        - containerPort: 80           #追加
・・・
---------------------------------------------

deploymentconfig.apps.openshift.io/nginx edited

$ oc get po
NAME             READY   STATUS      RESTARTS   AGE
nginx-1-deploy   0/1     Completed   0          2m41s
nginx-1-hmg7w    1/1     Running     0          2m37s
nginx-2-deploy   1/1     Running     0          18s

$ oc logs dc/nginx
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2022/03/21 06:14:27 [notice] 1#1: using the "epoll" event method
2022/03/21 06:14:27 [notice] 1#1: nginx/1.21.6
2022/03/21 06:14:27 [notice] 1#1: built by gcc 10.2.1 20210110 (Debian 10.2.1-6)
2022/03/21 06:14:27 [notice] 1#1: OS: Linux 4.18.0-305.34.2.el8_4.x86_64
2022/03/21 06:14:27 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2022/03/21 06:14:27 [notice] 1#1: start worker processes
2022/03/21 06:14:27 [notice] 1#1: start worker process 31
2022/03/21 06:14:27 [notice] 1#1: start worker process 32
2022/03/21 06:14:27 [notice] 1#1: start worker process 33
2022/03/21 06:14:27 [notice] 1#1: start worker process 34
2022/03/21 06:14:27 [notice] 1#1: start worker process 35
2022/03/21 06:14:27 [notice] 1#1: start worker process 36
2022/03/21 06:14:27 [notice] 1#1: start worker process 37
2022/03/21 06:14:27 [notice] 1#1: start worker process 38
```

```
Note
oc logs：コンテナのログを出力します
出力できるのは「BuildConfig」 / 「DeploymentConfig」 / 「pod」です
オプション
-c：コンテナを指定します(Pod に複数のコンテナが起動している場合)
-f：追跡表示(リアルタイム表示)します
--tail=N：末尾から N 行分出力します
```

**Step 2** Pod を外部公開します。

```
$ oc expose dc nginx
service/nginx exposed

$ oc expose svc nginx
route.route.openshift.io/nginx exposed

$ URL=http://$(oc get route nginx -o=jsonpath={.spec.host})

$ curl -I $URL
HTTP/1.1 200 OK
・・・
```

**Step 3** curl でアクセスした際のログを確認します。

```
$ oc logs dc/nginx --tail=3
2022/03/21 06:14:27 [notice] 1#1: start worker process 37
2022/03/21 06:14:27 [notice] 1#1: start worker process 38
10.217.0.1 - - [21/Mar/2022:06:15:21 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.68.0" "192.168.130.1"
```

時間があれば、```echo $URL``` で URL を確認し、ブラウザでアクセスしてログを確認してみましょう。

<div style="page-break-before:always"></div>

## 6.2. モニタリング
今回の環境ではモニタリングが試せなかったため、参考 URL を展開させてください。

> 2.6. OPENSHIFT CLI 管理者コマンドリファレンス
> https://access.redhat.com/documentation/ja-jp/openshift_container_platform/4.10/html/cli_tools/cli-administrator-commands

> 第9章 モニタリングダッシュボードの確認
> https://access.redhat.com/documentation/ja-jp/openshift_container_platform/4.10/html/monitoring/reviewing-monitoring-dashboards
