---
name: pr-review
description: |
  GitHub PR を worktree 隔離でレビューする。仕様書 (CLAUDE.md / docs / spec 等) を参照し、
  codex が利用可能なら並列に独立レビューを依頼、両者の指摘を統合一本化。
  最終出力前にファクトチェックとPRスコープチェックの2パスを自己実施。
  トリガー: `/pr-review` （明示起動のみ）
argument-hint: "[URL|#n] [--no-codex] [--base <ref>]"
---

# pr-review

PR を深く読み、仕様書と照合してレビューするグローバルスキル。

## 起動形式

```
/pr-review                                  # 引数なし=現ブランチ
/pr-review https://github.com/o/r/pull/123  # PR URL
/pr-review #123                             # PR 番号
/pr-review 123 --no-codex                   # codex 並列レビューをスキップ
/pr-review #123 --base develop              # base ブランチをオーバーライド
```

引数解析:
- 第1引数が `https://github.com/.../pull/N` / `#N` / `N` のいずれか、または省略
- 省略時は「現ブランチ」モード
- `--no-codex`: codex 並列レビューを無効化（存在しない場合も自動でスキップ）
- `--base <ref>`: PRが報告するbaseを上書き

## 全体フロー（7フェーズ）

```
Phase 1 Setup → Phase 2 並列レビュー → Phase 3 マージ
              → Phase 4 ファクトチェック → Phase 5 スコープチェック
              → Phase 6 出力 → Phase 7 クリーンアップ
```

---

## Phase 1: Setup

### 1.1 PR メタデータ取得

引数が省略された場合は現ブランチモード。それ以外は PR 番号を抽出して `gh` で取得。

```bash
# PR 番号抽出（URL or #N or N）
PR_NUM=$(echo "$ARG" | grep -oE '[0-9]+$' || true)

# 現ブランチモード
if [ -z "$PR_NUM" ]; then
  PR_NUM=$(gh pr view --json number --jq .number 2>/dev/null || echo "")
fi

# 同一リポ確認（cross-repo は非対応）
gh pr view "$PR_NUM" --json url,headRepository,baseRepository
# headRepository.nameWithOwner と baseRepository.nameWithOwner が
# `gh repo view --json nameWithOwner` と一致すること。違えばエラー終了。

# 必要情報をまとめて取得
gh pr view "$PR_NUM" --json \
  number,title,body,headRefName,baseRefName,headRefOid,additions,deletions,changedFiles,url
```

`--base` 指定があればその値で `baseRefName` を上書き。

### 1.2 worktree 作成

```bash
WT=/tmp/pr-review-${PR_NUM}   # 現ブランチモードは /tmp/pr-review-current
git fetch origin "${BASE_REF}" "${HEAD_REF}"
git worktree add --detach "$WT" "${HEAD_REF}"
cd "$WT"
```

既に同名 worktree があれば `git worktree remove --force` してから再作成。

### 1.3 diff のファイル化

```bash
DIFF_FILE=/tmp/pr-${PR_NUM}.diff
gh pr diff "$PR_NUM" > "$DIFF_FILE"
# 現ブランチモードの場合は:
# git diff origin/${BASE_REF}...HEAD > "$DIFF_FILE"
```

### 1.4 package.json 検出時のみ依存セットアップ確認

`$WT` に `package.json` / `Cargo.toml` などが存在する場合のみ、`AskUserQuestion` で
「LSP / 型チェックを使うので依存をインストールするか？」を聞く。Yes なら該当コマンド
（`pnpm install` / `npm install` / `cargo build` 等）を実行。No ならスキップ。

依存ファイルが無ければ何も聞かずに次へ。

---

## Phase 2: 並列レビュー

### 2.1 codex を background で起動（`--no-codex` 未指定 & `which codex` 成功時のみ）

`templates/codex-prompt.md` をベースにプロンプトを組み立て、Bash の `run_in_background: true`
で起動。タイムアウト 600000ms。失敗・タイムアウトは警告ログのみで続行。

```bash
codex exec --full-auto --sandbox read-only --cd "$WT" \
  "$(cat templates/codex-prompt.md | envsubst)" \
  > /tmp/pr-${PR_NUM}-codex.md 2>&1
```

`templates/codex-prompt.md` の中で diff ファイルパス・PR タイトル・PR Description を
埋め込み、出力フォーマット（Severity / File:Line / 問題 / 根拠 / 修正案）を厳密に指示。
末尾に「確認や質問は不要です。具体的な提案まで自主的に出力してください。」を含める。

### 2.2 Claude メインレビュー（codex 並列実行と同時に進める）

順序:

1. **仕様書探索**
   - 最優先: `$WT/CLAUDE.md`, `$WT/AGENTS.md`, `$WT/.cursor/rules/*`
   - fallback glob: `docs/`, `doc/`, `spec/`, `specs/`, `submodule/*/docs/`, `.github/`
     から、変更ファイル名・ディレクトリ名と関連しそうなものを選んで Read
   - CLAUDE.md に「レビュー時はこれを必ず読め」リストがあればそれを優先

2. **PR Description / AC を精読**
   - Phase 1.1 で取得した `body` を読み、AC（Acceptance Criteria / 受け入れ条件）を抽出
   - 「このPRが達成すべきこと」リストを作る

3. **変更ファイル全 Read**
   - `gh pr view "$PR_NUM" --json files --jq '.files[].path'` で一覧
   - 各ファイルを Read（周辺コンテキスト含めて全体）

