---
title: "[Rust] プロシージャル生成向け非同期・並列タスクランナーの実装"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

新しい Rust プロジェクトで地形生成をするときに、並列処理でパフォーマンスを向上させたくなりました。

先に結果から提示すると、次のようにズームやパンによって視界に入った部分の地形を順次生成していくのが見て取れると思います。

https://youtu.be/Dk_N4domMHU

これが並列化前だと次のようになります。順次処理されていく様子が見て取れ、前の動画よりも明らかに遅いことが分かると思います。

https://youtu.be/s1nbg-eHYaY

GitHub リポジトリは以下ですが、開発中のため内容はまとまっていません。どんなアプリケーションなのかは別の記事でまとめたいと思います。

https://github.com/msakuta/trains-rs?tab=readme-ov-file


# 動的地形生成

これだけでも面白いトピックですが、長くなりそうなので本件の主題である並列処理に関係する部分だけ説明します。

動的生成はプロシージャル生成とも呼ばれ、事前に地形などのデータを保存しておくのではなく、必要になったタイミングでアルゴリズムによって生成するアプローチです。この手法を使っているゲームで最も有名なのは Minecraft でしょう。

本アプリケーションは 2D であり、プレイヤーの視点の近くだけではなく、ズームレベルも重要になります。例えば、最もズームレベルの低い(=視界の広い)状態で地形の全てを生成すると、見えないほど小さな地形を生成することになり無駄が多いです。このため、一般的な 3D ゲームとは異なり、次のようなキーでタイルの集合を管理しています。元々標高マップの動的生成だったので `HeightMapKey` という名前にしていますが、今となっては標高以外にも資源の分布なども生成しているので若干ネーミングに改善の余地があります。

```rust
pub(crate) struct HeightMapKey {
    pub level: usize,
    pub pos: [i32; 2],
}
```

`level` がズームレベルに相当し、 0 が最も詳細なレベルで、一つ上がるごとにタイルサイズが2倍になります。 `pos` はそのズームレベルでのタイルの位置です。

「タイル」というのは地形を一定のサイズに区切った単位で、次のように定義しています。 `map` フィールドが標高マップをフラット化した配列で、 `contours` は等高線のレンダリング用のキャッシュです。

```rust
pub(crate) struct HeightMapTile {
    pub map: Vec<f32>,
    pub contours: ContoursCache,
}
```

上記 2 つをハッシュマップの Key, Value とすることでタイルの集合を管理しています。

```rust
tiles: HashMap<HeightMapKey, HeightMapTile>
```

ここでの問題は、 `HeightMapTile::map` を生成するのに計算負荷が大きく、事前に計算しておくにはサイズが大きくなりすぎるという点です。また、世界のサイズを事実上無限にしたかったので、画面がスクロールするにつれタイルを生成していくという手法を取りたいところです。


# 非同期並列処理

Rust で並列処理によるパフォーマンス向上という観点で外せないのが [rayon](https://docs.rs/rayon/latest/rayon/) というクレートです。これはループを自動で並列化してくれる優れたライブラリですが、本アプリケーションでは簡単には使えません。なぜなら、 rayon は同期ロジックでループを効率化するライブラリであり、本件のように任意のタイミングでタイルの生成が必要になり、タスクキューに登録するような使い方は想定していないからです。別の言い方をすると、本アプリケーションでは非同期処理が必要になります。

非同期処理で関連するのは `async` ロジックですが、これは主に I/O の非同期ロジックに使われるもので、 `tokio` などのランタイムも必要になるので、今回の要件にはオーバーキルです。

それではどうすればよいでしょうか。今回はスレッドプールで非同期タスクランナーを自作することにしました。

スレッドプールというと難しく聞こえるかもしれませんが、並列化に関する部分だけ取り出した[こちらのソース](https://github.com/msakuta/trains-rs/blob/master/src/app/heightmap/parallel_tile_gen.rs)を見てもらえばわかる通り、 100 行にも満たない簡単なものです。

本記事の以降ではこのタスクランナーの動作を解説していきます。


# 非同期タスクランナーの実装

まず、タスクランナーに必要なデータは次の構造体で定義されています。

```rust
/// The tile generator that can run tasks in parallel threads.
/// It is very high performance but does not work in Wasm.
/// (Maybe there is a way to use it on Wasm with SharedMemoryBuffer,
/// but it seems tooling and browser support are not readily available.)
pub(super) struct TileGen {
    /// The set of already requested tiles, used to avoid duplicate requests.
    requested: HashSet<HeightMapKey>,
    /// Request queue and the event to notify workers.
    notify: Arc<(Mutex<VecDeque<(HeightMapKey, usize)>>, Condvar)>,
    // We should join the handles, but we can also leave them and kill the process, since the threads only interact
    // with memory.
    _workers: Vec<std::thread::JoinHandle<()>>,
    /// The channel to receive finished tiles.
    finished: std::sync::mpsc::Receiver<(HeightMapKey, HeightMapTile)>,
}
```

`requested` は、すでにタスクランナーに要求したタイルを覚えておいて2度要求することを防ぐためのフィールドです。メインスレッドでのみアクセスするので Mutex は必要ありません。

`notify` はワーカースレッドに要求がきたことを伝えるための条件変数 (Condition variable) です。

`_workers` はワーカースレッドの JoinHandle の集合です。スレッドがきれいに終了してからアプリケーションを終了するためにはこれらのハンドルを Join する必要がありますが、本アプリケーションではワーカースレッドの仕事はメモリにしか反映されないので、計算中に強制的に中断しても弊害はなく、今のところ使っていません[^1]。

[^1]: これがもしファイルやネットワークへの出力を含む場合は、正しく終了したことを確認してからプログラムを終了する必要があります。さもないと中途半端に壊れたファイルが生成されたりする可能性があります。プロセスメモリはプログラム終了と同時に破棄されるので、そのような弊害はありません。

`finished` は計算が終了したタスクをメインスレッドが受け取るためのチャンネルです。


## スレッドプールの初期化

スレッドプールの初期化は `TileGen::new` 関数で行います。簡単化すると次のようになります。

```rust
impl TileGen {
    pub fn new(params: &HeightMapParams) -> Self {
        let num_threads = std::thread::available_parallelism().map_or(8, |v| v.into());
        let notify = Arc::new((Mutex::new(VecDeque::new()), Condvar::new()));
        let (finished_tx, finished_rx) = std::sync::mpsc::channel();
        let workers = (0..num_threads)
            .map(|_| {
                let params_copy = params.clone();
                let notify_copy = notify.clone();
                let finished_tx_copy = finished_tx.clone();
                std::thread::spawn(move || {
                    // ...
                })
            })
            .collect();
        Self {
            requested: HashSet::new(),
            notify,
            _workers: workers,
            finished: finished_rx,
        }
    }
}
```

まず、スレッドの数を `std::thread::available_parallelism()` で初期化します。取得できなければとりあえず 8 スレッドにします。

次に、リクエスト送信用キューと条件変数を `notify` に初期化します。

次に、終了したタスクを返すチャンネルを `finished_tx, finished_rx` というペアで生成します。

次に、ワーカースレッドを `num_threads` だけ生成して `workers` に記憶します。ここで `std::thread::spawn` の中身はかなり肝なので後で解説します。
