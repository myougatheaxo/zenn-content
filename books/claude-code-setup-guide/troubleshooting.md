---
title: "トラブルシューティングFAQ — Zenn版限定"
---

現場でよく遭遇するトラブルを一問一答形式でまとめました。公式ドキュメントに載っていない実運用レベルの原因と対処法を記載しています。

---

## CLAUDE.md / 設定系

### Q1. CLAUDE.mdが読み込まれていない気がする

**症状**: 毎回同じ説明をしないといけない、禁止事項を守ってくれない。

**原因と対処:**

1. **配置場所の問題** — Gitリポジトリのルートから上位ディレクトリまで探索されますが、`~/.claude/CLAUDE.md`（ユーザーレベル）と`プロジェクトルート/CLAUDE.md`（プロジェクトレベル）の両方が存在する場合、競合する記述があると混乱します。

2. **確認方法** — 会話の冒頭で「CLAUDE.mdの内容を要約して」と聞くと、実際に読み込まれている内容が確認できます。

3. **ファイルサイズ超過** — CLAUDE.mdが大きすぎるとコンテキストウィンドウを圧迫します。目安は500行以内。不要な説明は削除し、詳細は別ファイルに切り出して「詳細は `docs/xxx.md` を参照」と書きましょう。

4. **BOM問題（Windows）** — WindowsのメモアプリやVS Codeの一部設定でBOM付きUTF-8で保存されることがあります。`file -i CLAUDE.md` で確認し、BOMなしUTF-8で保存し直してください。

---

### Q2. settings.jsonの設定が反映されない

**症状**: `~/.claude/settings.json` に書いたはずのpermissionsが効いていない。

**原因と対処:**

1. **JSONシンタックスエラー** — カンマ漏れや閉じ括弧の不足でファイル全体が無視されます。`python -m json.tool ~/.claude/settings.json` でバリデーションしてください。

2. **設定の階層優先順位** — ユーザー設定 < プロジェクト設定 < 管理者設定の順で上書きされます。プロジェクトの `.claude/settings.json` が存在する場合、そちらが優先されます。

3. **再起動が必要** — 設定変更後はClaude Codeを完全に終了して再起動してください。ホットリロードはされません。

```json
// 設定確認コマンド（Claude Codeのセッション内で）
// 「現在の設定を確認して」と聞くか、/statusコマンドを使う
```

---

## Hooks系

### Q3. Hooksが実行されない

**症状**: PreToolUseに設定したスクリプトが動いていない。

**チェックリスト:**

```json
// settings.jsonのHooks設定例
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python /absolute/path/to/hook.py"
          }
        ]
      }
    ]
  }
}
```

- **絶対パスを使う**: 相対パスは動作しません。スクリプトのパスは必ず絶対パスで記述してください。
- **実行権限の確認（Mac/Linux）**: `chmod +x hook.py` で実行権限を付与してください。
- **matcherの書き方**: `"Bash"` はツール名の完全一致です。`"bash"` と小文字で書くと動きません。

---

### Q4. Hooksでexit codeエラーが出て処理が止まる

**症状**: `Hook exited with code 1` というエラーが出て操作がブロックされる。

**仕様の確認:**

- `exit 0` → 処理を続行
- `exit 1` — `exit 2` → 処理をブロック（エラー扱い）
- `exit 2` → セッション停止

**よくある原因:**

```python
# 悪い例: 例外が発生するとexit 1になる
import sys
data = json.loads(sys.stdin.read())
# json.JSONDecodeError が出ると意図せず exit 1

# 良い例: try/exceptで明示的にハンドリング
import sys, json

try:
    data = json.loads(sys.stdin.read())
except Exception:
    sys.exit(0)  # 読めなくても続行させる
```

スクリプトの標準出力は `{"decision": "approve"}` や `{"decision": "block", "reason": "理由"}` の形式で返す必要があります。

---

## MCP系

### Q5. MCPサーバーの接続が切れる / 再接続できない

**症状**: しばらく使っていると `MCP server disconnected` が出る。

**対処法:**

1. **タイムアウト設定を確認** — stdioベースのMCPサーバーはプロセスが死ぬと接続が切れます。ロングランニングな処理はタイムアウト設定を見直してください。

2. **Claude Codeを再起動** — `/mcp` コマンドでMCPの状態を確認し、接続が切れていたら `claude` を再起動するのが最速です。

3. **ログ確認** — `~/.claude/logs/` にMCPサーバーのログが出力されます。エラーの詳細を確認してください。

4. **SSEとstdioの使い分け** — ローカルプロセスはstdio、リモートサーバーはSSE(HTTP)を使うのが安定します。stdioは軽量ですが、プロセス管理が必要です。

```json
// 安定するMCP設定例
{
  "mcpServers": {
    "myserver": {
      "command": "node",
      "args": ["/absolute/path/to/server.js"],
      "env": {
        "NODE_ENV": "production"
      }
    }
  }
}
```

