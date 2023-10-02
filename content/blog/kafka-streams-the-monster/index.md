---
title: Kafka Streamsというモンスターな標準ライブラリについて
summary: Kafkaネイティブなストリーム処理 - Kafkaのスタンダードライブラリでありながらぶっ飛んだKafka Streamsの仕組みと機能について
authors:
  - hashi
tags:
  - Stream Processing
  - Exactly Once Semantics
categories: 
  - Blog
  - Kafka Strams
projects: 
date: '2023-10-20T00:00:00Z'
lastmod: '2023-10-20T00:00:00Z'
---

## はじめに
[Kafka Streams](https://docs.confluent.io/ja-jp/platform/7.1/streams/index.html)はApacheソフトウェア財団が運営する[Apache Kafkaプロジェクト](https://kafka.apache.org/)に含まれています。Kafka Connectも同様で、これらは全て同じバージョニングで管理されています。Apache KafkaにおいてKafka Connectは外部ストレージやサービスとの統合、Kafka Streamsはストリーム処理、それぞれ異なるサブプロジェクトの様な存在です。

Kafka Connectは仕様であり各種Connectorの共通クラスの定義がメインである為、コードの大部分はインターフェースと抽象クラスで構成されています。もちろんKafka Streamsも同様の共通クラスやインターフェースを含みますが、それ以外にも大量の実装を含み、中にはそれなりに脂っこいコードも多く含まれます。

Kafka StreamsはJarファイルとして利用するので普通のJavaアプリケーションと同様の方法でパッケージ/デプロイ出来る一方、通常大規模分散ミドルウェアに使われる[RocksDB](https://rocksdb.org/)を内部で利用しています。ライブラリとしてはかなり異質な存在ではないかと思います。

Kafka Streamsをうっかり「Kafka標準のアクセスライブラリ」的な観点で触れると火傷することにもなりかねません。今エントリでは、あまりKafka Streamsに馴染みの無い方にもその思想と仕組みに触れてもらう事を目的にしています。ユースケースからではなく、今回は機械的な仕組みからご説明します。

## Kafka Streams デプロイモデル

