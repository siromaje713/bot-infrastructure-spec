# PATTERN: Claude.ai / Claude Code / Chrome の役割分担

Bot自動化プロジェクトで使う3つのツール/環境を **何にどこまで使うか** を明確化。

## Claude.ai (Web / Desktop app)

### 適した用途
- **高レベル戦略相談**: 「次に何をすべきか」「このアーキテクチャで良いか」
- **大きな設計判断**: API選定・ペルソナ設計・投稿方針
- **コードレビュー**: 既存コードのセキュリティ/パフォーマンス観点の指摘
- **調査系タスク**: Web検索・一般知識の活用

### 避けるべき
- ローカルファイル読み書き（Claude Codeに委譲）
- Bashコマンド実行
- gitの実作業

### トークン残留リスク
- 会話ログに値が残る。**API Key / アクセストークンは絶対に貼らない**

## Claude Code (CLI)

### 適した用途
- **リポジトリ内の実作業**: ファイル編集・構文チェック・pytest実行
- **git操作**: commit/push/stash（破壊的操作は要確認）
- **API叩き**: curl/requests経由の実行・結果保存
- **スクリプト実行**: `scripts/*.py` の起動・ログ収集
- **複数ファイル跨ぎの grep / refactor**

### 制約事項
- `.env`系ファイルは Read/Writeツールで拒否されることがある（権限ポリシー）
  - → **Pythonサブプロセス経由** (`dotenv_values`) で読むのが確実
- 長時間実行プロセスは `run_in_background` で起動
- 破壊的操作（force push / rm -rf）は明示承認必須

### 絶対ルール
- `echo $SECRET` 禁止
- 値を `description` 欄に書かない
- 件数・長さ・マーカー（`loaded: True len: 32`）のみstdout出力

## Chrome (ユーザー操作)

### 適した用途
- **Renderダッシュボード**: Events / Logs 目視確認・Manual Deploy・Settings変更
- **Threadsアプリ（Web）**: 投稿実表示確認・shadowban判定・エンゲージメント確認
- **OAuth認証フロー**: ブラウザ必須の認証（Meta / Google 等）
- **各種Webhook設定UI**: Slack / GitHub Apps等の設定

### Claudeから移譲するケース
- Claude Code で API経由実行できない場合（認証トークンが無い等）
- Manual操作が必要な設定（UI依存の設定）
- **404やShadowban の目視確認**: ブラウザでの表示は人間のブラウザで確認

## 典型的な作業フロー

```
[Claude.ai]         [Claude Code]          [Chrome]
 戦略決定       →     実装・commit      →   push後の本番確認
 方針相談              pytest実行             Threads表示
 設計レビュー          ログ取得                Renderダッシュボード
```

## 情報の受け渡しルール

### Claude.ai → Claude Code
- 「このファイルの〇〇を修正して」とClaude Codeに指示する形
- Claude.aiで貼られたコードは Claude Code側で Read→Edit の流れで反映

### Claude Code → ユーザー (→ Chrome)
- permalink / deploy id / PRリンク 等をユーザーに報告
- ユーザーがブラウザで確認 → 結果を Claude Code にフィードバック

### Chrome → Claude Code
- ダッシュボード画面のスクリーンショット or 数値の文字起こし
- 値を直接コピペする場合は機密情報に注意

## 関連

- INFRA.md — 認証情報の所在
- PATTERNS/session_startup.md — 新セッション時の読み込み順
