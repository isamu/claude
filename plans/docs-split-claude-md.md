# Plan: CLAUDE.md を分割し docs/ へ抽出、スキルポインタを索引化

Issue: https://github.com/isamu/claude/issues/35

## 目的

CLAUDE.md(299行・毎セッション読み込み)から、特定作業のときだけ要る長い参照材料を docs/ へ抽出し本体を薄くする。常時適用の短いルールは本体に残す。docs/windows-gotchas.md の前例に倣う。

## 変更(ユーザー選択: A〜D + G)

- **A. Testing 後半** → `docs/testing.md`(パターン10種 / golden / ファイル構成 / Designing for testability)
- **B. CI / Cross-Platform** → `docs/cross-platform-ci.md`(windows-gotchas を隣接リンク)
- **C. Debugging 詳細方法論** → `docs/debugging-methodology.md`(bug family / call-site sweep / retry レビュー / 決定的再現 / cross-repo fetch / error-string)
- **D. 手動 Playwright 手順** → `docs/web-debugging.md`
- **G. スキル索引**: 単独1行の New Project Setup / npm Package Release を廃し、`## Skills` 索引に集約(既存の文脈内スキルは相互参照)

各セクションに1行ポインタを残し、ルールは失わない。

## 非対象(ユーザー選択で見送り)

- E(Comments を docs 化): 常時参照したいので本体維持
- F(Bug Fix Workflow の skill 化): 常時効かせたい MUST なので本体維持

## 結果

CLAUDE.md 299 → 224行(-75)。docs/ に4ファイル追加。リンク先実在・移動内容欠落なしを確認。

## 検証

- `wc -l CLAUDE.md` = 224
- CLAUDE.md 内 `docs/*.md` リンク5本すべて実在
- 各 doc に移動内容の代表語が存在(bug family / testability / windows link / browser tools)
