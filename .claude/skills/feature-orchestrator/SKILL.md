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

## 実行フロー（TDD: Red → Green → Refactor + 各ステップレビュー）

各ステップ完了後にそのステップ専用のレビューを実施し、**軽微な指摘を含め全指摘が0件になるまで**
該当ステップをやり直す。これにより、後工程に品質問題が持ち越されることを防ぐ。

```
[タスク受け取り]
      │
      ▼
1. Designer        ──→ docs/adr/NNNN-*.md, docs/design/*.md
      │
      ▼
1R. Reviewer (設計)  ──→ 設計レビュー
      │                    指摘 > 0 → Designer に差し戻し（1 に戻る）
      │                    指摘 = 0 → 次へ
      ▼
2. Tester (Red)    ──→ テストを先行作成（テストが失敗する状態）
      │
      ▼
2R. Reviewer (テスト) ──→ テストレビュー
      │                    指摘 > 0 → Tester に差し戻し（2 に戻る）
      │                    指摘 = 0 → 次へ
      ▼
3. Coder (Green)   ──→ src/ 以下に実装してテストを通す
      │
      ▼
3R. Reviewer (実装) ──→ 実装レビュー
      │                    指摘 > 0 → Coder に差し戻し（3 に戻る）
      │                    指摘 = 0 → 次へ
      ▼
4. Visual Verifier ──→ UI 変更時のみ Playwright で外観確認（該当しなければスキップ）
      │
      ▼
5. Refactorer      ──→ src/ / tests/ を改善
      │
      ▼
5R. Reviewer (リファクタ) ──→ リファクタリングレビュー
      │                        指摘 > 0 → Refactorer に差し戻し（5 に戻る）
      │                        指摘 = 0 → 次へ
      ▼
6. Reviewer (総合)  ──→ 全成果物の横断レビュー
      │                  指摘 > 0 → 該当エージェントに差し戻し → 再レビュー（6 に戻る）
      │                  指摘 = 0 → 次へ
      ▼
7. Git Operator    ──→ lint + test → commit → main マージ → push
```

**各ステップレビューの目的**: 問題を早期に検出・修正することで、後工程での手戻りコストを最小化する。
最終段の総合レビュー（Step 6）では、個別ステップレビューでは検出できない横断的な整合性問題を検出する。

---

## オーケストレーターの責務

### 1. タスクの解析とサブタスク分割

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
  # ── サブタスク分割（大規模タスクの場合） ──
  is_large_task:      <true | false>
  subtasks:           <サブタスク一覧（is_large_task: true の場合）>
```

#### 大規模タスクの判定とサブタスク分割

タスクの解析後、以下の基準で大規模タスクかどうかを判定する：

**大規模タスク判定基準**（いずれか1つに該当すれば `is_large_task: true`）：
- 複数のエンティティ・集約（Aggregate）を新規に作成する必要がある
- 3つ以上のレイヤー（Domain / Use Case / Adapter / Infrastructure）に跨る大幅な変更が必要
- 複数の独立したユースケースを含む（例:「CRUD全部作って」「認証と認可を実装して」）
- UIとバックエンドの両方に大きな変更がある
- 既存の複数モジュールに対する横断的な変更が必要

**`is_large_task: true` の場合、以下の手順でサブタスクに分割する：**

##### Step A: 依存関係の分析

タスクに含まれる機能を洗い出し、依存関係を整理する。
依存される側（下位レイヤー、共通部品）から順に実装できるよう依存グラフを構築する。

##### Step B: サブタスクへの分割

以下の原則に従い、タスクを独立した小さなサブタスクに分割する：

1. **1サブタスク = 1つの凝集した変更**: 1つのユースケース、1つのエンティティ、1つのAPIエンドポイントなど
2. **依存順に並べる**: 依存される側から先に実装（例: Domain → Use Case → API → UI）
3. **各サブタスクが単独でテスト可能**: サブタスク完了時点でテストが Green になる状態を保つ
4. **各サブタスクが単独でコミット可能**: 中間状態でもコードベースが壊れない

##### Step C: サブタスク一覧の作成と確認

分割結果を以下の形式で作成し、**ユーザーに提示して確認を得てから実装を開始する**：

```
SUBTASK_PLAN:
  total_subtasks: <件数>
  subtasks:
    - id: 1
      name: <サブタスク名（英語スネークケース）>
      description: <何を実装するか>
      depends_on: []  # 依存する先行サブタスクの id リスト
      estimated_scope: <small | medium | large>
      layers_affected: [domain, usecase, adapter, infra]  # 変更対象レイヤー
    - id: 2
      name: <サブタスク名>
      description: <何を実装するか>
      depends_on: [1]
      estimated_scope: <small | medium | large>
      layers_affected: [usecase, adapter]
    ...
