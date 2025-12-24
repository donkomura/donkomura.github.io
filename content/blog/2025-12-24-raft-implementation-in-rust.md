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

分散システムにおける合意形成アルゴリズムとして知られているRaftを、Rustで実装してみた。この記事では、Rustの型システムとトレイトを活用した設計について解説する。

<!-- more -->

## なぜRustで分散システムを実装するのか

分散システムの実装では、データ競合や状態不整合が実行時エラーを引き起こす可能性がある。また、拡張性を確保しつつ実装の誤りを防ぐ必要があり、複数ノード間の通信でスレッド安全性を保証する必要もある。

これらの課題に対して、Rustの型システムとトレイトを使うことで、抽象インターフェースを定義し実装の誤りをコンパイル時に検出できる。さらに、所有権システムによりデータ競合をコンパイル時に防げるため、並行処理の安全性を保証しやすい。

この記事では、[ikada](https://github.com/donkomura/ikada)という、[Raftの論文](https://raft.github.io/raft.pdf)のコア機能を実装したライブラリを例に、型システムとトレイトをどう活用したかを紹介する。

実装した主な機能：
- Leader Election (リーダー選出)
- Log Replication (ログレプリケーション)
- Safety Guarantees (安全性の保証)
- State Persistence (状態の永続化)
- Client Interaction (クライアントとの通信)

## アーキテクチャ概要

### 状態管理：型で仕様を表現する

Raftの論文Figure 2には、ノードが管理すべき状態が定義されている。これを`RaftState`構造体で忠実に実装した：

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

永続化が必要な状態を`PersistentState`構造体として物理的に分離している。これにより、再起動時に復旧すべき状態が型として明確になる。また、型パラメータ`T: Send + Sync`でコマンド型を抽象化し、`SM: StateMachine<Command = T>`でステートマシンを差し替え可能にしている。

ノードのロールはEnumで表現している：

```rust
#[derive(Debug, Clone, Copy, Default, PartialEq)]
pub enum Role {
    #[default]
    Follower,
    Candidate,
    Leader,
}
```

Enumによって、ノードのロールが常に3つのうちいずれか1つであることが保証される。

## トレイトと型システムによる設計

Rustの型システムを活用した設計について説明する。

### 1. StateMachineトレイト：拡張可能な状態機械

Raftは合意形成のコアアルゴリズムで、実際にどんな状態機械（State Machine）を動かすかはユーザー次第である。`StateMachine`トレイトを定義することで、任意の状態機械を統合できるようにしている：

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

インターフェースはシンプルにしている。`apply`メソッド1つだけを定義し、コマンドを受け取って結果を返す形にした。`Command`と`Response`を関連型にすることで、ユーザーが任意の型を指定できる。したがって、Key-Value Store、データベース、計算エンジンなど、様々な状態機械に対応できる。

また、`Send + Sync`制約により、コンパイラが自動的にスレッド安全性をチェックする。これにより、マルチスレッド環境で非スレッド安全な型を使った場合、コンパイル時にエラーとなる。

デフォルト実装としてKey-Value Storeを提供している：

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

### 2. Storageトレイト：永続化層の抽象化

Raftの状態を永続化するため、`Storage`トレイトを定義している：

```rust
#[async_trait::async_trait]
pub trait Storage<T: Send + Sync>: Send + Sync {
    async fn save(&mut self, state: &PersistentState<T>) -> anyhow::Result<()>;
    async fn load(&self) -> anyhow::Result<Option<PersistentState<T>>>;
}
```

このトレイトにより、バックエンドの切り替えが容易になる。したがって、メモリ、ファイルシステム、データベースなど、実装を差し替えるだけで対応できる。例えば、テストではメモリストレージを使い、本番環境では永続化ストレージを使うといった使い分けができる。

### 3. RaftRpcトレイト：型安全なRPC定義

ノード間通信には[tarpc](https://github.com/google/tarpc)を使用し、RPCインターフェースをトレイトで定義している：

```rust
#[tarpc::service]
pub trait RaftRpc {
    async fn append_entries(req: AppendEntriesRequest) -> AppendEntriesResponse;
    async fn request_vote(req: RequestVoteRequest) -> RequestVoteResponse;
    async fn client_request(req: CommandRequest) -> CommandResponse;
}
```

リクエスト/レスポンスの型が一致しない場合、コンパイル時にエラーとなる。また、tarpcが自動的にクライアント/サーバーコードを生成するため、ボイラープレートを書く必要がない。

### 4. 共有状態の安全な管理

分散システムでは状態の共有が不可欠だが、Rustの所有権システムと調和させる必要がある。そこで、`Arc<Mutex<RaftState>>`を使用することで、安全な共有可変状態を実現している：

```rust
pub struct Node<T: Send + Sync, SM: StateMachine<Command = T>> {
    state: Arc<Mutex<RaftState<T, SM>>>,
    // ...
}
```

`Mutex`により、同時に1つのスレッドしか状態を変更できないため、データ競合を防げる。また、ロックの取得忘れはコンパイルエラーとなる。`Arc`により、どのスレッドが状態を共有しているか追跡でき、参照カウントが0になると自動的にメモリが解放される。

なお、各種通知に関してはチャネルで実装した。Goと似たメンタルモデルで実装できたのは、Goの経験が生きたように感じた。

## Raftアルゴリズムの実装

Rustの型システムを活用した設計について説明してきたが、次にRaftの具体的なアルゴリズム実装について触れる。

### Leader Election：非同期並列処理

Followerは一定時間Leaderからのハートビートを受信しないと、Candidateに遷移して選挙を開始する。すべてのピアに並列でRequestVote RPCを送信する。

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

`tokio::task::JoinSet`を使用することで、複数のRPCを並列実行している。過半数の投票を獲得した時点でLeaderに遷移する。

### Log Replication：並列実行

LeaderはAppendEntries RPCを使ってFollowerにログエントリを複製する。Log Replicationも並列実行している。各Followerの`next_index`に基づいて送信すべきエントリを決定し、並列でAppendEntries RPCを送信する。

## 今後の展望

現在の実装はRaftのコア機能に焦点を当てているが、実用化に向けていくつかの機能を追加する予定である。

まず、Cluster Membership Changes（論文§6）の実装である。これにより、クラスタの稼働中にノードを動的に追加・削除できるようになる。現在の実装では、クラスタの構成が固定されているため、スケールアウトやノード交換の際にシステムを停止する必要がある。

次に、Log Compaction and Snapshots（論文§7）の実装である。これは、ログが無限に増え続けることを防ぐための機能で、スナップショットを作成することでストレージ使用量を削減する。

また、統合テストの充実も進めている。複数ノードを起動して実際のRaftクラスタをシミュレートする統合テストを実装中で、Leader Electionの正常系やLeader障害時の再選出、Log Replicationの検証、ネットワーク分断のシミュレーション、State Persistenceのテストなど、実際の分散環境での動作を検証している。分散システムのバグは単体テストだけでは発見が困難なため、統合テストは重要だと考えている。

最後に、パフォーマンス測定と最適化を行う予定である。現時点では機能の正確性を優先しているが、実用化にはスループットとレイテンシーの改善が必要になると思う。そのため、ベンチマークを作成してボトルネックを特定し、最適化を適用していく予定だ。

## まとめ

RustでRaftを実装してみた。型システムとトレイトを活用することで、安全かつ拡張可能な設計ができたと感じている。

トレイトによる抽象化は有用だった。`StateMachine`、`Storage`、`RaftRpc`など、インターフェースを明確に定義することで、ユーザーが独自実装を差し込める拡張性を確保できた。また、型システムによる安全性も重要で、`Send + Sync`境界やジェネリクスにより、コンパイル時にスレッド安全性や型の整合性をチェックできる。さらに、所有権システムは、`Arc<Mutex<T>>`によるデータ競合の防止と、明確な所有権管理を実現する。

一方で、型システムだけでは防げない複雑な安全性ルールも存在する。例えば、現在のtermのエントリのみコミットするというRaftの安全性ルールは、ロジックとして実装する必要がある。したがって、これらは明確な構造体設計と単体テストで補完している。

分散システムの実装は難しいが、Rustの型システムを活用することで、実行時エラーをコンパイル時に検出できる。これにより、安全性と開発効率の両立がしやすいと思う。

興味のある方は、[GitHubリポジトリ](https://github.com/donkomura/ikada)をご覧いただきたい。

## 参考文献

- [In Search of an Understandable Consensus Algorithm (Extended Version)](https://raft.github.io/raft.pdf)
- [Raft Visualization](http://thesecretlivesofdata.com/raft/)
- [Raft GitHub Pages](https://raft.github.io/)
