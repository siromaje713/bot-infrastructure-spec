# INCIDENT: 2026-04-22 Render API Key 平文が public repo の git 履歴に残存

## タグ
`#project:affiliate-bot` `#category:auth` `#severity:high` `#secret-leak` `#git-history-blob` `#public-repo` `#key-rotation`

## 症状

- `affiliate-bot/CLAUDE.md` に Render API Key 平文 `rnd_EkjoD9DODsbQNf0VIrj0zfN4wkVh` が 2026-04-01(commit `0a86d30`) 〜 2026-04-15(commit `f079d0f`) の期間記載
- `siromaje713/affiliate-bot` は public repo のため、削除 commit 以降も git 履歴 blob に永久保存
- `git log -p -- CLAUDE.md` および `https://github.com/siromaje713/affiliate-bot/blob/0a86d30/CLAUDE.md` で誰でも取得可能な状態
- 影響: Render アカウント全体への読み書き権限流出（旧Keyは "affiliate-bot-copy" 名で rotation 直前まで有効）

## タイムライン

- **2026-04-01** `0a86d30` "docs: CLAUDE.md完全更新（2026-04-01作業引き継ぎ版）" で CLAUDE.md に平文 Key 追加
- **2026-04-15** `f079d0f` "feat: 指示書v2一括実装" で CLAUDE.md から削除。**ただし git blob は残存**
- **2026-04-22 20時頃** `bot-infrastructure-spec` public 化作業中、Claude.ai (web_fetch経由) が affiliate-bot/CLAUDE.md の raw を見て「まだ main HEAD に Key が残存」と誤報告
- **2026-04-22 22時頃** Claude Code の独立検証で事実確認:
  - `git grep -c` で working tree/HEAD/全 tracked+untracked ファイルから検索 → 0 ヒット
  - `git log -S "<key>"` で `0a86d30` 追加 / `f079d0f` 削除 を特定
  - 結論: **main HEAD は clean、git 履歴 blob のみ残存**
- **2026-04-22 23時頃** Render ダッシュボードで旧Key 確認 → `affiliate-bot-copy` 名で使用中と判明（rotation 直前まで 200 OK）
- **2026-04-22 深夜** Key rotation 実施:
  1. Render ダッシュボードで新Key 発行
  2. `~/affiliate-bot/.env` と `~/hoshi-musubi/.env` に新Key 追記
  3. `awk 'NR != N'` で両 .env の旧Key 行削除（line 5 / line 30）
  4. 旧Key を Render ダッシュボードで revoke
- **2026-04-23** 動作確認: `python3 scripts/_render_keytest.py` で 新Key=200 / 旧Key=401 を確認

## 誤診断

### 誤診断 #1: 2026-04-15 の `f079d0f` で「解決済み」と認識していた

- 疑った前提: CLAUDE.md から Key を削除する commit を push すれば漏洩は止まる
- 取った対応: 削除 commit のみで完結、rotation は未実施
- なぜ違ったか: **public repo では一度 commit した secret は永久に取得可能**。GitHub は blob を GC せず、過去の commit SHA を URL に含めて raw fetch できる。削除 commit は "current HEAD を clean にする" だけで、漏洩は止まらない

### 誤診断 #2: 2026-04-22 Claude.ai が「main HEAD に残存」と誤報告

- 疑った前提: Claude.ai が raw.githubusercontent.com/.../main/CLAUDE.md を fetch した結果として「Key が見える」と報告
- 取った対応: main HEAD clean 化を優先するフローを組みかけた
- なぜ違ったか: 実際には main HEAD は既に clean (`700173a`)。Claude.ai が見たのは過去 commit 版（おそらく `0a86d30` 付近の SHA URL か、別の履歴ブラウズ経路）。**fetch URL の branch/SHA 指定を確認せず「現在も残存」と結論づけた**

## 真因

**層: 運用（ドキュメント記述ルール）+ 外部要因（public repo の git blob 不削除性）**

- 真の原因は **secret を CLAUDE.md に書いた** こと自体。`.env` に留めていれば起こらなかった
- public repo では **削除 commit は無効**、rotation（Key 無効化）のみが実効性のある対処

## 復旧手順

```bash
# 1. 漏洩 Key の特定（値は stdout に出さない）
cd ~/affiliate-bot
git log --all --pretty=format:"%h %ad %s" --date=short -S "<compromised_key>" -- CLAUDE.md
# → 追加 commit (0a86d30) と削除 commit (f079d0f) を特定

# 2. 現在の working tree / HEAD / 全 file に残存していないか確認
git grep -c "<compromised_key>" HEAD -- CLAUDE.md
grep -rl "<compromised_key>" . --exclude-dir=.git
# → すべて 0 を確認

# 3. Render ダッシュボードで新Key 発行
#    https://dashboard.render.com/u/settings/api-keys → "Create API Key"

# 4. 新Key を両 .env に反映（ユーザー側で追記、値は stdout に出さない）

# 5. 重複している旧Key 行を削除（行番号指定が安全）
awk 'NR != 5' ~/affiliate-bot/.env > ~/affiliate-bot/.env.tmp && mv ~/affiliate-bot/.env.tmp ~/affiliate-bot/.env
awk 'NR != 30' ~/hoshi-musubi/.env > ~/hoshi-musubi/.env.tmp && mv ~/hoshi-musubi/.env.tmp ~/hoshi-musubi/.env
chmod 600 ~/affiliate-bot/.env ~/hoshi-musubi/.env

# 6. Render ダッシュボードで旧Key を revoke

# 7. 動作確認（新Key 200 / 旧Key 401）
python3 scripts/_render_keytest.py
```