4. **レビュー観点**
   - コード品質（命名、設計、再利用、エラー処理、セキュリティ）
   - 仕様書との整合性
   - PR Description / AC との照合（未実装 / スコープ外）
   - 変更行のみを指摘対象とする（例外は後述）

5. **指摘ドラフト作成**
   - 各指摘に Severity を付与: `Must` / `Should` / `Nit`
   - フォーマット: `Severity / File:Line / 問題 / 根拠 / 修正案`

### 2.3 codex 完了待ち

Claude レビュー完了後、codex の出力ファイル `/tmp/pr-${PR_NUM}-codex.md` を Read。
未完了ならもう少し待つ。タイムアウトしたら codex 抜きで進む。

---

## Phase 3: マージ

Claude 指摘リストと codex 指摘リストを統合し、一本化:

- 同じ file:line で同趣旨の指摘は **1件にマージ**（内部メモ: 「両者一致」=信頼度高）
- 片方しか挙げていない指摘もそのまま採用
- Severity が割れたら **高い方** を採用（Must > Should > Nit）
- 「両者一致」の Should は Must に格上げ検討

統合後リストを内部状態として保持。**この段階ではユーザーに見せない**。

---

## Phase 4: ファクトチェックパス（Claude 自己）

統合リストの各項目を順に検証:

1. **file:line が diff に実在する**
   - `grep -nE "^\+" "$DIFF_FILE"` でPRが触った行を確認
   - 該当行が diff の追加・変更行に含まれない場合 → 後段のスコープチェックで処理（ここでは触らない）
2. **引用した仕様書記述が実在する**
   - 「CLAUDE.md にこう書いてある」「docs/foo.md にこう書いてある」と主張した箇所を
     再 Read して文字列存在を確認
   - 存在しなければ → **黙って削除**
3. **「存在しない」と主張した関数・型・定数の実在性**
   - 「この関数は呼ばれていない」「この型は定義されていない」と主張した項目を
     `grep -r` で再確認
   - 実在していれば → **黙って削除**
4. **行番号が現在の HEAD のファイル内容と一致する**
   - 該当ファイルを Read して指摘内容と整合するか
   - ずれていれば修正、整合不能なら削除

棄却項目はサイレント削除（理由欄に残さない）。

---

## Phase 5: スコープチェックパス（Claude 自己）

ファクトチェック後の残リストに対し:

1. **指摘の file:line が diff 行を指しているか**
   - Phase 1.3 の `$DIFF_FILE` を参照
   - diff 外行を指している指摘は **黙って削除**
2. **例外: AC 未実装ギャップ**
   - 「PR Description/AC に X とあるが diff に該当変更がない」タイプの指摘は
     diff 外でも残す。ただし「該当の変更が入るべき最寄りファイル」を
     `gh pr view --json files` の一覧から推定してポインタを付与する形に整形
   - 推定不能な場合はファイルパスを付けずに「未実装」セクションへ
3. **Description にない変更（スコープ外変更）の指摘は残す**
   - これは diff 内の指摘なので、スコープチェックでは削除しない。Phase 6 出力時に
     "スコープ外変更" として明示

---

## Phase 6: 出力

`templates/report.md` 構造に沿ってターミナルに Markdown で出力。投稿はしない。

```markdown
# PR Review: <PR title> (#<n>)

**URL**: <url>
**Branch**: <head> ← <base>
**Files**: <changed> | +<add> / -<del>

## TL;DR

<1〜2 行サマリ>

**Must**: N 件 / **Should**: N 件 / **Nit**: N 件
**Verdict**: ✅ Merge 可 / ⚠️ 要修正 / ❌ Block

---

## 指摘一覧

### [Must] <タイトル>

- **File**: `path/to/file.ts:42`
- **問題**: <何が問題か>
- **根拠**: <仕様書引用 or コード参照>
- **修正案**:
  ```ts
  // 具体的なコード
  ```

(以下、Must → Should → Nit の順に並べる)

---

## 仕様未実装ギャップ（AC との差分）

- AC#3「<内容>」が未実装。`path/to/related-file.ts` あたりに追加されるべき

(なければセクションごと省略)

---

## スコープ外変更（Description にない変更）

- `path/to/file.ts:120` の <変更内容> は Description に記載がない

(なければセクションごと省略)

---

## 参照した仕様書

- `CLAUDE.md`
- `docs/foo/README.md`
- ...

## レビュー体制

- Claude: 本体レビュー
- Codex: 並列レビュー（--no-codex でスキップ / unavailable / timeout の場合は明記）
```

---

## Phase 7: クリーンアップ

```bash
cd -  # 元の cwd へ戻る
git worktree remove --force /tmp/pr-review-${PR_NUM}
rm -f "$DIFF_FILE" "/tmp/pr-${PR_NUM}-codex.md"
```

失敗してもユーザーに警告だけ出して終了（残骸が残っても致命的ではない）。

---

## 設計上の注意

- **指摘は diff 内行のみ**（例外: AC 未実装ギャップ）
- **棄却項目は黙って削除**（読みやすさ優先、トレーサビリティ犠牲）
- **Description ↔ diff 両方向のずれを指摘**（仕様未実装 + スコープ外変更）
- **インラインコメント投稿はしない**（ターミナル出力のみ）
- **CLAUDE.md / プロジェクトルールが「重要度分類を使わない」と明示している場合**でも、
  このグローバルスキルは Must/Should/Nit を使う。プロジェクトのレビューフローに
  寄せたい場合は、そのリポジトリ用に `code-review` 等の別スキルを使うこと
- **Lint / type-check は走らせない**（CI 任せ。worktree のセットアップ無しでも動く）
- **クロスリポジトリの PR には非対応**（明示エラー）
