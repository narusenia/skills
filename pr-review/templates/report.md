# PR Review: {{title}} (#{{number}})

**URL**: {{url}}
**Branch**: `{{head_ref}}` ← `{{base_ref}}`
**Files**: {{changed_files}} | +{{additions}} / -{{deletions}}

## TL;DR

{{one_line_summary}}

**Must**: {{must_count}} 件 / **Should**: {{should_count}} 件 / **Nit**: {{nit_count}} 件
**Verdict**: {{verdict_emoji}} {{verdict_text}}

> Verdict ガイドライン:
> - ✅ **Merge 可**: Must 0 件
> - ⚠️ **要修正**: Must が存在するが致命的でない
> - ❌ **Block**: バグ・セキュリティ・仕様違反の Must あり

---

## 指摘一覧

Must → Should → Nit の順。同 Severity 内は file path のアルファベット順。

### [Must] {{title}}

- **File**: `{{path}}:{{line}}`
- **問題**: {{problem}}
- **根拠**: {{rationale}}
- **修正案**:

```{{lang}}
{{suggested_code}}
```

<!-- 指摘が無い Severity のセクションは丸ごと省略する -->

---

## 仕様未実装ギャップ（AC との差分）

PR Description / AC に記載があるが diff に該当変更が見当たらない項目。

- **AC#{{n}}**: 「{{ac_text}}」
  - 想定追加先: `{{nearest_file}}`（推定）
  - 理由: {{why_missing}}

<!-- ギャップが無ければセクションごと省略 -->

---

## スコープ外変更（Description にない変更）

PR Description に記載がないが diff に含まれている変更。レビュアーが意図確認すべき項目。

- `{{path}}:{{line}}` — {{change_summary}}

<!-- 該当が無ければセクションごと省略 -->

---

## 参照した仕様書

- {{spec_path_1}}
- {{spec_path_2}}
...

## レビュー体制

- **Claude**: 本体レビュー
- **Codex**: {{codex_status}}
  <!-- "並列レビュー実施" / "--no-codex でスキップ" / "unavailable" / "timeout" のいずれか -->
