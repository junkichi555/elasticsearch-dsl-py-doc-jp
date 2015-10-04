.. _persistence:

永続化
===========

DSLライブラリを使って、アプリケーションにおけるマッピングと基本的な永続化レイヤを定義することができます。

マッピング
--------

マッピングの定義はクエリのDSLと同様のパターンに従います:

.. code:: python

    from elasticsearch_dsl import Mapping, String, Nested

    # タイプ追加
    m = Mapping('my-type')

    # フィールドの追加
    m.field('title', 'string')

    # 複数のフィールドを簡単に扱えます
    m.field('category', 'string', fields={'raw': String(index='not_analyzed')})

    # 手動でフィールドを生成することもできます
    comment = Nested()
    comment.field('author', String())
    comment.field('created_at', Date())

    # それをマッピングに追加します
    m.field('comments', comment)

    # メタフィールドにマッピングを定義することもできます
    m.meta('_all', enabled=False)

    # マッピングを'my-index'インデックスに保存します
    m.save('my-index')

.. note::

    デフォルトでは、(``Nested`` を除いて) すべてのフィールドは単一の値を要求します。
    フィールドを作成あるいは定義する際に、コンストラクタに対して ``multi=True`` を渡すことで、
    複数の値を受け取るようにオーバーライドすることは可能です。
    (``m.field('tags', String(index='not_analyzed', multi=True))``)
    フィールドが設定されていない場合は、フィールドの値は空のリストになります。
    これにより、 ``doc.tags.append('search')`` と記述することを可能にしています。

特に、動的なマッピングを利用する場合は、
Elasticsearchにおける既存のタイプに基いてマッピングを更新するか、
既存のタイプから直接マッピングを生成することができれば便利かもしれません:

.. code:: python

    # prodクラスタからマッピングを取得する
    m = Mapping.from_es('my-index', 'my-type', using='prod')

    # QAクラスタ内のデータに基いて更新する
    m.update_from_es('my-index', using='qa')

    # prodクラスタのマッピングを更新する
    m.save('my-index', using='prod')

共通のフィールドオプション:

``multi``
  ``True`` を設定すると、初回のアクセス時にフィールドの値に ``[]`` が設定されます。

``required``
  ドキュメントにおいてフィールドが値を必要とするかどうかを示します。

解析
--------

``String`` フィールドに ``analyzer`` の値を指定するために、アナライザの名前を文字列で指定します。
このとき、ビルトインのアナライザのような既に定義されたアナライザを使うか、手動でアナライザを定義することもあります。

自身で定義したアナライザを生成し、生成のための永続化レイヤを持つことが可能です:

.. code:: python

    from elasticsearch_dsl import analyzer, tokenizer

    my_analyzer = analyzer('my_analyzer',
        tokenizer=tokenizer('trigram', 'nGram', min_gram=3, max_gram=3),
        filter=['lowercase']
    )

それぞれの解析オブジェクトは名前(例では ``my_analyzer`` と ``trigram``) を持ち、
トークナイザ、トークンフィルタ、文字列フィルタはタイプ(例では ``nGram``) の指定を必要とします。

.. note::

    カスタムアナライザに依存するマッピングを生成する場合は、インデックスが存在しないか停止している状態にしなければなりません。
    マッピングが定義された複数の ``DocType`` を生成したいときは、:ref:`index` オブジェクトを使用します。

タイプ
-------

ドキュメントを扱うためにモデルのようなラッパを生成したい場合は ``DocType`` クラスを使用します:

.. code:: python

    from datetime import datetime
    from elasticsearch_dsl import DocType, String, Date, Nested, Boolean, analyzer

    html_strip = analyzer('html_strip',
        tokenizer="standard",
        filter=["standard", "lowercase", "stop", "snowball"],
        char_filter=["html_strip"]
    )

    class Post(DocType):
        title = String()
        created_at = Date()
        published = Boolean()
        category = String(
            analyzer=html_strip,
            fields={'raw': String(index='not_analyzed')}
        )

        comments = Nested(
            properties={
                'author': String(fields={'raw': String(index='not_analyzed')}),
                'content': String(analyzer='snowball'),
                'created_at': Date()
            }
        )

        class Meta:
            index = 'blog'

        def add_comment(self, author, content):
            self.comments.append(
              {'author': author, 'content': content})

        def save(self, ** kwargs):
            self.created_at = datetime.now()
            return super().save(** kwargs)


ドキュメントのライフサイクル
~~~~~~~~~~~~~~~~~~~

新しい ``Post`` ドキュメントを生成するために、クラスをインスタンス化し、フィールドに指定したい値を渡します。
標準のアトリビュート設定を使って、フィールドを追加したり変更することができます。
明示的に定義されたフィールドに限定されないことに注意してください:

.. code:: python

    # ドキュメントのインスタンス化
    first = Post(title='My First Blog Post, yay!', published=True)
    # フィールドの値の指定(値か値のリスト)
    first.category = ['everything', 'nothing']
    # すべてのドキュメントはmetaの中にidを持ちます
    first.meta.id = 47


    # クラスタにドキュメントを保存します
    first.save()


すべてのメタデータフィールド( ``id`` 、 ``parent`` 、 ``routing`` 、 ``index`` など) には、
metaアトリビュートを使うか、アンダースコアをつけた変数名で直接アクセスすることができます:

.. code:: python

    post = Post(meta={'id': 42})

    # 42を表示(post.idでも同様)
    print(post.meta.id)

    # デフォルトのインデックスをオーバーライド
    post._index = 'my-blog'

