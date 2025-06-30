---
title: "Rust 1.88 の let chains 構文について"
emoji: "⛓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "構文", "パターンマッチ"]
published: true
---

## Let chains 構文

先週 [Rust 1.88](https://blog.rust-lang.org/2025/06/26/Rust-1.88.0/) がリリースされました。普段はリリースへの反応記事は書かないのですが、今回は前から気になっていた機能が安定化されたので少し触れておきたいと思います。

その機能というのは let chains と呼ばれています。これは一つの条件文に let 式でマッチすると同時に bool 型の条件を混ぜることができる機能です。例えば次のように書くことができます。

```rust
fn do_something_over_18(age: Option<i32>) {
    if let Some(age) = age && 18 < age {
        // do something
    }
}
```

以前は次のように if 式をネストして書く必要がありました。

```rust
fn do_something_over_18(age: Option<i32>) {
    if let Some(age) = age {
        if 18 < age {
            // do something
        }
    }
}
```

これによってブロックのネストを減らし、インデントを不要に深くすることを避けられます。

機能自体は以前からありましたが、 nightly で feature flag を有効にしないと使えませんでした。これからは安定版のプロジェクトでも気兼ねなく使うことができます。ただし、 2021 edition 以前では使えないとのことです。これは else 節のライフタイムに関係しているようで、 2024 edition で真の分岐にのみ条件文に現れる変数のライフタイムが拡張されるようになったことにより可能になった構文とのことです。詳しくは公式ページからのリンクにもある[こちら](https://doc.rust-lang.org/edition-guide/rust-2024/temporary-if-let-scope.html)を参照してください。

しかしながら、これだけの機能ならそれほど大きなインパクトがあるものではありません。 Rust には豊富なコンテナ型のコンビネータがあり、以前から次のように書くこともできたからです。

```rust
fn do_something_over_18(age: Option<i32>) {
    if age.is_some_and(|age| 18 < age) {
        // do something
    }
}
```

実際、 let chains 構文では表現できない `ok_or_else` や `map_or` などのコンビネータもありますので、従来の書き方が不要になるわけではありません。それでも、これらの数多くのコンビネータを暗記しなくても直感的に書けるというのは利点と言えます。

しかし、真価を発揮するのは条件の間に値のバインディングができる点です。例えば、 `Option` 型に包まれた2次元ベクトルの距離が 10 以下である時に何かするようなコードは次のように書けます。

```rust
fn do_something_if_close(a: Option<Vec2>, b: Option<Vec2>) {
    if let Some((a, b)) = a.zip(b)
        && let dist = (a - b).length()
        && dist < 10.
    {
        // do something
    }
}
```

ここで `let dist = (a - b).length()` はマッチに必ず成功するので irrefutable なパターンと呼ばれます。このような irrefutable なパターンも条件式に混ぜて書くことができるのが最大の特徴と言えます。

`(a - b).length()` は `Some` にマッチした後にしか計算できませんので、従来は次のような書き方になってしまいます[^1]。

[^1]: もちろん、この程度の式なら `a.zip(b).is_some_and(|(a, b)| (a - b).length() < 10.)` と書いてしまうとは思います。ここでは簡単のため単純な式を例に使っていますが、実際のコードではコンビネータでは対応できないもっと複雑な式のネストが起きることを想定して書いています。

```rust
fn do_something_if_close(a: Option<Vec2>, b: Option<Vec2>) {
    if let Some((a, b)) = a.zip(b) {
        let dist = (a - b).length();
        if dist < 10. {
            // do something
        }
    }
}
```

let chains の前段でバインドされた変数が以降の条件に使えるので、このような中間的な計算が以降の段に使えます。 OCaml の `let ... in` 構文を彷彿とさせます。

## 実使用例

もう少し実際のコードに近い例を挙げたいと思います。以下のコードは実際のプロジェクトで使用しているものです。やりたいことは内側のブロックの中のコードを条件に応じて実行したいだけなのですが、その条件が複雑すぎるため、一時的な変数 `found_node` を使ったりして条件式が複雑になりすぎないようにしています。また、ネストした条件式の外側の変数を内側のブロック内で使いたいため、 `and_then` のようなコンビネータを使って一段の `if` 式にすることができません。その結果、ロジックが一直線にならず視線を上下に振らないと全体が把握できません。

```rust
let found_node = response.hover_pos().and_then(|pointer| {
    let thresh = SELECT_PIXEL_RADIUS / self.transform.scale() as f64;
    self.tracks
        .find_path_node(paint_transform.from_pos2(pointer), thresh)
});
if let Some((path_id, seg_id, _)) = found_node {
    if let Some(path) = self.tracks.paths.get(&path_id) {
        let color = Color32::from_rgba_premultiplied(127, 0, 127, 63);
        let seg_track = path.seg_track(seg_id);
        self.render_track_detail(seg_track, &painter, &paint_transform, 5., color);
    }
}
```

これを let chains 構文で書き直すと次のようになります。読みやすいかどうかは主観的な評価にはなりますが、少なくとも条件式が一つであり、いくつかのマッチの結果がすべて成功した時にブロック内を実行したいという意図は一目でわかります。

```rust
if let Some(pointer) = response.hover_pos()
    && let thresh = SELECT_PIXEL_RADIUS / self.transform.scale() as f64
    && let Some((path_id, seg_id, _)) = self
        .tracks
        .find_path_node(paint_transform.from_pos2(pointer), thresh)
    && let Some(path) = self.tracks.paths.get(&path_id)
{
    let color = Color32::from_rgba_premultiplied(127, 0, 127, 63);
    let seg_track = path.seg_track(seg_id);
    self.render_track_detail(seg_track, &painter, &paint_transform, 5., color);
}
```

## C++ の条件式と初期化式

条件文の内部にのみスコープを制限したい変数の初期化というのはよくあることで、 C++ でも似たような機能が導入されました。

C++17 での「[条件式と初期化を分離](https://cpprefjp.github.io/lang/cpp17/selection_statements_with_initializer.html)」という機能です。これは `for` 文のように初期化文と条件式をセミコロンで区切ることができるという機能です。

```cpp
if (auto variable = init_variable(); variable.is_ok()) {
    // do something...
}
```

もちろん、 Rust に比べると機能はかなり限定的です。 C++ にはパターンマッチはないので、 bool 値の式を評価することしかできませんし、ネストすることもできません。それでも、このような書き方が求められていることは確かであり、 Rust でもそれが可能になったのは喜ばしいことです。

## まとめ

let chains 構文は可読性を向上するかもしれない有用な機能です。古いコンパイラや edition の互換性を気にする必要がなければ使っていきたい機能です。

また、自作言語でもこのような構文は可能にしたいところです。
