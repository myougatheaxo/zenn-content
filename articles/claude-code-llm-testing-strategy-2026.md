---
title: "LLM統合コードのテスト戦略：モック・評価・回帰テストの実践"
emoji: "🧪"
type: "tech"
topics:
  - claudecode
  - testing
  - ai
  - llm
published: true
---

## LLMテストの難しさ：非決定的システムをどう検証するか

通常のコードテストと違い、LLMは同じ入力に対して毎回異なる出力を返す。「期待値と一致するか」というアサーションが使えない。これがLLM統合テストを困難にする根本的な理由だ。

しかし、テストできないシステムは信頼できない。LLMを使ったシステムには独自のテスト戦略が必要だ。

テストすべき項目を分類すると:
1. **ビジネスロジック**: プロンプトに依存しないコード部分（通常のユニットテスト）
2. **プロンプト統合**: LLMとのインターフェース（モック・スタブ）
3. **LLM動作品質**: 出力の品質・安全性（評価フレームワーク）
4. **エンドツーエンド**: システム全体の動作（本物のLLMを使った統合テスト）

## LLMのモック戦略

プロンプトのユニットテストにはモックが不可欠だ:

```python
from unittest.mock import AsyncMock, patch, MagicMock
import pytest
from anthropic.types import Message, TextBlock, Usage

def create_mock_message(text: str) -> Message:
    """Anthropicレスポンスのモックを生成"""
    return Message(
        id="msg_test_123",
        type="message",
        role="assistant",
        content=[TextBlock(type="text", text=text)],
        model="claude-opus-4-5",
        stop_reason="end_turn",
        stop_sequence=None,
        usage=Usage(input_tokens=100, output_tokens=50),
    )

@pytest.fixture
def mock_anthropic():
    with patch("anthropic.AsyncAnthropic") as MockClient:
        instance = AsyncMock()
        MockClient.return_value = instance
        instance.messages.create.return_value = create_mock_message(
            "モックレスポンスです"
        )
        yield instance

# テスト対象のコード
from myapp.llm_service import summarize_text

@pytest.mark.asyncio
async def test_summarize_calls_correct_model(mock_anthropic):
    await summarize_text("長いテキスト...")

    mock_anthropic.messages.create.assert_called_once()
    call_kwargs = mock_anthropic.messages.create.call_args.kwargs
    assert call_kwargs["model"] == "claude-opus-4-5"
    assert call_kwargs["max_tokens"] > 0

@pytest.mark.asyncio
async def test_summarize_includes_text_in_prompt(mock_anthropic):
    input_text = "テストの本文テキスト"
    await summarize_text(input_text)

    call_kwargs = mock_anthropic.messages.create.call_args.kwargs
    # ユーザーメッセージに入力テキストが含まれているか確認
    user_message = call_kwargs["messages"][0]["content"]
    assert input_text in user_message
```

## シナリオベーステスト：入出力ペアでプロンプトを検証

プロンプト変更の影響を検知するためのシナリオテスト:

```python
import json
from pathlib import Path
from typing import Callable

@pytest.fixture
def test_scenarios():
    """テストシナリオをJSONファイルから読み込む"""
    scenarios_path = Path("tests/fixtures/llm_scenarios.json")
    return json.loads(scenarios_path.read_text(encoding="utf-8"))

# tests/fixtures/llm_scenarios.json の例:
# [
#   {
#     "name": "positive_sentiment",
#     "input": "今日は最高の一日だった！",
#     "expected_label": "positive",
#     "min_confidence": 0.8
#   },
#   ...
# ]

async def run_scenario_test(
    scenario: dict,
    llm_fn: Callable,
) -> dict:
    output = await llm_fn(scenario["input"])
    result = {
        "name": scenario["name"],
        "passed": True,
        "failures": [],
    }

    if "expected_label" in scenario:
        if output.get("label") != scenario["expected_label"]:
            result["passed"] = False
            result["failures"].append(
                f"Label: expected {scenario['expected_label']}, got {output.get('label')}"
            )

    if "min_confidence" in scenario:
        conf = output.get("confidence", 0)
        if conf < scenario["min_confidence"]:
            result["passed"] = False
            result["failures"].append(
                f"Confidence too low: {conf} < {scenario['min_confidence']}"
            )

    return result
```

## プロンプト回帰テスト：変更前後の比較

プロンプトを更新するたびに品質が下がっていないかチェックする:

