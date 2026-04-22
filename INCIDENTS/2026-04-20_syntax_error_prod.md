# INCIDENT: 2026-04-20（推定） SyntaxError 本番流出

## タグ
`#project:affiliate-bot` `#category:deploy` `#severity:high`

## 症状
- Python構文エラーを含むコードが `main` にpushされ本番cron起動時にSyntaxErrorで停止
- Slackに `❌ Orchestrator エラー発生: SyntaxError: ...` が通知
- 投稿パイプラインが全停止

## 真因

**層: 運用手順（ローカル動作確認の欠落）**

- ローカルで `python3 -c "import orchestrator"` の最小構文チェックを怠った
- GitHub UI で直接編集 → 構文破損してpush というケースも過去にあり

## 復旧手順

1. 直前commitを特定して revert or fix commit
2. ローカルで再検証:
   ```bash
   python3 -c "import orchestrator"
   python3 -m pytest tests/ -v
   ```
3. 両方通過してから push

## 学び（= CLAUDE.md 絶対ルール化済）

1. **コード変更後は必ず `python3 -c "import orchestrator"` で構文チェック**
2. **GitHub UI直接編集は禁止**（Claude Code経由のみ）
3. **`pytest tests/ -v` が 全件 pass してから push**

## 関連事例

- PATTERNS/session_startup.md — 起動時の必須チェック

## 再発判定キーワード

- `SyntaxError`
- `import orchestrator 失敗`
- `GitHub UI 直接編集`
- `本番 構文エラー`
- `pytest 飛ばし`