```

**分割の粒度の目安**:
- `small`: 1〜2ファイルの変更。1つの関数・メソッド追加程度
- `medium`: 3〜6ファイルの変更。1つのユースケース全体の実装
- `large`: 7ファイル以上の変更。できればさらに分割する

**分割例**:

```
タスク: 「ユーザー管理機能（CRUD + 認証）を実装して」

サブタスク分割:
  1. user_entity        - User エンティティと Value Object の定義（Domain 層）
  2. user_repository    - UserRepository インターフェース定義（Domain 層）
  3. create_user        - ユーザー作成ユースケース（Use Case + Infrastructure）
  4. get_user           - ユーザー取得ユースケース（Use Case + Infrastructure）
  5. update_user        - ユーザー更新ユースケース（Use Case + Infrastructure）
  6. delete_user        - ユーザー削除ユースケース（Use Case + Infrastructure）
  7. user_api           - ユーザー CRUD の API エンドポイント（Adapter 層）
  8. auth_token         - 認証トークン発行・検証（Domain + Infrastructure）
  9. auth_middleware     - 認証ミドルウェア（Adapter 層）
  10. auth_api          - ログイン・ログアウト API（Adapter 層）
```

#### サブタスク実行フロー

`is_large_task: true` の場合、実行フローが以下のように変わる：

```
[タスク受け取り]
      │
      ▼
[タスク解析 + サブタスク分割]
      │
      ▼
[サブタスク計画レビュー] ── Reviewer (subtask_plan) で分割計画をレビュー
      │                       指摘 > 0 → 計画を修正 → 再レビュー（指摘0件まで）
      │                       指摘 = 0 → 次へ
      ▼
[ユーザーにサブタスク計画を提示 → 承認]
      │
      ▼
[全体設計] ── Designer が全サブタスクを俯瞰した ADR・設計書を作成
      │         └→ 設計レビュー（指摘0件まで）
      ▼
┌─ サブタスク 1 ───────────────────────────────────┐
│  Tester (Red) → テストレビュー(0件まで)           │
│  Coder (Green) → 実装レビュー(0件まで)            │
│  Visual Verifier（該当時のみ）                    │
│  Refactorer → リファクタレビュー(0件まで)         │
│  Git Operator → commit（まだマージしない）        │
└──────────────────────────────────────────────────┘
      │
      ▼
┌─ サブタスク 2 ───────────────────────────────────┐
│  （同じフロー）                                   │
│  ※ 前サブタスクの成果物を PREVIOUS_OUTPUT に含める│
└──────────────────────────────────────────────────┘
      │
      ▼
    ... (サブタスク N まで繰り返し)
      │
      ▼
[全サブタスク完了後]
      │
      ▼
Reviewer (総合) ── 全成果物の横断レビュー（指摘0件まで）
      │
      ▼
