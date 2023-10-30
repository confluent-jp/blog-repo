---
title: Apache Flink 1.18 アップデート
summary: Apache Kafkaの最新バージョンが公開されました。Tiered Storageがアーリーアクセスとして登場した他、Zookeeperのアップデート、KRaft Metadata Transaction等様々な新機能が追加されました。
authors:
  - hashi
tags:
  - Performance
  - Watermark
  - Operator Fusion
categories: 
  - Announcement
  - Flink
projects: []
date: '2023-10-30T00:00:00Z'
lastmod: '2023-10-30T00:00:00Z'

links:
url_code: ''
url_pdf: ''
url_slides: ''
url_video: ''
---
Apache Kafkaの新バージョン1.18が公開されました。[Conflunet Blog](https://www.confluent.io/blog/announcing-apache-flink-1-18/)ではその具体的な改善点をエリア毎に詳しく説明しており、ConfluentだけでなくVerverica、Aiven、Alibaba CloudのFlinkコミッターも共著として参加し、結果としてFlinkの情報発信として非常なものとなっております。

本エントリでは、一部ではありますがそのうちの幾つかをご紹介します。

### [FLIP-293: Introduce Flink Jdbc Driver For Sql Gateway](https://cwiki.apache.org/confluence/display/FLINK/FLIP-293%3A+Introduce+Flink+Jdbc+Driver+For+Sql+Gateway)
FlinkクラスタへのRESTエンドポイントを提供する[Flink SQL Ga†eway](https://github.com/ververica/flink-sql-gateway/blob/master/README.md)へのアクセスに、新たに汎用的なJDBC経由で通信できる[Flink JDBC Driver](https://github.com/ververica/flink-jdbc-driver)が接続出来るようになりました。

これまでSQL Gatewayにはコンソールベースでのアクセスは可能でしたが、セッションを保持したアプリケーションからのアクセスは出来ませんでした。一方JDBC Driverの基本利用はFlink Jobの登録にあり、インタラクティブなクエリはサポートされていませんでした。本FLIPによりJDBC接続が可能な多くのデータベースに対してJDBC Driverから接続出来るようになります。

### [FLIP-311: Support Call Stored Procedure](https://cwiki.apache.org/confluence/display/FLINK/FLIP-311%3A+Support+Call+Stored+Procedure)
これまでFlinkから見たデータソースはSourceでありSinkであり、あくまでデータストアという扱いにおける接続に限られました。本FLIPによってFlinkからStored Procedureの一覧取得と実行が可能となります。

Stored Procedure実行におけるインターフェース変更に合わせ、[Catalog Interface](https://nightlies.apache.org/flink/flink-docs-master/api/java/org/apache/flink/table/catalog/Catalog.html)にもStored Procedure用のメソッドが追加されており一覧の取得も可能です。

### [FLIP-308: Support Time Travel](https://cwiki.apache.org/confluence/display/FLINK/FLIP-308%3A+Support+Time+Travel)
[SQL:2011 Standard](https://en.wikipedia.org/wiki/SQL:2011)のTime Travel Queryがサポートされます。どちらも標準であるようタイムスタンプでの指定となりますが、特定時点ならびに期間指定がサポートされます。

用途としてはデータレイクに長期格納しているデータに対してFlinkからソースアタッチする際に特定の過去時点でのデータも同様の方法で取得可能となります。IcebergやDelta Lake等、Time Travel Queryをサポートしているストレージに限られた機能となり、またConnectorが新しいインターフェースに沿って実装する必要があります。

### [FLIP-292: Enhance COMPILED PLAN to support operator-level state TTL configuration](https://cwiki.apache.org/confluence/display/FLINK/FLIP-292%3A+Enhance+COMPILED+PLAN+to+support+operator-level+state+TTL+configuration)
Table APIやSQLを利用してステートフルなストリームパイプラインを構築する際のステート管理に関わる改善です。JOINをしたり同じTableデータに異なる条件で集約したりする場合に、そのステートのベースとなるイベントの有効期間 (TTL: Time To Live) の制御によっては処理の対象となるイベントが変わります。

本FLIPでは、それぞれの対象ソースに対して個別のTTLを設定出来るようになります。これにより要件に即したステート管理を行うことができるようになります。

### [FLIP-296: Extend watermark-related features for SQL](https://cwiki.apache.org/confluence/display/FLINK/FLIP-296%3A+Extend+watermark-related+features+for+SQL)
ストリーム処理においてデータの整合性をいかに評価/制御することは極めて重要ですが、Flinkでは[Event TimeとWatermark](https://www.youtube.com/watch?v=sdhwpUAjqaI)を利用する事により明示的にそれぞれのデータ処理ウィンドウを決定しています。

Watermarkはその振る舞いを制御する重要な仕組みであり、[DataStream API](https://nightlies.apache.org/flink/flink-docs-release-1.18/docs/dev/datastream/overview/)であればその[関連性の定義を制御(Watermark Alignment)](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/event-time/generating_watermarks/#watermark-alignment)する事も出来ました。但しこの機能はよりローレベルなDataStream APIを利用する必要がありました。

本FLIPでは、Flink SQLによってその制御が出来るようになりました。具体的にはTable作成時やクエリにアノテーションを指定する事で：
```sql
CREATE TABLE user_actions (
  ...
  user_action_time TIMESTAMP(3),
  WATERMARK FOR user_action_time AS user_action_time - INTERVAL '5' SECOND
) WITH (
  'scan.watermark.emit.strategy'='on-event',
  ...
);
```
とWatermark生成インターバルを指定したり：
```sql
select
  ...
from source_table /*+ OPTIONS('scan.watermark.emit.strategy'='on-event') */
```
SELECT時にWatermarkの出力タイプを指定できます。

### バッチ処理速度改善
[FLIP-324: Introduce Runtime Filter for Flink Batch Jobs](https://cwiki.apache.org/confluence/display/FLINK/FLIP-324%3A+Introduce+Runtime+Filter+for+Flink+Batch+Jobs)

[FLIP-315 Support Operator Fusion Codegen for Flink SQL](https://cwiki.apache.org/confluence/display/FLINK/FLIP-315+Support+Operator+Fusion+Codegen+for+Flink+SQL)

全バージョン(Flink 1.17)ではバッチ処理におけるスループットが大きく改善しました。その改善は本リリースでも継続して行われており、さらにそのパフォーマンスが向上しました。今回のリリースにおける主要な改善は：
- **FLIP-324** [Runtime Filter](https://www.alibabacloud.com/blog/query-performance-optimization-runtime-filter_598126)のアプローチは集約処理の前段階で対象レコードを絞るアプローチで、これにより集約やJoinにかかるネットワーク通信や必要処理の大規模化を削減する事が可能です。クエリのプラン中に関連処理の中からローカルでの集約可能な処理を特定しRuntime Filterとして実行する事により達成します。
- **FLIP-315** 利用可能メモリの増加からCPUの処理能力にボトルネックが移る中、処理プロセスにおける無駄が全体スループットに大きな影響を与えています。幾つかの改善ポイントを表化した結果、ベクター化とコード生成方式のうちコード生成方式の[Operator Fusion](https://www.vldb.org/pvldb/vol4/p539-neumann.pdf)の実装を導入しました。

![TPC-DS ベンチマーク結果](blogs/apache-flink-1.8/tpc-ds-benchmark-on-10t.png)
結果として[TPC-DS](https://www.tpc.org/tpcds/)のベンチマーク結果がFlink 1.17と比べて13%、1.16とでは35%改善しました。

### おわりに
今回のご紹介はApache Flink1.18で導入された新機能や改善のごく一部ではありますが、ストリーム処理からバッチ、クラウドネイティブ化に向けた改善等、非常に多岐に渡る改善が含まれています。ksqlDBを知る身としてはFlinkの分散データ処理基盤としての重厚さを感じることにもなりました。是非[オリジナルのブログ](https://www.confluent.io/blog/announcing-apache-flink-1-18/)も併せて参照してください。
