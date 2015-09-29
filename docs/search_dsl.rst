Search DSL
==========

``Search`` オブジェクト
---------------------

``Search`` オブジェクトは検索のリクエスト全般を表現します:

  * クエリ

  * フィルタ

  * 集約

  * ソート

  * ページネーション

  * 追加のパラメータ

  * 関連付けられたクライアント


このAPIはチェーンができるように設計されています。
``Search`` オブジェクトがイミュータブルであり、オブジェクトへのすべての変更は、その変更を含むコピーの中で反映されます。
つまり、元のオブジェクトの修正はなしに ``Search`` オブジェクトを他のコードに安全に渡すことができます。

``Search`` オブジェクトをインスタンス化するときには
低水準の `elasticsearch client <http://elasticsearch-py.readthedocs.org/>`_ インスタンスを渡す必要があります:

.. code:: python

    from elasticsearch import Elasticsearch
    from elasticsearch_dsl import Search

    client = Elasticsearch()

    s = Search(client)

クライアントを後から定義することも可能です(:ref:`connections` の章でさらにオプションを知ることができます) :

.. code:: python

    s = s.using(client)

.. note::

    すべてのメソッドはオブジェクトのコピーを返却するため、外部のコードへ安全に渡すことができます。

このAPIはチェーンが可能で、1つのステートメント内に複数のメソッド呼び出しを記述可能です:

.. code:: python

    s = Search().using(client).query("match", title="python")

.. note::

    識別子において、Pythonの制約によりこのアプローチが難しいケースもあります。
    例えば、呼び出したいフィールドが ``@timestamp`` の場合は
    辞書として括り出すことで代替します: ``s.query('range', **{'@timestamp': {'lt': 'now'}})``

リクエストをElasticsearchに送ります:

.. code:: python

    response = s.execute()

デバッグをしたいときは ``Search`` オブジェクトを ``dict`` にシリアライズすることができます:

.. code:: python

    print(s.to_dict())

Queries
~~~~~~~



The library provides classes for all Elasticsearch query types. Pass all the
parameters as keyword arguments:

.. code:: python

    from elasticsearch_dsl.query import MultiMatch

    # {"multi_match": {"query": "python django", "fields": ["title", "body"]}
    MultiMatch(query='python django', fields=['title', 'body'])

You can use the ``Q`` shortcut to construct the instance using a name with
parameters or the raw ``dict``:

