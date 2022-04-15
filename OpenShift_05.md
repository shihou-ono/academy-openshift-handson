# 5. アプリケーションデプロイ（Rolling Update、Blue-Green、Canary）
ここではアプリケーションのデプロイ方法について学習していきます。コンテナデプロイのなかでもRolling-Update、Blue-Green、Canary について実施していきます。

**ゴール**

- イメージを変更し、Rolling-Update が実行されていることを確認できること
- rollout コマンドを使ってデプロイを操作できること
- Blue-Green デプロイメントが実施できること
- Canary リリースが実施できること
- マニフェストファイルの更新について理解できること

**セクション**

- 5.1. Rolling Update
- 5.2. rollout コマンド
- 5.3. Blue-Green デプロイメント
- 5.4. Canary リリース
- 5.5. マニフェストファイルの更新

<div style="page-break-before:always"></div>

## 5.1. Rolling Update
ここでは Rolling Update を試してみます。今回は nginx のバージョンアップを例として実施してみます。<br>

**Step 1** まずは、nginx の pod をでデプロイします。

```
$ oc login -u kubeadmin

$ cd $HOME/nginx

$ oc create dc nginx --image=nginx:1.17 --dry-run=client -o yaml > nginx.yaml

$ vi nginx.yaml

以下行を修正・追加
---------------------------------------------
spec:
  replicas: 2                         ★ 1 ⇒ 2 に修正
・・・
    spec:
      containers:
      - image: nginx:1.17
        name: default-container
        resources: {}
        ports:                        ★追加
        - containerPort: 80           ★追加
・・・
---------------------------------------------

$ oc apply -f nginx.yaml
deploymentconfig.apps.openshift.io/nginx created

$ oc get all
NAME                 READY   STATUS      RESTARTS   AGE
pod/nginx-1-8lxhd    1/1     Running     0          15s
pod/nginx-1-9gxr8    1/1     Running     0          15s
pod/nginx-1-deploy   0/1     Completed   0          17s

NAME                            DESIRED   CURRENT   READY   AGE
replicationcontroller/nginx-1   2         2         2       17s

NAME                                       REVISION   DESIRED   CURRENT   TRIGGERED BY
deploymentconfig.apps.openshift.io/nginx   1          2         2         config
```

**Step 2** イメージのバージョンを指定したので、対象バージョンのイメージで Pod が起動しているか確認します。

```
$ oc get dc nginx -o=jsonpath={.spec.template.spec.containers[0].image}
nginx:1.17
```

URL を公開して、curl で確認してみます。

```
$ oc expose dc nginx
service/nginx exposed

$ oc expose svc nginx
route.route.openshift.io/nginx exposed

$ URL=http://$(oc get route nginx -o=jsonpath={.spec.host})
$ echo $URL
http://nginx-n-sakamaki.apps-crc.testing

$ curl -I $URL
HTTP/1.1 200 OK
server: nginx/1.17.10
・・・
```

※以下のように名前解決が上手くいかない場合は、hostsファイルに新しくエントリを追加してみてください。
```
$ curl -I $URL
curl: (6) Could not resolve host: nginx-n-sakamaki.apps-crc.testing

$ echo "192.168.130.11 nginx-n-sakamaki.apps-crc.testing" | sudo tee -a /etc/hosts

$ curl -I $URL
HTTP/1.1 200 OK
・・・
```

確認方法をコンテナ内部から nginx コマンドでバージョンを確認します。

```
$ oc exec $(oc get po -l deploymentconfig=nginx -o=jsonpath={.items[0].metadata.name}) \
-- nginx -v
↑この行までコピペして実行してください

nginx version: nginx/1.17.10
・・・
```

**Step 3** アップデートする前に進捗確認用にターミナルをもう一つ起動します。
手元端末から追加でVMにSSH接続でログインし、もう一つターミナルを用意します。

新しくログインしたターミナルで project を確認し、pod の状態を監視するコマンドを実行しておきます。

※projectが異なる場合、ご自身のprojectに切り替えてください。

```
$ oc project
Using project "n-sakamaki" on server "https://api.crc.testing:6443".

$ oc get po -w
NAME             READY   STATUS      RESTARTS   AGE
nginx-1-deploy   0/1     Completed   0          2m11s
nginx-1-dvj5m    1/1     Running     0          2m1s
nginx-1-l9xrz    1/1     Running     0          2m1s

※プロンプトは戻らない状態です
※最終的には「Ctrl + c」でコマンドを停止させます
```

