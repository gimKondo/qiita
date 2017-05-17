# RailsのActionCableにOriginに指定したURL以外から接続する方法
## 概要
- RailsのActionCableはサーバで設定したURL以外からはアクセスできない
- しかし、テストなどで他のURLからアクセスしたいことがある
- HTTPヘッダのOriginを偽装することでアクセスできる

## 背景
ActionCableを使ったアプリケーションの負荷試験のため、任意のクライアントで動作するスクリプトからActionCalbleに接続したくなった。
しかし、単純にやると接続に失敗してしまう。
これはCSRF(Cross Site Request Forgeries)を防ぐための機能によって、Railsで指定したサイト以外からのアクセスを拒否するようになっているためである。

- `config.action_cable.disable_request_forgery_protection = true` と設定し、セキュリティ機能を無効化する
- `config.action_cable.allowed_request_origins` でスクリプトを実行するURLを許可設定に追加する
    - 正規表現の配列を指定し、許可するURLを設定できる
    - 通常、ActionCableを使用するアプリケーションのURLをセットする

のようなサーバ側の設定変更による対応(参考: http://qiita.com/wakaba260/items/b23721d5cfd73cd47ada)も可能ではあるが、テストのためにサーバの設定を変更するのは、できれば避けたい。


## 対応
ということで、テストスクリプト側で工夫する。

まず、ActionCableのCSRF対策機能が何をやっているかを見てみる。
この機能は単純で、HTTPヘッダのOriginを見て、それが設定した許可URLに合致した場合にだけ接続を許可するようになっている。
つまり、HTTPヘッダさえ偽装すれば、CSRF対策を回避し、接続できるという寸法である。

以下はその実例である。
テストスクリプトにはRubyを使用し、ActionCalbeのwebsocketへの接続は先人の作ってくれたgemを利用している。

```ruby
# Rails側で、このようにOriginが設定されているものとする
config.action_cable.allowed_request_origins = ["https://myapp.example.com"]
```

```ruby
# websocket-client-simpleを使う場合の例
origin_url = "https://myapp.example.com"
ws_url = "https://myapp.example.com/cable"
ws = WebSocket::Client::Simple.connect(ws_url, headers: { Origin: origin_url })
```

```ruby
# websocket-eventmachine-clientの場合
ws = WebSocket::EventMachine::Client.connect(uri: ws_url, headers: {"Origin" => origin_url})
```

以上で、CSRF対策を有効にしたまま、ActionCableに接続することができる。
