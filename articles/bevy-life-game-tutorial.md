---
title: "Bevy Engineでライフゲームを作ろう"
emoji: "🐣"
type: "tech"
topics:
  - "rust"
  - "bevy"
published: true
published_at: "2026-01-25 20:30"
---

## イントロダクション

この記事では、Rustの [Bevy Engine](https://bevy.org/) を使って、ライフゲームを実装していきます。

この記事のコードサンプルは以下のリポジトリで公開しています。
[https://github.com/taktiks2/bevy-life](https://github.com/taktiks2/bevy-life)

---

## Bevy Engineの紹介

### Bevy Engineとは

Bevy Engineは、Rustで書かれたデータ駆動型のゲームエンジンです。活発なコミュニティに
よって開発が進められており、永久に無料で使えるOSSとして提供されています。
また、GUIツールを持たないコードファーストのゲームエンジンであり、Rustのコンパイラとの親和性が高いことから、AIを使ったバイブコーディングとの相性が良いという特徴があります。
この記事で扱うBevyのバージョンは`0.18`です。Bevyはまだ開発段階のエンジンであり、頻繁にアップデートが行われています。APIの破壊的変更も多いため、最新の情報は公式ドキュメントを参照してください。

Bevy Engineの最大の特徴は、`ECS（Entity Component System）`アーキテクチャを採用している点です。

### ECS（Entity Component System）の基本概念

ECSは、ゲーム開発における設計パターンの一つで、以下の3つの要素から構成されます。

#### Entity（実体）

エンティティは、ゲーム世界に存在する「もの」です。ユニークなIDを持ち、それ自体にはデータやロジックを持ちません。ライフゲームでは、各セル（マス目）が1つのエンティティになります。

#### Component（コンポーネント）

コンポーネントは、エンティティに付与されるデータです。構造体として定義され、エンティティの性質を表現します。ライフゲームでは、セルの位置を表す`Position`や、生死を表す`Alive`などがコンポーネントになります。

#### System（システム）

システムは、ロジックを実行する関数です。特定のコンポーネントを持つエンティティに対して処理を行います。ライフゲームでは、「スペースキーが押すと1世代進む」というロジックをシステムとして実装します。

### 従来のゲームエンジンとの違い

従来のオブジェクト指向型のゲームエンジンでは、ゲームオブジェクトが継承階層を持ち、データとロジックが密結合していました。

一方、ECSでは以下のような利点があります。

- データとロジックが分離されており保守性が高い
- 再利用性が高い（コンポーネントを組み合わせて新しいエンティティを作成）
- 大量のオブジェクトを扱う際のパフォーマンスが良い

---

## 環境構築

### プロジェクトの作成

まず、Cargoで新しいプロジェクトを作成します。

```bash
cargo new bevy-life
cd bevy-life
```

### Bevy 0.18の追加

以下を実行してBevyを依存関係に追加します。

```bash
cargo add bevy@0.18
```

### BevyでHello Worldを実行

以下を`src/main.rs`に記述して、コンソールに出力してみましょう。

```rust
use bevy::prelude::*;

fn main() {
  App::new()
    .add_systems(Startup, hello_world)
    .run();
}

fn hello_world() {
  println!("Hello, World!");
}
```

`cargo run`を実行して、"Hello, World!"が表示されれば成功です。

---

## ライフゲームのルール説明

### ライフゲームの基本ルール

ライフゲームは、1970年にイギリスの数学者ジョン・ホートン・コンウェイが考案したシミュレーションゲームです。以下の3つのシンプルなルールで動作します：

- 生存：生きているセルの周囲に2〜3個の生きたセルがあれば、次世代も生存
- 誕生：死んでいるセルの周囲に3個の生きたセルがあれば、次世代で誕生
- 死亡：その他の場合は死亡（過疎または過密）

### この記事で作るもの

今回作成するライフゲームの仕様は以下の通りです。

- 40x40のグリッド状にセルを配置
- 初期パターンとしてグライダーパターンを配置
- マウスクリックでセルの生死を切り替え
- スペースキーを押すと1世代進む

グライダーパターンは、5つのセルで構成される有名なパターンで、斜め方向に移動し続ける性質があります。

```txt
■ ■ ■
□ ■ □
■ □ □
```

---

## ステップバイステップ実装

それでは、実際にコードを書いていきましょう。`src/main.rs`を開いて、順番に実装していきます。

### 基本的なウィンドウの表示

まず、必要なモジュールをインポートし、定数を定義します。

```rust
use bevy::prelude::*;
use bevy::window::WindowResolution;

const GRID_WIDTH: i32 = 40;
const GRID_HEIGHT: i32 = 40;
const CELL_SIZE: f32 = 20.0;
const WINDOW_WIDTH: u32 = (GRID_WIDTH as f32 * CELL_SIZE) as u32;
const WINDOW_HEIGHT: u32 = (GRID_HEIGHT as f32 * CELL_SIZE) as u32;
```

次に、Bevyアプリケーションのエントリーポイントとなる`main`関数を実装します。

```rust
fn main() {
    App::new()
        .add_plugins(DefaultPlugins.set(WindowPlugin {
            primary_window: Some(Window {
                title: "ライフゲーム".to_string(),
                resolution: WindowResolution::new(WINDOW_WIDTH, WINDOW_HEIGHT),
                resizable: false,
                ..default()
            }),
            ..default()
        }))
        .run();
}
```

ここでは以下のことを行っています。

- `App::new()`：Bevyアプリケーションを作成
- `add_plugins(DefaultPlugins)`：Bevyの基本機能（ウィンドウ、入力、描画など）を追加
- `WindowPlugin`のカスタマイズ：
  - タイトル：「ライフゲーム」
  - ウィンドウサイズ：40 * 20 = 800ピクセルの正方形
  - リサイズ不可

この状態で`cargo run`を実行し、指定したウィンドウが表示されることを確認してください。

### セルの描画

次に、グリッド状にセルを描画していきます。ここでは、カメラの設定とセルの生成を行う2つのシステムを追加します。

#### Startupスケジュールについて

Bevyでは、システムを実行するタイミングを「スケジュール」で管理します。`Startup`スケジュールは、アプリケーションの起動時に一度だけ実行されるスケジュールです。初期化処理やゲーム開始時のセットアップに使用します。

その他のスケジュールには、毎フレーム実行される`Update`スケジュールなどがあります。今回は、ゲーム開始時にカメラとボードを生成したいので、`Startup`スケジュールを使用します。

#### システムパラメータについて

Bevyのシステムは、引数を通じて様々な機能にアクセスできます。主なパラメータは以下の通りです。

- Commands：エンティティやコンポーネントの追加・削除を行う
- Query：特定のコンポーネントを持つエンティティを検索・操作する
- Res：リソース（ゲーム内のグローバルなデータ）にアクセスする

これらのパラメータは、Bevyが自動的に依存性注入してくれるため、必要なものを引数に指定するだけで使用できます。例えば、`mut commands: Commands`と書けば、エンティティを生成するための`Commands`が自動的に渡されます。

#### カメラの設定

2Dゲームを表示するには、カメラが必要です。以下のシステムでカメラを生成します。

```rust
fn spawn_camera(mut commands: Commands) {
    commands.spawn(Camera2d);
}
```

`spawn`メソッドでカメラエンティティを生成しています。

#### セルの生成

次に、グリッド状にセルを配置するシステムを実装します。

```rust
fn spawn_board(mut commands: Commands) {
    // 2重ループですべてのセルを生成
    for y in 0..GRID_HEIGHT {
        for x in 0..GRID_WIDTH {
            // 色は暗いグレーに設定
            let color = Color::srgb(0.1, 0.1, 0.1);

            commands.spawn((
                // Spriteコンポーネントでセルを表現
                Sprite {
                    color,
                    // - 1.0でグリッド線を表現
                    custom_size: Some(Vec2::new(CELL_SIZE - 1.0, CELL_SIZE - 1.0)),
                    ..default()
                },
                Transform::from_xyz(
                    ((x as f32 + 0.5) - GRID_WIDTH as f32 / 2.0) * CELL_SIZE,
                    ((y as f32 + 0.5) - GRID_HEIGHT as f32 / 2.0) * CELL_SIZE,
                    0.0,
                ),
            ));
        }
    }
}
```

#### Bevyの2D座標系について

Bevyの2Dカメラを使用する際、[座標系について](https://bevy-cheatbook.github.io/fundamentals/coords.html#2d-and-3d-scenes-and-cameras)理解しておくことが重要です。

- 原点の位置：画面中央が座標(0, 0)
- X軸：右方向がプラス（正の値）、左方向がマイナス（負の値）
- Y軸：上方向がプラス（正の値）、下方向がマイナス（負の値）

```txt
        Y軸
        +
        ↑
        |
        |
- ------+------→ + X軸
        |
        |
        |
        -
```

この座標系は、[Godotエンジン](https://godotengine.org/ja/)と同じ右手座標系です。一般的なUI座標系（左上が原点でY軸が下向き）とは異なるため、注意が必要です。

そのため、セルを配置する際には、グリッド座標から画面座標への変換が必要になります。後述の`spawn_board`関数では、`- GRID_WIDTH as f32 / 2.0`のような計算で、グリッドの左下を負の座標、右上を正の座標に配置しています。

システムを登録するため、main関数を以下のように更新します。

```rust
fn main() {
    App::new()
        .add_plugins(DefaultPlugins.set(WindowPlugin {...}))
        .add_systems(Startup, (spawn_camera, spawn_board)) // システムの登録
        .run();
}
```

- add_systems(Startup, ...)：起動時に1度だけ実行されるStartupスケジュールに2つのシステムを追加

この状態で`cargo run`を実行すると、暗いグレーのグリッドが表示されます。

### 初期パターンの配置

次に、グライダーパターンをグリッドの中央に配置するため、セルの位置と生死状態を管理するコンポーネントを定義します。

#### コンポーネントの定義

まず、セルの位置を表す`Position`コンポーネントを定義します。

```rust
// セルの位置を表すコンポーネント
#[derive(Component, Clone, Copy, PartialEq, Eq, Hash)]
struct Position {
    x: i32,
    y: i32,
}
```

`Position`は、グリッド上のx, y座標を保持します。`#[derive(...)]`で以下のトレイトを自動実装しています。

- `Component`：Bevyのコンポーネントとして使用できるようにする
- `Clone, Copy`：値のコピーを可能にする
- `PartialEq, Eq, Hash`：位置の比較やハッシュマップでの使用を可能にする

次に、セルの生死状態を表す`Alive`コンポーネントを定義します。

```rust
// セルが生きているかどうかを表すコンポーネント
#[derive(Component)]
struct Alive(bool);
```

`Alive`は、セルが生きているか（`true`）死んでいるか（`false`）を表すシンプルなコンポーネントです。

#### グライダーパターンの定義

グライダーパターンを配列で定義します。

```rust
// グライダーパターン（相対座標）
// 2 ■ ■ ■
// 1 □ □ ■
// 0 □ ■ □
//   0 1 2
const GLIDER_PATTERN: [(i32, i32); 5] = [(1, 0), (2, 1), (0, 2), (1, 2), (2, 2)];
```

このパターンは、5つのセルの相対座標を表しています。

#### spawn_board関数の更新

`spawn_board`関数を更新して、グライダーパターンを中央に配置し、各セルに`Position`と`Alive`コンポーネントを追加します。

```rust
fn spawn_board(mut commands: Commands) {
    // 初期パターンを中央に配置するためのオフセット
    let offset_x = GRID_WIDTH / 2;
    let offset_y = GRID_HEIGHT / 2;

    // すべてのセルを生成
    for y in 0..GRID_HEIGHT {
        for x in 0..GRID_WIDTH {
            let position = Position { x, y };
            // このセルがグライダーパターンに含まれるかチェック
            let is_alive = GLIDER_PATTERN
                .iter()
                .any(|(px, py)| x == offset_x + px && y == offset_y + py);

            let color = if is_alive {
                Color::WHITE
            } else {
                Color::srgb(0.1, 0.1, 0.1)
            };

            commands.spawn((
                Sprite {
                    color,
                    custom_size: Some(Vec2::new(CELL_SIZE - 1.0, CELL_SIZE - 1.0)),
                    ..default()
                },
                Transform::from_xyz(
                    ((x as f32 + 0.5) - GRID_WIDTH as f32 / 2.0) * CELL_SIZE,
                    ((y as f32 + 0.5) - GRID_HEIGHT as f32 / 2.0) * CELL_SIZE,
                    0.0,
                ),
                position, // Positionコンポーネントを追加
                Alive(is_alive), // Aliveコンポーネントを追加
            ));
        }
    }
}
```

この状態で`cargo run`を実行すると、グリッドの中央にグライダーパターンが白いセルで表示されます。

### セルの生死を切り替える機能

次に、マウスでセルをクリックして生死を切り替える機能を実装します。Bevyでは、エンティティに対するイベントを簡単に処理できるオブザーバー機能が導入されています。

#### Bevyのオブザーバーシステムとは

Bevyのオブザーバーは、特定のエンティティに対するイベントを監視し、反応する仕組みです。以下の特徴があります。

- エンティティ単位でイベントを処理できる
- イベントが発火したエンティティに直接アクセスできる
- エンティティのスポーン時に登録できる

今回は、セルがクリックされたときに反応するオブザーバーを実装します。

#### クリック処理の実装

次に、クリックされたときの処理を行う`handle_cell_click`関数を実装します。

```rust
// セルのクリックを処理するオブザーバー
fn handle_cell_click(trigger: On<Pointer<Click>>, mut cells: Query<(&mut Sprite, &mut Alive)>) {
    let entity = trigger.original_event_target();
    let Ok((mut sprite, mut alive)) = cells.get_mut(entity) else {
        return;
    };

    // セルの状態を切り替え
    alive.0 = !alive.0;
    sprite.color = if alive.0 {
        Color::WHITE
    } else {
        Color::srgb(0.1, 0.1, 0.1)
    };
}
```

- `trigger: On<Pointer<Click>>`：クリックイベントのトリガー情報を受け取る
- `mut cells: Query<(&mut Sprite, &mut Alive)>`：Queryによるエンティティ検索
  - Queryは、特定のコンポーネントを持つエンティティを検索・操作するためのシステムパラメータです
  - `Query<(&mut Sprite, &mut Alive)>`は、「`Sprite`と`Alive`の両方のコンポーネントを持つエンティティ」を検索します
  - `&mut`を付けることで、コンポーネントを変更可能な形で取得します
- `trigger.original_event_target()`：クリックされたエンティティのIDを取得
- `cells.get_mut(entity)`：特定のエンティティIDからコンポーネントを取得
- `alive.0 = !alive.0`：セルの生死状態を反転
- `sprite.color`：状態に応じて色を更新（生：白、死：暗いグレー）

#### Pickableコンポーネントの追加 & オブザーバーの登録

クリック可能にするために、セルに`Pickable`コンポーネントを追加します。`spawn_board`関数のセル生成部分を以下のように更新します。

```rust
commands
    .spawn((
        Sprite {
            color,
            custom_size: Some(Vec2::new(CELL_SIZE - 1.0, CELL_SIZE - 1.0)),
            ..default()
        },
        Transform::from_xyz(
            ((x as f32 + 0.5) - GRID_WIDTH as f32 / 2.0) * CELL_SIZE,
            ((y as f32 + 0.5) - GRID_HEIGHT as f32 / 2.0) * CELL_SIZE,
            0.0,
        ),
        position,
        Alive(is_alive),
        Pickable::default(),  // クリック可能にする
    ))
    .observe(handle_cell_click);  // クリックイベントのオブザーバーを登録
```

`Pickable::default()`を追加することで、このエンティティがクリック可能になります。また、`.observe(handle_cell_click)`で、クリックイベントを処理するオブザーバーを登録します。

この状態で`cargo run`を実行すると、セルをクリックすることで生死を切り替えられるようになります。

### セルの世代を進める機能

最後に、スペースキーを押すとライフゲームの世代を進める機能を実装します。この機能では、Bevyのメッセージングシステムを使用します。

#### Bevyのメッセージングシステムとは

Bevy 0.17で、システム間で通信するための新しい仕組みとしてメッセージングが導入されました。以下の特徴があります。

- イベントよりも軽量で高速
- システム間の疎結合を実現
- 1フレーム内でのみ有効（次フレームには持ち越されない）

今回は、キー入力を検知するシステムから、世代を進めるシステムへメッセージを送信します。

#### 必要なインポートの追加

まず、`std::collections::HashMap`を使用するため、ファイルの先頭に以下を追加します。

```rust
use std::collections::HashMap;
```

#### NextGenerationEventの定義

次世代に進むイベントを表すメッセージを定義します。

```rust
// 次世代に進むイベント
#[derive(Message)]
struct NextGenerationEvent;
```

`#[derive(Message)]`を使用することで、この構造体をメッセージとして使用できるようになります。

#### キー入力処理の実装

スペースキーが押されたときにメッセージを送信するシステムを実装します。

```rust
// キー入力処理システム
fn handle_input_key(
    keyboard: Res<ButtonInput<KeyCode>>,
    mut next_gen_event: MessageWriter<NextGenerationEvent>,
) {
    if keyboard.just_pressed(KeyCode::Space) {
        next_gen_event.write(NextGenerationEvent);
    }
}
```

- `keyboard: Res<ButtonInput<KeyCode>>`：キーボード入力状態を取得
- `mut next_gen_event: MessageWriter<NextGenerationEvent>`：メッセージ送信用のライター
- `just_pressed(KeyCode::Space)`：スペースキーが押された瞬間を検知
- `next_gen_event.write(...)`：メッセージを送信

#### 隣接セルのカウント関数

ライフゲームのルールを適用するため、各セルの周囲8マスの生きたセルをカウントする関数を実装します。

```rust
// 隣接する生きたセルの数をカウントする関数
fn count_neighbors(position: &Position, alive_cells: &HashMap<Position, Entity>) -> usize {
    let mut count = 0;

    // 8方向の隣接セルをチェック
    for dy in -1..=1 {
        for dx in -1..=1 {
            if dx == 0 && dy == 0 {
                continue; // 自分自身はスキップ
            }

            let nx = position.x + dx;
            let ny = position.y + dy;

            // 境界チェック：盤面外のセルは無視
            if !(0..GRID_WIDTH).contains(&nx) || !(0..GRID_HEIGHT).contains(&ny) {
                continue;
            }

            let neighbor_pos = Position { x: nx, y: ny };

            if alive_cells.contains_key(&neighbor_pos) {
                count += 1;
            }
        }
    }

    count
}
```

#### 次世代の計算システム

最後に、ライフゲームのルールに基づいて次世代を計算するシステムを実装します。

```rust
// 次世代の状態を計算するシステム
fn next_generation(
    mut next_gen_events: MessageReader<NextGenerationEvent>,
    mut cells: Query<(Entity, &Position, &mut Alive, &mut Sprite)>,
) {
    // NextGenerationEventが発火していない場合は何もしない
    if next_gen_events.read().next().is_none() {
        return;
    }

    // 現在生きているセルのマップを作成
    let mut alive_cells = HashMap::new();
    for (entity, position, alive, _) in cells.iter() {
        if alive.0 {
            alive_cells.insert(*position, entity);
        }
    }

    // 各セルの次世代の状態を計算
    let mut next_states = Vec::new();

    for (entity, position, alive, _) in cells.iter() {
        let neighbor_count = count_neighbors(position, &alive_cells);

        // ライフゲームのルールに基づいて次の状態を決定
        let should_be_alive = if alive.0 {
            neighbor_count == 2 || neighbor_count == 3 // 生存条件
        } else {
            neighbor_count == 3 // 誕生条件
        };

        // 状態が変化するセルだけを記録
        if should_be_alive != alive.0 {
            next_states.push((entity, should_be_alive));
        }
    }

    // 状態を更新
    for (entity, should_be_alive) in next_states {
        if let Ok((_, _, mut alive, mut sprite)) = cells.get_mut(entity) {
            alive.0 = should_be_alive;
            sprite.color = if should_be_alive {
                Color::WHITE
            } else {
                Color::srgb(0.1, 0.1, 0.1)
            };
        }
    }
}
```

- `mut next_gen_events: MessageReader<NextGenerationEvent>`：メッセージ受信用のリーダー
- `mut cells: Query<(Entity, &Position, &mut Alive, &mut Sprite)>`：セルのエンティティとコンポーネントを取得
- `next_gen_events.read().next().is_none()`：メッセージが発火しているかのチェック

メッセージとシステムを登録するため、`main`関数を以下のように更新します。

```rust
fn main() {
    App::new()
        .add_plugins(DefaultPlugins.set(WindowPlugin {
            primary_window: Some(Window {
                title: "ライフゲーム".to_string(),
                resolution: WindowResolution::new(WINDOW_WIDTH, WINDOW_HEIGHT),
                resizable: false,
                ..default()
            }),
            ..default()
        }))
        .add_message::<NextGenerationEvent>()  // メッセージの登録
        .add_systems(Startup, (spawn_camera, spawn_board))
        .add_systems(Update, (handle_input_key, next_generation))  // 新しいシステムを追加
        .run();
}
```

- `add_message::<NextGenerationEvent>()`：メッセージシステムに登録
- `add_systems(Update, ...)`：毎フレーム実行される`Update`スケジュールに2つのシステムを追加

この状態で`cargo run`を実行し、スペースキーを押すと、グライダーパターンが1世代ずつ進み、斜め方向に移動していく様子が確認できます。

---

## まとめ

お疲れさまでした！この記事では、Bevy Engineを使って、シンプルながら奥深いライフゲームを実装しました。

こちらをベースに、機能拡張に挑戦してみてください！

- 自動再生機能：タイマーを使って一定間隔で自動的に世代を進める
- 初期パターン集：ビーハイブ、ボート、パルサーなど様々なパターンを追加
- UI要素の追加：世代数カウンター、再生/停止ボタンなど

---

## 参考資料

- [Bevy公式ドキュメント](https://docs.rs/bevy/latest/bevy/)
- [Bevy Examples](https://github.com/bevyengine/bevy/tree/main/examples)
- [Bevy Cheat Book](https://bevy-cheatbook.github.io/)
- [Tainted Coder](https://taintedcoders.com/)
- [ライフゲーム - Wikipedia](https://ja.wikipedia.org/wiki/%E3%83%A9%E3%82%A4%E3%83%95%E3%82%B2%E3%83%BC%E3%83%A0)

---

:::message
本記事は生成AI（Claude Code）を使用して作成し、筆者が内容を確認・編集しています
:::
