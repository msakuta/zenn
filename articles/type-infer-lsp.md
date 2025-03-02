---
title: "[Rust] 型推論結果を LSP でエディターにライブ表示してみた"
emoji: "🖋️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "自作言語", "型推論", "Typeinference", "LSP"]
published: false
---

Rust で作るプログラミング言語シリーズです。

https://www.amazon.co.jp/dp/4297141922

## 概要

[前回](https://zenn.dev/msakuta/articles/10c571093538d7) は [Mascal](https://github.com/msakuta/mascal) 言語に型推論を実装してみましたが、これはコマンドラインで `-at` オプションをつけて出力を見るか、コンパイルエラーのメッセージでフィードバックされるのみで、あまり「生きている」感じがしませんでした。そこで、今回は VSCode の拡張機能として LSP (Language Server Protocol) を実装することによってエディタ上で型推論結果をライブプレビューしてみたいと思います。

この結果、次の GIF アニメのように編集するにつれてリアルタイムに型推論結果がフィードバックされ、また構文エラーや型推論エラーがある場合は赤文字で表示されます。

![lsp](/images/lsp.gif)

Mascal 言語は整数型に `i32` と `i64` の2種類を持ち、どちらかに確定しないと型チェックエラーとなる仕様です。整数リテラルに接尾辞をつけて `0i32` や `0i64` とするか、型が確定している他の変数と演算子で結びつけることによって確定することができます[^1]。

[^1]: Rust は型制約のない整数リテラルは `i32` とみなします。もしかしたらこの仕様に変えるかもしれません。

なお、エラー時にインラインで表示される赤文字は Error Lens という拡張機能によるものであり、当拡張機能の機能ではありません。エラーの内容は「PROBLEMS」ビューに表示されるものと同じです。

![lsp-error](/images/lsp-error.png)

複数の文に関わる型推論も、正しく表示されます。型アノテーションの順番が先でも後でも推論できていることが分かります。

![lsp2](/images/lsp2.gif)

さらに、配列の要素型についても型推論が行われていることが下のように見て取れます。要素へ代入される値の型によって、元の配列の型を推論しています。

![lsp3](/images/lsp3.gif)

さらに、タプルの要素型についても同様です。下のアニメでは 3 行目 -> 4 行目 -> 2 行目と型制約が伝搬していくことが見て取れます。

![lsp4](/images/lsp4.gif)


## LSP とは

LSP については Zenn にも良い記事が沢山紹介されているので、改めて説明するのも野暮という気もしますが、簡単に説明します。

LSP はエディタの拡張機能として、主にプログラミング言語の補助機能を提供するためのサーバと、クライアントとして動作するエディタとの間のプロトコルです。言語作者側は、プロトコルに対応するだけで複数のエディタに対応でき、エディタの作者はプロトコルに対応するだけで複数の言語をサポートできるというメリットがあります。また、サーバはプロトコルに従っていればどんな言語でも実装でき、 JavaScript 系である必要がありません(VSCode 自体は Electron ベースのため拡張機能は JS で実装するのが基本です)。 Microsoft が VSCode において提唱したのが最初ですが、今では vim や Emacs を始めとした様々なエディタがサポートしています。

詳しくは下記の公式紹介ページとプロトコル仕様書を参考にしてください。

* [公式紹介ページ](https://code.visualstudio.com/blogs/2016/06/27/common-language-protocol)
* [プロトコル仕様](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/)

VSCode の場合は、標準入出力を通して子プロセスをサーバとして立ち上げるという実行モデルが使われているようです。標準入出力を使って通信する内容は JSON-RPC で符号化されます。 Rust で言語を実装している場合、このようなサーバを構築するのは非常に簡単です。

実は、 LSP 対応は昔からやりたいとは思っていたのですが、型推論を実装するまで先延ばしにしていました。型推論がないと LSP によるライブフィードバックのメリットがあまりないのが理由です。


## 実装方針

まずは VSCode の拡張機能を作ります。これは以前から構文ハイライトのために作っていたものをそのまま使っていますが、 LSP を含む [公式のサンプル](https://github.com/microsoft/vscode-extension-samples/tree/main/lsp-sample) が参考になります。さらに、 Rust の場合は Tower を使って LSP サーバを実装する [tower-lsp](https://github.com/ebkalderon/tower-lsp/?tab=readme-ov-file) というクレートが使えます。これを使ったトイ言語のサンプル [tower-lsp-boilerplate](https://github.com/IWANABETHATGUY/tower-lsp-boilerplate) が参考になります。

次に、型推論の結果をどのように通信するかを考えてみます。 Mascal の場合は、すでにテキストから AST を構築するロジックは実装しているので、それを使いまわしたいところです。 LSP には `textDocument/didOpen` や `textDocument/didChange` といったイベントに応じたメッセージがあり、これに応じてサーバがバッファの内容を受け取ることができます。また、 Inlay hint の capability を有効にしていれば、表示するタイミングで `textDocument.inlayHint` リクエストが送られます。このタイミングで AST から型ヒントを表示したい部分を抽出してやればよさそうです。

## ライフタイムの問題

しかし、ちょっとした問題があります。 Mascal の AST はソース文字列をライフタイム束縛に持つ次のようなデータ構造ですので、ソーステキストより長生きすることはできません。

```rust
#[derive(Debug, PartialEq, Clone)]
pub enum Statement<'a> {
    Comment(&'a str),
    // ...
}
```

一方、 LSP サーバのデータは次のように定義されています (`tower-lsp-boilerplate` はソーステキストの格納に `Rope` というデータ構造を使っていますが、 Mascal では AST の構築に文字列スライスを必要とするので普通の `String` にしています)。

```rust
#[derive(Debug)]
struct Backend {
    client: Client,
    document_map: DashMap<String, String>,
}
```

`document_map` はエディタ開いているドキュメントの中でも Mascal 言語のソースとして解析する対象を集めたものです。これは `textDocument/didChange` などのメッセージによって更新され得ます。これが更新されてしまうと AST のライフタイムが無効になるので、次のように `Backend` の一部に `AST` を持つことはできません。

```rust
#[derive(Debug)]
struct Backend {
    client: Client,
    document_map: DashMap<String, String>,
    ast_map: DashMap<String, Vec<Statement<'_>>>,
}
```

これは広い意味での「自己参照データ構造(Self-referential data types)」の一例です。このようなデータ構造は Rust で表現することはできません。

このため、ソーステキストから AST の構築とヒントの生成までの全てを `inlayHint` のイベントの中で終わらせる必要があります。別の言い方をすれば、中間状態を取っておくことができません。

このような処理を書くことは、もちろん可能ですが、コード量が増えてくるとキーストロークのたびに文書全体をパースし直さなければならず、効率に難があります。 Mascal はまだ機能が貧弱なので大規模なプログラムは書けませんが、実用性がスケールした時には問題になる可能性が高いです。

これは AST データ構造にソース文字列への参照を持っている限り避けられず、 (Rust のライフタイムを考慮したうえでの) データ構造の設計の問題といえます。 `tower-lsp-boilerplate` では、 Rope という非同期更新が可能な特殊な文字列型と、ソーステキストへの参照を持たない AST を使って Backend の一部としています。奇しくも `tower-lsp-boilerplate` と Mascal のどちらも `Span` という名前のデータ型を定義していますが、その定義は次のように違います。

```rust
// tower-lsp-boilerplate
pub type Span = Range<usize>;

// mascal
pub type Span<'a> = LocatedSpan<&'a str>;
```

要するに、目的に応じて最適化な AST のデータ構造は異なるということです。コードの複製を避けるため同じ AST をコンパイラと LSP サーバで使い回そうとすれば、パフォーマンスが犠牲になるトレードオフが存在します。

エディタでのコーディング補佐を目的として性能に全振りしたデータ構造として Concrete Syntax Tree (具象構文木)というものも存在し、その有名な実装の一つに [tree-sitter](https://tree-sitter.github.io/tree-sitter/) があります。これはコメントや空白も含めたテキストの全てを表現する構文木で、更新の性能に優れていますが、コンパイラやインタプリタの実装には不要な情報が多いです。


## デバッグとパッケージ

VSCode で拡張機能を動かすには主に2つの方法があります。一つはデバッガからの起動、もう一つはパッケージ化してからインストールする方法です。

LSP を使う場合は拡張機能パッケージとは別にサーバを提供する方法を考える必要があります。 VSCode の拡張機能は JavaScript ベースのソースパッケージであり (`package.json` を使っていますが、 npm パッケージからの独自拡張のようなものです)、ネイティブ実行可能形式をプラットフォーム毎に配布することはできません。今回は開発段階なので配布のことは考えず、デバッガからの実行のみに注力することにします。

デバッガでのサーバの起動は[サーバの起動](#サーバの起動)セクションで解説します。

なお、パッケージ化は vsce というツールで可能です。詳しくは[こちらの公式ページ](https://code.visualstudio.com/api/working-with-extensions/publishing-extension)で解説されています。構文ハイライトなどだけなら LSP を使わなくても拡張機能としてパッケージ化できるのでお手軽です。

開発段階ではもちろんデバッガが便利ですが、一つだけはまった点がありました。


### 落とし穴

[tower-lsp-boilerplate](https://github.com/IWANABETHATGUY/tower-lsp-boilerplate) を真似しようとしたら、デバッグができなくて苦労しました。

tower-lsp-boilerplate では client というサブディレクトリに `client` という子パッケージを配置しており、 `./client/src/extension.ts` を `extension.js` にトランスパイルして拡張機能のエントリポイントとしているのですが、その出力先が `./dist` ディレクトリでした。 `package.json` には次のように書かれています。

```
"main": "./dist/extension.js",
```

これを[公式の lsp-sample](https://github.com/microsoft/vscode-extension-samples/blob/main/lsp-sample/package.json)と比較すると、次のようになっています。

```
"main": "./client/out/extension",
```

tower-lsp-boilerplate では `./dist` が出力先となっているため、 `tsc -b` でコンパイルすると `./client/node_packages` にインストールされているライブラリが参照できなくなり、起動できませんでした。これは Node.js の `require` の仕様で、親ディレクトリに存在する `node_packages` を検索するためです。これを動作させるには、ライブラリも含めてバンドルする `npm run compile` (それによって `node esbuild.js --production` を実行する)か、出力先を `./client` に変更する必要があります。これは恐らく tower-lsp-boilerplate の作者が変更したと思われますが、 README では `tsc -b` を勧めているので間違いだと思われます。この拡張はほとんどクライアントサイドのコードを変更しないのでバンドルしても良いのですが、私は`./client` に変更しました。


## サーバの起動

LSP サーバは子プロセスとして起動しますが、その実行ファイルは `extension.ts` で指定されています。 `tower-lsp-boilerplate` では次のように指定されています。

```js
const command = process.env.SERVER_PATH || "nrs-language-server";
```

これを自前の言語サーバで置き換えますが、実行ファイルへのパスを通すか相対パスで指定する必要があります。デバッグ時は `.vscode/launch.json` で `${workspaceRoot}` からの相対パスを設定するのが楽です。

`tower-lsp-boilerplate` に倣って次のように `extension.ts` でサーバプロセスそのものはデバッグセッションとともに自動的に起動しておけば、別途実行する必要はありません。

```js
  const run: Executable = {
    command,
    options: {
      env: {
        ...process.env,
        // eslint-disable-next-line @typescript-eslint/naming-convention
        RUST_LOG: "debug",
      },
    },
  };
  const serverOptions: ServerOptions = {
    run,
    debug: run,
  };
```


## デバッグ出力

[TUI デバッガ](https://zenn.dev/msakuta/articles/723ee6ae3b7eca)でも経験しましたが、 LSP も標準入出力を通信に使うので、サーバのログを標準出力に出すと通信が壊れてしまいします。その上、分かり易いエラーメッセージなども出ません。デフォルトでは標準エラー出力がデバッガのログに出力されるようです。このため、サーバを printf デバッグしようと思ったら `print!` や `println!` は使えず、 `eprint!` `eprintln!` を使う必要があります。

`tower-lsp-boilerplate` では log クレートを使って `debug!` マクロなどを使ってログを出力していました。この方法でも機能すると思いますが、私はコンパイラのクレートに依存先を増やしたくなかったので、コールバックを設定することでクレート使用者が出力ストリームを設定できるようにしておきました。



## 今後の展望

本稿で実装した LSP サーバは極めて基本的な機能しか持ちません。少なくとも変数や関数の定義位置の参照、自動補間、リネームぐらいの機能には対応したいところです。それでも基本的な機能だけでも Capability に追加して実装して味見できるのはプロトコルの優れた点だと思います。

また、今回は tower-lsp (およびその連鎖依存先の tokio )を使いましたが、ロジックは実質的にシングルスレッドで、リクエストは同期的に処理しているので、非同期ランタイムを使うメリットがあまりありません。データモデルを Mutex に包んで非同期ロジックにするか、元の AST から Rc を排除するか Arc で置き換えて Sync にする必要があります。あるいは、コンパイラ・インタプリタに使っているものに元の AST は影響を及ぼすのを避けるため、同期式サーバで置き換えることも考えられます。

最終的にはサーバの配布方法を考える必要があります。 Rust の場合を参考にすると、 Rust analyzer のサーバは拡張機能が自動的にプラットフォーム毎のバイナリをインストールするようになっているようです。この場合 VSCode の拡張機能マーケットプレイスの他にファイルサーバを立てる必要があります。

サーバに関しては一つ試したいことがあり、 WebAssembly にコンパイルして配布できないかと考えています。 WebAssembly はプラットフォーム非依存で、 Electron および V8 がインストールされている VSCode の環境であれば追加のネイティブバイナリのインストールなしに実行できるのではないかと思います。ただし、パフォーマンスではネイティブバイナリに劣る可能性があるので、ベンチマークが必要です。


## まとめ

* LSP を実装して型推論の結果をほぼリアルタイムにエディタに反映することができました。
* 小さなソーステキストには十分ですが、コードの規模が大きくなってきたら用途に応じて最適化したデータ構造にする必要があるかもしれません。
* エディタでの編集に伴ってエラーや型ヒントがリアルタイムでフィードバックされることの効果は絶大です。その証拠に、本機能の実装中に型推論のバグを発見しました。
