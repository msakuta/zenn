---
title: "Bevy 0.18 時点での Event 周りの変遷"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "Bevy"]
published: false
---

過去に [Bevy についての記事](https://zenn.dev/msakuta/articles/40c1ad41b1c62e)を書いてから早くも4年近く経ちました。 Bevy もその間に進化を遂げ、様々な変化を経ています。しかし、 Bevy は未だに 1.0 に達しておらず、破壊的変更が度々なされています。ここではその中でも混乱を招きやすい、 Event 周りについて解説したいと思います。

# Event と Message

Bevy 0.16 までの Event は、 0.17 でいう Message に相当します。また、新たに Event と呼ばれる（より名前に即した）機構が追加されています。また、 Component の追加や削除といったイベントに対応する Component Hook という機構があり、これは Message とも Event とも異なります。これについては [Migration Guide](https://bevy.org/learn/migration-guides/0-16-to-0-17/#event-trait-split-rename) に書かれていますが、非常に混乱しやすいので、以下の表でまとめます。

| Bevy Version  | Message |  Event | Hook   |
| ------------- | -------- | ------ | ------ |
| ～0.16        | Event    | -     | Hook   |
| 0.17～        | Message  | Event | Hook   |

# Bevy 0.16 までの Event = Bevy 0.17 以降の Message

Bevy 0.16 までの Event は、専用の `EventWriter` と `EventReader` を System の引数にとり、 System 間でのデータのやり取りに使われていました。これは System の引数が増えすぎてきたときにロジックを分割したり、 Query の占有期間を最適化するのに使えましたが、主な用途は Entity 間でのメッセージのやり取りでした。

```rust
use bevy::prelude::*;

#[derive(Event)]
struct MyEvent;

fn main() {
    App::new()
        .add_event::<MyEvent>()
        .add_systems(Update, (event_producer, event_consumer))
    	.run();
}

fn event_producer(mut writer: EventWriter<MyEvent>) {
    writer.write(MyEvent);
}

fn event_consumer(mut reader: EventReader<MyEvent>) {
    for _ in reader.read() {
        println!("MyEvent received!");
    }
}
```

Bevy 0.17 では実態に合わせて Message という名前に変更され、対応する reader/writer も MessageReader および MessageWriter という名前に変えられました。


```rust
use bevy::prelude::*;

#[derive(Message)]
struct MyMessage;

fn main() {
    App::new()
        .add_message::<MyMessage>()
        .add_systems(Update, (message_producer, message_consumer))
    	.run();
}

fn message_producer(mut writer: MessageWriter<MyMessage>) {
    writer.write(MyMessage);
}

fn message_consumer(mut reader: MessageReader<MyMessage>) {
    for _ in reader.read() {
        println!("MyMessage received!");
    }
}
```

# Message の挙動

Bevy 0.17 で新たに Message と名付けられた機構は、まさにメッセージキューとして機能します。しかし、いくつか使用上の制約があり、後ほど Event と呼ばれる機能が別に用意されることになりました。

最も大きな注意点は、メッセージの配信タイミングです。 MessageReader は同じフレームで MessageWriter に書き込まれたメッセージを受信できる保証はありません。例えば次の例を見てください。

```rust
use bevy::prelude::*;

#[derive(Message)]
struct MyMessage(usize);

#[derive(Resource)]
struct Counter(usize);

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_message::<MyMessage>()
        .insert_resource(Counter(0))
        .add_systems(Update, (start_update, message_producer, message_consumer))
        .run();
}

fn start_update(counter: Res<Counter>, mut close_writer: MessageWriter<AppExit>) {
    if 3 <= counter.0 {
        close_writer.write(AppExit::Success);
    } else {
        println!("start_update ({})!", counter.0);
    }
}

fn message_producer(mut counter: ResMut<Counter>, mut writer: MessageWriter<MyMessage>) {
    println!("Sending MyMessage({})!", counter.0);
    writer.write(MyMessage(counter.0));
    counter.0 += 1;
}

fn message_consumer(mut reader: MessageReader<MyMessage>) {
    for event in reader.read() {
        println!("MyMessage({}) received!", event.0);
    }
}
```

ここでは、 `event_producer` で3回 Message を書き込み、 `event_consumer` で読み込むことを意図しています。 `start_update -> event_producer -> event_consumer` という順に実行され、 `event_producer` からの出力が `event_consumer` で読みだされることを期待したいところです。カウンターを Resource として記憶し、メッセージを書き込むたびにインクリメントして出力してこれを確かめたところ、少なくとも私の環境での出力は次のようになりました。

```
Sending MyMessage(0)!
start_update (1)!
MyMessage(0) received!
Sending MyMessage(1)!
MyMessage(1) received!
start_update (2)!
Sending MyMessage(2)!
MyMessage(2) received!
Sending MyMessage(3)!
MyMessage(3) received!
Sending MyMessage(4)!
MyMessage(4) received!
```

２行目と３行目に注目です。 `start_update (1)` が `MyMessage(0) received!` よりも前に来ています。これはメッセージが同じフレームで消費されず、次のフレームまで遅れたことを意味します。

実際、[ドキュメント](https://docs.rs/bevy_ecs/0.17.3/bevy_ecs/message/struct.Messages.html)を見てみると並列実行される System では Writer と Reader のどちらが先に実行されるかは予測不能とされています。

> If no ordering is applied between writing and reading systems, there is a risk of a race condition. This means that whether the messages arrive before or after the next Messages::update is unpredictable.

これは即応性が求められる用途においては問題になりえます。特にメッセージを複数連鎖させると、段数の数だけフレームが遅れる可能性があります。

もう一つの注意点は、使用する全ての Message 型を `add_message` で登録しておかなくてはならないということです。この点も後述の Event には不要であるという利点があります。

```rust
    App::new()
        .add_message::<MyMessage>()
```

ただし、これを行わないと起動時のエラーになるので気づかないままになることはないでしょう。
