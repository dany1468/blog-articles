---
title: "ローカルの開発中に CORS で失敗するときはポートの確認をしてもいい"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Flask, CORS]
published: true
---

今回は Flask を例に出しているのですが、他にも当てはまるところはあると思います。

## Flask の CORS の設定

ここに関しては `flask-cors` を入れておけば、まずは開始できるので、そこは問題ないです。  
https://flask-cors.readthedocs.io/en/latest/

```python
from flask import Flask
from flask_cors import CORS

app = Flask(__name__)
CORS(app)
```

今回は Flask のサーバーを立てて、そこに対して Angular からアクセスしているとします。

## ローカルで開発しているときに CORS で失敗する

ローカルで開発しているときに、CORS で失敗することがあります。

以下は、Angular から Flask にアクセスした時のブラウザの開発者ツールに出力されたエラーです。

Flask は 5000 番のポートがデフォルトになっているので、それに対して Angular (4200 番ポート) からアクセスしています。

```
Access to fetch at 'http://127.0.0.1:5000/sample' from origin 'http://localhost:4200' has been blocked by CORS policy: 
The 'Access-Control-Allow-Origin' header contains multiple values '*, http://localhost:4200', 
but only one is allowed. Have the server send the header with a valid value, or, 
if an opaque response serves your needs, set the request's mode to 'no-cors' to fetch the resource with CORS disabled.
```

開発者ツールの Network の方を見ると以下のように CORS とは出ているのですが内容は POST になっており、preflight の OPTIONS リクエストの結果がありません。

![](https://storage.googleapis.com/zenn-user-upload/8d5c99bb8db7-20230605.png)

### 厄介な点

私の環境のこの事象の厄介な点は、「これが必ず発生するわけではない」というところでした。

うまくいく日は全く問題がなく、突然 CORS で失敗し続け、再起動したりなんなりをするとまたうまくいく（時もある）という感じで頭を悩ませていました。  

## 調査

### OPTIONS のリクエストが来ていない？

上記の通り、常に発生する事象でもないということで、うまく調査ができていなかったのですが、本腰入れて調査しようとした時にまず気づいたのは OPTIONS のリクエストが来ていないということでした。
もちろんログには出ておらず、デバッガを使っても、他のエンドポイントは叩けるのに、OPTIONS は break point で停止しませんでした。

おかしい。

### preflight リクエストの詳細を見る

いよいよパケットキャプチャかと思いましたが、以下の記事で開発者ツールにも以前はあったということを知りました。

https://zenn.dev/yktakaha4/articles/analyze_cors_request_with_chrome

しかも HAR ファイルを取得すれば内容を確認できるということも！

![](https://storage.googleapis.com/zenn-user-upload/3420fb90a71f-20230605.png)

**Server AirTunes** ！！！！

AirTunes を調べてみると、Apple の AirPlay で使われているプロトコルでした。なるほど、、そもそもサーバーに飛んでいないというのはこういうことだったのか。

## ポートの確認

```
sudo lsof -P -i:5000
Password:
COMMAND     PID USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
ControlCe  1046 xxxx    7u  IPv4 0xxxxxaf16a0132b15      0t0  TCP *:5000 (LISTEN)
ControlCe  1046 xxxx    8u  IPv6 0xxxxxaf0d0b8bcbed      0t0  TCP *:5000 (LISTEN)
```

`ControlCe` 

ここまで辿り着くと、`ControlCe` と Flask で困っている人はたくさんいました :sweat_smile:

- [MacをMontereyにアップデートしたらFlaskが5000番ポートで起動できなくなった - Sweet Escape](https://www.keisuke69.net/entry/2021/10/29/012608)
- [Macでport:5000が使えないときの対処 - Qiita](https://qiita.com/JNJDUNK/items/6e0573448ac180ee9777)
- [【雑学】Macでlocalの5000番ポートのサーバーをたてようとしたら出来なかったお話し....!](https://tektektech.com/mac-local-5000-port-already-in-use/)

## まとめ

今回、私の場合は、preflight のリクエストが AirPlay のポートに引っかかり、通常の Flask の起動や他のエンドポイントには影響がなかったので、気づくのに時間がかかってしまいました。  

これ以外にも Firebase Local Emulator Suite の Storage アクセスも同じように成功する時と失敗する時があり、これも同じではなかったもののポートを調べてみると他のプロセスが混じっており、それを kill するとうまくいくという事象がありました。
