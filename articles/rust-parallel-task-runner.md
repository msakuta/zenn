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

タイルのサイズは次の定数で定義しています。これは1軸のサイズであり、2次元のタイルの配列のサイズはこの二乗になります。

```rust
pub(super) const TILE_SIZE: usize = 128;
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

スレッドプールというと難しく聞こえるかもしれませんが、並列化に関する部分だけ取り出した[こちらのソース](https://github.com/msakuta/trains-rs/blob/e2666b0397d7ff678a32c6bb222b2d39a7a3e75a/src/app/heightmap/parallel_tile_gen.rs)を見てもらえばわかる通り、 100 行にも満たない簡単なものです。

本記事の以降ではこのタスクランナーの動作を解説していきます。本アプリケーションは開発中なので実装は以降で説明しているものから変わっているかもしれませんのでご了承ください。


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

`_workers` はワーカースレッドの JoinHandle の集合です。スレッドがきれいに終了してからアプリケーションを終了するためにはこれらのハンドルを Join する必要がありますが、そうしていない理由については[後述](#join-はいらないのか？)します。

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

ここまでで気づいた人もいるかもしれませんが、ワーカースレッドへタスクを送るキューと結果を受け取るキューの型がずいぶん異なることが分かると思います。前者は `Arc<(Mutex<VecDeque<(HeightMapKey, usize)>>, Condvar)>` というやたら仰々しい型ですが、後者は `std::sync::mpsc::Receiver<(HeightMapKey, HeightMapTile)>` という、見た目の通り結果を受け取るチャンネルです。この理由は簡単で、前者がメインスレッドから複数のワーカースレッドへ要求を分配する (Single Producer Multiple Consumer) メッセージキューであるのに対し、後者は複数のスレッドから一つのメインスレッドへ結果を集約する (Multiple Producer Single Consumer) という動作であるためです。なお、 `mpsc` というのは Multiple Producer Single Consumer を意味します。 Single Producer Multiple Consumer (またはより一般に Multiple Producer Multiple Consumer) なチャンネルは安定化されていません[^2]。

[^2]: 本アプリケーションで使いたいようなタスクの分配キューは [crossbeam クレート](https://docs.rs/crossbeam/latest/crossbeam/) に [deque](https://docs.rs/crossbeam/latest/crossbeam/deque/index.html) という名前でも提供されていますが、前述のように標準ライブラリで実装するのも100行足らずで済むので、外部クレートに依存するほどのことでもないと判断しました。

下記の 3 つの変数はワーカースレッドへ移動する前に `.clone()` でコピーされています。これは Rust ではライフタイムが `thread::scope` で制限されていないスレッドについては参照で共有できないからです。共有する変数は `Arc` で囲んだうえで `.clone()` しています。 Rust でマルチスレッドを触ったことのある人なら恐らく説明するまでもないぐらいよくあるパターンです。

```rust
let params_copy = params.clone();
let notify_copy = notify.clone();
let finished_tx_copy = finished_tx.clone();
```

## ワーカースレッドの中身

前節で省略したワーカースレッドの中身は次のようになっています。

```rust
std::thread::spawn(move || {
    let (lock, cvar) = &*notify_copy;
    loop {
        let mut queue = lock.lock().unwrap();
        while queue.is_empty() {
            queue = cvar.wait(queue).unwrap();
        }

        if let Some((key, contour_grid_step)) = queue.pop_back() {
            // Drop the lock just before heavy lifting
            drop(queue);
            let tile = HeightMapTile::new(
                HeightMap::new_map(&params_copy, &key).unwrap(),
                contour_grid_step,
            );
            finished_tx_copy.send((key, tile)).unwrap();
        }
    }
})
```

まず最初の行を解読します。

```rust
let (lock, cvar) = &*notify_copy;
```

これは `notify_copy` の中身への参照を分解してローカル変数へバインドするコードです。 `lock` は `&Mutex<VecDeque<(HeightMapKey, usize)>>` という型を持ち、タスク要求キューを表します。複数のスレッドで読み書きするため `Mutex` で包んであります。 `cvar` は条件変数への参照 `&CondVar` です。

条件変数とは、マルチスレッドプログラミングで使われる同期プリミティブで、「スレッド間イベント」のようなものです。 Rust では標準ライブラリの [`CondVar`](https://doc.rust-lang.org/std/sync/struct.Condvar.html) で提供されていますが、 C++ では `std::condition_variable` という名前になっています。

`CondVar` の `.wait()` メソッドでスレッドを待ち状態にすることができ、他のスレッドから `.notify_one()` や `.notify_all()` が来るまでブロックします。これを使わないと、ワーカースレッド側でいつタスク要求キューにタスクが挿入されるかわからないので、ビジーループで待つ必要があり、 CPU を無駄に使うことになります。

条件変数に関して特徴的なのは、`.notify_*()` が呼び出されなくてもときどき続行してしまう (spurious wakeups) ことが起こりうるということです。このため、実際のタスクキューを確認して本当にイベントが挿入されているかを確認する必要があります。

もう一つの特徴として、条件変数は常に Mutex と一緒に使われることで、実際のイベントが起きたか否かを示す変数はこの Mutex の中に入れる必要があります。我々のケースでは `lock` がそれにあたります。条件変数はイベントを受信すると同時に Mutex をロックする機能を備えており、タスク要求キューを直接確認することができます。ここまでの動作が下記の 3 行です。

```rust
let mut queue = lock.lock().unwrap();
while queue.is_empty() {
    queue = cvar.wait(queue).unwrap();
}
```

タスク要求キューに何かが入っていることが確認されたら、それを以下のように `pop_back` してタイルの生成を実行します。ここで 3 行目に `drop(queue)` はロジックの正確性という意味では必要ありませんが、タスク要求キューをできる限り早く解放することで他のワーカースレッドがタスクを受け取るチャンスを増やすことを目的としています。

```rust
if let Some((key, contour_grid_step)) = queue.pop_back() {
    // Drop the lock just before heavy lifting
    drop(queue);
    let tile = HeightMapTile::new(
        HeightMap::new_map(&params_copy, &key).unwrap(),
        contour_grid_step,
    );
    finished_tx_copy.send((key, tile)).unwrap();
}
```

最後の行 `finished_tx_copy.send((key, tile)).unwrap();` では計算結果を完了タスクキューを介してメインスレッドを送信します。


## タスク要求メソッド

タスク要求は次のメソッドで実行します。前述のとおり `notify` というフィールドでタスク送信キューと条件変数を共有しているので、そこへ新たなタスクの `key` を渡しています。 `contours_grid_step` というのは等高線の間隔を示すパラメータです。また、ここでは `self.requested` という集合で重複したリクエストを避けています。

```rust
impl TileGen {
    pub fn request_tile(&mut self, key: &HeightMapKey, contours_grid_step: usize) {
        // We could not use entry API due to a lifetime issue.
        if !self.requested.contains(key) {
            let (lock, cvar) = &*self.notify;
            let mut queue = lock.lock().unwrap();
            queue.push_back((*key, contours_grid_step));
            cvar.notify_one();
            self.requested.insert(*key);
        }
    }
}
```

## 完了タスク受信メソッド

最後に `update` メソッドを紹介します。これは完了タスクキューを受け取るメソッドで、メインスレッドで定期的に呼び出すことを想定しています。

```rust
impl TileGen {
    pub fn update(&mut self) -> Result<Vec<(HeightMapKey, HeightMapTile)>, String> {
        let tiles = self.finished.try_iter().collect::<Vec<_>>しゅう

        Ok(tiles)
    }
}
```

## `join` はいらないのか？

マルチスレッドプログラミングを触ったことのある人なら、ワーカースレッドを `join` せずにそのまま放置していることに不安を覚えると思います。これはプログラム終了時にメインスレッド以外を強制的に中断することを意味し、中途半端な状態でクリーンアップせずに終了してしまいます。通常であればもう一つの条件変数か `AtomicBool` で終了フラグを立てて、スレッドが穏健に終了するまで待機することが多いです。

本アプリケーションではワーカースレッドが触るのはメインメモリだけであり、ファイルハンドルやソケットなどのプロセス外のリソースは触りません。プロセスメモリは終了時にすべて破棄されます。このため、プロセスを強制終了してもクリーンアップしないことによる弊害は生じません。これがもしファイルやネットワークへの出力を含む場合は、中途半端に壊れたファイルが生成されたりする可能性があります。


# WebAssembly 対応

本ゲームは [WebAssembly バージョン](https://msakuta.github.io/trains-rs/)でも遊べますが、 WebAssembly でスレッドを扱うのは色々ハードルが高いので、[シングルスレッドのロジック](https://github.com/msakuta/trains-rs/blob/e2666b0397d7ff678a32c6bb222b2d39a7a3e75a/src/app/heightmap/progressive_tile_gen.rs)で置き換えています。このため上記の並列化の恩恵にはあずかれません。

この切り替えは次のような条件コンパイルで行っています。

```rust
#[cfg(not(target_arch = "wasm32"))]
use self::parallel_tile_gen::TileGen;

