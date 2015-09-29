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

You can also define the client at a later time (for more options see the
~:ref:`connections` chapter):

.. code:: python

    s = s.using(client)

.. note::

    All methods return a *copy* of the object, making it safe to pass to
    outside code.

The API is chainable, allowing you to combine multiple method calls in one
statement:

.. code:: python

    s = Search().using(client).query("match", title="python")

.. note::

    In some cases this approach is not possible due to python's restriction on
    identifiers - for example if your field is called ``@timestamp``. In that
    case you have to fall back to unpacking a dictionary: ``s.query('range', **
    {'@timestamp': {'lt': 'now'}})``

To send the request to Elasticsearch:

.. code:: python

    response = s.execute()

For debugging purposes you can serialize the ``Search`` object to a ``dict``
explicitly:

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


Filters
~~~~~~~

Filters behave similarly to queries - just use the ``F`` shortcut and
``.filter()`` method. When you use the ``.filter()`` method, the query will be
automatically wrapped in a ``filtered`` query.

If you want to use the post_filter element for faceted navigation, use the
``.post_filter()`` method.


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
    
or

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


Sorting
~~~~~~~

To specify sorting order, use the ``.sort()`` method:

.. code:: python

    s = Search().sort(
        'category',
        '-title',
        {"lines" : {"order" : "asc", "mode" : "avg"}}
    )

It accepts positional arguments which can be either strings or dictionaries.
String value is a field name, optionally prefixed by the ``-`` sign to specify
a descending order.

To reset the sorting, just call the method with no arguments:

.. code:: python

  s = s.sort()


Pagination
~~~~~~~~~~

To specify the from/size parameters, use the Python slicing API:

.. code:: python

  s = s[10:20]
  # {"from": 10, "size": 10}


Highlighting
~~~~~~~~~~~~

To set common attributes for highlighting use the ``highlight_options`` method:

.. code:: python

    s = s.highlight_options(order='score')

Enabling highlighting for individual fields is done using the ``highlight`` method:

.. code:: python

    s = s.highlight('title')
    # or, including parameters:
    s = s.highlight('title', fragment_size=50)

The fragments in the response will then be available on reach ``Result`` object
as ``.meta.highlight.FIELD`` which will contain the list of fragments:

.. code:: python

    response = s.execute()
    for hit in response:
        for fragment in hit.meta.highlight.title:
            print(fragment)

Suggestions
~~~~~~~~~~~

To specify a suggest request on your ``Search`` object use the ``suggest`` method:

.. code:: python

    s = s.suggest('my_suggestion', 'pyhton', term={'field': 'title'})

The first argument is the name of the suggestions (name under which it will be
returned), second is the actual text you wish the suggester to work on and the
keyword arguments will be added to the suggest's json as-is.

Extra properties and parameters
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To set extra properties of the search request, use the ``.extra()`` method:

.. code:: python

  s = s.extra(explain=True)
 
To set query parameters, use the ``.params()`` method:

.. code:: python

  s = s.params(search_type="count")


Serialization and Deserialization
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The search object can be serialized into a dictionary by using the
``.to_dict()`` method.

You can also create a ``Search`` object from a ``dict``:

.. code:: python

  s = Search.from_dict({"query": {"match": {"title": "python"}}})


Response
--------

You can execute your search by calling the ``.execute()`` method that will return
a ``Response`` object. The ``Response`` object allows you access to any key
from the response dictionary via attribute access. It also provides some
convenient helpers:

.. code:: python

  response = s.execute()

  print(response.success())
  # True
      
  print(response.took)
  # 12

  print(response.hits.total)

  print(response.suggest.my_suggestions)

If you want to inspect the contents of the ``response`` objects, just use its
``to_dict`` method to get access to the raw data for pretty printing.


Hits
~~~~

To access to the hits returned by the search, access the ``hits`` property or
just iterate over the ``Response`` object:

.. code:: python

    response = s.execute()
    print('Total %d hits found.' % response.hits.total)
    for h in response:
        print(h.title, h.body)


Result
~~~~~~

The individual hits is wrapped in a convenience class that allows attribute
access to the keys in the returned dictionary. All the metadata for the results
are accessible via ``meta`` (without the leading ``_``):

.. code:: python

    response = s.execute()
    h = response.hits[0]
    print('/%s/%s/%s returned with score %f' % (
        h.meta.index, h.meta.doc_type, h.meta.id, h.meta.score))

.. note::

    If your document has a field called ``meta`` you have to access it using
    the get item syntax: ``hit['meta']``.


Aggregations
~~~~~~~~~~~~

Aggregations are available through the ``aggregations`` property:

.. code:: python

    for tag in response.aggregations.per_tag.buckets:
        print(tag.key, tag.max_lines.value)
    

