---
title: Apache Kafka 3.6 アップデート
summary: Apache Kafkaの最新バージョンが公開されました。Tiered Storageがアーリーアクセスとして登場した他、Zookeeperのアップデート、KRaft Metadata Transaction等様々な新機能が追加されました。
authors:
  - hashi
tags:
  - Tiered Storage
  - KRaft
  - Kafka Streams
  - Kafka Connect
categories: 
  - Announcement
  - Kafka Core
projects: []
date: '2023-10-12T00:00:00Z'
lastmod: '2023-10-12T00:00:00Z'

links:
url_code: ''
url_pdf: ''
url_slides: ''
url_video: 'https://www.youtube.com/watch?v=GW3625sEJyc'
---
Apache Kafkaの新バージョン3.6が公開されました。
ZookeeperモードからKRaftモードへの移行ではありますが、KRaftの強化だけでなく新たな機能も多く追加されております。詳細は[Confluentのアナウンスメント](https://www.confluent.io/blog/introducing-apache-kafka-3-6/)と[YouTube](https://www.youtube.com/watch?v=GW3625sEJyc)で説明されています。より詳細には[本家のリリースノート](https://kafka.apache.org/blog#apache_kafka_360_release_announcement)には全ての関連kIPのリストが公開されています。

本エントリでは、中でも重要なKIPについてご紹介します。

### [KIP-405: Kafka Tiered Storage (Early Access)](https://cwiki.apache.org/confluence/display/KAFKA/KIP-405%3A+Kafka+Tiered+Storage)
[こちらのブログエントリ](../kip405-why-tiered-storage-important/)でもご紹介していたTiered Storageがアーリーアクセスとして利用可能となりました。単純に古いセグメントがオブジェクトストレージに退避されるだけでなく、既存のKafkaの設計やパフォーマンスへの影響を与えずに、Kafka自身がよりクラウドネイティブな姿へと変わる上で重要な機能です。

今回3.6に登場したバージョンはまだ本番環境における利用を想定していない旨にご留意ください。機能の安定性だけでなく、JBODやCompacted Topic等機能制限もあります。既存Topicもバージョンを3.6にアップグレードすればTiered Storageに変更出来ますが、2.8.0より前に作成されたTopicには適用出来ない点もご注意下さい。アーリーアクセス版の制限はこちらの[Tiered Storage アーリーアクセスリリースノート](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Tiered+Storage+Early+Access+Release+Notes)に記載されています。

### [KIP-868 Metadata Transactions](https://cwiki.apache.org/confluence/display/KAFKA/KIP-868+Metadata+Transactions)
[KRaft](https://developer.confluent.io/learn/kraft/)の内部処理に関する改善です。KRaftではメタデータの更新時に関連レコード (例：Topic登録時の全Partitionのレコード) をアトミックに更新する仕様となっています。この為Controllerが処理中に障害に陥った場合でも部分的なメタデータの更新がなされないようになっています。

一方このバッチサイズはKRaftのフェッチサイズが上限となっており、アップデート前ではこのサイズは8kbとなっています。この為非常に大きなメタデータの更新時にはフェッチ上限を超えるバッチが生成される可能性がありました。

この改善で新たにメタデータにトランザクションの概念が導入され、トランザクションの開始/終了等のマーカーレコードを挿入するようになります。これによりKRaftのフェッチサイズを超える更新バッチサイズになった場合でも処理が可能となります。

### [KIP-941: Range queries to accept null lower and upper bounds](https://cwiki.apache.org/confluence/display/KAFKA/KIP-941%3A+Range+queries+to+accept+null+lower+and+upper+bounds)
Kafka StreamsにてマテリアライズしたState Storeに対してアクセスするには[Interactive Query](https://docs.confluent.io/platform/current/streams/developer-guide/interactive-queries.html)を利用します。これにより、アクセスするデータが分散配置されているKafka Streamsのどのインスタンスにて保存されているのかを意識せずとも適切なデータを取得する事が出来ます。

一方内部ではそれぞれのデータはKafka Streamsインスタンスに部分的に保存されている為、レンジ指定をして取得する場合には処理に大きな負荷がかかります。この為レンジ指定のクエリは制限が多く、アップデート前ではnullを指定した取得が出来ませんでした。この為：
```java
private RangeQuery<String, ValueAndTimestamp<StockTransactionAggregation>> createRangeQuery(String lower, String upper) {
        if (isBlank(lower) && isBlank(upper)) {
            return RangeQuery.withNoBounds();
        } else if (!isBlank(lower) && isBlank(upper)) {
            return RangeQuery.withLowerBound(lower);
        } else if (isBlank(lower) && !isBlank(upper)) {
            return RangeQuery.withUpperBound(upper);
        } else {
            return RangeQuery.withRange(lower, upper);
        }
    }
```
このような回避的なコーディングが必要でした。

今回レンジクエリにnull指定が出来るようになった事により：
```java
RangeQuery.withRange(lower, upper);
```
これだけでnullを回避した実装が可能となります。

### [KIP-875: First-class offsets support in Kafka Connect](https://cwiki.apache.org/confluence/display/KAFKA/KIP-875%3A+First-class+offsets+support+in+Kafka+Connect)
Kafka Connectはその処理状況をKafkaネイティブにオフセットを管理する事により把握/管理しています。Connectorタスクが異常終了した場合でも、コミットされたオフセットを元に継続処理できるので、Connector自身には独自のステート管理のストレージ等が無くとも障害耐性を確保する事が出来ています。

一方このオフセットはKafka上では参照できるもののKafka Connectとしては外部からアクセス出来るようにはなっていませんでした。何かしらの理由でオフセットを制御したい（特定レコードレンジを飛ばしたい、あるオフセットから再読み込みしたい、etc）場合にはハック的にKafka上のオフセット用Topicをいじる必要がありました。

この改善によってKafka Connect API経由でオフセットの取得、更新、削除が可能となります。

### おわりに
Apache Kafka 3.6にはその他多くの改善が含まれています。今回のエントリではその一部しか触れていませんが、是非本家の[リリースノート](https://kafka.apache.org/blog#apache_kafka_360_release_announcement)も併せてご参照ください。
