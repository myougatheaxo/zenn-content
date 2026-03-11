---
title: "RAG実装パターン完全ガイド：Claude Codeで検索拡張生成を構築する"
emoji: "🔍"
type: "tech"
topics:
  - claudecode
  - ai
  - rag
  - llm
published: true
---

## RAGとは何か：なぜ今必要なのか

RAG（Retrieval-Augmented Generation）は、LLMの知識制限を外部ドキュメントで補完するアーキテクチャパターンだ。Claude Codeでプロダクションシステムを構築する際、社内ドキュメント・コードベース・リアルタイムデータを活用するために不可欠な技術となっている。

単純なプロンプトにコンテキストを詰め込む方式と比べて、RAGには明確な利点がある。コストが下がり、精度が上がり、ハルシネーションが減る。

基本的なフローは以下の通り:

1. ドキュメントをチャンクに分割
2. 各チャンクをベクター埋め込みに変換
3. ベクターDBに保存
4. クエリ時に類似チャンクを検索
5. 取得したチャンクをコンテキストとしてLLMに渡す

## ドキュメント分割戦略

チャンク分割は RAG の精度に直結する。サイズが小さすぎるとコンテキストが失われ、大きすぎると無関係な情報が混入する。

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

def create_chunks(text: str) -> list[str]:
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=512,
        chunk_overlap=64,
        separators=["\n\n", "\n", "。", ".", " ", ""],
    )
    return splitter.split_text(text)

# Markdownドキュメント向け：ヘッダーで分割
from langchain.text_splitter import MarkdownHeaderTextSplitter

headers_to_split_on = [
    ("#", "Header 1"),
    ("##", "Header 2"),
    ("###", "Header 3"),
]

md_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=headers_to_split_on
)
md_header_splits = md_splitter.split_text(markdown_document)
```

コードファイルを扱う場合は、関数・クラス単位で分割する方が意味的に正確だ:

```python
import ast

def split_python_file(source: str) -> list[dict]:
    tree = ast.parse(source)
    chunks = []
    for node in ast.walk(tree):
        if isinstance(node, (ast.FunctionDef, ast.ClassDef)):
            start = node.lineno - 1
            end = node.end_lineno
            lines = source.split("\n")[start:end]
            chunks.append({
                "content": "\n".join(lines),
                "type": type(node).__name__,
                "name": node.name,
                "start_line": start + 1,
            })
    return chunks
```

## 埋め込みモデルの選択と実装

埋め込みモデルの選択はコストと精度のトレードオフだ。

```python
from openai import AsyncOpenAI
import numpy as np
from typing import List

client = AsyncOpenAI()

async def embed_texts(texts: list[str]) -> list[list[float]]:
    """テキストリストをバッチで埋め込む"""
    # OpenAI text-embedding-3-small: 低コスト・高速
    response = await client.embeddings.create(
        model="text-embedding-3-small",
        input=texts,
        encoding_format="float",
    )
    return [item.embedding for item in response.data]

async def embed_with_retry(texts: list[str], batch_size: int = 100) -> list[list[float]]:
    """バッチ処理 + リトライ付き埋め込み"""
    all_embeddings = []
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        embeddings = await embed_texts(batch)
        all_embeddings.extend(embeddings)
    return all_embeddings
```

ローカルモデルを使う場合はコストゼロで実行できる:

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("intfloat/multilingual-e5-small")

def embed_local(texts: list[str]) -> np.ndarray:
    # 日本語対応の多言語モデル
    prefixed = [f"query: {t}" for t in texts]
    return model.encode(prefixed, normalize_embeddings=True)
```

## ベクターDB統合とハイブリッド検索

pgvectorを使ったPostgreSQL統合は、既存のRDBMSインフラをそのまま活用できる:

