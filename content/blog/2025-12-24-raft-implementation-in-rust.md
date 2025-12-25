+++
title = "RustでRaft Consensus Algorithmを実装した"
slug = "raft-implementation-in-rust"
date = 2025-12-24

[taxonomies]
categories = ["記事"]
tags = ["rust", "distributed-systems", "raft", "consensus"]

[extra]
lang = "jp"
toc = true
comment = false
copy = true
math = false
mermaid = false
outdate_alert = false
outdate_alert_days = 120
display_tags = true
truncate_summary = false
featured = false
reaction = false
+++

分散システムにおける合意形成アルゴリズムとして知られている Raft を、Rust で実装してみた。この記事では、Rust の型システムとトレイトを活用した設計について紹介する。

<!-- more -->

## なぜ Rust で分散システムを実装するのか

分散システムの実装では、データ競合や状態不整合が実行時エラーを引き起こす可能性がある。また、拡張性を確保しつつ実装の誤りを防ぐ必要があり、複数ノード間の通信でスレッド安全性を保証する必要もある。

これらの課題に対して、Rust の型システムとトレイトを使うことで、抽象インターフェースを定義し実装の誤りをコンパイル時に検出できる。さらに、所有権システムによりデータ競合をコンパイル時に防げるため、並行処理の安全性を保証しやすい。

