---
title: ストリーム処理でKafka TopicのKeyを扱う in KSQL
summary: ksqlDBにおけるストリーム処理の概念と、ストリーム処理でTopicのKeyを扱う方法について。
authors:
  - hashi
tags:
  - Stream Processing
  - Topic
categories: 
  - Blog
  - ksqDB
projects: 
date: '2023-11-02T00:00:00Z'
lastmod: '2023-11-02T00:00:00Z'
---

## はじめに
Kafkaをベースとしたストリーム処理では、Kafka Topicに流れるイベントを取り込み、処理後に別のTopicに書き込む事を繋げる事によりパイプラインを構築します。ksqlDBの様なストリーム処理基盤はこの処理を開発者から隠蔽化し、SQLを用いてそのロジックのみに注力できるよう補助してくれます。

一方、この隠蔽化によってksqlDBが行なっている内部処理の多くは開発者にはタッチする事が出来ず、出来た場合でも直感的に扱えない事も多くあります。Kafka TopicのKeyもその一つです。

今回はksqlDBにおけるStreamの概念と、Topic Keyを扱う方法について説明します。

## StreamとTable
ksqlDBのストリーム処理では、Topicと紐付けるデータモデルを定義しそのモデルに対してクエリを実行する形でデータにアクセスします。具体的には```Stream```と```Table```で：

- **STREAM** - Topicを流れるイベントを、時系列を維持したデータの流れとして体現。ステートレス。
- **TABLE** - Topicを流れるイベントからKey単位でデータの最新状態をマテリアライズ。ステートフル。

同じKafka上で扱うデータですが、モデルによってksqlDBにおける扱いも、そしてそれを支える内部の仕組みも異なります。[^1] ただksqlDBのクエリ上はどちらも```CREATE```構文を利用してSQL同様の手順で作成します。例としてStreamの定義のサンプルは以下の様になります。
```sql
CREATE STREAM Clickstream (
    IP VARCHAR,
    USERID INT,
    REMOTE_USER VARCHAR,
    TIME VARCHAR,
    _TIME INT,
    REQUEST VARCHAR,
    STATUS VARCHAR,
    BYTES VARCHAR,
    REFERRER VARCHAR,
    AGENT VARCHAR
) WITH (
KAFKA_TOPIC='datagen-topic', VALUE_FORMAT='JSON');
```
ここでは```datagen-topic```というJSONでシリアライズされたTopicに対して```Clickstream```という名前のStreamデータモデルを定義しています。

#### CSASとCTAS
Topicをモデル化したStreamやTableを定義したとして、今度はそのモデルに対して処理をする必要性が出てきます。

一般的なデータフロープログラミングのモデルではプログラム内で処理をチェイニングした結果を別Topicに出力しますが、SQLにはそのような構文モデルはありません。一方リレーショナルDBではストアドプロシージャ等を利用しますが、DB毎にその仕様は異なります。ksqlDBではSQLに近い構文でありながら、かつより汎用的な方法でデータの加工処理とストアを結び付ける必要があります。

ksqlDBではもっと汎用的な```SELECT```と```CREATE STREAM```を組み合わせる事により処理とストアを結びつけます。具体的には```CREATE STREAM AS SELECT```構文を利用します。
```sql
CREATE STREAM EventsWithoutKey
WITH (KAFKA_TOPIC='404events', VALUE_FORMAT='JSON')
AS SELECT
    IP,
    USERID,
    _TIME TIME_IN_INT,
    STATUS,
    BYTES
FROM Clickstream
WHERE STATUS = '404'
EMIT CHANGES;
```
ここでは前述した```Clickstream```というStreamから必要なカラムを指定してSELECTした結果を新たなStreamとして定義する処理です。ここではWHERE句を利用してフィルタリングした結果のみ抽出し、かつ新たに```404events```というTopicに対して出力しています。物理的には```datagen-topic``` にあるデータが変換/加工され```404events```という新たなTopicに登録されます。

