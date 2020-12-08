# 幅優先探索を実装する

前の章で紹介した幅優先探索をRustで実装していきます。

隣接する頂点を返す関数をトレイトとして定義します。こうすることで探索対象がグラフとしての性質を満たすものであればなんでも探索できるようになります。テスト用のデータを入力することができるようになるので、アルゴリズムの実装が正しいかを試すことができるようになります。

アルゴリズム自体は訪問した頂点を返すイテレータとして実装したいと思います。`mini-crawler`の実装ではウェブページを訪問する際に待機時間を挟みたいので、一つの頂点を訪問したら呼び出し側のコードに制御を戻したいのです。また、イテレータにすることで訪問する頂点の個数を制御できます。

幅優先探索の実装は`src/crawler.rs`に書くことにします。このファイルをコンパイルの対象にするために、`src/lib.rs`に
```rust:src/lib.rs
pub mod crawler;
```
を書き加えておきましょう。

## `trait AdjacentNodes`

頂点`v`に隣接する頂点を返す関数`adjacent_nodes`を定義します。頂点の型はトレイトの関連型としておきます。関連型はあとで具体的な実装を作るときに指定する必要があります。

```rust:src/crawler.rs
pub trait AdjacentNodes {
    type Node;

    fn adjacent_nodes(&self, v: &Self::Node) -> Vec<Self::Node>;
}
```

`src/crawler.rs`の`test`モジュールに隣接する頂点を`Vec`で保持する構造体`AdjVec`を作っておきます。`AdjVec`は`usize`の値を頂点として、`i`番目の要素に頂点`i`に隣接する頂点を格納した`Vec`を保持します。

この`AdjVec`に`AdjacentNodes`トレイトを実装します。関数`adjacent_nodes`では`Vec`の`get`メソッドで隣接する頂点を取り出します。この戻り値は`Option<&Vec<usize>>`になるので`cloned`メソッドを呼んでコピーします。最後に中身が`None`だった場合は空の`Vec`を返すために`unwrap_or`を呼びます。

```rust:src/crawler.rs
#[cfg(test)]
mod test {
    use super::*;

    struct AdjVec(Vec<Vec<usize>>);

    impl AdjacentNodes for AdjVec {
        type Node = usize;

        fn adjacent_nodes(&self, v: &Self::Node) -> Vec<Self::Node> {
            self.0.get(*v)
                .cloned()
                .unwrap_or(Vec::new())
        }
    }
}
```

## `struct Crawler`

さて、幅優先探索を実装する構造体を設計しましょう。名前は`Crawler`としましょう。コンストラクタは`AdjacentNodes`トレイトを実装したオブジェクトと頂点を受け取るようにします。そして`Crawler`には`Iterator`トレイトを実装します。

使い方としては以下のコードのような感じでしょうか。下のコードは今はコンパイルできませんが、このコードが動くようにプログラムを書いていきましょう。

```rust
    fn test_bfs() {
        let graph = AdjVec(vec![
            vec![1, 2],
            vec![0, 3],
            vec![3],
            vec![2, 0]
        ]);

	let bfs = Crawler::new(&graph, 0);
	let nodes: Vec<usize> = bfs.iter().collect();
	assert_eq!(nodes, vec![0, 1, 2, 3]);
    }
```

## 幅優先探索の実装

それでは幅優先探索を実装していきましょう。まずは保持しなければならないデータを考えます。前の章で紹介したアルゴリズムを再度載せます。ただし、訪問するページ数の上限に関する記述は削除します。なぜならイテレータとして実装するので呼び出し側から制御できるからです。

- 入力
  - 頂点に対して隣接する頂点の集合を返す関数`T`
  - 探索を始める頂点`a`
- 出力
  - 訪問したページのリスト
- 手順
  1. キュー`Visit`の末尾に`a`を挿入する。
  2. キュー`Visit`が空ならばハッシュ`Visited`を出力して終了する。
  3. キュー`Visit`の先頭から頂点`v`を取り出す。
  4. 頂点`v`がハッシュ`Visited`に含まれていれば、手順2にジャンプする。
  5. `T(v)`の頂点のうち、ハッシュ`Visited`に含まれないものをキュー`Visit`の末尾に挿入する。
  6. `v`をハッシュ`Visited`に挿入する。
  7. 手順2にジャンプする。