```python
import hashlib
import json
from datetime import datetime
from pathlib import Path

class PromptRegressionTracker:
    def __init__(self, baseline_dir: Path):
        self.baseline_dir = baseline_dir
        self.baseline_dir.mkdir(parents=True, exist_ok=True)

    def _prompt_hash(self, prompt: str) -> str:
        return hashlib.sha256(prompt.encode()).hexdigest()[:8]

    def save_baseline(self, prompt: str, outputs: list[str]) -> str:
        """現在の出力をベースラインとして保存"""
        prompt_id = self._prompt_hash(prompt)
        baseline = {
            "prompt_hash": prompt_id,
            "timestamp": datetime.utcnow().isoformat(),
            "outputs": outputs,
        }
        path = self.baseline_dir / f"{prompt_id}.json"
        path.write_text(json.dumps(baseline, ensure_ascii=False, indent=2))
        return prompt_id

    def compare_to_baseline(
        self,
        prompt: str,
        new_outputs: list[str],
        evaluator: Callable[[str, str], float],
    ) -> dict:
        """新しい出力とベースラインを比較"""
        prompt_id = self._prompt_hash(prompt)
        path = self.baseline_dir / f"{prompt_id}.json"

        if not path.exists():
            return {"status": "no_baseline", "message": "ベースラインなし"}

        baseline = json.loads(path.read_text())
        scores = []
        for new, old in zip(new_outputs, baseline["outputs"]):
            score = evaluator(new, old)
            scores.append(score)

        avg_score = sum(scores) / len(scores) if scores else 0
        return {
            "status": "regression" if avg_score < 0.7 else "ok",
            "similarity_score": avg_score,
            "sample_count": len(scores),
        }
```

## LLM出力評価：品質を数値化する

```python
from anthropic import AsyncAnthropic

class LLMEvaluator:
    """LLMを使ってLLM出力を評価する（LLM-as-judge）"""

    def __init__(self):
        self.client = AsyncAnthropic()

    async def evaluate_factuality(
        self,
        question: str,
        answer: str,
        reference: str,
    ) -> dict:
        prompt = f"""以下の質問、回答、参考情報を評価してください。

質問: {question}
回答: {answer}
参考情報: {reference}

以下の基準で1-5点で評価し、JSONで返してください:
- factuality: 事実の正確さ（参考情報と一致しているか）
- relevance: 質問への関連性
- completeness: 回答の完全性

形式: {{"factuality": N, "relevance": N, "completeness": N, "reason": "..."}}"""

        response = await self.client.messages.create(
            model="claude-haiku-4-5",  # 評価には安価なモデルを使う
            max_tokens=256,
            messages=[{"role": "user", "content": prompt}],
        )

        import re
        text = response.content[0].text
        json_match = re.search(r"\{.*\}", text, re.DOTALL)
        if json_match:
            return json.loads(json_match.group())
        return {"error": "パース失敗"}

    async def evaluate_safety(self, text: str) -> dict:
        """安全性評価"""
        prompt = f"""以下のテキストの安全性を評価してください。

テキスト: {text}

問題があれば該当する分類を列挙し、JSONで返してください:
{{"is_safe": true/false, "issues": [], "severity": "none/low/medium/high"}}"""

        response = await self.client.messages.create(
            model="claude-haiku-4-5",
            max_tokens=256,
            messages=[{"role": "user", "content": prompt}],
        )
        # パース処理...
        return {}
```

## CI/CDへの統合：自動化されたLLMテスト

```yaml
# .github/workflows/llm-tests.yml
name: LLM Integration Tests
on:
  push:
    paths:
      - 'src/prompts/**'
      - 'src/llm/**'

jobs:
  llm-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run unit tests (mocked LLM)
        run: pytest tests/unit/ -v
        env:
          ANTHROPIC_API_KEY: "dummy"

      - name: Run prompt scenario tests
        run: pytest tests/scenarios/ -v --timeout=60
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        if: github.event_name == 'push'

      - name: Run regression check
        run: python scripts/check_regression.py
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

```python
# scripts/check_regression.py
import asyncio
import sys
from pathlib import Path

async def main():
    tracker = PromptRegressionTracker(Path("tests/baselines"))
    evaluator = LLMEvaluator()

    # プロンプトファイルを読み込んで回帰チェック
    results = []
    for prompt_file in Path("src/prompts").glob("*.txt"):
        prompt = prompt_file.read_text()
        # テスト実行...
        result = tracker.compare_to_baseline(prompt, new_outputs, ...)
        results.append(result)

    regressions = [r for r in results if r["status"] == "regression"]
    if regressions:
        print(f"REGRESSION DETECTED: {len(regressions)} prompts degraded")
        sys.exit(1)
    print("All prompts OK")

asyncio.run(main())
```

LLMテストは100%完璧にはならないが、継続的な品質監視の仕組みを作ることで、プロンプト変更やモデルアップデートによる劣化を早期に発見できる。

---

この記事の内容は、Claude Code完全攻略ガイド（全7章）の一部。CLAUDE.md設計、Hooks実践、MCPセットアップ、マルチエージェント構成まで全7章にまとめた完全版はnoteで公開している。

https://note.com/myougatheaxo/n/ncf4f814d6b68

*みょうが (@myougaTheAxo) ― ウーパールーパーのVTuber。AIツールの実践的な使い方を発信中。*