当然この構文はksqlDBでのストリーム処理には頻出構文であり、CSAS (```CREATE STREAM AS SELECT```)、CTAS (```CREATE TABLE AS SELECT```)と呼ばれます。

#### Keyとデータモデル
```CREATE STREAM```と```CREATE TABLE```ではTopicのKeyの扱い方が異なります。

以前のバージョンではStreamでもTableでも```ROWKEY```という名称でTopicのKeyに紐づくフィールドをモデルに追加します。この```ROWKEY```は物理的にはTopicのKeyでありながら、Valueを扱うksqlDBから参照出来るという、他のフィールドとは扱いも振る舞いも異なります。また、コミュニティではこの勝手に追加される```ROWKEY```というフィールドの扱い方に関して多くの混乱を招きました。ストリーム処理をSQLで処理をする上では直感的では無いという判断でした。

このKeyの扱い方は[ksqlDB 0.10で大きく変わりました](https://www.confluent.io/blog/ksqldb-0-10-updates-key-columns/#keyless-streams)。これは一律ではなくStreamとTableで定義を変える事で：
- **CREATE TABLE** - 明示的にKeyを指定する必要があり、それには```PRIMARY KEY```と指定するフィールドが必要。
- **CREATE STREAM** - キーをそもそも指定しない。

このアプローチはTABLEの構文としても自然であり、かつSTREAMを扱う際にはKeyが存在すらしないという潔いものです。結果としてこの変更と思想が以降のksqlDBのコミュニティに広がりました。

## StreamとKey
それでもStreamでKeyを扱いたいというのが本エントリの主旨です。

実際にStreamでKeyを指定する必要性がある場合は存在し、具体的には[JOINの際にはKey指定したものしかJOIN出来ません](https://docs.ksqldb.io/en/latest/developer-guide/joins/partition-data/#keys)。この為StreamでもKeyを指定する事は可能になっています。

具体的にはフィールドに```Key```と指定する事により、そのフィールドをValueの一部ではなくKeyとして扱います。先程の```CREATE STREAM```を例に取ると：
```sql
CREATE STREAM ClickstreamWithKey (
    IP VARCHAR Key,
    USERID INT,
    REMOTE_USER VARCHAR,
    TIME VARCHAR,
    _TIME INT,
    REQUEST VARCHAR,
    STATUS VARCHAR,
    BYTES VARCHAR,
    REFERRER VARCHAR,
    AGENT VARCHAR
) WITH (
KAFKA_TOPIC='datagen-topic', VALUE_FORMAT='JSON');
```
同じTopicを参照していますが、ここで生成されるStreamにはレコードKeyに```IP```が指定され、と言うよりValueからKeyに移動します。

この振る舞いを実際に確認するには```CSAS```で再定義したものに新たなTopicを割り当て、その結果を比較する必要があります。これら2つのStreamに以下のCSASを適用すると：
```sql
CREATE STREAM TransformedEvents
WITH (KAFKA_TOPIC='events', VALUE_FORMAT='JSON')
AS SELECT
    IP,
    USERID,
    _TIME TIME_IN_INT,
    STATUS,
    BYTES
FROM Clickstream -- Keyがある方はClickstreamWithKeyと指定
EMIT CHANGES;
```
通常の```CREATE STREAM```で生成した場合、Topicには
![CREATE STREAM without Key - Key](blogs/handling-keys-in-ksql/keyless-key.png)
![CREATE STREAM without Key - Value](blogs/handling-keys-in-ksql/keyless-value.png)
となり、```Key```を指定したStreamに対する```CSAS```の結果は
![CREATE STREAM with Key - Key](blogs/handling-keys-in-ksql/withkey-key.png)
![CREATE STREAM with Key - Value](blogs/handling-keys-in-ksql/withkey-value.png)
となります。IPがTopicのKeyに移っている事が確認できます。

## KeyをValueの中にも持つ
先述した通り、StreamにおいてKeyを扱う事は混乱を招く恐れがある為、JOIN等明確な利用がある場合のみに利用する事が推奨されます。それでもJoinもするがValueの中でもKeyを参照する、つまりksqlDB内でKeyを参照したいというユースケースも存在します。

この場合、KeyのコピーをValue内に持たせるする必要がありますが、ハックに近い対応が必要となります。

KeyをValueにコピーする関数は存在し、[AS_VALUE](https://docs.ksqldb.io/en/latest/developer-guide/ksqldb-reference/scalar-functions/#as_value)関数を利用すればコピー出来ます。

しかしながら、```AS_VALUE```はTableを前提とした関数であり、Tableには明示的に PRIMARY KEYを指定するのでKeyとそのフィールド名も参照出来ますが、Streamの場合にはKeyは必須ではありません。また、先程の```Key```を利用した```CREATE STREAM```の結果にあるように、Keyには値のみ、ここでは```IP```で指定した値のみが入っています。なので```AS_VALUE```を```ROWKEY```に対して実行したいのですが：
```sql
CREATE STREAM TransformedEventsWithKey
WITH (KAFKA_TOPIC='events-with-key', VALUE_FORMAT='JSON')
AS SELECT
    AS_VALUE(ROWKEY) AS IP,
    USERID,
    _TIME TIME_IN_INT,
    STATUS,
    BYTES
FROM ClickstreamWithKey
EMIT CHANGES;
```
このクエリは構文エラーとなります。```AS_VALUE```はTableの```PRIMARY KEY```は引数として受け付けますが、値だけのROWKEYは受け付けません。つまり```AS_VALUE```は通常の使用法ではStreamに対しては利用出来ません。[^2]

ハックとしての解答は以下になります：
```sql
CREATE STREAM TransformedEventsWithKey
WITH (KAFKA_TOPIC='events-with-key', VALUE_FORMAT='JSON')
AS SELECT
    IP AS ROWKEY,
    AS_VALUE(IP) AS IP,
    USERID,
    _TIME TIME_IN_INT,
    STATUS,
    BYTES
FROM ClickstreamWithKey
EMIT CHANGES;
```
```ROWKEY```に対して```IP```というフィールド名を割り当て、そのフィールドを```AS_VALUE```で利用する方法になります。上記クエリを分解解釈すると以下となります：
- StreamにはないPrimary Keyのフィールドを```IP```として明示的に指定。
- そのフィールドを```AS_VALUE```で参照。この際元々あるフィールドと同名で定義。

妙な構文になりますが、結果は：
![CREATE STREAM with Key and Value - Key](blogs/handling-keys-in-ksql/withkeyvalue-key.png)
![CREATE STREAM with Key and Value - Value](blogs/handling-keys-in-ksql/withkeyvalue-value.png)
と期待通りの結果となります。

## おわりに
このハック的なアプローチを見ると「ksqlDBは面倒くさい。直感的ではない。」と思うかも知れません。確かにハックだけを見るとその通りで、無意味な制約のようにも思えます。しかしながらこれには背景があり、より直感的なデータモデル定義へと変更した事による副作用である事を理解して頂ければと思います。また、ksqlDB、というよりデータフロー処理内でKeyを参照するというのは特殊な要件です。このハックを怪しい要件/ロジックのスメルと捉える事もできます。

何より、やや面倒くさいksqlDBにおけるKeyの扱いを理解すると、ksqlDBの仕組みやデータモデルへの理解が深まります。ksqlDBの裏側を少し垣間見る機会と思って頂ければ幸いです。

[^1]:実際のステート管理はksqlDBが内部で利用する[Kafka Streamsの仕組み](https://docs.confluent.io/platform/current/streams/architecture.html#state)を利用している。
[^2]:これはStreamの構文だからエラーではなく、そもそも```AS_VALUE```に```ROWKEY```を指定出来ないという仕様によるもの。
