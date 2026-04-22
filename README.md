# bot-infrastructure-spec

Bot自動化プロジェクト（affiliate-bot / hoshi-musubi / 今後追加分）を横断する **運用知識の単一情報源**。

## このrepoの目的

1. **過去事故を二度と繰り返さない** — 障害・誤診断・真因・復旧手順をINCIDENTSに全部残す
2. **新セッションのClaude（任意モデル）が即戦力になる** — `INFRA.md` を最初に読めば全体像が分かる
3. **プロジェクト横断のベストプラクティスを蓄積** — PATTERNSで再利用可能な手順を定型化

## 各ディレクトリの役割

| パス | 用途 |
|---|---|
| `INFRA.md` | インフラ全体地図・認証情報の所在・Renderサービス一覧・障害復旧テンプレ |
| `INCIDENTS/` | 時系列の障害事例DB。1事故=1ファイル。再発防止キーワード付き |
| `PATTERNS/` | 繰り返し使う運用手順（session_startup, render_recovery, claude_roles 等） |
| `PROJECTS/` | プロジェクト別の基本情報・関連INCIDENTS一覧 |

## 使い方（Claude向け）

**新セッション開始時の必須手順**:
1. そのプロジェクトの `CLAUDE.md`
2. そのプロジェクトの `HANDOFF.md`
3. このrepoの `INFRA.md`（URL: `https://raw.githubusercontent.com/siromaje713/bot-infrastructure-spec/main/INFRA.md`）
4. 問題発生時は `INCIDENTS/` で類似事例を検索（grep: 症状キーワード・エラー文字列）

**新しい事故が起きたら**:
1. `INCIDENTS/_template.md` をコピーして `YYYY-MM-DD_<snake_case>.md` で保存
2. タグ・症状・誤診断・真因・復旧手順・学びを埋める
3. 関連プロジェクトの `PROJECTS/*.md` の関連INCIDENTS一覧にもリンク追加
4. PR or 直接commitで反映

## memory_user_edits との役割分担

| 対象 | repo | memory |
|---|---|---|
| 普遍的な運用ルール（全AI共通・Claudeモデル依存しない） | ✅ このrepo | — |
| ユーザー個人の好み（口調・回答スタイル） | — | ✅ user_edits |
| 過去の具体的事故詳細 | ✅ INCIDENTS/ | — |
| Claudeのセッション中に得た一時的観察 | — | ✅ auto memory |
| プロジェクト構造・Renderサービスid 等の安定情報 | ✅ INFRA.md | — |
| 「次に〇〇したい」という未完了タスク | ✅ プロジェクトの HANDOFF.md | — |

## 絶対ルール

- **API Key値・アクセストークンは一切書かない**。`.env` のキー名だけ記載して「値は `.env` から」と明記
- private repo固定（パス情報が含まれるため）
- `force push` / `rm -rf` はこのrepoでは禁止（履歴保全）
- INCIDENTSは追記のみ。過去事故を書き換えない（誤診断経緯も学びとして残す）
