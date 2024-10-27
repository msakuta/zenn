---
title: "[Rust] 自作言語での TUI デバッガのススメ"
emoji: "🐀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "自作言語", "デバッガ", "TUI", "Ratatui"]
published: true
---

Rust で作るプログラミング言語シリーズです。

https://www.amazon.co.jp/dp/4297141922

## バイトコードコンパイラの難しさ

書籍での心残りの一つが、第６章のバイトコードの難易度です。本書ではプログラミング言語を作るにあたってステップ・バイ・ステップで難易度を上げていくように構成を考えていますが、第６章のバイトコードの実装の時点で難易度が急激に上昇します。例えば、次のようなジャンプアドレスの計算を行う必要があるのですが、これはかなり込み入ったロジック計算が必要になります。

条件分岐では、下図のようにジャンプしますが、このアドレス計算が厄介です。

![cond](/images/cond.png)

ループについても同じくジャンプ命令が必要ですが、こちらはループの先頭にアドレスを戻します。

![for](/images/for.png)

この実装の難しさは、プログラムの意味を理解する(意味論)ロジックと、それをバイトコードへ変換するロジックを一つとしてデバッグしなくてはならないところにあります。どちらに問題があるかを切り分けるのは簡単ではありません。書籍では第4章で AST インタプリタを実装するので、意味論については少し準備運動ができますが、それでも難しいと言わざるを得ません。書籍ではページ数の都合もあり、あまり図をふんだんに使って詳しく解説できなかったという事情もあります。