Git Operator ── 全コミットを squash or まとめて main マージ → push
```

**サブタスク実行時の注意点**：

1. **設計は最初に全体を作る**: Designer は全サブタスクを俯瞰した設計書を1度で作成する。サブタスクごとに設計書を分割しない。これにより、全体の整合性を保証する
2. **各サブタスクは前のサブタスクの成果を引き継ぐ**: PREVIOUS_OUTPUT に全先行サブタスクの結果を含める
3. **各サブタスク完了時にテスト全体を実行**: 新規テストだけでなく、先行サブタスクのテストも全て Green であることを確認する（リグレッション防止）
4. **各サブタスクで commit する**: サブタスクごとに Git Operator で commit する。ただし main へのマージは全サブタスク完了後にまとめて行う
5. **進捗をユーザーに報告**: 各サブタスクの開始時と完了時にユーザーへ報告する

```
📋 サブタスク 3/10: create_user（ユーザー作成ユースケース）を開始します...
✅ サブタスク 3/10: create_user 完了（テスト: 12 passed, レビュー: 2回で0件達成）
```

#### 小規模タスクの場合

`is_large_task: false` の場合は、従来通りサブタスク分割なしで直接実行フローに入る。

### 2. エージェントの呼び出し方法

各エージェントは Claude Code の **Agent ツール** で呼び出す。
エージェント定義ファイル（`agents/*.md`）の内容を読み取り、
タスクコンテキストと前ステップの出力をプロンプトに含めて委譲する。

```
Agent ツール呼び出しテンプレート（作業エージェント）:

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

```
Agent ツール呼び出しテンプレート（ステップレビュー）:

description: "Reviewer (<review_mode>) - <feature_name> (iteration <N>)"
prompt: |
  あなたは Reviewer です。以下のエージェント定義に従ってレビューしてください。

  ## エージェント定義
  <agents/reviewer.md の内容>

  ## タスクコンテキスト
  <TASK_CONTEXT>

  ## レビュー設定
  REVIEW_CONFIG:
    review_mode: <design | test | implementation | refactoring | comprehensive>
    iteration: <N>
    previous_issues: <前回の指摘（初回は空）>

  ## 前ステップの出力
  <PREVIOUS_OUTPUT>

  全チェックリスト項目を漏れなく確認し、軽微な指摘も含めて全て報告してください。
  指摘が0件の場合のみ needs_revision: false としてください。
```

エージェントの呼び出しは直列で行う（前ステップの出力が次ステップの入力になるため）。
**各作業エージェント完了後、必ずステップレビューを実行し、指摘0件になるまでループする。**

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

### 6. 各ステップレビューの管理

各ステップ完了後に、そのステップ専用のレビューモードで Reviewer を呼び出す。
**軽微な指摘（advisory）を含め、指摘が1件でも残っている場合は該当ステップをやり直す。**
指摘が0件になるまでループを繰り返す。

#### ステップレビューの呼び出し方

Reviewer を呼び出す際、`review_mode` を指定して対象を限定する：

| ステップ完了後 | review_mode       | レビュー対象                             |
| -------------- | ----------------- | ---------------------------------------- |
| Designer       | `design`          | ADR・設計書の品質                        |
| Tester (Red)   | `test`            | テスト設計の網羅性・品質                 |
| Coder (Green)  | `implementation`  | 実装の品質・設計書との整合               |
| Refactorer     | `refactoring`     | リファクタリング品質・テスト維持         |
| 全完了後       | `comprehensive`   | 全成果物の横断的整合性チェック           |

#### ループの流れ

```
1. ステップのエージェントを実行
2. Reviewer を該当 review_mode で呼び出し
3. total_issues（critical + advisory の合計）を確認
   - total_issues > 0 → 指摘内容を該当エージェントに渡して再実行 → 2 に戻る
   - total_issues = 0 → 次のステップへ進む
```

#### ユーザーへの進捗報告

各レビューループの開始時に、ユーザーへ現在のステップとレビュー回数を報告する：

```
📋 設計レビュー（2回目）: 前回の指摘 3件を修正中...
```

#### 総合レビュー（Step 6）のループ

総合レビューで指摘が出た場合は、指摘の `layer` に応じて該当エージェントに差し戻す：
- `design` → Designer
- `implementation` → Coder
- `test` → Tester
- 構造・全体 → Refactorer

差し戻し修正後、再度 Reviewer を `comprehensive` モードで呼び出す。
これを指摘が0件になるまで繰り返す。

### 7. 完了報告

Git Operator 完了後、以下をユーザーに報告する。
大規模タスクの場合はサブタスクごとの実施結果も含める。

```
## 実装完了レポート

### 概要
- 機能名: <feature_name>
- ブランチ: <branch_name>
- push: <push_status>
- タスク規模: <small（単一タスク） | large（サブタスク分割: N件）>

### 生成ファイル
- 設計書: <adr_path>, <design_doc_path>
- 実装: <src_files>
- テスト: <test_files>

### テスト結果
<test_summary>

### レビュー実施サマリー
| ステップ       | レビュー回数 | 検出指摘数（累計） | 最終結果 |
| -------------- | ------------ | ------------------ | -------- |
| 設計           | <N>回        | <件数>             | 全件解消 |
| テスト         | <N>回        | <件数>             | 全件解消 |
| 実装           | <N>回        | <件数>             | 全件解消 |
| リファクタリング | <N>回      | <件数>             | 全件解消 |
| 総合           | <N>回        | <件数>             | 全件解消 |

### レビュー指摘（全件対応済み）
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
