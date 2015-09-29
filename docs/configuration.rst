設定
=============

このライブラリでは、コネクションを設定するいくつかの方法があります。
最も簡単で便利な方法は、APIコールのたびに使用されるデフォルトのコネクションを1つ定義することです。
明示的に他のコネクションを渡してやる必要はありません。

``elasticsearch_dsl`` を使用する際は、添付のシリアライザ(``elasticsearch_dsl.serializer.serializer``) の使用を推奨します。
これで毎回オブジェクトがjsonに適切にシリアライズされることが確認できます。
ここで紹介する ``create_connection`` メソッドは、明示的にシリアライザを指定しなくても、自動でこのシリアライズを使用します。
(そして ``create_connection`` メソッドは ``configure`` メソッドで内部的に使用されています)

.. note::

    アプリケーションから複数のクラスタにアクセスしたいとき以外は、
    ``create_connection`` メソッドを使用して、すべての処理がコネクションを自動で使用するようにすることを推奨します。


手引き
------

グローバルな構成を設定したくないのであれば、 ``using`` のパラメータとして
``elasticsearch.Elasticsearch`` のインスタンスを渡します:

.. code:: python

    s = Search(using=Elasticsearch('localhost'))

コネクションとして、既に関連付けているオブジェクトをオーバーライドすることもできます:

.. code:: python

    s = s.using(Elasticsearch('otherhost:9200')


.. _default connection:

デフォルトのコネクション
------------------

グローバルに使用されるデフォルトのコネクションを定義するときは
``connections`` モジュールと ``create_connection`` メソッドを使用します:

.. code:: python

    from elasticsearch_dsl import connections

    connections.connections.create_connection(hosts=['localhost'], timeout=20)

このとき、いくつかのキーワード引数(例では ``hosts`` と ``timeout`` ) が ``Elasticsearch`` クラスに渡されます。

複数のクラスタ
-----------------

``configure`` メソッドを使用して、複数のクラスタに対して複数のコネクションを定義することができます:

.. code:: python

    from elasticsearch_dsl import connections

    connections.configure(
        default={'hosts': 'localhost'},
        dev={'hosts': ['esdev1.example.com:9200'], sniff_on_start=True}
    )

これらのコネクションは最初のリクエストを受け取ったときに遅延して構成されます。

あるいは、1つずつ加えることもできます:

.. code:: python

    # Elasticsearch.__init__ に対して設定を行う場合
    connections.create_connection('qa', hosts=['esqa1.example.com'], sniff_on_start=True)

    # Elasticsearchのインスタンスがすでに存在する場合
    connections.add_connection('qa', my_client)

エイリアスの使用
~~~~~~~~~~~~~

複数のコネクションを使用する場合、事前に設定したエイリアス文字列でコネクションを参照することができます:

.. code:: python

    s = Search(using='qa')

もしエイリアスで設定されたコネクションが存在しない場合は ``KeyError`` が発生します。
