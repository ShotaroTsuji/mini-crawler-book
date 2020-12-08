# おわりに

最後に今回やったことをまとめておきましょう。

- [`reqwest`](https://crates.io/crates/reqwest)クレートを使って、HTTPサーバにGETリクエストを送信してウェブページを取得しました。レスポンスからボディ、ステータスコードやリダイレクト後のURLを取り出しました。
- [`select`](https://crates.io/crates/select)クレートを使って、HTML文書から`<a>`要素を抜き出し、その`href`属性の値を取り出しました。
- [`url`](https://crates.io/crates/url)クレートを使ってURLをパースしました。パースしたURLが相対URLだった場合にはリダイレクト後のURLをベースにして絶対URLに変換しました。
- [`log`](htpps://crates.io/crates/log)クレートを使って、ログを記録するようにしました。ログの表示には[`env_logger`](https://crates.io/crates/env_logger)クレートを使いました。
- [`thiserror`](https://crates.io/crates/thiserror)クレートを使って独自のエラー構造体を実装しました。呼び出し側でのエラー処理には[`eyre`](https://crates.io/crates/eyre)を使いました。
- 幅優先探索をイテレータとして実装しました。
- [`structopt`](https://crates.io/crates/structopt)クレートを使ってコマンドライン引数のパース機能を作りました。

以上のステップを踏んで、最低限の機能をもったウェブクローラを作ることができました。この冊子で知ったことを読者のみなさんが作るプログラムに役に立つことを願います。また、`mini-carwler`にさらに機能を付け加えてみたい人は、どうぞご自由に改造してみてください。完成版のソースコードは https://github.com/ShotaroTsuji/mini-crawler にMIT Licenseで公開しています。

## 補足

コマンドラインプログラムを作成するために使用するクレートについて少し補足をしておきます。

手続きマクロを使ってコマンドライン引数を処理するクレートは現在のところ`structopt`を使うのがベストだと思いますが、将来的には[`clap`](https://crates.io/crates/clap)に同等の機能が実装されるようです[^1]。

ロギングに関しては、シングルスレッドのプログラムを実装するのであれば`log`クレートで十分なように思います。しかし、マルチスレッドや非同期のプログラムに対しては不十分でしょう[^2]。より高機能なクレートとして[`slog`](https://crates.io/crates/slog)がありますが、筆者は使ったことがないのでこれに対してコメントはできません。

この冊子で作ったプログラムを実際に動かした方の中には、`env_logger`が色付けして文字を表示していることが気になった方もいるでしょう。ターミナルにエスケープシーケンスを使って文字を色付けするためのクレートとして、[`ansi_term`](https://crates.io/crates/ansi_term)があります。また、出力がターミナルか否かを判別するには[`atty`](https://crates.io/crates/atty)を使います。

エラー処理に関しては、ライブラリでは[`thiserror`](https://crates.io/crates/thiserror)を使い、アプリケーションでは[`anyhow`](https://crates.io/crates/anyhow)を使うのが標準的なようです。ただし、今回は`anyhow`の代わりにそのフォークである[`eyre`](https://crates.io/crates/eyre)を使いました。これは筆者が普段、エラー表示のために[`color-eyre`](https://crates.io/crates/color-eyre)を使っていて、`eyre`の方に慣れているからです。

## 参考文献

- [Rust Cookbook - Extract all links from a webpage HTML](https://rust-lang-nursery.github.io/rust-cookbook/web/scraping.html#extract-all-links-from-a-webpage-html)

  リンクの抽出はRust Cookbookに掲載されているサンプルを参考にコードを書きました。
- [データ構造とアルゴリズム](https://www.kyoritsu-pub.co.jp/kenpon/bookDetail/9784320120341)

  幅優先探索は、杉原厚吉「データ構造とアルゴリズム」（共立出版、2001年）を参考に記述しました。

[^1]: https://qiita.com/watawuwu/items/a6cbcd92dfb5336b9a01#%E5%BC%95%E6%95%B0%E5%87%A6%E7%90%86c

[^2]: `log`クレートにおいて排他処理は行われています。ただし`log::debug!`などのマクロを呼ぶたびに`env_logger`などのクレートが登録したロガー（[`Log`](https://docs.rs/log/0.4.11/log/trait.Log.html)トレイトを実装したオブジェクト）が呼び出されます。`env_logger`のロガーは呼び出されるとそのスレッドで印字処理が行われるようなので、おそらくログを記録するたびにブロックしてしまいます。また、非同期プログラミングにおいては、どのタスクを実行している時のログなのかも記録しないと、後から解析するのが困難になります。
