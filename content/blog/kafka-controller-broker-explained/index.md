---
title: 図解 Kafka Controller Broker
summary: 分散ストリーミング基盤に起こるカオスを水際で抑えるController Brokerの仕組み by Kafkaコミッタ。
authors:
  - stanislav
  - hashi
tags:
  - Translated
categories: 
  - Kafka Core
  - Blog
projects: 

date: '2023-08-20T00:00:00Z'
lastmod: '2023-08-20T00:00:00Z'
links:
url_code: ''
url_pdf: ''
url_slides: ''
url_video: ''
---
> このブログエントリはKafkaコミッタである Stanislav Kozlovski([𝕏](https://https://twitter.com/BdKozlovski)|[Ln](https://www.linkedin.com/in/stanislavkozlovski/)) のサイトで2018/10/31に公開された[Apache Kafka’s Distributed System Firefighter — The Controller Broker](https://stanislavkozlovski.medium.com/apache-kafkas-distributed-system-firefighter-the-controller-broker-1afca1eae302)の日本語訳です。Stanislav本人の了承を得て翻訳/公開しています。

## はじめに
Kafkaは成長を続ける分散ストリーミング基盤です。現時点での業界デファクト技術であり、広がるデータパイプラインの利用に対しても拡張しつつ安定的に稼働することが出来ます。もしKafkaの概要についてもう少し知りたい方は[A Thorough Introduction To Apache Kafka](https://hackernoon.com/thorough-introduction-to-apache-kafka-6fbf2989bbc1)をご覧ください。

この記事を書く中で、このKafkaの安定稼働を支える中の仕組みについて書きたいと思うようになりました。

このエントリではKafkaにおけるControllerの概念 - Kafkaという分散基盤を健康的に稼働し続ける使命を支える機能についてご紹介します。

## Controller Broker
分散システムは常に協調の中で稼働し続ける必要があります。何かしらのイベントがクラスタで発生した場合、クラスタ内のコンポーネントは同調してそれに反応しなければいけません。その中で、クラスタとしてどう反応するべきなのか、Brokerは何をすべきなのかを指示する存在が必要です。

その役割を担うのがControllerです。
Controller自身は複雑な仕組みではありません - ControllerもBrokerであり、通常のBrokerとしての役割と合わせて追加の役割も持つBrokerです。つまりController BrokerもPartitionを制御し、ReadとWriteのリクエストに対応し、裏ではレプリケーションに参加します。

今追加の役割の中で最も重要なのは、クラスタ内のBrokerノードの管理であり、Brokerが追加、クラスタから離脱、もしくは障害が発生した際に適切にそのメンバーシップを管理することです。これにはPartitionのリバランスや新たなPartion Leaderの特定も含まれます。

KafkaクラスタにはControllerが常に稼働し、唯一1つのControllerのみ稼働します。[^1]

## Controllerの役割
Controller Brokerは複数の複数を担います。Topicの作成/削除、Partitionの追加 (とLeaderの特定)、BrokerがClusterを離脱した際の諸々の制御等様々ありますが、基本的にはクラスタにおける管理者として振る舞います。

### Brokerノードの離脱
エラーや計画的な停止によってBrokerノードがクラスタから離脱した際、そのBrokerノードにLeaderのあったPartitionにはアクセス出来なくなります。 (クライアントはWrite/Readのいずれであっても常にLeader Partitionにのみアクセスします。[^2]) この為Broker離脱時のダウンタイムを短縮するには、いかに迅速に新たなLeaderを選出するかが重要になります。

Controller Brokerは他のBrokerノードが離脱した際に対処します。Zookeeperには[Zookeeper Watch](https://zookeeper.apache.org/doc/r3.4.8/zookeeperProgrammers.html#ch_zkWatches)と呼ばれる特定データの変更時に登録者に対して通知をする機能で、ControllerはこのZookeeper Watchを利用してBrokerの離脱を検知します。Zookeeper WatchはBroker離脱時のトリガーとして働くKafkaにとって非常に重要な機能です。

ここにおける「特定データ」とはBrokerデータの集合です。

下にあるのはBroker 2の[Zookeeper Session](https://zookeeper.apache.org/doc/r3.4.8/zookeeperProgrammers.html#ch_zkSessions)が無効化される事によりBroker 2のIDがリストから削除された場合の図解です。(Kafka BrokerはZookeeperへのハートビートを送り続けるが、遅れなくなるとセッションが無効化する)

![Zookeeper Watch and Controller](blogs/kafka-controller-broker-explained/zookeeper-watch.webp)

ControllerはこのBroker離脱の通知を受け取り作業に取り掛かりますが、まずはBrokerの離脱によって影響を受けたPartitionの新たなLeaderを決定します。この後、クラスタ内の全てのBrokerに対して通知し、このリクエストを受け取った各Partition毎にLeaderになったりFollowerとしてLeaderに[LeaderAndIsr](https://kafka.apache.org/protocol#The_Messages_LeaderAndIsr)リクエストを送付します。

### Brokerノードのクラスタ復帰
適切なPartition Leaderの配置はKafkaクラスタの負荷分散の上で重要な要素です。上記で説明したとおり、Brokerノードがクラスタを離脱した際には他のBrokerが代わって対応する必要があります。この場合Brokerは当初の想定以上のPartitionを各々が担うことになり、クラスタ全体の健全性やパフォーマンスに少なからず影響を及ぼします。当然なるべく早くバランスを取り戻す必要があります。

Kafkaは元々のPartitionアサインメントが、ある程度「適切」であるという想定を持っています。このアサインメントにおけるPartition Leaderはいわゆる *Preferreed Leader(優先リーダー)* として認識され、最初にそのPartitionが追加された時のPartition Leaderを指します。合わせてKafkaは[インフラ構成としてのラックやAvailability Zoneを意識したPartition配置の機能](https://cwiki.apache.org/confluence/display/KAFKA/KIP-36+Rack+aware+replica+assignment) (ラック/AZ障害耐性を確保する為にLeaderとFollowerを別のラック/AZに配置する) もサポートしており、Partition Leaderの配置はクラスタの信頼性に大きく寄与しています。

デフォルトでは```auto.leader.rebalance.enabled=true```となっており、KafkaはPreferred Leaderが存在し、かつ実際のPartition Leaderではない場合にはPreferred Leaderを再選出します。

Brokerノードのクラスタからの離脱も、多くの場合一時的であり、ある一定時間経過後に離脱したBrokerノードは再度クラスタメンバーとして復帰します。この為Brokerノードが離脱した際にも関連するメタデータは即座に削除されず、Follower Partitionも新たにアサインされません。

注意点として、再参加したBrokerノードも直ぐにPartition Leaderとして再選出される訳ではなく、その候補となる為には別の条件も必要となります。

### In-Sync Replicas
In-Sync Replica (ISR) は状態がPartition Leaderと同じ Followerを指します。言い方を変えると、ISRはそのLeaderのレプリケーションが追いついている状態にあります。Partion LeaderはどのFollowerがISRでどのFollowerがそうではないかをトラックする必要があり、その状態はZookeeperに永続化されます。

Kafkaの障害耐性と可用性の保証はデータのレプリケーションに基づいており、kafkaが機能するには常に十分なISRが確保されているかが極めて重要です。

FollowerがLeaderに昇格するにはまずISRである必要があります。全てのPartitionにはISRのリストがあり、Partition LeaderとControllerによって管理されています。ISRから新たなPartition Leaderを選出する処理は *Clean Leader Election* と呼ばれています。一方ユーザーにはこれとは異なる方法でLeaderを昇格させる事も可能で、Partion Leaderがクラスタを離脱した際にはISRではないFollowerを昇格させる事も可能です。これはLeaderもISRも存在しないという状況において、データ整合性より可用性を優先させる必要がある稀なケースです。





![2 Minutes Streaming](blogs/kafka-acks-explained/two-minites-streaming.png)
このエントリの著者である[Stanislav Kozlovski](../../authors/stanislav/) は[2 Minute Streaming](https://2minutestreaming.com/)というKafkaに関する隔週ニュースレターを発行しています。是非購読してみてください。

[^1]:(訳者注)これはZookeeperをクラスタのメタデータ管理に利用する場合の話で、[KRaft](https://cwiki.apache.org/confluence/display/KAFKA/KIP-595%3A+A+Raft+Protocol+for+the+Metadata+Quorum)と呼ばれるRaftベースのコンセンサスモデルでは3以上のControllerが存在する必要があります。
[^2]:(訳者注)Kafka2.4より、最寄りのレプリカからReadする([KIP-392](https://cwiki.apache.org/confluence/display/KAFKA/KIP-392%3A+Allow+consumers+to+fetch+from+closest+replica))機能が提供されています。