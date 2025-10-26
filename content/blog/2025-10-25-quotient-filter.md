+++
title = "Quotient Filter - 高効率な確率的データ構造の実装"
slug = "quotient-filter"
date = 2025-10-25

[taxonomies]
categories = ["記事"]
tags = ["rust", "algorithm", "data-structure", "hash"]

[extra]
lang = "jp"
toc = true
comment = false
copy = true
math = true
mermaid = false
outdate_alert = false
outdate_alert_days = 120
display_tags = true
truncate_summary = false
featured = false
reaction = false
cover = "/assets/cover.png"
+++

『大規模データセットのためのアルゴリズムとデータ構造』という本を読んでいる。
第一部では確率的な手法を使った簡潔なデータ構造について紹介されている。
Quotient Filter はそのようなデータ構造のひとつであり、 Bloom Filter と同様の機能を提供している一方で全く異なる設計となっている。
仕組みをより深く理解するべく、この Quotient Filter を実装してみる。
コードは [GitHub リポジトリ](https://github.com/donkomura/hash_bench/blob/2464dbfb9336c3d583a2a73d92506016d315eaf2/src/quotient_filter.rs) で公開している。

# Quotient Filter とは

Quotient Filter (QF) は 2012 年に Bender らによって提案された確率的データ構造[^1]で、集合のメンバーシップテストを効率的に行うことができる（[Approximate membership query filter; AMQ filter](https://en.wikipedia.org/wiki/Approximate_membership_query_filter) のひとつ）。
「要素が集合に含まれているか」を高速に判定できるが、偽陽性（False Positive）が発生する可能性がある。
つまり、実際には存在しない要素を「存在する」と誤って判定することがある。

[^1]: [Don't Thrash: How to Cache Your Hash on Flash](https://vldb.org/pvldb/vol5/p1627_michaelabender_vldb2012.pdf)

## Bloom Filter との違い

既存の手法として本にも登場した Bloom Filter (BF) との違いが言及されていたので整理する。

- アクセス局所性
    - BF ではデータの書き込みはランダムアクセスとなるが、 QF ではデータアクセスが局所化している
- 機能
    - BF では削除できないが、 QF では削除・動的リサイズ・マージをサポートしている
- 性能
    - 局所化されたアクセスが可能なため、挿入やRAMの容量を越える規模での操作が BF よりも良い性能となる
    - 特にストレージにデータがある場合では連続アクセスとなる QF の方が有利

BF では複数のハッシュ関数でスロットを埋める必要があるが、 QF ではハッシュ値をもとに線形探索でスロットを埋める。
基本的にはこの設計の違いが性能に影響を与えている。
なお後述するが、 QF の方が線形探索のために使うメタデータを保持する必要があるため BF よりも多くのメモリを使用する。

# 基本的な仕組み

## ハッシュ値の分割

ハッシュ値を **Quotient**（商）と **Remainder**（余り）に分割し、 Quotient を元に線形探索を施してスロットに Remainder 入れていく。
ハッシュ値を `h` ビットとすると、上位 `q` ビットを Quotient、下位 `r` ビットを Remainder として扱う（`h = q + r`）。

```
hash(key) = [quotient (q bits)][remainder (r bits)]
```

- **Quotient**: フィルタのどのスロットに格納するかを決定するインデックス（`2^q` 個のスロット）
- **Remainder**: スロットに格納される実際の値

例えば、`q=8`, `r=4` の場合、ハッシュ値 `0b111111110000` は以下のように分割される：

```rust
quotient  = 0b11111111  // 255
remainder = 0b0000      // 0
```

## Run とクラスタリング

探索をする際には Quotient が主に使われる。
Quotient が同じものや隣接する Quotient を線形探索して目的のハッシュ値を検索する。
同じ Quotient のスロット列とこれらを合わせた列について呼称があるので実装を説明する前に紹介する。

### Run
同じ Quotient を持つ要素の集まりを **Run** と呼ぶ。
Run 内では Remainder の昇順で要素が並べられる。

### Cluster
複数の Run が連続して配置されている場合、これを **クラスター** と呼ぶ。
クラスターが形成されると、本来の位置（ホームスロット）から離れた位置に要素が配置される（Shifted）。

## メタデータ

Remainder の挿入は Quotient を元に実施される。
Quotient が衝突した際には本来挿入されるべき位置からずれて挿入されることがある。
基本的には線形探索で後続のスロットに挿入されるが、スロットがどの Run のものなのか判別がつかなくなる。
そこで以下のフラグを使って探索を支援する。

1. **Occupied (is_occupied)**: そのスロットのインデックスを Quotient として持つ要素が存在するか
2. **Continued (is_continued)**: そのスロットが Run の一部（先頭以外）であるか
3. **Shifted (is_shifted)**: そのスロットに格納された Remainder が本来の位置からシフトされているか

# 実装の概要

## 主要な操作

Quotient Filter は以下の主要な操作をサポートする：

### 挿入（Insert）

挿入操作は以下の手順で行われる：

1. キーをハッシュし、Quotient と Remainder に分割
2. Quotient に対応するスロットを確認
3. スロットが空なら直接挿入
4. スロットが使用中なら Cluster の先頭から Run の位置を走査・特定し、適切な位置に挿入

衝突が発生した場合、Run 内で Remainder の昇順を保つように要素を配置し、必要に応じて後続の要素をシフトする。
昇順に保つことで探索時に Run 全体を検索することなく、見つかった時点で探索を打ち切ることができる。

### 検索（Lookup）

検索操作は以下の手順で行われる：

1. キーをハッシュし、Quotient と Remainder に分割
2. Quotient に対応するスロットが Occupied か確認
3. Occupied でなければ要素は存在しない
4. Cluster の先頭から Run を走査
5. Run 内を線形探索して Remainder が一致するか確認

### リサイズ（Resize）

フィルタの負荷が高くなる、つまりクラスタが大きくなると性能劣化も大きくなる傾向にある[^2]。
色々やり方はあるが、今回は分かり易さを重視してリサイズ・再挿入を行った。
他には本書で言及されているように Remainder のビットを Quotient に寄せることでテーブルを実質倍にすることができる。

[^2]: 本書では充填率が 75~80% になると性能が大きく低下すると説明している

### マージ（Merge）

複数の QF を結合する操作は、分散システムでの集約などに有用である。
両方のフィルタからすべてのキーを収集し、十分な容量を持つ新しいフィルタに再挿入する。

# 実装の詳細

## スロット構造とビット操作

各スロットは `u64` の1ワードで表現され、以下のようにビットを分割している。

```rust
const FLAG_BITS: u64 = 3;
const FLAG_MASK: u64 = (1 << FLAG_BITS) - 1;
const FLAG_OCCUPIED: u64 = 1 << 0;
const FLAG_CONTINUED: u64 = 1 << 1;
const FLAG_SHIFTED: u64 = 1 << 2;

struct Slot {
    data: u64,
}
```

下位3ビットがフラグ、残りが Remainder となる。

```
[Remainder (61 bits)][Shifted][Continued][Occupied]
```

フラグの取得・設定は以下のように実装される。

```rust
impl Slot {
    fn remainder(&self) -> u64 {
        self.data >> FLAG_BITS
    }

    fn set_remainder(&mut self, remainder: u64) {
        let flags = self.data & FLAG_MASK;
        self.data = (remainder << FLAG_BITS) | flags;
    }

    fn is_occupied(&self) -> bool {
        (self.data & FLAG_OCCUPIED) != 0
    }

    fn set_occupied(&mut self, value: bool) {
        if value {
            self.data |= FLAG_OCCUPIED;
        } else {
            self.data &= !FLAG_OCCUPIED;
        }
    }
}
```

## 挿入処理の詳細

### 空のスロットへの挿入

最もシンプルなケースは、Quotient のスロットが空の場合である。

```rust
if self.filter[q_idx].is_empty() {
    self.filter[q_idx].set_remainder(remainder);
    self.filter[q_idx].set_occupied(true);
    self.entries += 1;
    return;
}
```

### Run への挿入とソート保持

同じ Quotient を持つ要素を挿入する場合、Run 内で Remainder の昇順を保つ必要がある。

```rust
let run_head = self.find_run_head(q_idx);
let mut insert_pos = run_head;

if !self.filter[insert_pos].is_empty() && self.filter[insert_pos].remainder() < remainder {
    loop {
        insert_pos = self.next_index(insert_pos);
        if !(self.filter[insert_pos].is_continued()
            && self.filter[insert_pos].remainder() < remainder)
        {
            break;
        }
    }
}
```

### シフト操作

挿入位置にすでに別の要素がある場合、後続の要素を1つずつシフトして空きスロットを作る。
スロット列は循環バッファとしている。

```rust
let mut empty_pos = insert_pos;
while !self.filter[empty_pos].is_empty() {
    empty_pos = self.next_index(empty_pos);
}

let mut curr = empty_pos;
while curr != insert_pos {
    let prev = self.prev_index(curr);
    curr = prev;
}

self.filter[insert_pos].set_remainder(remainder);
self.filter[insert_pos].set_shifted(insert_pos != q_idx);
```

## 検索処理の詳細

### Run Head の探索

Run のヘッドを見つけるためにはまず所属する Cluster の先頭からみる必要がある。
Cluster の先頭は `Shifted` となっているスロットを戻ることで簡単に見つかる。
Cluster の先頭が見つかったら、当該 Run の先頭を探索する。

```rust
fn find_run_head(&self, home_idx: usize) -> usize {
    let mut bucket = home_idx;
    while self.filter[bucket].is_shifted() {
        bucket = self.prev_index(bucket);
    }

    let mut run_head = bucket;
    let mut probe = bucket;
    while probe != home_idx {
        run_head = self.next_index(run_head);
        while self.filter[run_head].is_continued() {
            run_head = self.next_index(run_head);
        }
        probe = self.next_index(probe);
        while !self.filter[probe].is_occupied() {
            probe = self.next_index(probe);
        }
    }
    run_head
}
```

### Run 内の線形探索

Run の先頭が見つかったら、Continued フラグを追いながら一致する Remainder を探す。
挿入時に昇順にしているので見つかったら途中で探索を打ち切ってよい。

```rust
pub fn lookup(&self, key: u64) -> bool {
    let (quotient, remainder) = self.split(key);
    let q_idx = quotient as usize;

    if !self.filter[q_idx].is_occupied() {
        return false;
    }

    let run_head = self.find_run_head(q_idx);
    if self.filter[run_head].remainder() == remainder {
        return true;
    }

    let mut idx = self.next_index(run_head);
    while self.filter[idx].is_continued() {
        if self.filter[idx].remainder() == remainder {
            return true;
        }
        idx = self.next_index(idx);
    }

    false
}
```

# まとめ

Bloom Filter のような単純な仕組みかと思いきや、連続アクセスにするための工夫として Quotient や Run、メタデータを導入していて結構複雑な印象だった。
内容が分かるとそれほど凝った実装にはならないはず...

Quotient Filter についての研究は直近でも行われており、特に [CQF](https://dl.acm.org/doi/10.1145/3035918.3035963) や [Quotient Filters: Approximate Membership Queries on the GPU](https://ieeexplore.ieee.org/document/8425199) あたりが気になるので時間のある時に読んでみようと思う。