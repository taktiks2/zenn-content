---
title: "`just` を導入しよう — Makefile を超えたタスクランナー"
emoji: "⚡"
type: "tech"
topics:
  - "just"
  - "taskrunner"
  - "makefile"
  - "rust"
  - "devops"
published: true
---

# `just` を導入しよう — Makefile を超えたタスクランナー

## はじめに

開発をしていると、こんな場面によく遭遇しませんか？

- 「あれ、テストはどうやって実行するんだっけ？」
- README に書いてあるコマンドが長くて毎回コピペしている
- チームメンバーによって微妙にコマンドが違って、環境差異が生じる

そんな悩みを一気に解決してくれるのが **[`just`](https://github.com/casey/just)** です。

`just` は [make](https://www.gnu.org/software/make/manual/make.html#Rules)に似た「タスクランナー」ですが、ビルドツールとしての複雑な依存関係管理を捨て、**「コマンドをまとめて実行する」** ことに特化したシンプルなツールです。

---

## `just` とは？

`just` は Rust 製のコマンドランナーです。`Justfile` というファイルにタスク（レシピ）を定義し、`just <タスク名>` で実行できます。

Make とは異なり、**ファイルのタイムスタンプによる依存解決は行いません**。あくまで「コマンドのショートカット集」です。シンプルな設計なので、誰でもすぐに使いこなせます。

---

## インストール

```sh
# macOS
brew install just

# Ubuntu / Debian
sudo apt install just

# Cargo (Rust)
cargo install just

# Windows (Scoop)
scoop install just
```

---

## 基本的な使い方

プロジェクトルートに `Justfile` を作成します。

```justfile
# コンテナを起動する
up:
    docker compose up -d

# コンテナを停止する
down:
    docker compose down

# テストを実行する
test:
    docker compose exec app php artisan test

# Docker イメージをビルドする
build:
    docker compose build --no-cache

# lint + テスト + ビルドをまとめて
ci: lint test build

lint:
    docker compose exec app ./vendor/bin/phpstan analyse
```

あとは実行するだけ：

```sh
just up       # コンテナ起動
just test     # テスト
just ci       # lint + test + build を一発実行
just          # レシピ一覧を表示
```

---

## Makefile との違い

| 比較項目 | Make | just |
|---|---|---|
| 目的 | ビルドツール | タスクランナー |
| タブ必須 | ✅（バグの温床） | ❌（スペースOK） |
| 変数展開 | `$$(VAR)` と複雑 | `{{VAR}}` でシンプル |
| 引数サポート | 限定的 | ✅ ネイティブサポート |
| .PHONY 宣言 | 必要 | 不要 |
| エラーメッセージ | 難解 | わかりやすい |
| クロスプラットフォーム | △ | ✅ |

Make は「タスクランナー」として使うには設計が合っておらず、謎のエラーに悩まされることがあります。`just` はその苦痛を取り除いてくれます。

---

## チームで嬉しい機能

### 1. 引数を受け取れる

```justfile
# 環境を指定してデプロイ
deploy env="staging":
    echo "{{env}} にデプロイ中..."
    ./scripts/deploy.sh {{env}}
```

```sh
just deploy           # staging にデプロイ（デフォルト）
just deploy prod      # prod にデプロイ
```

### 2. `.env` ファイルの自動読み込み

```justfile
set dotenv-load

up:
    docker compose up -d   # .env の変数が自動で使える
```

### 3. レシピにドキュメントコメントを書ける

```justfile
# データベースのマイグレーションを実行する
migrate:
    docker compose exec app php artisan migrate

# テストを特定のファイルに絞って実行する
# 例: just test-file tests/Feature/UserTest.php
test-file file:
    docker compose exec app ./vendor/bin/phpunit {{file}}
```

`just --list` を実行すると、コメントがそのままドキュメントになります：

```
Available recipes:
    migrate           # データベースのマイグレーションを実行する
    test-file file    # テストを特定のファイルに絞って実行する
```

### 4. 複数の言語・シェルに対応

```justfile
# Python スクリプトを実行
analyze:
    #!/usr/bin/env python3
    import json
    data = json.load(open("data.json"))
    print(f"件数: {len(data)}")

# PowerShell (Windows)
info:
    #!powershell
    Get-SystemInfo
```

### 5. OS 分岐も簡単

```justfile
open:
    {{ if os() == "macos" { "open" } else { "xdg-open" } }} http://localhost:3000
```

### 6. `mod` でファイルを分割できる

Justfile が大きくなってくると、1ファイルに全てのレシピを詰め込むのは見通しが悪くなります。`just` には **`mod`（モジュール）** 機能があり、Justfile を役割ごとに分割して管理できます。

```
project/
├── justfile              # メインの Justfile（エントリポイント）
└── just/
    ├── db.just           # DB 関連コマンド
    ├── docker.just       # Docker 操作
    └── deploy.just       # デプロイ関連
```

メインの `justfile` では `mod` でサブモジュールを宣言します：

```justfile
# DB マイグレーション・シード
mod db 'just/db.just'
# Docker コンテナ操作
mod docker 'just/docker.just'
# デプロイ関連
mod deploy 'just/deploy.just'

# コマンド一覧を表示
default:
    @just --list

# テストを実行
test *args:
    docker compose exec app ./vendor/bin/phpunit {{args}}

# 静的解析
lint *args:
    docker compose exec app ./vendor/bin/phpstan analyse {{args}}
```

サブモジュール内は通常の Justfile と同じ書き方です。例えば `just/db.just`：

```justfile
set shell := ["bash", "-euo", "pipefail", "-c"]

_container := "myapp-db"

# マイグレーション実行
migrate:
    docker exec -it {{_container}} php artisan migrate

# DB をリセットしてシード投入
fresh:
    docker exec -it {{_container}} php artisan migrate:fresh --seed

# DB のバックアップを取得
backup name:
    docker exec {{_container}} mysqldump mydb > backups/{{name}}.sql
```

実行時は **`just <モジュール名> <レシピ名>`** の形でサブモジュールのレシピを呼び出します：

```sh
just db migrate              # マイグレーション実行
just db fresh                # DB リセット
just db backup 20260313      # バックアップ取得
just docker logs app         # コンテナログ表示
just deploy staging          # ステージングにデプロイ
```

`just --list` を実行すると、サブモジュールも含めた一覧が表示されます：

```
Available recipes:
    default    # コマンド一覧を表示
    lint *args # 静的解析
    test *args # テストを実行
    db         # DB マイグレーション・シード
    deploy     # デプロイ関連
    docker     # Docker コンテナ操作
```

#### `mod` のメリット

- **見通しの良さ**: ドメインごとにファイルが分かれるため、関心事が整理される
- **コンフリクトの軽減**: チームで並行作業しても、別ファイルなので Git のコンフリクトが起きにくい
- **段階的な導入**: 最初はモノリシックな Justfile で始め、大きくなったら `mod` で分割できる
- **スコープの分離**: 各サブモジュール内の変数（`_container` など）は他のモジュールに影響しない

> **Tips**: サブモジュール内でも `default` レシピを定義しておくと、`just db` だけでそのモジュールのレシピ一覧を表示できて便利です。

### 7. `import` で名前空間なしにファイルを分割する

`mod` はサブコマンド形式（`just db migrate`）になりますが、**`import`** を使うと名前空間なしでレシピをそのまま取り込めます。「ファイルは分けたいが、フラットにコマンドを呼びたい」場合に便利です。

```justfile
import 'just/test.just'
import 'just/lint.just'
import 'just/deploy.just'

# コマンド一覧を表示
default:
    @just --list

# 開発環境を起動
up:
    docker compose up -d

# 開発環境を停止
down:
    docker compose down
```

```justfile
# just/test.just

# 全テストを実行
test *args:
    docker compose exec app ./vendor/bin/phpunit {{args}}

# 特定のテストファイルを実行
test-file file:
    docker compose exec app ./vendor/bin/phpunit {{file}}
```

実行時はプレフィックスなしでそのまま呼び出せます：

```sh
just test                    # プレフィックスなしで実行できる
just test-file tests/UserTest.php
just lint
just deploy staging
```

#### `mod` と `import` の使い分け

| | `mod` | `import` |
|---|---|---|
| 呼び出し方 | `just db migrate` | `just migrate` |
| 名前空間 | あり（モジュール名がプレフィックス） | なし（フラットに統合） |
| 変数スコープ | モジュールごとに独立 | 親の Justfile と共有 |
| レシピ名の衝突 | モジュールが分かれるので起きない | 同名レシピがあるとエラー |
| 向いている場面 | 独立性が高い機能群（DB操作、デプロイなど） | 共通の関心事をファイル分割したい場合 |

実際のプロジェクトでは、両方を組み合わせて使うのがおすすめです。例えば、日常的に使う `test` や `lint` は `import` でフラットに呼べるようにし、DB 操作やデプロイのように独立した機能群は `mod` でグループ化するとバランスが取れます。

---

## 実際のチームでの Justfile 例

```justfile
set dotenv-load

aws_account_id := env("AWS_ACCOUNT_ID", "123456789012")
aws_region     := env("AWS_REGION", "ap-northeast-1")
ecr_repo       := aws_account_id + ".dkr.ecr." + aws_region + ".amazonaws.com/myapp"

# デフォルト: レシピ一覧を表示
default:
    @just --list

# 開発環境を起動
up:
    docker compose up -d

# 開発環境を停止
down:
    docker compose down

# テストを実行
test *args:
    docker compose exec app ./vendor/bin/phpunit {{args}}

# 静的解析
lint *args:
    docker compose exec app ./vendor/bin/phpstan analyse {{args}}

# ECR にログインしてイメージをプッシュ
push tag="latest":
    aws ecr get-login-password --region {{aws_region}} | docker login --username AWS --password-stdin {{ecr_repo}}
    docker build -t {{ecr_repo}}:{{tag}} .
    docker push {{ecr_repo}}:{{tag}}

# ECS にデプロイ（確認あり）
deploy-prod:
    @echo "⚠️  本番環境にデプロイします。よろしいですか？ (Enter で続行)"
    @read
    just push prod
    aws ecs update-service --cluster myapp-cluster --service myapp-service --force-new-deployment

# 新メンバー向けセットアップ
setup:
    cp .env.example .env
    docker compose up -d --build
    docker compose exec app composer install
    docker compose exec app php artisan migrate --seed
    @echo "✅ セットアップ完了！ 'just up' で起動できます"
```

---

## 導入のすすめ

`just` の特に大きなメリットは **「オンボーディングコストの削減」** です。

新しいメンバーがジョインしたとき、`just setup` 一発で環境構築が完了し、`just --list` でそのプロジェクトで何ができるかが一目でわかる。README に長々とコマンドを書く必要もなくなります。

また、`Justfile` はコードとして Git 管理されるため、**「あの人しか知らないコマンド」がチームの共有資産** になります。

---

## まとめ

- ✅ Makefile のような記法で、より直感的
- ✅ 引数・デフォルト値・環境変数など柔軟な機能
- ✅ `just --list` でセルフドキュメント化
- ✅ チームのオンボーディングを大幅に改善
- ✅ インストールは1コマンドで完了

Makefile を「なんとなく」使っているチームや、README にコマンド手順を書き続けているチームには、ぜひ `just` を試してみてください。きっと「なんでもっと早く使わなかったんだろう」と思うはずです。

---

## 参考資料

- [just 公式ドキュメント](https://just.systems/man/en/)
- [GitHub](https://github.com/casey/just)

---

:::message
本記事は生成AI（Claude Code）を使用して作成し、筆者が内容を確認・編集しています
:::
