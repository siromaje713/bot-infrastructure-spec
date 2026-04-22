# Bot Infrastructure Status

手動更新ファイル。次スレ起動時に Claude.ai が raw fetch で最新状態確認する用。

最終更新: 2026-04-23 JST
更新者: Claude.ai（スレ "hoshi-musubi cron 修復・Cowork 丸投げ方針確定"）

---

## hoshi-musubi（占い bot）

### HEAD SHA（本ファイル更新時点）
`e865497` fix: remove backslash escape in workflow expressions (engage/post/insights)

### cron 基盤
**GitHub Actions**（Render cron ではない・美容と設計違い）
- `.github/workflows/post.yml`: JST 09:00 / 12:00 / 15:00 / 19:00
- `.github/workflows/engage.yml`: 別スケジュール
- `.github/workflows/insights.yml`: 別スケジュール

Render は web service `srv-d7brc0dm5p6s73f4eobg` のみ稼働、cron 不在。
`render.yaml` の cron 4 本定義（daily_post/engage/insights/daily_report）は
Blueprint 未連携で死蔵状態。連携は不要不急・後回し可。

### 直近の重要変化
- 2026-04-23 01:32 JST: commit `e865497` push
  → engage.yml / post.yml / insights.yml のバックスラッシュバグ修正（16 箇所置換）
  → 次 cron 発火から投稿復活見込み・検証は次スレ冒頭タスク

### 4/19 投稿停止の真因
`\${{ secrets.XXX }}` のバックスラッシュで secrets 展開失敗 → Threads API 401 → 全投稿失敗。
他の原因（Render Blueprint 未連携・コードバグ）は無関係。

### 未完タスク
1. **workflow 未修正 4 ファイル**のバックスラッシュ置換
   - apply_research.yml / auto_research.yml / sync_render_env.yml / token_reminder.yml
2. **safety_filter.py 実装**（景表法 NG ブロック・「確実」「絶対」「必ず当たる」「100%的中」「効果保証」）
3. **ペルソナ確定**（Cowork 市場リサーチ完了後に 12 次元分析で決定）
4. **Threads bio 書き換え**（ペルソナ確定後）
5. **投稿型剪定**（12 種 → 5 種・情緒散文系無効化・分布再調整）

### 進行中
- **Cowork 丸投げ市場リサーチ**（60-75 アカウント・3 バッチ）
- 成果物: `logs/market_research/2026-04-23_*.json`
- 仕様書: `docs/market_research_spec_2026-04-23.md`

### Phase 1/2 状態
- Phase 1 コード実装済み・cron 修正後に稼働再開見込み
- Phase 2 コード準備完了・無効化状態（LINE_STEP_ENABLED=false / STRIPE_ENABLED=false）

---

## affiliate-bot（美容 bot）

### HEAD SHA
`425c4e2`（2026-04-22 時点・本ファイル更新時点で変動の可能性あり）

### cron 基盤
**Render cron 5 本一本化**
GitHub Actions はメンテナンス系のみ（research/ci/reminder/scrape_benchmark/weekly_insights）
post.yml は存在しない。

### 稼働状態
通常稼働中。

### 逆移植状況（2026-04-17 完了）
keyword_manager・buzz_researcher keyword_search 経路・engage 優先順変更・
dynamic_distribution・短文 50%・Slack 日次レポート。

---

## 重要な設計差

| 項目 | hoshi-musubi（占い） | affiliate-bot（美容） |
|---|---|---|
| cron 基盤 | GitHub Actions cron | Render cron |
| post.yml | 存在・schedule 定義 | 不在 |
| render.yaml cron 定義 | 死蔵（Blueprint 未連携） | 本番稼働中 |
| 逆移植方針 | 「今動くものを活かす」優先 | ー |

設計を揃える作業は不要不急・後回し。

---

## 次スレ起動時の必須アクション（両 bot 共通プロトコル）

1. userMemories 全件読了
2. **この STATUS.md を raw fetch で取得**
3. 両 repo の HEAD SHA 確認（raw fetch で `git/refs/heads/main`）
4. fortune/beauty に生存確認指示:
   - `git log -5`
   - 占い: `gh run list --workflow=post.yml --limit 10`
   - 美容: `gh run list --limit 10`
   - Threads API で直近投稿確認（両 bot）
   - console.anthropic.com/settings/billing 残高確認
5. 結果と memory/STATUS.md の差分があれば memory 更新
6. ユーザーに優先度付きタスク提示、指示待ち

---

## 4/17 事件の教訓
Anthropic API クレジット切れで両 bot 同時停止。
原因: 伝説のAI使い組織（s_suzuki@siu-inc.com）で作った API キー 1 本を両 bot が共用。
Auto-reload OFF 運用継続中。セッション冒頭に残高確認必須。
全 Actions failed なら最優先でクレジット残高を疑う。

## 4/23 事件の教訓
fortune/Claude Code/別 Claude.ai の「完了報告」を独立検証せず次に進むな。
Claude.ai の web_fetch は GitHub raw URL の CDN キャッシュで最大数時間旧データを返す。
Claude.ai 同士で食い違い時はユーザーのブラウザ目視で決着。
