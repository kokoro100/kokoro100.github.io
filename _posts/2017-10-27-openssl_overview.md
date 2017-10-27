---
layout: post
title:  "OpenSSL メモ"
date:   2017-10-27 00:43:50 +0900
categories: ssl
---

OpenSSL は libssl と libcrypto の二つからできている

### [libssl](https://www.openssl.org/docs/manmaster/ssl/ssl.html)
---
通信まわりを実装したライブラリ。

### [libcrypt](https://www.openssl.org/docs/manmaster/crypto/crypto.html)
---
暗号関連を実装したライブラリ。

この記事では libssl を扱う。

### Session と Connection
---

Session は OSI参照モデル五層のセッション層に相当する。通信開始から通信終了までの一つの接続単位をさす。

Connection は OSI参照モデル四層のトランスポート層に相当する。SSL では 一般に TCPコネクションとなる。

一つの Session につき一つ以上の Connection が存在する。Session が維持されている限り Connection が切断されても Session を再開することができる。具体的には、Session 情報をファイルに保存しておき、再起動後に Session を再開することができる。

SSL の場合、Session は共有秘密情報（master_secret）を持つ。Session に関連付けられた各 Connection は、それぞれ暗号スイートやシーケンス番号、暗号化鍵、MAC 鍵を独立して持つことに注意する。

### 参照カウンタ
---
OpenSSL API が返す構造体は参照カウンタで寿命管理されている（ことが多い）。

OpenSSL API には `get0` と `get1` というサフィックスがついていることがあり、それぞれぞれ参照カウンタの扱いが異なる。

`get0` は呼び出し側が参照を保持しない（参照カウンタがインクリメントされない

`get1` は呼び出した側が参照を保持する（参照カウンタがインクリメントされる

無印の `get` は基本的に `get0` と同様である（関数によって異なる模様）

#### 例. `SSL` から `SSL_SESSION` を取得する get API

{% highlight c++ %}
SSL_SESSION *SSL_get_session(const SSL *ssl);
SSL_SESSION *SSL_get0_session(const SSL *ssl);
SSL_SESSION *SSL_get1_session(SSL *ssl);
{% endhighlight %}

`get1` で参照を保持した場合は各構造体の free を呼ぶことで適切に参照を手放すこと。基本的には `<構造体名>_free` という命名規則になっている。

{% highlight c++ %}
void SSL_SESSION_free(SSL_SESSION *session);
{% endhighlight %}

### データ構造
---

OpenSSL API の主なデータ構造について

#### SSL_CTX

SSL についてグローバルな設定を保持するオブジェクト。
プロトコルや暗号スイート、セッションキャッシュ設定、コールバック、鍵、証明書等を保持する

{% highlight c++ %}
SSL_CTX *SSL_CTX_new(const SSL_METHOD *method);
{% endhighlight %}

#### SSL_METHOD

SSL のプロトコルを実装する関数のデータ構造。`SSL_METHOD` を `SSL_CTX_new` に渡し利用するプロトコルを指定する。
特定のプロトコルバージョンを指定して利用する場合は下記のメソッドを指定する（非推奨）。

