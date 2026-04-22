# INFRA.md — Bot自動化インフラ全体地図

最終更新: 2026-04-22

## 1. プロジェクト一覧

| プロジェクト | cwd | 用途 | Threadsアカウント |
|---|---|---|---|
| affiliate-bot | `~/affiliate-bot` | 美容アフィリエイト Threads投稿 | @riko_cosme_lab |
| hoshi-musubi | `~/hoshi-musubi` | 占い鑑定 LINE/Stripe | - |

## 2. 認証情報の所在（値は一切書かない）

### ローカル `.env` の設置パス

| パス | 役割 |
|---|---|
| `~/affiliate-bot/.env` | affiliate-bot用。ANTHROPIC_API_KEY / THREADS_*系 |
| `~/hoshi-musubi/.env` | hoshi-musubi用。LINE系 / STRIPE系 / THREADS_*系（別アカウント） |

### Renderの環境変数グループ

| env group id | プロジェクト |
|---|---|
| `evg-d75m22chg0os73arufsg` | affiliate-bot |

### 各プロジェクト .env に期待されるキー（値は各.envに）

**共通系**:
- `ANTHROPIC_API_KEY` — Anthropic API（両プロジェクト同一キー可）
- `RENDER_API_KEY` — Render API（全プロジェクト共有可）
- `SLACK_WEBHOOK_URL` — プロジェクト別webhook

**affiliate-bot専用**:
- `THREADS_ACCESS_TOKEN` — riko_cosme_lab用（長期トークン）
- `THREADS_USER_ID` — riko_cosme_lab 数値ID
- `THREADS_TOKEN_EXPIRES_AT` — 期限日（2026-05-30現在）
- `BENCHMARK_ACCOUNT_IDS` — カンマ区切りusername

**hoshi-musubi専用**:
- `LINE_CHANNEL_ACCESS_TOKEN` / `LINE_CHANNEL_ID` / `LINE_CHANNEL_SECRET`
- `STRIPE_SECRET_KEY` / `STRIPE_WEBHOOK_SECRET` / `STRIPE_LINK_P1..P6`
- `DATABASE_URL`
- `THREADS_ACCESS_TOKEN` / `THREADS_APP_SECRET` / `THREADS_USER_ID`（別アカウント）

### プロジェクト境界越え流用ルール

| キー | 流用可否 |
|---|---|
| `ANTHROPIC_API_KEY` | ✅ 同一Anthropic契約なら両者共通 |
| `RENDER_API_KEY` | ✅ Renderアカウント共通 |
| `THREADS_ACCESS_TOKEN` / `THREADS_USER_ID` | 🚫 **絶対流用禁止**。別アカウント混同で投稿事故 |
| `SLACK_WEBHOOK_URL` | ⚠️ 通知先が別channel。注意して運用 |
| `LINE_*` / `STRIPE_*` / `DATABASE_URL` | 🚫 hoshi-musubi専用 |

## 3. Renderサービス一覧

### affiliate-bot

| service_id | 名称 | type | schedule (UTC) | JST換算 |
|---|---|---|---|---|
| `crn-d72ovqm3jp1c7386q0fg` | affiliate-bot (post) | cron | `0 0,4,8,12 * * *` | 9/13/17/21時 |
| `crn-d741a6q4d50c73bvbavg` | affiliate-bot-reply | cron | `0 2,6,10,14 * * *` | 11/15/19/23時 |
| (daily_report) | affiliate-bot-daily-report | cron | `0 16 * * *` | 1時 |
| (healthcheck) | affiliate-bot-healthcheck | cron | `0 * * * *` | 毎時 |

### hoshi-musubi

TBD（デプロイ準備中）

## 4. 障害復旧テンプレ（詳細は PATTERNS/render_recovery.md）

