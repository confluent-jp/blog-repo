---
title: Streaming Audio - Kafkaに本当にあった(まだある)ヤバいバグ5選
subtitle: 
summary: Anna McDonaldによるKafkaのヤバいバグに関するトークです。日本語字幕でどうぞ。
authors:
  - hashi
tags:
  - Bugs 
categories: 
  - Blog
  - Kafka Core
projects: []
date: '2023-07-29T00:00:00Z'
lastmod: '2023-07-29T00:00:00Z'

links:
url_code: ''
url_pdf: ''
url_slides: ''
url_video: 'https://www.youtube.com/watch?v=Az1oB6YsriE&pp=ugMICgJqYRABGAE%3D'
---

## はじめに
[Streaming Audio](https://www.youtube.com/watch?v=yFlvWRwRTT8&list=PLa7VYi0yPIH1B0i7mhzVi78TIkKSd-0vE)はConfluentがPodcast&YouTubeシリーズとして提供しています。毎回ゲストを迎え様々なトピックについてフリーにディスカッションするポッドキャストで、Kafka初期開発メンバーのJun RaoやKRaftの開発メンバー、Kafkaのリアルユーザー等様々なゲストスピーカーが参加します。中でも「アナネキ」ことAnna McDonald (Technical Voice of CUstomer @Confluent)登場回は毎回必見で、いつも何か新しい発見があります。

今回はその彼女の登場回の中でも最も最近の回のご紹介です：お題は「Kafkaに本当にあったヤバいバグ5選[^1]」です。(オリジナルの公開は2022/12/21) このトークで紹介されたJIRAバグの一覧を用意しました。結構最近になってようやく入ったものや、まだ直っていないものもあります。Kafkaのバージョンはなるべく追従する事を強くお勧めしていますが、ここにあるのは全体の一部で、なかなかに怖いバグへの修正も入っています。

あなたのKafkaクラスタはほんとに大丈夫です？

#### [KAFKA-10888: Sticky partition leads to uneven product msg, resulting in abnormal delays in some partitions](https://issues.apache.org/jira/browse/KAFKA-10888)
> Status: Resolved (3.0.0)

Sticky Partitionerを使用時、Partition間の処理数に大きな偏りが出る様な状況となり特定のPartitionのスループットが極端に下がる事がある: 場合によってはリカバリ不能なほどProducer側のバッチが肥大化する。

#### [KAFKA-9648: Add configuration to adjust listen backlog size for Acceptor](https://issues.apache.org/jira/browse/KAFKA-9648)
> Status: Resolved (3.2.0)

OSがLinuxの場合に発生。ローリングアップグレード等の際、BrokerからPartition Leaderが他のBrokerに移る、もしくは移ったのちに元のBrokerに戻る (Preferred Leader Election) が発生。この際Partitionに関するメタデータ更新が行われる為これらPartitionを参照するクライアントから一斉に再接続のリクエストが送られる。状況によってはLinuxのSYN cookieの機能が動きTCPバッファーが制限されスループットが大幅に低下する。これは再接続しない限り復旧しない。

#### [KAFKA-12686: Race condition in AlterIsr response handling](https://issues.apache.org/jira/browse/KAFKA-12686)
> Status: Resolved (3.0.0)

Partition.scala内の処理において、AlterIsrResponseとLeaderAndIsrRequestのレースコンディションが起因。クラスタサイズが小さくPartition数が多い場合、Brokerノードの変更時に大量のPartition変更が発生する。この際AlterIsrManagerがペンディング状態のリクエストをクリアしてしまう為、AlterIsrResponseが戻ってきた際に処理中 (in Flight) であるのに処理タスク (Pending) が無いという矛盾状態が発生する。

#### [KAFKA-12964: Corrupt segment recovery can delete new producer state snapshots](https://issues.apache.org/jira/browse/KAFKA-12964)
> Status: Resolved (3.0.0)

Brokerの停止時、猶予時間内に終了しない場合には Unclearn Shutdownと判断される。この際Broker復帰時に残っていたセグメントは不要と判断され非同期で削除が実行される。この削除が完了する前に同じオフセットのセグメントが書き込まれる状態となると、新しいProducer Stateスナップショットが誤って削除される事がある。

#### [KAFKA-14334: DelayedFetch purgatory not completed when appending as follower](https://issues.apache.org/jira/browse/KAFKA-14334)
> Status: Resolved (3.4.0, 3.3.2)

ConsumerがPulgatoryからフェッチするケースにおいて、通常通りPartitionリーダーからフェッチする場合には正しくフェッチの完了が認識される。しかしConsumerがフォローワーがフェッチする設定としている([KIP-932](https://cwiki.apache.org/confluence/display/KAFKA/KIP-392%3A+Allow+consumers+to+fetch+from+closest+replica))場合、フォローワーのPartitionはPulgatoryに存在しない為フェッチ出来ずタイムアウトする。

[^1]: オリジナルは6選であり、ここではそのうち[KAFKA-9211: Kafka upgrade 2.3.0 may cause tcp delay ack(Congestion Control)](https://issues.apache.org/jira/browse/KAFKA-9211)も含んでいますが、トークの中ではKafka-9646の中で合わせて語られているので割愛しました。


