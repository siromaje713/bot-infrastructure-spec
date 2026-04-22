# INCIDENT: 2026-04-08 engage_agent 閾値過大でリプライ送出ゼロ

## タグ
`#project:affiliate-bot` `#category:content` `#severity:mid`

## 症状
- `agents/engage_agent.py` を稼働させても**1件もリプライが飛ばない**
- `[EngageAgent] リプ対象なし` ログだけが出続ける
- ベンチマークアカウント群には投稿がある

## 真因

**層: 運用（閾値がアカウント規模にミスマッチ）**

- `min_likes_threshold=100` がデフォルト設定（`data/dynamic_benchmarks.json`）
- affiliate-bot がターゲットとする美容ジャンルの**中規模アカウント**では、平均いいね数 20-50 程度
- 100を超える投稿は1日数件・ほぼヒットしない
- → engage対象0件 → エンゲージメント取得機会を大量ロス

## 復旧手順

1. `data/dynamic_benchmarks.json` の `min_likes_threshold` を `100` → `20` に変更
2. または engage_agent.py 内のデフォルト値を下げる
3. 稼働後、1日10件程度の engage が走ることを確認

## 学び

1. **閾値は実測データから決める**（テンプレ値・思い込み禁止）
2. 新機能を稼働させたら**実行回数・結果件数を必ずログに出す**
3. 0件 / N件 / 上限N件の3ケースを想定してテスト
4. アカウント規模（フォロワー・平均いいね）に応じた適正閾値を記録しておく

### 参考: affiliate-bot (riko_cosme_lab 規模) の適正値
- 自分の投稿: 平均いいね 1-3・閾値5以上は事実上0件になる
- ベンチマーク（中規模美容アカウント）: 平均いいね 20-50・閾値 **20** が適切
- 大規模インフルエンサー: 100-1000・閾値100でも機能

## 関連事例

- PROJECTS/affiliate-bot.md — 運用パラメータ
- INFRA.md — 閾値調整の指針

## 再発判定キーワード

- `エージェント不作動`
- `engage 0件`
- `min_likes_threshold`
- `閾値過大`
- `アカウント規模ミスマッチ`
- `リプ対象なし`