### 症状: Slackに Writer失敗 / 投稿停止 通知が連続
**第一候補**: Render deploy 停滞（auto-deploy有効でも稀に発生）
1. `python3 scripts/_render_check.py` で direct 確認（最新commit が live か）
2. 古ければ空commit pushで auto-deploy誘発
3. 効かなければ `POST /v1/services/{id}/deploys` で手動deploy
4. `POST /v1/services/{id}/jobs` でcron即実行テスト
5. Threads実物で404でないこと確認（permalink形式要注意）

### 症状: ANTHROPIC_API_KEY未設定通知
→ Render Environment Group `evg-d75m22chg0os73arufsg` のenv varsを確認
→ 失効疑いなら `GET /v1/env-groups/evg-.../env-vars` で現在値取得（値でなく長さで検証）

### 症状: Threads投稿失敗（API 400/401）
→ `THREADS_TOKEN_EXPIRES_AT` を確認。期限7日前に `scripts/refresh_threads_token.py` で更新

## 5. `scripts/_render_*.py` の使い方（affiliate-bot/scripts/）

**今回のPhase1復旧で作成した運用スクリプト**（次セッションで `scripts/render_ops/` へ移動予定）:

| script | 役割 | env必要 |
|---|---|---|
| `_inject_render_key.py` | 環境変数から `.env` へ RENDER_API_KEY 注入 | `RENDER_API_KEY_INJECT` |
| `_inject_anthropic_key.py` | hoshi-musubi/.env → affiliate-bot/.env へ ANTHROPIC_API_KEY 注入 | — |
| `_render_check.py` | Renderサービス情報 + 直近5 deploys 一覧 | RENDER_API_KEY |
| `_render_trigger_job.py` | post cron 手動trigger + polling | RENDER_API_KEY |
| `_render_poll_job.py` | 既存jobの追加polling | RENDER_API_KEY, RENDER_JOB_ID |
| `_render_fetch_stdout.py` | job stdout/stderrテキスト取得 | RENDER_API_KEY, RENDER_JOB_ID |
| `_fetch_render_logs.py` | service logs一括取得 | RENDER_API_KEY |
| `_list_hoshi_keys.py` | hoshi-musubi/.env のキー名列挙（値は出さない） | — |
| `_repro_writer_local.py` | writer.py ローカル再現（5 trials × list/engage） | ANTHROPIC_API_KEY |

## 6. Claude.ai / Claude Code / Chromeの役割分担

詳細は `PATTERNS/claude_roles.md`。要点のみ:

| ロール | 役割 |
|---|---|
| Claude.ai (Web) | 高レベル戦略・相談・コードレビュー |
| Claude Code (CLI) | リポジトリ内の実作業・ファイル編集・コマンド実行 |
| Chrome (ユーザー操作) | Renderダッシュボード・Threadsアプリ・OAuth認証フロー |

## 7. 絶対ルール（全プロジェクト共通）

1. `force push` / `rm -rf` / `git reset --hard` は確認無しで実行しない
2. トークン・APIキー値を Slack / コミット / チャット / ログ に出さない
3. `.env` は Read/Write権限ポリシーで Claude Codeから直接読めない場合がある → Pythonサブプロセス経由で `dotenv` を使う
4. Python 3.9互換（3.10+構文禁止）
5. imgur CDN 画像投稿は shadowban リスクで禁止
6. Threads API `since` パラメータは 400エラー・使用禁止
7. Threads API で他人アカウントの `like_count` は取得不可（Playwrightスクレイプで迂回）
8. CLAUDE.md / HANDOFF.md は Claude Code経由でのみ編集（GitHub UI直編集はファイル破損実績あり）

## 8. permalink形式の注意（2026-04-22判明）

Threads公開URL形式:
- ❌ `https://www.threads.net/t/{post_id}` → **404**（試行で判明、例: 18061464617694785）
- ✅ `https://www.threads.net/@{username}/post/{shortcode}` が正規？

`post_id` → `shortcode` 変換方法は今後の調査課題（INCIDENTS/2026-04-22参照）。
