---
title: "ソースコードの複雑度メトリクスについて"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

突然ですが質問です。次の2つの関数の疑似コード、 `foo` と `bar` のどちらが読みやすいと思いますか？変数名は ChatGPT が適当にでっち上げたもので、意味はありません。純粋にコードの見た目の印象を考えてください。

* Foo

```
fn foo() {
    let userScore = init_userScore();
    let maxValue = init_maxValue();
    let tempList;
    let isAvailable = init_isAvailable();
    let countIndex = init_countIndex();
    let totalSum;
    let defaultFlag = init_defaultFlag();
    let itemPrice = init_itemPrice();
    let loadData;

    tempList = some_logic1(userScore, maxValue);

    totalSum = some_logic2(tempList, isAvailable, countIndex);

    loadData = some_logic3(totalSum, defaultFlag, itemPrice);

    return loadData;
}
```

* Bar

```
fn bar() {
    let tempList = {
        let userScore = init_userScore();
        let maxValue = init_maxValue();
        some_logic1(userScore, maxValue)
    };

    let totalSum = {
        let isAvailable = init_isAvailable();
        let countIndex = init_countIndex();
        some_logic2(tempList, isAvailable, countIndex)
    };

    let loadData = {
        let defaultFlag = init_defaultFlag();
        let itemPrice = init_itemPrice();
        some_logic3(totalSum, defaultFlag, itemPrice)
    };

    return loadData;    
}
```

個人の感覚によるところも大きいと思いますが、私は `bar` のほうが読みやすいと感じます。これはなぜなのかを考えてみました。

まず、 `foo` のほうは変数を関数の最初にまとめて宣言かつ初期化しており、それが後のロジックのどこで使われているのかを追うのに距離があります。 `bar` のほうはそれぞれのステップごとに使われている変数が局所化されており、あまり目を動かさなくても論理を追うことができます。 `bar` のほうは局所化されているので、ほとんど何も考えずに関数に分けることもできます。これは「関数を小分けにせよ」という一般的な読みやすいコードのガイドラインになっています。

```
fn bar() {
    let tempList = getTempList();
    let totalSum = getTotalSum(tempList);
    let loadData = getLoadData(totalSum);
    return loadData;
}

fn getTempList() {
    let userScore = init_userScore();
    let maxValue = init_maxValue();
    return some_logic1(userScore, maxValue);
}

fn getTotalSum(tempList) {
    let isAvailable = init_isAvailable();
    let countIndex = init_countIndex();
    return some_logic2(tempList, isAvailable, countIndex);
}

fn loadData(totalSum) {
    let defaultFlag = init_defaultFlag();
    let itemPrice = init_itemPrice();
    some_logic3(totalSum, defaultFlag, itemPrice)
}
```

しかしながら、関数を小分けにするのが常に読みやすさにつながるかというと、そうでもありません。長大なロジックを関数に小分けにしすぎた結果、ロジックを追うのにコードを上下に何度も行き来しなくてはならず、かえって読みにくくなっているのを見たことは何度もあります。

## 名前の出現パターン

人間がコードを読みにくいと感じるのは、変数などの名前が散らばっており、覚えておかなくてはいけない期間が長い時ではないかと思います。この仮説に従って、前述の `foo` と `bar` の名前の出現期間を可視化してみます。以下の図において、変数名は簡単のため出現順にアルファベットで置き換えています。濃い緑がそれぞれの行で出現する変数名、薄い緑がその関数内での最初と最後に出現した行の間の期間を表します[^1]。

[^1]: Rust を知っている人なら、これが Non-lexical lifetime の期間の考え方と同じであることに気づいたと思います。まさに今回の記事は NLL がヒントになっています。

* Foo

![](/images/complexity-foo.png)

* Bar

![](/images/complexity-bar.png)

foo のほうが薄緑の期間が長く、読解により多くのワーキングメモリを必要とします。この名前空間とスコープ空間の積の面積が、認知負荷につながるという仮説が立てられます。

これで、関数を小分けにしすぎるとかえって読みにくくなることの説明も付きます。関数名もワーキングメモリに入れる名前の一つであり、呼び出し個所が離れていると行空間が拡大するからです。

さらに、グローバル変数を避けるべき理由もこれで説明できます。グローバル変数のスコープはソースコード全体であり、ローカル変数に加えて常にワーキングメモリに置かなくてはならないからです。

つまり、最適なロジックの分割は、名前空間とスコープ空間の積の合計を最小化する分割であるという最適化問題に帰着します。

## コードメトリクス

さて、ここまでの話だけなら、ただの「チラシ裏のアイデア」ですが、我々はコードを解析するツールを持っているので、実際に測ることができます。ここでは私の開発している [Mascal]() という言語を対象に試してみます。

具体的には、次のように名前と出現範囲のマップを `NameRangeMap` という型で定義し、 AST をスキャンして範囲を調べる `complexity` という関数を用意します。 Mascal この範囲は行単位で、あまり粒度は高くありませんが、人間の認知能力を測るのでむしろ大体の値を見積もるという意味では適しています。

```rust
type NameRangeMap = HashMap<String, (u32, u32)>;

pub fn complexity(ast: &[Statement]) -> usize {
    let mut names = NameRangeMap::new();
    let sum = stmts_complexity(ast, &mut names);
    sum + names
        .iter()
        .fold(0, |acc, (_, val)| acc + (val.1 - val.0) as usize)
}
```

## その他のコードメトリクス

そもそも、なぜこの記事を書こうかと思ったかというと、コードメトリクスという概念は昔から存在しており、コードの複雑性を評価する指標としていろいろなところで使われているのですが、どうもこれらのメトリクスが読みやすさを体現しているとは思えないのです。

代表的なメトリクスにはサイクロマティック複雑度やファンクションポイントなどがありますが、これらはロジックのフローグラフの複雑度を測るものであり、「名前の多さ」は問題にしていません。確かにフローの分岐が多いコードは読みにくいですが、それよりも変数の名前の局所性で読みにくさを感じることが多いように思います。
