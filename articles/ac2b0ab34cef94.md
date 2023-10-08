---
title: "docker-composeによるmdBook執筆環境整備メモ"
emoji: "✏️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["markdown", "mdbook", "dockercompose", "docker"]
published: true
---

## TL;DR;

ホスト側で直接`cargo install mdbook`などをしたくない場合の環境構築の例。

https://github.com/junkor-1011/mdbook-example/tree/zenn_2023-10-08_2

の`work/book.toml`および`work/src`以下を適当に編集することでdocker-composeによる環境構築ができる。

## はじめに(動機、経緯)

一枚もののページではなく、階層構造などを持たせたそこそこのドキュメントを書きたくなることがある。そうした際でもMarkdownで記述できて、なおかつそれなりに拡張機能([mermaid](https://mermaid.js.org/)や[mathjax](https://www.mathjax.org/)など)も使えると便利そう、みたいなことを考えていた。

最近Rustなどをちょっとかじって遊んで見るなどしているときに、もろもろのドキュメントの作成に使われているツール(=mdBook)が結構良さそうだったので使い方を調べてみた次第。

mdBookは静的サイトジェネレーターの一種で、複数のmarkdownファイルをHTML形式にエクスポートできるRustベースのcliツール。

mdBookの公式ドキュメント:

https://rust-lang.github.io/mdBook/

mdBookの公式リポジトリ:

https://github.com/rust-lang/mdBook

で、mdBookおよびそのプラグインを導入していくにあたり、
普段からrustなどを使っているなら`cargo install mdbook`などでインストールすることもできるし、cargoコマンドが使えない人でもGitHubの[Releases](https://github.com/rust-lang/mdBook/releases)からバイナリをダウンロードして使うこともできる(公式の[Installation](https://rust-lang.github.io/mdBook/guide/installation.html)ページも参考)

が、なるべくホスト側の環境を汚したくない(=使い捨てにしたい)とか、
あまり詳細が分かってない人と環境を共有したいみたいなケースを色々考えると、

- `git clone`して`docker compose up -d`すれば使える
- 要らなくなったら使ったコンテナイメージを削除して終わり

みたいな形で環境構築を行うことも意味がありそうな気がしたので、そんな感じでやってみる。

## セットアップ

### (前提条件)

とりあえず、Dockerとdocker-composeが使えることが前提条件。

この記事を書くにあたっての手元の環境では以下のような感じ:

- Linux(WSL2)
  - WSL バージョン: 1.2.5.0
- Docker version 24.0.6
- Docker Compose version v2.21.0

### ディレクトリ構成

今回は下記のような感じで作る。

```tree
.
├── docker-compose.yaml
├── Dockerfile
├── README.md
└── work
    ├── book # 自動生成される静的アセット(HTML/JS/CSS/画像ファイル)
    ├── book.toml # 設定ファイル
    ├── mermaid-*.js # mdbook-mermaidにより自動配置されるファイル(gitignoreしておく)
    └── src
        ├── **.md # ドキュメントの本文(適当にディレクトリやファイルを追加していく)
        └── SUMMARY.md # ドキュメントの構成と目次
```

`work`以下をコンテナ側と共有する形になっており、実行時は`work/book.toml`(設定)および、`work/src`以下(ドキュメントの本体)を編集すればOK

### Dockerイメージの作成

作成するイメージについて説明しておく。

mdBook本体に加えて、mermaidを有効にするためにプラグインとして[mdbook-mermaid](https://github.com/badboy/mdbook-mermaid)も合わせてインストールを行う。

実行用のイメージを軽量にしたいので[alpine](https://hub.docker.com/_/alpine)をベースにイメージを作成する:

https://github.com/junkor-1011/mdbook-example/blob/zenn_2023-10-08_2/Dockerfile

alpine linuxは基本的にglibcが使えないので、インストールするバイナリはmuslとなっているもの(`*-x86_64-unknown-linux-musl`)にする必要がある。

今回は[mdBook](https://github.com/rust-lang/mdBook/releases), [mdbook-mermaid](https://github.com/badboy/mdbook-mermaid/releases)ともにtargetがmuslとなっているものがReleasesページに置いてあるのでそれを持ってきている。

また、`CMD`部分でコンテナのデフォルトの挙動を以下のようにしている:

1. mdbook-mermaidの有効化(`mdbook-mermaid install`)
   - 本当は初回だけ実行すれば良いが、面倒なので毎回実行している
2. `mdbook serve`を実行し、port番号3000でのプレビューサーバー起動と、変更を監視して自動ビルドを行う

### docker-compose

以下のような設定で記述している:

https://github.com/junkor-1011/mdbook-example/blob/zenn_2023-10-08_2/docker-compose.yaml

先述の通り、ホスト側の`work`ディレクトリ以下をコンテナ側にマウントしている。

また、実行ユーザーをホスト側のUID, GIDに合わせられるように上書きしている。
デフォルト値は1000で入れてしまっているので、異なる場合は[Compose内の環境変数](https://docs.docker.jp/compose/environment-variables.html)などを参考に変える。

同じく、プレビューサーバーのポート番号も変数`PORT`を設定することで3000以外に設定できる。

実行例:

```bash
# 例: port番号3001で起動する場合
PORT=3001 UID="$(id -u)" GID="$(id -g)" docker compose up -d

# 終了時
docker compose down
```

### book.toml

設定ファイル: `work/book.toml`の編集。

基本的には公式ドキュメントの[Configuration](https://rust-lang.github.io/mdBook/format/configuration/index.html)などを参照しながら編集していく。

追加機能なども入れず、ごくシンプルな設定で始める場合は[Creating a Book - Anatomy of a book](https://rust-lang.github.io/mdBook/guide/creating.html?highlight=book.toml#booktoml)に記載されているように、

```toml:book.toml
[book]
title = "My First Book" # 適当に編集する
```

くらいでもいいらしい。

とりあえず、`mdbook init`コマンドで自動生成される設定だと

```toml:book.toml
[book]
authors = []
language = "en"
multilingual = false
src = "src"
```

みたいな感じになっている。

今回はmathjaxを有効化してみたかったので、[Mathjax Support](https://rust-lang.github.io/mdBook/format/mathjax.html)を参考に、

```toml:book.toml(抜粋)
[output.html]
mathjax-support = true
```

を追記している。

また、[mdbook-mermaid](https://github.com/badboy/mdbook-mermaid)を有効化するため、

```toml:book.toml(抜粋)
[preprocessor]

[preprocessor.mermaid]
command = "mdbook-mermaid"

[output]

[output.html]
additional-js = ["mermaid.min.js", "mermaid-init.js"]
```

を追記している(これは`mdbook-mermaid install`コマンドで自動で追記される)。
参考: [mdbook-mermaid installation](https://github.com/badboy/mdbook-mermaid#installation)

最後に、mdbookでRustのコードを書くとplayground連携によりその場でコード実行が出来るが、要はリモートにコードを送信していることになるので環境によっては望ましくない場合がある。
そのような場合、公式ドキュメントの[Rust Playground](https://rust-lang.github.io/mdBook/format/mdbook.html?highlight=ANCHOR#rust-playground)を参考に、

```toml:book.toml(抜粋)
[output.html.playground]
runnable = false
```

などとすればドキュメント全体で一括して無効化することができる。

### src/SUMMARY.md

`work/src/SUMMARY.md`を編集することでドキュメントの構造と目次を作成できる。

公式ドキュメントに詳細と、mdbookのドキュメントそのものの`SUMMARY.md`の例が示されていてわかりやすいので基本はそちらを参照：

https://rust-lang.github.io/mdBook/format/summary.html

ざっくりとだけ書くと、

- 本文に相当するMarkdownファイルのリンク(`[見出し](相対パス)`みたいなやつ)を`SUMMARY.md`に記述するとbookにその参照先のMarkdownファイルの内容が1ページ分として取り込まれる
  - リンク先のファイルが存在しないと自動で生成してくれる(地味に便利)
- ↑のリンクをリスト表示すると自動でナンバリングされる
  - ネストの深さも反映される

みたいな感じ。

この辺は[公式](https://rust-lang.github.io/mdBook/format/summary.html#example)の記載例と実際の目次を見比べるのが早い。

後はリンク先で指定した各markdownファイルを編集していけばOK。

## (補足)

### 類似ツール

類似というか競合?するようなものの例としては、

- [MkDocs](https://www.mkdocs.org/)
  - こちらはPythonベースで、同じく複数のmarkdownをソースとする静的サイトジェネレーター。
    こちらも結構良さそうな雰囲気はあるが使ったことはない
- [Sphinx](https://www.sphinx-doc.org/ja/master/)
  - 基本は[reStructuredText](https://www.sphinx-doc.org/ja/master/usage/restructuredtext/basics.html)で記述するが、[MyST-Parser](https://myst-parser.readthedocs.io/en/latest/)を導入することで[Markdownでも書けるらしい](https://www.sphinx-doc.org/ja/master/usage/markdown.html)(これもまだ自分では試せていない)
- [gitbook-cli](https://github.com/GitbookIO/gitbook-cli)
  - CLIツールは開発が終了してdeprecatedになっている旨が書かれている

などが少なくともある。

探しきれてないだけで他にもありそうな気もするし、いずれにしても自分で試せていないので、他の類似ツールとmdBookとの比較検討は行っていない。
