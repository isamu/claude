# Plan: skills dir 総点検 + セッションマイニングからのスキル新規作成

Issue: https://github.com/isamu/claude/issues/32

## 目的

1. skills/ 配下の全スキルをレビューし、改善点をすべて修正する。
2. 過去30日の対話セッションを分析し、繰り返しワークフローをスキル化する。
3. 汎用スキルは git 登録、プロジェクト固有スキルは untracked のまま置く。

## 対象データ

- スキル: skills/ 配下 27 個(うち 2 個は外部リポジトリへの symlink — レビューのみ、編集しない)
- セッション: ~/.claude/projects/ の過去30日 transcript から抽出したユーザー発言ダイジェスト約 2MB
  - 除外: wf_*(workflow サブエージェント)、subagents/、GAIA ベンチマーク自動実行(チャットではないため)

## 実行構成

並列エージェント8体:
- スキルレビュー 3 バッチ(9+9+9): frontmatter / 古いモデルID・CLI / 死にパス / CLAUDE.md との矛盾 / スキル間重複
- セッションマイニング 5 体(mulmoclaude 系 4 + その他全プロジェクト 1): 3回以上の繰り返し依頼、繰り返し修正指示、手動マルチステップ手順を抽出

## 修正・作成の方針

- スキル修正: レビュー結果を親セッションで triage し、false positive を除いて全部修正。symlink 先(外部リポジトリ)は報告のみ。
- 新スキル: 候補ごとに 汎用(git 登録) / プロジェクト固有(untracked) を判定。
  - 汎用 → skills/<name>/SKILL.md としてコミット
  - プロジェクト固有 → 対象プロジェクトの .claude/skills/ または skills/ に untracked で配置
- 既存スキルでカバーできるのに手動でやっていたケース → スキルの trigger(description)を改善する。

## ブランチ / PR

- `feat/skills-refresh`(docs/refresh-claude-md の上に stacked。~/.claude/CLAUDE.md が repo に symlink されており、main に戻すと生きているグローバル設定が巻き戻るため)
- PR は #31 マージ後に main へ。未コミットだった skills/publish/SKILL.md の registry 固定改善も同梱。

## 検証

- 修正後、各 SKILL.md の frontmatter が valid YAML であること、参照パスが実在することを機械的にチェック。
- 新スキルは name/dir 一致・description に trigger 記述があることを確認。
