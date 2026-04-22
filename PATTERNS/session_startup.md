# PATTERN: 新セッション開始時の必須手順

Claude Code（任意モデル）で新しい会話を始めたとき、**最初の1メッセージの前に**読み込む/確認する順序。

## 必須読み込み順

1. **プロジェクトの `CLAUDE.md`**
   - 北極星・ペルソナ・現在地・過去の失敗・絶対ルール
   - `~/affiliate-bot/CLAUDE.md` or `~/hoshi-musubi/CLAUDE.md`

2. **プロジェクトの `HANDOFF.md`**
   - 前セッションの引き継ぎ・未完了タスク・最新の完了事項

3. **このrepoの `INFRA.md`**
   - 認証情報の所在・Renderサービスid・プロジェクト横断ルール
   - URL: `https://raw.githubusercontent.com/siromaje713/bot-infrastructure-spec/main/INFRA.md`

4. **問題発生時のみ** `INCIDENTS/` をgrep
   ```bash
   grep -rn "症状キーワード" ~/bot-infrastructure-spec/INCIDENTS/
   ```

## 最初のBashコマンド群（ヘルスチェック）

```bash
cd ~/<project>

# 1. branch / HEAD / origin同期
git status --short
git rev-parse HEAD
git rev-parse origin/main

# 2. stash残留確認
git stash list | head

# 3. untracked/unstaged 確認
git status

# 4. 構文チェック（コード変更するなら必須）
python3 -c "import orchestrator"

# 5. pytest（変更前のベースライン確立）
python3 -m pytest tests/ -v 2>&1 | tail -20
```

## プロジェクト取り違え防止

指示内容と現cwd/CLAUDE.md北極星が食い違っていたら **着手前に停止して確認**。

判定キー:
- cwdに存在しないファイル名が指示内容に含まれる
- ペルソナ名がCLAUDE.mdと不一致（affiliate-bot=りこ / hoshi-musubi=紬 等）
- 北極星・収益モデルが乖離

## タスク遂行中のルール

1. **コミットは明示依頼まで待つ**（勝手にcommit/pushしない）
2. **force push / rm -rf / reset --hard は確認必須**
3. **破壊的操作の前に必ず stashまたは branch 切替**
4. **APIキー値を stdout / Slack / コミットに絶対出さない**
   - Python dotenv経由で環境変数にロード → 値は参照のみ
   - 長さチェック `len(k)` や `bool(k)` で有無確認

## 終了時の記録

- 未完了の大きな作業は `HANDOFF.md` に追記
- 新しい事故が発生したら `INCIDENTS/YYYY-MM-DD_<slug>.md` を作成
- 学びが普遍化できるなら `PATTERNS/` に昇格

## 関連

- INFRA.md — インフラ全体地図
- INCIDENTS/2026-04-20_syntax_error_prod.md — 構文チェック怠ると本番エラー
- PATTERNS/render_recovery.md — deploy停滞時の復旧手順
