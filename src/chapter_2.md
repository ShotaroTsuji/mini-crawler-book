# ウェブページを取得する

まずは[`reqwest`](https://crates.io/crates/reqwest)クレートを使ってウェブページを取得するコードを書いてみましょう。最初は使い勝手を確かめるために`src/main.rs`に直に書いていくことにします。`reqwest`クレートにどんな機能があるかを知るために、 [Docs.rs](https://docs.rs/) にある[ドキュメント](https://docs.rs/reqwest/0.10.8/reqwest/)を見てみましょう。

最初に目に入ってくる文章をざっと読むと、`reqwest`クレートは非同期なHTTPクライアントとブロッキングHTTPクライアントの機能を提供していることが分かります。今回は非同期プログラミングは必要ないので、ブロックするクライアントを使うことにしましょう。

## サンプルコードを動かす

[`reqwest::blocking`](https://docs.rs/reqwest/0.10.8/reqwest/blocking/index.html)モジュールのドキュメントを開きます。ドキュメントの"§Making a GET request"に書かれたサンプルコードをひとまずコンパイルできるようにしてしまいます。

`Cargo.toml`の`dependencies`セクションに`reqwest`への依存を書きます。ドキュメントには`blocking` featureを有効にする必要があると書かれているので以下の記述を追加します。また、`main`関数で`?`演算子を使いたいのでエラーハンドリングのために[`eyre`](https://crates.io/crates/eyre)クレートへの依存も宣言します。

```toml:Cargo.toml
[dependencies]
reqwest = { version = "0.10.8", features = ["blocking"] }
eyre = "0.6"
```

[`reqwest::blocking`](https://docs.rs/reqwest/0.10.8/reqwest/blocking/index.html)モジュールのサンプルコードを`main`関数の中に貼り付けて、プログラムが動作するようにコードを書き足したものが以下のソースコードになります。`main`関数の戻り値を`Result`型にすることで`?`演算子を使ってエラーを返すことができるようになります。`std::result::Result<T, E>`を使うと`E`に適切な型を入れる必要があるので、ここでは簡単に済ませるために[`eyre::Result<T>`](https://docs.rs/eyre/0.6.3/eyre/type.Result.html)を使いました。

```rust
fn main() -> eyre::Result<()> {
    let body = reqwest::blocking::get("https://www.rust-lang.org")?
        .text()?;

    println!("body = {:?}", body);

    Ok(())
}
```

これで`cargo run`を実行すればコンパイルされた後にプログラムが実行されて、HTMLが出力される様子が見られるでしょう。（ちなみにWiFiをオフにするかEthernetケーブルを抜くかして、インターネットへの疎通がない状態にしてから`cargo run`を実行してみましょう。エラーメッセージが表示されるはずです。）

## ソースコードの読解

ソースコードを読んでいきましょう。まず、[`reqwest::blocking::get`](https://docs.rs/reqwest/0.10.8/reqwest/blocking/fn.get.html)関数を呼び出しています。この関数は`reqwest::IntoUrl`トレイトを実装した型を引数とする関数です。`Url`自身と`&str`型と`&String`型がこのトレイトを実装します[^1]。URLではない文字列が与えられるとエラーを返します。

このサンプルコードを読むと、プロトコルがHTTPなのか、あるいはHTTPSなのかはプログラマは気にしなくていいことがわかります。HTTPとHTTPSで通信をするのに必要な処理は異なりますが、そのあたりは`reqwest`が適切にやってくれるようです。

さて、`get`関数の戻り値の型は`Result<Response>`です。この`Result`は`reqwest::Result`のことです。エラー型には`reqwest`が定義している[`reqwest::Error`](https://docs.rs/reqwest/0.10.9/reqwest/struct.Error.html)が入っています。エラーについては飛ばして`Response`について調べることにしましょう。

[`Response`](https://docs.rs/reqwest/0.10.8/reqwest/blocking/struct.Response.html)のドキュメントを開きましょう。`Response`にはいろいろとメソッドが定義されていますが、今回呼び出している[`text`](https://docs.rs/reqwest/0.10.8/reqwest/blocking/struct.Response.html#method.text)メソッドのドキュメントを読んでみます。`text`メソッドの戻り値の型は`Result<String>`です。説明を読むとレスポンスボディをヘッダで指定されたエンコーディングに従ってデコードして返すようです。

最後に[`eyre::Result<T>`](https://docs.rs/eyre/0.6.3/eyre/type.Result.html)について説明しておきましょう。この型は`Result<T, eyre::Report>`として定義されています。[`eyre::Report`](https://docs.rs/eyre/0.6.3/eyre/struct.Report.html)は`Error + Send + Sync + 'static`を実装したオブジェクトから作ることができます。さまざまな型のエラーを一つの型に変換することで、最終的にプログラムが報告するエラーの取り扱いを容易にすることと、バックトレースを提供するのが`eyre::Report`の役目です。

## この章で作成したファイル

- [`Cargo.toml`](https://github.com/ShotaroTsuji/mini-crawler/blob/ch02/Cargo.toml)
- [`src/main.rs`](https://github.com/ShotaroTsuji/mini-crawler/blob/ch02/src/main.rs)

## 演習

1. インターネットへの疎通がない状態でプログラムを実行してエラーが出るか確かめてみましょう。
2. `get`関数に渡すURLをUTF-8以外でエンコーディングされているウェブページにしてプログラムを動かしてみましょう。例えば https://www.hmv.co.jp/ はShift-JISでエンコードされていて、`Content-Type`ヘッダで文字コードが指定されています。
    （`reqwest`は`Content-Type`ヘッダで文字コードが指定されていない場合はUTF-8と仮定してボディを読み込みます。http://abehiroshi.la.coocan.jp/ はShift-JISでエンコードされたページですが、`Content-Type`ヘッダがありません。）
3. `get`関数に渡すURLを存在しないページを指すURLにしてみましょう。このとき、レスポンスのステータスコードも表示するようにしてみましょう。（ステータスコードは`Response`構造体の`status`メソッドで取得することができます。）

[^1]: このことはドキュメントを読むだけではわかりません。公開されていないトレイト`PolyfillTryInto`に実装があります。これは[ソースコード](https://docs.rs/reqwest/0.10.9/src/reqwest/into_url.rs.html#8-38)を読めばわかります。
