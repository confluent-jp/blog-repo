---
title: Confluent Cloud Q3'23 Launch
summary: 2023年第3四半期、Confluent Cloud新機能のご紹介です。
authors:
  - hashi
tags:
  - Apache Flink
  - Enterprise Clusters
  - Terraform Provider
  - Cluster Linking
categories: 
  - Announcement
  - Confluent Cloud
projects: []
date: '2023-10-01T00:00:00Z'
lastmod: '2023-10-01T00:00:00Z'

links:
url_code: ''
url_pdf: ''
url_slides: ''
url_video: 'https://www.youtube.com/watch?v=TS00diWO5Ak'
---
Confluent Cloudはマネージドのプラットフォーム提供である為、様々な機能追加や改善は自動的に適用されます。これら改善はコアであるApache Kafkaのバージョンアップに限らず、またプラットフォーム製品であるConfluent Platformの機能にも限定されず、Confluent Cloud独自の機能も様々追加されています。

Quarterly Launchは、そんなConfluent Cloudの新規機能を四半期毎にまとめてご紹介する[ブログ](https://www.confluent.io/blog/build-deploy-consume-data-pipelines/)と[YouTube](https://www.youtube.com/watch?v=TS00diWO5Ak)エントリを指します。今回は[Current 2023](https://www.confluent.io/events/current/)の開催を待った為だいぶ遅くなってしまいましたが、改めてそのハイライトをご紹介します。

### [Apache Flink® on Confluent Cloud (Open Preview)](https://www.confluent.io/blog/build-deploy-consume-data-pipelines/#flink-on-cloud)
![Flink on Confluent Cloud](blogs/confluent-cloud-23Q3-launch/flink-on-confluent-cloud.png)
来年サービス提供開始予定のApache Flink on Confluent Cloudがオープンプレビューとして公開されました。提供インターフェースはFlink SQLのみ。現時点では[既知の機能制限](https://docs.confluent.io/cloud/current/flink/reference/op-supported-features-and-limitations.html#feature-limitations)があり、この為本番利用には向きません。また現時点で[利用可能なクラウド/リージョン](https://docs.confluent.io/cloud/current/flink/reference/op-supported-features-and-limitations.html#cloud-regions)は限定的ではあります。ただ、今日もうお試しいただけます。

### [Enterprise clusters](https://www.confluent.io/blog/build-deploy-consume-data-pipelines/#enterprise-clusters)
![Enterprise Clusters](blogs/confluent-cloud-23Q3-launch/enterprise-clusters.png)
Confluent Cloudのサーバーレスなクラスタ提供に新たにEnterpriseというオプションが追加されました。Basic、Standardといった既存のサーバーレスクラスタと異なり閉塞ネットワーク接続[^1]を可能としており、併せて標準でSLA 99.99%、最大1GBpsのスループット(Ingress/Egress合算)をサポートしています。

残念ながらローンチ時点ではサポートされているリージョンは限定的[^2]ですが、以降継続して拡張予定となっています。

### [Confluent Terraform provider updates](https://www.confluent.io/blog/build-deploy-consume-data-pipelines/#confluent-terraform-provider)
Confluent Terraform ProviderがHashiCorp Sentinel統合をサポートしました。これによりPolicy-as-Codeによる運用にConfluent Cloudを統合することが可能となります。

また、新たにResource Importer機能を提供開始しました。これにより既存のConfluent CloudからTerraformの構成 (main.tf) ならびに状態 (terraform.tfstate) を逆生成する事が可能となります。

### その他
- 先日発表した[Confluent Platform 7.5](../confluent-platform-7.5-announcement/)でもご紹介した双方向Cluster LinkingがConfluent Cloudでもサポートされております。
- PrivateLink接続のConfluent Cloudクラスタ同士を[直接Cluster Linkingで接続可能](https://docs.confluent.io/cloud/current/multi-cloud/cluster-linking/private-networking.html)することが可能となりました。

その他にも新たな機能が追加されておりますが、その全貌ならびに個々の詳細につきましては[Confluentブログのアナウンスメント](https://www.confluent.io/blog/build-deploy-consume-data-pipelines/)をご覧ください。

[^1]:2023年9月ローンチ時点では、AWS PrivateLink経由の接続のみサポートしております。その他クラウドの接続形態は後日提供となります。
[^2]:2023年9月ローンチ時点では、AWSのus-east-2(Ohio)、us-west-2(Oregon)、ap-southeast-1(Singapore)等8リージョンでのみ提供開始となっています。日本リージョンでの利用開始は現時点では未定です。