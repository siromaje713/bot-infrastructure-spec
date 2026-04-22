# INCIDENT: 2026-04-15（推定） imgur CDN 画像投稿による shadowban

## タグ
`#project:affiliate-bot` `#category:content` `#severity:mid`

## 症状
- Threads投稿に imgur CDN経由の画像を添付していた時期、**リーチが異常に低下**
- 投稿自体は成功するがインプレッションが伸びない
- `riko_cosme_lab` アカウント全体のリーチに影響

## 真因

**層: Threads 側の配信判定（外部要因）**

- Threads は外部CDN（特にimgur）からの画像埋め込みを**spam・低品質投稿とみなす傾向**
- 画像投稿が走ると shadowban的にリーチが抑制される
- 以降、画像投稿を完全停止しテキストのみ運用に切替

## 復旧手順

1. 画像投稿機能をコードベースから完全停止
2. `agents/poster.py` / `orchestrator.py` で `image_url` を渡す分岐を無効化
3. `media_type=TEXT` 固定で投稿

## 学び

1. **外部CDN画像投稿は Threads spam判定を受けやすい**
2. リーチ異常低下を見たら、**直近で画像投稿を導入していないか確認**
3. 画像を使うなら Amazon公式CDN (`m.media-amazon.com/images/`) 等の信頼性高いソースに限定

## 再発判定キーワード

- `imgur shadowban`
- `外部CDN 画像 Threads`
- `画像投稿 リーチ低下`
- `shadowban`
- `インプレッション 急落`
