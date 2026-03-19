---
name: feature-orchestrator
description: >
  機能実装のフルサイクルを自動化するオーケストレータースキル。
  「〇〇機能を実装して」「このIssueを対応して」「新しいエンドポイントを作って」のような実装依頼に対して、
  設計→テスト→コード→リファクタリング→レビュー→Git操作の各専門エージェントを順番に呼び出し、
  クリーンアーキテクチャに基づいた高品質な実装を完結させる。
  言語・フレームワーク非依存（汎用）。実装タスクが発生したら必ずこのスキルを使うこと。
---

# Feature Orchestrator

機能実装のフルサイクルを7つの専門 sub-agent に委譲して完結させるオーケストレーター。

**重要: オーケストレーター自身はコードの読み書き・テスト実行・git 操作を一切行わない。**
すべての作業は Agent ツールで専門エージェントに委譲する。
オーケストレーターの仕事は「タスク解析」「エージェント呼び出し」「結果の中継」「ユーザーへの報告」だけ。

## エージェント一覧

| Agent           | ファイル                        | 役割                                          |
| --------------- | ------------------------------- | --------------------------------------------- |
| Designer        | `agents/designer.md`            | ADR作成・クリーンアーキテクチャ設計           |
| Tester          | `agents/tester.md`              | テストを先行作成（TDD の Red フェーズ）       |
| Coder           | `agents/coder.md`               | テストを通す実装（TDD の Green フェーズ）     |
| Visual Verifier | `agents/visual-verifier.md`     | Playwright で UI の外観検証                   |
| Refactorer      | `agents/refactorer.md`          | 可読性・構造・モジュール分解の改善（Refactor）|
| Reviewer        | `agents/reviewer.md`            | コード・テスト・設計の総合レビュー            |
| Git Operator    | `agents/git-operator.md`        | lint + test → commit → main マージ → push     |

---

## 実行フロー（TDD: Red → Green → Refactor）

```
[タスク受け取り]
      │
      ▼
1. Designer        ──→ docs/adr/NNNN-*.md, docs/design/*.md
      │
      ▼
2. Tester (Red)    ──→ テストを先行作成（テストが失敗する状態）
      │
      ▼
3. Coder (Green)   ──→ src/ 以下に実装してテストを通す
      │
      ▼
4. Visual Verifier ──→ UI 変更時のみ Playwright で外観確認（該当しなければスキップ）
      │
      ▼
5. Refactorer      ──→ src/ / tests/ を改善
      │
      ▼
6. Reviewer        ──→ レビューレポート + 要修正リスト
      │
    修正あり？
    Yes ──→ Tester → Coder → Visual Verifier → Refactorer → Reviewer (ループ、最大2回)
    No  ──→
      │
      ▼
7. Git Operator    ──→ lint + test → commit → main マージ → push
```

---

## オーケストレーターの責務

### 1. タスクの解析

受け取ったタスクから以下を抽出する。
**自動検出**: `repo_root` 配下のファイル（`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `Makefile`, `pom.xml`, `build.gradle` 等）を確認し、
言語・フレームワーク・テストコマンド・Lintコマンドを自動推定する。推定できない場合はユーザーに確認する。

```
TASK_CONTEXT:
  feature_name: <機能名（英語スネークケース）>
  description:  <何を実装するか>
  language:     <使用言語（判別できない場合は "unknown"）>
  framework:    <フレームワーク（不明な場合は "unknown"）>
  constraints:  <既存コードとの整合性、パフォーマンス要件など>
  repo_root:    <リポジトリルートのパス>
  has_ui:       <true | false（UI/デザイン変更を含むか）>
  # ── ビルド・テスト・Lint コマンド（自動検出 or ユーザー指定） ──
  test_command:       <テスト実行コマンド（例: "npm test", "pytest", "go test ./..."）>
  lint_command:       <Lint 実行コマンド（例: "npm run lint", "ruff check .", 無ければ "none"）>
  lint_fix_command:   <Lint 自動修正コマンド（例: "npm run lint:fix", "ruff check --fix .", 無ければ "none"）>
  # ── UI 検証用（has_ui: true の場合のみ） ──
  dev_server_command: <dev サーバー起動コマンド（例: "npm run dev", "python manage.py runserver", 無ければ "none"）>
  dev_server_port:    <dev サーバーのポート番号（例: 3000, 8000, 無ければ "none"）>
  dev_server_url:     <検証用ベース URL（例: "http://localhost:3000", 無ければ "none"）>
  # ── Git ワークフロー ──
  git_workflow:       <"direct_merge" | "pull_request"（デフォルト: "pull_request"）>
