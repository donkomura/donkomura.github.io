+++
title = "自宅 Kubernetes クラスタに MPI Operator を導入した話"
slug = "homelab-k8s-mpi-operator"
date = 2025-10-11

[taxonomies]
categories = ["記事"]
tags = ["kubernetes", "argocd", "mpi", "homelab"]

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

自宅で Kubernetes クラスタを運用していて、最近 [MPI Operator](https://github.com/kubeflow/mpi-operator) などのコンポーネントを ArgoCD で整備した。
この記事ではその過程と学んだことをまとめる。

# 背景

自宅で Kubernetes クラスタを運用している理由はいくつかある。
- クラウドネイティブな技術を実践的に学びたい
- 実験的なワークロードを気兼ねなく動かせる環境が欲しい
- インフラストラクチャをコードで管理する経験を積みたい

これまでは基本的なアプリケーションのデプロイや監視基盤の構築などを行ってきたが、より高度なワークロード、特に分散計算に興味を持つようになった。
そこで MPI (Message Passing Interface) を使った並列計算を Kubernetes 上で実行できるようにしようと考えた。

# MPI Operator とは

[MPI Operator](https://github.com/kubeflow/mpi-operator) は Kubernetes 上で MPI ジョブを実行するための Kubernetes Operator である。
Kubeflow プロジェクトの一部として開発されており、機械学習の分散学習などでよく使われる。

MPI Operator を使うと以下のようなことが可能になる:
- 複数の Pod 間での並列計算の実行
- MPIJob という Custom Resource Definition (CRD) による宣言的なジョブ管理
- Launcher と Worker の自動的なセットアップ
- SSH 鍵の自動管理

従来、MPI プログラムを複数のマシンで実行するには SSH の設定やホストファイルの管理など、様々な手作業が必要だった。
MPI Operator はこれらを自動化し、Kubernetes のリソースとして管理できるようにしてくれる。

# ArgoCD による管理

インフラストラクチャをコードで管理する GitOps のアプローチを採用しているため、MPI Operator のデプロイには ArgoCD を使用した。

## ArgoCD とは

[ArgoCD](https://argo-cd.readthedocs.io/) は Kubernetes 向けの宣言的 GitOps 継続的デリバリーツールである。
Git リポジトリを信頼できる唯一の情報源 (Single Source of Truth) として扱い、リポジトリの状態とクラスタの実際の状態を同期させる。

ArgoCD を使う利点:
- Git リポジトリで全てのリソースをバージョン管理できる
- 変更履歴が明確に残る
- ロールバックが容易
- クラスタの状態を宣言的に管理できる

## デプロイの構成

MPI Operator のデプロイには以下のような構成を採用した:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mpi-operator
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/kubeflow/mpi-operator
    targetRevision: v0.5.0
    path: deploy/v2beta1
  destination:
    server: https://kubernetes.default.svc
    namespace: mpi-operator
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

`syncPolicy.automated` を設定することで、Git リポジトリの変更が自動的にクラスタに反映されるようになる。
`prune: true` は Git から削除されたリソースをクラスタからも削除し、`selfHeal: true` はクラスタ側で手動で変更されたリソースを Git の状態に戻す。

# 実際の運用

## MPIJob の実行

MPI Operator をデプロイした後、実際に簡単な MPI プログラムを実行してみた。
以下は "Hello World" を各ノードから出力する例である:

```yaml
apiVersion: kubeflow.org/v2beta1
kind: MPIJob
metadata:
  name: mpi-hello-world
spec:
  slotsPerWorker: 1
  runPolicy:
    cleanPodPolicy: Running
  mpiReplicaSpecs:
    Launcher:
      replicas: 1
      template:
        spec:
          containers:
          - image: mpioperator/mpi-pi:latest
            name: mpi-launcher
            command:
            - mpirun
            - -n
            - "2"
            - /home/mpiuser/pi
    Worker:
      replicas: 2
      template:
        spec:
          containers:
          - image: mpioperator/mpi-pi:latest
            name: mpi-worker
```

この設定により、Launcher Pod が起動し、2つの Worker Pod 間で MPI プログラムが実行される。

## トラブルシューティング

実際に運用してみて、いくつか課題にぶつかった。

### ネットワークの設定

MPI は Pod 間での通信が必要なため、ネットワークポリシーの設定に注意が必要だった。
特に自宅環境では CNI (Container Network Interface) の選択も重要で、Calico や Cilium など、Pod 間通信が安定して動作する CNI を選ぶ必要がある。

### リソースの割り当て

並列計算は CPU やメモリを大量に消費するため、適切なリソース制限とリクエストの設定が重要である。
初期の設定では Worker Pod が OOMKilled になることがあったため、メモリの制限を調整した。

```yaml
resources:
  requests:
    memory: "2Gi"
    cpu: "1000m"
  limits:
    memory: "4Gi"
    cpu: "2000m"
```

## モニタリング

MPI ジョブの実行状況を把握するため、以下のようなモニタリングを行っている:

- Prometheus と Grafana によるメトリクスの可視化
- ジョブの完了時間やリソース使用率の追跡
- ログの集約 (Loki などを使用)

これにより、ジョブのパフォーマンスを分析し、継続的に改善できるようになった。

# 学んだこと

## GitOps のメリット

今回 ArgoCD を使って MPI Operator を管理したことで、GitOps のメリットを実感できた。
特に以下の点が良かった:

- 設定の変更履歴が Git のコミットログとして残る
- 変更のレビューが Pull Request で行える
- 問題が発生した際のロールバックが容易
- クラスタの状態を宣言的に管理できる

## Operator パターンの理解

MPI Operator を使うことで、Kubernetes の Operator パターンについても理解が深まった。
Operator は Kubernetes API を拡張し、複雑なアプリケーションの管理を自動化する。
特に MPI のような分散システムでは、手動での管理が困難なため、Operator の恩恵が大きい。

## 自宅クラスタの可能性

自宅の Kubernetes クラスタでも、適切なツールを使えば本格的な分散計算環境を構築できることが分かった。
クラウドと比べてコストを抑えながら、実験的なワークロードを実行できるのは大きなメリットである。

# 今後の展望

MPI Operator の導入を機に、以下のようなことにも挑戦してみたいと考えている:

- より大規模な並列計算ワークロードの実行
- 機械学習の分散学習への応用
- GPU を使った計算の実験
- 他の Kubeflow コンポーネントの導入

自宅クラスタはまだまだ拡張の余地があるので、継続的に改善していきたい。

# まとめ

自宅 Kubernetes クラスタに MPI Operator を ArgoCD で導入した経験をまとめた。
GitOps のアプローチを採用することで、インフラストラクチャの変更を安全かつ追跡可能な形で管理できるようになった。
また、MPI Operator により、Kubernetes 上で分散計算を実行できる環境を整えることができた。

自宅で実験環境を持つことで、新しい技術を気兼ねなく試せるのは大きなメリットである。
今後もクラスタを拡張しながら、様々なワークロードに挑戦していきたい。
