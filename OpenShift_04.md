# 4. アプリケーション開発（Dockerfile、BuildConfig）
ここでは Dockerfile を使って アプリケーションをデプロイするまでを実施していきます。OpenShift には イメージのビルドからプッシュまで自動で行ってくれる機能があるため、そちらを試しに使っていきます。

**ゴール**

- Dockerfile を使って docker イメージの作成、イメージの push ができること
- 登録したイメージを使った Web アプリケーションをデプロイできること

**セクション**

- 4.1. Node.js アプリケーション開発
- 4.2. イメージのビルド
- 4.3. イメージを使ったアプリケーションのデプロイ

<div style="page-break-before:always"></div>

## 4.1. Node.js アプリケーション開発
今回は Node.js アプリケーションを作成していきます。

**Step 1** Node.js のアプリケーションプログラムを作成していきます。

```
$ mkdir $HOME/nodejs
$ cd $HOME/nodejs

$ vi package.json
```

※以下をコピペして保存してください。<br>
※入力モードになる前に、「:set paste」でペーストモードにすると綺麗にペーストができます。

```
{
  "name": "docker_web_app",
  "version": "1.0.0",
  "description": "Node.js on Docker",
  "author": "First Last <first.last@example.com>",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.16.1"
  }
}
```

アプリケーションプログラムを作成します。
```
$ vi server.js
```
※以下をコピペして保存してください。
```
'use strict';

const express = require('express');

// Constants
const PORT = 8080;
const HOST = '0.0.0.0';

// App
const app = express();
app.get('/', (req, res) => {
  res.send('Hello World');
});

app.listen(PORT, HOST);
console.log(`Running on http://${HOST}:${PORT}`);
```


```
Note
今回のプログラムは8080番ポートで待ち受けて、「Hello World」を出力するアプリとなります。
Node.js は昨今の Web アプリで利用されていることも多いため、参考記事やサンプルも多いと思います。
```

## 4.2. イメージのビルド
アプリケーションの準備ができたので、イメージのビルドを行っていきます。

**Step 1** まずは、Dockerfile を作成します。

```
$ vi Dockerfile
```
※以下をコピペして保存してください。
```
FROM docker.io/library/node:12

WORKDIR /usr/src/app

COPY package.json ./
RUN npm install

COPY server.js ./

EXPOSE 8080
CMD [ "node", "server.js" ]
```

あわせて、除外設定ファイルを作成します。
```
$ vi .dockerignore
```

※以下をコピペして保存してください。

```
node_modules
npm-debug.log
```

**Step 2** 準備ができたので、早速イメージのビルドを行っていきます。\
`kubeadmin`でログインし、カレントプロジェクトをご自身のプロジェクトに変更してください。\
※<your_name>はご自身のプロジェクト名を入力してください。

```
$ oc login -u kubeadmin

$ oc project <your_name>
Using project "n-sakamaki" on server "https://api.crc.testing:6443".
```

ビルド用のオブジェクト（BuildConfig）を作成します。\
イメージ名のタグは個人名を指定します。\
※<your_name>の部分をプロジェクト名と同じ個人名に置き換えてください。

```
$ oc new-build --name=hello-nodejs --binary \
--to=hello-nodejs:<your_name> --strategy=docker
↑この行までコピペして実行してください

    * A Docker build using binary input will be created
      * The resulting image will be pushed to image stream tag "hello-nodejs:n-sakamaki"
      * A binary build was created, use 'oc start-build --from-dir' to trigger a new build

--> Creating resources with label build=hello-nodejs ...
    imagestream.image.openshift.io "hello-nodejs" created
    buildconfig.build.openshift.io "hello-nodejs" created
--> Success

```

```
Note
oc new-build：新しいビルド用のオブジェクトを作成します。ビルドする際の情報が設定されています。
オプション
--name：BuildConfig の名前
--to：ビルドで作成する先のイメージ名/タグ
--strategy：ビルドの方式を指定します
--binary：ビルド時にローカルファイルを利用する場合に指定します
```

最後に作成した BuildConfigを使ってビルドを行います。

```
$ oc start-build hello-nodejs --from-dir=. --follow

Uploading directory "." as binary input for the build ...
.
Uploading finished
build.build.openshift.io/hello-nodejs-1 started
Receiving source from STDIN as archive ...
Caching blobs under "/var/cache/blobs".

Pulling image docker.io/library/node:12 ...
・・・
Storing signatures
Adding transient rw bind mount for /run/secrets/rhsm
STEP 1/9: FROM docker.io/library/node:12
STEP 2/9: WORKDIR /usr/src/app
・・・
STEP 9/9: LABEL "io.openshift.build.name"="hello-nodejs-1" "io.openshift.build.namespace"="n-sakamaki"
・・・
--> 4af61545975
4af6154597539b2b259b70fa798c8c78ca75e3251ebeaa63fb6d056ca24a4930

Pushing image image-registry.openshift-image-registry.svc:5000/n-sakamaki/hello-nodejs:n-sakamaki ...
・・・
Successfully pushed image-registry.openshift-image-registry.svc:5000/n-sakamaki/hello-nodejs@sha256:～
Push successful
```

```
Note
oc start-build：指定した BuildConfig を使って、イメージをビルド・プッシュします。
--from-dir：ビルド時に利用するディレクトリを指定します
--folow：ビルド時の様子をログ出力します
```

**Step 3** イメージが作成されていることを確認します。

```
$ oc get all
NAME                       READY   STATUS      RESTARTS   AGE
pod/hello-nodejs-1-build   0/1     Completed   0          7m50s

NAME                                          TYPE     FROM     LATEST
buildconfig.build.openshift.io/hello-nodejs   Docker   Binary   1