```
Note
-w オプション：実行するとプロンプトは戻らず、状態が変化する毎に結果が出力されます。watch の意味
```

**Step 4** イメージのバージョンを変更してみます。<br>
※作業は最初から起動しているターミナルで実施します。<br>
※今回は 1.17 から 1.18 にバージョンアップします

```
$ oc set image dc nginx default-container=nginx:1.18
deploymentconfig.apps.openshift.io/nginx image updated

$ oc get all
NAME                 READY   STATUS      RESTARTS   AGE
pod/nginx-1-deploy   0/1     Completed   0          3m44s
pod/nginx-2-5pqkp    1/1     Running     0          27s
pod/nginx-2-deploy   0/1     Completed   0          31s
pod/nginx-2-rx5gx    1/1     Running     0          11s

$ oc get dc nginx -o=jsonpath={.spec.template.spec.containers[0].image}
nginx:1.18

$ oc exec $(oc get po -l deploymentconfig=nginx -o=jsonpath={.items[0].metadata.name}) \
-- nginx -v
↑oc exec コマンドはこの行までコピペして実行してください

nginx version: nginx/1.18.0
```

※以下のエラーが出た場合は、nginx の DeploymentConfig を削除して **Step 1** の dc 作成からやり直してみてください。

```
error: You must be logged in to the server (Unauthorized)
Error from server: etcdserver: request timed out
```

**Step 5** 後から起動したターミナルで実際の動きを確認します。

```
$ oc get po -w
NAME             READY   STATUS      RESTARTS   AGE
nginx-1-deploy   0/1     Completed   0          25s
nginx-1-j5rd4    1/1     Running     0          21s
nginx-1-j96kk    1/1     Running     0          21s

--- 以下が追記されている
※1.18 デプロイ用の Pod 起動
nginx-2-deploy   0/1     Pending     0          0s
nginx-2-deploy   0/1     ContainerCreating   0          0s
nginx-2-deploy   1/1     Running             0          2s

※1.18 の 1 つ目の Pod 起動
nginx-2-5pqkp    0/1     Pending             0          0s
nginx-2-5pqkp    0/1     ContainerCreating   0          0s
nginx-2-5pqkp    1/1     Running             0          14s

※1.17 の 1 つ目の Pod 削除
nginx-1-l9xrz    1/1     Terminating         0          3m22s
nginx-1-l9xrz    0/1     Terminating         0          3m22s

※1.18 の 2 つ目の Pod 起動
nginx-2-rx5gx    0/1     Pending             0          0s
nginx-2-rx5gx    0/1     ContainerCreating   0          0s
nginx-2-rx5gx    1/1     Running             0          1s

※1.17 の 2 つ目の Pod 削除
nginx-1-dvj5m    1/1     Terminating         0          3m25s
・・・

※「Ctrl + c」 でコマンドを停止させます
```

※途中、同じ文字列が記載されているのは仕様です<br>
※実行時のノードの状態により、出力の順序に若干のズレはあります

**Step 6** 今度は curl を実行しながらアップデートしてみます。まずは curl を投げ続けます。<br>
※作業は後から起動したターミナルで実施します。

```
$ URL=http://$(oc get route nginx -o=jsonpath={.spec.host})

$ echo $URL
http://nginx-n-sakamaki.apps-crc.testing

$ while sleep 1; do curl -Is $URL -o /dev/null -w '%{http_code}\n'; done
```

**Step 7** 実際にアップデートしてみましょう
※作業は最初に起動したターミナルで実施します。

```
$ oc set image dc nginx default-container=nginx:1.19
deploymentconfig.apps.openshift.io/nginx image updated

$ oc get po -w
NAME             READY   STATUS              RESTARTS   AGE
nginx-1-deploy   0/1     Completed           0          22m
nginx-2-deploy   0/1     Completed           0          20m
・・・
nginx-2-rx5gx    1/1     Terminating         0          4m48s

※2つ目の Pod の「Terminating」が表示されたら、「Ctrl + c」でコマンドを停止させます

$ oc get po
NAME             READY   STATUS      RESTARTS   AGE
nginx-1-deploy   0/1     Completed   0          8m41s
nginx-2-deploy   0/1     Completed   0          5m28s
nginx-3-6n7cn    1/1     Running     0          34s
nginx-3-deploy   0/1     Completed   0          38s
nginx-3-mxzd8    1/1     Running     0          19s
```

