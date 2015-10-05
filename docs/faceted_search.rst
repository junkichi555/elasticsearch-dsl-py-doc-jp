.. _faceted_search:

ファセット検索
==============

このライブラリはファセットナビゲーションの実装を手助けするための
抽象化された機能を備えています。

.. note::

    この機能は試験的なものであり、変更される可能性があります。
    フィードバックは歓迎します。

設定
-------------

サブクラスである ``FacetSearch`` を宣言するときには、
いくつかの設定オプションをクラスの属性として指定することができます:

``index``
  | 検索の対象とするインデックス名（文字列型）
  | デフォルト: ``'_all'``

``doc_types``
  | 使用する ``DocType`` のサブクラスか文字列（リスト型）
  | デフォルト: ``['_all']``

``fields``
  | ドキュメントのタイプにおいて検索の対象とするフィールド（リスト型）
  | このリストは ``MultiMatch`` クエリに渡されるもので、ブーストの値（ ``'title^5'`` ）を含むことが可能
  | デフォルト: ``['*']``

``facets``
  | 表示あるいは非表示にするファセットを示す辞書（辞書型）
  | キーは表示される際の名前を示し、値はbucket集約に使用される（ネストはしない）
  | 例: ``{'tags': Terms{field='tags'}}``

応用
~~~~~~~~

もし特別な振る舞いや修正が必要な場合は、対象の機能を持つ1個以上のメソッドをオーバーライドします。
主に2つのメソッドが対象となります:

``search(self)``
  ``Search`` オブジェクトを生成するときに使用されます。
  もし検索のためのオブジェクトをカスタマイズしたいときはこれをオーバーライドします。
  (例えば、ある項目だけを表示するためのグローバルなフィルタを加える など)

``query(self, search)``
  検索においてクエリを指定する部分で、デフォルトでは ``MultiField`` クエリを使用します。
  使用されるクエリのタイプを変更したいときはこれをオーバーライドします。


使い方
-----

カスタムサブクラスをインスタンス化するときは、
（すべてをマッチさせるための）空の検索を行うために何も指定しないか、
``query`` と ``filters`` を用いることができます。

``query``
  | 実行されるクエリのテキストに値を渡す際に使用します。もし ``None`` （デフォルト）が渡されたときは ``MatchAll`` クエリが使用されます。
  | 例: ``'python web'``

``filters``
  適用したいファセットフィルタに関するすべての情報を含む辞書です。キーとして(``.facets`` 属性) におけるファセット名を指定し、値には取りうる値を指定します。

レスポンス
~~~~~~~~

``FacetedSearch`` オブジェクトの ``.execute()`` をコールすることで返却されるレスポンスは
標準の ``Response`` クラスのサブクラスです。
bucketsのリストの辞書を含む``facets``というプロパティが追加されています。
これらはキー、ドキュメント数、値がフィルタリングされているかのフラグから成るタプルで表現されています。

例
-------

.. code:: python

    from datetime import date

    from elasticsearch_dsl import FacetedSearch
    from elasticsearch_dsl.aggs import Terms, DateHistogram

    class BlogSearch(FacetedSearch):
        doc_types = [Article, ]
        # 検索に使用されるフィールド
        fields = ['tags', 'title', 'body']

        facets = {
            # ファセットを定義するためにbucket集約を使用する
            'tags': Terms(field='tags'),
            'publishing_frequency': DateHistogram(field='published_from', interval='month')
        }

        def search(self):
            # 部分的なカスタマイズのためにオーバーライドする
            s = super().search()
            return s.filter('range', publish_from={'lte': 'now/h'})

    bs = BlogSearch('python web', {'publishing_frequency': date(2015, 6)})
    response = bs.execute()

    # 通常通り、検索件数と他の属性にアクセスする
    print(response.hits.total, 'hits total')
    for hit in response:
        print(hit.meta.score, hit.title)

    for (tag, count, selected) in response.facets.tags:
        print(tag, ' (SELECTED):' if selected else ':', count)

    for (month, count, selected) in response.facets.publishing_frequency:
        print(month.strftime('%B %Y'), ' (SELECTED):' if selected else ':', count)
