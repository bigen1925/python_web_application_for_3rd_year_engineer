---
title: "「伸び悩んでいる3年目Webエンジニアのための、Python Webアプリケーション自作入門」を更新しました"
emoji: "🚶"
type: "tech"
topics: [python, web, http]
published: true
---

# 本を更新しました

[チャプター「リクエストを並列処理する」](https://zenn.dev/bigen1925/books/introduction-to-web-application-with-python/viewer/threading) を更新しました。

続きを読みたい方は、ぜひBookの「いいね」か「筆者フォロー」をお願いします ;-)

----

以下、書籍の内容の抜粋です。

------

## 現状の問題点

pythonはソースコードを上から順に読み込み、ある1行の処理が完了すれば次の1行、というふうに順番に実行していきます。

そのため、現状のソースコードだと、

- クライアントからの接続を受け付ける(`.accept()`)
- クライアントからのリクエストを処理する（`.handle_client()`）
- (必要であれば例外処理をしたあと)クライアントとの通信を終了する(`.close()`)
- クライアントからの接続を受け付ける(`.accept()`)
- ...

必ずこの順序を守って実行されます。

言い換えると、
**「1つのリクエストの処理が全て完了するまで、次のリクエストの受付が始まらない」**
ということになります。

![](https://storage.googleapis.com/zenn-user-upload/6u31xkraj4j5755xm5lyi8wz9fmm)

これはリクエストの数が増えてくると大きな問題になります。

-------

例えば10件のリクエストが同時にきたとします。

このとき、サーバーは早いもの勝ちで最初に来たリクエストの処理を開始します。
そしてこの最初のリクエストの処理がとても時間がかかる処理の場合、処理が完了するまで後続の9件のリクエストは待たされることになってしまいます。

よくあるのは、複雑な検索条件のクエリをDBサーバーに投げて応答を30秒間ずっと待ってるなどのケースです。
DBからの応答を待つ30秒の間、マシンが他の9件のリクエストを処理してくれたら全体的なパフォーマンスはぐっと向上します。

このように、プログラムがある処理の完了を待たずにに、その間に裏で別の処理を実行させることを **並列処理 **または** 並行処理**と呼びます。

並列処理は一般的なWebサーバーであれば当たり前に実装されていますので、私達も実装していくことにしましょう。


## ソースコード
並列処理を行うように改良したソースコードがこちらです。
ファイルが2つになっていますので、ご注意ください。

**`study/webserver.py`**
https://github.com/bigen1925/introduction-to-web-application-with-python/blob/main/codes/chapter13/WebServer2.py

**`study/workerthread.py`**
https://github.com/bigen1925/introduction-to-web-application-with-python/blob/main/codes/chapter13/WebServer2.py

## 解説
### `共通`
ファイルを分けたので、処理の記録を示すログにそれぞれ`Server: `, `Worker: `という文字を出力ようにしています。
例）

```python
print("=== Server: サーバーを起動します ===")
```

```python
print("=== Worker: クライアントとの通信を終了します ===")
```

### `webserver.py`

#### 28-31行目
```python
                # クライアントを処理するスレッドを作成
                thread = WorkerThread(client_socket)
                # スレッドを実行
                thread.start()
```

コネクションを確立したクライアントを処理する **スレッド** を作成し、スレッドの処理を開始させます。
前回まで`.handle_client()`というメソッドで行っていた処理と、前後の例外処理は全てこのスレッド内の処理にお引越ししました。

**スレッド**とはコンピュータが並列に処理を行うことが可能な処理系列のことで、後ほど詳細を説明します。

### `workerthread.py`

#### 9行目-57行目
```python
class WorkerThread(Thread):
    # 実行ファイルのあるディレクトリ
    BASE_DIR = os.path.dirname(os.path.abspath(__file__))
    # 静的配信するファイルを置くディレクトリ
    STATIC_ROOT = os.path.join(BASE_DIR, "static")

    def __init__(self, client_socket: socket):
        super().__init__()

        self.client_socket = client_socket

    def run(self) -> None:
        """
        クライアントと接続済みのsocketを引数として受け取り、
        リクエストを処理してレスポンスを送信する
        """
        
        # クライアントへ対応する処理...
```

`Thread`はpythonの組み込みライブラリである`threading`モジュールに含まれるクラスで、スレッドを簡単に作成するためのベースクラスです。

`Thread`を利用する際は、`Thread`を継承したクラスを作成し、`.run()`メソッドをオーバーライドします。
このクラスのインスタンスは`.start()`メソッドを呼び出すことで新しいスレッドを作成し、`.run()`メソッドにかかれた処理を開始します。

------

CPUは1つのスレッド内で実行されるプログラムは並列処理できませんが、複数のスレッドに分かれて実行されるプログラムは並列処理することが可能です。

これまでは、1つのスレッド内で`WebServer`クラスが全てのクライアントの対応を逐次に処理していました。

![](https://storage.googleapis.com/zenn-user-upload/6yl6fctmzsqszy1wo9bjyun6oal2)

しかし今回からは、スレッドを利用することで`WebServer`クラスはクライアントとのコネクション確立までは行うものの、リクエスト内容への対応は別スレッドで並列処理するようになりました。

![](https://storage.googleapis.com/zenn-user-upload/1wsxaxy25tb3ne1hz00uw1ncqnxo)

これにより、とあるリクエストの処理が長引いてるせいで他のリクエストが受け付けられない、という状況が回避されます。

:::details コラム: スレッドを扱う際の注意点

基本的に別々のスレッドの処理は別々のプログラムとして動くことになるので、あるスレッドで例外が発生しても別スレッドには伝搬しないということです。

例えば、以前までは同一スレッド内でリクエストを処理していたため、リクエストの処理中に例外が発生した場合、例外のハンドリングを行わなければメインの処理（リクエストの受付処理）まで終了していました。
今回からは別々のスレッド内でリクエストを処理しているため、とあるスレッドでリクエストの処理中に例外が発生しても、例外ハンドリングがない場合はそのスレッドが終了するだけでメインスレッドには影響がありません。

一見ありがたいようにも見えますが、サーバー全体に影響のある異常事態が発生した場合にも処理が続行してしまうようではいけません。

スレッドを使う際は、例外処理に敏感になっておきましょう。

------

また、スレッドをどんどん分岐させれば、処理はどこまでも早くなるというわけでもありません。

CPUが同時に処理できる量には限界があります。
並列実行している処理の多くがCPUを大量に利用するような状況では、CPUの処理量が限界に達して並列実行が滞り、パフォーマンスが上がらないor却って下がるケースがあります。
このようにCPUの性能がネックになって処理速度の限界を迎えるような処理を、`CPUバウンドな処理`と言います。

CPUバウンドな処理が多いWebサービスでは、むやみにスレッドを増やしすぎるとCPUの処理量が限界を迎えて他のプログラムの実行速度に影響を与えてしまうこともあります。
Webサーバーの多くがスレッド数やプロセス数に上限を設定できるのは、これを防ぐためです。

自分のプログラムが、最悪のケースでどれぐらいのスレッド分岐が行われるのかには十分気をつけましょう。

本書でも、本来は分岐できるスレッド数に上限を設けるべきで、pythonにはそのための`ThreadPoolExecutor`というクラスが用意されています。
興味がある方は調べてみてください。

:::

# 動かしてみる
では、実際に動かしてみましょう。

--------

# 続きはBookで！

[チャプター「リクエストを並列処理する」](https://zenn.dev/bigen1925/books/introduction-to-web-application-with-python/viewer/threading) 