**Step 8** curl を投げ続けているターミナルを確認します。

```
・・・
200
200
200


200

200
200
200
200

※「Ctrl + c」でコマンドを停止させます
```

※遅延はしていますが、正常にレスポンスを返しています<br>
※場合によってはタイムアウトなどでエラーが返されている場合も・・・

**Step 9** 最後にバージョンを確認してみます。

```
$ oc exec $(oc get po -l deploymentconfig=nginx -o=jsonpath={.items[0].metadata.name}) \
-- nginx -v
↑この行までコピペして実行してください

nginx version: nginx/1.19.10
```

## 5.2. rollout コマンド
ここではデプロイに使うコマンドとして rollout コマンドを使ってみます。

**Step 1** まずはデプロイ履歴を確認します。

```
$ oc rollout history dc nginx
deploymentconfig.apps.openshift.io/nginx
REVISION        STATUS          CAUSE
1               Complete        config change
2               Complete        config change
3               Complete        config change
```

**Step 2** 続いてデプロイを 1 つ前にロールバックさせてみます（バージョンを 1.18 に戻します）

```
$ oc rollout undo dc nginx
deploymentconfig.apps.openshift.io/nginx rolled back

※続けて以下コマンドを実行してください
※場合によっては出力内容が異なります

$ oc rollout status -w dc nginx
Waiting for rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for rollout to finish: 1 old replicas are pending termination...
Waiting for latest deployment config spec to be observed by the controller loop...
replication controller "nginx-4" successfully rolled out

※「successfully rolled out」まで出力されたら、「Ctrl + c」でコマンドを停止させます
```

**Step 3** ロールバック（v1.18）されている確認します。

```
$ oc get po
NAME             READY   STATUS      RESTARTS   AGE
nginx-1-deploy   0/1     Completed   0          10m
nginx-2-deploy   0/1     Completed   0          7m43s
nginx-3-deploy   0/1     Completed   0          2m53s
nginx-4-7fr5s    1/1     Running     0          53s
nginx-4-deploy   0/1     Completed   0          57s
nginx-4-kwjrq    1/1     Running     0          49s

$ oc exec $(oc get po -l deploymentconfig=nginx -o=jsonpath={.items[0].metadata.name}) -- nginx -v
nginx version: nginx/1.18.0
```

**Step 4** デプロイ履歴を確認するとともに、リビジョン毎の詳細情報を確認してみます。

```
$ oc rollout history dc nginx
deploymentconfig.apps.openshift.io/nginx
REVISION        STATUS          CAUSE
1               Complete        config change
2               Complete        config change
3               Complete        config change
4               Complete        config change

$ oc rollout history dc nginx --revision=4
deploymentconfig.apps.openshift.io/nginx with revision #4
Pod Template:
  Labels:       deployment=nginx-4
        deployment-config.name=nginx
        deploymentconfig=nginx
  Annotations:  openshift.io/deployment-config.latest-version: 4
        openshift.io/deployment-config.name: nginx
        openshift.io/deployment.name: nginx-4
  Containers:
   default-container:
    Image:      nginx:1.18
    Port:       80/TCP
    Host Port:  0/TCP
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
```

```
Note
rollout コマンドについては以下を参照ください。
https://access.redhat.com/documentation/ja-jp/openshift_container_platform/4.10/html/building_applications/deployment-operations
```


```
Important：
アプリケーションの変更、アップグレードを行う際の方法としてデプロイメントストラテジーを指定することができます。今回は Rolling が設定されていますが、他にも以下のストラテジーがあります。

・Rolling：ローリングアップデート
・Recreate：すべての Pod を削除してから、新しい Pod を作成
・その他：カスタム

参考：
https://access.redhat.com/documentation/ja-jp/openshift_container_platform/4.10/html/building_applications/deployment-strategies
```

## 5.3. Blue-Green デプロイメント（動作未確認）
ここではデプロイ方式の 1 つ Blue-Green デプロイメントを実施してみます。

**Step 1** 「4 章」でデプロイしたアプリケーションにアクセスできることを確認します。<br>

