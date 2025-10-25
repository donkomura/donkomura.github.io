+++
title = "Quotient Filter - 高効率な確率的データ構造の実装と性能評価"
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

確率的データ構造（Probabilistic Data Structures）は、メモリ効率と高速な操作を実現するために、正確性を一部犠牲にしたデータ構造である。
代表的なものとして Bloom Filter があるが、今回は Bloom Filter の欠点を改善した **Quotient Filter** について解説し、Rust での実装を紹介する。

実装コードは [GitHub リポジトリ](https://github.com/donkomura/hash_bench) で公開している。

# Quotient Filter とは

Quotient Filter は 2011 年に Bender et al. によって提案された確率的データ構造で、集合のメンバーシップテストを効率的に行うことができる。
「要素が集合に含まれているか」を高速に判定できるが、偽陽性（False Positive）が発生する可能性がある。
つまり、実際には存在しない要素を「存在する」と誤って判定することがある。

## Bloom Filter との違い

Bloom Filter と比較した Quotient Filter の主な利点は以下の通りである：

1. **削除操作のサポート**: Bloom Filter では要素の削除が困難だが、Quotient Filter では削除が可能
2. **リサイズのサポート**: 容量を動的に拡張できる
3. **マージ操作**: 複数の Quotient Filter を効率的に結合できる
4. **空間局所性**: データがキャッシュフレンドリーに配置される
5. **より良いメモリ効率**: 同じ False Positive Rate で Bloom Filter より少ないメモリで実装可能な場合がある

一方で、Bloom Filter に比べて実装の複雑さが増すというトレードオフがある。

## 用途

Quotient Filter は以下のような場面で活用される：

- データベースのキャッシュ（存在しないキーへのアクセスを高速に判定）
- 重複検出システム
- ネットワークルーティングテーブル
- ストレージシステムの Bloom Filter 代替

# 基本的な仕組み

## ハッシュ値の分割

Quotient Filter の核となるアイデアは、ハッシュ値を **Quotient**（商）と **Remainder**（余り）に分割することである。

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

## 3つのメタデータビット

Quotient Filter では、各スロットに3つのメタデータフラグを持つ：

1. **Occupied (is_occupied)**: そのスロットのインデックスを Quotient として持つ要素が存在するか
2. **Continued (is_continued)**: そのスロットが Run の一部（先頭以外）であるか
3. **Shifted (is_shifted)**: そのスロットに格納された Remainder が本来の位置からシフトされているか

これらのフラグにより、衝突が発生した際にどのように要素を配置・検索するかを管理する。

## Run とクラスタリング

同じ Quotient を持つ要素の集まりを **Run** と呼ぶ。
Run 内では Remainder の昇順で要素が並べられる。

複数の Run が連続して配置されている場合、これを **クラスター** と呼ぶ。
クラスターが形成されると、本来の位置（ホームスロット）から離れた位置に要素が配置される（Shifted）。

```
[Slot 0][Slot 1][Slot 2][Slot 3][Slot 4]
         └─ Run for q=1 ─┘ └─ Run for q=2 ─┘
         └────── Cluster ──────┘
```

# 主要な操作

## 挿入（Insert）

挿入操作は以下の手順で行われる：

1. キーをハッシュし、Quotient と Remainder に分割
2. Quotient に対応するスロット（ホームスロット）を確認
3. スロットが空なら直接挿入
4. スロットが使用中なら Run の位置を特定し、適切な位置に挿入

### 空のスロットへの挿入

最もシンプルなケースは、ホームスロットが空の場合である。

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

実装のポイントは以下の通り：

```rust
let run_head = self.find_run_head(q_idx);
let mut insert_pos = run_head;

// Run 内で適切な挿入位置を探索（昇順を保つ）
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

```rust
// 空のスロットを探す
let mut empty_pos = insert_pos;
while !self.filter[empty_pos].is_empty() {
    empty_pos = self.next_index(empty_pos);
}

// 要素を後方にシフト
let mut curr = empty_pos;
while curr != insert_pos {
    let prev = self.prev_index(curr);
    // prev の内容を curr にコピー
    curr = prev;
}

// 挿入位置に新しい Remainder を設定
self.filter[insert_pos].set_remainder(remainder);
self.filter[insert_pos].set_shifted(insert_pos != q_idx);
```

### ラップアラウンド

Quotient Filter はリングバッファとして実装されているため、配列の末尾に達したら先頭に戻る。

```rust
fn next_index(&self, idx: usize) -> usize {
    (idx + 1) % self.size
}

fn prev_index(&self, idx: usize) -> usize {
    (idx + self.size - 1) % self.size
}
```

## 検索（Lookup）

検索操作は以下の手順で行われる：

1. キーをハッシュし、Quotient と Remainder に分割
2. Quotient に対応するホームスロットが Occupied か確認
3. Occupied でなければ要素は存在しない
4. Run の先頭を特定
5. Run 内を線形探索して Remainder が一致するか確認

### Run Head の探索

Run の先頭を見つけるアルゴリズムは Quotient Filter の中核である。

```rust
fn find_run_head(&self, home_idx: usize) -> usize {
    // まず、シフトされていない最も近いスロットを見つける
    let mut bucket = home_idx;
    while self.filter[bucket].is_shifted() {
        bucket = self.prev_index(bucket);
    }

    // Occupied フラグを数えながら、対応する Run の先頭まで進む
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

このアルゴリズムは以下のロジックで動作する：

1. `home_idx` より前で Shifted フラグが立っていない位置まで戻る
2. そこから Occupied の数と Run の数を数えながら進む
3. `home_idx` に対応する Run の先頭を特定

### Run 内の線形探索

Run の先頭が見つかったら、Continued フラグを追いながら一致する Remainder を探す。

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

## リサイズ（Resize）

フィルタの負荷率が高くなると、False Positive Rate が上昇し、クラスタリングによる性能劣化が発生する。
そのため、適切なタイミングでリサイズを行う必要がある。

今回の実装では、エントリ数がテーブルサイズに達したらリサイズを行う（負荷率 100%）。

```rust
pub fn resize(&mut self) {
    let new_q = self.q + 1;  // テーブルサイズを2倍に拡張

    // 既存のすべてのキーを収集
    let keys = self.collect_keys();

    // 新しいフィルタを作成
    let mut new_qf = QuotientFilter::new(new_q, self.r);

    // すべてのキーを再挿入
    for key in keys {
        new_qf.insert(key);
    }

    *self = new_qf;
}
```

### collect_keys の実装

すべての要素を収集するには、各 Quotient について Run を辿る必要がある。

```rust
fn collect_keys(&self) -> Vec<u64> {
    let mut keys = Vec::with_capacity(self.entries);

    for quotient_idx in 0..self.size {
        if !self.filter[quotient_idx].is_occupied() {
            continue;
        }

        let run_head = self.find_run_head(quotient_idx);
        self.visit_run(run_head, |slot_idx| {
            // Quotient と Remainder を結合してキーを復元
            let key = ((quotient_idx as u64) << self.r)
                    | self.filter[slot_idx].remainder();
            keys.push(key);
        });
    }

    keys
}
```

## マージ（Merge）

複数の Quotient Filter を結合する操作は、分散システムでの集約などに有用である。

```rust
pub fn merge(&self, other: &Self) -> Self {
    assert_eq!(self.r, other.r,
        "cannot merge filters with different remainder sizes");

    // 両方のキーを収集
    let keys_self = self.collect_keys();
    let keys_other = other.collect_keys();
    let total_entries = keys_self.len() + keys_other.len();

    // 十分な容量を持つフィルタを作成
    let mut target_q = self.q.max(other.q);
    let mut capacity = 1usize << target_q;
    while capacity < total_entries {
        target_q += 1;
        capacity = 1usize << target_q;
    }

    // すべてのキーを挿入
    let mut merged = QuotientFilter::new(target_q, self.r);
    for key in keys_self.into_iter().chain(keys_other.into_iter()) {
        merged.insert(key);
    }

    merged
}
```

# 実装のポイント

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
    // is_continued, is_shifted についても同様...
}
```

## メモリ効率とパフォーマンスのトレードオフ

Quotient Filter のメモリ使用量は以下の式で表される：

```
Memory = (q + r + 3) * 2^q bits
```

- `q`: Quotient のビット数（テーブルサイズ = 2^q）
- `r`: Remainder のビット数
- `3`: メタデータビット（Occupied, Continued, Shifted）

Bloom Filter のメモリ使用量は一般に以下で表される：

```
Memory = -n * ln(p) / (ln(2))^2 bits
```

- `n`: 要素数
- `p`: False Positive Rate

例えば、`n = 1024`, `p = 0.01` の場合、Bloom Filter は約 9900 ビット必要だが、
Quotient Filter で `q = 10`, `r = 7` とすると `(10 + 7 + 3) * 1024 = 20480` ビットとなる。

ただし、Quotient Filter は削除・リサイズ・マージが可能であり、単純なメモリ効率だけでは比較できない。

# パフォーマンス評価

Criterion を使ってベンチマークを実施した。測定項目は挿入性能と検索性能である。

## 挿入性能

異なる負荷率（25%, 50%, 75%）とテーブルサイズ（`q=10`, `q=12`）で挿入性能を測定した。

```rust
fn bench_quotient_filter_insert(c: &mut Criterion) {
    let mut group = c.benchmark_group("quotient_filter_insert");
    let r = 8;
    let load_factors = [25usize, 50, 75];
    let qs = [10u64, 12u64];

    for &q in &qs {
        let capacity = 1usize << q;
        for &load in &load_factors {
            let target_entries = capacity * load / 100;
            // ベンチマーク実行...
        }
    }
}
```

負荷率が高くなるほど、クラスタリングが発生しシフト操作が増えるため、挿入時間が増加する。

## 検索性能

検索性能は、フィルタに格納された要素と存在しない要素を混在させて測定した。

```rust
fn bench_quotient_filter_lookup(c: &mut Criterion) {
    let mut group = c.benchmark_group("quotient_filter_lookup");
    let r = 8;
    let qs = [10u64, 12u64];
    let probe_ratio = 10; // 挿入数の10倍のクエリ

    for &q in &qs {
        let capacity = 1usize << q;
        let target_entries = capacity / 2;
        // probes の 1/10 は実際に存在する要素
        // 残りはランダムな要素（大半は存在しない）
        // ベンチマーク実行...
    }
}
```

検索は Run の長さに依存するため、負荷率が高いほど性能が劣化する。

### 性能特性のまとめ

- **挿入**: 平均 `O(1)`、最悪ケース `O(n)`（大きなクラスタが形成された場合）
- **検索**: 平均 `O(1)`、最悪ケース `O(n)`（Run の長さに依存）
- **リサイズ**: `O(n)`（全要素の再挿入が必要）
- **マージ**: `O(n + m)`（n, m は各フィルタの要素数）

# Bloom Filter との比較

## メモリ使用量

前述の通り、同じ False Positive Rate を実現する場合、Bloom Filter の方がメモリ効率が良いことが多い。
ただし、Quotient Filter は削除やリサイズをサポートするため、動的な環境では Quotient Filter が有利になる。

## False Positive Rate

Quotient Filter の False Positive Rate は以下で近似される：

```
FPR ≈ (load_factor)^(r+1)
```

- `load_factor`: 負荷率（`entries / size`）
- `r`: Remainder のビット数

例えば、負荷率 50%, `r=8` の場合、FPR ≈ 0.002 となる。

## 削除・リサイズ・マージのサポート

| 操作      | Bloom Filter | Quotient Filter |
|-----------|--------------|-----------------|
| 挿入      | ✓            | ✓               |
| 検索      | ✓            | ✓               |
| 削除      | ✗            | ✓               |
| リサイズ  | ✗            | ✓               |
| マージ    | ✓            | ✓               |

Bloom Filter は削除が困難（Counting Bloom Filter を除く）だが、Quotient Filter は削除をサポートできる。
また、リサイズも可能なため、動的な環境で有利である。

## 使い分けのガイドライン

- **Bloom Filter を選ぶべき場合**:
  - 要素数が事前に分かっている
  - 削除やリサイズが不要
  - メモリ効率を最優先したい

- **Quotient Filter を選ぶべき場合**:
  - 削除操作が必要
  - 動的にサイズを変更したい
  - 複数のフィルタをマージする必要がある
  - キャッシュ効率を重視したい

# まとめ

Quotient Filter は Bloom Filter の欠点を補う確率的データ構造であり、以下の特徴を持つ：

**強み**:
- 削除・リサイズ・マージをサポート
- キャッシュフレンドリーな設計
- 動的な環境に適している

**弱み**:
- 実装が複雑
- 同じ False Positive Rate では Bloom Filter よりメモリを多く使う場合がある
- クラスタリングによる性能劣化の可能性

今回の実装を通じて、Quotient Filter の内部構造とアルゴリズムを深く理解することができた。
特に Run Head の探索アルゴリズムは、メタデータビットをうまく活用した巧妙な設計である。

## 今後の拡張可能性

Quotient Filter にはいくつかの拡張版が提案されている：

- **Counting Quotient Filter**: 各要素の出現回数をカウント
- **Rank-and-Select Quotient Filter**: Rank/Select 操作を追加してメモリ効率を向上
- **Cuckoo Filter**: 別のハッシュ手法を組み合わせた確率的データ構造

これらの拡張版について、別の機会に実装と評価を行いたいと思う。

## 参考文献

- Bender, M. A., et al. (2011). "Don't Thrash: How to Cache Your Hash on Flash." *VLDB*.
- [hash_bench - GitHub Repository](https://github.com/donkomura/hash_bench)