#[cfg(target_arch = "wasm32")]
use self::progressive_tile_gen::TileGen;
```

`progressive_tile_gen` の内容は詳しく説明しませんが、 `parallel_tile_gen` と同じ型とメソッドを持ち、置き換えられるようにしています。コードはシングルスレッドで同期などを必要としないためかなりシンプルです。ただ、メインスレッドを長い間ブロックしないように、一度の `TileGen::update` で一つのタイルだけを生成するようにしています。これでもメインスレッドをある程度の時間占有するので、体感ではカクつきが強いです。


# まとめ

本稿では動的に要求が発生しうるタスクを非同期で処理するタスクランナーの実装を紹介しました。ゲームなどのリアルタイム性のあるアプリケーションでは時々必要になると思いますが、 rayon 等の既存のクレートで簡単に実現することができなかったので自作する方法を紹介しました[^3]。

[^3]: そのようなクレートは探せばあると思いますが、探して内容を理解するのにかかる時間と、標準ライブラリで実装するのにかかる時間は大して変わらないと思います。

ただし、本稿で扱っているのは CPU バウンドなタスクであり、 I/O バウンドなタスクは tokio などのランタイムを使った方が良いと思います。

WebAssembly でマルチスレッド化 (マルチ worker と SharedMemoryBuffer) は今後の課題です。
