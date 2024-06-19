---
title: "プロキシ認証を突破してApache Spark(pyspark)のconfigでjarを指定して追加する"
emoji: "🫠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["spark", "pyspark", "java"]
published: true
---

昔Qiitaで

https://qiita.com/junkor-1011/items/b12ec62f2615d068c1a5

みたいな記事を書いたことがあるが、
プロキシ認証が必要な環境だと追加設定をしないと同じことが出来ないので、それについて調べてメモをしたもの。

すなわち、

```python
from pyspark.sql import SparkSession

# 例としてApache Iceberg:
# https://mvnrepository.com/artifact/org.apache.iceberg/iceberg-spark-runtime-3.4
# のjarファイルを取得する場合↓
spark = (
    SparkSession
        .builder
        .config(
            "spark.jars.packages",
            "org.apache.iceberg:iceberg-spark-runtime-3.4_2.12:1.5.2"
        )
        .getOrCreate()
)
```

みたいなことをしたいときに、プロキシ環境で何とか動かすためのメモという感じ。

(あまり詳しくないが)Javaはプロキシ認証設定が特殊?なようで、
環境変数`https_proxy`や`http_proxy`を設定するだけではプロキシを通ってくれないことがあり、
そういう場合はJava独自のやり方でプロキシの認証に必要な情報を渡す必要があるらしい。

## (検証した環境のメモ)

- Linux(WSL2)
  - Ubuntu24.04LTS