## 結果

- **復旧所要時間**: 鍵流出判明 → rotation 完了まで約 6 時間（ユーザー手動の rotation 判断含む）
- **暴露期間**: 2026-04-01 〜 2026-04-22 深夜 の約 21 日間（public repo 公開期間のうち Key が git blob に存在していた期間）
- **金銭影響**: 確認範囲で不正使用痕跡なし（Render ダッシュボードのログで確認）
- **副次判明**:
  - `bot-infrastructure-spec/PROJECTS/affiliate-bot.md` の visibility 記述が `(private)` のまま古い → 同 commit で `(public)` に訂正
  - `awk 'NR != N'` は行番号指定削除として安全（grep -v で secret pattern を使うと権限ブロックに引っかかるケースあり）

## 学び

1. **secret は `.env` のみが正**
   - CLAUDE.md / README / docs / コミットメッセージ / Slack / チャット / ログに絶対書かない
   - 今回は「引き継ぎしやすくするため」CLAUDE.md に書いたが、完全に誤った判断

2. **public repo では削除 commit は無効、rotation のみ有効**
   - `git filter-branch` や `BFG Repo-Cleaner` で履歴書き換えは技術的に可能だが、force push が必要で他 clone/fork の cache には残る
   - 実効性のある唯一の対処は Key を **無効化（revoke）** すること

3. **Claude.ai の fetch 結果も検証必須**
   - raw URL の branch/SHA 指定を確認せずに「現在も残存」と結論づけない
   - `git log -S "<value>"` と `git grep -c "<value>" HEAD` で独立検証する

4. **Claude Code の「削除済み」自己申告も独立検証必須**
   - 自己申告だけで解決済み扱いにせず、raw fetch + git log + git grep のクロスチェックを必ず実行する

5. **新Key 発行時のブラウザ画面に値が表示される → AI には見せない運用**
   - 今回は画面経由でユーザーが一度見たが、AI（Claude Code）には .env 経由のみで渡し、memory / ログ / コミット / チャットには記録しなかった
   - 将来 rotation 時も同じ運用を徹底する

## 関連事例

- [INCIDENTS/2026-04-22_render_deploy_stuck.md](2026-04-22_render_deploy_stuck.md) — 同日 Render 関連の別事例
- [INCIDENTS/2026-04-20_syntax_error_prod.md](2026-04-20_syntax_error_prod.md) — 本番に誤ったコード/情報が残る類型
- [INFRA.md](../INFRA.md) §2「認証情報の所在」 — 今後 secret を書く場所を `.env` のみに限定する運用ルール

## 再発判定キーワード（grep用）

- `CLAUDE.mdに平文Key` / `CLAUDE.md に API Key`
- `public repoで過去commit残存` / `git blob 削除不能`
- `削除済みと自己申告` / `削除 commit で解決扱い`
- `main HEADと過去commit混同` / `raw.githubusercontent.com SHA指定`
- `rotation 未実施` / `Key revoke 忘れ`
- `secret-leak` `git-history-blob` `public-repo` `key-rotation`
- `Stop hook CLAUDE.md 自動再生成` / `update_claude_md.py` / `LLM 自動書き換え`

## 追加対応（2026-04-23）

- 根本原因の派生として `scripts/update_claude_md.py` の Stop hook 自動起動を特定
- 毎セッション終了時に Opus 4.6 が CLAUDE.md を再生成しており、プロンプトに secret 禁止句がないため平文 Key 再混入リスクが常在していた（log 実績: `logs/claude_md_update.log` に `[UpdateCLAUDE] CLAUDE.md を更新しました` が多数記録、commit レベルでは手動運用だが working tree レベルで LLM 書き換えが常時発生）
- 対応:
  - `~/affiliate-bot/.claude/settings.json` から Stop hook を削除（このファイルは `.gitignore` 済みのローカル設定、backup: `settings.json.backup.20260422_231233`）
  - `logs/claude_md_update.log` を `logs/claude_md_update.log.archived_20260422_231234` に退避
  - `scripts/update_claude_md.py` 冒頭に `DISABLED = True` 早期 return を追加（`--force` 引数ありでのみ動作）
- 再有効化条件:
  1. プロンプトに「API Key / token / secret / password / webhook URL を絶対に出力するな」を明記
  2. 出力前に secret パターン grep（`rnd_|sk-ant-|xoxb-|EAA[A-Za-z0-9]{20,}`）でブロック
  3. 設計再評価: CLAUDE.md は 60 行 goal-tree の手動管理（memory #5）と矛盾しないか判断

## 関連 commit

- affiliate-bot `scripts/update_claude_md.py` 早期 return 追加: `425c4e2` (2026-04-22)
- affiliate-bot `.claude/settings.json` Stop hook 削除: **commit なし**（`.gitignore` 登録済みのローカル設定、ファイル差分はローカル backup で保全）