.. note::

    すべてのメタデータにアクセス可能な ``meta`` を持っているということは、この名前は既に予約されており、
    ドキュメントにおいて ``meta`` というフィールドを持つべきではないことを意味しています。
    しかし、もし必要なのであればアイテムを取得するシンタックスとして ``post['meta']`` を使用してデータにアクセスできます。

既存のドキュメントを検索する場合は、 ``get`` クラスメソッドを使用します:

.. code:: python

    # ドキュメントを検索する
    first = Post.get(id=42)
    # フィールドの変更などが可能
    first.add_comment('me', 'This is nice!')
    # クラスタにもう一度変更を保存
    first.save()

    # update APIをコールすることで、特定のフィールドを修正し、その場でドキュメントを更新できます
    first.update(published=True, published_by='me')

Elasticsearch内にドキュメントが見つからない場合は例外(``elasticsearch.NotFoundError``) が発生します。
エラーの代わりに ``None`` を返して欲しい場合は、引数として ``ignore=404`` を渡します:

.. code:: python

    p = Post.get(id='not-in-es', ignore=404)
    p is None

``Mapping`` を含む ``DocType`` に関するすべての情報には、 ``_doc_type`` アトリビュートを通してアクセスできます:

.. code:: python

    # Elasticsearchにおけるタイプ名とインデックス名
    Post._doc_type.name
    Post._doc_type.index
    
    # 生のマッピングオブジェクト
    Post._doc_type.mapping

    # 親にあたるタイプの名称(定義されている場合)
    Post._doc_type.parent

``_doc_type`` アトリビュートは、Elasticsearchの ``DocType`` においてマッピングを更新するための ``refresh`` メソッドを持ちます。
これは、動的マッピングを使用していて、クラスにフィールドを認識させたいときに便利です(たとえば、 ``Date`` フィールドを適切にシリアライズしたいとき など) :

.. code:: python

    Post._doc_type.refresh()

検索
~~~~~~

対象のドキュメントタイプの検索をしたい場合は、 ``search`` クラスメソッドを使用します:

.. code:: python

    # .search をコールすることで、標準の検索オブジェクトを取得
    s = Post.search()
    # 検索時には既にインデックスとタイプが制限されています
    s = s.filter('term', published=True).query('match', title='first')


    results = s.execute()

    # 検索を実行したとき、検索結果はドキュメントのクラス(Post) 内でラップされています
    for posts in results:
        print(post.meta.score, post.title)

あるいは、 ``Search`` オブジェクトを定義してから、特定のドキュメントタイプを返却するように制限することもできます:

.. code:: python

    s = Search()
    s = s.doc_type(Post)

ドキュメントのクラスを標準のタイプ(文字列で表現されたもの)と複合することもできます。
複数の ``DocType`` サブクラスを渡すと、レスポンスにおけるそれぞれのドキュメントはそれぞれのクラスでラップされます。

ドキュメントを削除したい場合は、 ``delete`` メソッドを使用します:

.. code:: python

    first = Post.get(id=42)
    first.delete()

``class Meta`` オプション
~~~~~~~~~~~~~~~~~~~~~~

ドキュメント定義内の ``Meta`` クラスにおいて、様々なメタデータを定義することができます:

``doc_type``
  Elasticsearchにおけるタイプの名前
  デフォルト: クラス名から生成(MyDocument -> my_document)

``index``
  ドキュメントにおけるデフォルトのインデックス
  デフォルト: 空で、 ``get`` や ``save`` などのすべての操作で ``index`` パラメータが必須

``using``
  デフォルトで使用されるコネクションのエイリアス
  デフォルト: ``'default'``

``mapping``
  オプションとして使用される ``Mapping`` クラスのインスタンス
  ドキュメントのクラス上でフィールドから生成されるマッピングの基礎になる

``Meta`` クラスにおけるアトリビュート(``MetaField`` のインスタンス) はメタフィールド(``_all`` 、 ``_parent`` など) のマッピングを操作するために使用されます。
変数名として対象のフィールド名のアンダースコアを除く名前を使用し、 ``MetaField`` クラスに必要なパラメータを渡します:

.. code:: python

    class Post(DocType):
        title = String()

        class Meta:
            all = MetaField(enabled=False)
            parent = MetaField(type='blog')
            dynamic = MetaField('strict')

.. _index:

インデックス
-----

``Index`` はマッピングや様々な設定など、Elasticsearchのインデックスに関連するすべてのメタデータを保持するクラスです。
複数のマッピングを同時に生成可能なため、マッピングを定義する際に有用です。
特に、移行においてElasticsearchのオブジェクトをセットアップするときに便利です。

.. code:: python

    from elasticsearch_dsl import Index, DocType, String

    blogs = Index('blogs')

    # カスタムの設定を定義
    blogs.settings(
        number_of_shards=1,
        number_of_replicas=0
    )

    # エイリアスを定義
    blogs.aliases(
        old_blogs={}
    )

    # インデックスに対してタイプを登録
    blogs.doc_type(Post)

    # タイプを定義する際にはデコレータを利用することも可能
    @blogs.doc_type
    class Post(DocType):
        title = String()

    # インデックスを削除し、存在しない場合は無視
    blogs.delete(ignore=404)

    # Elasticsearchにインデックスを生成
    blogs.create()

テンプレートとなるインデックスを構成し、 ``clone`` メソッドを使用してコピーすることも可能です:

.. code:: python

    blogs = Index('blogs', using='production')
    blogs.settings(number_of_shards=2)
    blogs.doc_type(Post)

    # インデックスのコピーを異なる名前で生成
    company_blogs = blogs.clone('company-blogs')

    # 別のクラスタに異なるコピーを生成
    dev_blogs = blogs.clone('blogs', using='dev')
    # 設定を変更
    dev_blogs.setting(number_of_shards=1)
