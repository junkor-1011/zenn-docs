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

## 0. 試行時の環境および各主要ライブラリのバージョン

特に主要な使用ライブラリのバージョンが上がるとセットアップ方法が変わってくることが多いので

- Linux(Fedora38)
- Node.js v18.16.0
  - パッケージマネージャー: pnpm@8.6.1
- vite@4.3.9
- React@18.2.0
- storybook@7.0.x
- linaria@x

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
+     <h1 className={css`font-weight: bold; color: blue;`}>
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

## 2. storybook における linaria の有効化

storybook@7 で有効化を行う
