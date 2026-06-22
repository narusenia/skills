---
name: pr-comment
description: PRレビュー指摘をGitHubインラインコメントとして投稿する
argument-hint: "<PR番号 or URL> [--file <comments.md>]"
---

# pr-comment

PRレビュー指摘をGitHub PRインラインコメントとして投稿するグローバルスキル。
`/pr-review`の後段として独立起動する。

## 起動形式

```
/pr-comment #824
/pr-comment https://github.com/o/r/pull/824
/pr-comment 824 --file /tmp/pr-824-comments.md
```

- 第1引数: PR番号 or URL（必須）
- `--file <path>`: コメント記載の.mdファイル。省略時は会話コンテキストからmd生成。

---

## コメント.mdフォーマット

```markdown
## 1. `src/features/calls/validations.ts:101-108`

[must] 担当NGダイアログのsubmitが壊れる可能性があります。
`personInChargeDialogSchema`の`needs_*`が`.optional()`なしの必須オブジェクトなのに、
PRでFormGroupを全削除してるので、フィールド未登録→zodResolverが`Required`エラーかと。
最小修正: `needs_*`を`.optional()`にするとよいかもです。

## 2. `src/features/deals/field-reconciliation.test.ts:24`

[should] 型レベル保証が失われていそうです。
```

### フォーマット規則

- 見出し: `## N. \`filepath:line\`` または `## N. \`filepath:line-line\``
- 本文1行目: severity tag で始める
- severity: `[must]` / `[should]` / `[nit]` / `[question]` / `[suggestion]`
- 本文: ninomiyaスタイル（後述）で記述
- コードブロック: ` ```ts ` or ` ```suggestion `（GitHub suggested changes）

---

## ninomiyaライティングスタイル

### 基本ルール

- ですます調
- 英数字・バッククォート前後に半角スペース不要（`zodResolver`が ← 正）
- 技術用語は英語のまま
- 具体的な修正提案をできるだけ含める
- file:line参照はバッククォートで括る

### 文末パターン（コメント長に応じてカジュアル度を調整）

**短文（1〜2文、[nit]など軽い指摘）→ カジュアル寄り:**
- `〜かもです。`
- `〜いいかもです！`
- `〜の方がいいかもです。`
- `〜良さそうです。`
- `〜で統一して行ければと…！`
- `〜でお願いします！`

**長文（問題→原因→修正案、[must]/[should]など重い指摘）→ 丁寧寄り:**
- `〜かなと思いました。` / `〜かなとおもいました。`
- `〜かと。`
- `〜気がします。`
- `〜可能性があります。`
- `〜よいかもです。`
- `〜いいのかなと。`
- `〜安心かもです。`
- `〜推奨します。`（具体的修正案のとき）

### 構造パターン

- 1行目: `[severity]` + 要約1文
- 本文: 問題→原因→影響の順
- 修正案: コードブロックまたは「最小修正:」で提示
- スコープ外の指摘: 「PRスコープ外であれば別issueで」「後続Issue化でもいいかもです」を添える
- 非指摘の補足: `**差分補足**（指摘ではありません）`で明示

---

## 実行フロー（6フェーズ）

```
Phase 1 PR情報取得 → Phase 2 コメント生成/パース → Phase 3 diff検証&リダイレクト
→ Phase 4 ユーザー確認 → Phase 5 GitHub API投稿 → Phase 6 結果レポート
```

### Phase 1: PR情報取得

```bash
# PR番号抽出
PR_NUM=$(echo "$ARG" | grep -oE '[0-9]+$')

# メタデータ取得
gh pr view "$PR_NUM" --json number,headRefOid,url

# owner/repo取得
gh repo view --json nameWithOwner -q '.nameWithOwner'

# diff取得（hunk範囲検証用）
gh pr diff "$PR_NUM" > /tmp/pr-${PR_NUM}.diff

# 変更ファイル一覧
gh pr view "$PR_NUM" --json files --jq '.files[].path'
```

取得する情報:
- `headRefOid` → commit_id（コメント投稿に必須）
- diff hunk範囲（Phase 3の検証用）
- 変更ファイル一覧

### Phase 2: コメント生成/パース

**--file指定あり（編集済み再投稿）:**
- .mdファイルをパースしてコメント一覧を構築

**--file指定なし（新規生成）:**
- 会話コンテキスト（直近の`/pr-review`出力など）からコメントを構成
- ninomiyaスタイルガイドを適用（長短でカジュアル度調整）
- 一時mdファイルを `/tmp/pr-${PR_NUM}-comments.md` に書き出し

