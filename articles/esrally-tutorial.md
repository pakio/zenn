---
title: "Elasticsearch公式のベンチマーキングツール : Rally"
emoji: "🚙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Elasticsearch]
published: false
---

:::message
本記事は [ZOZO Advent Calendar 2021](https://qiita.com/advent-calendar/2021/zozo) の6日目の記事です。
:::

# 概要
RallyはElastic社公式のベンチマーキングツールで、jsonで定義されたシナリオをベースにElasticsearchに対して負荷をかけるためのPython製のCLIツールです。
過去実際に大幅な新規フィールド追加時のパフォーマンス検証時に利用しとても便利だったため、今回はその大まかな使い方をご紹介します。

https://github.com/elastic/rally

# ユースケース
## ベンチマーキングツールとして
上述の通りRallyはベンチマーキングツールであるため、例えばバージョンアップ時の負荷検証であったり、新規プラグインを導入した際の負荷検証などに利用可能なツールです。一通り使ってみた感想ですが、検索リクエストではなく主にインデキシングのリクエストに対する負荷状況を計測を得意としているように思えます。メトリクスとしてはざっくり以下のようなものが取得可能です。
* リクエスト関連
  * スループット
  * レイテンシ
  * エラーレート
* Elasticsearchのクラスター関連
  * GC duration
  * シャードのmergeに利用した時間
  * インデックスサイズ

またツール内にElasticsearchを建てる機能も内包しており、別途自前でクラスタを建てる必要もなく、日本語検索で必要となるプラグイン周りのインストールや辞書のインストールが定義１つ追加するだけで可能となっています。
既存のクラスタに対してベンチマーキングを行いたい際にも、コマンドラインの引数に接続情報を記述することで試験が可能です。

## データ抽出ツールとして
Rallyはデータをもとにトラックを作成 -> インデキシングの検証という使い方をされるツールです。がこれを応用して、ある環境からデータを引き抜いて他の環境に移すツールとしての使い方も可能です。普段だと独自のツールなどでやりがちなこの操作ですが、Rallyを使うことでお手軽に実施できてしまうのでなかなかオススメです。

# Rallyを使ってみる
## 前提環境
今回の検証は以下の環境で行いました。
* macOS BigSur
* Python : 3.8.1
* Java : openjdk 11.0.9

RallyはPython製のツールであるため、3.8以上を動作要件としています。
Javaのバージョンに関しては起動したいElasticsearchのバージョンによって異なるため、詳しくは以下のリンク先をご覧ください。
https://www.elastic.co/jp/support/matrix#matrix_jvm

## インストール
pipでインストール可能です。前述の通りPython3.8以上を必要とします。

```bash
pip3 install esrally
```

## サクッと走らせてみる
公式から提供されている負荷試験シナリオ(track)は以下のコマンドで確認可能です。

```bash
% esrally list tracks

Available tracks:

Name           Description                                                              Documents    Compressed Size    Uncompressed Size    Default Challenge        All Challenges
-------------  -----------------------------------------------------------------------  -----------  -----------------  -------------------  -----------------------  ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
geonames       POIs from Geonames                                                       11,396,503   252.9 MB           3.3 GB               append-no-conflicts      append-no-conflicts,append-no-conflicts-index-only,append-fast-with-conflicts,significant-text
percolator     Percolator benchmark based on AOL queries                                2,000,000    121.1 kB           104.9 MB             append-no-conflicts      append-no-conflicts
http_logs      HTTP server log data                                                     247,249,096  1.2 GB             31.1 GB              append-no-conflicts      append-no-conflicts,runtime-fields,append-no-conflicts-index-only,append-sorted-no-conflicts,append-index-only-with-ingest-pipeline,update,append-no-conflicts-index-reindex-only
geoshape       Shapes from PlanetOSM                                                    60,523,283   13.4 GB            45.4 GB              append-no-conflicts      append-no-conflicts
metricbeat     Metricbeat data                                                          1,079,600    87.7 MB            1.2 GB               append-no-conflicts      append-no-conflicts
geopoint       Point coordinates from PlanetOSM                                         60,844,404   482.1 MB           2.3 GB               append-no-conflicts      append-no-conflicts,append-no-conflicts-index-only,append-fast-with-conflicts
nyc_taxis      Taxi rides in New York in 2015                                           165,346,692  4.5 GB             74.3 GB              append-no-conflicts      indexing-querying,append-no-conflicts,append-no-conflicts-index-only,append-sorted-no-conflicts-index-only,update,append-ml,date-histogram
geopointshape  Point coordinates from PlanetOSM indexed as geoshapes                    60,844,404   470.8 MB           2.6 GB               append-no-conflicts      append-no-conflicts,append-no-conflicts-index-only,append-fast-with-conflicts
so             Indexing benchmark using up to questions and answers from StackOverflow  36,062,278   8.9 GB             33.1 GB              append-no-conflicts      append-no-conflicts
eql            EQL benchmarks based on endgame index of SIEM demo cluster               60,782,211   4.5 GB             109.2 GB             default                  default
nested         StackOverflow Q&A stored as nested docs                                  11,203,029   663.3 MB           3.4 GB               nested-search-challenge  nested-search-challenge,index-only
noaa           Global daily weather measurements from NOAA                              33,659,481   949.4 MB           9.0 GB               append-no-conflicts      sql,append-no-conflicts,append-no-conflicts-index-only,top_metrics,aggs
pmc            Full text benchmark with academic papers from PMC                        574,199      5.5 GB             21.7 GB              append-no-conflicts      indexing-querying,append-no-conflicts,append-no-conflicts-index-only,append-sorted-no-conflicts,append-fast-with-conflict
```

今回は、一番軽量なpercolatorを用いで検証してみます。

```bash
% esrally race --distribution-version=7.15.0 --track=percolator

[INFO] Race id is [6d4f67de-b291-4950-8318-072a638027c8]
[INFO] Preparing for race ...
[INFO] Downloading track data (121.1 kB total size)                               [100.0%]
[INFO] Decompressing track data from [/Users/pakio/.rally/benchmarks/data/percolator/queries-2.json.bz2] to [/Users/pakio/.rally/benchmarks/data/percolator/queries-2.json] (resulting size: [0.10] GB) ... [OK]
[INFO] Preparing file offset table for [/Users/pakio/.rally/benchmarks/data/percolator/queries-2.json] ... [OK]
[INFO] Racing on track [percolator], challenge [append-no-conflicts] and car ['defaults'] with version [7.15.0].

Running delete-index                                                           [100% done]
Running create-index                                                           [100% done]
Running check-cluster-health                                                   [100% done]
Running index                                                                  [100% done]
Running refresh-after-index                                                    [100% done]
Running force-merge                                                            [100% done]
Running refresh-after-force-merge                                              [100% done]
Running wait-until-merges-finish                                               [100% done]
Running percolator_with_content_president_bush                                 [100% done]
Running percolator_with_content_saddam_hussein                                 [100% done]
Running percolator_with_content_hurricane_katrina                              [100% done]
Running percolator_with_content_google                                         [100% done]
Running percolator_no_score_with_content_google                                [100% done]
Running percolator_with_highlighting                                           [100% done]
Running percolator_with_content_ignore_me                                      [ 46% done]
Running percolator_with_content_ignore_me                                      [100% done]
Running percolator_no_score_with_content_ignore_me                             [100% done]

------------------------------------------------------
    _______             __   _____
   / ____(_)___  ____ _/ /  / ___/_________  ________
  / /_  / / __ \/ __ `/ /   \__ \/ ___/ __ \/ ___/ _ \
 / __/ / / / / / /_/ / /   ___/ / /__/ /_/ / /  /  __/
/_/   /_/_/ /_/\__,_/_/   /____/\___/\____/_/   \___/
------------------------------------------------------

|                                                         Metric |                                       Task |       Value |   Unit |
|---------------------------------------------------------------:|-------------------------------------------:|------------:|-------:|
|                     Cumulative indexing time of primary shards |                                            |     2.94787 |    min |
|             Min cumulative indexing time across primary shards |                                            |   0.0317167 |    min |
|          Median cumulative indexing time across primary shards |                                            |    0.507367 |    min |
|             Max cumulative indexing time across primary shards |                                            |     0.80695 |    min |
|            Cumulative indexing throttle time of primary shards |                                            |   0.0583667 |    min |
|    Min cumulative indexing throttle time across primary shards |                                            |           0 |    min |
| Median cumulative indexing throttle time across primary shards |                                            |           0 |    min |
|    Max cumulative indexing throttle time across primary shards |                                            |   0.0303167 |    min |
|                        Cumulative merge time of primary shards |                                            |   0.0116333 |    min |
|                       Cumulative merge count of primary shards |                                            |           1 |        |
|                Min cumulative merge time across primary shards |                                            |           0 |    min |
|             Median cumulative merge time across primary shards |                                            |           0 |    min |
|                Max cumulative merge time across primary shards |                                            |   0.0116333 |    min |
|               Cumulative merge throttle time of primary shards |                                            |           0 |    min |
|       Min cumulative merge throttle time across primary shards |                                            |           0 |    min |
|    Median cumulative merge throttle time across primary shards |                                            |           0 |    min |
|       Max cumulative merge throttle time across primary shards |                                            |           0 |    min |
|                      Cumulative refresh time of primary shards |                                            |     1.15005 |    min |
|                     Cumulative refresh count of primary shards |                                            |          59 |        |
|              Min cumulative refresh time across primary shards |                                            |   0.0188167 |    min |
|           Median cumulative refresh time across primary shards |                                            |     0.22465 |    min |
|              Max cumulative refresh time across primary shards |                                            |    0.275417 |    min |
|                        Cumulative flush time of primary shards |                                            |      0.0307 |    min |
|                       Cumulative flush count of primary shards |                                            |           9 |        |
|                Min cumulative flush time across primary shards |                                            |           0 |    min |
|             Median cumulative flush time across primary shards |                                            |           0 |    min |
|                Max cumulative flush time across primary shards |                                            |      0.0307 |    min |
|                                        Total Young Gen GC time |                                            |      15.778 |      s |
|                                       Total Young Gen GC count |                                            |        8995 |        |
|                                          Total Old Gen GC time |                                            |       0.319 |      s |
|                                         Total Old Gen GC count |                                            |           3 |        |
|                                                     Store size |                                            |   0.0781473 |     GB |
|                                                  Translog size |                                            | 3.07336e-07 |     GB |
|                                         Heap used for segments |                                            |   0.0661278 |     MB |
|                                       Heap used for doc values |                                            |   0.0077858 |     MB |
|                                            Heap used for terms |                                            |    0.036499 |     MB |
|                                            Heap used for norms |                                            |           0 |     MB |
|                                           Heap used for points |                                            |           0 |     MB |
|                                    Heap used for stored fields |                                            |    0.021843 |     MB |
|                                                  Segment count |                                            |          45 |        |
~~~ 中略 ~~~
|                                                 Min Throughput | percolator_no_score_with_content_ignore_me |       15.05 |  ops/s |
|                                                Mean Throughput | percolator_no_score_with_content_ignore_me |       15.08 |  ops/s |
|                                              Median Throughput | percolator_no_score_with_content_ignore_me |       15.07 |  ops/s |
|                                                 Max Throughput | percolator_no_score_with_content_ignore_me |       15.11 |  ops/s |
|                                        50th percentile latency | percolator_no_score_with_content_ignore_me |     11.0354 |     ms |
|                                        90th percentile latency | percolator_no_score_with_content_ignore_me |     12.6811 |     ms |
|                                        99th percentile latency | percolator_no_score_with_content_ignore_me |     13.2097 |     ms |
|                                       100th percentile latency | percolator_no_score_with_content_ignore_me |     15.3898 |     ms |
|                                   50th percentile service time | percolator_no_score_with_content_ignore_me |     6.67196 |     ms |
|                                   90th percentile service time | percolator_no_score_with_content_ignore_me |     7.16561 |     ms |
|                                   99th percentile service time | percolator_no_score_with_content_ignore_me |     7.63037 |     ms |
|                                  100th percentile service time | percolator_no_score_with_content_ignore_me |     10.8921 |     ms |
|                                                     error rate | percolator_no_score_with_content_ignore_me |           0 |      % |
```

このように、コマンド一つでトラックを利用したベンチマークが可能なことが確認できました。

## 既存のインデックスからカスタムトラックを作成する
上記手順ではRally側で公式に提供しているトラックを用いた検証しましたが、実際のユースケースでは独自のデータを使いたいケースかと想定されます。
そのようなことも想定されており、Rallyでは既存のクラスタに接続し、任意のインデックスをもとにトラックを作成する機能を提供しています。

以下のコマンドで実行可能です。

```bash
# https, 認証ありクラスタの場合
esrally create-track --track={トラック名} --target-hosts={https抜きのクラスターのアドレス:ポート} --client-options="use_ssl:true,verify_certs:true,basic_auth_user:'{ユーザ名}',basic_auth_password:'{パスワード}',http_compression:true" --indices="{生成元インデックス名}" --output-path={出力先}

# http, 認証なしクラスタの場合
esrally create-track --track={トラック名} --target-hosts={http抜きのクラスターのアドレス:ポート} --client-options="http_compression:true" --indices="{生成元インデックス名}" --output-path={出力先}

    ____        ____
   / __ \____ _/ / /_  __
  / /_/ / __ `/ / / / / /
 / _, _/ /_/ / / / /_/ /
/_/ |_|\__,_/_/_/\__, /
                /____/
 
[INFO] Connected to Elasticsearch cluster [XXXX] version [7.15.0].
 
Extracting documents for index [example...   1000/1000 docs [100.0% done]
Extracting documents for index [example]...  nnnn/nnnn docs [100.0% done]
 
[INFO] Track example has been created. Run it with: esrally --track-path=/path/to/tracks/example
 
---------------------------------
[INFO] SUCCESS (took 266 seconds)
---------------------------------
```

ここで生成されたファイルを確認してみます。

```bash
% cd tracks/example
% tree .
.
├── track.json
├── example-documents-1k.json
├── example-documents-1k.json.bz2
├── example-documents.json
├── example-documents.json.bz2
└── example.json
```

トラックの定義とともにインデックスから取得されたデータ、そこから1000件のみ抽出されたデータ、インデックス定義がそれぞれ作成されたことを確認できました。

## カスタムトラックを実行する
上記手順を用いて作成したトラックをもとに、ベンチマークを行う方法です。

```bash
esrally race --track-path=./tracks/example --distribution-version=7.15.0
```

このとき、起動されているElasticsearchにはプラグインなどは何も導入されていない点にご注意ください。

## プラグイン込のクラスタを起動する
Rallyではプラグインをインストールした状態のクラスタでのベンチマークもサポートしています。
コマンド1つでインストール可能なものは基本的にElasticsearch公式で提供しているものとなり、一覧は以下のコマンドで確認可能です。

```bash
% esrally list elasticsearch-plugins

Available Elasticsearch plugins:

Name                     Configuration
-----------------------  ---------------
analysis-icu
analysis-kuromoji
analysis-phonetic
analysis-smartcn
analysis-stempel
analysis-ukrainian
discovery-azure-classic
discovery-ec2
discovery-file
discovery-gce
ingest-attachment
lang-javascript
lang-python
mapper-attachments
mapper-murmur3
mapper-size
repository-azure
repository-gcs
repository-hdfs
repository-s3
store-smb
transport-nio
transport-nio            transport
transport-nio            http
```

ここに記載のあるプラグイン込で実行したい場合、コマンドライン引数に`-elasticsearch-plugins="{プラグイン名}"`を追加します。

```bash
esrally race --distribution-version=7.15.0 --track=percolator --elasticsearch-plugins="analysis-icu,analysis-kuromoji"
```

この他にも、任意のプラグインをインストールした状態で起動することも可能なので、詳しくは公式のドキュメントを御覧ください。
https://esrally.readthedocs.io/en/stable/elasticsearch_plugins.html

## 辞書を追加済みのクラスタを起動する
日本語検索の場合、シノニムやカスタム辞書などを設定することが一般的です。Rallyでもカスタム辞書を内包したクラスタを起動することは可能ですが、少々手順が複雑であるため以下に手順を残します。

### 任意の試験環境(Car)を定義する
Rallyではベンチマークを実行するそれぞれの環境をCarと呼んでいます。Carは`~/.rally/benchmarks/teams/default/cars/v1`に定義されており、`XXX.ini`単体、もしくは同名のディレクトリとセットで取り扱われます。

```bash
% vi with-dictionary.ini
```

```text:with-dictionary.ini
[meta]
description=Includes custom dictionary
type=mixin

[config]
base=with-dictionary,vanilla
```

```bash
% mkdir -p with-dictionary/templates/config
```

これで、Carの定義と辞書を配置するディレクトリが完成しました。カスタム辞書やシノニムのファイルをこのフォルダに配置してください。

この時点で、Carの一覧を出力し、一覧に新しく定義したCarが含まれているかを確認します。表示されていれば成功です。
```bash
% esrally list cars

Available cars:

Name                     Type    Description
-----------------------  ------  --------------------------------------
16gheap                  car     Sets the Java heap to 16GB
1gheap                   car     Sets the Java heap to 1GB
24gheap                  car     Sets the Java heap to 24GB
2gheap                   car     Sets the Java heap to 2GB
4gheap                   car     Sets the Java heap to 4GB
8gheap                   car     Sets the Java heap to 8GB
defaults                 car     Sets the Java heap to 1GB
basic-license            mixin   Basic License
debug-non-safepoints     mixin   More accurate CPU profiles
ea                       mixin   Enables Java assertions
fp                       mixin   Preserves frame pointers
g1gc                     mixin   Enables the G1 garbage collector
parallelgc               mixin   Enables the Parallel garbage collector
trial-license            mixin   Trial License
unpooled                 mixin   Enables Netty's unpooled allocator
with-dictionary          mixin   Includes custom dictionary
x-pack-ml                mixin   X-Pack Machine Learning
x-pack-monitoring-http   mixin   X-Pack Monitoring (HTTP exporter)
x-pack-monitoring-local  mixin   X-Pack Monitoring (local exporter)
x-pack-security          mixin   X-Pack Security
```

### 実行するCarを指定する
上記のような手順で作成したCarやデフォルト以外のCarを利用する場合には、race実行時のコマンドライン引数に`--car="{Car名}"`を付与します。

```bash
% esrally race --distribution-version=7.15.0 --track=percolator --car="with-dictionary"
```

# まとめ
Elastic謹製のベンチマーキングツール、Rallyについてご紹介しました。
使ってみると便利な一方で、公式ドキュメント以外のまともな利用例が見つからず正直とっつきにくいのが第一印象でしたが、今回の検証でかなりイメージを変えることができました。
Javaさえインストールされていれば、トラック定義やCar定義はリポジトリから取得することも可能なので、例えばGitHub Actionsで定期的にベンチマークを走らせるなど夢が膨らみます。