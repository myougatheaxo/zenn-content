---
title: "Claude Codeで爆速デバッグ：バグの原因特定から修正まで"
emoji: "🔍"
type: "tech"
topics: ["claudecode", "debugging", "typescript", "python", "開発効率化"]
published: true
---

## はじめに

「このバグ、原因わからない」という状況でClaude Codeをどう使うか。

ただ「このエラーを直して」と貼り付けるのが最初のステップだが、それだけでは精度に限界がある。この記事では、Claude Codeのデバッグ活用パターンを整理する。

---

## パターン1: スタックトレースを丸ごと渡す

```
以下のエラーが発生しています。原因を特定して修正してください：

Error: Cannot read properties of undefined (reading 'email')
    at UserService.findById (src/services/user.service.ts:24:30)
    at UserController.getUser (src/controllers/user.controller.ts:15:35)
    at Layer.handle [as handle_request] (express/lib/router/layer.js:95:5)

UserServiceのコード：
[コードをここに貼る]
```

スタックトレース全体と関連コードを一緒に渡すと、ハルシネーションなく原因を特定できる。

---

## パターン2: 「なぜ動かないか」を先に考えさせる

修正より先に、原因の仮説を出させる。

```
このコードが期待通り動きません。修正する前に、原因の仮説を3つ挙げてください：

[問題のコード]

期待する動作：ユーザーIDでユーザーを取得する
実際の動作：nullが返ってくる
```

仮説を先に確認することで、的外れな修正を防げる。

---

## パターン3: ログを使った絞り込み

「どこにログを追加すれば原因を特定できるか」を聞く。

```
以下の関数でデータが正しく処理されているか確認したい。
どの箇所にどんなログを追加すれば原因を絞り込めますか？

[コード]
```

これでデバッグの戦略が立てやすくなる。

---

## パターン4: 環境差異の特定

「本番では動く、開発では動かない」系のバグに有効。

```
同じコードで、本番環境では正常動作するが開発環境ではエラーになります。

環境の違い：
- 本番: Node.js 18, PostgreSQL 15
- 開発: Node.js 20, PostgreSQL 16

エラー内容：[エラーメッセージ]

考えられる環境差異と確認ポイントを教えてください。
```

---

## パターン5: 再現手順の整理

バグレポートを書く前に、Claude Codeに再現手順を整理させる。

```
以下の不具合について、再現手順とバグレポートを作成してください：

現象：ユーザーが連続でログインしようとすると2回目でエラーになる
ログ：[ログをここに貼る]
関連コード：[コード]
```

---

## Hooksでデバッグ情報を自動収集

`.claude/hooks/debug_context.py` を設定すると、Bashコマンドのエラー出力を自動記録できる。

```python
# .claude/hooks/debug_context.py
import json
import sys
from pathlib import Path
from datetime import datetime

data = json.load(sys.stdin)
tool_result = data.get("tool_result", {})

# Bashコマンドが失敗した場合のみ記録
if data.get("tool_name") == "Bash" and tool_result.get("exit_code", 0) != 0:
    log_file = Path(".claude/debug_log.txt")
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    cmd = data.get("tool_input", {}).get("command", "")
    output = tool_result.get("output", "")

    with log_file.open("a", encoding="utf-8") as f:
        f.write(f"\n--- {timestamp} ---\n")
        f.write(f"Command: {cmd}\n")
        f.write(f"Output: {output[:500]}\n")

sys.exit(0)
```

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{"type": "command", "command": "python .claude/hooks/debug_context.py"}]
      }
    ]
  }
}
```

---

## CLAUDE.mdにデバッグコンテキストを書く

```markdown
## デバッグ情報
- ログ出力先: `logs/app.log` (JSON形式)
- エラー追跡: Sentry (本番のみ)
- DB接続確認: `python manage.py dbshell`
- キャッシュ確認: Redis CLI `redis-cli keys "*"`

## よくあるエラーパターン
- `NullReferenceError`: Prisma/ORMのリレーションを忘れてinclude/joinが必要
- `ValidationError`: Pydantic/Zodのスキーマと実データの不一致
- `TimeoutError`: 外部API呼び出しにtimeout設定が必要
```

こうしておくと「このエラーを直して」という指示でより正確な回答が得られる。

---

## まとめ

Claude Codeのデバッグ活用パターン：

1. **スタックトレース全体 + 関連コード**を一緒に渡す
2. **修正より先に仮説**を出させる
3. **ログ追加ポイント**を聞く
4. **環境差異**を整理させる
5. **CLAUDE.md**によくあるエラーパターンを書いておく

デバッグが速くなると開発全体のリズムが変わる。

---

*カスタムスキルでコードレビュー・デバッグ支援を自動化できます。**Code Review Pack（¥980）** はPromptWorksで購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