```
$ URL=http://$(oc get route hello-nodejs -o=jsonpath={.spec.host})

$ curl $URL
Hello World
```

**Step 2** バージョン 2 として新しいイメージを作成します。

```
$ cd $HOME/nodejs

$ vi server.js

※以下を修正します。

・・・
// App
const app = express();
app.get('/', (req, res) => {
  res.send('Hello World !!');    ★「!!」を追加
});
・・・
```

新しく「hello-nodejs2」という名前で BuildConfig を作成します。
※<your_name>の部分は個人名を設定してください。

```
$ oc new-build --name=hello-nodejs2 --binary \
--to=hello-nodejs:<your_name>2.0 --strategy=docker
↑この行までコピペして実行してください

    * A Docker build using binary input will be created
      * The resulting image will be pushed to image stream tag "hello-nodejs:n-sakamaki2.0"
      * A binary build was created, use 'oc start-build --from-dir' to trigger a new build

--> Creating resources with label build=hello-nodejs2 ...
    imagestreamtag.image.openshift.io "hello-nodejs:n-sakamaki2.0" created
    buildconfig.build.openshift.io "hello-nodejs2" created
--> Success
```

イメージをビルドします。

```
$ oc start-build hello-nodejs2 --from-dir=. --follow

Uploading directory "." as binary input for the build ...
・・・
Successfully pushed image-registry.openshift-image-registry.svc:5000/n-sakamaki/hello-nodejs@sha256:～
Push successful
```

「2.0」のイメージが作成されていることを確認します。

```
$ oc describe is hello-nodejs
Name:                   hello-nodejs
Namespace:              n-sakamaki
・・・
Tags:                   2

n-sakamaki
  no spec tag

  * image-registry.openshift-image-registry.svc:5000/n-sakamaki/hello-nodejs@sha256:752c9b8b8966eaee70aeffc940cdd23d565cb55a5449262718e78b47489c8108
      8 minutes ago

n-sakamaki2.0
  tagged from hello-nodejs:n-sakamaki2.0

  * image-registry.openshift-image-registry.svc:5000/n-sakamaki/hello-nodejs@sha256:c0f86a8d2a4af2809262714e39b3a5aafc646b66e9683306f7782e9a417f31a2
      2 minutes ago
```

**Step 3** 新しいバージョンのイメージを使ってアプリケーションをデプロイします
※<your_name>の部分は個人名を設定してください。

```
$ oc new-app --name hello-nodejs2 hello-nodejs:<your_name>2.0
--> Found image a21ba6c (41 seconds old) in image stream "n-sakamaki/hello-nodejs" under tag "n-sakamaki2.0" for "hello-nodejs:n-sakamaki2.0"


--> Creating resources ...
    deployment.apps "hello-nodejs2" created
    service "hello-nodejs2" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose service/hello-nodejs2'
    Run 'oc status' to view your app.
```

作成されたアプリケーションを確認します。

```
$ oc get all
NAME                                 READY   STATUS      RESTARTS   AGE
・・・
pod/hello-nodejs2-1-build            0/1     Completed   0          2m43s
pod/hello-nodejs2-55f68b89f9-4wj68   1/1     Running     0          27s
・・・
NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/hello-nodejs    ClusterIP   10.217.4.226   <none>        8080/TCP   22m
service/hello-nodejs2   ClusterIP   10.217.4.44    <none>        8080/TCP   27s

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-nodejs    1/1     1            1           22m
deployment.apps/hello-nodejs2   1/1     1            1           28s
・・・
NAME                                    HOST/PORT                              PATH   SERVICES       PORT       TERMINATION   WILDCARD
route.route.openshift.io/hello-nodejs   hello-nodejs-nodejs.apps-crc.testing          hello-nodejs   8080-tcp                 None
```

**Step 4** 「1.0」のアプリケーションから、「2.0」のアプリケーションに Blue-Green デプロイメントを実施します。

```
$ oc get route
NAME           HOST/PORT                                  PATH   SERVICES       PORT       TERMINATION   WILDCARD
hello-nodejs   hello-nodejs-n-sakamaki.apps-crc.testing          hello-nodejs   8080-tcp                 None

$ oc patch route/hello-nodejs -p '{"spec":{"to":{"name":"hello-nodejs2"}}}'
route.route.openshift.io/hello-nodejs patched

$ oc get route
NAME           HOST/PORT                                  PATH   SERVICES        PORT       TERMINATION   WILDCARD
hello-nodejs   hello-nodejs-n-sakamaki.apps-crc.testing          hello-nodejs2   8080-tcp                 None

$ curl $URL
Hello World !!
```

