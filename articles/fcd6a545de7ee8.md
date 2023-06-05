---
title: "vite + react + storybookにlinariaを導入してみる"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vite", "react", "storybook", "linaria"]
published: true
---

軽量な React 環境を構築したいモチベーションがあり、
ゼロランタイムな CSS in JS を実現できる[linaria](https://github.com/callstack/linaria)を vite でセットアップした React 環境に導入してみたのでメモしておく。
(なお、後述の通り特別な手順などは必要なく、かなり簡単にセットアップできた)

なお、linaria それ自体の詳細な説明やその他の css ライブラリとの比較などについてはここでは特に触れない。

## 0. 試行時の環境および各主要ライブラリのバージョン

特に主要な使用ライブラリのバージョンが上がるとセットアップ方法が変わってくることが多いので、
再現性のため一応メモしておく

- OS: Linux(Fedora38)
  - アーキテクチャ: x86_64
- Node.js v18.16.0
  - パッケージマネージャー: pnpm@8.6.1
- vite@4.3.9
- React@18.2.0
- storybook@7.0.18
- linaria
  - "@linaria/babel-preset": "^4.4.5",
  - "@linaria/core": "^4.2.10",
  - "@linaria/react": "^4.3.8",
  - "@linaria/vite": "^4.2.11",

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

## 2. linaria のインストール

[公式リポジトリ](https://github.com/callstack/linaria)にインストール方法が記述されているため、その通りにやれば OK

まずは構成に関わらず`@linaria/core`, `@linaria/react`, `@linaria/babel-preset`が必須なのでこれらをインストールする。

```bash
pnpm add @linaria/{core,react,babel-preset}
# dependencies:
# + @linaria/babel-preset 4.4.5
# + @linaria/core 4.2.10
# + @linaria/react 4.3.8
```

後は使用する bundler やフレームワークに合わせてオプションでインストール、セットアップが必要、といった感じ。

今回は`vite`を使うので[BUNDLERS_INTEGRATION - vite](https://github.com/callstack/linaria/blob/master/docs/BUNDLERS_INTEGRATION.md#vite)を参照し、vite 用のプラグインをインストールする。

```bash
# vite用プラグインのインストール
pnpm add -D @linaria/vite
# devDependencies:
# + @linaria/vite 4.2.11
```

また、`vite.config.ts`を編集して linaria 用プラグインを有効にする。

```diff:vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react-swc'
+ import linaria from '@linaria/vite'

// https://vitejs.dev/config/
export default defineConfig({
- plugins: [react()],
+ plugins: [
+   react(),
+   linaria(),
+ ],
})
```

これで linaria のセットアップは完了している。

確認のため、`src/App.tsx`の中身を適当にいじってみる。
(linaria の記載方法は[公式リポジトリの Syntax](https://github.com/callstack/linaria#syntax)などを参照)

```diff:src/App.tsx
import { useState } from 'react'
import reactLogo from './assets/react.svg'
import viteLogo from '/vite.svg'
+ import { css } from '@linaria/core'
import './App.css'

function App() {
  const [count, setCount] = useState(0)

  return (
    <>
      <div>
        <a href="https://vitejs.dev" target="_blank">
          <img src={viteLogo} className="logo" alt="Vite logo" />
        </a>
        <a href="https://react.dev" target="_blank">
          <img src={reactLogo} className="logo react" alt="React logo" />
        </a>
      </div>
-     <h1>Vite + React</h1>
+     <h1 className={css`color: blue;`}>
+       Vite + React
+     </h1>
      <div className="card">
        <button onClick={() => setCount((count) => count + 1)}>
          count is {count}
        </button>
        <p>
          Edit <code>src/App.tsx</code> and save to test HMR
        </p>
      </div>
      <p className="read-the-docs">
        Click on the Vite and React logos to learn more
      </p>
    </>
  )
}

export default App
```

上記の例では h1 タグの色を青に変えている。

`pnpm dev`および、`pnpm build`をそれぞれ行うことで、
確かに linaria によって色を変えられていることを確認できる。

before:
![react-before](https://storage.googleapis.com/zenn-user-upload/74eb27457e7b-20230605.png)

after:
![react-after](https://storage.googleapis.com/zenn-user-upload/233e5c5d3b75-20230605.png)

なお、余談だが vscode を使う場合`styled-components`や`emotion`と同様に、
[vscode-styled-components](https://marketplace.visualstudio.com/items?itemName=styled-components.vscode-styled-components)を使うと syntax highlight を効かせたり良い感じに補完を有効化出来たりする。

## 2. storybook における linaria の有効化

storybook@7.0.xをインストールし、そこで linaria を有効化する。

まずは storybook のインストールを行う。

```bash
# package.jsonがある場所で以下を実行
pnpm dlx storybook@latest init

# 自動でパッケージをインストールするか聞かれるので'Y'を選択する
```

ここで、React を使っていること及び vite を bundler に使っていることを自動検知してセットアップしてくれる。

storybook の起動およびビルドは以下のようにしてできる

```bash
# 起動
pnpm storybook

# 静的アセットのビルド
pnpm build-storybook
# →storybook-staticにアセットが格納
```

なお、storybook のビルドを行う場合は`.gitignore`に`storybook-static`を追記しておくと良い。

```diff:.gitignore
+ storybook-static
```

ここまででファイル構成は以下のような感じになっている：

```sh
tree --gitignore -a -C -I '.git'
# .
# ├── .eslintrc.cjs
# ├── .gitignore
# ├── .storybook # ★追加
# │   ├── main.ts
# │   └── preview.ts
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
# │   ├── stories # ★追加
# │   │   ├── Button.stories.ts
# │   │   ├── Button.tsx
# │   │   ├── Header.stories.ts
# │   │   ├── Header.tsx
# │   │   ├── Introduction.mdx
# │   │   ├── Page.stories.ts
# │   │   ├── Page.tsx
# │   │   ├── assets
# │   │   │   ├── code-brackets.svg
# │   │   │   ├── colors.svg
# │   │   │   ├── comments.svg
# │   │   │   ├── direction.svg
# │   │   │   ├── flow.svg
# │   │   │   ├── plugin.svg
# │   │   │   ├── repo.svg
# │   │   │   └── stackalt.svg
# │   │   ├── button.css
# │   │   ├── header.css
# │   │   └── page.css
# │   └── vite-env.d.ts
# ├── tsconfig.json
# ├── tsconfig.node.json
# └── vite.config.ts
#
# 7 directories, 35 files
```

また、再現性のため、パッケージが自動インストールされて更新された`package.json`も掲載しておく:

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
    "preview": "vite preview",
    "storybook": "storybook dev -p 6006",
    "build-storybook": "storybook build"
  },
  "dependencies": {
    "@linaria/babel-preset": "^4.4.5",
    "@linaria/core": "^4.2.10",
    "@linaria/react": "^4.3.8",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@linaria/vite": "^4.2.11",
    "@storybook/addon-essentials": "^7.0.18",
    "@storybook/addon-interactions": "^7.0.18",
    "@storybook/addon-links": "^7.0.18",
    "@storybook/blocks": "^7.0.18",
    "@storybook/react": "^7.0.18",
    "@storybook/react-vite": "^7.0.18",
    "@storybook/testing-library": "^0.0.14-next.2",
    "@types/react": "^18.2.8",
    "@types/react-dom": "^18.2.4",
    "@typescript-eslint/eslint-plugin": "^5.59.8",
    "@typescript-eslint/parser": "^5.59.8",
    "@vitejs/plugin-react-swc": "^3.3.2",
    "eslint": "^8.42.0",
    "eslint-plugin-react-hooks": "^4.6.0",
    "eslint-plugin-react-refresh": "^0.4.1",
    "eslint-plugin-storybook": "^0.6.12",
    "prop-types": "^15.8.1",
    "storybook": "^7.0.18",
    "typescript": "^5.0.4",
    "vite": "^4.3.9"
  }
}
```

storybook でも`linaria`が使えることを確認するため、`src/stories/Page.tsx`を適当に編集してみる

```diff:src/stories/Page.tsx
import React from 'react';
+ import { css } from '@linaria/core';

import { Header } from './Header';
import './page.css';

type User = {
  name: string;
};

export const Page: React.FC = () => {
  const [user, setUser] = React.useState<User>();

  return (
    <article>
      <Header
        user={user}
        onLogin={() => setUser({ name: 'Jane Doe' })}
        onLogout={() => setUser(undefined)}
        onCreateAccount={() => setUser({ name: 'Jane Doe' })}
      />

      <section className="storybook-page">
-       <h2>Pages in Storybook</h2>
+       <h2 className={css`color: green;`}>
+         Pages in Storybook
+       </h2>
-       <p>
+       <p className={css`font-weight: bold;`}>
          We recommend building UIs with a{' '}
          <a href="https://componentdriven.org" target="_blank" rel="noopener noreferrer">
            <strong>component-driven</strong>
          </a>{' '}
          process starting with atomic components and ending with pages.
        </p>
        <p>
          Render pages with mock data. This makes it easy to build and review page states without
          needing to navigate to them in your app. Here are some handy patterns for managing page
          data in Storybook:
        </p>
        <ul>
          <li>
            Use a higher-level connected component. Storybook helps you compose such data from the
            "args" of child component stories
          </li>
          <li>
            Assemble data in the page component from your services. You can mock these services out
            using Storybook.
          </li>
        </ul>
        <p>
          Get a guided tutorial on component-driven development at{' '}
          <a href="https://storybook.js.org/tutorials/" target="_blank" rel="noopener noreferrer">
            Storybook tutorials
          </a>
          . Read more in the{' '}
          <a href="https://storybook.js.org/docs" target="_blank" rel="noopener noreferrer">
            docs
          </a>
          .
        </p>
        <div className="tip-wrapper">
          <span className="tip">Tip</span> Adjust the width of the canvas with the{' '}
          <svg width="10" height="10" viewBox="0 0 12 12" xmlns="http://www.w3.org/2000/svg">
            <g fill="none" fillRule="evenodd">
              <path
                d="M1.5 5.2h4.8c.3 0 .5.2.5.4v5.1c-.1.2-.3.3-.4.3H1.4a.5.5 0 01-.5-.4V5.7c0-.3.2-.5.5-.5zm0-2.1h6.9c.3 0 .5.2.5.4v7a.5.5 0 01-1 0V4H1.5a.5.5 0 010-1zm0-2.1h9c.3 0 .5.2.5.4v9.1a.5.5 0 01-1 0V2H1.5a.5.5 0 010-1zm4.3 5.2H2V10h3.8V6.2z"
                id="a"
                fill="#999"
              />
            </g>
          </svg>
          Viewports addon in the toolbar
        </div>
      </section>
    </article>
  );
};
```

h2 タグの色を緑に変えたり、適当な段落を太字にしていたりする。

`pnpm storybook`および、`pnpm build-storybook`によって storybook でも linaria によって style が確かに変更されている様子を確認できる。

before:
![storybook-before](https://storage.googleapis.com/zenn-user-upload/02d98813ab23-20230606.png)

after:
![storybook-after](https://storage.googleapis.com/zenn-user-upload/0779fc9c0792-20230606.png)

特別な設定を行うことなく、storybook でも`linaria`を使えることが確認できた。

## 感想と補足など

storybook の設定の際、数週間前にやったときは色々設定を頑張った記憶があったのだが、
勘違いだったのかやり直すと追加設定ゼロで動いてびっくりした。ラッキーだが謎。
