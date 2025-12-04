---
title: "【初心者向け】イラストでわかるDockerとKubernetes - 第1章 コンテナの概要"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Docker", "Kubernetes", "devops", "dockercompose", "SRE"]
published: true
---

# はじめに
この技術記事は［改訂新版］イラストでわかるDockerとKubernetes の要約記事です。
私自身、高卒かつ数学偏差値48であるため、技術書の解読に苦労することがよくあります。
初心者の方だと技術書の解釈に困ることがあるので、そのような初心者の方でもわかりやすいようにまとめました。
つまり、DockerとKubernetesの概要を理解したい方はこの記事を読めば一発で理解できるはずです。
※ mac環境前提で進んでいきますこと、ご了承ください

## 対象者
- DockerとKubernetesの初心者向けの記事を読みたい人
- DockerとKubernetesの概要をサラっと図解で学習したい人
- DockerとKubernetesのことを学習したいけど、コンピュータサイエンスが全くわからない人

## この記事のゴール
実際に実行されるコマンドや図解を通してDockerとKubernetesの概要を"なんとなく"理解する。

## 参考書籍
[［改訂新版］イラストでわかるDockerとKubernetes](https://www.amazon.co.jp/%EF%BC%BB%E6%94%B9%E8%A8%82%E6%96%B0%E7%89%88%EF%BC%BD%E3%82%A4%E3%83%A9%E3%82%B9%E3%83%88%E3%81%A7%E3%82%8F%E3%81%8B%E3%82%8BDocker%E3%81%A8Kubernetes-Software-Design-plus%E3%82%B7%E3%83%AA%E3%83%BC%E3%82%BA-%E5%BE%B3%E6%B0%B8/dp/4297140551/ref=sr_1_1?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&dib=eyJ2IjoiMSJ9.yt5-kx7_-h-ADLap_tbKFo3wdAb9UuWm8Q84w1XUWd-rwLWlYz5fJEDpc9a2kjT-lJJa8GZwMYtaZbHCMKx_dAe-48zh4yhinp7qSPpoFo1Nrfv2wcmOmKxFT8mjfZh0qkKWCRn34P00ftUFRGQIoMwHTlmr_W32A24Fazck_Hjii1CSagQE6Ie5puOx9z7n1A_vK-rsOgEdgjWDV9uayWnfKKlBaqw4U4eiuJTtp6L8BmEfg7tKdsQDHc1oxO93IbUyzhLTDByMlGu0TeWmr6D4GIpMs6MjmjoDL1TzsyE.yzmw96bYRqbSFAFYvZAYFTSFwOsXMC5b3I17Dh0jR10&dib_tag=se&keywords=%E3%82%A4%E3%83%A9%E3%82%B9%E3%83%88%E3%81%A7%E3%82%8F%E3%81%8B%E3%82%8BDocker%E3%81%A8Kubernetes&qid=1764036366&sr=8-1)

![](/images/docker-illust-01/docker-kubernetes-book.jpg =300x)

## 参考情報
書籍の中身は以下のスライドを使用して作成されている。
スライドを見るだけでも参考になるので、ブクマ推奨。
[コンテナ未経験新人が学ぶコンテナ技術入門](https://www.slideshare.net/slideshow/ss-122754942/122754942)

## 【推奨】 実際にDocker環境を構築
実際にあなたのローカル環境でDocker環境を構築して実行してみてください。
やってみることで身につきます。

### mac 環境でのインストール方法
[Docker Desktop for Macのインストール方法](https://matsuand.github.io/docs.docker.jp.onthefly/desktop/mac/install/)
Docker Desktop をインストールすると同時にdockerコマンドもインストールされます。
ためしに、以下コマンドをターミナルでたたいてみてください。
```
yutahanda@F7Q72G63FD-yutahanda dev % docker -v
Docker version 29.0.1, build eedd969
```

# 【第1章】 コンテナ技術の概要
ここではWeb開発ではデファクトスタンダードになってきているコンテナの概要を解説していきます。
第1章では、コンテナの特徴・昔と比べて何が便利になったかを "なんとなく" イメージで捉えられることがゴールです。
※実際のコンテナ環境の中身や具体的なコマンド・動かし方は第2章で解説します。

## 1.1 コンテナを見てみよう
![](/images/docker-illust-01/learningcontainer-181112024240-6.webp)
コンテナは一つの共有されたOS上で独立されたアプリケーションを作成する技術です。
そして、そのコンテナ内で実行されるプロセスはあたかもOSを占有しているかのように見えます。
難しいですが、簡単に言えば「OSという『土地』は共有し、アプリごとに『専用の部屋』だけを作る技術」ですね。
この段階では意味がわからないと思いますが『ふーん』ぐらいで次に進みましょう。

### 1.1.1 コンテナの概要
まずはコンテナがどういうものなのか、実際にdockerコマンドを実行してみましょう。
コンテナは「コンテナイメージ」と呼ばれるコンテナの素から作成します。

**Ubuntu 22.04 環境（仮想環境）を新しく作成**
```
yutahanda@F7Q72G63FD-yutahanda docker-test % docker run -it ubuntu:22.04 
Unable to find image 'ubuntu:22.04' locally
22.04: Pulling from library/ubuntu
0ec3d8645767: Pull complete 
Digest: sha256:104ae83764a5119017b8e8d6218fa0832b09df65aae7d5a6de29a85d813da2fb
Status: Downloaded newer image for ubuntu:22.04
```
**その中に入ってコマンド操作（シェル操作）を開始**
```
root@761781a1c9ba:/# ls
bin  boot  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```
**実行環境内部で動いているプロセスを確認**
```
root@761781a1c9ba:/# ps ax
  PID TTY      STAT   TIME COMMAND
    1 pts/0    Ss     0:00 /bin/bash
   10 pts/0    R+     0:00 ps ax
```
ホストやほかのコンテナのプロセスは見えない。
本来なら、ホスト側で多くのプロセスが動いているはずだが、見えない。
つまり、コンテナとは独立された実行環境ということです。

### 1.1.2 コンテナイメージ
コンテナの実行には、その素となる「コンテナイメージ」が必要となる。
先ほどの場合だと 22.04: Pulling from library/ubuntu がコンテナイメージとなる。
ローカル環境にUbuntuのコンテナイメージがなかったため、リモートのdockerhubからダウンロードしていた。
コンテナイメージは、コンテナで実行するアプリケーションや、その実行に必要なファイル群を含んでいる。
コンテナイメージを用いることで
**チームで共有し、さまざまな場所でコンテナを実行可能。**

このワークフローを示すのが以下。
![](/images/docker-illust-01/learningcontainer-181112024240-11.webp)

- Build => コンテナイメージの作成
- Ship => コンテイメージのマシン間の共有
- Run => コンテナイメージを素にしたコンテナの実行

docker コマンドによるイメージのビルドの例は以下。
まず、Dockerfileを作成
```
yutahanda@F7Q72G63FD-yutahanda docker-test % cat <<EOF > ./sample/Dockerfile
heredoc> FROM ubuntu:22.04
heredoc> RUN echo hi > hi
heredoc> EOF
yutahanda@F7Q72G63FD-yutahanda docker-test % ls
sample
yutahanda@F7Q72G63FD-yutahanda docker-test % cd sample
yutahanda@F7Q72G63FD-yutahanda sample % ls
Dockerfile
yutahanda@F7Q72G63FD-yutahanda sample % cat Dockerfile 
FROM ubuntu:22.04
RUN echo hi > hi
```
そして、イメージ設計図（Dockerfile）からイメージをビルド
```
yutahanda@F7Q72G63FD-yutahanda sample % docker build -t myimage .
[+] Building 0.2s (6/6) FINISHED                                                                                                                                      docker:desktop-linux
 => [internal] load build definition from Dockerfile                                                                                                                                  0.0s
 => => transferring dockerfile: 72B                                                                                                                                                   0.0s
 => [internal] load metadata for docker.io/library/ubuntu:22.04                                                                                                                       0.0s
 => [internal] load .dockerignore                                                                                                                                                     0.0s
 => => transferring context: 2B                                                                                                                                                       0.0s
 => [1/2] FROM docker.io/library/ubuntu:22.04@sha256:104ae83764a5119017b8e8d6218fa0832b09df65aae7d5a6de29a85d813da2fb                                                                 0.0s
 => => resolve docker.io/library/ubuntu:22.04@sha256:104ae83764a5119017b8e8d6218fa0832b09df65aae7d5a6de29a85d813da2fb                                                                 0.0s
 => [2/2] RUN echo hi > hi                                                                                                                                                            0.1s
 => exporting to image                                                                                                                                                                0.0s
 => => exporting layers                                                                                                                                                               0.0s
 => => exporting manifest sha256:62d1a44559a8fc242a03fae50907375d78f44c87f3860380ad837c7584133537                                                                                     0.0s
 => => exporting config sha256:03c98973571ee33d869d159f87d3d5edef2f5b43130f7fa0b39ea0c78b7c4369                                                                                       0.0s
 => => exporting attestation manifest sha256:40d70f3dbff559b73c62f16052a8632c135a3e7116b72df651f79fcc906866e6                                                                         0.0s
 => => exporting manifest list sha256:666b03901b1852e95e1939c9357e40203a1cb5904eb39667d84c80c88c251144                                                                                0.0s
 => => naming to docker.io/library/myimage:latest                                                                                                                                     0.0s
 => => unpacking to docker.io/library/myimage:latest           
```
作成した myimage イメージがある
```
yutahanda@F7Q72G63FD-yutahanda sample % docker images
                                                                                                                                                                       i Info →   U  In Use
IMAGE            ID             DISK USAGE   CONTENT SIZE   EXTRA
myimage:latest   666b03901b18        107MB         27.4MB        
ubuntu:22.04     104ae83764a5        106MB         27.4MB    U   
```
ビルドしたイメージ（myimage）を実行
```
yutahanda@F7Q72G63FD-yutahanda sample % docker run --rm -it myimage
root@37ab8a195ed3:/# ps
  PID TTY          TIME CMD
    1 pts/0    00:00:00 bash
    9 pts/0    00:00:00 ps
```
docker pullによりレジストリサービス（Docker Hub）からイメージを取得することも可能
```
yutahanda@F7Q72G63FD-yutahanda sample % docker pull ubuntu:24.04
24.04: Pulling from library/ubuntu
97dd3f0ce510: Pull complete 
Digest: sha256:c35e29c9450151419d9448b0fd75374fec4fff364a27f176fb458d472dfc9e54
Status: Downloaded newer image for ubuntu:24.04
docker.io/library/ubuntu:24.04
```
実際に取得できたことを確認できる
```
yutahanda@F7Q72G63FD-yutahanda sample % docker images
                                                                                                                                                                       i Info →   U  In Use
IMAGE            ID             DISK USAGE   CONTENT SIZE   EXTRA
myimage:latest   666b03901b18        107MB         27.4MB        
ubuntu:22.04     104ae83764a5        106MB         27.4MB    U   
ubuntu:24.04     c35e29c94501        139MB         28.9MB   
```

## 1.2 コンテナ技術の基本的な特徴

## 1.3 本書で注目するDockerとKubernetes