```
Note
oc patch：オブジェクトの要素を変更・追加（patch）します。
```

**Step 5** 切り替え後、意図した動作をしない場合は、ロールバック（切り戻し）を実施します。ロールバックを試してみます。

```
$ oc patch route/hello-nodejs -p '{"spec":{"to":{"name":"hello-nodejs"}}}'
route.route.openshift.io/hello-nodejs patched

$ curl $URL
Hello World
```

## 5.4. Canary リリース
引き続き Canary リリースを実施します。重み付けをしながら徐々に新しいアプリケーションに振り分けを増やしていきます。

**Step 1**「5.3」の最後で「1.0」のアプリケーションに切り戻しました。ここではまず「1.0：80%、2.0：20%」に重み付けをして振り分け設定をします。

```
$ oc edit route hello-nodejs

※vi が起動するので、以下を追加します
※追加したら「:wq」で保存し、プロンプトに戻ります

・・・
spec:
  to:
    kind: Service
    name: hello-nodejs
    weight: 100　★削除
---ここから追加
    weight: 80
  alternateBackends:
  - kind: Service
    name: hello-nodejs2
    weight: 20
---ここまで追加
・・・

route.route.openshift.io/hello-nodejs edited
```

**Step 2** 80:20（既存：新） でカナリアリリースをしました。実際にアクセスして重み付けの確認をします。

```
$ for i in {1..10};do curl $URL -w "\n";done
Hello World
Hello World
Hello World
Hello World
Hello World !!
Hello World
Hello World
Hello World
Hello World
Hello World !!
```

**Step 3** 今度は「50:50」で重み付けを変更してみます。

```
$ oc edit route hello-nodejs

※以下を変更します。

・・・
  spec:
    alternateBackends:
    - kind: Service
      name: hello-nodejs2
      weight: 50         ★50に変更
・・・
    to:
      kind: Service
      name: hello-nodejs
      weight: 50         ★50に変更
・・・
```

**Step 4** 設定内容を確認し、アクセス確認を行います。

```
$ oc describe route hello-nodejs
Name:                   hello-nodejs
Namespace:              nodejs
・・・
Endpoint Port:          8080-tcp

Service:        hello-nodejs
Weight:         50 (50%)
Endpoints:      10.128.0.66:8080

Service:        hello-nodejs2
Weight:         50 (50%)
Endpoints:      10.128.0.69:8080

$ for i in {1..10};do curl $URL -w "\n";done
Hello World
Hello World !!
Hello World
Hello World !!
Hello World
Hello World !!
Hello World
Hello World !!
Hello World
Hello World !!
```

同様に重み付けを「20：80」、「0：100」に変更して、徐々に「2.0」をリリースしていく状況を確認できます。

## 5.5. マニフェストファイルの更新
最後にマニフェストの更新について確認していきます。<br>

**Step 1** 新しく nginx の マニフェストファイル(yaml) を作成し、Pod をデプロイします。

```
$ oc create dc nginx --image=nginx --dry-run=client -o yaml > nginx.yaml

$ oc apply -f nginx.yaml

$ oc get dc,po
```

**Step 2** マニフェストファイルを更新します。

```
$ vi nginx.yaml

※以下を追加

・・・
spec:
・・・
  template:
    metadata:
      creationTimestamp: null
      labels:
        deployment-config.name: nginx
        apps: nginx           ★追記
・・・

```

**Step 3** 更新前のチェックとして、diff をとって確認します。

```
$ oc diff -f nginx.yaml

※少し時間がかかります

@@ -5,11 +5,42 @@
・・・
-  generation: 1
+  generation: 2
・・・
@@ -117,6 +119,7 @@
     metadata:
       creationTimestamp: null
       labels:
+        apps: nginx
         deployment-config.name: nginx
     spec:
       containers:
```

```
Note
一部フィールドは edit/replace などでは更新できないので注意が必要です
```

```
Note
apply、replace などのコマンドの違いは以下を参考にしてみてください
https://qiita.com/tkusumi/items/0bf5417c865ef716b221
```
