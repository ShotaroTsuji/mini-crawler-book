# ロギングとエラー処理

ここまででリンクを抽出する機能は概ね完成しましたが、細かな機能としてロギングとエラー処理を追加します。

## ロギング

シングルスレッド環境でのロギングには[`log`](https://crates.io/crates/log)クレートを使います。
`log`クレートは debug や warn といったログレベル付きのメッセージを表示するためのマクロを提供します。
ただし、実際に表示を行う機能は分離されており別のクレートを使います。今回はシンプルな実装である[`env_logger`](https://crates.io/crates/env_logger)クレートを使います。

`src/lib.rs`に記述した、`LinkExtractor::get_links`にログを出力するコードを加えます。引数として渡されたURLと実際に取得したページのURLをログに記録しておくことにしましょう。ついでにステータスコードも記録しておきます。ログレベルは info でいいでしょう。では`get_links`メソッドでログを記録するために`log::info!`マクロにメッセージを渡すコードを書いておきましょう。

```rust:src/lib.rs
    pub fn get_links(&self, url: Url) -> Result<Vec<Url>, eyre::Report> {
        log::info!("GET \"{}\"", url);
        let response = self.client.get(url).send()?;
        let base_url = response.url().clone();
        let status = response.status();
        let body = response.text()?;
        let doc = Document::from(body.as_str());
        let mut links = Vec::new();
        log::info!("Retrieved {} \"{}\"", status, base_url);
```

また、ログの表示のために`main`関数で`env_logger::init`関数を呼んでおく必要があります。

```rust:src/main.rs
use reqwest::blocking::ClientBuilder;
use url::Url;
use mini_crawler::LinkExtractor;

fn main() -> eyre::Result<()> {
    env_logger::init();

    let url = std::env::args()
        .nth(1)
        .unwrap_or("https://www.rust-lang.org".to_owned());
    let url = Url::parse(&url)?;
    let client = ClientBuilder::new()
        .build()?;
    let extractor = LinkExtractor::from_client(client);

    let links = extractor.get_links(url)?;
    for link in links.iter() {
        println!("{}", link);
    }

    Ok(())
}
```

さて、`mini-crawler`を動かしてログが表示されるか確かめてみましょう。`env_logger`クレートでログを表示するためには、`RUST_LOG`環境変数にログレベルを設定する必要があります。`env_logger`クレートは`RUST_LOG`環境変数で指定されたログレベル以上のレベルが設定されたログを表示します。今回は`RUST_LOG=info`と設定しておきます。

以下の例では`mini_crawler`に短縮URLを渡しました。2つ目のログでリダイレクト後のURLが表示されていることがわかります。

```
% RUST_LOG=info cargo run -- https://bit.ly/2J6BnlL
   Compiling mini-crawler v0.1.0 (/Users/shotaro/projects/mini-crawler)
    Finished dev [unoptimized + debuginfo] target(s) in 3.04s
     Running `target/debug/mini-crawler 'https://bit.ly/2J6BnlL'`
[2020-11-24T01:44:06Z INFO  mini_crawler] GET "https://bit.ly/2J6BnlL"
[2020-11-24T01:44:07Z INFO  mini_crawler] Retrieved 200 OK "https://www.rust-lang.org/"
-- snip --
```

## エラー処理

`LinkExtractor::get_links`はその中で呼び出した関数から返ってきたエラーを`eyre::Report`に詰めて返していました。簡単なプログラムであればこれでも問題はないと思いますが、練習のために独自のエラー型を定義しましょう。

とはいえ自力で[`Error`](https://doc.rust-lang.org/std/error/trait.Error.html)トレイトなどの必要なトレイトを実装するのは大変なので、[`thiserror`](https://crates.io/crates/thiserror)クレートを使って自動で実装しましょう。

エラーが返ってくる箇所を洗い出しましょう。`?`演算子を使った部分を探すことで見つけられます。まず、GETリクエストを送信してレスポンスを得るところでエラーが返ってくる可能性があります。

```rust:src/lib.rs
        let response = self.client.get(url).send()?;
```

次に、レスポンスからボディを`String`にして取り出す部分です。

```rust:src/lib.rs
        let body = response.text()?;
```

そして、URLを絶対URLに変換するときに`join`を呼び出すところです。

```rust:src/lib.rs
                    let url = base_url.join(href)?;
```

それぞれのエラーの原因は、
- リクエストの送信に起因するもの
- レスポンスボディの取得に起因するもの
- 絶対URLへの変換に起因するもの
になります。これらの原因を表現するために次の列挙型を定義します。

```rust:src/lib.rs
pub enum GetLinksError {
    SendRequest,
    ResponseBody,
    AbsolutizeUrl,
}
```

まずはエラーメッセージを書きます。`use`宣言で`thiserror::Error`を使うことを宣言してから`#[derive(Error)]`を使うことで、`thiserror`クレートのマクロを使うことができるようになります。そして各ヴァリアントに`#[error("...")]`を付けることで`Display`トレイトを自動で実装することができます。

```rust
use thiserror::Error;

#[derive(Error,Debug)]
pub enum GetLinksError {
    #[error("Failed to send a request")]
    SendRequest,
    #[error("Failed to read the response body")]
    ResponseBody,
    #[error("Failed to make the link URL absolute")]
    AbsolutizeUrl,
}
```

次に、エラーの原因を保持できるように各ヴァリアントに`#[source]`属性をつけた値を持たせることにします。

```rust
#[derive(Error,Debug)]
pub enum GetLinksError {
    #[error("Failed to send a request")]
    SendRequest(#[source] reqwest::Error),
    #[error("Failed to read the response body")]
    ResponseBody(#[source] reqwest::Error),
    #[error("Failed to make the link URL absolute")]
    AbsolutizeUrl(#[source] url::ParseError),
}
```

そして、`get_links`メソッドの戻り値の型を`Result<Vec<Url>, GetLinksError>`に変更して、エラーを返す箇所を

```rust:src/lib.rs
        let response = self.client.get(url).send()
            .map_err(|e| GetLinksError::SendRequest(e))?;
```

のように、`map_err`を使って型を変換してあげるようにすれば、コンパイルが通るようになります。

## その他こまかな処理

### ステータスコードの処理

現在の実装ではレスポンスのステータスコードが404や500などのエラーであってもリンクの抽出をしてしまいます。多くのウェブサイトではエラーを表すステータスコードを持つレスポンスにもHTMLのボディを付加します。

例えば、存在しないページのURLを引数に渡して`mini-crawler`を実行すると次の結果が得られます。

```
% RUST_LOG=info cargo run -- https://example.com/xxx
    Finished dev [unoptimized + debuginfo] target(s) in 0.09s
     Running `target/debug/mini-crawler 'https://example.com/xxx'`
[2020-11-24T10:24:11Z INFO  mini_crawler] GET "https://example.com/xxx"
[2020-11-24T10:24:12Z INFO  mini_crawler] Retrieved 404 Not Found "https://example.com/xxx"
https://www.iana.org/domains/example
```

https://example.com/xxx をブラウザで開いてみるとリンクが一つだけあります。なのでこの結果は正しいといえば正しいです。しかしながら、存在しないページにアクセスした場合はエラーを返してほしいと考えるのが自然でしょう。

レスポンスのステータスコードがエラーを表すものだったときに、`get_links`メソッドがエラーを返すようにするために、ステータスコードを見て条件分岐するのも一つの手ですが、`reqwest`クレートにはそのような処理にお誂え向きのメソッドが用意されています。

`Response`構造体の[`error_for_status`](https://docs.rs/reqwest/0.10.9/reqwest/blocking/struct.Response.html#method.error_for_status)メソッドは、サーバーがエラーを返した場合にレスポンスを`reqwest::Error`に変換します。早速このメソッドを使うように修正してみましょう。

まず、`GetLinksError`に新たなヴァリアント`ServerError`を追加します。

```rust:src/lib.rs
#[derive(Error,Debug)]
pub enum GetLinksError {
    #[error("Failed to send a request")]
    SendRequest(#[source] reqwest::Error),
    #[error("Failed to read the response body")]
    ResponseBody(#[source] reqwest::Error),
    #[error("Failed to make the link URL absolute")]
    AbsolutizeUrl(#[source] url::ParseError),
    #[error("Server returned an error")]
    ServerError(#[source] reqwest::Error),
}
```

そして、受け取ったレスポンスを、`error_for_status`を呼ぶことでエラーに変換します。

```rust:src/lib.rs
        let response = self.client.get(url).send()
            .map_err(|e| GetLinksError::SendRequest(e))?;
        let response = response.error_for_status()
            .map_err(|e| GetLinksError::ServerError(e))?;
```

以上の変更を加えてから`mini-crawler`に存在しないページのURLを渡すと動作が変わります。

```
% RUST_LOG=info cargo run -- https://example.com/xxx
   Compiling mini-crawler v0.1.0 (/Users/shotaro/projects/mini-crawler)
    Finished dev [unoptimized + debuginfo] target(s) in 3.07s
     Running `target/debug/mini-crawler 'https://example.com/xxx'`
[2020-11-24T10:49:56Z INFO  mini_crawler] GET "https://example.com/xxx"
Error: Server returned an error

Caused by:
    HTTP status client error (404 Not Found) for url (https://example.com/xxx)

Location:
    src/main.rs:16:41
```

### フラグメントの削除

これまで忘れていたのですが、`<a>`要素から取得したリンクに付いたフラグメントを削除する処理を入れることにします。フラグメントとはURLの`#`以降の部分のことでした。この部分はHTMLに対してはフラグメントと同じ名前の`id`属性値を持つ要素を指します。ブラウザではフラグメント付きのURLを開いたときは、該当する要素の位置までページをスクロールします。

なのでフラグメントはあってもなくても同じページを指しますし、普通はフラグメントをサーバーに送ることはしません。また、あとで幅優先探索を行うときに同じページに違う名前が付いているのは困るのでフラグメントは消してしまいます。

フラグメントを削除するには[`set_fragment`](https://docs.rs/url/2.2.0/url/struct.Url.html#method.set_fragment)メソッドに`None`を渡すだけでできます。

フラグメントの削除をするのと同時に、URLの操作でエラーが起きたときにエラーを返さずにログを残すように書き換えます[^1]。今回は`log::info!`ではなく`log::warn!`を使おうと思います。

```rust:src/lib.rs
        for href in doc.find(Name("a")).filter_map(|a| a.attr("href")) {
            match Url::parse(href) {
                Ok(mut url) => {
                    url.set_fragment(None);
                    links.push(url);
                },
                Err(UrlParseError::RelativeUrlWithoutBase) => {
                    match base_url.join(href) {
                        Ok(mut url) => {
                            url.set_fragment(None);
                            links.push(url);
                        },
                        Err(e) => {
                            log::warn!("URL join error: {}", e);
                        },
                    }
                },
                Err(e) => {
                    log::warn!("URL parse error: {}", e);
                },
            }
        }
```

## この章で作成したファイル

- [`Cargo.toml`](https://github.com/ShotaroTsuji/mini-crawler/blob/v0.5.0/Cargo.toml)
- [`src/main.rs`](https://github.com/ShotaroTsuji/mini-crawler/blob/v0.5.0/src/main.rs)
- [`src/lib.rs`](https://github.com/ShotaroTsuji/mini-crawler/blob/v0.5.0/src/lib.rs)

## 演習

1. HTTPレスポンスのヘッダをログに残すようにしてください。ログレベルはdebugにします。

[^1]: `get_links`が`Vec<Url>`を返すためにエラーを握り潰す実装を採用してしまいました。ライブラリコードとしてきちんとエラーを報告するためには`Vec<Result<Url>>`を返す方がよいでしょう。この場合、エラーをどう処理するかは呼び出した側に委ねます。
