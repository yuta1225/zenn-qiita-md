---
title: "【初心者向け】イラストでわかるDockerとKubernetes - 第2章 DockerによるBuild、Ship、run"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Docker", "Kubernetes", "devops", "dockercompose", "SRE"]
published: false
---
# はじめに
この記事は過去に投稿した「【初心者向け】イラストでわかるDockerとKubernetes - 第1章 コンテナの概要」の続編です。
本当に端的にまとめた記事になりますので、詳しい内容は該当書籍をお読みください。

# 2.1 DockerによるBuild、Ship、Run
![](/images/docker-illust-01/learningcontainer-181112024240-11.webp)
1. Dockerがコンテナイメージを作成（Build）
2. Dockerがレジストリにイメージをpushしたりpullしたりしてshipする
3. Dockerが作成されたコンテナイメージを実行（RUN）してコンテナを実行

## 2.1.1 コンテナイメージの作成
![](/images/docker-illust-01/learningcontainer-181112024240-13.webp)
コンテナを作成するためには、コンテナイメージを作成（Build）する必要がある。
Dockerはコンテナイメージを元にコンテナとしてルートファイルシステムや隔離された実行環境を作成する。
コンテナイメージの作成にあたって、Dockerに以下の材料を与える必要がある。

- Dockerfile：コンテナの作成手順書
- コンテキスト：コンテナに格納したりビルド時に使用するファイル群（例：アプリケーションのソースコードなど）

これらをDockerに渡すと、Dockefileに沿ってコンテキストに含まれるファイルが適宜用いられながら、イメージにまとめ上げられる。

### 実際にコンテナイメージを作成してみる
#### hello, world！を出力するシェルスクリプトの作成
```
yutahanda@F7Q72G63FD-yutahanda docker-illust-02 % touch hello.sh
yutahanda@F7Q72G63FD-yutahanda docker-illust-02 % vi hello.sh 
yutahanda@F7Q72G63FD-yutahanda docker-illust-02 % ls
hello.sh
yutahanda@F7Q72G63FD-yutahanda docker-illust-02 % cat hello.sh 
#!/bin/bash
set -eu
echo "Hello World"
exec sleep infinity
yutahanda@F7Q72G63FD-yutahanda docker-illust-02 % chmod +x hello.sh
```
※ chmodコマンドで全てのユーザーに実行権限を与える

:::details シェルスクリプトの内容
#!/bin/bash

このスクリプトをBashシェルで実行することを指定しています。

set -eu

スクリプトの「安全性」を高めるための設定です。

-e: コマンドがエラー（終了ステータスが0以外）になったら、即座にスクリプトを強制終了します。

-u: 未定義の変数を使おうとしたらエラーにします。

echo "Hello World"

画面（標準出力）に "Hello World" と表示します。スクリプトが正常に動き出したことを確認するためのログ出力です。

exec sleep infinity

ここがこのスクリプトの最大のポイントです。

sleep infinity: 何もしないまま無限に待機します。これにより、スクリプト（およびコンテナ）が終了するのを防ぎます。

exec: シェルのプロセス（PID）を sleep コマンドに置き換えます。

これがないと、bash の子プロセスとして sleep が動きます。

exec を使うことで、sleep がメインプロセス（PID 1）になり、Docker停止時などのシグナル（SIGTERMなど）を正しく受け取って素早く終了できるようになります。
:::

#### Dockerfileの作成
```
yutahanda@F7Q72G63FD-yutahanda docker-illust-02 % touch Dockerfile
yutahanda@F7Q72G63FD-yutahanda docker-illust-02 % ls
Dockerfile      hello.sh
yutahanda@F7Q72G63FD-yutahanda docker-illust-02 % vi Dockerfile 
yutahanda@F7Q72G63FD-yutahanda docker-illust-02 % cat Dockerfile 
FROM ubuntu:22.04
COPY ./hello.sh /hello.sh
ENTRYPOINT ["/hello.sh"]
```

#### docker buildコマンドを実行してコンテナイメージを作成
```
yutahanda@F7Q72G63FD-yutahanda docker-illust-02 % tree
.
├── Dockerfile
└── hello.sh

yutahanda@F7Q72G63FD-yutahanda docker-illust-02 % docker build -t myimage:v1 .
[+] Building 0.2s (7/7) FINISHED                                                            docker:desktop-linux
 => [internal] load build definition from Dockerfile                                                        0.0s
 => => transferring dockerfile: 107B                                                                        0.0s
 => [internal] load metadata for docker.io/library/ubuntu:22.04                                             0.1s
 => [internal] load .dockerignore                                                                           0.0s
 => => transferring context: 2B                                                                             0.0s
 => [internal] load build context                                                                           0.0s
 => => transferring context: 94B                                                                            0.0s
 => CACHED [1/2] FROM docker.io/library/ubuntu:22.04@sha256:104ae83764a5119017b8e8d6218fa0832b09df65aae7d5  0.0s
 => => resolve docker.io/library/ubuntu:22.04@sha256:104ae83764a5119017b8e8d6218fa0832b09df65aae7d5a6de29a  0.0s
 => [2/2] COPY ./hello.sh /hello.sh                                                                         0.0s
 => exporting to image                                                                                      0.0s
 => => exporting layers                                                                                     0.0s
 => => exporting manifest sha256:91e48d2b1c5af7d39978fd74b5d842c0101442d57fdb8e7ec0e21a9756bfa254           0.0s
 => => exporting config sha256:7bf42877d9ee77b8f277d5152ba6bb598daa8d7c2de67e6eae4942126282aa9e             0.0s
 => => exporting attestation manifest sha256:2d32a6f71658e1f8c310ec3093be010ac487a88d875aca76c07cd744560cc  0.0s
 => => exporting manifest list sha256:3b412fba661b367d77be0fd6e1ada626d9e4975342f8fc128d7f286dc0f1fa8f      0.0s
 => => naming to docker.io/library/myimage:v1                                                               0.0s
 => => unpacking to docker.io/library/myimage:v1                                                            0.0s
```
#### myimage:v1が作成されたことが確認できる
```
yutahanda@F7Q72G63FD-yutahanda docker-illust-02 % docker image ls myimage:v1
                                                                                             i Info →   U  In Use
IMAGE        ID             DISK USAGE   CONTENT SIZE   EXTRA
myimage:v1   3b412fba661b        10f7MB         27.4MB      
```