---

### Q6. カスタムスキルが認識されない

**症状**: `/myskill` と打ってもスキルが見つからないと言われる。

**チェックリスト:**

1. **配置ディレクトリ** — スキルファイルは `~/.claude/commands/`（グローバル）または `.claude/commands/`（プロジェクト）に配置してください。

2. **ファイル拡張子** — `.md` 形式のみ有効です。`.txt` や `.yaml` では認識されません。

3. **frontmatterの形式** — 以下の形式が正しく書かれているか確認してください。

```markdown
---
description: スキルの説明（Claude Codeが選択時に参照）
argument-hint: 引数の説明（省略可）
allowed-tools: Bash, Read, Write
---

# スキル本文
$ARGUMENTS を使って処理する...
```

4. **スペースと文字コード** — ファイル名にスペースや日本語を使うと認識されないことがあります。英数字とハイフンのみにしてください。

---

## パーミッション系

### Q7. permissionエラーで実行が止まる

**症状**: `Permission denied` が頻発してタスクが進まない。

**対処の方向性:**

開発中の環境では、一時的にpermissionsを緩めてから必要なものだけ絞り込む方法が効率的です。

```json
// settings.json: 開発時の許可設定（本番環境では注意）
{
  "permissions": {
    "allow": [
      "Bash(npm *)",
      "Bash(git *)",
      "Bash(python *)",
      "Read(**)",
      "Write(**)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(curl * | bash *)"
    ]
  }
}
```

**パターンのコツ:**
- `Bash(git *)` → gitコマンド全般を許可
- `Bash(git push *)` → pushのみ許可
- `Read(**)` → 全ファイルの読み取りを許可（`**`はサブディレクトリ含む）

---

## パフォーマンス系

### Q8. Claude Codeが遅い / トークンを使いすぎる

**症状**: レスポンスが遅い、コスト請求が予想より大きい。

**原因と対策:**

1. **CLAUDE.mdが肥大化している** — 不要な記述を削除し、頻繁に参照する設定だけ残してください。

2. **コンテキストに大きなファイルが含まれる** — `cat` で大きなファイルを読ませるのではなく、必要な行だけを指定して読むよう指示してください。「`src/main.py`の100〜150行目を見て」という形で絞り込めます。

3. **長いセッションの使い回し** — 別件のタスクを始めるときは `/clear` でコンテキストをリセットしましょう。前の会話の内容がトークンを消費し続けます。

4. **Bashの出力が長い** — `npm install` の出力など、大量のログが流れるコマンドは `npm install 2>&1 | tail -20` のように末尾だけに絞る習慣をつけてください。

---

## Git系

### Q9. git操作で予期しない動作をする

**症状**: 意図せずコミットされた、ブランチが変わっていた。

**予防策:**

CLAUDE.mdに明示的な制約を書くのが最も効果的です。

```markdown
## Git運用ルール
- `git commit` は明示的に「コミットして」と言われた場合のみ実行
- `git push` は「pushして」と言われた場合のみ実行
- `git checkout` や `git switch` の前に必ず確認を取る
- `git reset --hard` は絶対に実行しない
- force pushは明示的な指示があっても一度確認を取る
```

**緊急時の対応:**

```bash
# 直前のコミットを取り消す（変更は残す）
git reset --soft HEAD~1

# 誤ってpushした場合（force pushが必要、要注意）
git push origin HEAD:branch-name --force-with-lease
```

---

### Q10. マルチエージェントが途中で止まる

**症状**: `Task` ツールで起動したサブエージェントが途中で応答を返さない。

**よくある原因:**

1. **タスクが曖昧すぎる** — サブエージェントに渡すプロンプトは「〇〇ファイルの△△関数をXXXに変更して結果を返して」のように完結した指示にしてください。「良い感じにして」という指示はエージェントが迷子になります。

2. **権限不足** — サブエージェントはデフォルトで親エージェントの権限設定を継承します。必要なツールが `allowedTools` に含まれているか確認してください。

3. **出力が大きすぎる** — サブエージェントが返す結果が大きすぎると処理が詰まります。「結果のサマリーだけ返して、詳細はファイルに書いて」という形でハンドオフすると安定します。

4. **並列数の上限** — デフォルトで同時起動できるサブエージェント数に上限があります。大量の並列タスクを出す場合は10件程度に分けて実行してください。

---

## Windows固有の問題

### Q11. WindowsでBashコマンドのパスが通らない

**症状**: `bash: python: command not found` や `No such file or directory` が出る。

**対処法:**

1. **WSL vs PowerShell** — Claude CodeはWindowsでもBashシェルを使います。`python` の代わりに `python3` が必要な場合があります。`where python` で実際のパスを確認してください。

2. **パス区切り文字** — WindowsのパスをClaude Codeに渡すときは、バックスラッシュ（`\`）ではなくスラッシュ（`/`）を使ってください。

```python
# 悪い例
path = "C:\Users\username\project"

