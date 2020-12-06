# リンクを抽出する

このチャプターでは、サーバから取得したHTMLのリンクを抽出するコードを書きます。HTMLを解析して`<a>`要素の`href`属性を抜き出すことでリンクの抽出を実装します。この操作を実現するために今回は[`select`](https://docs.rs/select/0.5.0/select/index.html)クレートを使います。

`Cargo.toml`に`select`クレートへの依存を追記しておきます。

```toml:Cargo.toml
[dependencies]
reqwest = { version = "0.10", features = ["blocking"] }
eyre = "0.6"
select = "0.5"
```

残念ながら`select`クレートにはドキュメントが書かれていません。いろいろなクレートを探していると、ドキュメントが整備されていないクレートに遭遇することが時々あります。しかし、めげずに`cargo doc`が生成したドキュメントをたどって使い方を確認しましょう。

## HTML文書の読み込み

ドキュメントのモジュール一覧をみると、`document`という名前のモジュールがあります。名前から察するにHTML文書を扱うモジュールなのでしょう。さて、[`document`](https://docs.rs/select/0.5.0/select/document/index.html)モジュールで定義されている構造体を見ると、`Document`という名前の構造体があり、"An HTML document" という説明があります。名前からしてこの構造体がHTML文書を保持するのでしょう。

次に[`Document`](https://docs.rs/select/0.5.0/select/document/struct.Document.html)構造体の実装一覧を参照します。`from_read`という`Read`トレイトを実装したオブジェクトを受け取って`Document`構造体を返す関連関数がありますが、今回はこのメソッドは使いません。その下のトレイト実装の節に`From<&'a str>`トレイトの実装があると書かれています。この関連関数の説明を読むと、"Parses the given `&str` into a `Document`" とあります。前のチャプターで書いたコードでは取得したページの内容は`String`のオブジェクトとして保持されるので、今回は`From`トレイトの実装を使うことにします。したがって、以下のコードで`Document`構造体のオブジェクトを作ることができます。

```rust
    use select::document::Document;
    let doc = Document::from(body.as_str());
```

## `<a>`要素の抽出

さて次にやるべきことは、読み込んだHTML文書から`<a>`要素を抜き出すことです。`Document`構造体に実装された`find`メソッドを呼び出せば特定の要素だけを抽出することができそうです。`find`メソッドの宣言は以下のようになっています。

```rust
pub fn find<P: Predicate>(&self, predicate: P) -> Find<P>
```

`find`メソッドに`Predicate`トレイトを実装したオブジェクトを渡すと`Find`構造体のオブジェクトが返ってきます。[`Predicate`トレイトを実装する型](https://docs.rs/select/0.5.0/select/predicate/trait.Predicate.html#implementors)を確認しましょう。

[`Element`](https://docs.rs/select/0.5.0/select/predicate/struct.Element.html)構造体は使えそうでしょうか。説明を読むと、どんな要素ノードにもマッチすると書いてあるので、これは我々が求めているものではないようです。

[`Name`](https://docs.rs/select/0.5.0/select/predicate/struct.Name.html)構造体はどうでしょうか。名前`T`を持つ要素ノードにマッチするという説明があるのでこれが使えそうです。この構造体の宣言は`pub struct Name<T>(pub T);`となっているので、`Name("a")`とすれば`<a>`要素を抽出できそうです。ちなみに、`Name`は`T`が`&str`のときにだけ[`Predicate`を実装します](https://docs.rs/select/0.5.0/select/predicate/struct.Name.html#impl-Predicate)。

`find`メソッドの戻り値を確認しておきましょう。`find`の戻り値は[`Find`](https://docs.rs/select/0.5.0/select/document/struct.Find.html)構造体でした。トレイト実装の欄を見ると、`Node`構造体を返す`Iterator`トレイトを実装しているようです。この[`Node`](https://docs.rs/select/0.5.0/select/node/struct.Node.html)構造体はHTMLの要素やテキストなどを表すようです。

以上のことから、`Document`のオブジェクトの`find`メソッドを呼んで、`for`文を使えば`<a>`要素だけを抜き出せそうです。

```rust:src/main.rs
    use select::predicate::Name;
    for a in doc.find(Name("a")) {
        println!("{:?}", a);
    }
```

上のコードを`src/main.rs`に追加して実行してみると以下のような出力が得られます。どうやらうまく行っているようです。

```
Element { name: "a", attrs: [("href", "/"), ("class", "brand")], children: [Text("\n      "), Element { name: "img", attrs: [("class", "v-mid ml0-l"), ("alt", "Rust Logo"), ("src", "/static/images/rust-logo-blk.svg")], children: [] }, Text("\n      \n    ")] }
Element { name: "a", attrs: [("href", "/tools/install")], children: [Text("Install")] }
Element { name: "a", attrs: [("href", "/learn")], children: [Text("Learn")] }
Element { name: "a", attrs: [("href", "https://play.rust-lang.org/")], children: [Text("Playground")] }
Element { name: "a", attrs: [("href", "/tools")], children: [Text("Tools")] }
-- snip --
```

## `href`属性の取得

最後に`a`要素から`href`属性を取り出すことで、リンクのURLを得ることができます。[`Node`](https://docs.rs/select/0.5.0/select/node/struct.Node.html)構造体には`attr`というメソッドがあります。

```
pub fn attr(&self, name: &str) -> Option<&'a str>
```

このメソッドは`name`という名前を持つ属性があればその値を、なければ`None`を返します。`Document`構造体の`find`メソッドはイテレータを返すので、[`filter_map`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter_map)を使えば`href`属性の値だけを取り出すことができます。

```rust:src/main.rs
    for href in doc.find(Name("a")).filter_map(|a| a.attr("href")) {
        println!("{:?}", href);
    }
```

先ほど追加したコードを上のコードに置き換えてプログラムを実行すれば、以下のような結果が得られるはずです。

```
"/"
"/tools/install"
"/learn"
"https://play.rust-lang.org/"
"/tools"
"/governance"
"/community"
"https://blog.rust-lang.org/"
"/learn/get-started"
-- snip --
```

## リンクを絶対URLに変換する

ここまでの作業でHTML文書からリンクを抽出することができました。
しかし、抽出したリンクを`reqwest::blocking::get`に渡すことはできません。
なぜなら同じサイト内にあるページへのリンクは相対URLとして記述されているので、これを絶対URLに変換する必要があります。

今回はURLの扱いを簡単に行うために[`url`クレート](https://crates.io/crates/url)を使います。
[`url`クレートのドキュメント](https://docs.rs/url/2.2.0/url/)を読むと、`Url::parse`メソッドを使うとURLのパースができるようなので、ひとまず抽出したリンクをパースして表示するコードを書いてみます。

```rust
    for href in doc.find(Name("a")).filter_map(|a| a.attr("href")) {
        println!("{:?}", Url::parse(href));
    }
```

`src/main.rs`の`for`文を上のように書き換えて実行すると次のような出力が得られるはずです。
絶対URLになっているものはパースが成功して、相対URLになっているものは[`ParseError::RelativeUrlWithoutBase`](https://docs.rs/url/2.2.0/url/enum.ParseError.html)が返ってきます。

```
Err(RelativeUrlWithoutBase)
Err(RelativeUrlWithoutBase)
Err(RelativeUrlWithoutBase)
Ok(Url { scheme: "https", host: Some(Domain("play.rust-lang.org")), port: None, path: "/", query: None, fragment: None })
Err(RelativeUrlWithoutBase)
Err(RelativeUrlWithoutBase)
Err(RelativeUrlWithoutBase)
Ok(Url { scheme: "https", host: Some(Domain("blog.rust-lang.org")), port: None, path: "/", query: None, fragment: None })
Err(RelativeUrlWithoutBase)
Ok(Url { scheme: "https", host: Some(Domain("blog.rust-lang.org")), port: None, path: "/2020/11/19/Rust-1.48.html", query: None, fragment: None })
-- snip --
```

URLをパースした結果に応じて以下の通りに処理を分けましょう。

- 絶対URLだった場合はそのまま表示する。
- 相対URLだった場合は絶対URLに変換して表示する。
- それ以外の場合は無視する。

`for`文の中には次のような`match`文を書くことになります。

```
	use url::ParseError as UrlParseError;
        match Url::parse(href) {
            Ok(url) => { println!("{}", url); },
            Err(UrlParseError::RelativeUrlWithoutBase) => {
	        // `href`を絶対URLに変換する。
            },
            Err(e) => {},
        }
```

相対URLを絶対URLに変換するのに[`join`メソッド](https://docs.rs/url/2.2.0/url/struct.Url.html#method.join)が使えます。
`join`メソッドは`self`をベースURLとして`input`を結合したURLを返します。
ここでベースURLとして最初に`reqwest::blocking::get`に渡したURLを使いたくなりますが、リダイレクトが発生する可能性に注意しなければなりません。
リダイレクト後のURLは`reqwest`クレートの[`Response`構造体の`url`メソッド](https://docs.rs/reqwest/0.10.9/reqwest/blocking/struct.Response.html#method.url)で取得することができます。

まずはレスポンスからリダイレクト後のURLを取り出すコードを書きます。
レスポンスのボディを取り出す[`text`メソッド](https://docs.rs/reqwest/0.10.9/reqwest/blocking/struct.Response.html#method.text)が`Response`オブジェクトを消費してしまうことに注意します。
ボディを取り出す前にURLを取り出してクローンしておきましょう。

```rust
    let response = reqwest::blocking::get("https://www.rust-lang.org")?;
    let base_url = response.url().clone();
    let body = response.text()?;
    let doc = Document::from(body.as_str());
```

`<a>`要素を走査するループの中では相対URLをベースURLに`join`して表示します。

```rust
    for href in doc.find(Name("a")).filter_map(|a| a.attr("href")) {
        match Url::parse(href) {
            Ok(url) => { println!("{}", url); },
            Err(UrlParseError::RelativeUrlWithoutBase) => {
                let url = base_url.join(href)?;
                println!("{}", url);
            },
            Err(e) => { println!("Error: {}", e); },
        }
    }
```

これで取得したウェブページから抽出したリンクを絶対URLで表示できるようになりました。
