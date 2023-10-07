---
title: "docker-composeによるmdBook執筆環境整備メモ"
emoji: "✏️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["markdown", "mdbook", "dockercompose", "docker"]
published: false
---

## TL;DR;

ホスト側で直接`cargo install mdbook`などをしたくない場合の環境構築の例。

https://github.com/junkor-1011/mdbook-example/tree/zenn_2023-10-07_2

の`work/book.toml`および`work/src`以下を適当に編集することでdocker-composeによる環境構築ができる。

## はじめに(動機、経緯)

一枚もののページではなく、階層構造などを持たせたそこそこのドキュメントを書きたくなることがある。そうした際でもMarkdownで記述できて、なおかつそれなりに拡張機能([mermaid](https://mermaid.js.org/)や[mathjax](https://www.mathjax.org/)など)も使えると便利そう、みたいなことを考えていた。

最近Rustなどをちょっとかじって遊んで見るなどしているときに、もろもろのドキュメントの作成に使われているツール(=mdBook)が結構良さそうだったので使い方を調べてみた次第。

mdBookは静的サイトジェネレーターの一種で、複数のmarkdownファイルをHTML形式にエクスポートできるRustベースのcliツール。

mdBookの公式ドキュメント:

https://rust-lang.github.io/mdBook/

mdBookの公式リポジトリ:

https://github.com/rust-lang/mdBook

類似というか競合?するようなものの例としては、

- [MkDocs](https://www.mkdocs.org/)
  - こちらはPythonベースで、同じく複数のmarkdownをソースとする静的サイトジェネレーター。
    こちらも結構良さそうな雰囲気はあるが使ったことはない
- [Sphinx](https://www.sphinx-doc.org/ja/master/)
  - 基本は[reStructuredText](https://www.sphinx-doc.org/ja/master/usage/restructuredtext/basics.html)で記述するが、[MyST-Parser](https://myst-parser.readthedocs.io/en/latest/)を導入することで[Markdownでも書けるらしい](https://www.sphinx-doc.org/ja/master/usage/markdown.html)(これもまだ自分では試せていない)
- [gitbook-cli](https://github.com/GitbookIO/gitbook-cli)
  - CLIツールは開発が終了してdeprecatedになっている旨が書かれている

などが少なくともある。

(探しきれてないだけで他にもありそうな気もするし、他の類似ツールとmdBookとの比較検討は別にここでは行わず、あくまでmdBookを使う場合の一例としてメモする)

で、mdBookおよびそのプラグインを導入していくにあたり、
普段からrustなどを使っているなら`cargo install mdbook`などでインストールすることもできるし、
cargoコマンドが使えない人でもGitHubの[Releases](https://github.com/rust-lang/mdBook/releases)からバイナリをダウンロードして使うこともできる(公式の[Installation](https://rust-lang.github.io/mdBook/guide/installation.html)ページも参考)

が、なるべくホスト側の環境を汚したくない(=使い捨てにしたい)とか、
あまり詳細が分かってない人と環境を共有したいとか色々考えると、

- `git clone`して`docker compose up -d`すれば使える
- 要らなくなったら使ったコンテナイメージを削除して終わり

みたいな形で環境構築を行うことも意味がありそうな気がしたので、そんな感じでやってみる。