```python
import asyncpg
from pgvector.asyncpg import register_vector

async def setup_db(conn_str: str) -> asyncpg.Connection:
    conn = await asyncpg.connect(conn_str)
    await register_vector(conn)
    await conn.execute("""
        CREATE TABLE IF NOT EXISTS documents (
            id SERIAL PRIMARY KEY,
            content TEXT NOT NULL,
            metadata JSONB,
            embedding vector(1536)
        )
    """)
    await conn.execute("""
        CREATE INDEX IF NOT EXISTS documents_embedding_idx
        ON documents USING ivfflat (embedding vector_cosine_ops)
        WITH (lists = 100)
    """)
    return conn

async def hybrid_search(
    conn: asyncpg.Connection,
    query_text: str,
    query_embedding: list[float],
    limit: int = 5,
    semantic_weight: float = 0.7,
) -> list[dict]:
    """セマンティック検索 + キーワード検索のハイブリッド"""
    rows = await conn.fetch("""
        WITH semantic AS (
            SELECT id, content, metadata,
                   1 - (embedding <=> $1::vector) AS sem_score
            FROM documents
            ORDER BY embedding <=> $1::vector
            LIMIT 20
        ),
        keyword AS (
            SELECT id, content, metadata,
                   ts_rank(to_tsvector('japanese', content),
                           plainto_tsquery('japanese', $2)) AS kw_score
            FROM documents
            WHERE to_tsvector('japanese', content) @@ plainto_tsquery('japanese', $2)
            LIMIT 20
        )
        SELECT s.id, s.content, s.metadata,
               ($3 * COALESCE(s.sem_score, 0) + (1-$3) * COALESCE(k.kw_score, 0)) AS score
        FROM semantic s
        LEFT JOIN keyword k USING (id)
        ORDER BY score DESC
        LIMIT $4
    """, query_embedding, query_text, semantic_weight, limit)
    return [dict(row) for row in rows]
```

## LLMへのコンテキスト注入パターン

検索結果をLLMに渡す際の構造化方法が重要だ:

```python
import anthropic

async def rag_query(
    query: str,
    retrieved_docs: list[dict],
    system_prompt: str = "",
) -> str:
    client = anthropic.AsyncAnthropic()

    # コンテキストを構造化して注入
    context_parts = []
    for i, doc in enumerate(retrieved_docs, 1):
        source = doc.get("metadata", {}).get("source", "unknown")
        context_parts.append(
            f"[Document {i}] (source: {source})\n{doc['content']}"
        )
    context = "\n\n---\n\n".join(context_parts)

    messages = [
        {
            "role": "user",
            "content": f"""以下の参考ドキュメントを元に質問に答えてください。

<context>
{context}
</context>

<question>
{query}
</question>

参考ドキュメントに記載されていない情報については、その旨を明示してください。""",
        }
    ]

    response = await client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2048,
        system=system_prompt or "あなたは技術的な質問に答えるアシスタントです。",
        messages=messages,
    )
    return response.content[0].text

# RAGパイプライン全体
async def rag_pipeline(query: str) -> str:
    # 1. クエリを埋め込む
    query_embedding = (await embed_texts([query]))[0]
    # 2. 類似ドキュメントを検索
    docs = await hybrid_search(conn, query, query_embedding, limit=5)
    # 3. LLMに渡して回答生成
    return await rag_query(query, docs)
```

## 評価とデバッグ：RAGの精度を測る

RAGシステムの評価なしには改善できない。RAGASフレームワークを使った評価:

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,           # 回答が文書に忠実か
    answer_relevancy,       # 回答がクエリに関連しているか
    context_precision,      # 検索文書の精度
    context_recall,         # 検索文書の再現率
)
from datasets import Dataset

def evaluate_rag(test_cases: list[dict]) -> dict:
    """
    test_cases: [
        {
            "question": "...",
            "answer": "...",        # RAGの回答
            "contexts": ["..."],    # 検索されたコンテキスト
            "ground_truth": "...",  # 正解
        }
    ]
    """
    dataset = Dataset.from_list(test_cases)
    result = evaluate(
        dataset,
        metrics=[
            faithfulness,
            answer_relevancy,
            context_precision,
            context_recall,
        ],
    )
    return result

# チャンク品質のデバッグ
def debug_retrieval(query: str, retrieved: list[dict]) -> None:
    print(f"Query: {query}")
    print(f"Retrieved {len(retrieved)} documents:")
    for i, doc in enumerate(retrieved, 1):
        score = doc.get("score", 0)
        preview = doc["content"][:100].replace("\n", " ")
        print(f"  [{i}] score={score:.3f} | {preview}...")
```

本番環境では検索ログを蓄積し、スコア分布・ヒット率・ユーザーフィードバックを定期的に分析することで、チャンク戦略とモデル選択を継続的に改善できる。

---

この記事の内容は、Claude Code完全攻略ガイド（全7章）の一部。CLAUDE.md設計、Hooks実践、MCPセットアップ、マルチエージェント構成まで全7章にまとめた完全版はnoteで公開している。

https://note.com/myougatheaxo/n/ncf4f814d6b68

*みょうが (@myougaTheAxo) ― ウーパールーパーのVTuber。AIツールの実践的な使い方を発信中。*
