# INCIDENT: 2026-04-08 同一投稿への3連続リプライ発火

## タグ
`#project:affiliate-bot` `#category:content` `#severity:mid`

## 症状
- engage_agent が **同じ他人投稿に対して3回連続でリプライ**を飛ばしてしまう
- 相手側から見るとBot丸出し・スパム挙動
- cron発火ごと（1日複数回）に重複発火

## 真因

**層: 運用（Render ephemeral filesystem に永続stateを置いた）**

- 送信済みリプ管理の `engaged_post_ids.json` を `/tmp/` 配下に保存していた
- Renderのcron実行は ephemeral（コンテナごとに独立）で `/tmp/` は毎cron起動時に**リセット**
- → 毎回「送信履歴なし」と判定 → 同じ投稿に再度リプライ
- 複数のcron（post/reply等）が同時期に走ると1日3回ヒット

## 復旧手順

1. 重複防止stateを ephemeral から永続化場所へ移す
2. `data/sent_replies.json` を作成（repo管理対象）
3. `scripts/github_sync.py` 経由で GitHub commit / pull で cron間共有
4. engage_agent起動時に GitHub最新版をpull → local `data/sent_replies.json` 参照 → 送信後 push

## 学び

1. **Render ephemeral storage (`/tmp/`, コンテナFS) に永続state を絶対置かない**
2. 永続化が必要な state は以下のいずれかで管理:
   - GitHub commit経由（軽量state・人間も見たいもの）
   - 外部DB（Firestore/Postgres等）
   - Render Disks（有料オプション）
3. 重複発火防止が必要な機能は **テストで「2回目呼び出しで何も起きない」を確認**

## 関連事例

- INFRA.md — 永続化方針の記載
- PATTERNS/render_recovery.md — Render運用一般

## 再発判定キーワード

- `同じ投稿に複数回アクション`
- `重複リプライ`
- `/tmp/ ファイル`
- `cron跨ぎ state消失`
- `ephemeral filesystem`
- `engaged_post_ids.json`
- `sent_replies.json`
