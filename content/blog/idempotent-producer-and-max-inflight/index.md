---
title: Exactly Onceとmax.in.flightについて
subtitle: max.in.flightが>1でもIdempotent Producerが機能する件について
summary: Exactly Once Deliveryに関わる重要な変更がKIP-95にて対応されて暫く経ちます。仕組み上Producer側の並列送信数は1に設定する必要がありましたが、後日のエンハンスメントで最大5まで対応出来る事になっています。
authors:
  - hashi
tags:
  - Exactly Once Semantics
categories: 
  - Blog
  - Kafka Core
  - KIP
projects: []
date: '2023-07-03T00:00:00Z'
lastmod: '2023-07-03T00:00:00Z'
draft: false
featured: false
image:
  caption: ''
  focal_point: ''
---
## はじめに
メッセージブローカー界隈でのデリバリー保証はAt Least Once (必ず送信するが1度以上送信する可能性がある) というのが常識であり、データを受け取るConsumer側で冪等性を保証する必要がありました。そのExactly Once SemantisがKafkaでサポートされた時には多くの反響を呼びましたが、この設定は最近DefaultでOnになる程Kafkaコミュニティでは広く利用されています。

ただこのエンハンスメントにも制限がありました。この制限は後日、ひっそりと一つのPRによって解消されています。話題には上りませんでしたが、この機能が広く利用される上では非常に重要なエンハンスメントでした。

## Exactly Once Semantics
Kafka初期において最も注目を集めたエンハンスメントの一つに[KIP-98 - Exactly Once Delivery and Transactional Messaging](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging) があります。「メッセージ基盤においてExactly Onceは不可能」という[ビザンチン将軍問題](https://ja.wikipedia.org/wiki/%E3%83%93%E3%82%B6%E3%83%B3%E3%83%81%E3%83%B3%E5%B0%86%E8%BB%8D%E5%95%8F%E9%A1%8C)観点からの懐疑的な意見も多く議論を呼びました。そもそもKafkaが唱えるExactly Onceのスコープは何か、そして何がその前提となっているのかについてはKafka初期開発者であるネハさんを始めとして[具体的な説明](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)もたくさんなされています。[^1]

実際のKIPに記載されている設定条件は以下で、これらも同様に適切に設定しない限りは```enable.idempotency=true```と設定してもProducerの冪等性を確保する保証はないと記載されています (仮にIdempotent Producerとして動いてPIDに値が設定されているとしても)。
```bash
ack=all
retries > 1
max.inflight.requests.per.connection=1
```
必ずISRへの同期が完了し、エラー時にはリトライする様にし、かつProducerからの並列送信は許容しない、という条件です。理には適っています。

## KAFKA-5494
[KAFKA-5494: enable idempotence with max.in.flight...](https://github.com/apache/kafka/pull/3743) このPRではKIP-98実装における課題の説明と、それに対する解決策が記載されています。具体的には2つの課題への対応が纏まったPRとなっており、結果としてmax.in.flight.requests.per.connectionが1である制限を最大5まで増やす対応となっています。[^2]

対応としてのポイントは、Brokerとの通信途絶時のProducer側 (Client) のシーケンス番号の採番ルールです。送信エラーとなった場合にはシーケンス番号を採番し直す事により処理を自動復旧すること、また再採番の前に送信処理中のバッチが全て処理済みである確認等が考慮されています。[^3]

## おわりに
KIPではなくPRとして実装されたこの変更ですが、シーケンス例外が出た際に事後復旧出来るようになる事、max.in.flightを1より大きく指定できる事、より広くIdempotent Producerを利用する上で重要な改善が含まれています。

[^1]: オリジナルのデザイン資料は[ここ](https://docs.google.com/document/d/11Jqy_GjUGtdXJK94XGsEIK7CP1SnQGdp2eF0wSw9ra8/)にあります。 
[^2]: 合わせて```OutOfSequenceException```が発生してしまうとクライアント側での後続処理は全て同じ例外が発生する課題についても対応されています。
[^3]: またこのPRに関する前提情報や設計については別途[こちら](https://docs.google.com/document/d/1EBt5rDfsvpK6mAPOOWjxa9vY0hJ0s9Jx9Wpwciy0aVo/edit)にまとめられています。





