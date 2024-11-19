---
title: "[AWS CDK]L2コンストラクタを自作してuvでパッケージ管理をしているLambda/LambdaLayersをデプロイする"
emoji: "👷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["awscdk", "aws", "uv", "python"]
published: true
---

標準ライブラリや`boto3`など、Lambdaに標準で組み込まれているものではない3rdパーティーライブラリを含めたPython Lambdaをデプロイしたいことがある。
cdkだと

https://docs.aws.amazon.com/cdk/api/v2/docs/aws-lambda-python-alpha-readme.html

などが存在するが、2024年11月時点では[uv](https://docs.astral.sh/uv/)には対応していないので、
`uv`でLambdaのパッケージ管理をしたい場合は工夫が必要になる。

## 先に結論

Lambda FunctionとLayerを作るL2コンストラクタをそれぞれ自作した例↓

:::details uv-python-lambda.construct.ts

```python:uv-python-lambda.construct.ts
import { execSync } from 'node:child_process';
import { randomUUID } from 'node:crypto';
import fs from 'node:fs';
import process from 'node:process';

import { AssetHashType, DockerImage, aws_lambda as lambda } from 'aws-cdk-lib';

import type { Construct } from 'constructs';

type OmitKey = 'code';
export type PythonFunctionProps = Omit<lambda.FunctionProps, OmitKey> & {
  readonly entry: string;
  readonly build?: {
    readonly image?: DockerImage;
  };
};

export class PythonFunction extends lambda.Function {
  constructor(scope: Construct, id: string, props: PythonFunctionProps) {
    if (props.runtime && props.runtime.family !== lambda.RuntimeFamily.PYTHON) {
      throw new Error('Only `PYTHON` runtimes are supported.');
    }

    super(scope, id, {
      ...props,
      code: new lambda.AssetCode(props.entry, {
        assetHashType: AssetHashType.OUTPUT,
        bundling: {
          image: props?.build?.image ?? DockerImage.fromRegistry('dummy'),
          local: {
            tryBundle: (outputDir, _options): boolean => {
              const originalDir = process.cwd();
              const tmpRequirementsTxtPath = `/tmp/requirements-${randomUUID()}.txt`;

              process.chdir(props.entry); // uvプロジェクトのパスに移動

              fs.cpSync(props.entry, outputDir, {
                recursive: true,
                filter: (source, _destination): boolean => {
                  if (source.includes('.venv/')) return false;
                  if (source.includes('.gitignore')) return false;
                  if (source.includes('uv.lock')) return false;

                  return true;
                },
              });

              execSync(
                `uv export --no-dev --frozen --no-editable --output-file ${tmpRequirementsTxtPath}`,
              );
              execSync('uv venv');

              execSync(
                `uv pip install -r ${tmpRequirementsTxtPath} --target ${outputDir} --quiet`,
              );

              fs.rmSync(tmpRequirementsTxtPath);

              process.chdir(originalDir);
              return true;
            },
          },
        },
      }),
    });
  }
}

export type PythonLayerVersionProps = Omit<
  lambda.LayerVersionProps,
  OmitKey
> & {
  readonly entry: string;
  readonly build?: {
    readonly image?: DockerImage;
  };
};

export class PythonLayerVersion extends lambda.LayerVersion {
  constructor(scope: Construct, id: string, props: PythonLayerVersionProps) {
    super(scope, id, {
      ...props,
      code: new lambda.AssetCode(props.entry, {
        assetHashType: AssetHashType.OUTPUT,
        bundling: {
          image: props?.build?.image ?? DockerImage.fromRegistry('dummy'),
          local: {
            tryBundle: (outputDir, _options): boolean => {
              const originalDir = process.cwd();
              const tmpRequirementsTxtPath = `/tmp/requirements-${randomUUID()}.txt`;

              process.chdir(props.entry);

              execSync(
                `uv export --no-dev --frozen --no-editable --output-file ${tmpRequirementsTxtPath}`,
              );
              execSync('uv venv');
              execSync(
                `uv pip install -r ${tmpRequirementsTxtPath} --target ${outputDir}/python --quiet`,
              );

              fs.rmSync(tmpRequirementsTxtPath);

              process.chdir(originalDir);
              return true;
            },
          },
        },
      }),
    });
  }
}
```

:::

https://github.com/junkor-1011/cdk-uv-lambda-bundle-example/blob/zenn_20241119/lib/uv-python-lambda.construct.ts

- cdk標準で提供されている`lambda.Function`および、`lambda.LayerVersion`をそれぞれextendsし、ラップしたクラスを作ることで実装している。
- [LocalBundling](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.ILocalBundling.html)という機能を使うことでローカル環境にある`uv`コマンドを使い、`uv.lock`ファイルの内容からエクスポートした`requirements.txt`を使ってデプロイアセットを作成できるようにしている
- 必要であればDockerfileを書いてより柔軟にアセット内容を作れるようにもしている
- いわゆるdev depenciesはアセット内容に含めないようにしており、余計なパッケージが含まれないようにできる
- (※後述するように色々改善の余地があるので、参考程度でお願いします)

みたいな内容で作っている。

使用感としては、

Lambda Function:

```typescript:Lambda Function
// (...)
import { PythonFunction } from './uv-python-lambda.construct';

export class CdkAppStack extends cdk.Stack {
    constructor(scope: Construct, id: string, props?: cdk.StackProps) {
      super(scope, id, props);

      // (...)

      // uvでパッケージ管理しているLambdaを宣言↓
      new PythonFunction(this, 'python-lambda', {
        functionName: 'hello-world-function',
        runtime: lambda.Runtime.PYTHON_3_12, // ★uvで管理しているプロジェクトのpyproject.tomlおよび.python-versionの内容と整合させること
        handler: 'index.handler',
        entry: path.join(__dirname, '../python-lambda/hello-world'), // ★uvで管理しているプロジェクトのパス(※Lambdaのエントリポイントも同じ場所にあるとする)
      });

      // (...)
    }
}
```

Lambda Layer:

```typescript:Lambda Layer
// (...)
import { PythonLayerVersion } from './uv-python-lambda.construct';

export class CdkAppStack extends cdk.Stack {
    constructor(scope: Construct, id: string, props?: cdk.StackProps) {
      super(scope, id, props);

      // (...)

      // ↓uvで管理している3rdパーティーライブラリをPython Lambdaからインポートできる形式でLayerに配置
      const dependenciesLayer = new PythonLayerVersion(this, 'PythonLayer', {
        entry: path.join(__dirname, '../python-lambda/hello-world-with-layer'), // ★uvで管理しているプロジェクトのパス
        compatibleRuntimes: [lambda.Runtime.PYTHON_3_12], // ★uvで管理しているプロジェクトのpyproject.tomlおよび.python-versionの内容と整合させること
        compatibleArchitectures: [lambda.Architecture.X86_64],
      });

      new lambda.Function(this, 'python-function', { // 何らかのpython lambda
        runtime: lambda.Runtime.PYTHON_3_12, // ★uvで管理しているプロジェクトのpyproject.tomlおよび.python-versionの内容と整合させること
        code: lambda.Code.fromAsset(
          // (...)
        ),
        layers: [
          dependenciesLayer, // ★↑で作成したLayerを追加することで、Lambdaから追加のライブラリをインポート可能にしている
        ],
      });

      // (...)
    }
}
```

のような感じになり、
呼び出し元からはuv周りの複雑な処理を意識しないで済むようにできる。

## 解説

### Lambdaにおける3rdパーティーライブラリへのパスの通し方

まず前提として、Lambda(Python)において、3rdパーティーライブラリをどう配置すればよいか(=どうするとパスを通せるか)という話がある。

それについては、

https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/python-package.html#python-package-create-dependencies

https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/python-layers.html#python-layer-paths

を参照すると記載されている。(他にも有用な情報が多くあるので、参考になる)

作成すべきデプロイアセット(簡単のためzipファイルと考える)に関して、

- Lambda Function: zip内のトップ階層にフラットに各ライブラリおよびエントリポイント(handlerを含むファイル)が配置
- Lambda layers: zip内の`python/`ディレクトリ下に各ライブラリが配置

とすれば良い。

### uv周り

uvのCLIリファレンス:

https://docs.astral.sh/uv/reference/cli/

使うのは大きく2つで、

- `requirements.txt`をエクスポートする`uv export`
- `requirements.txt`からライブラリをダウンロードし、指定した場所に格納するための`uv pip install`

となる。それぞれ以下で補足する。

#### uv export

リファレンス:

https://docs.astral.sh/uv/reference/cli/#uv-export

色々オプションがあるが、ざっくり

```she
uv export \
    --no-dev \ # dev dependenciesを含まないようにする
    --frozen \ # uv.lockファイルを更新しないようにする(uv.lockファイルは必ず存在する必要がある)
    --output-file /tmp/requirements.txt # requirements.txtの出力先を指定(一時領域を指定しておき、後で消す)
```

みたいな感じにすれば、おそらく最低限動作する。

なお、`--frozen`は`--locked`の方が良いかもしれない。
(その他、細かいオプションは見直し、調整の余地がある。)

#### uv pip install

リファレンス:

https://docs.astral.sh/uv/reference/cli/#uv-pip-install

前節で作った`requirements.txt`を元に、所望のパッケージを所定の場所に配置する。
uvはpipの代替ツールとしても動作するため、素のpipではなくuvで`pip install`を行う。

内容としては大まかに

```she
uv pip install \
    -r /path/to/requirements.txt \ # requirements.txtのパスを参照
    --target /path/to/target_dir \ # ダウンロードしたライブラリを格納する場所を指定する
    --quiet # 余計なメッセージが表示されないようにする
```

といった感じ。

(先ほど同様、細かいオプション設定は調整の余地があると思われる)

### bundling

実際にアセット内容を作る部分のコードについて

#### LocalBundling

Lambda FunctionでもLayersでも、デプロイされるアセットの中身は`AssetCode`クラス:

https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda.AssetCode.html

によって記述することができる。
また、このコンストラクタのオプションにある`bundling`によってどうアセットを作るかを指定することができる:

https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.BundlingOptions.html

プロパティに関して、`image`が必須となっており、LocalBundlingに関わる`local`はオプションとなっているが、

```typescript
declare;
entry: string; // パスの指定
new lambda.AssetCode(entry, {
  bundling: {
    image: DockerImage.fromRegistry("dummy"), // ★本来はdocker.ioなどのレジストリにある各イメージを指定するが、dummyなどと書いておくとdockerによるbundlingが起こらない
    local: {
      tryBundle: (outputDir: string): boolean => {
        // ★outputDirにファイルやディレクトリを格納すると、それがLambda FunctionやLayersにデプロイされるアセット内容になる
        // ★先ほどのuvのコマンドを参考に、適宜処理を書く
      },
    },
  },
});
```

などのように書いておくと、dockerによるビルドではなく、ローカルのuvを使ってアセットを作るようにできる。

FunctionでもLayersでも内容はあまり変わらないので、Functionの場合のtryBundleの例を載せる：

:::details tryBundle例

```typescript:uv-python-lambda.construct.ts(抜粋)
declare entry: string; // uvで管理しているLambdaのコードを管理しているプロジェクトのパス
new lambda.AssetCode(entry, {
  assetHashType: AssetHashType.OUTPUT,
  bundling: {
    image: DockerImage.fromRegistry('dummy'),
    local: {
      tryBundle: (outputDir, _options): boolean => {
        const originalDir = process.cwd();
        const tmpRequirementsTxtPath = `/tmp/requirements-${randomUUID()}.txt`; // 念の為、衝突回避目的でuuidを名前に付加

        process.chdir(entry);

        fs.cpSync(entry, outputDir, { // ★Functionの場合、コード本体もoutputDirにいれる必要
          recursive: true,
          filter: (source, _destination): boolean => { // 不要なファイルは除外する
            if (source.includes('.venv/')) return false;
            if (source.includes('.gitignore')) return false;
            if (source.includes('uv.lock')) return false;
            // 他にも除外対象があれば適宜追加

            return true;
          },
        });

        execSync( // uv exportコマンドによるrequirements.txtの生成
          `uv export --no-dev --frozen --no-editable --output-file ${tmpRequirementsTxtPath}`,
        );
        execSync('uv venv'); // git clone直後でvenv仮想環境が無い場合だとエラーになるので入れている

        execSync( // uv pip installコマンドにより、outputDitに3rdパーティーライブラリを格納
          `uv pip install -r ${tmpRequirementsTxtPath} --target ${outputDir} --quiet`,
        );

        fs.rmSync(tmpRequirementsTxtPath); // 不要になったrequirements.txtを削除

        process.chdir(originalDir); // はじめの場所に戻らないとエラーになる
        return true;
      },
    },
  },
})
```

なお、Layersの場合、Lambda本体のソースコードは不要になるのでその部分が無くなるのと、
格納先が`${outputDir}`から`${outputDir}/python`に変わる、といった違いがでてくる。
(が、基本は全く同じ)

:::

#### Dockerfileを使ったBundle

前節の`lambda.AssetCode`の作成部分で、bundlingオプションのimage部分に`DockerImage`クラス:

https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.DockerImage.html

のインスタンスを与えることで、今度はDockerを使ってアセットを作成するようにできる。
特に、`DockerImage.fromBuild`:

https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.DockerImage.html#methods

使うことで、指定したDockerfileの内容を元にアセットを作れる。

基本的な仕組みは

https://zenn.dev/junkor/articles/982aefa4f8e300

で書いているのでそちらも参照。

要点としては

https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda.Code.html#static-fromwbrdockerwbrbuildpath-options

を使うことで、Dockerfileによりアセット内容をほぼいくらでも自由に作ることができるので、
uvプロジェクト内の適当なファイルをDockerfileに渡して処理することでFunctionやLayerの中身を作っている、とう感じ。

Dockerfileでアセットを作る場合、

- Function: `/asset/`直下に依存ライブラリを配置
- Layer: `/asset/python/`直下に依存ライブラリを配置

とするとパスを通すことが出来る。

あまり改めて解説する要素は無いが一応実装例は

https://github.com/junkor-1011/cdk-uv-lambda-bundle-example/tree/zenn_docker_20241119

に置いてある。

:::details (各ファイル抜粋)

Functionのアセットを作るDockerfile

https://github.com/junkor-1011/cdk-uv-lambda-bundle-example/blob/zenn_docker_20241119/python-lambda/hello-world/Dockerfile

```dockerfile:Dockerfile(Lambda Function)
FROM ghcr.io/astral-sh/uv:bookworm-slim

ENV PYTHON_VERSION=3.12

COPY ./ /asset/

WORKDIR /work

RUN --mount=type=bind,target=. uv export --no-dev --frozen --output-file /tmp/requirements.txt

RUN uv python install $PYTHON_VERSION && \
    uv venv && \
    uv pip install -r /tmp/requirements.txt --no-cache-dir --target /asset
```

Layerのアセットを作るDockerfile

https://github.com/junkor-1011/cdk-uv-lambda-bundle-example/blob/zenn_docker_20241119/python-lambda/hello-world-with-layer/dependencies-layer.Dockerfile

```dockerfile:Dockerfile(Lambda Layer)
FROM ghcr.io/astral-sh/uv:bookworm-slim

ENV PYTHON_VERSION=3.12

WORKDIR /work

RUN --mount=type=bind,target=. uv export --no-dev --frozen --output-file /tmp/requirements.txt

RUN uv python install $PYTHON_VERSION && \
    uv venv && \
    uv pip install -r /tmp/requirements.txt --no-cache-dir --target /asset/python
```

:::

やっていることは相変わらず、

- `uv export`コマンドを使うことで、lockファイルの内容を反映させつつ不要なパッケージを除去した`requirements.txt`を生成
- ↑で作った`requirements.txt`を使い、`uv pip install`コマンドを使い、場所を指定してライブラリをインストール・配置

という感じ。

なお、ベースイメージには`astral`が公開している`ghcr.io/astral-sh/uv`:

https://docs.astral.sh/uv/guides/integration/docker/#available-images

を使っている。

## 改善点

いくらでもあると思うが、とりあえず2点。

### uvのgroup機能を使った柔軟なパッケージ管理

`uv add`(ライブラリのインストール)に関して、

https://docs.astral.sh/uv/reference/cli/#uv-add

のオプションを見ると、`--group`なるものが存在する。

exportの方にもgroupに関するオプションが存在する(`--group`, `--only-group`など)

https://docs.astral.sh/uv/reference/cli/#uv-export

すなわち、インストールするライブラリに関して、必須ライブラリとそれ以外(dev dependencies)の2択ではなく、
より細かい単位でグループ分けすることができる。

例えば、Lambdaの開発で使うライブラリに関して、

- 必須: デプロイパッケージに含める必要あり
- AWS管理: 実行時に必要なライブラリだが、boto3などLambdaに初めから入っているためデプロイ内容に含める必要性無し
- Layer1: Lambda Layerその1にライブラリを分離してデプロイする
- Layer2: Lambda Layerその2にライブラリを分離してデプロイする
- 開発時のみ必要(dev dependencies)
  - uv的には`dev`グループの別名扱いになる

みたいに分類して、関数本体と複数のLayerにパッケージをそれぞれ分けてデプロイする、といったことも可能性としてはできることになる。

が、きれいに対応して実装に落とし込むのは大変そうだったのと、個人的にそこまでは今のところ不要そうだったので、
先ほど書いた例にはこうした対応は含まれていない。

他にも、uvはworkspace機能:

https://docs.astral.sh/uv/concepts/workspaces/

などもあるので、そうした部分にも対応できるようになるとより良い感じになりそう？

### 複数言語(TypeScript以外)のサポート

cdkで作ったコンストラクタは`jsii`:

https://github.com/aws/jsii

を使うことで、TypeScriptやJavaScript以外の言語からも使えるようになるとのこと。

個人的にはcdk自体のコードはTypeScriptで書ければ良いのでそこまで頑張っていないが、
PythonでLambdaを書くケースだとcdkの実装もpythonに合わせたいケースなどもあるのかもしれない。

Construct Hubなどに公開できるようなレベルの再利用性の高いコンストラクタを作るのであれば、
こちらも使ったことはないが`projen`:

https://aws.amazon.com/jp/blogs/news/getting-started-with-projen-and-aws-cdk/

などを使うと良いのかもしれない。

## 参考

先行例:

https://constructs.dev/packages/uv-python-lambda/v/0.0.2?lang=typescript

https://github.com/fourTheorem/uv-python-lambda/tree/v0.0.2

[Construct Hub](https://constructs.dev/)に上がっていたもの。

まだ安定しているとは言えなかったり(`uv`自体も)、イメージ通りの挙動でない部分もあるので自分で作ることにしたが、
参考にはなりそう。

あと、`@aws-cdk/aws-lambda-python-alpha`

https://github.com/aws/aws-cdk/tree/main/packages/%40aws-cdk/aws-lambda-python-alpha/lib

や、`aws-cdk-lib/aws-lambda-nodejs`

https://github.com/aws/aws-cdk/tree/main/packages/aws-cdk-lib/aws-lambda-nodejs/lib

など、AWSの公式リポジトリで公開されている、lambda.FunctionをラップしているL2コンストラクタなどは実装の参考になる。
(比較的読みやすい部類に入るとは思う)
