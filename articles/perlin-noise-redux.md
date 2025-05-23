---
title: "パーリンノイズともう一度真剣に向き合う"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["アルゴリズム", "パーリンノイズ"]
published: false
---

パーリンノイズは自然な地形を自動生成するのに使われるアルゴリズムですが、実のところ、色々誤解や良くない実装が使われることの多いアルゴリズムです。何を隠そう、私自身も過去の記事で[不正確な言及](https://zenn.dev/msakuta/articles/ef996762a9daf2)をしてしまいました。今回は自戒も兼ねて整理したいと思います。

まず最初に、パーリンノイズとフラクタルノイズは別物です。パーリンノイズは Ken Perlin 氏が考案した、滑らかな曲線を描くようなノイズです。

![Perlin noise](/images/noise.jpg)

このノイズの特性は後ほど詳しく調べるとして、まずはこのノイズだけでは「自然な地形」を描くには程遠いことが分かると思います。

これに対して、フラクタルノイズは同じノイズをスケールを変えながら重ねることによって自己相似性を持たせ、自然の地形のような特性を持たせるものです。まず、ベースとなるノイズ源が必要ですが、ここでは次のような一様なホワイトノイズをベースとします。

![White noise](/images/white-noise.png)

これを複数のスケールで重ね合わせることによって次のようなフラクタルノイズが生成できます。下の例ではオクターブ(スケールを半分にして重ね合わせる回数)は 5 、 persistence (一段小さなスケールのノイズの振幅比率)は 0.5 のパラメータで生成しています。

![White fractal noise](/images/white-fractal-noise.png)

もちろん、パーリンノイズをフラクタルノイズのベースに使うこともできます。しかし、前述のホワイトノイズを使ったフラクタルノイズと同じパラメータを使った限り、それほど大きな違いはなさそうに見えます。

![Perlin fractal noise](/images/perlin-fractal-noise.png)

この他にも多次元の分布や計算量を改善した Simplex Noise などのアルゴリズムが存在しますが、本稿ではパーリンノイズに集中します。
