# PATTERN: Render 障害復旧テンプレ

deploy停滞 / cron停止 / 環境変数未反映 等のRender起因障害の復旧手順を定型化。

## 前提条件

- `RENDER_API_KEY` が `.env`（affiliate-bot または hoshi-musubi）にあること
- `scripts/_render_*.py` シリーズが利用可能（affiliate-bot/scripts/ 直下・次セッションで `scripts/render_ops/` へ移動予定）

## Step 1: 健康診断（まず現状把握）

```bash
cd ~/<project>
python3 scripts/_render_check.py
```

出力で確認:
- `autoDeploy=yes` であること
- `suspended=not_suspended` であること
- `deploys[0].status=live` であること
- `deploys[0].commit` が `git rev-parse origin/main` と一致すること

**不一致** → Step 2 へ / **全部OK** → Step 5 へ

## Step 2: deploy停滞判定

```bash
# ローカルとリモートの最新commitを取得
LOCAL_HEAD=$(git rev-parse origin/main)
# Render側の最新deploy commit を _render_check.py の出力から読む
```

Render側commitがLOCAL_HEADより古ければ **deploy停滞**。

## Step 3: 空commit pushでauto-deploy誘発（コスト最小）

```bash
git commit --allow-empty -m "chore: trigger render auto-deploy"
git push origin main   # force pushではない
```

2〜5分待って `scripts/_render_check.py` で live commit が更新されたか確認。

## Step 4: 手動deploy（空commit pushでダメなら）

```python
# Python経由でPOST /v1/services/{id}/deploys
import os, requests, json
from pathlib import Path
from dotenv import load_dotenv
load_dotenv(Path.home()/'<project>'/'.env')
headers = {"Authorization": f"Bearer {os.environ['RENDER_API_KEY']}", "Content-Type": "application/json"}
r = requests.post(
    "https://api.render.com/v1/services/<service-id>/deploys",
    headers=headers,
    data=json.dumps({"clearCache": "do_not_clear"}),
    timeout=30,
)
print(r.status_code, r.json())
```

## Step 5: cron即時trigger（次発火まで待てない場合）

```bash
cd ~/affiliate-bot
python3 scripts/_render_trigger_job.py
```

- `startCommand` は service の `serviceDetails` から自動取得
- poll 5分。jitter のため最大 9分程度かかる
- `succeeded` / `failed` / `canceled` で終了

## Step 6: cron stdout確認

```bash
RENDER_JOB_ID=job-xxx python3 scripts/_render_fetch_stdout.py
```

重要ログ:
- `[Orchestrator] 環境変数:` — 4環境変数OK確認
- `[Writer] engageN: 採用（N字）` — writer成功
- `[Poster] 投稿完了: post_id=...` — 投稿成功
- `[Orchestrator] エンゲージメント投稿完了: post_id` — パイプライン完了

失敗ログ:
- `🚫 Writer3 call全失敗` — writer失敗
- `バリデーション失敗 →` — reject理由
- `ANTHROPIC_API_KEY未設定` — env vars失効
- `Traceback` — 予期せぬ例外

## Step 7: 実物確認（絶対必須）

### post_id から permalink
```
https://www.threads.net/t/{post_id}   ← 404になる場合がある
https://www.threads.net/@{username}/post/{shortcode}   ← 正規形式（調査中）
```

**ユーザーにブラウザで実物確認してもらう**:
- 404でないこと
- 本文が想定通り表示されていること
- インプレッション・いいねが動き始めたこと

Threadsアプリで username プロフから最新投稿を開くのが確実。

## 判定フローチャート

```
Slackに 🚫 Writer失敗通知 連続
        ↓
_render_check.py で live commit 確認
        ↓
   古い? → 空commit push → 2-5分待って再確認
        ↓ 更新されない
   手動deploy
        ↓
   live になった
        ↓
   _render_trigger_job.py で cron即実行
        ↓
   stdout取得 → writer 採用 / 投稿完了を確認
        ↓
   ユーザーがThreadsで実物確認
        ↓
   ✅ 復旧完了
```

## 関連

- INCIDENTS/2026-04-22_render_deploy_stuck.md — 本パターン制定のきっかけ
- INFRA.md — Renderサービスid一覧
- PATTERNS/session_startup.md — 起動時の初期チェック
