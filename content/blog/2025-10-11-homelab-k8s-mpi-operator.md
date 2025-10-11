+++
title = "自宅 Kubernetes クラスタに MPI Operator を導入した話"
slug = "homelab-k8s-mpi-operator"
date = 2025-10-11

[taxonomies]
categories = ["記事"]
tags = ["kubernetes", "argocd", "homelab"]

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

自宅で Kubernetes クラスタを運用していて、最近 [MPI Operator](https://github.com/kubeflow/mpi-operator) などのコンポーネントを ArgoCD で整備したのでまとめてみる。

# 背景

自宅で Kubernetes クラスタを運用している理由はいくつかある。
- クラウドネイティブな技術を実践的に学びたい
- 実験的なワークロードを気兼ねなく動かせる環境が欲しい
- 作ったり壊したりを気軽にできる計算設備があると嬉しい

もともとHPC出身なこともあったのと、近年の Kubernetes での AI ワークロードの解像度が低かったので家の資源を活用しつつ実験できる環境を整えようと考えた。
そこでとりあえず MPI (Message Passing Interface) を使った並列計算を Kubernetes 上で実行できるようにした。

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

## デプロイの構成

MPI Operator のデプロイには以下のような構成を採用した。
ArgoCD Application と Kustomize を組み合わせて管理している:

```yaml
# argocd/applications/mpi-operator.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mpi-operator
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:donkomura/donkooolab.git
    targetRevision: main
    path: argocd/apps/mpi-operator
  destination:
    server: https://kubernetes.default.svc
    namespace: mpi-operator
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
      - Replace=true
      - SkipDryRunOnMissingResource=true
```

実際のマニフェストは Kustomize で管理している:

```yaml
# argocd/apps/mpi-operator/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - https://raw.githubusercontent.com/kubeflow/mpi-operator/v0.6.0/deploy/v2beta1/mpi-operator.yaml

commonAnnotations:
  argocd.argoproj.io/sync-options: ServerSideApply=true

patches:
  - target:
      kind: CustomResourceDefinition
    patch: |-
      - op: add
        path: /metadata/annotations/argocd.argoproj.io~1sync-wave
        value: "-1"
```

Sync Wave で CRD を先にデプロイしてからリソースをデプロイするようにすることで CRD に更新があっても sync がこけない。

## 他に追加したコンポーネント

MPI Operator と合わせて、クラスタ全体の運用に必要な以下のコンポーネントも ArgoCD で管理している。

### Argo Workflows

[Argo Workflows](https://github.com/argoproj/argo-workflows) は Kubernetes ネイティブなワークフローエンジンである。
複雑な処理を DAG (Directed Acyclic Graph) として定義し、並列実行や条件分岐を含むワークフローを実行できる。

MPI Operator が単一の並列計算ジョブを実行するのに対し、Argo Workflows は複数のステップから成るパイプラインを構築できる。
例えば、データの前処理 → MPI ジョブの実行 → 結果の後処理といった一連の流れを一つのワークフローとして定義できる。

### kube-prometheus-stack

[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) は Prometheus Operator、Grafana、Alertmanager などを含む包括的なモニタリングスタックである。

導入している主な機能:
- **Prometheus**: メトリクスの収集と保存
- **Grafana**: メトリクスの可視化
- **Alertmanager**: アラートの管理と通知

それぞれに Ingress を設定し、cert-manager と連携して Let's Encrypt の証明書を自動取得している。
MPI ジョブのリソース使用状況やパフォーマンスを追跡するために活用している。

### cert-manager と external-dns

HTTPS 対応と DNS 管理を自動化するために以下も導入している:

- **[cert-manager](https://cert-manager.io/)**: Let's Encrypt の証明書を自動取得・更新
- **[external-dns](https://github.com/kubernetes-sigs/external-dns)**: Kubernetes の Ingress リソースに基づいて Cloudflare の DNS レコードを自動管理

これらにより、新しいサービスをデプロイする際に手動で証明書を取得したり DNS レコードを設定したりする必要がなくなった。
ingress-nginx に annotation で cluster-issuer を指定しておくと、SSL 終端してくれるので楽でよい。
ドメインは Cloudflare で管理している。

### ingress-nginx

外部からクラスタへのアクセスを管理するために [ingress-nginx](https://github.com/kubernetes/ingress-nginx) を使用している。
Prometheus、Grafana、その他のサービスについて ingress を生やすと external-dns がレコードを生やしてくれるので、ドメインでアクセスできるようになっている。
Tailscale などの VPN でも繋げば外出時にもアクセスできる。

# まとめ

自宅 Kubernetes クラスタに MPI Operator を ArgoCD で導入した経験をまとめた。
GitOps のアプローチを採用することで、インフラストラクチャの変更を安全かつ追跡可能な形で管理できるようになった。
また、MPI Operator により Kubernetes 上で分散計算を実行できる環境を整えることができた。
[Kubeflow](https://www.kubeflow.org/) などのモダンな AI ワークフローを構築することも考えたが要件に見合う計算資源は無かったため、今回は断念した。

雑にバッチジョブも投げられるようになったし、簡単な学習なら回せるようになったので遊んでみることにする。
