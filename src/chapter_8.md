# コマンドラインインターフェースを作る

この章では`LinkExtractor`と`Crawler`を結合し、コマンドライン引数で探索を始めるURLと訪問ページ数の上限を与えられるプログラムを作成します。そのために、`LinkExtractor`に`AdjacentNodes`トレイトを実装します。そして、コマンドライン引数を`structopt`クレートでパースして得たパラメタを`Crawler`に渡して探索を実行するコードを書きます。

## `LinkExtractor`による`AdjacentNodes`トレイトの実装

`LinkExtractor`に`AdjacentNodes`トレイトを実装します。`adjacent_nodes`関数の中で`get_links`関数を呼び出すようにします。

```rust:src/lib.rs
impl crawler::AdjacentNodes for LinkExtractor {
    type Node = Url;

    fn adjacent_nodes(&self, v: &Self::Node) -> Vec<Self::Node> {
        self.get_links(v.clone())
            .unwrap()
    }
}
```

おっと、ここで問題が発生しました。`get_links`の戻り値は`Result`型ですが`adjacent_nodes`の戻り値はそうではありません。前の章で幅優先探索を実装するときにこのことをすっかり忘れていました。仕方がないのでひとまず`unwrap`を呼んでお茶を濁すことにしましょう。

また、`get_link`は`Url`型のオブジェクトを受け取ります。`adjacent_nodes`は参照を受け取るので`clone`を呼んでコピーを渡すようにします。

## `LinkExtractor`と`Crawler`の結合

