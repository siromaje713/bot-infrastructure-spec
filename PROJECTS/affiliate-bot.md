# PROJECT: affiliate-bot

## 基本情報

- **cwd**: `~/affiliate-bot`
- **GitHub**: `siromaje713/affiliate-bot` (private)
- **Threadsアカウント**: `@riko_cosme_lab`
- **ペルソナ**: りこ（27歳・敏感肌・プチプラ美容オタク、皮膚科勤務の姉32歳）
- **北極星**: 月50万円自動収益（美容Threads × アフィリエイト）

## Render サービスid

| service_id | 用途 | schedule (UTC) | schedule (JST) |
|---|---|---|---|
| `crn-d72ovqm3jp1c7386q0fg` | affiliate-bot (post) | `0 0,4,8,12 * * *` | 9/13/17/21時 |
| `crn-d741a6q4d50c73bvbavg` | affiliate-bot-reply | `0 2,6,10,14 * * *` | 11/15/19/23時 |
| (TBD) | daily_report | `0 16 * * *` | 1時 |
| (TBD) | healthcheck | `0 * * * *` | 毎時 |

- **Environment Group**: `evg-d75m22chg0os73arufsg`
- **Auto-Deploy**: `yes`（ただし稀に停滞するので INCIDENTS/2026-04-22 参照）

## 現在地（2026-04-22時点）

- Phase 1完全成功: writer.py JSON廃止 + 3 call逐次分割 + ask_plain導入
- 姉シリーズ / 保存型 / engage8型 運用中
- アフィリプ全停止（アカウントパワー育成フェーズ）
- 画像投稿停止（imgur shadowban対策）
- 初動で5分630インプレッション記録

## 環境変数（値は `~/affiliate-bot/.env` から読む）

- `ANTHROPIC_API_KEY` — writer/engage/researcher用
- `THREADS_ACCESS_TOKEN` — riko_cosme_lab投稿用
- `THREADS_USER_ID` — riko_cosme_labの数値ID
- `THREADS_TOKEN_EXPIRES_AT` — 2026-05-30
- `SLACK_WEBHOOK_URL` — 投稿結果通知
- `RENDER_API_KEY` — Render API叩く用
- `BENCHMARK_ACCOUNT_IDS` — エンゲージ対象アカウント（カンマ区切りusername）

## 関連 INCIDENTS

- [2026-04-22 Render deploy停滞](../INCIDENTS/2026-04-22_render_deploy_stuck.md) — Phase 1成功の決定打
- [2026-04-20 SyntaxError本番流出](../INCIDENTS/2026-04-20_syntax_error_prod.md) — コミット前checkの重要性
- [2026-04-15 imgur shadowban](../INCIDENTS/2026-04-15_imgur_shadowban.md) — 画像投稿停止の理由

## 運用スクリプト（`scripts/`）

- `post_with_jitter.sh` — 投稿時刻のjitter付与
- `refresh_threads_token.py` — Threadsトークン更新
- `run_daily_report.py` — 日次レポート生成 → Slack
- `slack_notify.py` — Slack通知ラッパー
- `_render_*.py` / `_inject_*.py` / `_repro_writer_local.py` — 2026-04-22 Phase 1復旧で作成（次セッションで `scripts/render_ops/` へ移動予定）

## 投稿型

**engage（70%抽選）**:
- A知識暴露 / B行動訂正 / C やり方暴露 / H独白本音 / I論争 / K姉ショート

**list（30%抽選）**:
- 姉シリーズ（5割） / 保存型（5割）

## NG行動チェックリスト（過去失敗）

- [ ] SyntaxError本番流出 → push前に `python3 -c "import orchestrator"` + pytest必須
- [ ] imgur画像投稿 → shadowbanリスク・使用禁止
- [ ] Threads API `since` → 400エラー・使用禁止
- [ ] Threads他人アカウントの`like_count` → 取得不可（Playwrightで迂回）
- [ ] GitHub UI直接編集 → ファイル破損実績
- [ ] git push --force / rm -rf / 任意のreset --hard
- [ ] APIキー・トークンをログ / ファイル / チャット出力

## 次セッション保留タスク

詳細は `~/affiliate-bot/HANDOFF.md` 参照。

1. Step E/F/G: benchmark_raw 収集
2. scripts/_render_*.py を scripts/render_ops/ へ移動
3. CLAUDE.md Render API Key 平文削除
4. healthcheck cron実装
5. permalink URL正規形式の調査（/t/{id}は404問題）
