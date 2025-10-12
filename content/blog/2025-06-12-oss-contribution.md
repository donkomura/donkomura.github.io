+++
title = "やってみて分かった「OSS への貢献」"
slug = "oss-contribution"
date = 2025-06-13

[taxonomies]
categories = ["記事"]
tags = ["rust", "oss"]

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
cover = "/assets/cover.png"
+++

[以前のブログ](@/blog/2025-03-16-consistent-hashing-in-rust.md) で Rust を書いて以降、 Rust に興味を持ち始めました。
これまでは業務でも趣味でも Rust をあまり書いてこなかったので、初心者と言っても良いでしょう [^1]。

この記事では、そんな Rust 初心者がどのように Rust Linter として有名な rust-clippy に貢献したのかを紹介します。
実際に貢献してみて分かった「OSS へ貢献する」とは何かについてもまとめます。
これから Rust に限らず OSS へ貢献したいと少しでも思っている人が動き出すきっかけになると嬉しいです。

# rust-clippy に貢献しようと思ったきっかけ

[rust-clippy](https://github.com/rust-lang/rust-clippy) は Rust の linter のひとつで、デファクトスタンダードといってよい程には有名です。
最近では VS Code でコーディングすることが多い(?) そうなので、VS Code を例に取ると、 rust-analyzer による Quick Fix や下線で変更すべき点を教えてくれることがあると思います。
これらの裏では rust-clippy が動いていることが多いです。

私が rust-clippy に貢献しようと思った理由は大きく２つあります。
ひとつは、もともと Rust に興味があったということ。
これまで研究室に籠ってストレージを書いていた身なので、Linux に Rust が採用されているなどの低レイヤーで Rust が活躍していることには気づいていて、徐々に興味を持っていました。
大学では結局触ることなく、それ以降も趣味程度にしか触っていなかったので、きっかけがあれば本格的に触りたいと思っていました。
また、直近でプログラミング言語や型について勉強にしていたので、その文脈で興味を持ったという経緯もあります。

理由の二つめは、日本人が rust-clippy に貢献しているところをよくみるようになったからです。
特に [lapla さんのブログ](https://lapla.dev/posts/clippy/) で興味を持ちました。
単純に50以上のパッチを送っていたのもすごいのですが、 rust-clippy のコミュニティが新しい参画者に対してウェルカムな姿勢であったことが伺えたので、Rust という領域に絞ってコミットするのであれば、rust-clippy を選ぼうと思いました。

一応やり続けるための理由も考えましたが、そこまで意識してません。
- システムソフトウェアに関わる仕事を将来的にしたい
- インフラのことをやっており、コードを定常的に書かない

# 実際にやったこと

[lapla さんのブログ](https://lapla.dev/posts/clippy/) に記載されている内容とほぼ同じです。

- ドキュメントを読む
    - [CONTRIBUTING.md](https://github.com/rust-lang/rust-clippy/blob/master/CONTRIBUTING.md)
    - [Clippy Book](https://doc.rust-lang.org/clippy/)
    - [Clippy Lints](https://rust-lang.github.io/rust-clippy/master/index.html)
- 最近提出された issue をみる
    - 手を付けられそうなやつとそうでないものを分類する
- 手を付けてみて、実装できそうなら自分をアサインする

Issue については `good-first-issue` ラベルのあるものを一通り見たのですが既にアサインされているものばかりでした。
なので、以下の点に気を付けながら issue を分類していきました。

- 再現手順があるか
- Label が適切か
    - `l-false-positive` / `C-bug` / `l-suggestion-causes-error`

経験則にはなりますが、自明な false positive は手を付けやすいです。
再現コードが記載されていて、変更する箇所も明確なことが多いからです。

`ICE` は緊急性の高い場合もあるためひとまず手を付けないでおきました。
ちなみに、緊急で修正が必要な場合は Rust 本体にバックポートされるようになっています。

実際に何をやったかというと、直近の数週間でドキュメントの修正やいつかのバグ修正、 issue のトリアージを行いました。
細かいバグの内容などはまた別の機会にまとめようと思います。

- [関わった Issue](https://github.com/rust-lang/rust-clippy/issues?q=is%3Aissue%20involves%3Adonkomura)
- [Pull request](https://github.com/rust-lang/rust-clippy/pulls?q=is%3Apr+author%3Adonkomura)

余談ですが、週間のコミット数で全体４位になることもありました。

{{ figure(src="/assets/2025-06-12-commits.jpg", alt="commit ranking", caption="") }}

また、 rust-clippy に機能変更が加わると This Week in Rust に掲載されます。
メリットは特にありませんが、少し嬉しいですね。

- [This Week in Rust 602](https://this-week-in-rust.org/blog/2025/06/04/this-week-in-rust-602/)

# 貢献の前後で変わったこと

rust-clippy に貢献し始めて、今まで持っていた OSS への貢献という概念が大きく変わったと感じました。
これまでは、OSS への貢献は 高いスキルを持つエンジニアが行うものという印象でしたが、実際に OSS にコミットするようになってからは誰でも気軽に参加できるコミュニティ活動と思うようになりました。
コミュニティ活動というのは実態通りの表現なのですが、道端のゴミを拾ったり、靴を並べたりとそんな感覚に近いです。
それと同時に「趣味」という表現もまた適切でしょう。
私はボルダリングが趣味なのですが、それと同等の感覚で rust-clippy へのコミットも行っています。
つまりスキルは後付けで良くて（ただし全く無ければ何もできないのでその場合を除く）、
やろうと思ったか、やる意思が芽生えているのかの方が重要だと思うようになりました。
靴が散らかっていることに気づくことの方が、綺麗に靴を並べる技術よりもよほど大事なのです。

実際、リポジトリの issue や実装を見ていると、よくある考慮もれがあったり軽微なバグが多くあったりと、思ったよりも修正箇所が多いことに気づきます。
そのように思ってからは、靴が散らかった玄関を片づけるかのようにできる issue から取り掛かるようになりました。
まずは靴を並べるところから始めて、後から綺麗に並べれるようになれば良いという考えが湧いたからです。

# まとめ

ポエムチックなことを多く書いてしまったのですが、要するに迷ったらまずはやってみれば良いと思います。
しくじったと思ったなら、メンテナーなどに相談すれば必ず助けになってくれるはずです。
もしそうでないのなら、別のコミュニティに移れば良いだけの話です。
みんな仕事でやっている分けではないので、気軽に一歩を踏み出して良いでしょう。

しばらくは rust-clippy を触ろうと思います。
次何をやるかなのですが、未定です。
Rust コンパイラを触るからそれとも元から興味のあったクラウドネイティブな技術を触るかもしれません。
将来の自分に運命を委ねます [^2] 。

ともあれ、
楽しんで！

[^1]: [試しに触った程度でしか書いてない](https://github.com/donkomura?tab=repositories&q=&type=&language=rust&sort=)

[^2]: [HAMA](https://www.nicovideo.jp/user/123621495)