それでは`main`関数を書き換えましょう。`LinkExtractor`のオブジェクトを作るところまではこれまでと変わりません。`LinkExtractor`のオブジェクトを作ったら、それへの参照とスタート地点となるURLを`Crawler`のコンストラクタに渡します。そして、`for`の中で`Crawler`のオブジェクトを使います。とりあえず10件のページを取得するために`take(10)`を呼び出しておきます。また、ループの中で[`std::thread::sleep`](https://doc.rust-lang.org/std/thread/fn.sleep.html)を呼び出します。とりあえず1件ごとに100ミリ秒スリープすることにしましょう。

```rust:src/main.rs
use reqwest::blocking::ClientBuilder;
use url::Url;
use mini_crawler::LinkExtractor;
use mini_crawler::crawler::Crawler;
use std::time::Duration;

fn main() -> eyre::Result<()> {
    env_logger::init();

    let url = std::env::args()
        .nth(1)
        .unwrap_or("https://www.rust-lang.org".to_owned());
    let url = Url::parse(&url)?;
    let client = ClientBuilder::new()
        .build()?;
    let extractor = LinkExtractor::from_client(client);

    let crawler = Crawler::new(&extractor, url);

    let wait = Duration::from_millis(100);

    for url in crawler.take(10) {
        println!("{}", url);
        std::thread::sleep(wait.clone());
    }

    Ok(())
}
```

さて、`cargo run`を実行して`mini-crawler`を動かしてみましょう。訪問したURLが10件表示されると思います。しかし、スリープさせた時間である100ミリ秒より長い間隔で1件ずつ表示されるでしょう。これはHTMLの解析に時間がかかるのと、それぞれのページを実際に読み込んでリンクの抽出までやっているからです。

この動作を早くしたければリンクの抽出を後回しにするように`Crawler`を作り替えることができるでしょう。しかし、そうすると各ページが持つリンクも取り出したくなったときに、かなり面倒なことになるでしょうからそのような修正はしないことにします。

## CLIの作成

さて、コマンドライン引数から探索を始めるページのURLと訪問するページ数の上限を渡せるようにしましょう。

コマンドライン引数をパースするために[`structopt`](https://crates.io/crates/structopt)を使います。`Cargo.toml`に以下の行を加えましょう。

```toml:Cargo.toml
structopt = "0.3"
```

`structopt`は手続きマクロによってコマンドライン引数を構造体に自動で変換してくれる非常に便利なクレートです。さっそく、オプションを格納する`struct Opt`を定義して、`derive`アトリビュートを付けて`structopt`を使うことを宣言しましょう。

```rust:src/main.rs
use structopt::StructOpt;

#[derive(StructOpt)]
struct Opt {
}
```

必要なオプションは、
- 探索を始めるページのURL
- 上限ページ数
の二つでした。それぞれ、`Url`構造体、`usize`型で持つことにしましょう。

```rust:src/main.rs
#[derive(StructOpt)]
struct Opt {
    maximum_pages: usize,
    start_page: Url,
}
```

`structopt`への指示は構造体の各フィールドに属性を付けることで行います。上限ページ数は`-n`オプションで指定したいので`maximum_pages`に`#[structopt(short="n")]`を付けます。

```rust:src/main.rs
#[derive(StructOpt)]
struct Opt {
    #[structopt(short="n")]
    maximum_pages: usize,
    start_page: Url,
}
```

属性を何も付けなければ、位置引数として扱われオプションを付けずに指定できる引数となります。また、文字列からフィールドの型への変換は、`FromStr`トレイトが実装されていれば自動で行われます。

コマンドライン引数は`struct Opt`に自動で実装される`from_args`メソッドを呼び出すことでパースできます。

```rust:src/main.rs
    let opt = Opt::from_args();
```

さて、コマンドライン引数を使うように`main`関数を書き換えましょう。

```rust:src/main.rs
fn main() -> eyre::Result<()> {
    env_logger::init();

    let opt = Opt::from_args();

    let client = ClientBuilder::new()
        .build()?;
    let extractor = LinkExtractor::from_client(client);

    let crawler = Crawler::new(&extractor, opt.start_page);

    let wait = Duration::from_millis(100);

    for url in crawler.take(opt.maximum_pages) {
        println!("{}", url);
        std::thread::sleep(wait.clone());
    }

    Ok(())
}
```

`cargo run`を実行すると以下のメッセージが表示されます。

```
error: The following required arguments were not provided:
    <start-page>
    -n <maximum-pages>

USAGE:
    mini-crawler <start-page> -n <maximum-pages>

For more information try --help
```

`cargo run -- --help`を実行してみましょう。

```
mini-crawler 0.1.0

USAGE:
    mini-crawler <start-page> -n <maximum-pages>

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

OPTIONS:
    -n <maximum-pages>

ARGS:
    <start-page>
```

ヘルプメッセージが表示されていますが説明がついていません。ドキュメントコメントを書くことで引数の説明が生成されるようになります。

```rust:src/main.rs
/// A toy web crawler
#[derive(StructOpt)]
struct Opt {
    /// Maximum number of pages to be crawled
    #[structopt(short="n")]
    maximum_pages: usize,
    /// URL where this program starts crawling
    start_page: Url,
}
```

ドキュメントコメントを書いてから`cargo run -- --help`を実行するとメッセージが次のように変化します。これで引数の説明が表示されるようになりました。

```
mini-crawler 0.1.0
A toy web crawler

USAGE:
    mini-crawler <start-page> -n <maximum-pages>

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

OPTIONS:
    -n <maximum-pages>        Maximum number of pages to be crawled

ARGS:
    <start-page>    URL where this program starts crawling
```

それでは`mini-crawler`を実行してみましょう。

```
% RUST_LOG=info cargo run -- -n 100 https://docs.rs/reqwest/0.10.9/reqwest/
    Finished dev [unoptimized + debuginfo] target(s) in 0.10s
     Running `target/debug/mini-crawler -n 100 'https://docs.rs/reqwest/0.10.9/reqwest/'`
[2020-12-04T07:23:18Z INFO  mini_crawler] GET "https://docs.rs/reqwest/0.10.9/reqwest/"
[2020-12-04T07:23:19Z INFO  mini_crawler] Retrieved 200 OK "https://docs.rs/reqwest/0.10.9/reqwest/"
https://docs.rs/reqwest/0.10.9/reqwest/
[2020-12-04T07:23:19Z INFO  mini_crawler] GET "https://docs.rs/"
[2020-12-04T07:23:19Z INFO  mini_crawler] Retrieved 200 OK "https://docs.rs/"
https://docs.rs/
[2020-12-04T07:23:20Z INFO  mini_crawler] GET "https://docs.rs/about"
[2020-12-04T07:23:20Z INFO  mini_crawler] Retrieved 200 OK "https://docs.rs/about"
https://docs.rs/about
[2020-12-04T07:23:20Z INFO  mini_crawler] GET "https://docs.rs/about/badges"
[2020-12-04T07:23:20Z INFO  mini_crawler] Retrieved 200 OK "https://docs.rs/about/badges"
https://docs.rs/about/badges
[2020-12-04T07:23:20Z INFO  mini_crawler] GET "https://docs.rs/about/builds"
[2020-12-04T07:23:20Z INFO  mini_crawler] Retrieved 200 OK "https://docs.rs/about/builds"
https://docs.rs/about/builds
-- snip --
[2020-12-04T07:23:25Z INFO  mini_crawler] GET "https://rust-lang-nursery.github.io/rust-cookbook/"
[2020-12-04T07:23:25Z INFO  mini_crawler] Retrieved 200 OK "https://rust-lang-nursery.github.io/rust-cookbook/"
https://rust-lang-nursery.github.io/rust-cookbook/
[2020-12-04T07:23:26Z INFO  mini_crawler] GET "https://crates.io/"
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: ServerError(reqwest::Error { kind: Status(403), url: Url { scheme: "https", host: Some(Domain("crates.io")), port: None, path: "/", query: None, fragment: None } })', src/lib.rs:79:14
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

おっと、`unwrap`の呼び出しでパニックしてしまいましたね。この`unwrap`は`LinkExtractor`に`AdjacentNodes`を実装する際に書いたものです。最後にこの部分の処理を修正して`mini-crawler`を完成させましょう。

## 最後の修正

今回はリンクの抽出でエラーが起きた場合は、エラーを無視して空の`Vec`を返してしまうようにします。ただし、発生したエラーはログに残すようにしておきます。エラーログはエラーの発生原因も表示するようにしました。

```rust:src/lib.rs
impl crawler::AdjacentNodes for LinkExtractor {
    type Node = Url;

    fn adjacent_nodes(&self, v: &Self::Node) -> Vec<Self::Node> {
        match self.get_links(v.clone()) {
            Ok(links) => links,
            Err(e) => {
                use std::error::Error;
                log::warn!("Error occurred: {}", e);

                let mut e = e.source();
                loop {
                    if let Some(err) = e {
                        log::warn!("Error source: {}", err);
                        e = err.source();
                    } else {
                        break;
                    }
                }

                vec![]
            },
        }
    }
}
```

これで
```
% RUST_LOG=warn cargo run --release -- -n 200 https://docs.rs/reqwest/0.10.9/reqwest/
```
を実行すれば訪問したURLが表示されることでしょう。

以上の例では`reqwest`のドキュメントのURLを`mini-crawler`に与えていましたが、`cargo doc`が生成するドキュメントは意外にたくさんのリンクを含んでいるので、例えば[`mdbook`](https://rust-lang.github.io/mdBook/)のマニュアルはそれほど大きくないので、こちらの方が例としてよかったかもしれません。
```
% cargo run --release -- -n 100 https://rust-lang.github.io/mdBook/format/summary.html
```

以上で`mini-crawler`は完成です。

## この章で作成したファイル

- [`Cargo.toml`](https://github.com/ShotaroTsuji/mini-crawler/blob/v0.7.0/Cargo.toml)
- [`src/main.rs`](https://github.com/ShotaroTsuji/mini-crawler/blob/v0.7.0/src/main.rs)
- [`src/lib.rs`](https://github.com/ShotaroTsuji/mini-crawler/blob/v0.7.0/src/lib.rs)
- [`src/crawler.rs`](https://github.com/ShotaroTsuji/mini-crawler/blob/v0.7.0/src/crawler.rs)

