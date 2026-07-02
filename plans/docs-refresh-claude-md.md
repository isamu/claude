# Plan: CLAUDE.md を現行の Claude Code スキル/メモリ機構に合わせて刷新

Issue: https://github.com/isamu/claude/issues/30

## 目的

CLAUDE.md の大半は現役だが、Claude Code のスキル/メモリ機構の登場を反映できていない箇所がある。以下を更新し、現行の運用と噛み合わせる。

## 変更項目

### 1. Web Design & Debugging with Playwright（最優先）
- Playwright MCP 直叩き前提を改め、優先順位を明示する。
- 「まず `/verify` / `run` スキルを使う。UI 回帰は `/pr-ui-test`、性能は `/web-perf`。それらで届かない場合に手動 Playwright MCP（`browser_*`）」という段階に書き換える。
- Playwright MCP が常にロードされている前提の記述を外す。

### 2. Continuous Learning がメモリ機構を知らない（最優先）
- 「学んだこと = CLAUDE.md 追記」だけの前提を改める。
- 使い分けを追記: ルール/ワークフロー → CLAUDE.md、ユーザーの事実・フィードバック・プロジェクト文脈 → ファイルメモリ（`~/.claude/.../memory/` + `MEMORY.md` インデックス）。
- どちらもユーザー確認の上で保存する点は維持。

### 3. Code Quality / Debugging がスキルを参照していない
- Code Quality に「重要な実装後は `/code-review`（必要なら `--fix`/`--comment`）、品質のみのリファクタは `/simplify`」を追記。
- セキュリティ観点が要る変更では `/security-review` を挙げる。

### 4. Testing の `node:test` 固定を緩める
- 「Node ネイティブ `node:test`/`node:assert` を基本とする。ただしプロジェクトに既存のテストランナー（vitest 等）があればそれに従う」に変更。

### 5. PR Bot Review Handling の圧縮
- 詳細手順は `gh-review-loop` スキルに委譲。
- CLAUDE.md 側は「PR へ push 後は `/gh-review-loop` で全 bot（CodeRabbit, Sourcery 等）を triage し、対応/見送りを PR コメントで要約」に短縮。核となる原則（全 bot を確認 / 盲目的に従わない / 対応を要約）は残す。

## 対象ファイル

- `CLAUDE.md`（このリポジトリ = グローバル CLAUDE.md のソース）
- `plans/docs-refresh-claude-md.md`（本ファイル）

## 検証

- ドキュメントのみの変更のためビルド/テストなし。
- 変更後、参照しているスキル名（`/verify`, `run`, `pr-ui-test`, `web-perf`, `/code-review`, `/simplify`, `/security-review`, `/gh-review-loop`）が実在することを確認済み。

## 作業手順

1. feature ブランチ `docs/refresh-claude-md` 作成（済）
2. 本プラン file をコミット
3. CLAUDE.md に 1〜5 を反映しコミット
4. PR 作成（マージはしない）
