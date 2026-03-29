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

Bevy 0.17 では実態に合わせて Message という名前に変更され、対応する reader\writer も MessageReader および MessageWriter という名前に変えられました。また Reader のメソッド名は `read()`, Writer のメソッド名は `write()` へと統一されました。


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