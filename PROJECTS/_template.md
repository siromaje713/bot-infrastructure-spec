# PROJECT: <project_name>

## 基本情報

- **cwd**: `~/<project_name>`
- **GitHub**: `<org>/<repo>` (private/public)
- **対象プラットフォーム**: (Threads / X / LINE / Instagram 等)
- **アカウント / ID**: `@<handle>` or ID
- **ペルソナ**: (名前・年齢・設定)
- **北極星 / KPI**: (収益目標 / エンゲージメント目標 等)
- **現状フェーズ**: (立ち上げ / 育成 / 運用 / 収益化)

## Render サービスid

| service_id | 用途 | schedule (UTC) | schedule (JST) |
|---|---|---|---|
| `crn-xxxx` | (例: post cron) | `0 0 * * *` | 9:00 |

- **Environment Group**: `evg-xxxx`
- **Auto-Deploy**: yes/no
- **Branch tracked**: main

## 環境変数（値は `~/<project_name>/.env` から読む）

### 共通系
- `ANTHROPIC_API_KEY`
- `RENDER_API_KEY`
- `SLACK_WEBHOOK_URL`

### プロジェクト固有
- (APIキー・トークン名を列挙)

## 他プロジェクトとのキー流用ルール

| キー | 流用可否 | 理由 |
|---|:-:|---|
| `ANTHROPIC_API_KEY` | ? | 同一契約なら可 |
| `THREADS_*` | 🚫 | 別アカウント |
| `SLACK_WEBHOOK_URL` | ⚠️ | channel別 |

## 投稿型 / 機能フロー（該当する場合）

- 型1: 説明
- 型2: 説明

## NG行動チェックリスト（過去失敗の固有パターン）

- [ ] （プロジェクト固有の失敗パターン）
- [ ] （既知の地雷）

## 関連 INCIDENTS

- [YYYY-MM-DD xxx](../INCIDENTS/YYYY-MM-DD_xxx.md)

## 運用スクリプト

`scripts/` 配下の主要スクリプト:
- `script1.py` — 役割
- `script2.sh` — 役割

## 次セッション課題 / 未完了タスク

- [ ] タスク1
- [ ] タスク2

## ベストプラクティス参照元

- affiliate-bot / hoshi-musubi で確立された仕組みのどれを再利用するか
