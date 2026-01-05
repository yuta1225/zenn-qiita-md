---
title: "Terraformさん「最小キャパシティは希望キャパシティより大きくできません」ワイ「してるんだが？」"
emoji: "😭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Kubernetes", "devops", "SRE", "AWS", "Terraform"]
published: true
---
# はじめに
新人2年目SREエンジニアのデバッグ備忘録

# nvalidParameterException: Minimum capacity 1 can't be greater than desired size 0
年末年始で起動用のオートスケーリング設定を削除した後、復活させようとした。
そのautoscaling groupをTerraformで適用しようとしたら、以下のようなエラーが出た。
```
╷
│ Error: updating EKS Node Group (welcome-study-eks-cluster:welcome_study_nodes-20251201090620580600000001) config: operation error EKS: UpdateNodegroupConfig, https response error StatusCode: 400, RequestID: 05870a2d-f7b4-4d79-a558-62f6959f22f8, InvalidParameterException: Minimum capacity 1 can't be greater than desired size 0
│ 
│   with module.eks.module.eks_managed_node_group["welcome_study_nodes"].aws_eks_node_group.this[0],
│   on .terraform/modules/eks/modules/eks-managed-node-group/main.tf line 447, in resource "aws_eks_node_group" "this":
│  447: resource "aws_eks_node_group" "this" {
│ 
╵
```
ちなみに、以下の起動用オートスケジューリングスケジュールを平日12:00ぐらいにapplyしようとしたンゴ
```tf
// 起動用
resource "aws_autoscaling_schedule" "start_schedule" {
  autoscaling_group_name = module.eks.eks_managed_node_groups_autoscaling_group_names[0]
  scheduled_action_name  = "start-schedule"
  start_time             = null
  end_time               = null
  desired_capacity       = 2
  min_size               = 1
  max_size               = 3
  time_zone              = "Asia/Tokyo"
  recurrence             = "0 9 * * 1-5" // 平日毎日09:00に起動
}
```
和訳すると
> エラー：EKSノードグループ（ID: welcome-study-eks-cluster...）の設定更新中にエラーが発生しました。
> EKSの操作「UpdateNodegroupConfig」に失敗しました（ステータスコード: 400）。
>無効なパラメータ例外：最小キャパシティ「1」を、希望サイズ「0」より大きくすることはできません。

```tf
desired_capacity       = 2
min_size               = 1
max_size               = 3
```
希望キャパシティを2にしてるのに、なぜか０って判定されてる。
どうしよう。gemini助けて

## EKSノードグループの基本設定を更新すればいいっぽい
どうやら、Terraformは現在の希望キャパシティの状態（夜間停止中の希望キャパシティ0）で定義通りに作ろうとするらしい。
つまり、Terraformが希望キャパシティはコード通りでない0（2を適用しようとしているけど）のままで作ろうとしていて、最小最大キャパシティはコード通りで作ろうとするらしい。
分かりづらいが、オートスケーリングスケジュールは関係なく、その前段のmanaged_node_groupだけ修正すればいい。
:::message
もしオートスケールで台数を増やしているなら、その状況に対して希望キャパシティが干渉してしまうとマズいという理屈らしい。つまり、Terraformの問題。というより正しい仕様って感じ。Terraformが私の酷いapplyを制御してくれていたみたい。
:::
module eksの以下が影響していて、terraformが希望キャパシティに関して干渉できないみたい 
https://github.com/search?q=repo%3Aterraform-aws-modules%2Fterraform-aws-eks%20%20ignore_changes&type=code

## 解決法
![](/images/aws-nodegropu-minimam-capcity/nodegroup-scaling.png)
そのため、対処法として現行のキャパシティをAWSコンソール側から変更するという強行手段をとった。
> EKS -> cluster -> 該当のクラスターを選択 -> コンピューティング -> ノードグループ -> 編集

今回は研修用だし特にオートスケールされている状況ではないため、問題ない。
とにかく現状のノードグループの設定をいじらないとダメみたいですね。

夜間停止状態だった
```tf
desired_capacity       = 0
min_size               = 0
max_size               = 0
```
を元の設定である
```tf
  eks_managed_node_groups = {
    welcome_study_nodes = {
        min_size = 1
        desired_size = 2
```
に手動で変更した。
その後、起動用のTerraformをapplyしたら成功した。

# 最後に
TerraformもAWSもKubernetesも難しすぎでしょ。
