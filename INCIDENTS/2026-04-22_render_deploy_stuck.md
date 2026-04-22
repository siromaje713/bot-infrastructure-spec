# INCIDENT: 2026-04-22 Render auto-deploy 停滞によるWriter 7日間停止

## タグ
`#project:affiliate-bot` `#category:deploy` `#severity:high`

## 症状
- 2026-04-15〜2026-04-22 の **7日間、Threads新規投稿がゼロ**
- Slackに毎時間 `🚫 Writer失敗: 投稿スキップ` 通知が連続
- 直近投稿のpermalink（`https://www.threads.net/t/18100556527995035`）は **404**
- riko_cosme_labフォロワー32人・直近リーチ2.6万（既存投稿のみ）

## 誤診断（あれば）

### 誤診断#1: writer.py 内部バグ
- 疑った原因: Phase 1 commit (fd73efe) で writer.py をJSONパース廃止→3 call逐次化した際に**validate_post で全弾reject**されている
- 具体仮説: max_tokens=200 で途中打ち切り / prefill無しで Sonnetが前置き混入 / プロンプトが空応答誘発
- 取った対応:
  1. writer.py / utils/claude_cli.py 熟読
  2. `scripts/_repro_writer_local.py` 作成・ANTHROPIC_API_KEY で 30 API call 再現試験
  3. → **30 call中 27 採用（90%）・3 call全失敗は 0 件**・前置き混入0件で**全仮説反証**
- なぜ違ったか: ローカルでは新writer.py正常動作。**pytestもCI通過**。pytest/ローカル再現とProductionの挙動差分は**コードではなくdeployでしか説明できない**

### 誤診断#2: Threadsサイレント非公開（shadowban仮説H）
- 疑った原因: publish APIは200でpost_id返すが Threads側で非公開化
- 取った対応: `verify_post_exists()` 追加案を diff 提案まで
- なぜ違ったか: そもそも投稿自体が行われていなかった（writer.py の`post_id`はcoder側で生成されていない）

## 真因

**層: deploy（Render auto-deploy 停滞）**

- Phase 1 commit `fd73efe` → `c0b6689`（2026-04-22 08:50頃 JST push）
- GitHub側 `origin/main` は `c0b6689` に更新済み
- **しかし Render側 active deploy は `d04057d9` (2026-04-15 commit) のまま**
- autoDeploy設定は **`yes`** だったにもかかわらず7日間追従せず
- 4/22 JST 17:34 頃に Phase 1 push がトリガとなり **ようやく auto-deploy開始**（`dep-d7k8fucvikkc7380mrq0`）
- その間（4/15〜4/22）、cron実行は**旧JSON版writer.pyで回り続け・毎回Writer失敗通知**を出していた

Slackで見えていた `🚫 Writer失敗: 投稿スキップ` の文言は:
- コミット `3e9e502` で追加された**旧バージョンの通知文**
- Phase 1 commit `fd73efe` で `🚫 Writer3 call全失敗・プロセス異常終了` に書き換えられていた
- つまりSlack文言が古いまま = 古いコードが稼働中 という決定的証拠だった

## 復旧手順

```bash
# 1. 真因特定: git log -S で文言が過去に存在し現在は削除されていることを確認
git log --all -S "Writer失敗: 投稿スキップ" -p | grep -E "^(commit|[+-].*Writer失敗)"

# 2. ローカルとorigin/mainの同期確認
git rev-parse HEAD         # c0b6689
git rev-parse origin/main  # c0b6689 一致

# 3. Render deploy停滞を疑い、空commit pushでauto-deploy誘発
git commit --allow-empty -m "chore: trigger render auto-deploy for c0b6689"
git push origin main   # force pushではない通常push

# 4. RENDER_API_KEY を .env に注入（値はechoせず環境変数経由）
RENDER_API_KEY_INJECT="<value>" python3 scripts/_inject_render_key.py

# 5. Render側状態確認
python3 scripts/_render_check.py
# → deploys[0] が live で commit=c546bf1 を確認

# 6. post cron 手動trigger + polling
python3 scripts/_render_trigger_job.py
# → JOB_ID取得・9-10分で succeeded

# 7. job stdout取得
RENDER_JOB_ID=job-xxx python3 scripts/_render_fetch_stdout.py
# → `Poster 投稿完了: post_id=...` を確認

# 8. ユーザーがThreadsアプリで実物確認（permalink形式は注意）
```

## 結果

- **復旧所要時間**: 真因特定後 30分以内
- **投稿復活**: 2026-04-22 JST 19:41 `post_id=18061464617694785`
- **初動インプレッション**: 5分で **630**（過去最高ペース）
- **Writer失敗**: 0件 / engage writer 3 call 全採用（61字/88字/30字）
- **副次判明**:
  - permalink形式 `/t/{post_id}` は404（別形式が正規）
  - `render.yaml` と実Render側 cron schedule が不一致（実態が正・CLAUDE.md記述と一致）
  - cron jitter (orchestrator.py:440) で 0〜30分ランダム待機が発生

## 学び

1. **「Slack通知文言が現行コードにgrepしてヒットしない」= 古いコードが稼働中の強力な傍証**
   - 即座に `git log -S "通知文言"` で削除タイミングを調べる

2. **autoDeploy=yes でも Render deploy停滞は起こる**
   - 原因は不明（Render側の内部問題と推定）
   - 定期的に `scripts/_render_check.py` で active deploy commit と main HEAD を照合するのが健康診断として有効

3. **pytest 38/38 pass + ローカル再現正常 = 本番も正常、とは限らない**
   - pytest は確率的挙動を捉えない（Sonnet応答の変動）
   - ローカル成功 + 本番失敗 = deploy層の疑いに即切り替える

4. **CLAUDE.mdの「過去の失敗」セクションは最初に読む**
   - 「Threads API: 他人のlike_count取得不可」等、同じ罠を踏まないための備忘録
   - 今回も直前まで Graph API経路の検証に時間を使いかけた

5. **新セッション開始時の優先読み込み**
   - そのプロジェクトの CLAUDE.md → HANDOFF.md → このrepoの INFRA.md → 関連INCIDENTS（grep）の順
   - これを PATTERNS/session_startup.md に固定化

## 関連事例

- INCIDENTS/2026-04-15_imgur_shadowban.md — 別軸の投稿失敗例
- INCIDENTS/2026-04-20_syntax_error_prod.md — 別軸のprod混入事例
- PATTERNS/render_recovery.md — 今回の復旧手順を手順化
- PATTERNS/session_startup.md — 新セッション時の必須読み込み

## 再発判定キーワード（grep用）

- `Writer失敗: 投稿スキップ`
- `Writer3 call全失敗`
- `投稿スキップ`
- `deploy stuck`
- `auto-deploy 効かない`
- `autoDeploy=yes なのに反映されない`
- `古いコード 稼働`
- `Slack通知 文言 grep 0件`
- `pytest pass 本番失敗`
