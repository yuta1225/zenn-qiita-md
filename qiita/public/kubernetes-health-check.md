---
title: 【初心者向け】EKS構築後のネットワーク疎通確認手順とコマンド解説
private: false
tags:
  - Kubernetes
  - Docker
  - AWS
  - CLI
  - EKS
updated_at: '2026-01-22T04:22:52.448Z'
id: null
organization_url_name: null
slide: false
---

# はじめに：なぜこれが必要なのか？
EKSクラスタを構築した際、AWSコンソール上のステータスが Active となったとしても、Kubernetesとしてのネットワーク機能が正常に動作しているとは限りません。

本記事では、構築直後のEKSクラスタに対して実施すべき一連の疎通確認手順と、使用する kubectl コマンドの詳細なオプション、および確認意図を解説します。

## 検証の目的
以下のネットワークコンポーネントが正常に機能しているかを、実際のワークロードを用いて検証します。

- NAT Gateway / Route Table: ノードからインターネットへのOutbound通信（コンテナイメージのPull等）
- CoreDNS: クラスタ内部の名前解決
- VPC CNI / Security Group: Pod間およびノード間の通信
- Control Plane: APIサーバー経由のトンネリング通信

# Phase 1: ワークロードの準備とトラブルシュート
まずはテスト用のWebサーバー（Nginx）を立てる。

## 1. Deployment作成（と、うっかりミス）

```
$ kubectl create deployment nginx-test --image=nginx:apline
deployment.apps/nginx-test created
```
create deployment: 指定した名称でDeploymentリソースを作成します。
nginx-test: 作成するDeploymentの名前です。
--image=nginx:alpine: 使用するコンテナイメージを指定します。alpine タグを使用することで軽量なイメージを展開します。

## 2. 修正と起動確認（Outbound通信の証明）
※これは手順ではないです。単なる修正
aplineでコンテナ作成しようとしてミスったので、イメージ変更。
ここで慌てず、正しいイメージを設定し直す。
```
$ kubectl set image deployment/nginx-test nginx=nginx:alpine
deployment.apps/nginx-test image updated

$ kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
nginx-test-66cf5ddf9b-9m94v   1/1     Running   0          8s
```
set image: 既存リソースのコンテナイメージを更新します。
deployment/nginx-test: 対象リソースを指定します。
nginx=: コンテナ名を指定します（Deployment作成時に自動的に付与された名前）。
nginx:alpine: 正しいイメージタグを指定します。

## 3. Serviceの公開（DNS検証の準備）
```
$ kubectl expose deployment nginx-test --port=80 --target-port=80 --name=nginx-svc
service/nginx-svc exposed
```
Podにアクセスするための「窓口（Service）」を作った。 これを作らないと、この後の「名前解決（DNS）」のテストができない。

expose deployment: 指定したDeploymentをServiceとして公開します。
--port=80: Serviceが待ち受けるポート番号です（クラスタ内で利用されるポート）。
--target-port=80: 転送先となるコンテナ側のポート番号です（Nginxのデフォルトポート）。
--name=nginx-svc: 作成するServiceの名前です。これがクラスタ内DNS名として登録されます。

# Phase 2: 外部からの到達性確認
次に、「俺たちのPCからクラスタの中まで手が届くか」を確認する。

## 4. Port-Forwardによるトンネリング
```
$ kubectl port-forward svc/nginx-svc 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Handling connection for 8080
```
ローカル環境からクラスタ内部のPodへ、APIサーバーを経由して直接アクセスします。

svc/nginx-svc: 接続先のServiceを指定します。
8080:80: [ローカルポート]:[リモートポート] の形式です。ローカルの8080番へのアクセスを、Serviceの80番へ転送します。

ブラウザ等で localhost:8080 にアクセスできれば、以下の経路が正常であることが証明されます。

- IAM認証: クライアントが正しく認証されている。
- API Server到達性: ローカルからAWS上のコントロールプレーンへの通信が可能である。
- Pod制御: コントロールプレーンからデータプレーン（Pod）へのトンネリングが可能である。

# Phase 3: 内部ネットワークの健全性（最重要）
ここからが本番だ。外から見るだけじゃなく、「Podの中に入って」 ネットワークを検証する。 これこそがSREの流儀だ。

## 5. 調査用Podへの潜入
```
$ kubectl run debug-check -it --rm --image=curlimages/curl --restart=Never -- sh
```
最後に、クラスタ内部に検証用Podを立ち上げ、Podの中からネットワーク状況を確認します。これが最も重要な検証工程です。

run debug-check: debug-check という名前でPodを単独起動します。
-it: インタラクティブモード（標準入力を接続し、TTYを割り当てる）で起動します。シェル操作に必須です。
--rm: シェル終了後、Podを自動的に削除します（ゴミを残さないため）。
--image=curlimages/curl: curl コマンドが含まれる軽量イメージを使用します。
--restart=Never: 使い捨てのため、終了しても再起動させません。
-- sh: コンテナ起動後に実行するコマンド（シェル）を指定します。

## 6. 内部DNSとEast-West通信（サーバー間通信）の確認
```
~ $ curl -I http://nginx-svc
HTTP/1.1 200 OK
Server: nginx/1.29.4
```
nginx-svc という名前でアクセスして 200 OK が返ってきた。これが何を意味するか。
CoreDNSが生きてる: nginx-svc → 10.x.x.x への名前解決に成功した。
VPC CNI / Kube-proxyが正常: 解決されたIPに対して、パケットが正しく転送された。
Security Groupが正常: ノード間（またはPod間）の通信がファイアウォールで遮断されていない。

## 7. 外部へのEgress通信の確認
```
~ $ curl -I https://www.google.com
HTTP/2 200 
```
Pod内部からインターネット上のサイトへアクセスを試行します。
NAT Gatewayの動作: プライベートIPしか持たないPodからの通信が、NAT変換されてインターネットに出られているか。
Security Group (Egress): アウトバウンド通信が許可されているか。
