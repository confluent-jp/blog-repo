---
title: Kafka Acks再入門
summary: KafkaのAcksを完全に理解する by Kafkaコミッタ。
authors:
  - stanislav
  - hashi
tags:
  - Kafka Client
  - Translated
categories: 
  - Kafka Core
  - Blog
projects: 

date: '2023-08-01T00:00:00Z'
lastmod: '2023-08-01T00:00:00Z'
links:
url_code: ''
url_pdf: ''
url_slides: ''
url_video: ''
---
> このブログエントリはKafkaコミッタである Stanislav Kozlovski([𝕏](https://https://twitter.com/BdKozlovski)|[Ln](https://www.linkedin.com/in/stanislavkozlovski/)) のサイトで2022/11/06に公開された[Kafka Acks Explained](https://www.linkedin.com/pulse/kafka-acks-explained-stanislav-kozlovski/)の日本語訳です。Stanislav本人の了承を得て翻訳/公開しています。

Kafkaに関する仕事を始めて4年になりますが、経験上未だに2つの設定について広く誤解されていると感じる事があります。それは```acks```と```min.insync.replicas```であり、さらにはこの2つの設定がどう影響し合うかについてです。このエントリはこの非常に重要な2つの誤解を解き、適切に理解してもらう事を目的としています。

## Replication
この2つの設定を理解するためにはまずKafka Replicationプロトコルについて少しおさらいする必要があります。

このブログの読者の皆さんはある程度Kafkaについてご存知だと想定しています - もし自信がない場合はぜひ[Thorough Introduction to Apache Kafka](https://medium.com/hackernoon/thorough-introduction-to-apache-kafka-6fbf2989bbc1)もご参照ください。

各Partitionには1つのLeader Broker(1)と複数のFollower Broker(N)がアサインされます。この複製の数は```replication.factor```で設定する事ができ(1+N)つまり総数を表します。つまりこの設定では「対象となるPartitionに対してクラスタ上で何個の複製が出来るか」を指定します。

デフォルトであり通常推奨する設定値は```3```です。
![Replication Factor](blogs/kafka-acks-explained/replication-factor.png)
ProducerクライアントはLeader Brokerにのみ書き込みに行きます - つまりFollower Brokerへのレプリケーションは非同期に行われます。ここで分散システムとして考慮しないといけないのは、何かしらの方法で「これらレプリケーションされる処理がどのようにLeaderに追従すべきか」を指定する方法です。具体的には「Leaderに書き込まれた更新がFollowerにも反映されているか否か」です。

## In-Sync Replicas
in-sync replica(ISR)は対象Partitionの最新状態と同期が取れているBrokerを指します。当然Leaderは常にISRとなり、Followerの場合はLeaderの更新に追い付き同期が取れた状態のもののみISRとなります。仮にFollowerがLeaderに追従できなくなった場合、そのFollowerはISRではなくなります。
![In-Sync Replicas](blogs/kafka-acks-explained/in-sync-replicas.png)
上の図ではBroker 3は同期されていないのでISRではない、つまりout-of-syncとなります。

ちなみに、厳密にはISRか否かという判断はもう少し複雑で、ここで説明されているようにすんなり「このFollowerは最新の状態か」と判断出来る訳ではありません。ただ厳密な話をし始めるとこのエントリの主旨から外れるので、ここでは上の図にある赤いBrokerは同期が取れていないと、見たまま捉えてください。

## Acks
 Acksはクライアント (Producer) 側の設定で、「どこまでFollowを含めて書き込みの確認が取れてからクライアントに返答するか」を指定するものです。有効な値は```0```、```1```、そして```all```の3つです。

 ### acks=0
```0```が設定された場合、クライアントはBrokerまで到達したかの確認さえ行いません - メッセージがKafka Brokerに対して送られたタイミングでackを返します。
 ![acks=0](blogs/kafka-acks-explained/ack-0.png)
 ackと呼びますがBrokerからの返答さえ待ちません。送れたらOKです。

 ### acks=1
```1```が設定された場合、クライアント (Producer) はLeaderにまでメッセージが到達した時点で書き込みの完了と判断します。Leader Brokerはメッセージを受け取った時点でレスポンスを返します。
![acks=0](blogs/kafka-acks-explained/ack-1.png)
クライアントはレスポンスが返ってくるのを待ちます。Leaderからの返答が到着した時点で完了と判断しackとします。Leaderは受け取り次第レスポンスを返すので、Followerへのレプリケーションはレスポンスとは非同期に処理されます。

### acks=all
```all```と設定された場合、クライアントは全てのISRにメッセージが到達した時点で書き込みの完了と判断します。この際Leader BrokerがKafka側の書き込み判定を行なっており、全てのISRへのメッセージ到達の上クライアントにレスポンスを返します。
![acks=all incomplete](blogs/kafka-acks-explained/ack-all.png)
上の図の状態ではBroker 3はまだメッセージを受け取っていません。この為Leaderはレスポンスを返しません。
![acks=all completed](blogs/kafka-acks-explained/acs-all-completed.png)
全てのISRに渡って初めてレスポンスが返されます。

### acksの機能性
この通りacksはパフォーマンスとデータ欠損耐性のバランスを決める非常に有益な設定です。データ保全を優先するのであれば```acks=all```の設定が適切です。[^1] 一方レイテンシやスループットに関する要件が極めて高い場合には```0```に設定すれば最も効率が良くなりますが、同時にメッセージロスの可能性は高まります。

## Minimum In-Sync Replicas
```acks=all```の設定に関して、もう一つ重要な要素があります。

例えばLeaderが全てのISRへの書き込み完了した上でレスポンスを返すとして、LeaderのみがISRだった場合、結果として```acks=1```と振る舞いは同じとなるのでしょうか？

ここで```min.insync.replicas```の設定が重要になります。

```min.insync.replicas```というBroker側の設定は、```acks=all```の際に「最低いくつのISRとなっているか (Leaderを含めて幾つのレプリカが最新状態か) を指定するものです。つまりLeaderは、```acks=all```のリクエストに対して指定されたISRに満たないまでは返答せず、またそれが何かしらの理由で達成できない場合にはエラーを返します。データ保全観点でのゲートキーバーの様な役割を果たします。 

![acks=all and ISR=2](blogs/kafka-acks-explained/acks-all-isr-2.png)
上記の状態だとBroker 3は同期されていない状態です。しかしながら```min.insync.replicas=2```となっている場合には条件を満たす為この時点でレスポンスが返されます。

![acks=all and ISR below min.insync.replicas](blogs/kafka-acks-explained/ack-all-error.png)
Broker 2と3が同期されていない状態です。この場合指定された```min.insync.replicas```を下回るためLeaderからはエラーレスポンスが返る、つまり書き込みは失敗します。一方同じ状況であっても```acks```の設定が```0```もしくは```1```の場合には正常なレスポンスが返されます。

### 注意点
一般的に```min.insync.replicas```は「Leaderがクライアントに返答する際に、どれだけレプリケーションが完了しているかを指定する」と解釈されていますが、これは誤りです。正確には「リクエストを処理する為に最低いくつのレプリカが存在するか」を指定する設定です。
![acks=all incomplete](blogs/kafka-acks-explained/ack-all.png)
上記の場合、Broker 1から3までが全て同期状態です。この時に新たなリクエスト (ここではメッセージ```6```) を受け取った場合、Broker 2への同期が完了してもレスポンスは返しません。この場合、処理時にISRとなっているBroker 3への同期が完了して初めてレスポンスが返されます。

## まとめ
図で説明したことによって理解が深まったのではないかと思います。

おさらいすると、```acks```と```min.insync.replicas```はKafkaへの書き込みにおける欠損体制を指定する事ができます。
- ```acks=0``` - 書き込みはクライアントがLeaderにメッセージを送った時点で成功とみなします。Leaderからのレスポンスを待つことはしません。
- ```acks=1``` - 書き込みはLeaderへの書き込みが完了した時点で成功とみなします。
- ```acks=all``` - 書き込みはISR全てへの書き込みが完了した時点で成功とみなします。ISRが```min.insync.replicas```を下回る場合には処理されません。

### その他情報
Kafkaは複雑な分散システムであり、学ばなければいけない事が多いのも事実です。Kafkaの他の重要な要素については以下も参考にしてください。
- [Kafka consumer data-access semantics](https://www.confluent.io/blog/apache-kafka-data-access-semantics-consumers-and-membership/) - クライアント (Consumer) における欠損耐性、可用性、データ整合性の確保に関わる詳細。
- [Kafka controller](https://medium.com/@stanislavkozlovski/apache-kafkas-distributed-system-firefighter-the-controller-broker-1afca1eae302) - Broker間の連携がどの様になされるのかの詳細。特にレプリカが非同期 (Out-of-Sync) となるのはどういう条件下かについて説明しています。
- [“99th Percentile Latency at Scale with Apache Kafka](https://www.confluent.io/blog/configure-kafka-to-minimize-latency/) - Kafkaのパフォーマンスに関するConfluent Blogエントリ。
- [Kafka Summit SF 2019 videos](https://www.confluent.io/resources/kafka-summit-san-francisco-2019/)
- [Confluent blog](https://www.confluent.io/blog/) - Kafkaに関する様々なトピックを網羅。
- [Kafka documentation](https://kafka.apache.org/documentation/)

Kafkaは継続的かつアクティブに開発されていますが、機能追加や改善は活発なコミュニティにより支えられています。開発の最前線で何が起こっているか興味がある場合はぜひ[メーリングリスト](https://kafka.apache.org/contact) に参加してください。

![2 Minutes Streaming](blogs/kafka-acks-explained/two-minites-streaming.png)
このエントリの著者である[Stanislav Kozlovski](../../authors/stanislav/) は[2 Minute Streaming](https://2minutestreaming.com/)というKafkaに関する隔週ニュースレターを発行しています。是非購読してみてください。


[^1]:(訳者注)ほとんどのユースケースでは```acks=all```が適切であり、Kafkaのデフォルトでもあります。
[^2]:(訳者注)Leader以外にも最低限1つISRが存在するので、突然Leaderがオフラインになっても最新データは確保できているという設定となります。忠しKafkaのデフォルトは```1```です。