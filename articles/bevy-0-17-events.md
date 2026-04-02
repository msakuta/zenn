---
title: "[Rust] Bevy 0.18 時点での Event 周りの変遷"
emoji: "⚽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "Bevy", "ゲーム開発", "ECS"]
published: true
---

過去に [Bevy についての記事](https://zenn.dev/msakuta/articles/40c1ad41b1c62e)を書いてから早くも4年近く経ちました。 Bevy もその間に進化を遂げ、様々な変化を経ています。しかし、 Bevy は未だに 1.0 に達しておらず、破壊的変更が度々なされています。ここではその中でも混乱を招きやすい、 Event 周りについて解説したいと思います。

ここで紹介するコードサンプルは [Github](https://github.com/msakuta/bevy-events-test) にも置いてあります。

# Event と Message

Bevy 0.16 までの Event には、 Buffered と Obsesrvable という種別がありましたが、 0.17 ではそれぞれ Message および Event にリネームされました。これはより機能に即した名前とするためです。また、 Component の追加や削除といったイベントに対応する Component Hook という機構があり、これは Message とも Event とも異なります。これについては [Migration Guide](https://bevy.org/learn/migration-guides/0-16-to-0-17/#event-trait-split-rename) に書かれていますが、非常に混乱しやすいので、以下の表でまとめます。

| Bevy Version  | Message |  Event | Event Handler type |  Hook   |
| ------------- | -------- | ------ | ------ | ------- |
| ～0.16        | Event (buffered)    | Event (observable)     | Trigger | Hook   |
| 0.17～        | Message  | Event | On | Hook   |

# Bevy 0.16 までの Event = Bevy 0.17 以降の Message

Bevy 0.16 までの Buffered Event は、専用の `EventWriter` と `EventReader` を System の引数にとり、 System 間でのデータのやり取りに使われていました。これは System の引数が増えすぎてきたときにロジックを分割したり、 Query の占有期間を最適化するのに使えましたが、主な用途は Entity 間でのメッセージのやり取りでした。

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

Bevy 0.16 での Buffered Event あるいは Bevy 0.17 で新たに Message と名付けられた機構は、まさにメッセージキューとして機能します。しかし、いくつか使用上の制約があり、後ほど Event と呼ばれる機能との使い分けが重要になります。

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

# Event の挙動

Bevy 0.16 では Observable Event 、あるいは Bevy 0.17 では単に Event と呼ばれるのは、メッセージキューというよりはイベントと呼ばれるもののイメージに近いです。前述の例を Event で書き直すと次のようになります。

```rust
use bevy::prelude::*;

#[derive(Event)]
struct MyEvent(usize);

#[derive(Resource)]
struct Counter(usize);

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .insert_resource(Counter(0))
        .add_systems(Update, (start_update, event_producer))
        .add_observer(event_consumer)
        .run();
}

fn start_update(counter: Res<Counter>, mut close_writer: MessageWriter<AppExit>) {
    if 3 <= counter.0 {
        close_writer.write(AppExit::Success);
    } else {
        println!("start_update ({})!", counter.0);
    }
}

fn event_producer(mut commands: Commands, mut counter: ResMut<Counter>) {
    println!("Triggering MyEvent({})!", counter.0);
    commands.trigger(MyEvent(counter.0));
    counter.0 += 1;
}

fn event_consumer(event: On<MyEvent>) {
    println!("MyEvent({}) received!", event.0);
}
```

違いとしては、 `.add_event()` でイベントの型を登録しておかなくてもよいということと、 `.add_systems{}` の代わりに `.add_observer()` でハンドラを登録することと、そのハンドラの第一引数の型に `On<MyEvent>` を使う点です。

また、イベントの発行には `commands.trigger()` を使います。 `MessageWriter` と異なり、メッセージの型ごとに `Writer` を用意しなくても済みます。これは一つの System で多数のイベントを扱うときに冗長性を減らすのに役立ちます。

上のプログラムの出力は次のようになります。

```
start_update (0)!
Triggering MyEvent(0)!
MyEvent(0) received!
start_update (1)!
Triggering MyEvent(1)!
MyEvent(1) received!
start_update (2)!
Triggering MyEvent(2)!
MyEvent(2) received!
Triggering MyEvent(3)!
MyEvent(3) received!
```

Messageと異なり、発行されたイベントは同じフレームで消費されていることがわかると思います。これは次のフレームまでのキューではなく、トリガされた System にできるだけ近いタイミングで消費される性質によるものです。

## ハンドラの型

もう一つの混乱の元が、 Event のハンドラの型です。

Bevy 0.16 までは `Trigger<T>` という名前だったハンドラ 0.17 で `On<T>` に変更されました。 `Trigger` も deprecated のままで使い続けることもできるので、実際のコードでは混ざる可能性もあります。

```rust
fn event_consumer(event: On<MyEvent>) {
    println!("MyEvent({}) received!", event.0);
}
```

また、 Message と異なり内部でループを回していないことがわかります。これはイベントが生じるたびにこの System が呼び出されるということですが、イベントが生じなければそのフレームでは一度も呼び出されないということでもあります。これは発生頻度の低い(UI ボタンを押すなどの)イベントの処理に向いています。

## Entity ターゲットの Event ハンドラ

Event には特定の Entity に向けて発行されたときのみ応答する変種があります。このやり方には色々ありますが、使い勝手が良いのは　`EntityEvent` トレイトを `derive` する方法です。 `#[derive(EntityEvent)]` でそのような Event を定義できますが、要素が一つのタプル構造体の場合はその要素、構造体の場合は `entity` という名前のフィールドがターゲットになります。

具体的には、次の例を見てください。

```rust
use bevy::prelude::*;

#[derive(EntityEvent)]
struct GreetEvent {
    entity: Entity,
    name: String,
}

#[derive(Component)]
struct Hello;

#[derive(Component)]
struct Bye;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Startup, setup)
        .add_systems(Update, event_producer.run_if(elapsed(1.)))
        .run();
}

fn setup(mut commands: Commands) {
    commands.spawn(Hello).observe(on_hello);
    commands.spawn(Bye).observe(on_bye);
}

fn on_hello(event: On<GreetEvent>) {
    println!("Hello, {}", event.name);
}

fn on_bye(event: On<GreetEvent>) {
    println!("Bye, {}", event.name);
}

fn elapsed(threshold: f32) -> impl Fn(Res<Time>) -> bool {
    move |time: Res<Time>| {
        time.elapsed_secs() < threshold && threshold < time.elapsed_secs() + time.delta_secs()
    }
}

fn event_producer(
    mut commands: Commands,
    hello: Single<Entity, With<Hello>>,
    bye: Single<Entity, With<Bye>>,
) {
    commands.trigger(GreetEvent {
        entity: *hello,
        name: "Carl".to_string(),
    });
    commands.trigger(GreetEvent {
        entity: *bye,
        name: "Carl".to_string(),
    });
}
```

ここで出力は次のようになります。それぞれの Entity (Hello, Bye) に応じたハンドラが呼び出されていることがわかると思います。

```
Hello, Carl
Bye, Carl
```

このように特定の Entity 向けのイベント通知は Message で書くのが難しいので、有力な方法といえます。

# Component Hook の挙動

Component Hook の目的は、 Message や Event とは少し異なります。主に関連する Entity の Component の生成や消滅といったイベントに応じて関連データの整合性を保つのに使います。親子関係では `ChildOf`, `Children` というコンポーネントが存在しますが、この相互参照の整合性を保つことなどに使われています。他にも [Relationship](https://docs.rs/bevy/latest/bevy/ecs/relationship/trait.Relationship.html) トレイトを実装することで似たような関連コンポーネントを実装することもできます。しかし、これらの方法では表現できない関係(多対多など)では Component Hook の自前実装が必要になります。

よくあるのが、インベントリシステムです。インベントリはアイテムの集まりですが、アイテムが削除されたときはインベントリからの参照を削除し、インベントリが削除されたらそれに含まれるすべてのアイテムを削除したいです。この構造は Relationship と RelationshipTarget の実装でも実現できますが、ここでは Component Hook の自前実装でするとしたらどうなるかを示します。

```rust
use std::collections::HashSet;

use bevy::prelude::*;

#[derive(Clone, Debug, Default, Component)]
struct Inventory(HashSet<Entity>);

#[derive(Clone, Component)]
struct Item {
    inventory: Entity,
    name: String,
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Startup, setup)
        .add_systems(
            Update,
            (
                start_update.run_if(elapsed(1.)),
                delete_item.run_if(elapsed(2.)),
                delete_inventory.run_if(elapsed(3.)),
                exit.run_if(elapsed(4.)),
            ),
        )
        .run();
}

fn setup(world: &mut World) {
    world
        .register_component_hooks::<Inventory>()
        .on_replace(|mut world, context| {
            info!("on_replace<Inventory>: {:?}", context.entity);
            let value = world.get::<Inventory>(context.entity).unwrap().0.clone();
            for target in value {
                world.commands().entity(target).despawn();
            }
        });
    world
        .register_component_hooks::<Item>()
        .on_replace(|mut world, context| {
            info!("on_replace<Item>: {:?}", context.entity);
            let inventory = world.get::<Item>(context.entity).unwrap().inventory;
            if let Some(mut target) = world.get_mut::<Inventory>(inventory) {
                target.0.remove(&context.entity);
            }
        });
}

fn elapsed(threshold: f32) -> impl Fn(Res<Time>) -> bool {
    move |time: Res<Time>| {
        time.elapsed_secs() < threshold && threshold < time.elapsed_secs() + time.delta_secs()
    }
}

fn start_update(mut commands: Commands) {
    let inventory = commands.spawn(()).id();
    let item = commands
        .spawn(Item {
            inventory: inventory,
            name: "Item A".to_string(),
        })
        .id();
    commands
        .entity(inventory)
        .insert(Inventory(std::iter::once(item).collect()));
    info!("Spawned an inventory");
}

fn delete_item(mut commands: Commands, items: Query<(Entity, &Item)>) {
    for (entity, item) in items {
        if item.name == "Item A" {
            info!("Deleting item!");
            commands.entity(entity).despawn();
        }
    }
}

fn delete_inventory(mut commands: Commands, inventories: Query<(Entity, &Inventory)>) {
    for (entity, inventory) in inventories {
        info!("Deleting inventory: {inventory:?}!");
        commands.entity(entity).despawn();
    }
}

fn exit(mut writer: MessageWriter<AppExit>) {
    writer.write(AppExit::Success);
}
```

このプログラムを実行すると次のように出力されます。

```
2026-04-01T15:26:29.765147Z  INFO hooks: Spawned an inventory
2026-04-01T15:26:30.765002Z  INFO hooks: Deleting item!
2026-04-01T15:26:30.765254Z  INFO hooks: on_replace<Item>: 13v0
2026-04-01T15:26:31.765198Z  INFO hooks: Deleting inventory: Inventory({})!
2026-04-01T15:26:31.765360Z  INFO hooks: on_replace<Inventory>: 12v0
```

`on_replace<Inventory>` は Inventory が削除されたイベント、 `on_replace<Item>` は Item が削除されたイベントのハンドラが呼び出されていることを示します。 Component Hook に使えるイベントには以下のように種類があります。コンポーネントが削除か置き換えされたときの応答には on_replace が便利です。

* on_add
* on_insert
* on_replace
* on_remove
* on_despawn

Component Hook は Message および Event よりも効率的に実装されているとされています。もし相互参照のような構造があれば、使ってみるのもよいでしょう。これは Rc と Weak の挙動に似ていますが、 Entity の寿命を明示的に管理するところが異なります。


# 使い分け

Message は一フレームに大量に発行されるようなメッセージについては、 Event よりも高いパフォーマンスが期待されるとしています。これに対し、 Event は使い勝手が良いのと、イベントが消費されるタイミングが決定論的であるというのが利点です。用途に応じて使い分けるとよいでしょう。

とはいえ、個人的にはほとんど全てを Event で書いています。 Message のわずかなパフォーマンスの利点が、 Event の利点を上回るケースはそれほど多くないと思います。何よりも、 System が複雑になってくると引数に多数の `Query` を書く羽目になりがちですが、 Event を使えばほぼ関数を分けるような感覚で複雑性を分解できるのが良いです。

相互参照のような構造があれば、 Component Hook を使って整合性を保つのもよいでしょう。


# おわりに

Bevy は未だに安定化されず、ちょっと触らない間に互換性が壊れていたりしますが、徐々に経験値がたまって堅牢になってきていると思います。日本語の情報も未だに見つけづらいです(とはいえ、 LLM がだいぶ助けになってはいますが)。未だに触るなら人柱になる覚悟は必要ですが、昔のように焚火で火あぶりになるのではなく、ロウソクでチョロチョロ炙られるぐらいの人柱にならなってもよい、という人は触ってみてはいかがでしょうか。
