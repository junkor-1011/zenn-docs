---
title: "vite + react + storybookにlinariaを導入してみる"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vite", "react", "storybook", "linaria"]
published: false
---

軽量な React 環境を構築したいモチベーションがあり、
ゼロランタイムな CSS in JS を実現できる[linaria](https://github.com/callstack/linaria)を vite でセットアップした React 環境に導入してみたのでメモしておく。

なお、linaria それ自体の詳細な説明やその他の css ライブラリとの比較などについては特に触れない。

## 0. 試行時の環境および各ライブラリのバージョンについて

特に主要な使用ライブラリのバージョンが上がるとセットアップ方法が変わってくることが多いので:

- Linux(Fedora38)
- Node.js v18.16.0
  - パッケージマネージャー: pnpm@8.6.1

## 1. vite による React 環境のセットアップ

[公式の Getting Start](https://vitejs.dev/guide/#scaffolding-your-first-vite-project)
を参照すると、`react-swc-ts`というテンプレートがあるので今回はこれを使う。
(`react-ts`と比較すると、dev モードのときに[babel](https://babeljs.io/)ではなく[swc](https://swc.rs/)を使うため高速に動作するとのこと。詳細は[vite-plugin-react-swc](https://github.com/vitejs/vite-plugin-react-swc)を参照)

公式リポジトリの[create-vite](https://github.com/vitejs/vite/tree/main/packages/create-vite#create-vite)の記述を参照しつつ、以下のようにしてプロジェクトを立ち上げる。

```bash
# 今回は`linaria-sample`という名前で、プロジェクトを作成する
pnpm create vite linaria-sample --template react-swc-ts

# 作成したプロジェクトに移動
cd linaria-sample

# ライブラリのインストールを実行
pnpm install

# (微妙にライブラリが古かったりするので、必要なら適宜アップデートする)
pnpm up <package名> # 個別にアップグレード
pnpm up --latest # 一括してアップグレード
```

以上でとりあえず Vite + React 環境が作成出来ている。

参考と再現性のため、ここまででのファイル構成概要と`package.json`の中身を載せておく:

```sh:ファイル構成
tree --gitignore -a -I '.git'
# .
# ├── .eslintrc.cjs
# ├── .gitignore
# ├── index.html
# ├── package.json
# ├── pnpm-lock.yaml
# ├── public
# │   └── vite.svg
# ├── src
# │   ├── App.css
# │   ├── App.tsx
# │   ├── assets
# │   │   └── react.svg
# │   ├── index.css
# │   ├── main.tsx
# │   └── vite-env.d.ts
# ├── tsconfig.json
# ├── tsconfig.node.json
# └── vite.config.ts
#
# 4 directories, 15 files
```

```json:package.json
{
  "name": "linaria-sample",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "lint": "eslint src --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.8",
    "@types/react-dom": "^18.2.4",
    "@typescript-eslint/eslint-plugin": "^5.59.8",
    "@typescript-eslint/parser": "^5.59.8",
    "@vitejs/plugin-react-swc": "^3.3.2",
    "eslint": "^8.42.0",
    "eslint-plugin-react-hooks": "^4.6.0",
    "eslint-plugin-react-refresh": "^0.4.1",
    "typescript": "^5.0.4",
    "vite": "^4.3.9"
  }
}
```

また、プロジェクトのビルドと開発用サーバー起動はそれぞれ以下のようにしてできる:

```bash
# 開発用サーバーの起動
pnpm dev

# プロジェクトのビルド
pnpm build
# →dist以下にアセットが格納される
```
