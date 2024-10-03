---
title: "[Rust] コンパイラ最適化 - 定数畳み込み"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust", "自作言語", "最適化"]
published: true
---

Rust で作るプログラミング言語シリーズです。

https://www.amazon.co.jp/dp/4297141922

コンパイラによる最適化というと、実行バイナリを高速化する技術であり、程度の差こそあれ、事実上すべてのネイティブコンパイル言語に備わっている機能です。

書籍では紙面の都合で紹介できなかったのですが、要望もあったので補足しておきます。

一口に最適化と言っても、それが適用されるタイミングによって様々な手法があり、実際にはそれを組み合わせたものになります。例えば：

* AST最適化
  * 定数畳み込み・伝搬
  * ループの展開
  * インライン展開
* 中間コード最適化
* 機械語コードの最適化
* リンク時最適化
* CPUによる最適化

などです。

他には機械コード依存か、機械コード非依存の最適化という分類もあります。本稿では機械コード非依存の最適化を扱います。

本稿では最も基本となる定数畳み込み・伝搬を扱います。コードは Ruscal のリポジトリにプッシュしてあります。

https://github.com/msakuta/ruscal

主に [optimizer.rs](https://github.com/msakuta/ruscal/blob/master/src/optimizer.rs) を参照してください。

Ruscal はバイトコードインタプリタ言語であり、ネイティブコンパイル言語ではありませんが、基本的なアイデアはネイティブコンパイル言語にも使いまわせると思います。


## 最適化の基本ルール

すべての最適化に共通する重要な前提条件として、**プログラムの意味を変えてはならない**というものがあります。どれほど高速になっても、プログラムの意味が変わってしまうのであればそれは間違った最適化です。

プログラミング言語によっては、オプティマイザが変えても良い挙動をあらかじめ規定しているものがあります。このような挙動に依存するプログラムを書いた結果、予測不可能な動作が生じる場合を **未定義動作** (**Undefined behavior**) といいます[^1]。例えば、 Rust で同じオブジェクトに対して複数の可変参照を作った時は(unsafe を使わないと起こせませんが)未定義動作です。

[^1]: 全ての未定義動作がオプティマイザによって変わる動作とは限りません。例えば C 言語でバッファオーバーランを起こしたときの動作はオプティマイザの如何に関わらず未定義動作です。ほどんどの場合が segfault でしょう。

また、予測される動作のうちいずれかが起こると規定されているものを**未規定動作** (**Unspecified behavior**) といいます[^2]。例えば C 言語や C++ では関数の引数が評価される順番は規定されていません。第一引数から順に評価されるのか、最後の引数から評価されるかはコンパイラ依存で、最適化の具合によって変わるかもしれません。とはいえ、どちらかが正しく起こることはわかっているので、ほとんどのプログラムでは問題になりません。評価順に依存する副作用を持つ実引数を与えたときだけ問題になります。

[^2]: Unspecified behavior については、日本語で呼ばれることが非常に少なく、 Wikipedia の未定義動作の一部で触れられている以外に文献で見たことがありません。


## AST最適化の基本構成

まずは、最適化をコンパイルのどのタイミングで適用するかを考えます。定数畳み込みは AST の定数計算のサブツリーを定数リテラルで置き換える操作なので、ソースのパースが終わり、 AST が構築された後、ただし中間コードへコンパイルする前である必要があります。型チェックも同じタイミングで走りますが、最適化と型チェックはどちらが先でもかまいません。以下はかなり概念的な最適化関数 `optimize` の挿入位置です。最適化はデバッグ時にはオフにしたいのでコンパイルスイッチで適用しないこともできるようにしておきます。

```diff rust
 pub fn write_program(
   ..
 ) -> Result<(), Box<dyn std::error::Error>> {
   let mut compiler = Compiler::new();
   let mut stmts = parse_program(source_file, source)?;

+  if args.optimize {
+    optimize(&mut stmts)?;
+  }

   let mut tc_ctx = TypeCheckContext::new();
   for (fname, f) in &args.additional_funcs {
     tc_ctx.add_fn(fname.clone(), f());
   }

   compiler.compile(&stmts)?;

   compiler.write_funcs(writer)?;
 }
```

次に、最適化ロジックが満たすべきインターフェースを考えてみます。 AST を変更するには、純粋な関数型言語のアプローチでは、 AST を引数に取り、改変した AST を返す次のようなインターフェースを思いつきます。

```rust
pub fn optimize(ast: &[Statement]) -> Result<Vec<Statement>, String>;
```

しかし、実際の AST はほとんど変更せず、一部を単純化したもので置き換えるだけなので、可変参照を渡して直接更新する方が効率的でしょう。

```rust
pub fn optimize(ast: &mut Statements) -> Result<(), String>;
```

この関数を最適化コンパイルスイッチがオンの場合のみ呼び出すようにすれば、入り口の定義は終了です。

```rust
if args.optimize {
    optimize(&mut stmts)?;
}
```


## 定数折り畳み

次に、具体的に定数折り畳みがどのような AST の変形を施すかを考えてみます。まずは AST 内の全ての文をスキャンし、式を含むものであればその式を最適化する関数を呼び出します。最も直感的なのは式文 (`Statement::Expression`) ですが、その他にも変数定義 `Statement::VarDef` や変数代入文 `Statement::VarAssign` が式を含みます。

```rust
pub fn optimize(ast: &mut Statements) -> Result<(), String> {
  for stmt in ast {
    match stmt {
      Statement::Expression(ex) => optim_expr(ex)?,
      Statement::VarDef { ex, .. } => optim_expr(ex)?,
      Statement::VarAssign { ex, .. } => optim_expr(ex)?,
      _ => {}
    }
  }
  Ok(())
}
```

これを処理する関数を次のように定義します。現時点では単純に「定数式であればその値で置き換える」という処理です。定数式かどうかは `const_expr` 関数で定義します。

```rust
fn optim_expr(expr: &mut Expression) -> Result<(), String> {
  if let Some(val) = const_expr(expr) {
    println!("optim const_expr: {val:?}");
    *expr = val;
  }
  Ok(())
}
```

さて、ここで肝となる定数式の定義を `const_expr` とします。この関数は引数の式が定数のみで構成されている場合（変数や実引数を含まない場合）その定数を `Some` で返し、そうでなければ `None` を返します。少し特徴的なのは式の値を AST のサブツリー (`Expression`) で返すということで、値型 (`Value`) は返しません。後ほどサブツリーを置き換えることを考えるとこの方が使い勝手が良いです。

```rust
fn const_expr<'a>(
  expr: &Expression<'a>,
) -> Option<Expression<'a>> {
  Some(match &expr.expr {
    ExprEnum::NumLiteral(_) => expr.clone(),
    ExprEnum::StrLiteral(_) => expr.clone(),
    ExprEnum::Add(lhs, rhs) => optim_bin_op(
      |lhs, rhs| lhs + rhs,
      |lhs, rhs| Some(lhs.to_owned() + rhs),
      lhs,
      rhs,
      expr.span,
    )?,
    ExprEnum::Sub(lhs, rhs) => optim_bin_op(
      |lhs, rhs| lhs - rhs,
      |lhs, rhs| None,
      lhs,
      rhs,
      expr.span,
    )?,
    // ..
    ExprEnum::FnInvoke(span, args) => {
      let args = args
        .iter()
        .map(|arg| {
          const_expr(arg).unwrap_or_else(|| arg.clone())
        })
        .collect();
      Expression::new(
        ExprEnum::FnInvoke(*span, args),
        expr.span,
      )
    }
    _ => return None,
  })
}
```

リテラルである `NumLiteral` や `StrLiteral` はそのまま返すのは当然とわかりますが、四則演算のロジックは繰り返しが多いので `optim_bin_op` というジェネリック関数でまとめています。この関数の第一引数は数値リテラル同士の演算を定義するラムダ式、第二引数は文字列同士です。文字列は `+` 演算子しか許されていないので、 `Option` を返すラムダ式となっており、 `-`, `*`, `/` 演算子は `None` を返します。

二項演算子の処理 `optim_bin_op` は以下の通りです。

```rust
fn optim_bin_op<'a>(
  op_num: impl Fn(f64, f64) -> f64,
  op_str: impl Fn(&str, &str) -> Option<String>,
  lhs: &Expression<'a>,
  rhs: &Expression<'a>,
  span: Span<'a>,
  constants: &Constants<'a>,
) -> Option<Expression<'a>> {
  use ExprEnum::*;
  if let Some((lhs, rhs)) =
    const_expr(&lhs, constants).zip(const_expr(&rhs, constants))
  {
    match (&lhs.expr, &rhs.expr) {
      (NumLiteral(lhs), NumLiteral(rhs)) => Some(
        Expression::new(NumLiteral(op_num(*lhs, *rhs)), span),
      ),
      (StrLiteral(lhs), StrLiteral(rhs)) => op_str(lhs, rhs)
        .map(|ex| Expression::new(StrLiteral(ex), span)),
      _ => None,
    }
  } else {
    None
  }
}
```

## AST の出力

さて、この辺で最適化が上手く機能しているかを可視化するための機能を拡充します。今まではコマンドラインで `-a` オプションを渡したときに `Debug` トレイトによって AST の中身を出力していましたが、これだと以下のように非常に冗長な表現になり、むしろ理解を阻みます。

```
AST: [
    Expression(
        Expression {
            expr: FnInvoke(
                LocatedSpan {
                    offset: 0,
                    line: 1,
                    fragment: "print",
                    extra: (),
                },
                [
                    Expression {
                        expr: Sub(
                            Expression {
                                expr: Mul(
                                    Expression {
                                        expr: Add(
                                            Expression {
                                                expr: NumLiteral(
                                                    1.0,
                                                ),
                                                span: LocatedSpan {
                                                    offset: 7,
                                                    line: 1,
                                                    fragment: "1",
                                                    extra: (),
                                                },
                                            },
                                            ...
```

ここで AST をソースコードのようにフォーマットするロジックを `Statement::format` および `Expression::format` メソッドとして定義しておきます。これはパースの逆変換(ソースから AST へ変換する代わりに、 AST からソースへ変換)と考えると分かりやすいでしょう。詳細は省きますが、これで最適化前後が次のように出力されます。スクリプト例: [34-optim_num.rscl](https://github.com/msakuta/ruscal/blob/master/scripts/34-optim_num.rscl)

```
AST:
  print((((1+2)*3)-4));
AST after optimization:
  print(5);
```

もちろん、文字列式も折りたためます。スクリプト例: [34-optim_str.rscl](https://github.com/msakuta/ruscal/blob/master/scripts/34-optim_str.rscl)

```
AST:
  print(("1"+"2"));
AST after optimization:
  print("12");
```


## 定数伝搬

定数の折り畳みは最も基本的な AST の最適化でした。もう一歩進んだ最適化が定数伝搬です。これは定数式で初期化された変数が以降の式で出てきたら定数で置き換えるというものです。

まずは各時点で定義されている定数の名前と値の集合を `Constants` という型で持っておきます。

```rust
type Constants<'a> = HashMap<String, Expression<'a>>;
```

これを最適化エントリポイント `optimize` 関数の中で初期化して `optim_expr` へ渡します。変数定義が定数式で初期化されていればそれを定数テーブルへ記録します。また、変数への再代入があればその時点での定数式であるかどうかを再度評価します。

```diff rust
 pub fn optimize(ast: &mut Statements) -> Result<(), String> {
+  let mut constants = Constants::new();
   for stmt in ast {
     match stmt {
-      Statement::Expression(ex) => optim_expr(ex)?,
-      Statement::VarAssign { ex, .. } => optim_expr(ex)?,
+      Statement::Expression(ex) => optim_expr(ex, &constants)?,
+      Statement::VarDef { name, ex, .. } => {
+        optim_expr(ex, &constants)?;
+        if let Some(ex) = const_expr(ex, &constants) {
+          constants.insert(name.to_string(), ex);
+        }
+      }
+      Statement::VarAssign { name, ex, .. } => {
+        optim_expr(ex, &constants)?;
+        if let Some(ex) = const_expr(ex, &constants) {
+          constants.insert(name.to_string(), ex);
+        }
+      }
       _ => {}
     }
   }
   Ok(())
 }
```

`optim_expr` の中身は変わらず、 `const_expr` に定数テーブルを渡すだけです。

```diff rust
-fn optim_expr(expr: &mut Expression) -> Result<(), String> {
-  if let Some(val) = const_expr(expr) {
+fn optim_expr<'a>(
+  expr: &mut Expression<'a>,
+  constants: &Constants<'a>,
+) -> Result<(), String> {
+  if let Some(val) = const_expr(expr, constants) {
     println!("optim const_expr: {val:?}");
     *expr = val;
   }
```

`const_expr` は定数テーブルを使って変数名が出てきたら定数で置き換えるということをします。

```diff rust
 fn const_expr<'a>(
   expr: &Expression<'a>,
+  constants: &Constants<'a>,
 ) -> Option<Expression<'a>> {
   match &expr.expr {
+    ExprEnum::Ident(name) => constants.get(**name).cloned(),
     ExprEnum::NumLiteral(_) => Some(expr.clone()),
     ExprEnum::StrLiteral(_) => Some(expr.clone()),
     ExprEnum::Add(lhs, rhs) => optim_bin_op(
```

これをテストするため、次のようなコードを最適化に渡してみます。

```
var a: f64 = 1 + 2;     // (1)
print(a * 3);           // (2)
a = 40 + 2;             // (3)
print(a * 1);

fn non_const() -> f64 {
    90 + 6
}

a = non_const();        // (4)

print(a);
```

(1) `a` に定数式 `1 + 2` を代入し、それを (2) 定数式 `print(a * 3)` で使っています。また再代入 (3) (`a = 40 + 2`) をしています。 (4) の関数の評価は非定数式とみなし、再代入によって最適化が無効になるはずです。

最適化前後で次のようになりました。

```
AST: 
  var a: f64 = (1+2);
  print((a*3));
  a = (40+2);
  print((a*1));

  fn non_const() -> f64 {
    (90+6);
  }

  a = non_const();
  print(a);
AST after optimization:
  var a: f64 = 3;
  print(9);
  a = 42;
  print(42);

  fn non_const() -> f64 {
    96;
  }

  a = non_const();
  print(a);
```

### 制御構文内の定数伝搬

さて、少しややこしいのがループ内の定数伝搬です。今まではプログラムの流れが上から下へ順番であったので、定数テーブルを更新するのも上から下でシンプルでした。しかしループの場合は処理の流れが巻き戻るので、必ずしもその順に定数を評価すればいいわけではありません。例えば、次のコードを見てください。

```
var a: f64 = 1;
print(a);          // (1)

for i in 1 to 10 {
    print(a);      // (2)
    a = a + i;
}

a = 42;
print(a);          // (3)
```

(1) の時点で `a` は明らかに定数ですが、 (2) の時点ではどうでしょうか？ループの最初のイテレーションは定数ですが、二回目以降は `i` の値に依存するため定数ではなくなります。このような変数は一般に定数で置き換えることはできません。

ところが、 (3) の時点ではどうでしょうか。再度定数 `42` が代入されているので、これは定数式となります。

もう一つの例が、条件分岐です (`non_const()` は非定数式と見なします)。

```
var a: f64 = 24;

if non_const() {
    print(a);     // (1)
} else {
    a = 42;
    print(a);     // (2)
}

print(a);         // (3)
```

(1) と (2) の時点では `a` はそれぞれ定数式ですが、 (3) の時点ではそうではありません。 `non_const()` の返り値次第で変わるからです。

このように、制御構文が関わってくると最適化は非常にややこしくなります。一般に、処理の分岐がない一連のコードを基本ブロック (Basic block) といい、基本ブロック内で完結する最適化は局所論理で簡単にできますが、基本ブロックを跨ぐ最適化は大域的なデータフロー解析が必要になります。この辺から最適化のための解析とコンパイル時間のトレードオフが生じ始めます。

本稿では「基本ブロックを跨ぐ(ループや分岐がある)場合にはその中に登場する変数を定数テーブルから削除する」という単純な戦略でこれを回避することにします。これは次の `check_assigned` と `check_assigned_expr` で実施します。

```rust
/// Check if there is a variable assigned in this statements and remove it from the constant table if exists.
fn check_assigned(
  stmts: &[Statement],
  constants: &mut Constants,
) {
  for stmt in stmts {
    match stmt {
      Statement::Expression(ex)
      | Statement::Return(ex)
      | Statement::Yield(ex) => {
        check_assigned_expr(ex, constants);
      }
      Statement::VarDef { name, ex, .. }
      | Statement::VarAssign { name, ex, .. } => {
        check_assigned_expr(ex, constants);
        constants.remove(**name);
      }
      Statement::For {
        start, end, stmts, ..
      } => {
        check_assigned_expr(start, constants);
        check_assigned_expr(end, constants);
        check_assigned(stmts, constants);
      }
      _ => {}
    }
  }
}

/// Check if there is a variable assigned in this sub-expression and remove it from the constant table if exists.
fn check_assigned_expr(
  expr: &Expression,
  constants: &mut Constants,
) {
  match &expr.expr {
    ExprEnum::FnInvoke(_, args) => {
      for arg in args {
        check_assigned_expr(arg, constants);
      }
    }
    ExprEnum::Add(lhs, rhs)
    | ExprEnum::Sub(lhs, rhs)
    | ExprEnum::Mul(lhs, rhs)
    | ExprEnum::Div(lhs, rhs)
    | ExprEnum::Gt(lhs, rhs)
    | ExprEnum::Lt(lhs, rhs) => {
      check_assigned_expr(lhs, constants);
      check_assigned_expr(rhs, constants);
    }
    ExprEnum::If(cond, t_branch, f_branch) => {
      check_assigned_expr(cond, constants);
      check_assigned(t_branch, constants);
      if let Some(f_branch) = f_branch {
        check_assigned(f_branch, constants);
      }
    }
    ExprEnum::Await(ex) => check_assigned_expr(ex, constants),
    _ => {}
  }
}
```

これを `optim_expr` の中で条件分岐だった時に呼び出します。

```diff rust
 fn optim_expr<'a>(
   expr: &mut Expression<'a>,
-  constants: &Constants<'a>,
+  constants: &mut Constants<'a>,
 ) -> Result<(), String> {
+  if let ExprEnum::If(_, t_branch, f_branch) = &expr.expr {
+    check_assigned(t_branch, constants);
+    if let Some(f_branch) = f_branch {
+      check_assigned(f_branch, constants);
+    }
+  }
+
   if let Some(val) = const_expr(expr, constants) {
     *expr = val;
   }
```

また、ループの場合も呼び出します。

```diff rust
 pub fn optimize(ast: &mut Statements) -> Result<(), String> {
   // ...
   for stmt in ast {
     match stmt {
       // ...
+      For { stmts, .. } => {
+        // Variables assigned in a loop is generally not a constant, so let's remove them.
+        check_assigned(stmts, &mut constants);
+      }
       _ => {}
     }
   }
 }
```

しかしながら、これでは最も最適化したいループの中身が最適化できないので、効果は限定的です。プログラムの実行時間を大幅に増やすのは、ループか再帰呼び出ししかありませんので、ループの最適化こそが最も効果が期待できます。

基本ブロックを跨ぐ最適化は長くなりそうなので次の記事にします^^;
