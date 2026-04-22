# INCIDENT: 2026-04-08 Threads API /search endpoint が常に400

## タグ
`#project:affiliate-bot` `#category:api` `#severity:mid`

## 症状
- `GET https://graph.threads.net/v1.0/search?q=<username>&type=USER&fields=id,username&access_token=...`
- **常に HTTP 400** を返す
- username → user_id 解決が全く機能しない
- `agents/insights_analyzer.py:_lookup_user_id` / `agents/engage_agent.py:_lookup_user_id` で呼び出し → 全件失敗

## 真因

**層: 外部API（Threads Graph API の仕様と公式ドキュメントの乖離）**

- Threads Graph API 公式ドキュメントには `/search?type=USER` の記載がある
- しかし実際には**機能せず400を返す死に関数**
- Meta側の実装が documentation に追いついていない（または deprecate 未告知）
- コード内にコメントとして「検索API(/search)は400エラーのため使わない」と明記

## 復旧手順

1. `/search` 経路を完全に諦める
2. `data/dynamic_benchmarks.json` に `username` ↔ `user_id` の**手動マッピング**を保管
3. 初回は Threads公開ページのHTML / 手動確認で user_id を特定して記録
4. 以降、コードは json 参照のみ
5. 対象5アカウント（popo.biyou / km.room / momo_cosme_b / kajierimakeup / ior_coco）を permanent フラグ付きで格納

## 学び

1. **Threads API仕様は実証主義で確認する**（公式ドキュメント盲信禁止）
2. 各API endpoint を使う前に**最小リクエストで疎通テスト**する
3. 死に関数は「使わない」コメントを残しつつコードは削除せず保留（後で復活する可能性）
4. user_id のような**安定した対応関係は JSONに手動格納**のほうが堅牢

## 関連事例

- INCIDENTS/2026-04-08_since_param_400.md — 同じくThreads APIの仕様乖離
- INFRA.md — Threads API の既知制約
- PROJECTS/affiliate-bot.md — dynamic_benchmarks.json の運用

## 再発判定キーワード

- `Threads /search 400`
- `graph.threads.net/v1.0/search`
- `username → user_id 解決失敗`
- `_lookup_user_id 400`
- `公式ドキュメントと実挙動の乖離`
- `死に関数`