NAME                                      TYPE     FROM     STATUS     STARTED         DURATION
build.build.openshift.io/hello-nodejs-1   Docker   Binary   Complete   7 minutes ago   1m48s

NAME                                          IMAGE REPOSITORY                                                                  TAGS         UPDATED
imagestream.image.openshift.io/hello-nodejs   default-route-openshift-image-registry.apps-crc.testing/n-sakamaki/hello-nodejs   n-sakamaki   6 minutes ago

$ oc describe is hello-nodejs
Name:                   hello-nodejs
Namespace:              n-sakamaki
Created:                8 minutes ago
Labels:                 build=hello-nodejs
Annotations:            openshift.io/generated-by=OpenShiftNewBuild
Image Repository:       default-route-openshift-image-registry.apps-crc.testing/n-sakamaki/hello-nodejs
Image Lookup:           local=false
Unique Images:          1
Tags:                   1
・・・
```

## 4.3. イメージを使ったアプリケーションのデプロイ
作成したイメージを使ってアプリケーションをデプロイします。

**Step 1** 作成したイメージを使って、アプリケーションをデプロイします。\
※<your_name>は個人名を指定します。

```
$ oc new-app hello-nodejs:<your_name>
--> Found image e7546b1 (7 minutes old) in image stream "n-sakamaki/hello-nodejs" under tag "n-sakamaki" for "hello-nodejs:n-sakamaki"


--> Creating resources ...
    deployment.apps "hello-nodejs" created
    service "hello-nodejs" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose service/hello-nodejs'
    Run 'oc status' to view your app.
```

```
Note
oc new-app：アプリケーションを新規に作成します。
作成はイメージだけではなく、ソースコード、Templateなどを指定してデプロイすることもできます。

【参考】https://access.redhat.com/documentation/ja-jp/openshift_container_platform/4.10/html/building_applications/creating-applications-using-cli
```

**Step 2** 作成されたアプリケーションを確認します。

```
$ oc get all
NAME                                READY   STATUS      RESTARTS   AGE
pod/hello-nodejs-1-build            0/1     Completed   0          9m56s
pod/hello-nodejs-847fc56464-6v45s   1/1     Running     0          28s

・・・

NAME                   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/hello-nodejs   ClusterIP   10.217.4.78   <none>        8080/TCP   28s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-nodejs   1/1     1            1           28s

・・・
```
Deployment / Pod / Service が作成されていることが確認できます。

**Step 3** 外部からアクセスできるように route を作成します。

```
$ oc expose svc hello-nodejs
route.route.openshift.io/hello-nodejs exposed

$ oc get route
NAME           HOST/PORT                                  PATH   SERVICES       PORT       TERMINATION   WILDCARD
hello-nodejs   hello-nodejs-n-sakamaki.apps-crc.testing          hello-nodejs   8080-tcp                 None
```

**Step 4** CLI からアクセスして確認してみます。\
※route で確認できた URL にアクセスします。

```
$ curl -I hello-nodejs-n-sakamaki.apps-crc.testing
HTTP/1.1 200 OK
x-powered-by: Express
content-type: text/html; charset=utf-8
content-length: 11
etag: W/"b-Ck1VqNd45QIvq3AZd8XYQLvEhtA"
date: Sun, 03 Apr 2022 13:46:29 GMT
keep-alive: timeout=5
set-cookie: 1c847944e870f40f3d1bab6501bdea9b=8cfff468171a82f141446919e2e48b63; path=/; HttpOnly
cache-control: private
```

※以下のように名前解決ができない場合はhostsファイルにエントリーを追加してください。

```
$ curl hello-nodejs-n-sakamaki.apps-crc.testing
curl: (6) Could not resolve host: hello-nodejs-n-sakamaki.apps-crc.testing

$ echo '192.168.130.11 hello-nodejs-n-sakamaki.apps-crc.testing' | sudo tee -a /etc/hosts
```

**Step 5** 次はブラウザからアクセスして確認してみます。\
手元端末のhostsファイルに route で確認したホスト名を追加します。

```
127.0.0.1 hello-nodejs-n-sakamaki.apps-crc.testing
```

コマンドプロンプト or PowerShell 等のコマンドラインから、SSHポートフォワードを行います。\
※xxx.xxx.xxx.xxx はAzure VMのパブリックIPを指定する。\
※ユーザー名（user01）の部分はご自身が使用するユーザー名を指定してください。
```
ssh user01@xxx.xxx.xxx.xxx -N -L 443:192.168.130.11:443 -L 80:192.168.130.11:80
user01@xxx.xxx.xxx.xxx's password:
```

`oc get route`で確認したURLにブラウザからアクセスすると以下のように表示されます。
```
URL例: http://hello-nodejs-n-sakamaki.apps-crc.testing
```
![4-3-1.jpg](./img/4-3-1.jpg)

「5. アプリケーションデプロイ（Rolling Update、Blue-Green、Canary）」で今回デプロイしたアプリケーションを使用するため、消さずに残しておきます。

### 【参考文献】

> Projectとアプリケーションデプロイ<br>
> https://thinkit.co.jp/article/15696

> 第2章 アプリケーションの作成<br>
> https://access.redhat.com/documentation/ja-jp/openshift_container_platform/4.10/html/building_applications/_creating-applications

> Performing and configuring basic builds<br>
> https://docs.openshift.com/container-platform/4.10/cicd/builds/basic-build-operations.html

> Creating applications using the CLI<br>
> https://docs.openshift.com/container-platform/4.10/applications/creating_applications/creating-applications-using-cli.html

> Node.js Web アプリケーションを Docker 化する<br>
> https://nodejs.org/ja/docs/guides/nodejs-docker-webapp/