そこで、本稿ではデバッグを容易にするためのインタラクティブな TUI デバッガを実装してみます。対象となる言語は、書籍で作った Ruscal ではなく、もう少し実用的な言語を目指している [Mascal](https://github.com/msakuta/mascal) です。

## TUI

TUI とは、 Terminal User Interface のことで、端末で文字を使って描画するユーザーインターフェースです。見た目は下図のような感じです。

![mascal-tui](/images/mascal-tui.png)

今回は Rust では有名な Ratatui というライブラリを使います。

https://ratatui.rs/

端末なので、基本的にキーボードを使って操作します。マウスも使う方法はあるようですが、デバッガの主な役割はステップ実行やブレークポイントの設定なので、あまりマウスでの入力にメリットがありません。そこで今回はキーボードのみで操作する UI にします。

とはいえ、ウィンドウ (Ratatui では Widget と呼びます) ごとのフォーカスやスクロールもサポートします。

この操作方法では、キーバインディングが重要になるので、それをすぐに確認できるようにヘルプメニューも実装します。これは H キーを押すことでいつでも表示できます。

![mascal-tui-help](/images/mascal-tui-help.png)


## 技術選定の理由

デバッガ UI を作るには他にもいろいろな選択肢がありますが、 TUI を選んだ理由の前に他の選択肢を選ばなかった理由を述べます。

### ネイティブ GUI ツールキット

ここで言っているのは Qt や Gtk のようなデスクトップ向け GUI ツールキットです[^2]。最も高機能ではありますが、プラットフォームへの依存度が高くなり、開発の手間もかかります。 Rust ではバインディングを使わなくてはなりませんが、正直にいってあまり良いバインディングライブラリがありません。

[^2]: その昔であれば MFC や WTL や wxWidgets や FLTK や Tk などもありました。

デバッガに必要な機能がそれほど高度ではないことを考えるとこれはオーバーキルです。

### Web UI

最近よく見られるようになった、レンダリングにブラウザを使うタイプの GUI アプリケーションです。 Electron が最も有名でしょうが、 Rust でも Tauri の人気が急上昇しています。

クロスプラットフォーム化が簡単で、 Web ページと互換のアプリケーションも簡単に作れますが、レンダリングのためには Web サーバを実装する必要があります。 Rust では web サーバを実装するのは難しくありませんが、 async ランタイムを使うことになるでしょう。コマンドラインツールのデバッガに async ランタイムを使うのはやはりオーバーキルです。

また、フロントエンドは Web ページとして実装するので技術スタックが2重になってしまうのも、デバッガには必要以上の複雑さでしょう。

### LSP

Language Server Protocol は、既存のエディタと各種言語を結びつけるための言語プロトコルです。これを自作言語に実装すれば、既存のエディタを使って IDE 並みの開発者経験が得られます。主にシンタックスハイライティングやコードヒントやリファクタリングに向いていますが、デバッガには使えません。

デバッガに使うには [Debug Adapter Protocol](https://code.visualstudio.com/blogs/2018/08/07/debug-adapter-protocol-website) という別のプロトコルを実装する必要があります。しかしこのプロトコルはステートフルである事の他にも実行ランタイムへの依存性が強い印象です。例えば Rust のデバッガには LLDB が使えますが、 VSCode で使うと編集のシャドーイングを未だにサポートしていなかったりと、完成度がいまいちです。


### TUI

TUI は機能性こそシンプルですが、本稿でのテーマである実行ランタイムのデバッグ(その上で書かれたプログラムではなく)という目的から考えると最適と言えます。ネイティブ GUI のような複雑度もなく、 Web UI のような技術スタックの複雑さもなく、 Debug Adapter Protocol の想定している実行モデルに合わせる必要がないので、内部状態を完全にコントロールできます。

いずれは Web UI か Debug Adapter Protocol をサポートするにしても、デバッガに必要な機能を経験するには、最初から巨大なフレームワークに合わせるよりは、シンプルさは役に立つでしょう。

さらにおまけのメリットとして、後述するようにタイムトラベルデバッガを実装することができます。

デメリットとしては、機能性があまり高くなく、複雑なグラフィックスは表示できないということがあります。しかし、デバッガが表示するのは基本的にテキストだけなので、それほど問題にはなりません。


## 基本機能

### ステータス表示

左上隅には、現在の実行ステータスを表示します。このデバッガはプログラム実行中か否かを区別しているので、その情報と、タイムトラベルデバッガのバッファのサイズと現在位置も表示しています。 Continue したときは最後に実行した時点からの実行したインストラクションの数も表示します。

エラーがあれば2行目に表示されます。これは結構重要で、 TUI は標準出力を占有するので、 TUI 自体のエラーをユーザにフィードバックする手段がありません。 TUI に何らかのバグがあった場合にその手掛かりとなるエラー情報はどこか常に見えるところに表示しておく必要があります。

![](/images/mascal-tui-status.png)

### スクロール

それぞれのウィジェットは個別にスクロールすることができます。スクロール可能なウィジェットには右端にスクロールバーが表示されます。

インストラクションが移動するステップ実行などの操作を行ったときには自動で現在地を表示するようにスクロールしますが、手動で上下キーを使ってスクロールすることもできます。

スクロール対象のウィジェットは枠線を太く表示してフォーカスがあることを示します。フォーカスの移動には Tab キーを使います[^1]。

[^1]: Shift+Tab で前のウィジェットに戻るようにしたかったのですが、なぜかそのイベントが取れませんでした。


### 出力ウィンドウ

Output は、 `print` 関数などを通して標準出力に出力された内容を表示するウィジェットです。

![](/images/mascal-tui-output.png)

あまり特徴的な機能はありませんが、 TUI を使う際には注意点があります。 TUI はそれ自体標準出力を使っているので、スクリプトからの出力をそのまま標準出力に出すと、2つの出力が競合を起こして表示が狂ってしまいます。このため、スクリプトからの出力をリダイレクトして内部バッファに書き込み、ウィジェットに表示しています。詳しくは[出力ストリームの処理](#出力ストリームの処理)に後述します。


### ソースコードのリスト表示

Source listing というウィジェットでソースコードを表示します。ソースはバイトコードと関連付けられていないと、どこを実行しているかわからないので、デバッグ情報をコンパイル時に有効にする (Mascal CLI では `-g` オプション)必要があります。

![](/images/mascal-source-listing.png)

かなりベーシックですがシンタックスハイライティングもしています。

実行中であればその行の先頭に `*` がつきます。

ブレークポイントが設定されている行には赤文字の `o` がつきます。

実行中の行の中でも位置が特定できるときは、文字の背景を黄色でハイライトしています。

### ディスアセンブル

Disassembly ウィジェットでディスアセンブルしたテキスト表現を表示します。

![](/images/mascal-tui-disassembly.png)

実行中の行があれば先頭に `*` がつきます。

それぞれの行の最初の4桁の数字は、デバッグ情報があれば対応するソースの行番号です。

`[34]` のような角括弧内の数字はインストラクションのインデックスで、インストラクションポインタが指す先の数字です。

その後に現れる `Mul` や `Add` や `LoadLiteral` といった単語はOpコードのニーモニックです。

そのさらに後に現れる2つの数字は、インストラクションの引数です(Mascal では固定長インストラクションで全ての命令が2つの引数を取ります)。引数の意味はOpコードに依存します。

ちなみに、ディスアセンブルの雰囲気はどんなバイトコードでも似たような感じです。以下に Python のバイトコードのディスアセンブルを示しますが、元のソースコードとは似ても似つかず、知らない人にはただの謎の英字と数字の羅列です。

```
  2           0 LOAD_FAST                0 (x)
              2 LOAD_CONST               1 (2)
              4 BINARY_MULTIPLY
              6 STORE_FAST               1 (y)

  3           8 LOAD_FAST                1 (y)
             10 LOAD_FAST                0 (x)
             12 BINARY_MULTIPLY
             14 STORE_FAST               2 (z)

  4          16 LOAD_GLOBAL              0 (print)
             18 LOAD_FAST                2 (z)
             20 CALL_FUNCTION            1
             22 POP_TOP
             24 LOAD_CONST               0 (None)
             26 RETURN_VALUE
```

これは次のような関数のディスアセンブルでした。 Python はバイトコードの仕様が意図的に安定化されていないので、バージョンによって上記とは異なるディスアセンブル結果になることがあります。

```py
def hello(x):
    y = x * 2
    z = y * x
    print(z)
```

### スタック変数のリスト

現在実行中の関数のスタック変数を表示します。 Mascal では変数はすべてスタックに置かれ、インデックスで参照されます。デバッグ情報があれば、対応する変数の名前も `(Local: xmin)` 等のように表示されます。一時変数など、名前の付かない変数もあるので、すべてのスタック変数に名前が表示されるわけではありません。

![](/images/mascal-tui-stack.png)

ユーザー向けには名前の付いた変数だけを表示する方が分かり易いとはいえますが、インストラクション単位でのステップ実行では、名前のない変数も含めたすべての変数をチェックできるようにする必要があります。

### スタックトレース

Stack trace ウィジェットでは、関数の呼び出しスタックフレームを表示します。

![](/images/mascal-tui-stack-trace.png)

`U` と `D` でデバッガで表示するスタックフレームを移動できます。選択されたスタックフレームの先頭には `>` が表示されます。 `U` で呼び出し元に移動、 `D` で呼び出し先に移動します。 Stack values ウィジェット、 Disassembly ウィジェット、 Source listing ウィジェットはここで選択したスタックフレームの内容を表示します。


### ステップ実行

`S` キーを押すたびにバイトコードのインストラクションを一つずつ実行します。 Disassembly ウィジェットでは現在実行中のインストラクションがハイライトされます。 Source listing ウィジェットでは、デバッグ情報がコンパイルされていればソースマップから実行中のソースコードの位置を示します。これはまだ少し精度が低いです。

![](/images/mascal-tui-step.gif)


### ブレークポイントの設定

Source listing ウィジェットでは、行ごとにブレークポイントの設定ができます。上下キーで行を選択し、 `B` キーでブレークポイントをトグルします。ブレークポイントが設定された行の先頭には `o` がつきます。

![](/images/mascal-tui-breakpoints.gif)


### 実行続行

いわゆる Continue です。次のブレークポイントか、プログラムの終了まで実行します。 `C` キーを押すと実行します。

下のアニメーションでは、マンデルブロー集合のレンダリングの一行ごとにブレークポイントがヒットするようにしています。

![](/images/mascal-tui-continue.gif)


## 高度な機能

### タイムトラベルデバッガ

このデバッガはタイムトラベルデバッガも備えています。タイムトラベルデバッガとは、実行した後で前の実行状態を復元する機能です。不可逆な操作、例えば書き換えた後のメモリを時間を遡って確認したりすることができます。ネイティブコンパイル言語では rr などのツールがあります。

通常のタイムトラベルデバッガは、実行時にログをファイルに吐き出し、それを後ほどデバッガに読み込むことで動作しますが、 Mascal は比較的単純な VM なので、全てインメモリで処理しています。ただし、この機能はメモリを大量に必要としますので、履歴は 100 件までに制限しています。それを超えると古いものから削除されます。

インストラクションを実行するたびに、その時点での VM のスナップショットが履歴に保存されます。 `P` と `N` キーで履歴を行き来できます。下のアニメーションでは、 Stack values の値が書き込まれ、また戻っていく様子がわかります。

![](/images/mascal-tui-timetravel.gif)

この機能は、条件分岐の条件の演算結果や、関数の返却後に、その前段階の条件を後から確認するのに役立ちます。


## 実装の詳細

デバッガによるステップ実行を可能にするためには、まずプログラムを状態マシン化する必要があります。

以前は、バイトコードの「実行」を次のような関数で行っていました。この実体は巨大なループであり、インストラクションを一つずつ読み出し、Opコードに応じて処理を分岐していました。これだと、任意の実行地点で中断し、その時点での変数やスタックフレームを調べることはできません。

```rust
fn interpret_fn(
    bytecode: &FnBytecode,
    functions: &HashMap<String, FnProto>,
) -> Result<Value, EvalError> {
    let mut vm = Vm::new(bytecode.stack_size);
    let mut call_stack = vec![CallInfo {
        // ...
    }];

    while call_stack.clast()?.has_next_inst() {
        let ci = call_stack.clast()?;
        let ip = ci.ip;
        let inst = ci.fun.instructions[ip];
        match inst.op {
            OpCode::LoadLiteral => {
                vm.set(inst.arg1, ci.fun.literals[inst.arg0 as usize].clone());
            }
            OpCode::Move => {
                // ...
            }
            // ...
        }
    }
}
```

状態マシン化とは、この実行状態を定義するすべての変数を状態変数 (`Vm`) の一部にするということです。ここで、外部の関数への参照のために `'a` ライフタイムを導入しています。

```rust
/// The virtual machine state run by the bytecode.
/// Now it is a full state machine that has complete information to suspend and resume at any time,
/// given that the lifetime of the functions are valid.
pub struct Vm<'a> {
    /// A stack for function call stack frames.
    stack: Vec<Value>,
    /// The stack base address of the currently running function.
    stack_base: usize,
    call_stack: Vec<CallInfo<'a>>,
    /// A special register to remember the target index in Set instruction, updated by SetReg instruction.
    /// Similar to x64's RSI or RDI, it indicates the index of the array to set, because we need more arguments than
    /// a fixed length arguments in an instruction for Set operation.
    set_register: usize,
    functions: &'a FnProtos,
}
```

こうすることで、 `Vm` の `next_inst` メソッドでステップ実行が来出ます。通常のスクリプト実行も、 `next_inst` を繰り返し呼び出すように書き換えることができます。

```rust
fn interpret_fn(
    bytecode: &FnBytecode,
    functions: &HashMap<String, FnProto>,
) -> Result<Value, EvalError> {
    let mut vm = Vm::new(bytecode, functions);
    let value = loop {
        if let Some(res) = vm.next_inst()? {
            break res;
        }
    };
    Ok(value)
}
```

### タイムトラベルデバッガ

実行状態の状態マシン化ができれば、タイムトラベルデバッガは簡単に実現できます。それぞれの時点での `Vm` のクローンを覚えておけばよいのです。

デバッガの状態を表す `AppMode` 列挙型は次のように定義されています。実行中であれば `VecDeque<Vm<'a>>` が過去の `Vm` のスナップショットを保持します。

```rust
enum AppMode<'a> {
    None,
    StepRun {
        vm_history: VecDeque<Vm<'a>>,
        // ...
    }
}
```

ところが、 Clone するだけではこのスナップショットにはなりません。 Mascal の配列型は参照 (`Rc`) であり、実体はコピーされないからです。これだと、 Clone 先で配列を書き換えたら、過去の履歴まで書き換わってしまいます。

![](/images/mascal-tui-timetravel.png)

これを解決するには、新たに `deepclone` というインターフェースを `Vm` に追加します。これは、参照型を参照としてコピーするのではなく、参照先の実体のコピーとするセマンティクスの Clone です。

```rust
impl<'a> Vm<'a> {
    pub fn deepclone(&self) -> Self {
        Self {
            stack: self.stack.iter().map(|v| v.deepclone()).collect(),
            stack_base: self.stack_base,
            call_stack: self.call_stack.clone(),
            set_register: self.set_register,
            functions: self.functions,
        }
    }
}
```

`deepclone` の呼び出しは `Value` にも再帰的に行われます。 `Value` のバリアントで参照型は `Array` と `Tuple` だけなので、それ以外は `self.clone()` を呼び出します。

```rust
#[derive(Debug, PartialEq, Clone)]
pub enum Value {
    F64(f64),
    F32(f32),
    I64(i64),
    I32(i32),
    Str(String),
    Array(Rc<RefCell<ArrayInt>>),
    Tuple(Rc<RefCell<TupleInt>>),
}

impl Value {
    pub fn deepclone(&self) -> Self {
        match self {
            Self::Array(a) => {
                let a = a.borrow();
                let values = a.values.iter().map(|v| v.deepclone()).collect();
                Self::Array(Rc::new(RefCell::new(ArrayInt {
                    type_decl: a.type_decl.clone(),
                    values,
                })))
            }
            Self::Tuple(a) => {
                let a = a.borrow();
                let values = a
                    .iter()
                    .map(|v| TupleEntry {
                        decl: v.decl.clone(),
                        value: v.value.deepclone(),
                    })
                    .collect();
                Self::Tuple(Rc::new(RefCell::new(values)))
            }
            _ => self.clone(),
        }
    }
}
```

ただし、この方法にも欠点がないわけではありません。 `deepclone` すると `Vm` の内部の参照先もコピーされてしまうので、参照を通して同じオブジェクトを変更するというセマンティクスが再現できなくなってしまいます(もちろん、メモリ効率も悪いです)。つまり、 `deepclone` した後の `Vm` を使って実行を続行することはできません(実行すると動作に互換性が無くなります)。

![](/images/mascal-tui-timetravel-deepclone.png)

このため、ステップ実行の処理では次のように非常にトリッキーな操作をしています。

まず、通常の `VecDeque` の先頭から最新の `Vm` のオブジェクトを `deepclone` し、それを `push_front` してしまうと、上記の続行してはいけない `Vm` を使ってしまいます。これを避けるため、一度 `VecDeque` の先頭要素を `pop_front()` し、それを `deepclone` し、`push_front()` し戻した後で、オリジナルの `Vm` を `push_front()` しています。

```rust
impl<'a> App<'a> {
    fn run_step(&mut self) -> Result<(), Box<dyn std::error::Error>> {
        if let AppMode::StepRun {
            ref mut vm_history,
            ref mut selected_history,
            ..
        } = self.mode
        {
            let Some(mut next_vm) = vm_history.pop_front() else {
                return Err("Missing Vm".into());
            };
            let prev_vm = next_vm.deepclone();
            next_vm.next_inst()?;
            self.widgets.update(/* ... */)?;
            vm_history.push_front(prev_vm);
            vm_history.push_front(next_vm);
            if 100 < vm_history.len() {
                vm_history.pop_back();
            }

            // Reset history to most recent to reflect real time state
            *selected_history = 0;
        } else {
            // ...
        }
        Ok(())
    }
}
```


### 出力ストリームの処理

標準出力が TUI と競合するのを防ぐため、標準ライブラリ関数を定義する時に次のようにします。まず、標準出力に何かを出力する関数は、 Rust の `println!` などを使って直接出力するのではなく、出力ストリームを取り、 `writeln!` などを使って出力するようにします。例えば、スクリプトでの標準関数 `print` は次のようなシグネチャを取ります。第一引数が出力ストリームを表すトレイトオブジェクトです。

```rust
pub(crate) fn s_print(out: &mut dyn Write, vals: &[Value]) -> EvalResult<Value>;
```

これを関数テーブルに登録するための `std_functions` を次のようにします。複数の標準ライブラリ関数が出力を行うので、 `Rc<RefCell<dyn Write>>` を出力先に取り、リファレンスカウンタ式のオブジェクトのクローンを作り、それぞれの標準ライブラリ関数に move しています。

```rust
pub fn std_functions(
    out: Rc<RefCell<dyn Write>>,
    f: &mut impl FnMut(String, Box<dyn Fn(&[Value]) -> Result<Value, EvalError>>),
) {
    let out2 = out.clone();
    f(
        "print".to_string(),
        Box::new(move |values| {
            let mut borrow = out2.borrow_mut();
            s_print(&mut *borrow, values)
        }),
    );
    let out3 = out.clone();
    f(
        "puts".to_string(),
        Box::new(move |values: &[Value]| -> Result<Value, EvalError> {
            s_puts(&mut *out3.borrow_mut(), values)
        }),
    );
    // ...
}
```

通常スクリプトを実行するときは次のように `std::io::stdout()` で標準出力を取得し、`Rc<RefCell<_>>` に包むことで普通に使うことができます。

```rust
    let out = Rc::new(RefCell::new(std::io::stdout()));
    std_functions(out, ...);
```

インタラクティブデバッガで実行するときは、次のように `Rc<RefCell<Vec<u8>>>` のオブジェクトを作って代わりに渡します。これが動くのは、 `Vec<u8>` が `Write` トレイトを実装しているためです。実際書き込むと、バイト列の最後に追記されます。

```rust
    let output_buffer: Rc<RefCell<Vec<u8>>> = Rc::new(RefCell::new(vec![]));
    std_functions(output_buffer, ...);
```


## 課題

いくつか課題が残っていますが、とりあえず動作するものができたのでここまでの知見をまとめました。

* Ratatui がキーボードイベントを取ってしまうので、無限ループなどの際に Ctrl-C で終了することができない。
* 相互参照している配列があった場合、 `deepclone` は無限再帰によってクラッシュする。
* 行内のインストラクションのマッピングの精度が高くない。
* ウィジェットが固定レイアウトであり、モニタのサイズなどに応じてユーザがカスタマイズできない。
* スタック変数は、ユーザ向けには名前の付いた変数だけを表示するべき。
* ステップアウト、ステップオーバーに相当するステップ実行がなく、常にステップインしてしまう。


## 感想まとめ

* TUI はシンプルなデバッガを作るのには向いています。
* Web UI や Debug Adapter Protocol を実装するまでの「渡り」としても十分機能するでしょう。
* デバッガで実行中の状態をチェックできるようにするためには、バイトコードの実行状態を状態マシン化する必要があります。
* このデバッガのためにリファクタリングを行った結果、コンパイラのバグは解消してしまい、デバッガを役に立てる相手がいなくなっていまいました。まあ、よくあることです。
* 総合的には自作言語へのデバッガには TUI をお勧めできると言えます。
