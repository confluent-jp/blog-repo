---
title: Confluent Platform 7.5リリース
summary: Apache Kafka ３.5、双方向Cluster Linking、REST Proxy Producer API v3、Control CenterのOpenID Connectアクセス、などなど。
authors:
  - hashi
tags:
  - Cluster Linking
  - Control Cluster
  - OpenID Connect
  - REST Proxy
categories: 
  - Announcement
  - Confluent Platform
projects: []
date: '2023-08-31T00:00:00Z'
lastmod: '2023-08-31T00:00:00Z'

links:
url_code: ''
url_pdf: ''
url_slides: ''
url_video: ''
---
Confluent Platform 7.5がリリースされました。 [Confluent Blog](https://www.confluent.io/blog/introducing-confluent-platform-7-5/) 内包されるApache Kafkaのバージョンは[3.5](https://www.youtube.com/watch?v=BVxDFL5iTx8)となります。

コアエンジンであるApache Kafkaのアップグレードだけでなく、エンタープライズ ソリューションとしてのConfluent Platformとしての機能追加や改善も含まれています。

### SSO for Control Center (C3) for Confluent Platform
Confluent Control CenterはConfluent Platformにおける管理ポータルとしての役割から、ごく一部のSREメンバーからのみアクセスされるという特殊なコンポーネントです。この為これまではアクセス制御のアプローチについては少し限定的でしたが、Broker等と同様OAuth2ベースの認証/認可の方法でアクセスする事が可能となりました。
![OIDC SSO to Conntrol Center](blogs/confluent-platform-7.5-announcement/sso.png)

### Confluent REST Proxy Produce API v3
Confluent REST ProxyはKafka BrokerへのRESTベースのアクセスを可能としており、Confluent Platform/Cloudの双方で幅広く活用されています。一方、通常のKafkaプロトコルベースのアクセスに比べると制限もあり、これまでも段階的に改善がなされています。今回のProduce API v3では：
- カスタムヘッダーの追加 (トレーシングID、等)
- KeyとValueで異なるシリアライザの設定

が可能となります。
![REST Proxy Produce API v3](blogs/confluent-platform-7.5-announcement/rest-proxy-produce-v3.png)

### Bidirectional Cluster Linking
双方向のCluster Linkingの設定が可能となりました。これまでも一方向のリンクを2本貼れば実際のレプリケーションを双方向にすることは可能でした。Consuemr観点ではローカルのTopicとMirror Topicそれぞれ個別のTopicを同時にConsumeするモデルであり、それぞれへのOffset Commitを実行します。この際、Mirror TopicへのOffset CommitはソースとなるTopicには反映されないので、障害時には部分的なConsumer Offset情報しか連携されていない状況となります。
![2 Unidirectional Cluster Linking](blogs/confluent-platform-7.5-announcement/uni-directional-cluster-linking-offset.png)
双方向のCluster Linkingは双方向へのリンクが1セットとして扱われる為、双方のクラスタでのOffset Commit情報も合わせて同期されます。
![Bidirectional Cluster Linking](blogs/confluent-platform-7.5-announcement/bidirectional-cluster-linking-cp-7-5.png)

### 参考
- [Introducing Confluent Platform 7.5 (Confluent Blog)](https://www.confluent.io/blog/introducing-confluent-platform-7-5/)
- [Confluent Platform 7.5 Release Notes](https://docs.confluent.io/platform/7.5/release-notes/index.html)
- [REST Proxy](https://docs.confluent.io/platform/current/kafka-rest/index.html)
- [Single Sign-On (SSO) for Confluent Control Center](https://docs.confluent.io/platform/7.5/control-center/security/sso/overview.html#sso-for-c3)
- [Cluster Linking - Bidirectional Mode](https://docs.confluent.io/cloud/current/multi-cloud/cluster-linking/cluster-links-cc.html#bidirectional-mode)

