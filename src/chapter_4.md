# 構造体とメソッドを作る

これまで書いてきたリンクを抽出するコードを`main`関数の外に移します。ウェブページを取得してリンクを抽出する機能を部品として扱えるようにして、後で作る幅優先探索の機能から呼び出しやすいようにします。

## `src/lib.rs`

リンクを抽出する機能を実装する構造体の名前を`LinkExtractor`としましょう。この構造体は[`reqwest::blocking::Client`](https://docs.rs/reqwest/0.10.9/reqwest/blocking/struct.Client.html)のオブジェクトを保持します。`Client`はHTTPクライアントとしての設定等を保持する構造体です。[`reqwest::blocking::get`](https://docs.rs/reqwest/0.10.9/reqwest/blocking/fn.get.html)関数は呼び出すたびにこの`Client`オブジェクトを作るので、効率のために一度作ったら使い回すようにします。

`Client`は[`ClientBuilder`](https://docs.rs/reqwest/0.10.9/reqwest/blocking/struct.ClientBuilder.html)を使って作成することでUser-Agentなどの設定が可能です。`LinkExtractor`が直接`Client`を作るようにすると、`LinkExtractor`に設定を渡す必要が出てきて煩雑になるので、`LinkExtractor`には`main`関数が作った`Client`オブジェクトを渡すようにします。

```rust:src/lib.rs
use reqwest::blocking::Client;

pub struct LinkExtractor {
    client: Client,
}

impl LinkExtractor {
    pub fn from_client(client: Client) -> Self {
        Self {
            client: client,
        }
    }
}
```

それではこれまで`src/main.rs`に書いたコードを`LinkExtractor`の実装に移しましょう。`LinkExtractor`の実装を`src/lib.rs`に書きます。`LinkExtractor`には`&self`と`Url`を受け取って`Result<Vec<Url>, eyre::Report>`を返す、`get_links`という名前のメソッドを定義します。また、抽出したURLを保存するために、`Vec<Url>`型の変数`links`を宣言します。そして、抽出したURLを`links`に`push`していくようにコードを書き換えます。

```rust:src/lib.rs
    pub fn get_links(&self, url: Url) -> Result<Vec<Url>, eyre::Report> {
        let response = self.client.get(url)?;
        let base_url = response.url().clone();
        let body = response.text()?;
        let doc = Document::from(body.as_str());
        let mut links = Vec::new();

        for href in doc.find(Name("a")).filter_map(|a| a.attr("href")) {
            match Url::parse(href) {
                Ok(url) => { links.push(url); },
                Err(UrlParseError::RelativeUrlWithoutBase) => {
                    let url = base_url.join(href)?;
                    links.push(url);
                },
                Err(e) => { println!("Error: {}", e); },
            }
        }

        Ok(links)
    }
```

`cargo build`を実行してプログラムをビルドしてみましょう。今のコードではエラーが出てコンパイルが通らないはずです。なぜなら、`Client`構造体の[`get`](https://docs.rs/reqwest/0.10.9/reqwest/blocking/struct.Client.html#method.get)メソッドは`Response`ではなく[`RequestBuilder`](https://docs.rs/reqwest/0.10.9/reqwest/blocking/struct.RequestBuilder.html)を返すからです。

`RequestBuilder`構造体は、ヘッダやボディを設定してHTTPリクエストを表現する`Request`構造体を生成するビルダーです。今回は特に何も設定せずに[`send`](https://docs.rs/reqwest/0.10.9/reqwest/blocking/struct.RequestBuilder.html#method.send)メソッドを呼んでしまいましょう。

というわけで、
```rust
        let response = self.client.get(url)?;
```
となっている行を
```rust
        let response = self.client.get(url).send()?;
```
に書き換えましょう。これでコンパイルできるようになります。

## `src/main.rs`

それでは、`main`関数に書いていたコードを消してしまって、今書いた`LinkExtractor`を使ってリンクを抽出するようにします。`ClientBuilder`で`Client`を作成して、`LinkExtractor`に渡します。そして、`get_links`メソッドに`Url`オブジェクトを渡してURLのベクタを得ます。最後にベクタの中身を表示します。

```rust:src/main.rs
use reqwest::blocking::ClientBuilder;
use url::Url;
use mini_crawler::LinkExtractor;

fn main() -> eyre::Result<()> {
    let url = Url::parse("https://www.rust-lang.org")?;
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

## この章で作成したファイル

- [`Cargo.toml`](https://github.com/ShotaroTsuji/mini-crawler/tree/v0.4.0)
- [`src/main.rs`](https://github.com/ShotaroTsuji/mini-crawler/blob/v0.4.0/src/main.rs)
- [`src/lib.rs`](https://github.com/ShotaroTsuji/mini-crawler/blob/v0.4.0/src/lib.rs)

## 演習

1. コマンドライン引数から取得したいページのURLを渡せるようにしましょう。コマンドライン引数でURLが指定されなかった場合は、https://www.rust-lang.org を読みにいくようにしましょう。
