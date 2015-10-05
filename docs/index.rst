Elasticsearch DSL
=================

Elasticsearch DSLは、Elasticsearchのクエリ記述や実行を手助けすることを目的とした高水準ライブラリです。
公式の低水準なクライアント（ ``elasticsearch-py`` ）の上位レイヤに位置づけられます。

Elasticsearch DSLは、より便利で慣用的なクエリの記述や操作の手段を提供します。
用語や構造はElasticsearchのJSON DSLに似ています。
クラスやクエリセットのような表現として、全体をPythonで扱うことができます。

Elasticsearch DSLは、ドキュメントをPythonのオブジェクトとして扱うためのラッパも提供します。
ここにはマッピングの定義、ドキュメントの検索や保存、ユーザ定義クラスにおけるドキュメントデータのラッピングを含みます。

他のElasticsearchのAPI（例えばクラスタのヘルスチェックなど）を利用する場合は、低水準クライアントを使用します。

検索の例
--------------

典型的な検索リクエストを、 ``dict`` として直接書いてみましょう:

.. code:: python

    from elasticsearch import Elasticsearch
    client = Elasticsearch()

    response = client.search(
        index="my-index",
        body={
          "query": {
            "filtered": {
              "query": {
                "bool": {
                  "must": [{"match": {"title": "python"}}],
                  "must_not": [{"match": {"description": "beta"}}]
                }
              },
              "filter": {"term": {"category": "search"}}
            }
          },
          "aggs" : {
            "per_tag": {
              "terms": {"field": "tags"},
              "aggs": {
                "max_lines": {"max": {"field": "lines"}}
              }
            }
          }
        }
    )

    for hit in response['hits']['hits']:
        print(hit['_score'], hit['_source']['title'])

    for tag in response['aggregations']['per_tag']['buckets']:
        print(tag['key'], tag['max_lines']['value'])



このアプローチには、とても冗長で、ネストの誤りなどの書き間違いが起こりやすく、
他のフィルタを加えるなどの修正もしづらく、書いていて全く楽しくないという問題があります。

それでは、Python DSLを使ってこの例を書きなおしてみましょう:

.. code:: python

    from elasticsearch import Elasticsearch
    from elasticsearch_dsl import Search, Q

    client = Elasticsearch()

    s = Search(using=client, index="my-index") \
        .filter("term", category="search") \
        .query("match", title="python")   \
        .query(~Q("match", description="beta"))

    s.aggs.bucket('per_tag', 'terms', field='tags') \
        .metric('max_lines', 'max', field='lines')

    response = s.execute()

    for hit in response:
        print(hit.meta.score, hit.title)

    for tag in response.aggregations.per_tag.buckets:
        print(tag.key, tag.max_lines.value)

見てわかる通り、このライブラリは以下の点に留意して書かれています:

  * クエリの名称（"match"など）を使って 適切な ``Query`` オブジェクトを生成する

  * 複合の ``bool`` クエリとしてクエリをまとめる

  * ``.filter()`` が使用されたときには ``filtered`` クエリを生成する

  * レスポンスデータへのアクセスを容易にする

  * どこにも波括弧や角括弧を使わない


さらなる例
-------------------

ブログシステムの記事を表現するための単純なPythonのクラスを書いてみましょう:

.. code:: python

    from datetime import datetime
    from elasticsearch_dsl import DocType, String, Date, Integer
    from elasticsearch_dsl.connections import connections

    # デフォルトのElasticsearchクライアントを定義する
    connections.create_connection(hosts=['localhost'])

    class Article(DocType):
        title = String(analyzer='snowball', fields={'raw': String(index='not_analyzed')})
        body = String(analyzer='snowball')
        tags = String(index='not_analyzed')
        published_from = Date()
        lines = Integer()

        class Meta:
            index = 'blog'

        def save(self, ** kwargs):
            self.lines = len(self.body.split())
            return super(Article, self).save(** kwargs)

        def is_published(self):
            return datetime.now() < self.published_from

    # Elasticsearchのマッピングを生成する
    Article.init()

    # 記事を作成して保存する
    article = Article(id=42, title='Hello world!', tags=['test'])
    article.body = ''' looong text '''
    article.published_from = datetime.now()
    article.save()

    article = Article.get(id=42)
    print(article.is_published())

    # クラスタのヘルスチェックについて表示する
    print(connections.get_connection().cluster.health())


このコードからは以下のようなことわかります:

  * デフォルトのコネクションを提供する

  * マッピングの設定とともにフィールドについて定義する

  * インデックス名を設定する

  * カスタムのメソッドを定義する

  * 永続的なライフサイクルのために、ビルトインの ``.save()`` メソッドをオーバーライドする

  * オブジェクトを検索し、Elasticsearchに保存する

  * 他のAPIを利用するために低水準のクライアントにアクセスする

`永続化 <https://github.com/nanakenashi/elasticsearch-dsl-py-doc-jp/blob/master/docs/persistence.rst>`_ の章でさらに詳細を知ることができます。


ビルド済みのファセット検索
------------------------

もし定義済みの ``DocType`` があるのであれば、簡単にファセット検索のクラスを生成することができます。
これにより、検索やフィルタリングを簡単にすることができます。

.. note::

    この機能は試験的なものであり、変更される可能性があります。

.. code:: python

    from elasticsearch_dsl import FacetedSearch
    from elasticsearch_dsl.aggs import Terms, DateHistogram

    class BlogSearch(FacetedSearch):
        doc_types = [Article, ]

        # 検索されるフィールド
        fields = ['tags', 'title', 'body']

        facets = {
            # facetsの定義においてbucketを使用します
            'tags': Terms(field='tags'),
            'publishing_frequency': DateHistogram(field='published_from', interval='month')
        }

    # 空の検索
    bs = BlogSearch()
    response = bs.execute()

    for hit in response:
        print(hit.meta.score, hit.title)

    for (tag, count, selected) in response.facets.tags:
        print(tag, ' (SELECTED):' if selected else ':', count)

    for (month, count, selected) in response.facets.publishing_frequency:
        print(month.strftime('%B %Y'), ' (SELECTED):' if selected else ':', count)

`faceted_search <https://github.com/nanakenashi/elasticsearch-dsl-py-doc-jp/blob/master/docs/faceted_search.rst>`_ の章でさらに詳細を知ることができます。

``elasticsearch-py`` からの移行
-----------------------------------

Python DSLの恩恵を受けるためにすべてのアプリケーションを修正する必要はありません。
既存の ``dict`` から ``Search`` オブジェクトを生成し、それをAPIで修正したり ``dict`` に戻して利用できます:

.. code:: python

    body = {...} # 複雑なクエリをここに代入する

    # Searchオブジェクトに変換する
    s = Search.from_dict(body)

    # filter, aggregation, queryなどを追加する
    s.filter("term", tags="python")

    # 既存のコードに合わせるため、dict型に戻す
    body = s.to_dict()


ライセンス
-------

Copyright 2013 Elasticsearch

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

コンテンツ
--------
| `設定 <https://github.com/nanakenashi/elasticsearch-dsl-py-doc-jp/blob/master/docs/configuration.rst>`_
| `検索のためのDSL <https://github.com/nanakenashi/elasticsearch-dsl-py-doc-jp/blob/master/docs/search_dsl.rst>`_
| `永続化 <https://github.com/nanakenashi/elasticsearch-dsl-py-doc-jp/blob/master/docs/persistence.rst>`_
| `ファセット検索 <https://github.com/nanakenashi/elasticsearch-dsl-py-doc-jp/blob/master/docs/faceted_search.rst>`_