- Python3.12.3([rye](https://rye.astral.sh/))
- pyspark3.4.3
  - Scala version 2.12.17
- openjdk 17.0.11(Amazon Corretto)
  - [sdkman](https://sdkman.io/install)でインストール

## 設定方法の例

### 1. `JAVA_OPTS`相当の設定実施

まずは`JAVA_OPTS`にセットするようなものに相当する内容(文字列)を準備する。

実際にセットする内容は環境(プロキシサーバーの設定)によってある程度変化するので注意。

まず、

https://docs.oracle.com/javase/jp/6/technotes/guides/net/proxies.html

などにも記載されているように、

- `http.proxyHost`
- `http.proxyPort`
- `https.proxyHost`
- `https.proxyPort`

の4つは基本的に必要になる。

例として、IPアドレスが`x.x.x.x`で80番ポートでリクエストを受けるプロキシサーバーを経由する設定を行いたい場合、

```sh
# ↓適宜編集する
PROXY_HOST="x.x.x.x"

# ↓適宜編集する
PROXY_PORT="80"

export JAVA_OPTS="-Dhttp.proxyHost=${PROXY_HOST} -Dhttp.proxyPort=${PROXY_PORT} -Dhttps.proxyHost=${PROXY_HOST} -Dhttps.proxyPort=${PROXY_PORT}"
```

のような感じで`JAVA_OPTS`を設定する感じになる。

※検証した環境では、環境変数`JAVA_OPTS`を設定してもpysparkで自動で読み取ってくれることはなかったので、sparkに上記の値を渡す部分は別途必要

また、プロキシに認証がかけられている場合は追加で

- `http.proxyUser`
- `http.proxyPassword`
- `https.proxyUser`
- `https.proxyPassword`

も必要になり、`JAVA_OPTS`の設定内容は

```sh
# ↓適宜編集する
PROXY_HOST="x.x.x.x"

# ↓適宜編集する
PROXY_PORT="80"

# ↓適宜編集する
PROXY_USER="proxy_user"

# ↓適宜編集する
PROXY_PASSWORD="proxy_password"

export JAVA_OPTS="-Dhttp.proxyHost=${PROXY_HOST} -Dhttp.proxyPort=${PROXY_PORT} -Dhttp.proxyUser=${PROXY_USER} -Dhttp.proxyPassword=${PROXY_PASSWORD} -Dhttps.proxyHost=${PROXY_HOST} -Dhttps.proxyPort=${PROXY_PORT} -Dhttps.proxyUser=${PROXY_USER} -Dhttps.proxyPassword=${PROXY_PASSWORD}"
```

のような感じになる。

ただし、上記だけでは不十分な場合があり、

https://www.oracle.com/java/technologies/javase/8u111-relnotes.html

に記載されているように、
プロキシ認証にBASIC認証が使われている場合はデフォルトでは無効化されており、
追加設定をしないとプロキシの認証が通らない。

そのような場合、

https://stackoverflow.com/questions/41505219/unable-to-tunnel-through-proxy-proxy-returns-http-1-1-407-via-https

などを参考にすると

- `jdk.http.auth.tunneling.disabledSchemes=`
- `jdk.http.auth.proxying.disabledSchemes=`

を追加で設定する必要がある。

以上を踏まえると、(人にとっては)

```sh
# ↓適宜編集する
PROXY_HOST="x.x.x.x"

# ↓適宜編集する
PROXY_PORT="80"

# ↓適宜編集する
PROXY_USER="proxy_user"

# ↓適宜編集する
PROXY_PASSWORD="proxy_password"

export JAVA_OPTS="-Dhttp.proxyHost=${PROXY_HOST} -Dhttp.proxyPort=${PROXY_PORT} -Dhttp.proxyUser=${PROXY_USER} -Dhttp.proxyPassword=${PROXY_PASSWORD} -Dhttps.proxyHost=${PROXY_HOST} -Dhttps.proxyPort=${PROXY_PORT} -Dhttps.proxyUser=${PROXY_USER} -Dhttps.proxyPassword=${PROXY_PASSWORD} -Djdk.http.auth.tunneling.disabledSchemes= -Djdk.http.auth.proxying.disabledSchemes="
```

のような感じで`JAVA_OPTS`を設定する必要がある。

### 2. Apache Sparkにおける設定

筆者が手元で試した範囲では、残念ながら先程の手順で設定した環境変数`JAVA_OPTS`をよしなに読み込んでくれることは無かったので、先程の`JAVA_OPTS`相当の内容をApache Sparkのコンフィグで設定する必要がある。

Apache Sparkの設定方法は複数通りあり:

https://spark.apache.org/docs/latest/configuration.html

コンフィグファイルを記入しておくやり方や、
シェルでオプションを渡す方法もある。
(その時のケースで適切なものを選択する)

結論としては、いずれかの方法で

- `spark.driver.defaultJavaOptions`

に先ほどの`JAVA_OPTS`の内容の文字列をセットすればOK
(※未検証だが、`spark.driver.extraJavaOptions`などでももしかすると動くかも?)

今回はある程度動的に設定できるようにしたかったので、冒頭に書いたコードを↓のような感じで編集している:

```diff python
+ import os # 環境変数を読み込むために追加でimport
from pyspark.sql import SparkSession

spark = (
    SparkSession
        .builder
+       .config("spark.driver.defaultJavaOptions", os.getenv("JAVA_OPTS") or "")
        .config(
            "spark.jars.packages",
            "org.apache.iceberg:iceberg-spark-runtime-3.4_2.12:1.5.2"
        )
        .getOrCreate()
)
```

上記はあくまで一例だが、上記のようにして`JAVA_OPTS`が空であれば何も起きず、
事前に`JAVA_OPTS`をセットしたときだけプロキシサーバーを経由するようになるので、
プロキシあり/なしで同じコードを使えるようにしている。

(なお、他にも[`spark-env.sh`](https://www.ibm.com/docs/en/pasc/1.1.1?topic=files-spark-envsh)などを使う方もあると思うし、そっちの方がきれいかもしれない)

方法にバリエーションはあるが、とりあえず上記のような設定を組み込むことでプロキシ環境でも冒頭のようなことが可能になる。

## おまけ

プロキシ環境下であれば、大体の人は環境変数`https_proxy`をセットしていることが考えられ、
形式が一部違うとは言え、この値をJavaの`JAVA_OPTS`向けに入れ直すのはDRYでなくて面倒に感じられる。

プロキシ認証におけるユーザー名に`@`や`:`が使われていない場合に限られるなど、完全なソリューションでは無いが、
個人的なワークアラウンドとしては

https://zenn.dev/junkor/articles/82aba65d2f0880

に書いたような感じで`awk`コマンドなどを使って`https_proxy`の値を分解して`JAVA_OPTS`にセットしている。

すなわち、

```sh:.env
PROXY_HOST=$(echo $https_proxy | awk -F":" '{print $3}' | awk -F"@" '{print $2}')
PROXY_PORT=$(echo $https_proxy | awk -F":" '{print $4}')
PROXY_USER=$(echo $https_proxy | awk -F":" '{print $2}' | awk -F"//" '{print $2}')
PROXY_PASSWORD=$(echo $https_proxy | awk -F":" '{print $3}' | awk -F"@" '{print $1}')

export JAVA_OPTS="-Dhttp.proxyHost=${PROXY_HOST} -Dhttp.proxyPort=${PROXY_PORT} -Dhttp.proxyUser=${PROXY_USER} -Dhttp.proxyPassword=${PROXY_PASSWORD} -Dhttps.proxyHost=${PROXY_HOST} -Dhttps.proxyPort=${PROXY_PORT} -Dhttps.proxyUser=${PROXY_USER} -Dhttps.proxyPassword=${PROXY_PASSWORD} -Djdk.http.auth.tunneling.disabledSchemes= -Djdk.http.auth.proxying.disabledSchemes="
```

といった内容の`.env`ファイルを作成し、

```sh
source .env
```

などによって読み込むことでユーザーによらず同一のコードで`JAVA_OPTS`を設定できるようにしている。

(先述の通り、ユーザー名およびパスワードの内容によっては動作しない可能性があり、不完全なので注意して使う感じにはなる)
