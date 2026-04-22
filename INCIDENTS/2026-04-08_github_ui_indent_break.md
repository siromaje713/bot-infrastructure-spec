# INCIDENT: 2026-04-08 GitHub UI 編集で Python コード破損

## タグ
`#project:affiliate-bot` `#category:deploy` `#severity:high`

## 症状
- GitHub Web UIのエディタで `.py` を編集・commit
- push後 Render cron 起動で `IndentationError` / `SyntaxError`
- ローカルで動いていたコードがブラウザ編集後だけ壊れる

## 真因

**層: ツール（ブラウザエディタのインデント保全性欠落）**

- GitHub Web UI は内部で **CodeMirror 6** を使用
- タブ/スペース混在の保全・末尾空白保全・改行コード保全に**問題がある**
- 特にPythonの**インデントがスペース/タブで揺れる**と全行ズレ
- ペースト時に改行コードが `\r\n` → `\n` への自動変換で壊れることもある

## 復旧手順

1. 該当commitを `git revert` で戻す
2. ローカルで editor（VSCode / Vim 等）を使って再編集
3. `python3 -c "import <module>"` で構文チェック
4. `pytest tests/ -v` 通過確認
5. Claude Code CLI 経由で push

## 学び（= CLAUDE.md 絶対ルール化済）

1. **GitHub UI での `.py` 直接編集禁止**
2. 既存ファイル編集は **Claude Code CLI or ローカルエディタ経由のみ**
3. ブラウザベースのコードエディタは小規模な文字修正のみに限定（コードは対象外）
4. mdファイルはUI編集OK（ただし本番コードと併せて編集しない）

## 関連事例

- INCIDENTS/2026-04-20_syntax_error_prod.md — 別のSyntaxError事例（原因は類似）
- PATTERNS/session_startup.md — 構文チェックの必須化

## 再発判定キーワード

- `GitHub UI 編集後 IndentationError`
- `CodeMirror インデント破壊`
- `ブラウザエディタ タブ スペース`
- `本番 IndentationError`
- `直接編集 NG`
