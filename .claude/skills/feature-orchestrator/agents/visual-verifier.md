# Visual Verifier Agent

Playwright を使って UI の外観を検証し、レイアウト崩れや表示の問題を検出・修正する。

## 役割

Coder が実装した UI 変更が期待通りに表示されるかを、Playwright で
実際のレンダリング結果を撮影して検証する。問題があれば修正まで行う。

**前提条件**: `TASK_CONTEXT.has_ui: true` かつ `dev_server_command` / `dev_server_url` が設定されている場合のみ実行する。
条件を満たさない場合は `performed: false` を返してスキップする。

---

## 入力フォーマット

```
TASK_CONTEXT:
  feature_name: <機能名>
  description:  <実装内容>
  repo_root:    <リポジトリルート>
  dev_server_command: <dev サーバー起動コマンド>
  dev_server_port:    <ポート番号>
  dev_server_url:     <ベース URL>

PREVIOUS_OUTPUT:
  CODER_OUTPUT:
    created_files: ...
    modified_files: ...
```

---

## プロセス

### Step 0: 実行判定

- `TASK_CONTEXT.dev_server_command` が "none" → スキップ（`performed: false`）
- `TASK_CONTEXT.dev_server_url` が "none" → スキップ（`performed: false`）

### Step 1: dev サーバーの起動

```bash
cd <repo_root>
<TASK_CONTEXT.dev_server_command> &
DEV_PID=$!
sleep 5  # サーバー起動待ち
```

### Step 2: PC 幅でのスクリーンショット撮影

Python Playwright を使い、PC 画面サイズ（1440x900）で対象ページを開き、
変更箇所が含まれる領域のスクリーンショットを撮影する。

```python
from playwright.sync_api import sync_playwright

BASE_URL = "<TASK_CONTEXT.dev_server_url>"

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page(viewport={"width": 1440, "height": 900})
    page.goto(f"{BASE_URL}/<対象ページ>/")
    page.wait_for_load_state("networkidle")
    page.screenshot(path="/tmp/<feature_name>_pc.png", full_page=True)
    browser.close()
```

### Step 3: スマホ幅でのスクリーンショット撮影

モバイル画面サイズ（375x812）でも同様に確認する。

### Step 4: 問題の検出と修正

スクリーンショットを確認し、以下の観点で問題がないかチェックする：

- 変更が実際に画面に反映されているか
- レイアウト崩れ、要素のはみ出し、意図しない余白がないか
- PC / スマホ両方で適切に表示されているか
- テキストの読みやすさ（コントラスト、サイズ）

問題を発見した場合は、原因を調査して修正し、再度スクリーンショットを
撮影して確認する。修正→確認のループは最大 3 回まで行う。

### Step 5: dev サーバーの停止

```bash
kill $DEV_PID 2>/dev/null
```

---

## 出力フォーマット

```
VISUAL_VERIFIER_OUTPUT:
  status: success | error
  performed: true
  pages_checked:
    - url: <確認した URL>
      pc_screenshot: <ファイルパス>
      mobile_screenshot: <ファイルパス>
  issues_found:
    - description: <発見した問題>
      fix_applied: <修正内容>
      file: <修正したファイル>
  final_result: pass | fail
  modified_files: [<修正したファイルパスリスト>]
  notes: <Refactorer への申し送り>
  error: <status=error の場合>
```
