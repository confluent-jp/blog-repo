---
title: KafkaとトランザクションとExactly Once
summary: KafkaにおけるトランザクションはリレーショナルDBにおけるそれとどう違うのか？KafkaトランザクションとExactly Onceとの関係は？
authors:
  - hashi
tags:
  - Kafka Producer
  - Exactly Once Semantics
  - Transaction
categories: 
  - Blog
  - Kafka Core
projects: 
date: '2023-09-09T00:00:00Z'
lastmod: '2023-09-09T00:00:00Z'
---

## はじめに
Kafkaの利用は[結果整合性](https://www.youtube.com/watch?v=9uCP3qHNbWw)の概念の浸透とその実践的な活用ユースケースの登場によって飛躍的に広がりました。それまでのリレーショナルモデルに見られる[ACID特性](https://ja.wikipedia.org/wiki/ACID_(%E3%82%B3%E3%83%B3%E3%83%94%E3%83%A5%E3%83%BC%E3%82%BF%E7%A7%91%E5%AD%A6))を前提とした整合性の管理ではなく、[Change Data Capture](https://martin.kleppmann.com/2015/06/02/change-capture-at-berlin-buzzwords.html)によって整合性を整理するという大きく異なるアプローチであり、既存の概念に挑戦するものでした。

リレーショナルモデルにおけるトランザクションではない「今ではない近い将来にはデータは整合性を保った状態で連携先に届く」というアプローチである為、Change Data Captureを活用したソリューションにトランザクションの概念が登場すると混乱を招く事も多くあります。

Kafkaもトランザクションをサポートしており、データを整合性を保ったままリアルタイムに扱う上で非常に重要な概念です。しかしながら、Kafkaのトランザクションの目的はリレーショナルモデルのそれとは大きく異なります。

このエントリは、Kafkaにおけるトランザクションがどういうものであるかの説明と、トランザクションにまつわる様々な誤解を解く事を目的としています。

## トランザクションとExactly Once
メッセージングの世界では「確実に1度だけメッセージをデリバリーする」という事が極めて難しいとされてきました。[^1] 一方、メッセージを (最低1回以上) 確実にデリバリーする手法は論理的にも実装的にも比較的容易である為、ほとんどのメッセージング基盤はこの手法を主に採用しています。OSSとして公開された当時のKafkaもその一つでした。データを送る側 (Producer) で1回だけ送るというのが難しい為、受け取り側 (Consumer) 側で重複メッセージの処理を行う必要があります。 [^2]

Kafkaが[Exactly Once Delivery](https://www.confluent.io/ja-jp/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)機能をサポートしたのはバージョン0.11です。この頃より、結果整合性を前提としたソリューションの土台となり得るKafkaの利用が飛躍的に広がりました。

Exactly Once Deliveryに関するKIPの正式名は[Exactly Once Delivery and Transactional Messaging](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging)であり、Idempotent Producerとトランザクションの双方を纏めて1つのKIPで定義しています。この為、Exactly Once Deliveryを達成する為には必ずトランザクションの導入が必要なよう誤解されている事も多いと思います。当然これら2つには強い関連性がある為同じKIP内で説明されていますが、それぞれ異なる機能であり、分解して理解する必要があります。

Idempotent Producerについては[こちらのブログエントリ](../idempotent-producer-and-max-inflight/)でご紹介していますが、具体的にはProducerがKafkaに対してExactly Onceでメッセージを送る為に必要な機能であり、Kafkaトランザクション機能とは異なります。つまり明示的にトランザクションを使用しなくても、ProducerからKafkaへの書き込みはExactly Onceに指定できます。

### Kafka Transaction
Kafkaトランザクション自体はリレーショナルDBにおけるトランザクションと近い思想を持つもので、異なるエンティティへの書き込み処理をアトミックに扱える機能ですが、Kafkaの世界では対象がTopicとなります。つまり、異なるTopicへの書き込みをCommit/Abortする事ができる機能です。利用方法もリレーショナルDB APIへのプログラムアクセスと似ており：
```java
void initTransactions() throws IllegalStateException;
void beginTransaction() throws ProducerFencedException;
void commitTransaction() throws ProducerFencedException;
void abortTransaction() throws ProducerFencedException;
void sendOffsetsToTransaction(Map<TopicPartition, OffsetAndMetadata> offsets,
     String consumerGroupId) throws ProducerFencedException;
```
これらメソッドを扱いプログラムコード内でアトミック処理の制御を行う事が出来ます。当然、```Topic A```と```Topic B```への書き込みをトランザクションで括ることも出来ます。しかしながら、リレーショナルモデル同様のトランザクションの使い方では、Kafkaへのトランザクション処理の導入が高い優先度で扱われたのも、この機能がExactly Once Deliveryと関連づけられ同一のKIP内で設計されていることも説明できません。

上記には一般的なトランザクション処理では見られないメソッドも含まれていますが
```java
void sendOffsetsToTransaction(Map<TopicPartition, OffsetAndMetadata> offsets,
     String consumerGroupId) throws ProducerFencedException;
```
このメソッドがExactly Once Deliveryとトランザクションを関連付ける重要な役割を担っています。

### sendOffsetsToTransaction
このメソッドが何をするかは名前からもある程度推測できますが、「処理した一連のConsumer OffsetをConsumer Group Coordinatorに送りつつ、 **現在のトランザクションに関連付ける** 」メソッドです。少々アクロバティックな処理ですが、この処理をトランザクションAPIに定義するには必要性があります。

このメソッドは、全く異なる2つの処理を1メソッドに纏めるというタブーに近いメソッドです。さらに注意すべきなのは、トランザクションはProducerに関わる機能であり、Consumer OffsetはConsumerに関わる機能です。つまりこのメソッドはProducerでもありConsumerでもある処理でしか存在意義が無いメソッドです。

一般的なメッセージングモデルではあまり検討しないこの処理ですが、Kafkaのエコシステムでは[ストリーム処理](https://www.confluent.io/ja-jp/online-talks/benefits-of-stream-processing-and-apache-kafka-use-cases-on-demand/)における根本的かつ重要な要件となります。

## ストリーム処理とExactly Once
### ストリーム処理
ストリーム処理とは、一般的なバッチ処理同様にInputとOutputを持つデータ処理を、中間的なストレージを経由する事により一連のデータフローとして形成するアプローチです。バッチとの違いは、その中間的なストレージがcsvファイルやDBの中間テーブルではなくKafka Topicを利用する事であり、これによりバッチ同様のデータフローをリアルタイム[^3]に実行する事が出来る点です。

![ストリーム処理 - 論理フロー](blogs/kafka-transaction-how-it-works/stream-processing-logical-view.png)

この例ではSourceから発行されたイベントを、エンリッチしてアプリやDBに格納するフローと、集約した後ダッシュボードに転送するフローを表現しています。イベントはKafkaに発行されてから、Kafka内で変更/加工された後にSinkへとリアルタイムに繋げるデータフローとなっています。

これはあくまで論理的なフローですが、実際にはデータフロー内の処理は全てKafkaとの通信で成り立っています。

![ストリーム処理 - コミュニケーションフロー](blogs/kafka-transaction-how-it-works/stream-processing-communication-view.png)

SourceはProducerとして機能し、DBやダッシュボードへの転送はSink Connectorを利用しています。これらデータフローの両端はProducerもしくはConsumerのいずれかの役割を果たします。

その他の処理はイベントを受け取り、加工の上次の処理に渡す為、ProducerでありConsumerである必要があります。より具体的には「前処理がProducerでイベント送ったTopicを、その次の処理がConsumeする」、この繰り返しでデータフローのトポロジーを形成します。このトポロジーをTopicも含めたフローで表現すると：

![ストリーム処理におけるTopic](blogs/kafka-transaction-how-it-works/stream-procerssing-and-topics.png)

このように分解できます。ここではProduceを点線、Consumeを実線で表現しています。処理自体はKafkaのTopicを境に完全に分離された構成となっており、バッチ処理における中間ストレージと同じ役割を果たしています。

### Exactly Once
この処理のEnd-to-EndでExactly Onceを達成するには？という命題の為にKafka Transactionは定義されています。

Kafka Consumerは、メッセージの処理後にConsumer Offsetを```commitSync/commitAsync```というメソッドを使って更新します。このオフセットコミットが行われる事により、仮に処理後にConsumerのプロセスが落ちたとしても、新しいプロセスが引き継いで処理を継続する事が出来ます。

しかしながら、処理後 (ストリーム処理ではConsumeし、データの処理を実行し、別のTopicにProduce後) に何らかの障害によってプロセスを失った場合、引き継いだプロセスはコミットされる前のオフセットをもとに処理、つまり同じメッセージを消費し再処理することとなります。これではProduceに関してはIdempotent Producerの設定によってExactly OnceでKafkaに書き込めても、End-to-EndのフローではExcatly Onceを保証出来ません。

![ストリーム処理とトランザクション](blogs/kafka-transaction-how-it-works/topics-and-transaction.png)

Kafkaにおけるトランザクション、そして中でも先ほど言及した```sendOffsetsToTransaction```はまさしくこのProduceとConsumer Offsetのコミットをアトミックな処理としてに定義する事ができます。これにより、どのタイミングで障害が発生してもEnd-to-EndでExactly Once Deliveryを達成する事が出来ます。[^4]

### まとめ
Kafka TransactionとIdempotent Producerは同じKIP内で定義されており、また併せて説明される事が多い機能ではありますが、使われる場所と用途は全く異なります。ただこれらはKafkaにおけるExactly Once Deliveryにおいてお互いを補完するものであり、双方が揃って初めてEnd-to-EndのExactly Once Deliveryが達成出来る事が分かります。

[^1]:分散システムにおける[二人の将軍問題](https://ja.wikipedia.org/wiki/%E4%BA%8C%E4%BA%BA%E3%81%AE%E5%B0%86%E8%BB%8D%E5%95%8F%E9%A1%8C)等を用いて言及されています。
[^2]:この為Consumer側で冪等性を確保した処理が求められます。
[^3]:一般的に「リアルタイム処理」とは数ミリ秒誤差のものを指す為、厳密には準リアルタイムと呼ぶのが相応しいかも知れません。Kafkaのストリーム処理におけるEnd-to-Endのレイテンシは、処理にもよりますが数百ミリ秒から数秒程度のレイテンシとなります。
[^4]:図にもありますが、Consumerも未コミット状態のデータにアクセスしてはいけないので、分離レベルをRead Committedに指定する必要があります。ここはリレーショナルモデルの概念をそのまま踏襲しています。
