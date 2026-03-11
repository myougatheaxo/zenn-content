---
title: "AI APIのレート制限・コスト管理：Claude Codeで実装する実践パターン"
emoji: "💰"
type: "tech"
topics:
  - claudecode
  - ai
  - cost
  - architecture
published: true
---

## AI APIコスト管理がなぜ重要か

LLMのAPI料金は従量制だ。設計を間違えると月末に予想外の請求が届く。プロダクション環境では、コスト管理とレート制限は最初から設計に組み込む必要がある。

Claude APIの料金体系（2026年現在）:
- Claude Opus: 入力$15/MTok・出力$75/MTok
- Claude Sonnet: 入力$3/MTok・出力$15/MTok
- Claude Haiku: 入力$0.25/MTok・出力$1.25/MTok

「とりあえずOpusで全部やる」は最もよくある失敗だ。用途に応じてモデルを使い分けるだけでコストが10〜60倍変わる。

## トークン使用量の計測とモニタリング

まずは使用量を正確に把握することから始める:

```python
import asyncio
from dataclasses import dataclass, field
from datetime import datetime, date
from collections import defaultdict
import anthropic

@dataclass
class TokenUsage:
    model: str
    input_tokens: int
    output_tokens: int
    cost_usd: float
    timestamp: datetime = field(default_factory=datetime.utcnow)

# モデル別料金（1M tokens当たり）
MODEL_PRICING = {
    "claude-opus-4-5":   {"input": 15.0,  "output": 75.0},
    "claude-sonnet-4-5": {"input": 3.0,   "output": 15.0},
    "claude-haiku-4-5":  {"input": 0.25,  "output": 1.25},
}

def calculate_cost(model: str, input_tokens: int, output_tokens: int) -> float:
    pricing = MODEL_PRICING.get(model, {"input": 3.0, "output": 15.0})
    return (
        input_tokens * pricing["input"] / 1_000_000
        + output_tokens * pricing["output"] / 1_000_000
    )

class CostTracker:
    def __init__(self):
        self._daily: dict[date, list[TokenUsage]] = defaultdict(list)
        self._total_cost: float = 0.0

    def record(self, model: str, input_tokens: int, output_tokens: int) -> float:
        cost = calculate_cost(model, input_tokens, output_tokens)
        usage = TokenUsage(model, input_tokens, output_tokens, cost)
        self._daily[date.today()].append(usage)
        self._total_cost += cost
        return cost

    def daily_cost(self, day: date | None = None) -> float:
        day = day or date.today()
        return sum(u.cost_usd for u in self._daily.get(day, []))

    def daily_budget_check(self, budget_usd: float) -> bool:
        return self.daily_cost() <= budget_usd

tracker = CostTracker()
```

## レート制限の実装：トークンバケット

APIのレート制限（RPM/TPM）を超えないようにするトークンバケット実装:

```python
import asyncio
import time
from dataclasses import dataclass

@dataclass
class RateLimiter:
    requests_per_minute: int
    tokens_per_minute: int

    _request_tokens: float = field(init=False)
    _token_tokens: float = field(init=False)
    _last_refill: float = field(init=False)
    _lock: asyncio.Lock = field(init=False)

    def __post_init__(self):
        self._request_tokens = float(self.requests_per_minute)
        self._token_tokens = float(self.tokens_per_minute)
        self._last_refill = time.monotonic()
        self._lock = asyncio.Lock()

    def _refill(self):
        now = time.monotonic()
        elapsed = now - self._last_refill
        self._last_refill = now

        # 経過時間に応じてトークンを補充
        self._request_tokens = min(
            self.requests_per_minute,
            self._request_tokens + elapsed * self.requests_per_minute / 60,
        )
        self._token_tokens = min(
            self.tokens_per_minute,
            self._token_tokens + elapsed * self.tokens_per_minute / 60,
        )

    async def acquire(self, estimated_tokens: int = 1000) -> None:
        async with self._lock:
            while True:
                self._refill()
                if self._request_tokens >= 1 and self._token_tokens >= estimated_tokens:
                    self._request_tokens -= 1
                    self._token_tokens -= estimated_tokens
                    return
                # 待機時間を計算
                wait_time = max(
                    (1 - self._request_tokens) * 60 / self.requests_per_minute,
                    (estimated_tokens - self._token_tokens) * 60 / self.tokens_per_minute,
                )
                await asyncio.sleep(min(wait_time, 1.0))

# Claude API制限値（Tier 1の例）
claude_limiter = RateLimiter(
    requests_per_minute=50,
    tokens_per_minute=40_000,
)

async def rate_limited_call(prompt: str, model: str = "claude-sonnet-4-5") -> str:
    client = anthropic.AsyncAnthropic()

    # 推定トークン数でレート制限を取得
    estimated = len(prompt) // 4 + 1000
    await claude_limiter.acquire(estimated_tokens=estimated)

    response = await client.messages.create(
        model=model,
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}],
    )

    # 実際の使用量を記録
    tracker.record(
        model,
        response.usage.input_tokens,
        response.usage.output_tokens,
    )
    return response.content[0].text
```

