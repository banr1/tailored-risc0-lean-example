# Lean 4 in RISC Zero Guest

[Lean 4](https://lean-lang.org/) を [RISC Zero](https://risczero.com/) zkVM 上で証明実行するためのテンプレートです。Lean が生成する C コードを、カスタムランタイムおよび RISC Zero 向けにプリコンパイルされた Lean Init ライブラリと組み合わせてビルドします。

自分の Lean 4 プログラムを RISC Zero zkVM で実行するには、`guest/` ディレクトリを自分の Lean 4 ソースに置き換えるだけです。対応する Lean バージョンは **4.22.0** です。

## アーキテクチャ

### ビルドパイプライン

```
Lean 4 ──Lake──▶ C IR ──CMake──▶ RISC-V static lib ──Cargo──▶ Guest ELF
                                                                 │
                                                          Host が証明実行
```

### ディレクトリ構成

| ディレクトリ | 役割 |
|-------------|------|
| `guest/` | Lean 4 ソース。Lake でビルドして C IR を生成 |
| `guest_build/` | CMake プロジェクト。C IR を RISC-V 32bit 静的ライブラリにクロスコンパイル |
| `methods/` | Rust ゲストクレート。FFI で Lean 静的ライブラリをリンクし ELF を生成 |
| `host/` | Rust ホスト。ゲスト ELF をロードし zkVM で証明実行 |

### FFI 境界とデータフロー

```
Lean: @[export risc0_main] def risc0_main (input : ByteArray) : ByteArray
  ↕ FFI
C:    lean_risc0_main(lean_object* input) → lean_object*
  ↕ safe wrapper
Rust: risc0_main(input: &[u8]) → Vec<u8>
  ↕ Host I/O (env::read / env::commit)
Host: u32 → ByteArray → ByteArray → u32
```

## Getting Started

### 前提条件

- [Lean 4](https://lean-lang.org/) 4.22.0（[elan](https://github.com/leanprover/elan) で管理）
- CMake
- Cargo（Rust 1.85 以上。edition2024 を使用するクレートがあるため）
- [just](https://github.com/casey/just) コマンドランナー

### 1. just のインストール

```bash
brew install just
```

### 2. RISC Zero ツールチェーンのインストール

```bash
curl -L https://risczero.com/install | bash
rzup install cpp 2024.1.5
rzup install rust
rzup install r0vm
```

> **注意**: `r0vm` のインストールを忘れると実行時に `No such file or directory` エラーが発生します。

### 3. 環境変数の設定

シェルの設定ファイル（`~/.zshrc` 等）に以下を追加します:

```bash
export LEAN_RISC0_PATH="$HOME/.lean-risc0"
export RISC0_TOOLCHAIN_PATH="$HOME/.risc0/cpp"
```

### 4. Lean RISC0 ランタイムのビルド・インストール

```bash
git clone https://github.com/anoma/lean-risc0-runtime
cd lean-risc0-runtime
just build
just install
```

### 5. Lean RISC0 Init 標準ライブラリのビルド・インストール

```bash
git clone https://github.com/anoma/lean-risc0-init
cd lean-risc0-init
just build    # 約 394 ファイルのビルドで時間がかかります
just install
```

### 6. ビルドと実行

```bash
# ライブラリ出力ディレクトリの作成（justfile に含まれていないため手動で必要）
mkdir -p methods/guest/lib

# フルビルド
just build

# 動作テスト
target/release/host 1
```

個別ステップでビルドする場合:

```bash
cd guest && lake build              # Lean → C IR
cd guest_build && just build        # CMake クロスコンパイル (RISC-V static lib)
cargo build --release               # Cargo リンク → ゲスト ELF + ホストバイナリ
```

## メインの例

`main` ブランチには、Lean の `Nat` に対する `sum` 関数の例が含まれています。バイト配列を経由して Lean 4 と通信する汎用インターフェースを実装しており、Lean 側でバイト配列を `Nat` にパースし、結果をバイト配列で返します。Rust 側でそれをパースします。ランタイムの初期化も適切に行われます。

## Sum の例

`sum-example` ブランチには、32 ビット符号なし整数に対する軽量な `sum` 関数の例が含まれています。この例ではランタイムの初期化を行いません。

## カスタム Lean プログラムの組み込み

`guest/` ディレクトリを自分の Lean 4 プロジェクトで置き換えます。対応する Lean バージョンは 4.22.0 です。プロジェクトには以下が必要です:

1. ルートディレクトリに [Guest.lean](https://github.com/anoma/risc0-lean-example/blob/main/guest/Guest.lean) ファイルを配置し、全プロジェクトモジュールをインポートする
2. 以下のシグネチャで C にエクスポートされるエントリポイント関数を定義する:
   ```lean
   @[export risc0_main]
   def risc0_main (input : ByteArray) : ByteArray := ...
   ```
3. 入力型を変更する場合は [methods/guest/src/main.rs](https://github.com/anoma/risc0-lean-example/blob/main/methods/guest/src/main.rs) と [host/src/main.rs](https://github.com/anoma/risc0-lean-example/blob/main/host/src/main.rs) も修正する

## パフォーマンス

`sum-example` ブランチの軽量な例は、C で書かれた同等の関数を RISC Zero zkVM で実行した場合と同程度の時間（数秒）で完了します。`UInt32` を使用した場合、Lean が生成する C コードは手書きの再帰的な `sum` 関数と同等であり、Lean 固有の初期化も行われないためです。

`main` の完全な例は、Init ライブラリモジュールの初期化のためにより多くの時間がかかります。ランタイム自体の初期化は比較的高速（数秒）ですが、Init モジュールの初期化には約 13 分かかります（16 コア、GPU 未使用の環境）。これは Init に約 400 のモジュールがあり、それぞれを再帰的に辿って初期化を実行する必要があるためです。

## 実装詳細

Lean 4 の RISC Zero へのコンパイルには以下が必要でした:

- 特定の機能（IO、例外、シグナル、スレッド）を除外した [Lean ランタイム](https://github.com/anoma/lean-risc0-runtime)のカスタム版
- [Lean Init ライブラリ](https://github.com/anoma/lean-risc0-init)の RISC Zero 向けコンパイル
- RISC Zero ツールチェーンが提供する `libc` および `libstdc++` ライブラリの[リンク](https://github.com/anoma/risc0-lean-example/blob/main/methods/guest/build.rs)
- 一部の C 関数の[シム](https://github.com/anoma/risc0-lean-example/blob/main/methods/guest/shims.c)の提供

ゲストのリンカーには `--allow-multiple-definition` フラグが必要です。これは脆弱な設定であり、異なるシステムでは失敗する可能性があります。

## 関連プロジェクト

[György Kurucz](https://kuruczgy.com/) が Lean を ESP32-C3 RISC-V マイクロコントローラにクロスコンパイルするポートを作成しています:

- [ブログ記事](https://kuruczgy.com/blog/2024/07/31/lean-esp32/)
- [リポジトリ](https://codeberg.org/kuruczgy/lean-esp32)
