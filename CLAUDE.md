# tailored-risc0-lean-example

Lean 4 プログラムを RISC Zero zkVM 上で証明可能な実行ファイルにビルドするテンプレート。

## Build / Run

```bash
just build          # フルビルド: Lean → C IR → CMake → Cargo
just clean          # 全ビルド成果物を削除
target/release/host 42  # 数値入力で実行（ホストがゲストを証明実行）
```

個別ステップ:

```bash
cd guest && lake build              # Lean → C IR
cd guest_build && just build        # CMake クロスコンパイル (RISC-V static lib)
cargo build --release               # Cargo リンク → ゲスト ELF + ホストバイナリ
```

## 必須環境変数

| 変数                   | 説明                                                 |
| ---------------------- | ---------------------------------------------------- |
| `LEAN_RISC0_PATH`      | Lean RISC0 ランタイムへのパス (通常 `~/.lean-risc0`) |
| `RISC0_TOOLCHAIN_PATH` | RISC0 ツールチェーンへのパス                         |

## アーキテクチャ

```
Lean 4 ──Lake──▶ C IR ──CMake──▶ RISC-V static lib ──Cargo──▶ Guest ELF
                                                                 │
                                                          Host が証明実行
```

### ディレクトリ構成

| ディレクトリ   | 役割                                                                      |
| -------------- | ------------------------------------------------------------------------- |
| `guest/`       | Lean 4 ソース。Lake でビルドして C IR を生成                              |
| `guest_build/` | CMake プロジェクト。C IR を RISC-V 32bit 静的ライブラリにクロスコンパイル |
| `methods/`     | Rust ゲストクレート。FFI で Lean 静的ライブラリをリンクし ELF を生成      |
| `host/`        | Rust ホスト。ゲスト ELF をロードし zkVM で証明実行                        |

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

## カスタム Lean プログラムの組み込み

1. `guest/` を自分の Lean 4 プロジェクトで置き換える
2. `Guest.lean` に以下のシグネチャのエントリポイントを定義する:
   ```lean
   @[export risc0_main]
   def risc0_main (input : ByteArray) : ByteArray := ...
   ```
3. Lean バージョンは **4.22.0** に固定（`lean-toolchain` で指定）
