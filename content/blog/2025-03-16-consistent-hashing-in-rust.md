+++
title = "Consistent Hashing を Rust で実装してみる"
slug = "consistent-hashing-in-rust"
date = 2025-03-16

[taxonomies]
categories = ["記事"]
tags = ["rust", "algorithm", "hash"]

[extra]
lang = "jp"
toc = true
comment = false
copy = true
math = true
mermaid = true
outdate_alert = false
outdate_alert_days = 120
display_tags = true
truncate_summary = false
featured = false
reaction = false
cover = "assets/cover.png"
+++

[大規模データセットのためのアルゴリズムとデータ構造](https://book.mynavi.jp/ec/products/detail/id=143918) という本を読んでいる。
この本では表題にもあるように大規模データセットで使えるアルゴリズムやデータ構造を紹介している。
その序盤に、分散システムで良く用いられるコンシステントハッシュが紹介されていた。
C/C++ で実装したことはあるが、練習がてら Rust で実装してみようと思う。

なお、私の Rust は初心者レベルなので悪しからず。
[The Rust Programming Language 日本語版](https://doc.rust-jp.rs/book-ja/) を斜め読みした程度である。

# コンシステントハッシュとは

[コンシステントハッシュ法(Consistent Hashing)](https://en.wikipedia.org/wiki/Consistent_hashing) はハッシュテーブルを分散管理するための手法と言えるだろう。
よく使われるハッシュテーブルは $2^N$ で拡張するような単一のテーブルを意味するが、コンシステントハッシュ法で使われるハッシュテーブルは複数のハッシュテーブルから構成される。

コンシステントハッシュ法ではハッシュ空間を分割し、これらをノードと呼ばれるサーバーに分割する。
ハッシュ空間は $0 \sim 2^N - 1$ で構成されることが多い。
これらのノードはよくハッシュリングとしてリング状に配置される。


{% mermaid() %}
flowchart LR
    A --> B;
    B --> C;
    C --> D;
    D --> A;
{% end %}

各サーバーはハッシュ空間に分散されたリソース（例えば、キーバリューの組）を管理する。
逆に言えば、各リソースはハッシュ空間内に配置されたサーバーのいずれかで管理される。

コンシステントハッシュの特徴はノードが参加・脱退するときに必要なデータ移動の少なさである。
剰余でデータ分散を行っていると、除数であるサーバー代数 `N` が変化するため、サーバーの参加・脱退が発生するたびにほぼすべてのデータの再ハッシュと配置を行う必要がある。
このような操作はネットワーク負荷を高めるだけでなく、整合性の確保が難しくなったり移行中のパフォーマンス低下を引き起こすなど様々な問題の要因となる。
したがって、ノードの参加・脱退に伴うデータ移動は最小限にしたい。
コンシステントハッシング法ではノードの参加・脱退で移動するデータ量を平均 `K / N` (`K` はキーの数、`N` はノードの数) に抑えることができる。
詳細な証明は他の記事に譲るが、 以下のようなイメージである。
- ノードの参加
    - 各ノードでは全体の約 `1 / N` のデータを担当する
    - 別のノードで担当していたデータの一部を追加されるノードに移動する
- ノードの脱退
    - 各ノードでは約 `K / N` のデータがある
    - 隣接するノードへこのデータを引き継ぐ

# Rust で実装

実装はノードをリング状に配置する（以下、Hash ring と呼ぶ）方法が一般的なので今回の実装でもノードをリング状に繋いだ実装を行う。
ちなみにリング状に配置するのはアルゴリズムを動作させる上では必須ではなく、例えばグラフ構造でハッシュテーブルを分散させることもできる（が、その分実装は複雑になる）。

## Hash ring の実装

まず初めに Hash ring に必要な操作を定義したい。
Rust では [Trait](https://doc.rust-jp.rs/book-ja/ch10-02-traits.html) と呼ばれる仕組みがあり、 Go の `interface` に似た仕組みであると理解している。

```rust
pub trait HashRingInterface<T: std::hash::Hash> {
    fn add_node(&mut self, hash: T);
    fn remove_node(&self, hash: T);
    fn lookup(&self, hash: T) -> Node;
    fn add_resource(&self, hash: T);
    fn move_resource(&self, dest: T, src: T, is_delete: bool);
}
```

今回は実装を簡易にするためにデータのハッシュ値は事前に分かっていることとする。
- `add_node` はノードの追加を行う
    - ノードを追加する際に他のノードのデータの管理を委譲される可能性がある
- `remove_node` はノードの削除を行う
    - 削除時には当該ノードが持っているデータを別のノードに移動する操作が含まれる
- `lookup` はハッシュ値がどのノードに管理されているかを返す
    - 与えられるハッシュ値を $H$ として $hash(N_{i}) \le H < hash(N_{i-1})$ を満たすノード $N_{i}$ を返す
- `add_resource` はデータを追加する
    - 実装を簡易にするためにハッシュ値をデータとしている
- `move_resource` は `src` から `dest` へデータを移動させる
    - 移動の対象となるデータが hash ring 上で `dest` に近ければ移動させる
    - `is_delete` フラグが立っている場合は近さに関係なく `src` のデータをすべて移動させる

### Node, HashRing

ノードの実装は以下の通りである。

```rust
pub struct Node<T> {
    value: T,
    resource: HashMap<T, T>,
    prev: Option<Arc<Mutex<Node<T>>>>,
    next: Option<Arc<Mutex<Node<T>>>>,
}
```

実装時に、`Node` の参照の取り扱いに困った。
他の言語で実装する際にはあまり気にしてこなかった所有権を考慮する必要があった。

`Arc` はスレッドセーフな参照カウンタ付きのスマートポインタである。
同様のスマートポインタとして `Rc` や `RefCell` がある。
マルチスレッド時に有効なスマートポインタという認識であるが、共有可能なスマートポインターを利用したかったので `Arc` を使っている
（後から分かったことだが、シングルスレッドであれば `Rc` で十分だ。ただ、特別 `Rc` する理由もなかったので `Arc` を使う）。

`Arc` はデータ競合を防ぐための仕組みであって、実際にデータにアクセスして内容を変更するためには別の仕組みが必要だ。
`Mutex` はそのための機能を提供していて、一般に Mutex と呼ばれる機構と同様の仕組みを提供しながら、データアクセスを可能にする。
例えば、ノードの追加を行う際には以下のようにノードの向き先を変更している。

```rust
let mut new_node_mut = new_node.try_lock().unwrap(); // 挿入するノードをロック、データ操作が可能になる
new_node_mut.prev = Some(Arc::clone(&prev_node_ref)); // 前方向のリンクを変更する
new_node_mut.next = Some(Arc::clone(&target)); // 後ろ方向のリンクを変更する
```

テストコードを含めた他の実装は [GitHub リポジトリ](https://github.com/donkomura/hash_bench/blob/main/src/hash_ring.rs) にアップロードしている。

## エコシステム

Rust を書いていてかなり良いと思ったのは充実したエコシステムである。

### Test

テストを書きながら開発するのに向いていると感じた。
テストは単純に関数を書けば良いので初学者が躓くであろうテストパッケージのための準備などは不要である。
実行も `cargo test` だけなので分かりやすい。

```rust
fn distance_in_ring() {
    let h = HashRing::new(5);
    assert_eq!(h.distance(0, 5), 5);
    assert_eq!(h.distance(29, 12), 15);
    assert_eq!(h.distance(5, 29), 24);
}
```

### CI

ついでに GitHub Actions で CI パイプラインを構築した。
躓くところは特になかった。
付け加えるならビルドやテストが速くて、別言語・フレームワークとの差を感じた。

```yaml
jobs:
  build:
    strategy:
      matrix:
        BUILD_TARGET: [release, dev]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: Swatinem/rust-cache@v2
        with:
          key: ${{ matrix.BUILD_TARGET}}
      - name: Check
        run: cargo check --profile ${{ matrix.BUILD_TARGET }}
      - name: Build with profiling
        run: cargo build --profile ${{ matrix.BUILD_TARGET }}
      - name: Test
        run: cargo test --profile ${{ matrix.BUILD_TARGET }} --all -- --nocapture
```

### Benchmark

ベンチマークは公式のツールチェーンとして存在しているものはないようなので、サードパーティーのクレートやツールを使う必要がある。
[Criterion](https://github.com/bheisler/criterion.rs) というツールがデファクトスタンダードとなっているようなので、こちらを利用している。

デフォルトでグラフを描画してくれるので、ある程度の結果を眺めるには向いているが実際にデータを使って分析しようとするとやや手間がかかる印象である。
実際に今回の実装についてベンチマークを取った結果はまだまとめられていない。
Rust アプリケーションのチューニングのいい練習になると思うので、別の機会に記事としてまとめようと思う。

# 感想

思い付きでやってみたが、Rust のエコシステムに助けられたおかげで思っていたよりもスムーズに実装できたと感じる。
エコシステム以外でも支援を受けていたモノがある：AI である。

前半は自力で実装し、テストや一部のコード実装は GitHub Copilot を使ってみた。
AI コーディングにはまだ慣れないが、気軽に質問できるのは実装中かなり助かった。
ChatGPT も併用していたが、やはり Cline 等の AI コーディングツールの方が効率が良いのではないかと思う。
実際に使ったことがあるのは GitHub Copilot のみなので、質と効率の両面を向上させてくれるものだと期待しつつ、次の実装の機会にでも導入したいと思う
（GitHub Copilot のみだと解決しきれない部分が度々あった）。

エコシステムとしては書かなかったがエラーメッセージが分かりやすかったのも良かった。
大半が所有権や借用の不正についてだった。
慣れてくると直すべきところの勘もついてくるので、AIに頼り切りになるようなことは無かった。

ただデバッグは結局 Printf デバッグになっていたので、次回はデバッガをうまく使ったでバッギングを行おうと思う。