この記事では、[ikada](https://github.com/donkomura/ikada) という、[Raft の論文](https://raft.github.io/raft.pdf)のコア機能を実装したライブラリを例に、型システムとトレイトをどう活用したかを紹介する。

実装した主な機能：
- Leader Election (リーダー選出)
- Log Replication (ログレプリケーション)
- Safety Guarantees (安全性の保証)
- State Persistence (状態の永続化)
- Client Interaction (クライアントとの通信)

## アーキテクチャ概要

### 状態管理：型で仕様を表現する

Raft の論文 Figure 2 には、ノードが管理すべき状態が定義されている。これを `RaftState` 構造体で忠実に実装した：

```rust
pub struct RaftState<T: Send + Sync, SM: StateMachine<Command = T>> {
    // Persistent state on all servers（永続化が必要）
    pub persistent: PersistentState<T>,

    // Volatile state on all servers（揮発性）
    pub commit_index: u32,
    pub last_applied: u32,

    // Volatile state on leader（Leaderのみ）
    pub next_index: HashMap<SocketAddr, u32>,
    pub match_index: HashMap<SocketAddr, u32>,

    pub role: Role,
    storage: Box<dyn Storage<T>>,  // トレイトオブジェクトで抽象化
    sm: SM,                        // ジェネリクスで型安全性
}
```

永続化が必要な状態を `PersistentState` 構造体として物理的に分離している。これにより、再起動時に復旧すべき状態が型として明確になる。また、型パラメータ `T: Send + Sync` でコマンド型を抽象化し、`SM: StateMachine<Command = T>` でステートマシンを差し替え可能にしている。

ノードのロールは Enum で表現している：

```rust
#[derive(Debug, Clone, Copy, Default, PartialEq)]
pub enum Role {
    #[default]
    Follower,
    Candidate,
    Leader,
}
```

Enum によって、ノードのロールが常に3つのうちいずれか1つであることが保証される。

## トレイトと型システムによる設計

Rust の型システムを活用した設計について説明する。

### 1. StateMachine トレイト：拡張可能な状態機械

Raft は合意形成のコアアルゴリズムで、実際にどんな状態機械（State Machine）を動かすかはユーザー次第である。`StateMachine` トレイトを定義することで、任意の状態機械を統合できるようにしている：

```rust
#[async_trait::async_trait]
pub trait StateMachine: Send + Sync {
    type Command: Send + Sync + Clone;
    type Response: Send + Sync;

    async fn apply(
        &mut self,
        command: &Self::Command,
    ) -> anyhow::Result<Self::Response>;
}
```

インターフェースはシンプルにしている。`apply` メソッド1つだけを定義し、コマンドを受け取って結果を返す形にした。`Command` と `Response` を関連型にすることで、ユーザーが任意の型を指定できる。したがって、Key-Value Store、データベース、計算エンジンなど、様々な状態機械に対応できる。

また、`Send + Sync` 制約により、コンパイラが自動的にスレッド安全性をチェックする。これにより、マルチスレッド環境で非スレッド安全な型を使った場合、コンパイル時にエラーとなる。

デフォルト実装として Key-Value Store を提供している：

```rust
// デフォルトのKVストア実装
#[derive(Default, Debug)]
pub struct KVStateMachine {
    data: HashMap<String, String>,
}

#[derive(Clone, Debug, Serialize, Deserialize)]
pub enum KVCommand {
    Set { key: String, value: String },
    Get { key: String },
    Delete { key: String },
}
```

### 2. Storage トレイト：永続化層の抽象化

Raft の状態を永続化するため、`Storage` トレイトを定義している：

```rust
#[async_trait::async_trait]
pub trait Storage<T: Send + Sync>: Send + Sync {
    async fn save(&mut self, state: &PersistentState<T>) -> anyhow::Result<()>;
    async fn load(&self) -> anyhow::Result<Option<PersistentState<T>>>;
}
```

このトレイトにより、バックエンドの切り替えが容易になる。したがって、メモリ、ファイルシステム、データベースなど、実装を差し替えるだけで対応できる。例えば、テストではメモリストレージを使い、本番環境では永続化ストレージを使うといった使い分けができる。

### 3. RaftRpc トレイト：型安全な RPC 定義

ノード間通信には [tarpc](https://github.com/google/tarpc) を使用し、RPC インターフェースをトレイトで定義している：

```rust
#[tarpc::service]
pub trait RaftRpc {
    async fn append_entries(req: AppendEntriesRequest) -> AppendEntriesResponse;
    async fn request_vote(req: RequestVoteRequest) -> RequestVoteResponse;
    async fn client_request(req: CommandRequest) -> CommandResponse;
}
```

リクエスト/レスポンスの型が一致しない場合、コンパイル時にエラーとなる。また、tarpc が自動的にクライアント/サーバーコードを生成するため、ボイラープレートを書く必要がない。

### 4. 共有状態の安全な管理

分散システムでは状態の共有が不可欠だが、Rust の所有権システムと調和させる必要がある。そこで、`Arc<Mutex<RaftState>>` を使用することで、安全な共有可変状態を実現している：

```rust
pub struct Node<T: Send + Sync, SM: StateMachine<Command = T>> {
    state: Arc<Mutex<RaftState<T, SM>>>,
    // ...
}
```

`Mutex` により、同時に1つのスレッドしか状態を変更できないため、データ競合を防げる。`Arc` により、どのスレッドが状態を共有しているか追跡でき、参照カウントが0になると自動的にメモリが解放される。

ただし、Rust の型システムは「Mutex 保護領域を触るにはロックが必要」という制約を保証できるが、デッドロックや非同期コンテキストでの `.await` をまたいだロック保持、タスク飢餓などは別途設計で対処する必要がある。これらの課題については「今後の展望」セクションで詳しく触れる。

なお、各種通知に関してはチャネルで実装した。Go と似たメンタルモデルで実装できたのは、Go の経験が生きたように感じた。

## Raft アルゴリズムの実装

Rust の型システムを活用した設計について説明してきたが、次に Raft の具体的なアルゴリズム実装について触れる。

### Leader Election：非同期並列処理

Follower は一定時間 Leader からのハートビートを受信しないと、Candidate に遷移して選挙を開始する。すべてのピアに並列で RequestVote RPC を送信する。

```rust
// Request votes from all peers in parallel
let mut tasks = JoinSet::new();
for (addr, client) in peers {
    tasks.spawn(Self::send_request_vote(addr, client, req, rpc_timeout));
}

// Collect responses
while let Some(result) = tasks.join_next().await {
    // レスポンスを収集
}
```

`tokio::task::JoinSet` を使用することで、複数の RPC を並列実行している。過半数の投票を獲得した時点で Leader に遷移する。

### Log Replication：並列実行

Leader は AppendEntries RPC を使って Follower にログエントリを複製する。Log Replication も並列実行している。各 Follower の `next_index` に基づいて送信すべきエントリを決定し、並列で AppendEntries RPC を送信する。

## 今後の展望

現在の実装は Raft のコア機能に焦点を当てているが、実用化に向けていくつかの機能を追加していく予定である。

### Raft の追加機能

まず、Cluster Membership Changes（論文§6）の実装である。これにより、クラスタの稼働中にノードを動的に追加・削除できるようになる。現在の実装では、クラスタの構成が固定されているため、スケールアウトやノード交換の際にシステムを停止する必要がある。

次に、Log Compaction and Snapshots（論文§7）の実装である。これは、ログが無限に増え続けることを防ぐための機能で、スナップショットを作成することでストレージ使用量を削減する。

### 並行性とロック設計の改善

現在の実装では、`Arc<Mutex<RaftState>>` を使って状態を管理しているが、これにはいくつかの改善の余地がある。

Rust の型システムは「Mutex 保護領域を触るにはロックが必要」という制約を保証できるが、以下の点は別途設計で対処する必要がある：

- デッドロックの防止（ロックを取得する順序の管理）
- 非同期コンテキストでの `.await` をまたいだロック保持の回避
- タスク飢餓や優先度逆転の防止

特に、本実装では `tokio::task::JoinSet` を使った非同期並列処理を行っているため、`std::sync::Mutex` を使う場合は `.await` をまたいでロックを保持しないよう注意が必要である。ロック保持中に `.await` すると、そのスレッドがブロックされ、他のタスクの実行を妨げる可能性がある。

このような問題への対処法として、以下のパターンを検討している：

1. **2フェーズパターン**: ロックを取得して必要な情報をコピー → ロックを解放 → 非同期処理(`.await`) → 再度ロックを取得して結果を反映
2. **tokio::sync::Mutex の使用**: `.await` をまたいでもロックを保持できる非同期対応の Mutex を使用する
3. **Actor モデル**: Raft の状態を1つのタスクに閉じ込め、外部とはチャネル経由でメッセージをやり取りする

また、単一の `Arc<Mutex<RaftState>>` に全ての状態を入れると、Leader が AppendEntries の応答を処理する際など、頻繁にロックの競合が発生しやすい。より高いパフォーマンスが必要な場合は、以下のような最適化を検討する：

- 永続状態 (persistent) と揮発性状態 (volatile) でロックを分割する
- Leader 専用の状態を別のロック（または別タスク）で管理する
- peer 毎のレプリケーション状態（nextIndex/matchIndex）を個別に管理する

### Raft 実装の重要論点

Raft の実装において、正確性と性能の両面で重要だが、現在の記事では詳しく触れていない論点がいくつかある。

#### 1. タイマーとランダム化

Raft の正確性は、ランダム化された election timeout に依存している。これは、複数の Follower が同時に Candidate に遷移して投票が分散するのを防ぐための仕組みである。論文でも明確に重要視されている。

現在の実装では「一定時間ハートビートが来ないと選挙開始」としているが、実装には election timeout のランダム化レンジや tick 設計の詳細が必要になる。これは liveness の保証に直結する重要な設計要素である。

#### 2. ログ整合性と AppendEntries 失敗時の最適化

AppendEntries が失敗したときの `nextIndex` 調整は、Raft 実装の難所の一つである。単純に `nextIndex -= 1` とすると、Follower のログが大きく遅れている場合に最悪 O(n²) の時間がかかる。

論文には最適化のヒント（conflict term と conflict index）が記載されており、これを実装することでログの整合性を効率的に取ることができる。本実装では並列送信まで実装しているため、この最適化も検討する価値がある。

#### 3. StateMachine apply とクライアント応答

`StateMachine::apply` が `&mut self` を取るのは適切だが、実装で事故りやすいのは以下の点である：

- commit された順序で**必ず一度だけ** apply する
- apply 結果をクライアントに返す仕組み（特にリーダーが変わった場合）
- リトライ時の重複対策（idempotency 保証、client request id の管理）

これらは Raft の運用上の「体感難易度」を大きく左右する要素である。

#### 4. 永続化のタイミング

`Storage` トレイトで永続化層を抽象化しているが、Raft では「いつ fsync してから応答するか」が安全性を左右する。典型例は以下の通り：

- `currentTerm` と `votedFor` を書き込んでから RequestVote に応答する
- ログエントリを永続化してから AppendEntries に応答する

これらの書き込み順序と、RPC 応答の前後関係を誤ると、再起動後に Raft の不変条件が破られる可能性がある。本実装は学習目的のライブラリだが、この点について注意書きがあると、読者の誤実装を減らせる。

### テストの充実

統合テストの充実も進めている。複数ノードを起動して実際の Raft クラスタをシミュレートする統合テストを実装中で、Leader Election の正常系や Leader 障害時の再選出、Log Replication の検証、ネットワーク分断のシミュレーション、State Persistence のテストなど、実際の分散環境での動作を検証している。

さらに、以下のような障害シナリオのテストも追加していく予定である：

- ネットワークパーティション（split-brain）
- クロックスキュー
- メッセージの重複・遅延・順序入れ替え

分散システムのバグは単体テストだけでは発見が困難なため、統合テストは重要だと考えている。

### パフォーマンス測定と最適化

パフォーマンス測定と最適化を行う予定である。現時点では機能の正確性を優先しているが、実用化にはスループットとレイテンシーの改善が必要になると思う。そのため、ベンチマークを作成してボトルネックを特定し、最適化を適用していく予定だ。

## まとめ

Rust で Raft を実装してみた。型システムとトレイトを活用することで、安全かつ拡張可能な設計ができたと感じている。

トレイトによる抽象化は有用だった。`StateMachine`、`Storage`、`RaftRpc` など、インターフェースを明確に定義することで、ユーザーが独自実装を差し込める拡張性を確保できた。また、型システムによる安全性も重要で、`Send + Sync` 境界やジェネリクスにより、コンパイル時にスレッド安全性や型の整合性をチェックできる。さらに、所有権システムは、`Arc<Mutex<T>>` によるデータ競合の防止と、明確な所有権管理を実現する。

一方で、型システムだけでは防げない複雑な安全性ルールも存在する。例えば、現在の term のエントリのみコミットするという Raft の安全性ルールは、ロジックとして実装する必要がある。したがって、これらは明確な構造体設計と単体テストで補完している。

分散システムの実装は難しいが、Rust の型システムを活用することで、実行時エラーをコンパイル時に検出できる。これにより、安全性と開発効率の両立がしやすいと思う。

興味のある方は、[GitHub リポジトリ](https://github.com/donkomura/ikada)をご覧いただきたい。

## 参考文献

- [In Search of an Understandable Consensus Algorithm (Extended Version)](https://raft.github.io/raft.pdf)
- [Raft Visualization](http://thesecretlivesofdata.com/raft/)
- [Raft GitHub Pages](https://raft.github.io/)
