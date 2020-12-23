---
title: "Elasticsearch SQL入門"
emoji: "🔎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Java, Elasticsearch, SQL]
published: true
---

:::message
本投稿は[Elastic Stack (Elasticsearch) Advent Calendar 2020](https://qiita.com/advent-calendar/2020/elasticsearch) 24 日目の記事です。
:::

# まえがき

みなさん、SQL は好きですか。という書き出しで記事を書くのは[2 度目](/articles/aa7d9c12616cd4219503)ですが、私は SQL のことを理解しきっているわけではないので好きになり途中です。

本投稿では、Elaticsearch に対して SQL でリクエスト可能なオフィシャルのプラグイン、[Elasticsearch SQL](machttps://www.elastic.co/jp/what-is/elasticsearch-sql)をご紹介いたします。

## 前回の記事との違い

[前回の記事](/articles/aa7d9c12616cd4219503)でも Elasticsearch に対して SQL を発行しリクエストする方法をご紹介しましたが、あちらは Open Distro for Elasticsearch のプラグインでした。両者の大きな違いは以下の表の通りです。

|              | Elasticsearch SQL                         | Open Distor for Elasticsearch SQL |
| -----------: | :---------------------------------------- | :-------------------------------- |
|   ライセンス | Elastic License (Basic Subscription) [^1] | Apache License                    |
| コネクション | JDBC, ODBC                                | JDBC, ODBC                        |
|          GUI | ×                                         | Query Workbench                   |
|          CLI | ○                                         | ○                                 |
| DSL への変換 | ○(translate API)                          | ○ (exlain API)                    |

[^1]: https://github.com/elastic/elasticsearch/blob/master/licenses/ELASTIC-LICENSE.txt 各課金体系と利用可能な機能は[こちら](https://www.elastic.co/jp/subscriptions)をご覧ください。

## Elasticsearch SQL の制限について

Elasticsearch SQL には様々な制限があり、[SQL Limitations](https://www.elastic.co/guide/en/elasticsearch/reference/current/sql-limitations.html)ページに一覧の記載があります。
2020/12 現在でクリティカルなものをピックアップすると以下の様なものが挙げられます。

- [Array がサポートされていない](https://www.elastic.co/guide/en/elasticsearch/reference/current/sql-limitations.html)
- 件数制約
  - [Group By 結果をベースとしたソート](https://www.elastic.co/guide/en/elasticsearch/reference/current/sql-limitations.html)

他にも、記載されていない部分では以下の様な制限が今回の検証を通して見つかりました。

- DISTINCT 句がサポートされていない
- JOIN 句がサポートされていない

# Elatic License 版コンテナの立ち上げ

検証のための環境構築には docker-compose を用います。主な環境は以下の通りです。

|                | バージョン |
| -------------- | ---------- |
| mac OS         | 10.15.7    |
| docker for mac | 2.4.0.0    |
| Elaticsearch   | 7.10.0     |
| Kibana         | 7.10.0     |

また、今回の検証では Kibana の初回起動時にデフォルトでセットアップ可能な**Sample eCommerce orders**データセットを用います。

## docker-compose.yml の用意

今回用意した検証用クラスタの構成は以下の通りです。特に複数台構成にする必要がないため、single node となっています。

```yaml:docker-compose.yml
version: '2.2'
services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.10.0
    container_name: es01
    environment:
      - node.name=es01
      - discovery.seed_hosts=es02
      - cluster.initial_master_nodes=es01
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms256m -Xmx256m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    ports:
      - 9200:9200
    networks:
      - esnet
  kibana:
    image: docker.elastic.co/kibana/kibana:7.10.0
    links:
     - es01:elasticsearch
    ports:
      - 5601:5601
    networks:
      - esnet
    environment:
      - xpack.monitoring.ui.container.elasticsearch.enabled=true

networks:
  esnet:
```

この際の注意点として、docker イメージには`*-oss`が付与されていないものを利用します。
Elaticsearch の公式で公開されている docker イメージには 2 種類あり、Apache ライセンス下で利用可能な elasticsearch-oss コンテナと Elastic ライセンス下の機能も利用可能な elasticsearch コンテナの 2 種類があります。
[elasticsearch | Docker](https://www.docker.elastic.co/r/elasticsearch)

今回利用したい機能は先述の通り Elastic ライセンス下の機能であるため、後者のものを明示的に指定し利用しています。

# 実践

## 全権取得クエリを投げてみる

まずはお試しで全件取得する SQL を投げてみます。
Kibana > Dev Tools からリクエスト実行画面を表示します。
![way to devtools](https://storage.googleapis.com/zenn-user-upload/cr67sak5f2hrr8zqvtre4avd6wqm)

▼ Dev Tools
![devtools](https://storage.googleapis.com/zenn-user-upload/ilgmif2qzofcpgq2lo4e8po0x228)

投げるクエリは一般的な SELECT \*です。

```sql:select_all.sql
SELECT * FROM "kibana_sample_data_ecommerce"
```

![入力画面](https://storage.googleapis.com/zenn-user-upload/c8igt459bcz9jbmy8u2i9ktlsu40)

### 結果

```json:取得結果
{
  "error" : {
    "root_cause" : [
      {
        "type" : "ql_illegal_argument_exception",
        "reason" : "Arrays (returned by [manufacturer]) are not supported"
      }
    ],
    "type" : "ql_illegal_argument_exception",
    "reason" : "Arrays (returned by [manufacturer]) are not supported"
  },
  "status" : 500
}
```

…エラーとなりました。Array は対応していないとのことです。実際にデータをみて、何が原因かをみてみるために、translate API を用いてで DSL に変換してみます。

```json:リクエスト
GET /_sql/translate
{
  "query": """
  SELECT * FROM "kibana_sample_data_ecommerce"
  """
}
```

### 結果

```json
{
  "size": 1000,
  "_source": {
    "includes": [
      "category",
      "customer_first_name",
      ... 中略
      "total_unique_products"
    ],
    "excludes": []
  },
  "docvalue_fields": [
    {
      "field": "currency"
    },
    ... 中略
    {
      "field": "user"
    }
  ],
  "sort": [
    {
      "_doc": {
        "order": "asc"
      }
    }
  ]
}
```

こちらを実際元に、実際のインデックスに対して\_search リクエストをかけてみます。

```json:_search結果.json
{
  "took" : 8,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 4675,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [
      {
        "_index" : "kibana_sample_data_ecommerce",
        "_type" : "_doc",
        "_id" : "uzf7j3YBp8s7-c5IvNTe",
        "_score" : null,
        "_source" : {
          "customer_full_name" : "Eddie Underwood",
          "customer_last_name" : "Underwood",
          "customer_first_name" : "Eddie",
          "day_of_week_i" : 0,
          "total_quantity" : 2,
          "taxless_total_price" : 36.98,
          "total_unique_products" : 2,
          "category" : [
            "Men's Clothing"
          ],
          "manufacturer" : [
            "Elitelligence",
            "Oceanavigations"
          ],
          "products" : [
            {
              "tax_amount" : 0,
              "taxful_price" : 11.99,
              "quantity" : 1,
              "taxless_price" : 11.99,
              "discount_amount" : 0,
              "base_unit_price" : 11.99,
              "discount_percentage" : 0,
              "product_name" : "Basic T-shirt - dark blue/white",
              "manufacturer" : "Elitelligence",
              "min_price" : 6.35,
              "unit_discount_amount" : 0,
              "price" : 11.99,
              "product_id" : 6283,
              "base_price" : 11.99,
              "_id" : "sold_product_584677_6283",
              "category" : "Men's Clothing"
            },
            {
              "tax_amount" : 0,
              "taxful_price" : 24.99,
              "quantity" : 1,
              "taxless_price" : 24.99,
              "discount_amount" : 0,
              "base_unit_price" : 24.99,
              "discount_percentage" : 0,
              "product_name" : "Sweatshirt - grey multicolor",
              "manufacturer" : "Oceanavigations",
              "min_price" : 11.75,
              "unit_discount_amount" : 0,
              "price" : 24.99,
              "product_id" : 19400,
              "base_price" : 24.99,
              "_id" : "sold_product_584677_19400",
              "category" : "Men's Clothing"
            }
          ],
          "taxful_total_price" : 36.98
        },
        "fields" : {
          "products.sku" : [
            "ZO0299602996",
            "ZO0549605496"
          ],
          "customer_phone" : [
            ""
          ],
          "geoip.city_name" : [
            "Cairo"
          ],
          "geoip.region_name" : [
            "Cairo Governorate"
          ],
          "type" : [
            "order"
          ],
          "order_date" : [
            "1609752528000"
          ],
          "geoip.location" : [
            "30.09999997448176, 31.29999996162951"
          ],
          "geoip.country_iso_code" : [
            "EG"
          ],
          "products.created_on" : [
            "1482744528000",
            "1482744528000"
          ],
          "currency" : [
            "EUR"
          ],
          "geoip.continent_name" : [
            "Africa"
          ],
          "customer_id" : [
            "38"
          ],
          "sku" : [
            "ZO0299602996",
            "ZO0549605496"
          ],
          "order_id" : [
            "584677"
          ],
          "user" : [
            "eddie"
          ],
          "customer_gender" : [
            "MALE"
          ],
          "email" : [
            "eddie@underwood-family.zzz"
          ],
          "event.dataset" : [
            "sample_ecommerce"
          ],
          "day_of_week" : [
            "Monday"
          ]
        },
        "sort" : [
          0
        ]
      }
    ]
  }
}
```

取得結果を見ると、`category`や`manufacture`等 Array 型のフィールドが含まれていることがわかります。
これは制約に記載のある通りの事項なので、以降は Array 型のフィールドを選択しないことで回避します。

## order id 降順で Top10 の顧客名を取得

```sql:query
SELECT customer_full_name FROM "kibana_sample_data_ecommerce" ORDER BY order_id DESC LIMIT 10
```

```json:DSL
{
  "size" : 10,
  "_source" : {
    "includes" : [
      "customer_full_name"
    ],
    "excludes" : [ ]
  },
  "sort" : [
    {
      "order_id" : {
        "order" : "desc",
        "missing" : "_first",
        "unmapped_type" : "keyword"
      }
    }
  ]
}
```

```json:result
{
  "columns" : [
    {
      "name" : "customer_full_name",
      "type" : "text"
    }
  ],
  "rows" : [
    [
      "Jim Pratt"
    ],
    [
      "Robbie Shaw"
    ],
    [
      "Irwin Walters"
    ],
    [
      "Ahmed Al Gomez"
    ],
    [
      "Ahmed Al Strickland"
    ],
    [
      "Jackson Byrd"
    ],
    [
      "Wagdi Shaw"
    ],
    [
      "Samir Holland"
    ],
    [
      "Elyssa Hubbard"
    ],
    [
      "Wilhemina St. Graham"
    ]
  ]
}
```

## customer_first_name を重複排除して取得

```sql:query
SELECT DISTINCT customer_first_name from "kibana_sample_data_ecommerce"
```

```json:result
{
  "error" : {
    "root_cause" : [
      {
        "type" : "verification_exception",
        "reason" : "Found 1 problem\nline 2:8: SELECT DISTINCT is not yet supported"
      }
    ],
    "type" : "verification_exception",
    "reason" : "Found 1 problem\nline 2:8: SELECT DISTINCT is not yet supported"
  },
  "status" : 400
}
```

Limitation のページに記載はありませんが、どうやら DISTINCT 句はサポートされていないようです。Aggregation で取得できそうな内容だけに少し残念ですね。

## taxless_total_price が 100 以上 200 以下の order id を 10 件取得

```sql:query
SELECT order_id
FROM kibana_sample_data_ecommerce
WHERE taxless_total_price BETWEEN 100 AND 200
LIMIT 10
```

```json:DSL
{
  "size" : 10,
  "query" : {
    "range" : {
      "taxless_total_price" : {
        "from" : 100,
        "to" : 200,
        "include_lower" : true,
        "include_upper" : true,
        "time_zone" : "Z",
        "boost" : 1.0
      }
    }
  },
  "_source" : false,
  "stored_fields" : "_none_",
  "docvalue_fields" : [
    {
      "field" : "order_id"
    }
  ],
  "sort" : [
    {
      "_doc" : {
        "order" : "asc"
      }
    }
  ]
}
```

```json:result
{
  "columns" : [
    {
      "name" : "order_id",
      "type" : "keyword"
    }
  ],
  "rows" : [
    [
      "584058"
    ],
    ... 中略
    [
      "578650"
    ]
  ]
}
```

## 顧客名ごとに注文件数を取得

```sql:query
SELECT COUNT(customer_full_name), customer_full_name FROM kibana_sample_data_ecommerce GROUP BY customer_full_name
```

```json:DSL
{
  "size" : 0,
  "_source" : false,
  "stored_fields" : "_none_",
  "aggregations" : {
    "groupby" : {
      "composite" : {
        "size" : 1000,
        "sources" : [
          {
            "f8f23918" : {
              "terms" : {
                "field" : "customer_full_name.keyword",
                "missing_bucket" : true,
                "order" : "asc"
              }
            }
          }
        ]
      },
      "aggregations" : {
        "8df1ff4b" : {
          "filter" : {
            "exists" : {
              "field" : "customer_full_name",
              "boost" : 1.0
            }
          }
        }
      }
    }
  }
}
```

```json:result
{
  "columns" : [
    {
      "name" : "COUNT(customer_full_name)",
      "type" : "long"
    },
    {
      "name" : "customer_full_name",
      "type" : "text"
    }
  ],
  "rows" : [
    [
      2,
      "Abd Adams"
    ],
    ... 中略
    [
      1,
      "Abd Bradley"
    ]
  ]
}
```

## JOIN

### データの準備

JOIN 先のテーブルを作成するため、以下の定義でインデックスを作成します。

```json
PUT /ecommerce_customer_info
{
  "mappings": {
    "properties": {
      "customer_id": {
        "type": "integer",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "customer_full_name": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      }
    }
  }
}
```

仮でいくつかデータを投入します。

```json
POST /_bulk
{"index": {"_index": "ecommerce_customer_info"}}
{"customer_id": 1, "customer_full_name": "Eddie Underwood"}
{"index": {"_index": "ecommerce_customer_info"}}
{"customer_id": 2, "customer_full_name": "Mary Bailey"}
{"index": {"_index": "ecommerce_customer_info"}}
{"customer_id": 3, "customer_full_name": "Gwen Butler"}
{"index": {"_index": "ecommerce_customer_info"}}
{"customer_id": 4, "customer_full_name": "Diane Chandler"}
```

### クエリの実行

```sql:query
SELECT e.customer_id
FROM kibana_sample_data_ecommerce k
INNER JOIN ecommerce_customer_info e
  ON k.customer_full_name = e.customer_full_name
```

```json:result
{
  "error" : {
    "root_cause" : [
      {
        "type" : "parsing_exception",
        "reason" : "line 4:2: Queries with JOIN are not yet supported"
      }
    ],
    "type" : "parsing_exception",
    "reason" : "line 4:2: Queries with JOIN are not yet supported"
  },
  "status" : 400
}
```

JOIN 句も DISTINCT 句と同じく、Limitations のページに記載はありませんがサポートされていないようです。
そもそも Elasticsearch のピュアな機能で JOIN ができないので、サポートしないことについては納得ができます。

# 全文検索

ここからは通常の SQL から外れますが、Elasticsearch らしく全文検索をしてみます。
[Full-Text Search Functions](https://www.elastic.co/guide/en/elasticsearch/reference/master/sql-functions-search.html)を見ると、2020/12 現在以下がサポートされている様でした。

- Match Query
- query_string Query
- Score

スコアリングもできるのはすごいですね。早速いくつか試してみます。

## 名前が smith にマッチする顧客の order id を取得

```sql:query
SELECT order_id FROM kibana_sample_data_ecommerce WHERE MATCH(customer_full_name, 'smith')
```

```json:DSL
{
  "size" : 1000,
  "query" : {
    "match" : {
      "customer_full_name" : {
        "query" : "smith",
        "operator" : "OR",
        "prefix_length" : 0,
        "max_expansions" : 50,
        "fuzzy_transpositions" : true,
        "lenient" : false,
        "zero_terms_query" : "NONE",
        "auto_generate_synonyms_phrase_query" : true,
        "boost" : 1.0
      }
    }
  },
  "_source" : false,
  "stored_fields" : "_none_",
  "docvalue_fields" : [
    {
      "field" : "order_id"
    }
  ],
  "sort" : [
    {
      "_doc" : {
        "order" : "asc"
      }
    }
  ]
}
```

```json:result
{
  "columns" : [
    {
      "name" : "order_id",
      "type" : "keyword"
    }
  ],
  "rows" : [
    [
      "557262"
    ],
    ... 中略
    [
      "573691"
    ]
  ]
}
```

## 名前が smith にマッチする顧客をマッチ度順にスコアリング

```sql:query
SELECT customer_full_name
FROM kibana_sample_data_ecommerce
WHERE MATCH(customer_full_name, 'smith')
ORDER BY SCORE()
```

```json:DSL
{
  "size" : 1000,
  "query" : {
    "match" : {
      "customer_full_name" : {
        "query" : "smith",
        "operator" : "OR",
        "prefix_length" : 0,
        "max_expansions" : 50,
        "fuzzy_transpositions" : true,
        "lenient" : false,
        "zero_terms_query" : "NONE",
        "auto_generate_synonyms_phrase_query" : true,
        "boost" : 1.0
      }
    }
  },
  "_source" : {
    "includes" : [
      "customer_full_name"
    ],
    "excludes" : [ ]
  },
  "sort" : [
    {
      "_score" : {
        "order" : "asc"
      }
    }
  ]
}
```

```json:result
{
  "columns" : [
    {
      "name" : "customer_full_name",
      "type" : "text"
    }
  ],
  "rows" : [
    [
      "Wilhemina St. Smith"
    ],
    ... 中略
    [
      "Tariq Smith"
    ]
  ]
}
```

名前に smith が含まれる、それっぽい結果を取得することができました。

# 最後に

本投稿では、Elasticsearch SQL を使ってどこまでの操作が可能かについて検証しました。

個人的には SCORE 関数がしっかり用意しているあたり、全文検索ソリューションらしさを感じて好きだった一方、まだまだサポートしていないクエリも多く、ログの解析やデータの抽出で使うにしてもすこし苦労する場面は多そうです。
また、translate API を用いることで、SQL はかけるけど Elasticsearch のクエリはかけない、というケースでも変換を通して検索可能なため、初学者の学びのサポートにもなりそうな印象を持ちました。

# 参考

[公式ドキュメント](https://www.elastic.co/guide/en/elasticsearch/reference/master/xpack-sql.html)