## コスト削減：キャッシュとモデル選択最適化

```python
import hashlib
import json
from typing import Any

class LLMCache:
    """同一プロンプトのキャッシュ（コスト削減）"""

    def __init__(self, redis_client=None):
        self._memory: dict[str, str] = {}  # フォールバック用インメモリ
        self.redis = redis_client

    def _cache_key(self, model: str, messages: list[dict]) -> str:
        content = json.dumps({"model": model, "messages": messages}, sort_keys=True)
        return hashlib.sha256(content.encode()).hexdigest()

    async def get(self, model: str, messages: list[dict]) -> str | None:
        key = self._cache_key(model, messages)
        if self.redis:
            value = await self.redis.get(f"llm:{key}")
            return value.decode() if value else None
        return self._memory.get(key)

    async def set(
        self,
        model: str,
        messages: list[dict],
        response: str,
        ttl: int = 3600,
    ) -> None:
        key = self._cache_key(model, messages)
        if self.redis:
            await self.redis.setex(f"llm:{key}", ttl, response.encode())
        else:
            self._memory[key] = response

cache = LLMCache()

def select_model_by_complexity(prompt: str) -> str:
    """プロンプトの複雑さに応じてモデルを選択"""
    length = len(prompt)
    # 単純なタスク（分類・抽出など）
    if length < 500:
        return "claude-haiku-4-5"
    # 中程度のタスク（要約・変換など）
    if length < 2000:
        return "claude-sonnet-4-5"
    # 複雑なタスク（コード生成・複雑な推論）
    return "claude-opus-4-5"

async def cost_optimized_call(
    messages: list[dict],
    force_model: str | None = None,
) -> str:
    prompt_text = " ".join(m.get("content", "") for m in messages)
    model = force_model or select_model_by_complexity(prompt_text)

    # キャッシュチェック
    cached = await cache.get(model, messages)
    if cached:
        return cached  # コストゼロ

    client = anthropic.AsyncAnthropic()
    response = await client.messages.create(
        model=model,
        max_tokens=1024,
        messages=messages,
    )
    result = response.content[0].text

    # キャッシュに保存
    await cache.set(model, messages, result)
    return result
```

## バジェットアラートとコスト上限

```python
import os
from typing import Callable

class BudgetGuard:
    def __init__(
        self,
        daily_limit_usd: float,
        alert_threshold: float = 0.8,
        on_alert: Callable | None = None,
        on_exceeded: Callable | None = None,
    ):
        self.daily_limit = daily_limit_usd
        self.alert_threshold = alert_threshold
        self.on_alert = on_alert
        self.on_exceeded = on_exceeded
        self._alerted = False

    async def check(self) -> None:
        current = tracker.daily_cost()
        ratio = current / self.daily_limit

        if ratio >= 1.0:
            if self.on_exceeded:
                await self.on_exceeded(current, self.daily_limit)
            raise BudgetExceededException(
                f"日次予算を超過: ${current:.2f} / ${self.daily_limit:.2f}"
            )

        if ratio >= self.alert_threshold and not self._alerted:
            self._alerted = True
            if self.on_alert:
                await self.on_alert(current, self.daily_limit)

class BudgetExceededException(Exception):
    pass

async def send_slack_alert(current: float, limit: float):
    # Slack webhook等でアラート送信
    print(f"ALERT: AI API costs ${current:.2f} / ${limit:.2f} today")

budget_guard = BudgetGuard(
    daily_limit_usd=10.0,
    alert_threshold=0.8,
    on_alert=send_slack_alert,
)

# FastAPIでのミドルウェア統合
from fastapi import Request, HTTPException
from starlette.middleware.base import BaseHTTPMiddleware

class BudgetMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        if "/api/llm" in request.url.path:
            try:
                await budget_guard.check()
            except BudgetExceededException as e:
                raise HTTPException(status_code=503, detail=str(e))
        return await call_next(request)
```

コスト管理の実装は地味だが、プロダクション運用では必須だ。早期に導入することで、スケールアップ時の請求ショックを防げる。

---

この記事の内容は、Claude Code完全攻略ガイド（全7章）の一部。CLAUDE.md設計、Hooks実践、MCPセットアップ、マルチエージェント構成まで全7章にまとめた完全版はnoteで公開している。

https://note.com/myougatheaxo/n/ncf4f814d6b68

*みょうが (@myougaTheAxo) ― ウーパールーパーのVTuber。AIツールの実践的な使い方を発信中。*