.. code:: python

    Q("multi_match", query='python django', fields=['title', 'body'])
    Q({"multi_match": {"query": "python django", "fields": ["title", "body"]})

To add the query to the ``Search`` object, use the ``.query()`` method:

.. code:: python

    q = Q("multi_match", query='python django', fields=['title', 'body'])
    s = s.query(q)

The method also accepts all the parameters as the ``Q`` shortcut:

.. code:: python

    s = s.query("multi_match", query='python django', fields=['title', 'body'])

If you already have a query object, or a ``dict`` representing one, you can
just override the query used in the ``Search`` object:

.. code:: python

    s.query = Q('bool', must=[Q('match', title='python'), Q('match', body='best')])

Query combination
^^^^^^^^^^^^^^^^^

Query objects can be combined using logical operators:

.. code:: python

    Q("match", title='python') | Q("match", title='django')
    # {"bool": {"should": [...]}}

    Q("match", title='python') & Q("match", title='django')
    # {"bool": {"must": [...]}}

    ~Q("match", "title"="python")
    # {"bool": {"must_not": [...]}}

You can also use the ``+`` operator:

.. code:: python

    Q("match", title='python') + Q("match", title='django')
    # {"bool": {"must": [...]}}

When using the ``+`` operator with ``Bool`` queries, it will merge them into a
single ``Bool`` query:

.. code:: python

    Q("bool") + Q("bool")
    # {"bool": {"..."}} 

When you call the ``.query()`` method multiple times, the ``+`` operator will
be used internally:

.. code:: python

    s = s.query().query()
    print(s.to_dict())
    # {"query": {"bool": {...}}}

If you want to have precise control over the query form, use the ``Q`` shortcut
to directly construct the combined query:

.. code:: python

    q = Q('bool',
        must=[Q('match', title='python')],
        should=[Q(...), Q(...)],
        minimum_should_match=1
    )
    s = Search().query(q)


フィルタ
~~~~~~~

フィルタはクエリと似たように振る舞います。 - ``.filter()`` メソッドのショートカットとして ``F`` を使用します。
``.filter()`` メソッドを使用するときには、クエリが自動的に ``filtered`` クエリの中にラップされます。

ファセットナビゲーションの実装にpost_filterを使用したいときは、 ``.post_filter`` メソッドを使用します。


集約
~~~~~~~~~~~~

集約を定義するために、 ``A`` というショートカットを使うことができます:

.. code:: python

    A('terms', field='tags')
    # {"terms": {"field": "tags"}}

集約をネストしたいときは ``.bucket()`` メソッドと ``.metric()`` メソッドを利用します:

.. code:: python

    a = A('terms', field='category')
    # {'terms': {'field': 'category'}}

    a.metric('clicks_per_category', 'sum', field='clicks')\
        .bucket('tags_per_category', 'terms', field='tags')
    # {
    #   'terms': {'field': 'category'},
    #   'aggs': {
    #     'clicks_per_category': {'sum': {'field': 'clicks'}},
    #     'tags_per_category': {'terms': {'field': 'tags'}}
    #   }
    # }

集約を ``Search`` オブジェクトに追加するときは、 ``.aggs`` プロパティを使います。
これは集約のリクエストにおいてもっとも上位に位置します。

.. code:: python

    s = Search()
    a = A('terms', field='category')
    s.aggs.bucket('category_terms', a)
    # {
    #   'aggs': {
    #     'category_terms': {
    #       'terms': {
    #         'field': 'category'
    #       }
    #     }
    #   }
    # }

あるいは

.. code:: python

    s = Search()
    s.aggs.bucket('per_category', 'terms', field='category')\
        .metric('clicks_per_category', 'sum', field='clicks')\
        .bucket('tags_per_category', 'terms', field='tags')

    s.to_dict()
    # {
    #   'aggs': {
    #     'per_category': {
    #       'terms': {'field': 'category'},
    #       'aggs': {
    #         'clicks_per_category': {'sum': {'field': 'clicks'}},
    #         'tags_per_category': {'terms': {'field': 'tags'}}
    #       }
    #     }
    #   }
    # }


既存のbucketには名前を使ってアクセスすることができます:

.. code:: python

    s = Search()

    s.aggs.bucket('per_category', 'terms', field='category')
    s.aggs['per_category'].metric('clicks_per_category', 'sum', field='clicks')
    s.aggs['per_category'].bucket('tags_per_category', 'terms', field='tags')

.. note::

    複数の集約をチェーンするときには、 ``.bucket()`` メソッドと ``.metric()`` メソッドで返り値が異なります。
    ``.bucket()`` は新しく定義されたbucketを返し、 ``.metric()`` はさらなるチェーンを可能にするために親となるbucketを返します。

``Search`` オブジェクトにおいて、オブジェクトのコピーを返す他のメソッドとは異なり、
集約の定義はオブジェクトそのものに実行されます。


ソート
~~~~~~~

ソートの順序を指定するためには、 ``.sort()`` メソッドを使用します:

.. code:: python

    s = Search().sort(
        'category',
        '-title',
        {"lines" : {"order" : "asc", "mode" : "avg"}}
    )

このメソッドは位置指定引数で文字列か辞書を受け取ります。
文字列の値はフィールド名で、オプションとして ``-`` を指定すれば降順になります。

ソートをリセットするためには、引数なしでこのメソッドを呼び出します:

.. code:: python

  s = s.sort()


ページネーション
~~~~~~~~~~

開始位置や件数を指定するためには、PythonのスライスAPIを使用します:

.. code:: python

  s = s[10:20]
  # {"from": 10, "size": 10}


ハイライト
~~~~~~~~~~~~

ハイライトのための共通の属性を設定するためには、 ``highlight_options`` メソッドを使用します:

.. code:: python

    s = s.highlight_options(order='score')

``highlight`` メソッドを使用することで、それぞれのフィールドのハイライトが可能になります:

.. code:: python

    s = s.highlight('title')
    # or, including parameters:
    s = s.highlight('title', fragment_size=50)

レスポンス内でハイライトされる要素を利用する場合は
``.meta.highlight.FIELD`` を使って ``Result`` オブジェクトにアクセスします。
これはハイライトされる要素のリストを含んでいます:

.. code:: python

    response = s.execute()
    for hit in response:
        for fragment in hit.meta.highlight.title:
            print(fragment)

サジェスト
~~~~~~~~~~~

``Search`` オブジェクトにおいてサジェストのリクエストを指定するために、 ``suggest`` メソッドを使用します:

.. code:: python

    s = s.suggest('my_suggestion', 'pyhton', term={'field': 'title'})

最初の引数は返却される際のサジェストの名前で、2つ目の引数はサジェストが必要な文字列を示します。
キーワード引数はサジェスト用のjsonにそのままで追加されます。

その他のプロパティとパラメータ
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

検索リクエストにおいて他のプロパティを設定するために、 ``.extra()`` メソッドを使用します:

.. code:: python

  s = s.extra(explain=True)

クエリパラメータを設定するためには、 ``.params()`` メソッドを使用します:

.. code:: python

  s = s.params(search_type="count")


シリアライズとデシリアライズ
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

検索用のオブジェクトを ``.to_dict()`` メソッドを使用して辞書型にシリアライズすることができます。

``dict`` から ``Search`` オブジェクトを生成することも可能です:

.. code:: python

  s = Search.from_dict({"query": {"match": {"title": "python"}}})


レスポンス
--------

``.execute()`` メソッドをコールすることで検索を行います。
このメソッドは ``Response`` オブジェクトを返します。
``Response`` オブジェクトは、アトリビュートへのアクセスとレスポンスを格納した辞書オブジェクトのキーを対応づけます。
これにより、レスポンスへのアクセスを手助けします:

.. code:: python

  response = s.execute()

  print(response.success())
  # True

  print(response.took)
  # 12

  print(response.hits.total)

  print(response.suggest.my_suggestions)

``response`` オブジェクトの内容を確認したい場合は、
``to_dict`` メソッドを利用して、表示用に整形された生のデータにアクセスします。


件数
~~~~

検索によって返された件数を知るためには、``hits`` プロパティにアクセスするか、
単純に ``Response`` オブジェクトをイテレートします:

.. code:: python

    response = s.execute()
    print('Total %d hits found.' % response.hits.total)
    for h in response:
        print(h.title, h.body)


検索結果
~~~~~~

それぞれの検索結果は使いやすいかたちでクラスとしてラップされています。
クラスのアトリビュートから、返却された辞書型オブジェクトのキーにアクセス可能です。
検索結果に関する全てのメタデータには ``meta`` を通してアクセスすることが可能です(先頭に ``_`` は不要) :

.. code:: python

    response = s.execute()
    h = response.hits[0]
    print('/%s/%s/%s returned with score %f' % (
        h.meta.index, h.meta.doc_type, h.meta.id, h.meta.score))

.. note::

    If your document has a field called ``meta`` you have to access it using
    the get item syntax: ``hit['meta']``.


集約
~~~~~~~~~~~~

集約の結果には ``aggregations`` プロパティを通してアクセスします。

.. code:: python

    for tag in response.aggregations.per_tag.buckets:
        print(tag.key, tag.max_lines.value)