頂点を取り出すグラフは`AdjacentNodes`トレイトを実装したオブジェクトへの参照を持つことにします。なので型パラメータに参照のライフタイムが必要になります。

幅優先探索の実装に必要なデータ構造は`Visit`に使うキューと`Visited`に使うハッシュでした。それぞれ[`VecDeque`](https://doc.rust-lang.org/std/collections/struct.VecDeque.html)と[`HashSet`](https://doc.rust-lang.org/std/collections/struct.HashSet.html)が使えます。したがって、`Crawler`の宣言は次のようになります。`VecDeque`と`HashSet`のパラメタには`AdjacentNodes::Node`を入れる必要があるので、`G`にトレイト制約を付けています。

```rust:src/crawler.rs
pub struct Crawler<'a, G: AdjacentNodes> {
    graph: &'a G,
    visit: VecDeque<<G as AdjacentNodes>::Node>,
    visited: HashSet<<G as AdjacentNodes>::Node>,
}
```

次にコンストラクタを作りましょう。受け取った頂点を`visit`に挿入するのを忘れないようにしておきましょう。

`<G as AdjacentNodes>::Node`に複数のトレイト制約を付けました。`visit`から取り出した頂点を`visited`に挿入してイテレータから返すために、`Clone`トレイトが必要になります。なぜなら、`HashSet`は保持するデータを所有する必要があり、戻り値も所有権が呼び出し側に移るため、取り出した頂点のコピーが必要になるからです。また、`HashSet`のメソッドを呼び出すために、[`Hash`](https://doc.rust-lang.org/std/hash/trait.Hash.html)トレイト、`Eq`トレイト、[`Borrow`](https://doc.rust-lang.org/std/borrow/trait.Borrow.html)トレイトが必要になります。

```rust:src/crawler.rs
impl<'a, G> Crawler<'a, G>
where
    G: AdjacentNodes,
    <G as AdjacentNodes>::Node: Clone + Hash + Eq + Borrow<<G as AdjacentNodes>::Node>,
{
    pub fn new(graph: &'a G, start: <G as AdjacentNodes>::Node) -> Self {
        let mut visit = VecDeque::new();
        let visited = HashSet::new();

        visit.push_back(start);

        Self {
            graph: graph,
            visit: visit,
            visited: visited,
        }
    }
}
```

それでは`Iterator`トレイトの実装に取り掛かりましょう。

まずは、必要な宣言を書いてしまいます。そして、`next`メソッドの中には、キュー`visit`の要素を取り出し続けるループを書きます。ループが終わった後には`None`を返すようにします。

`VecDeque`から要素を取り出すには[`pop_front`](https://doc.rust-lang.org/std/collections/struct.VecDeque.html#method.pop_front)メソッドを呼び出します。このメソッドは`Option<T>`型のオブジェクトを返すので、`while let`を使うことでキューが空になるまでループすることができます。

ここでループが必要になるのは`visit`から取り出した頂点がすでに`visited`に含まれていた場合に、隣接する頂点を`visit`に挿入する操作をスキップする必要があるからです。ループを抜けたら`visit`は空になっているのでもう訪れるべき頂点はありません。なので`None`を返します。

```rust:src/crawler.rs
impl<'a, G> Iterator for Crawler<'a, G>
where
    G: AdjacentNodes,
    <G as AdjacentNodes>::Node: Clone + Hash + Eq + Borrow<<G as AdjacentNodes>::Node>,
{
    type Item = <G as AdjacentNodes>::Node;

    fn next(&mut self) -> Option<Self::Item> {
        while let Some(v) = self.visit.pop_front() {
        }

        None
    }
}
```

`visit`から取り出した頂点が訪問済みであれば処理をスキップするコードを書きます。`HashSet`構造体の[`contains`](https://doc.rust-lang.org/std/collections/struct.HashSet.html#method.contains)メソッドを呼び出すことで、ハッシュに要素が含まれるかどうかを調べることができます。ここでは、`self.visited.contains(&v)`を呼び出すことで、`visit`から取り出した頂点`v`が`visited`に含まれるかどうかを調べます。そして、含まれていた場合は`continue`で後の処理をスキップします。

```rust:src/crawler.rs
impl<'a, G> Iterator for Crawler<'a, G>
where
    G: AdjacentNodes,
    <G as AdjacentNodes>::Node: Clone + Hash + Eq + Borrow<<G as AdjacentNodes>::Node>,
{
    type Item = <G as AdjacentNodes>::Node;

    fn next(&mut self) -> Option<Self::Item> {
        while let Some(v) = self.visit.pop_front() {
            if self.visited.contains(&v) {
                continue;
            }
        }

        None
    }
}
```

次に、頂点`v`に隣接する頂点を取得してキュー`visit`に挿入します。まず、`self.graph`の`adjacent_nodes`メソッドを呼び出して隣接する頂点を取得します。このメソッドの戻り値は頂点の`Vec`でした。

そして、隣接する頂点の`Vec`の各要素を`into_iter`メソッドで取り出していきます。隣接する頂点をキュー`visit`に挿入したいので所有権を奪う`into_iter`を使いました。隣接する頂点`u`が`visited`に含まれていなければ`visit`に挿入します。

```rust:src/crawler.rs
impl<'a, G> Iterator for Crawler<'a, G>
where
    G: AdjacentNodes,
    <G as AdjacentNodes>::Node: Clone + Hash + Eq + Borrow<<G as AdjacentNodes>::Node>,
{
    type Item = <G as AdjacentNodes>::Node;

    fn next(&mut self) -> Option<Self::Item> {
        while let Some(v) = self.visit.pop_front() {
            if self.visited.contains(&v) {
                continue;
            }

            let adj = self.graph.adjacent_nodes(&v);
            for u in adj.into_iter() {
                if !self.visited.contains(&u) {
                    self.visit.push_back(u);
                }
            }
        }

        None
    }
}
```

あとは頂点`v`のコピーを`visited`に挿入してから`return`で`Some(v)`を返せば、イテレータの実装は完了です。

```rust:src/crawler.rs
impl<'a, G> Iterator for Crawler<'a, G>
where
    G: AdjacentNodes,
    <G as AdjacentNodes>::Node: Clone + Hash + Eq + Borrow<<G as AdjacentNodes>::Node>,
{
    type Item = <G as AdjacentNodes>::Node;

    fn next(&mut self) -> Option<Self::Item> {
        while let Some(v) = self.visit.pop_front() {
            if self.visited.contains(&v) {
                continue;
            }

            let adj = self.graph.adjacent_nodes(&v);
            for u in adj.into_iter() {
                if !self.visited.contains(&u) {
                    self.visit.push_back(u);
                }
            }

            self.visited.insert(v.clone());

            return Some(v);
        }

        None
    }
}
```

## テスト

最後に、`test`モジュールにテストコードを書いて`cargo test`を実行してみましょう。テストに通ることが確認できるはずです。これで`Crawler`は完成です。

```rust:src/crawler.rs
    #[test]
    fn bfs() {
        let graph = AdjVec(vec![
            vec![1, 2],
            vec![0, 3],
            vec![3],
            vec![2, 0]
        ]);

        let crawler = Crawler::new(&graph, 0);
        let nodes: Vec<usize> = crawler.collect();

        assert_eq!(nodes, vec![0, 1, 2, 3]);
    }
```

## この章で作成したファイル

- [`Cargo.toml`](https://github.com/ShotaroTsuji/mini-crawler/blob/v0.6.0/Cargo.toml)
- [`src/main.rs`](https://github.com/ShotaroTsuji/mini-crawler/blob/v0.6.0/src/main.rs)
- [`src/lib.rs`](https://github.com/ShotaroTsuji/mini-crawler/blob/v0.6.0/src/lib.rs)
- [`src/crawler.rs`](https://github.com/ShotaroTsuji/mini-crawler/blob/v0.6.0/src/crawler.rs)

## 演習

1. 次の隣接関係を持つグラフに、頂点`0`をスタートとして幅優先探索を適用すると、`0, 1, 2, 4, 3`の順で訪問することを手計算と`Crawler`で確かめてみてください。

    | 頂点   | 隣接する頂点 |
    |--------|--------------|
    |   0    |  1           |
    |   1    |  0, 2, 4     |
    |   2    |  0, 3        |
    |   3    |  0           |
    |   4    |  0           |