**どちらの場合も、投稿前に必ず一時mdを生成する。**

パース結果を以下の構造に変換:

```
{ path: string, line: number, endLine?: number, severity: string, body: string }[]
```

### Phase 3: diff検証 & リダイレクト

各コメントについて順に検証:

**Step 1: ファイルがdiffに含まれるか**
- 含まれる → Step 2へ
- 含まれない → リダイレクト先を探索:
  1. diff内ファイルのimport文を`grep`して、diff外ファイルを参照しているdiff内ファイルを特定
  2. 見つかった → そのファイルの関連するhunk行にリダイレクト。コメント冒頭に補記:
     `(実ファイル: \`src/features/calls/validations.ts:101-108\`)`
  3. 見つからない → レビューbodyに移動

**Step 2: 行番号がdiff hunk範囲内か**
- diff hunkの `@@ -old,count +new_start,new_count @@` をパース
- `new_start` 〜 `new_start + new_count - 1` がRIGHT sideの有効範囲
- hunk内 → そのまま投稿
- hunk外 → レビューbodyに移動

**Step 3: side判定**
- 常に `"side": "RIGHT"`（変更後ファイル）

### Phase 4: ユーザー確認

**4.1 一時mdファイルの書き出し（Phase 2で未生成の場合）**

検証・リダイレクト結果を反映した最終版を `/tmp/pr-${PR_NUM}-comments.md` に書き出し。

**4.2 eventの自動提案**

| 条件 | 提案するevent |
|------|---------------|
| `[must]`が1件以上 | `REQUEST_CHANGES` |
| 指摘あり、`[must]`なし | `COMMENT` |
| 指摘0件 | `APPROVE` |

**4.3 AskUserQuestionで確認**

選択肢:
1. **このまま投稿** — 提案されたeventでPOST
2. **修正して返す** — ユーザーがmdファイルを編集後、会話で再開。Phase 2に戻る。
3. **中止** — 投稿せず終了

eventの変更もここで可能にする（例: 「REQUEST_CHANGESじゃなくてCOMMENTで」）。

### Phase 5: GitHub API投稿

```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
  --method POST \
  --input /tmp/pr-${PR_NUM}-review-payload.json
```

payload構造:

```json
{
  "commit_id": "<headRefOid>",
  "event": "COMMENT",
  "body": "<bodyに回したコメントがあればここ。なければ空文字>",
  "comments": [
    {
      "path": "src/components/domain/calls/call-person-in-charge-dialog.tsx",
      "line": 46,
      "side": "RIGHT",
      "body": "(実ファイル: `src/features/calls/validations.ts:101-108`)\n\n[must] 担当NGダイアログの..."
    }
  ]
}
```

### Phase 6: 結果レポート

投稿結果をテーブルで報告:

```
| # | Severity | ファイル | 投稿先 |
|---|----------|----------|--------|
| 1 | [must]   | calls/validations.ts:101-108 | call-person-in-charge-dialog.tsx:46 (リダイレクト) |
| 2 | [should] | field-reconciliation.test.ts:24 | インライン :24 |
| 3 | [nit]    | validations.test.ts:385 | インライン :385 |
| 4 | [should] | use-handle-call-details.ts:430 | レビュー本文 (diff外) |
```

- リダイレクト発生時は「(リダイレクト)」「(diff外)」を明示
- 全件diff内に収まった場合はシンプルなテーブル
- 失敗があればエラー内容を表示しリトライを提案

---

## GitHub PR Review API 制約メモ

| 制約 | 対処 |
|------|------|
| インラインコメントはdiff内ファイルのhunk範囲内のみ | Phase 3で事前検証 |
| diff外ファイル → 422 | import逆引きリダイレクト or body移動 |
| hunk外行番号 → 422 | body移動 |
| commit_idはPRのHEAD必須 | `headRefOid`を使う |

## エラーハンドリング

| エラー | 対応 |
|--------|------|
| 422 Unprocessable Entity | Phase 3検証漏れ → 該当コメントをbodyに回して再投稿 |
| 404 Not Found | commit_id古い or PR不存在 → PR情報再取得 |
| 403 Forbidden | 権限不足 → `gh auth status`確認を案内 |
| パース失敗 | .mdフォーマット不正 → フォーマット例を提示 |
