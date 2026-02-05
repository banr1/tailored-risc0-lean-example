# README 更新計画

## 概要

既存の `README.md`（英語）を日本語に書き換え、以下の3セクションを追加・整理する。

## 変更対象ファイル

- `README.md`

## 新しい README 構成

### 1. タイトル + 概要

Lean 4 を RISC Zero zkVM 上で証明実行するテンプレートの説明。既存の冒頭文を日本語化して拡充。

### 2. アーキテクチャ

CLAUDE.md のアーキテクチャ図をベースに、以下を含む:

- ビルドパイプライン図（ASCII）:
  ```
  Lean 4 ──Lake──▶ C IR ──CMake──▶ RISC-V static lib ──Cargo──▶ Guest ELF
                                                                   │
                                                            Host が証明実行
  ```
- ディレクトリ構成テーブル（`guest/`, `guest_build/`, `methods/`, `host/`）
- FFI データフロー図

### 3. Getting Started

段階的なセットアップ手順:
1. 前提条件（Lean 4.22.0 via elan, CMake, Cargo）
2. `just` インストール
3. RISC0 ツールチェーンインストール（rzup → cpp, rust, r0vm）
4. 環境変数設定
5. lean-risc0-runtime のビルド・インストール
6. lean-risc0-init のビルド・インストール
7. `mkdir -p methods/guest/lib`
8. `just build`
9. 動作テスト（`target/release/host 1`）

### 4. 既存セクションの保持（日本語化）

- カスタム Lean プログラムの組み込み方法
- パフォーマンスノート
- 実装詳細
- 関連プロジェクト

## 方針

- 既存 README の情報は全て保持しつつ日本語に変換
- CLAUDE.md の内容と重複する部分は README 側を充実させる
- Getting Started は実際の構築経験（r0vm 必要、Rust 1.85+ 必要等）を反映した実用的な手順にする

## 検証

- `README.md` の Markdown が正しくレンダリングされることを確認（リンク、テーブル、コードブロック）
