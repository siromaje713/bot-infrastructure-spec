# PROJECT: hoshi-musubi

## 基本情報

- **cwd**: `~/hoshi-musubi`
- **GitHub**: `siromaje713/hoshi-musubi` (private・推定)
- **用途**: 占い鑑定（LINE配信 + Stripe決済）
- **ペルソナ**: 紬（占い師・詳細は hoshi-musubi/CLAUDE.md 参照）
- **現状**: **デプロイ準備中**（本番未稼働）

## Renderサービスid

TBD（デプロイ後追記）

- `sync_render_env.yml` GitHub Actionsで環境変数をRenderへ同期する仕組みあり

## 環境変数（値は `~/hoshi-musubi/.env` から読む）

### 共通系（affiliate-botと同一契約想定）
- `ANTHROPIC_API_KEY`

### 占い専用
- `LINE_CHANNEL_ACCESS_TOKEN` / `LINE_CHANNEL_ID` / `LINE_CHANNEL_SECRET`
- `STRIPE_SECRET_KEY` / `STRIPE_WEBHOOK_SECRET`
- `STRIPE_LINK_P1` / `P2` / `P3` / `P5` / `P6`
- `DATABASE_URL`
- `THREADS_ACCESS_TOKEN` / `THREADS_APP_SECRET` / `THREADS_USER_ID`（affiliate-botとは別アカウント）
- `SLACK_WEBHOOK_URL`（affiliate-botとは別channel）

## 重要な境界線

**hoshi-musubi の Threads系環境変数は affiliate-bot で絶対に使わない**:
- `THREADS_ACCESS_TOKEN` — 別アカウントのトークン。流用すると riko_cosme_lab ではなく占い垢に投稿事故
- `THREADS_USER_ID` — 別アカウントID
- `SLACK_WEBHOOK_URL` — 通知channel相違

**共有してよいもの**:
- `ANTHROPIC_API_KEY` — 同一Anthropic契約なら共通
- `RENDER_API_KEY` — Renderアカウント共通（ただし hoshi-musubi/.env に現状未登録）

## 関連 INCIDENTS

現状特記事項なし。デプロイ後、障害発生時に `INCIDENTS/` に追記。

## 次セッション課題

- Renderへの本番デプロイ
- デプロイ後の serviceid を本ファイルに追記
- monitoring体制（affiliate-botで確立したパターンを移植）

## ベストプラクティス移植元

affiliate-botで確立された仕組みは hoshi-musubi でも再利用:
- `scripts/_render_*.py` の運用スクリプト
- Phase 1的 writer.py 設計（ask_plain + 3 call逐次）
- Slack通知 / cron / jitter / healthcheck