# 良い例
path = "C:/Users/username/project"
# または
path = r"C:\Users\username\project"
```

3. **スペースを含むパス** — パスに空白が含まれる場合は必ずダブルクォートで囲んでください。`"C:/Program Files/..."` の形式で指定します。

4. **CLAUDE.mdでの宣言** — Windows環境では冒頭に明記しておくと誤操作を防げます。

```markdown
## 環境
- OS: Windows 11
- Shell: bash (Unix構文を使う、forward slash必須)
- Python: `py -3` または `python` (要確認)
```

---

### Q12. 日本語ファイル名・パスで文字化けする

**症状**: 日本語を含むパスでエラーになる、出力が文字化けする。

**対処法:**

1. **Windowsのコンソールエンコーディング** — デフォルトがCP932（Shift-JIS）の場合があります。`chcp 65001` でUTF-8に変更するか、環境変数で設定してください。

```bash
# セッション内で一時的に変更
export PYTHONIOENCODING=utf-8
```

2. **CLAUDE.mdへの明記** — エンコーディング設定を明示しておくとClaude Codeが適切なオプションを付けてくれます。

```markdown
## エンコーディング
- ソースコード: UTF-8 (BOMなし)
- Python実行時: PYTHONIOENCODING=utf-8
- 日本語パスは使わない（英数字ディレクトリに配置）
```

---

## その他のよくある問題

### Q13. 「変更しないで」と言ったのに変更される

**症状**: 「このファイルは触らないで」と指示したのに変更される。

**対処法:**

口頭指示は忘れられます。CLAUDE.mdに恒久的に記述してください。

```markdown
## 変更禁止ファイル・ディレクトリ
- `legacy/` ディレクトリ（本番稼働中、変更不可）
- `config/production.yaml`（手動管理、自動変更禁止）
- `*.lock` ファイル（パッケージマネージャーに任せる）
```

さらに強制したい場合は Hooks の PreToolUse で書き込みをブロックできます。

```python
import sys, json

data = json.loads(sys.stdin.read())
tool = data.get("tool_name", "")
tool_input = data.get("tool_input", {})

if tool in ("Write", "Edit"):
    path = tool_input.get("file_path", "")
    if "legacy/" in path or "production.yaml" in path:
        print(json.dumps({"decision": "block", "reason": f"{path} は変更禁止です"}))
        sys.exit(0)

print(json.dumps({"decision": "approve"}))
```

---

### Q14. コードレビューの精度が低い

**症状**: 「レビューして」と言うと表面的なコメントしか返ってこない。

**改善のポイント:**

指示を具体的にすると精度が上がります。

```
# 曖昧な指示
「このコードをレビューして」

# 具体的な指示
「src/api/users.py をレビューして。特に以下の観点で:
1. SQLインジェクションのリスク
2. エラーハンドリングの漏れ
3. N+1クエリの可能性
4. テストが不足している箇所
問題があれば修正案のコードも示して」
```

---

### Q15. 同じミスを繰り返す

**症状**: 一度指摘した問題を次のセッションでまた繰り返す。

**根本解決:** Claude Codeはセッション間で学習しません。「一度言えば覚える」は期待できません。

再発防止策は2つです:

1. **CLAUDE.mdに追記** — 禁止事項や注意事項として恒久化する。
2. **Hooksで強制** — コードで物理的にブロックする。

「Claude Codeに覚えてほしいこと」＝「CLAUDE.mdに書くべきこと」と考えるのが正しい運用です。

---

### Q16. ファイルの読み込みが途中で切れる

**症状**: 大きなファイルを「読んで」と言うと途中までしか読まれない。

**対処法:**

Claude Codeの `Read` ツールはデフォルトで2000行を上限としています。

```
# 全体を読む場合
「src/big_file.py の全行を読んで（offset=0, limit=5000 で）」

# 特定箇所を読む場合
「src/big_file.py の200行目から300行目を読んで」
```

大きなファイルを扱う場合は、必要な箇所を絞り込んで読む習慣をつけてください。コンテキストウィンドウの節約にもなります。

---

### Q17. `/clear` 後に前の文脈が必要になった

**症状**: `/clear` でリセットしたら、前のセッションで決めた仕様が分からなくなった。

**予防策:**

重要な決定事項はセッション中にファイルに書き出しておきましょう。

```
「今決めたアーキテクチャの方針を docs/architecture-decisions.md に箇条書きでまとめて」
```

あるいはCLAUDE.mdに随時追記する習慣をつけると、次のセッションでも文脈が引き継がれます。

---

## それでも解決しない場合

1. `claude --version` でバージョンを確認し、最新版にアップデートしてください。
2. `~/.claude/logs/` のログファイルを確認してください。
3. 再現手順を整理してX（[@myougaTheAxo](https://x.com/myougaTheAxo)）に投げてください。一緒に調べます。
