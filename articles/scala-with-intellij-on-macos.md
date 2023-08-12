---
title: "Scala の IntelliJ 開発環境を macOS 上に構築する"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [scala]
published: true
---

「なっとく！関数型プログラミング」を写経したくて、10 年ぶりぐらいに Java の環境を作る必要がありました。

## 既存の Java のアンインストール

https://docs.oracle.com/javase/8/docs/technotes/guides/install/mac_jdk.html#uninstall

```bash
$ /usr/libexec/java_home -V
Matching Java Virtual Machines (1):
    17.0.6 (arm64) "Oracle Corporation" - "Java SE 17.0.6" /Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home
/Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home
```

```bash
$ sudo rm -rf /Library/Java/JavaVirtualMachines/jdk-17.jdk
```

## asdf を使って Java をインストール

https://github.com/halcyon/asdf-java

```bash
$ asdf plugin-add java https://github.com/halcyon/asdf-java.git
```

「Apple Silicon integration」という項目が README にありますね。  
native でも Rosetta でも切り替えられるようです。

### どのバージョンが必要？

https://docs.scala-lang.org/overviews/jdk-compatibility/overview.html

なるほど、JDK と Minimum Scala で対応があるのですね。
書籍の方が Scala 3.1.3 のようですが、JDK は 17 を入れるような指示がありますね。
うーん、とりあえず対応表通りに 18 にしておきますかね。

```bash
$ asdf install java adoptopenjdk-18.0.2+101
$ asdf list java
 *adoptopenjdk-18.0.2+101
$ asdf global java adoptopenjdk-18.0.2+101
```

続いて、公式の README によると `JAVA_HOME` と macOS 用の設定があるようです。`JAVA_HOME` ってなんか懐かしい。


私は zsh なので、`~/.zshrc` に以下を追記。

```bash
. ~/.asdf/plugins/java/set-java-home.zsh
```

続いて `~/.asdfrc` に以下を追記。こんな設定ファイルがあるとは知りませんでした。

```bash
java_macos_integration_enable = yes
```

シェルを再起動すれば無事 java -version が動きました。

が、あれ `/usr/libexec/java_home -V` は認識されてませんね。
これ、フックということは、インストール時に動くのでは？

```bash
$ asdf uninstall java adoptopenjdk-18.0.2+101
$ asdf install java adoptopenjdk-18.0.2+101
################################################################################## 100.0%
OpenJDK18U-jdk_aarch64_mac_hotspot_18.0.2.1_1.tar.gz
OpenJDK18U-jdk_aarch64_mac_hotspot_18.0.2.1_1.tar.gz: OK
Integrating with /usr/libexec/java_home needs root permission for it to create folders under /Library/Java/JavaVirtualMachines
Password:
```

お、それっぽいログが出てますね。

```bash
$ /usr/libexec/java_home -V
Matching Java Virtual Machines (1):
    18.0.2.1 (arm64) "Eclipse Adoptium" - "OpenJDK 18.0.2.1" /Library/Java/JavaVirtualMachines/adoptopenjdk-18.0.2+101/Contents/Home
/Library/Java/JavaVirtualMachines/adoptopenjdk-18.0.2+101/Contents/Home
```

バッチリです！

## Scala のインストール

書籍の方には Scala のインストールという項目はなく、sbt をインストールするとあります。

https://www.scala-lang.org/download/
https://www.scala-sbt.org/download.html

今回の目的は写経なので、sbt のみをインストールすることにします。

brew でインストールするとなっていますが asdf でもできるよう。が、、、なんかちゃんとメンテナンスされてるリポジトリが無いのかな。少し不安ですが、自己責任で以下のリポジトリを追加します。

https://github.com/nanne007/asdf-sbt

```bash
$ asdf plugin-add sbt https://github.com/lerencao/asdf-sbt
$ asdf list-all sbt
$ asdf install sbt 1.9.2
$ asdf list sbt
  1.9.2
$ asdf global sbt 1.9.2
$ sbt --version
copying runtime jar...
[info] [launcher] getting org.scala-sbt sbt 1.9.2  (this may take some time)...
sbt version in this project: 1.9.2
sbt script version: 1.9.2
```

起動してみます。

```bash
$ sbt console
[info] welcome to sbt 1.9.2 (Eclipse Adoptium Java 18.0.2.1)
[info] loading global plugins from /Users/ydan/.sbt/1.0/plugins
[info] loading project definition from /Users/ydan/project
[info] set current project to ydan (in build file:/Users/ydan/)
https://repo1.maven.org/maven2/org/scala-sbt/util-interface/1.9.1/util-interface-1.9.1.j…
  100.0% [##########] 4.3 KiB (29.8 KiB / s)
[info] Non-compiled module 'compiler-bridge_2.12' for Scala 2.12.18. Compiling...
[info]   Compilation completed in 4.082s.
[info] Starting scala interpreter...
Welcome to Scala 2.12.18 (OpenJDK 64-Bit Server VM, Java 18.0.2.1).
Type in expressions for evaluation. Or try :help.
```

どうやら、Scala 2.12.18 で起動するらしい。これは何かデフォルトで指定されているのかな？ `~/.sbt` があって、確かにここで 2.12.18 の jar があるけれど、よくわからない。

## IntelliJ の設定

![](https://storage.googleapis.com/zenn-user-upload/2f9ced040240-20230812.png)

sbt や Scala のバージョンを指定できます。IntelliJ の Scala Plugin の方で必要なバージョンを勝手に入れてくれるよう。

Java のバージョンだけ、先ほどインストールした 18 系を指定。

## 軽く Hello World

https://docs.scala-lang.org/ja/getting-started/sbt-track/getting-started-with-scala-and-sbt-on-the-command-line.html#

ここによると、sbt で作成されたプロジェクトの場合は `src/main/scala` にコードを配置するらしい。`Main.scala` でいいのかな。
なんで IntelliJ の作成直後だとファイルが無いのだろうか。

https://docs.scala-lang.org/scala3/book/taste-hello-world.html

上記によると、以下の 1 行でいいらしい。

```scala
@main def hello() = println("Hello, World!")
```

ルートで `sbt` を起動して `~run` しろと書いてあるのでやってみる。

```bash
sbt
[info] welcome to sbt 1.9.2 (Eclipse Adoptium Java 18.0.2.1)
[info] loading global plugins from /Users/ydan/.sbt/1.0/plugins
[info] loading project definition from /Users/ydan/Documents/IdeaProjects/NattokuFunctional/project
[info] loading settings for project root from build.sbt ...
[info] set current project to NattokuFunctional (in build file:/Users/ydan/Documents/IdeaProjects/NattokuFunctional/)
[info] sbt server started at local:///Users/ydan/.sbt/1.0/server/85ea6ee4b7ec605a936d/sock
[info] started sbt server

sbt:NattokuFunctional> ~run
[info] running hello
Hello, World!
[success] Total time: 0 s, completed Aug 12, 2023, 12:17:39 PM
[info] 1. Monitoring source files for root/run...
[info]    Press <enter> to interrupt or '?' for more options.
```

おお、実行された！

とりあえず、実行までできたので写経する環境はできたようです！