[openssl method](https://www.openssl.org/docs/manmaster/ssl/SSL_CTX_new.html)

{% highlight c++ %}
const SSL_METHOD *SSLv3_client_method(void);
const SSL_METHOD *SSLv3_server_method(void);
const SSL_METHOD *SSLv3_method(void);

const SSL_METHOD *TLSv1_client_method(void);
const SSL_METHOD *TLSv1_server_method(void);
const SSL_METHOD *TLSv1_method(void);

const SSL_METHOD *SSLv23_client_method(void);
const SSL_METHOD *SSLv23_server_method(void);
const SSL_METHOD *SSLv23_method(void);
{% endhighlight %}

プロトコルのバージョン上限/下限を指定して利用する場合は下記メソッドを指定する（こちらが推奨）。

{% highlight c++ %}
const SSL_METHOD *TLS_method(void);
const SSL_METHOD *TLS_server_method(void);
const SSL_METHOD *TLS_client_method(void);
{% endhighlight %}

上記メソッドでプロトコルバージョンを指定した場合、クライアント-サーバーのネゴシエーションにより SSLv3, TLSv1, TLSv1.1 , TLSv1.2 の中で選択可能なうち最も高いバージョンが選択される。
もしプロトコルバージョンの上限もしくは下限を指定したい場合、下記メソッドを用いる。


{% highlight c++ %}
int SSL_CTX_set_min_proto_version(SSL_CTX *ctx, int version);
int SSL_CTX_set_max_proto_version(SSL_CTX *ctx, int version);
int SSL_set_min_proto_version(SSL *ssl, int version);
int SSL_set_max_proto_version(SSL *ssl, int version);
{% endhighlight %}

version にはのいずれかを指定する。

{% highlight c++ %}
SSL3_VERSION
TLS1_VERSION
LS1_1_VERSION
TLS1_2_VERSION
DTLS1_VERSION
DTLS1_2_VERSION
{% endhighlight %}


特定バージョンの使用禁止をする `SSL_CTX_set_options` と併用することができる。
だが、必要が無い限りは上記メソッドを用いるべきである。

#### SSL_CHIPHERs

SSL/TLS における暗号スイートを規定する。
[https://www.openssl.org/docs/manmaster/ssl/SSL_CTX_set_cipher_list.html](https://www.openssl.org/docs/manmaster/ssl/SSL_CTX_set_cipher_list.html)

{% highlight c++ %}
char *SSL_CIPHER_description(SSL_CIPHER *cipher, char *buf, int len);
Write a string to buf (with a maximum size of len) containing a human readable description of cipher. Returns buf.

int SSL_CIPHER_get_bits(SSL_CIPHER *cipher, int *alg_bits);
Determine the number of bits in cipher. Because of export crippled ciphers there are two bits: The bits the algorithm supports in general (stored to alg_bits) and the bits which are actually used (the return value).

const char *SSL_CIPHER_get_name(SSL_CIPHER *cipher);
Return the internal name of cipher as a string. These are the various strings defined by the SSL3_TXT_xxx and TLS1_TXT_xxx definitions in the header files.

char *SSL_CIPHER_get_version(SSL_CIPHER *cipher);
Returns a string like &quot;SSLv3&quot; or &quot;TLSv1.2&quot; which indicates the SSL/TLS protocol version to which cipher belongs (i.e. where it was defined in the specification the first time).
{% endhighlight %}


#### SSL_SESSION

SSL Session を表すデータ構造。SSL Connection における SSL_CHIPHERs, 証明書, 鍵, etc を保持する。
Sesseion の再利用（再度 Handshake しない）でも利用する。再利用期間は最大でも24時間程度が推奨されている。

#### SSL

SSL Connectionを管理する構造体。
SSL/TLS 実装における主要なデータ構造であり、様々な OpenSSL API の起点となる


### 証明書関連
---
OpenSSL で証明書を取り扱う際に登場するデータ構造について

#### x509

証明書データ。

#### X509_STORE

証明書や CRL を保持するストア

#### X509_STORE_CTX

証明書関連の処理に生成されて、処理が終わると破棄される。テンポラリなオブジェクト

### その他
---
#### Session Ticket

SSL セッションのキャッシュをサーバーではなくクライアント側に保持する仕組みがある（RFC5077 Session Resumption without Server-Side State）。これは、セッション情報をサーバー側で暗号化して（クライアントには解読不可能）クライアント側で保持し、セッション再開時にそれをクライアントがサーバーへと送信、サーバー側での検証結果に問題なければ送られてきた Session 情報から Session を再開するというもの。このとき、この暗号化された Session 情報を Session Ticket という。

## callback 登録による挙動の変更
---

OpenSSL ではコールバック関数を登録することによって多くの実装をデフォルトからカスタマイズすることができる。

#### 例.session_id の生成とマッチング

{% highlight c++ %}
typedef int (*GEN_SESSION_CB)(const SSL *ssl, unsigned char *id, unsigned int *id_len);

int SSL_CTX_set_generate_session_id(SSL_CTX *ctx, GEN_SESSION_CB cb);
int SSL_set_generate_session_id(SSL *ssl, GEN_SESSION_CB, cb);
int SSL_has_matching_session_id(const SSL *ssl, const unsigned char *id,
                                unsigned int id_len);
{% endhighlight %}

後述の証明書チェーン検証もコールバック関数を仕込むことでカスタムできる。

## 実装の流れ-SSLコネクションをはる
---
OpenSSL 具体的な実装手順について述べる。cURL と併用した場合についてもいずれまとめる。


1. `SSL_library_init` で初期化
2. `SSL_CTX_new` で TLS/SSL Connection のためのコンテキストを作成する。このContext に対して Certificates や Algorithms の設定を行う
3. `SSL_new` で SSL_CTX から SSL を作成する
4. `SSL_set_fd` で (socket API を使うならば) socket (connect 済み) を `SSL` に割り当てる
5. `SSL_accept`（サーバー）もしくは <code>SSL_connect</code>（クライアント）によって TLS/SSL handshale を実行する
6. `SSL_read`, `SSL_write` で TLS/SSL データを送受信する
7. `SSL_shutdown`で TLS/SSL Connection を終了する


## 実装の流れ-証明書検証の流れ
---
`SSL_connect` 時の証明書検証について

#### SSL_get_verify_result

{% highlight c++ %}
long SSL_get_verify_result(const SSL *ssl);
{% endhighlight %}

証明証の検証結果を返す。注意として、本関数では証明書の正当性のみを検証し、証明書の送り主の正当性（CN チェック）については検証しない。そのため、CN チェックは別途自分で実装すること。
更に、`SSL_get_verify_result` は証明書が送られてこなかった場合にも成功が返る仕様となっている。そのため、 <code>SSL_get_peer_certificate</code> で NULL が返らないかも合わせてチェックすること。

#### SSL_CTX_set_verify

{% highlight c++ %}
void SSL_CTX_set_verify(SSL_CTX *ctx, int mode,
                        int (*verify_callback)(int, X509_STORE_CTX *));
{% endhighlight %}

証明書検証の処理は `SSL_CTX_set_verify` で独自に実装することができる。
独自実装しなかった場合は、`X509_STORE` に登録された RootCA に基づき OpenSSL のデフォルト実装によって検証される。</p>

callback 関数は Certificate chain 中の各 Certificate を検証する度に呼びだされる。
検証の過程でエラーとなる箇所の度に呼び出されるため、一つの証明書に対して複数回呼びだされることがある。
基本的には、帰ってきたエラーについて許可するようなら真を、許可しないなら偽を返す実装となる。
本関数の結果によって最終的な `SSL_get_verify_result` の結果が決まる。

mode には下記のいずれかを指定する。

#### SSL_VERIFY_NONE

|役割|解説|
|:-|:-|
|クライアント|anonymous cipher を利用していないかぎり（デフォルト無効）|
|サーバー|クライアントによって検証される。検証結果は `SSL_get_verify_result` によって取得することができるが、証明書検証の成功失敗に関わらず handshake は継続される。|

#### SSL_VERIFY_PEER

|役割|解説|
|:-|:-|
|サーバー|クライアントに証明書お湯級を送信する。クライアントから送信された証明書は（もし存在すれば）検証され、証明書が無効であれば handshake は 失敗原因メッセージのアラートとともに直ちに終了される。挙動はさらに `SSL_VERIFY_FAIL_IF_NO_PEER_CERT` と `SSL_VERIFY_CLIENT_ONCE` によって変更する事ができる。|
|クライアント|サーバー証明書は検証される。もし証明書が無効である場合、 handshake は直ちに終了される。もしanonymous cipher によりサーバー証明書が送られなかった場合、SSL_VERIFY_PEER は無視される|

#### SSL_VERIFY_FAIL_IF_NO_PEER_CERT

|役割|解説|
|:-|:-|
|サーバー|SSL_VERIFY_PEER と一緒に使われなければならない。もしクライアントから証明書が送られなかったら、handshake は&rdquo;handshake failure&rdquo; アラートとともに直ちに終了する|
|クライアント|無視される|

#### SSL_VERIFY_CLIENT_ONCE

|役割|解説|
|:-|:-|
|サーバー|SSL_VERIFY_PEER と一緒に使われなければならない。一番最初の handshake でのみクライアント証明書を要求する。再ネゴシエーションでは要求しない|
|クライアント|無視される|
