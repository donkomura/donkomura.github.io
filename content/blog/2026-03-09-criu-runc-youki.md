+++
title = "Youki, runc における CRIU によるチェックポイント・リストアの仕組み"
slug = "criu-checkpoint-restore-in-youki-runc"
date = 2026-03-09

[taxonomies]
categories = ["記事"]
tags = ["container", "checkpoint/restore", "youki", "runc"]

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
cover = "/assets/cover.png"
+++

[Difference between the checkpoint command in runc and youki](https://github.com/youki-dev/youki/issues/3394) を実装する前に、 CRIU の役割と youki や runc での実装方法について調べたことをまとめる。

## CRIU, rust-criu とは
[CRIU](http://criu.org/)（Checkpoint/Restore In Userspace）とは、Linux上で稼働しているコンテナや単一のアプリケーションのプロセスを一時停止（フリーズ）し、その状態をディスクに保存（チェックポイント）して、後から完全に同じ状態から再開（リストア）できるようにするためのソフトウェアである。アプリケーションのプロセスで使われるメモリやディスクの状態だけでなく、TCPコネクションや cgroup 構成など幅広いリソースをチェックポイント・リストアすることができる。

[Rust-criu](https://github.com/checkpoint-restore/rust-criu) はこの CRIU を Rust で利用できるようにした実装で、似たようなものとして [go-criu](https://github.com/checkpoint-restore/go-criu) という Go 言語のバインディングも存在する。
Rust-criu が誕生したのは youki でチェックポイント・リストアを実装するためであったことが [issue](https://github.com/youki-dev/youki/issues/142) （実際の提案 [issue](https://github.com/checkpoint-restore/criu/issues/1722)）から読み取れる。

ここで登場した [Youki](https://github.com/youki-dev/youki) は Rust で実装されたコンテナランタイムである。
Youki は [OCI runtime-spec](https://github.com/opencontainers/runtime-spec) というコンテナランタイムの標準仕様に従っており、 [runc](https://github.com/opencontainers/runc) もまたこの標準に従った実装である（こちらは Go で書かれている）。
## Runc における CRIU の使い方
CLIから使うことができる。
```bash
runc checkpoint [option ...] container-id
runc restore [option ...] container-id
```

このコマンドの裏では CRIU が動いている訳だが、CRIU の起動方法にもいくつかあり、 runc では `SWRK mode` を使っている。
SWRK mode では `fork() + exec()` コマンドで CRIU を起動し、UNIXドメインソケットを通じて起動した親プロセスと通信する。
つまり、 runc で CRIU を SWRK mode で呼び出すことで、 criu-go を使って runc からチェックポイントすることができるというわけである。
なお、ここでチェックポイントを行うために必要な runc から CRIU への指示は RPC で実行される。
Protobuf で定義されているので、実行の仕組みを踏まえてこれを見れば扱い方が想像しやすい。
Runc からチェックポイントの手続きを RPC で CRIU に指示した後、CRIU で処理が完了するとUNIXドメインソケットを通じて完了通知を受け取る。

Rust-criu でも RPC 定義が散見されるため基本的にはこの SWRK mode で CRIU を起動してランタイムから RPC を叩いて CRIU からの通知を受け取るという実装が一般的なのだろうと推察する。

## Youki でのチェックポイントの取得
Youki では `checkpointt` というサブコマンドでチェックポイントを行うことができる。

適当なコンテナを立てて動かした上でチェックポイントを取ってみる。
```bash
# https://youki-dev.github.io/youki/user/basic_usage.html
$ mkdir -p tutorial/rootfs
$ cd tutorial
$ docker export $(docker create busybox) | tar -C rootfs -xvf -
```
`tutorial/config.json` の process.args を編集して sleep を適当な長さ入れておき、チェックポイントを手動で取るための猶予を作る。
```json
"process": {
  ...
  "args": [
    "sleep", "60"
  ],
  ...
}
```
Youki のルートディレクトリに戻ってからコンテナを作成・チェックポイントを取得する。チェックポイントは `checkpoint` というディレクトリに保存されるのであらかじめ作っておく。
```
$ sudo ./youki create -b tutorial tutorial_container
$ sudo ./youki state tutorial_container
$ sudo ./youki start tutorial_container

# checkpoint が保存される場所
$ mkdir -p checkpoint
# checkpoint を取得する
$ sudo ./youki checkpointt --shell-job tutorial_container
$ ls checkpoint/
cgroup.img        files.img      ip6tables-10.img    netdev-10.img    pstree.img     stats-dump               tmpfs-dev-79.tar.gz.img
core-1.img        fs-1.img       ipcns-var-11.img    netns-10.img     route-10.img   timens-0.img             tmpfs-dev-80.tar.gz.img
descriptors.json  ids-1.img      iptables-10.img     nftables-10.img  route6-10.img  tmpfs-dev-71.tar.gz.img  tmpfs-dev-81.tar.gz.img
dump.log          ifaddr-10.img  mm-1.img            pagemap-1.img    rule-10.img    tmpfs-dev-76.tar.gz.img  tty-info.img
fdinfo-2.img      inventory.img  mountpoints-13.img  pages-1.img      seccomp.img    tmpfs-dev-78.tar.gz.img  utsns-12.img
```

`checkpoint` を見ると様々なファイルが確認できる。これを CRIU image file という。フォーマット等の仕様については[公式ページ](https://criu.org/Images) を参照のこと。
```
$ file checkpoint/cgroup.img
checkpoint/cgroup.img: CRIU image file v1.1
```
