---
description: １年間開発APIGateway内容を整理
---

# APIGateway

## [AWS コンテナサービス「Fargate」「ECS」「EKS」の違いを解説](https://xn--o9j8h1c9hb5756dt0ua226amc1a.com/?p=2025)

## 今回開発したAWSコンテナサービスはECS Fargate

> 理由としてはEKSは未だサービス開始して間もない事と、SLA条件がECS Fargateの方が良かったことでECS Fargateを選択
>
> * ECS FargateのSLA：99.99% [https://aws.amazon.com/jp/about-aws/whats-new/2017/12/amazon-compute-service-level-agreement-extended-to-amazon-ecs-and-aws-fargate/](https://aws.amazon.com/jp/about-aws/whats-new/2017/12/amazon-compute-service-level-agreement-extended-to-amazon-ecs-and-aws-fargate/)
> * EKSのSLA：99.9% -&gt; 99.95% upgrade [https://aws.amazon.com/jp/about-aws/whats-new/2020/03/amazon-eks-updates-level-agreement-to-99-95/](https://aws.amazon.com/jp/about-aws/whats-new/2020/03/amazon-eks-updates-level-agreement-to-99-95/)

## AWS構成図

## デモンストレーション

* CloudFormationを利用して環境構築 proxy作成所要時間：10分 mock作成所要時間：8分 postman作成所要時間：3分
* gatlingを用いてproxyサーバに負荷をかける：20分 ECS Fargateのタスクが1-&gt;2までスケールアウトされてスケールインまで確認 429の流量制限を確認
* postmanを用いてIT試験実施、実施した結果がSlackに通知されることを確認：5分

## プロジェクト管理は[スクラム](https://www.ogis-ri.co.jp/column/agile/agilescrum01.html)



