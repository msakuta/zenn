---
title: "[Rust] 並列 async タスクでローカル変数を共有する"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust"]
published: false
publication_name: "mapbox_japan"
---

Mapbox の EV team では、バックエンドの開発に Rust を使用しています。ここでは、 Rust でのバックエンド開発で遭遇しがちな問題について解説します。

## 非同期タスクの並列化

Tokioを使って非同期ランタイムを使っていると、複数の async タスクを並列に走らせたいことがあると思います。これを例えば次のようにナイーブに書いてしまうと、効率が悪くなることがあります。

```rust
#[tokio::main]
async fn main() {
    for i in 0..3 {
        println!("Starting async function...");
        waited_hello(i).await;
    }
    println!("Finished!");
}

async fn waited_hello(i: i32) {
    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
    println!("Hello from async function {i}!");
}
```

上の例では、 `tokio::time::sleep` を使って時間がかかるタスクの代わりにしていますが、それぞれのタスク `waited_hello` は 1 秒かかるところ、全体が走るのに 3 秒かかっています。

```
$ time cargo r
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.01s
     Running `target/debug/async-test`
Starting async function...
Hello from async function 0!
Starting async function...
Hello from async function 1!
Starting async function...
Hello from async function 2!
Finished!

real    0m3.088s
user    0m0.066s
sys     0m0.025s
```

これを並列に走らせるには、 `tokio::spawn` を使い、複数の `JoinHandle` をまとめて処理します(タスクの数が決め打ちで書き下せる場合は `tokio::join!` マクロを使うこともできます)。

```rust
#[tokio::main]
async fn main() {
    let handles: Vec<_> = (0..3)
        .map(|i| {
            tokio::spawn(async move {
                println!("Starting async function...");
                waited_hello(i).await
            })
        })
        .collect();

    for handle in handles {
        handle.await.unwrap();
    }
    println!("Finished!");
}
```

これで実行時間がほぼ 1 秒になりました。

```
$ time cargo r
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.01s
     Running `target/debug/async-test`
Starting async function...
Starting async function...
Starting async function...
Hello from async function 2!
Hello from async function 0!
Hello from async function 1!
Finished!

real    0m1.071s
user    0m0.049s
sys     0m0.026s
```

## タスク間での共有変数

ここまでは、よく知られた話であり、問題となることは少ないと思います。しかし、タスク間で共有したい変数(しかしこの関数のスコープ内のみ)があった場合はどうなるでしょうか。例えば、共有変数を `total_count` と言う名前の `AtomicUsize` にして、次のように書けるでしょうか。

```rust
use std::sync::atomic::AtomicUsize;

#[tokio::main]
async fn main() {
    let total_count = AtomicUsize::new(0);

    let handles: Vec<_> = (0..3)
        .map(|i| {
            let total_count = &total_count;
            tokio::spawn(async move {
                println!("Starting async function...");
                waited_hello(i).await;
                total_count.fetch_add(1, std::sync::atomic::Ordering::Relaxed);
            })
        })
        .collect();

    for handle in handles {
        handle.await.unwrap();
    }
    println!("Finished {} tasks!", total_count.load(std::sync::atomic::Ordering::Relaxed));
}

async fn waited_hello(i: i32) {
    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
    println!("Hello from async function {i}!");
}
```

こうすると、コンパイルエラーになります。エラーの内容は、 `total_count` の寿命が十分でないというものです。タスクは明らかにこの関数の内部で完了するはずなのに、なぜ `total_count` が `'static` でないといけないのでしょうか。

```
error[E0597]: `total_count` does not live long enough
  --> src/main.rs:9:32
   |
 5 |       let total_count = AtomicUsize::new(0);
   |           ----------- binding `total_count` declared here
...
 8 |           .map(|i| {
   |                --- value captured here
 9 |               let total_count = &total_count;
   |                                  ^^^^^^^^^^^ borrowed value does not live long enough
10 | /             tokio::spawn(async move {
11 | |                 println!("Starting async function...");
12 | |                 waited_hello(i).await;
13 | |                 total_count.fetch_add(1, std::sync::atomic::Ordering::Relaxed);
14 | |             })
   | |______________- argument requires that `total_count` is borrowed for `'static`
...
22 |   }
   |   - `total_count` dropped here while still borrowed
```

これは非常に興味深いエラーです。これは最初のナイーブな実装では起こらない問題だからです。例えば、次のように書けばコンパイルは通ります。

```rust
#[tokio::main]
async fn main() {
    let mut total_count = 0;
    for i in 0..3 {
        println!("Starting async function...");
        waited_hello(i, &mut total_count).await;
    }
    println!("Finished! Total count: {total_count}");
}

async fn waited_hello(i: i32, total_count: &mut i32) {
    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
    println!("Hello from async function {i}!");
    *total_count += 1;
}
```

なぜこのような問題が生じるかというと、 `tokio::spawn` で生成されたタスクは、ライフタイムが型に結びついておらず、関数内で終了することが静的に保証されないからです。これはタスクのタイミングとライフタイムを制御できるようにする柔軟性の代償です。

## 解決策(並列性と変数共有どっちも欲しい)

それでは、このように寿命がプログラマには見えている場合でも、 `Arc` に包むなどしてやらないと共有できないのでしょうか。もちろん、解決策はあります。ここでは `async-scoped` というクレートを使います。

```
cargo add async-scoped --features use-tokio
```

このクレートは [`std::thread::scope`](https://doc.rust-lang.org/std/thread/fn.scope.html) に近いもので、並列タスクがスコープ内で完了することを静的に保証するものです。例えば、次のように書き換えられます。

```rust
use std::sync::atomic::AtomicUsize;

#[tokio::main]
async fn main() {
    let total_count = AtomicUsize::new(0);

    let (_, _results) = async_scoped::TokioScope::scope_and_block(|scope| {
        for i in 0..3 {
            let total_count = &total_count;
            scope.spawn(async move {
                println!("Starting async function...");
                waited_hello(i).await;
                total_count.fetch_add(1, std::sync::atomic::Ordering::Relaxed);
            })
        }
    });

    println!(
        "Finished {} tasks!",
        total_count.load(std::sync::atomic::Ordering::Relaxed)
    );
}

async fn waited_hello(i: i32) {
    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
    println!("Hello from async function {i}!");
}
```

これによって、タスクの並列性と変数の共有を次のように同時に実現することができます。

```
$ time cargo r
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.02s
     Running `target/debug/async-test`
Starting async function...
Starting async function...
Starting async function...
Hello from async function 2!
Hello from async function 1!
Hello from async function 0!
Finished 3 tasks!

real    0m1.083s
user    0m0.060s
sys     0m0.029s
```