```

### 2. エージェントの呼び出し方法

各エージェントは Claude Code の **Agent ツール** で呼び出す。
エージェント定義ファイル（`agents/*.md`）の内容を読み取り、
タスクコンテキストと前ステップの出力をプロンプトに含めて委譲する。

```
Agent ツール呼び出しテンプレート:

description: "<agent名> - <feature_name>"
prompt: |
  あなたは <agent名> です。以下のエージェント定義に従って作業してください。

  ## エージェント定義
  <agents/<agent>.md の内容>

  ## タスクコンテキスト
  <TASK_CONTEXT>

  ## 前ステップの出力
  <PREVIOUS_OUTPUT>

  作業を完了し、定義された出力フォーマットで結果を報告してください。
```

エージェントの呼び出しは直列で行う（前ステップの出力が次ステップの入力になるため）。

### 3. オーケストレーターがやらないこと

以下の操作はすべてエージェントに委譲し、オーケストレーター自身は実行しない：

- ファイルの読み書き（Read / Write / Edit ツール）
- コマンド実行（Bash ツール）
- テストや Linter の実行
- git 操作（commit, push 等）
- Playwright でのスクリーンショット撮影

オーケストレーターが使うのは **Agent ツール** と **ユーザーへのテキスト出力** のみ。

### 4. コンテキストの引き継ぎ

各エージェントの出力（Agent ツールの戻り値）を次のエージェントへ渡す。
ファイルを生成したエージェントは生成したファイルパスの一覧を出力に含めるため、
次のエージェントはそのパスを使ってファイルを読み取れる。

### 5. Visual Verifier のスキップ判定

`has_ui: false` の場合、Visual Verifier はスキップする。
また `has_ui: true` でも `dev_server_command: "none"` の場合はスキップし、レポートに理由を記載する。

判定基準：

- コンポーネント、CSS、レイアウト、スタイリングの変更 → `has_ui: true`
- CLI ツール、ライブラリ、API サーバー、バッチ処理、ユーティリティ関数のみ → `has_ui: false`

### 6. レビューループの管理

Reviewer の出力に `NEEDS_REVISION: true` が含まれる場合、以下を再実行する（最大2回）：

1. Tester（指摘に対応するテストの追加・更新）
2. Coder（テストを通すための修正）
3. Visual Verifier（UI 変更がある場合のみ）
4. Refactorer（再リファクタリング）
5. Reviewer（再レビュー）

2回ループしても承認されない場合は、レビュー結果をユーザーに提示して判断を仰ぐ。

### 7. 完了報告

Git Operator 完了後、以下をユーザーに報告する：

```
## 実装完了レポート

### 概要
- 機能名: <feature_name>
- ブランチ: <branch_name>
- push: <push_status>

### 生成ファイル
- 設計書: <adr_path>, <design_doc_path>
- 実装: <src_files>
- テスト: <test_files>

### テスト結果
<test_summary>

### レビュー指摘（対応済み）
<resolved_issues>
```

---

## ディレクトリ規約

エージェントが生成するファイルは以下のディレクトリに配置する（リポジトリ構造が既に存在する場合はそれを優先）：

```
<repo_root>/
├── docs/
│   ├── adr/
│   │   └── NNNN-<feature-name>.md   # ADRはゼロ埋め4桁連番
│   └── design/
│       └── <feature-name>.md
├── src/                              # または lib/, app/ など既存構造に従う
│   └── <language-specific layout>
└── tests/                            # または test/, spec/ など
    └── <language-specific layout>
```

---

## エラーハンドリング

| 状況                             | 対応                                         |
| -------------------------------- | -------------------------------------------- |
| エージェントがエラーを返した     | エラー内容をユーザーに提示し、継続可否を確認 |
| テストが全件失敗                 | Coder に差し戻し（Testerの出力を添付）       |
| レビューループが上限超過         | 現状をユーザーに報告し手動介入を依頼         |
| ファイル競合が発生               | Git Operator が検出・報告する                |
| Visual Verifier で問題を検出     | Coder に差し戻し（スクリーンショットを添付） |
