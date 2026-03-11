---
title: "ベクターDB統合パターン：pgvector・Qdrant・Weaviateの使い分け"
emoji: "🗄️"
type: "tech"
topics:
  - claudecode
  - ai
  - database
  - pgvector
published: true
---

## ベクターDBを選ぶ基準

ベクターDBの選択はシステム全体のアーキテクチャに影響する。既存インフラ・スケール要件・精度要求・運用コストを総合的に判断する必要がある。

| DB | 特徴 | 向いているケース |
|-----|------|----------------|
| pgvector | PostgreSQLに統合 | 既存PGインフラ・小〜中規模 |
| Qdrant | 高性能・Rust実装 | 大規模・高スループット |
| Weaviate | GraphQL・スキーマ管理 | 構造化データ+ベクター |
| Pinecone | フルマネージド | 運用コスト最小化 |
| Chroma | 組み込み・開発向け | プロトタイプ・ローカル開発 |

選択の第一基準は「既存スタックとの統合容易性」だ。PostgreSQLをすでに使っているならpgvectorが最も低摩擦だ。

## pgvector：PostgreSQLに最短距離で統合する

```sql
-- pgvector拡張のインストール
CREATE EXTENSION IF NOT EXISTS vector;

-- ドキュメントテーブルの設計
CREATE TABLE embeddings (
    id          BIGSERIAL PRIMARY KEY,
    content     TEXT NOT NULL,
    metadata    JSONB DEFAULT '{}',
    embedding   vector(1536),  -- OpenAI text-embedding-3-small
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

-- HNSWインデックス（高精度・メモリ多め）
CREATE INDEX ON embeddings USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- IVFFlatインデックス（低メモリ・構築速い）
-- CREATE INDEX ON embeddings USING ivfflat (embedding vector_cosine_ops)
--     WITH (lists = 100);

-- 類似検索クエリ
SELECT id, content, metadata,
       1 - (embedding <=> $1::vector) AS similarity
FROM embeddings
ORDER BY embedding <=> $1::vector
LIMIT 10;
```

Pythonからの操作:

```python
import asyncpg
from pgvector.asyncpg import register_vector
import numpy as np

class PgVectorStore:
    def __init__(self, conn: asyncpg.Connection):
        self.conn = conn

    async def upsert(
        self,
        content: str,
        embedding: list[float],
        metadata: dict,
        doc_id: str | None = None,
    ) -> int:
        return await self.conn.fetchval("""
            INSERT INTO embeddings (content, embedding, metadata)
            VALUES ($1, $2::vector, $3::jsonb)
            ON CONFLICT ((metadata->>'doc_id'))
            DO UPDATE SET
                content = EXCLUDED.content,
                embedding = EXCLUDED.embedding,
                updated_at = NOW()
            RETURNING id
        """, content, embedding, {"doc_id": doc_id, **metadata})

    async def search(
        self,
        query_embedding: list[float],
        limit: int = 5,
        filter_metadata: dict | None = None,
    ) -> list[dict]:
        where_clause = ""
        params = [query_embedding, limit]

        if filter_metadata:
            conditions = []
            for i, (k, v) in enumerate(filter_metadata.items(), 3):
                conditions.append(f"metadata->>{chr(39)}{k}{chr(39)} = ${i}")
                params.append(str(v))
            where_clause = "WHERE " + " AND ".join(conditions)

        rows = await self.conn.fetch(f"""
            SELECT id, content, metadata,
                   1 - (embedding <=> $1::vector) AS similarity
            FROM embeddings
            {where_clause}
            ORDER BY embedding <=> $1::vector
            LIMIT $2
        """, *params)
        return [dict(row) for row in rows]
```

## Qdrant：大規模高速ベクター検索

Qdrantはゼロからベクター検索のために設計されており、フィルタリング性能が特に優れている:

```python
from qdrant_client import AsyncQdrantClient
from qdrant_client.models import (
    Distance, VectorParams, PointStruct,
    Filter, FieldCondition, MatchValue,
)
import uuid

client = AsyncQdrantClient(url="http://localhost:6333")

async def setup_collection(collection_name: str, vector_size: int = 1536):
    await client.create_collection(
        collection_name=collection_name,
        vectors_config=VectorParams(
            size=vector_size,
            distance=Distance.COSINE,
        ),
    )

async def upsert_documents(
    collection_name: str,
    documents: list[dict],
    embeddings: list[list[float]],
):
    points = [
        PointStruct(
            id=str(uuid.uuid4()),
            vector=embedding,
            payload={
                "content": doc["content"],
                "source": doc.get("source", ""),
                "tags": doc.get("tags", []),
                "created_at": doc.get("created_at", ""),
            },
        )
        for doc, embedding in zip(documents, embeddings)
    ]
    await client.upsert(collection_name=collection_name, points=points)

async def filtered_search(
    collection_name: str,
    query_embedding: list[float],
    tag_filter: str | None = None,
    limit: int = 5,
) -> list[dict]:
    query_filter = None
    if tag_filter:
        query_filter = Filter(
            must=[
                FieldCondition(
                    key="tags",
                    match=MatchValue(value=tag_filter),
                )
            ]
        )

    results = await client.search(
        collection_name=collection_name,
        query_vector=query_embedding,
        query_filter=query_filter,
        limit=limit,
        with_payload=True,
    )
    return [
        {
            "id": r.id,
            "score": r.score,
            **r.payload,
        }
        for r in results
    ]
```

## 複数ベクターDBの抽象化レイヤー

本番システムではベクターDBの実装を抽象化して切り替えやすくする:

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class SearchResult:
    id: str
    content: str
    score: float
    metadata: dict

class VectorStore(ABC):
    @abstractmethod
    async def insert(
        self,
        content: str,
        embedding: list[float],
        metadata: dict,
    ) -> str:
        ...

    @abstractmethod
    async def search(
        self,
        query_embedding: list[float],
        limit: int = 5,
        filters: dict | None = None,
    ) -> list[SearchResult]:
        ...

    @abstractmethod
    async def delete(self, doc_id: str) -> bool:
        ...

class PgVectorStoreImpl(VectorStore):
    async def insert(self, content, embedding, metadata) -> str:
        # pgvector実装
        ...

    async def search(self, query_embedding, limit=5, filters=None):
        # pgvector実装
        ...

    async def delete(self, doc_id) -> bool:
        # pgvector実装
        ...

class QdrantStoreImpl(VectorStore):
    async def insert(self, content, embedding, metadata) -> str:
        # Qdrant実装
        ...

    async def search(self, query_embedding, limit=5, filters=None):
        # Qdrant実装
        ...

    async def delete(self, doc_id) -> bool:
        # Qdrant実装
        ...

def create_vector_store(backend: str, **kwargs) -> VectorStore:
    if backend == "pgvector":
        return PgVectorStoreImpl(**kwargs)
    elif backend == "qdrant":
        return QdrantStoreImpl(**kwargs)
    raise ValueError(f"Unknown backend: {backend}")
```

## インデックス戦略とパフォーマンスチューニング

```python
# pgvectorのパフォーマンスチューニング
PGVECTOR_TUNING_SQL = """
-- HNSWのef_searchパラメータ（精度vs速度トレードオフ）
SET hnsw.ef_search = 100;  -- デフォルト40、高いほど精度向上・速度低下

-- 並列クエリ有効化
SET max_parallel_workers_per_gather = 4;

-- メモリ設定（大規模インデックス構築時）
SET maintenance_work_mem = '2GB';
"""

# バルクインサートの最適化
async def bulk_insert_optimized(
    conn: asyncpg.Connection,
    records: list[tuple[str, list[float], dict]],
    batch_size: int = 1000,
) -> int:
    """大量ドキュメントの効率的な一括挿入"""
    total = 0
    for i in range(0, len(records), batch_size):
        batch = records[i:i + batch_size]
        # copy_records_to_table を使うと INSERT より10倍以上速い
        await conn.copy_records_to_table(
            "embeddings",
            records=[(c, e, m) for c, e, m in batch],
            columns=["content", "embedding", "metadata"],
        )
        total += len(batch)
    return total
```

## 本番運用：監視・バックアップ・更新戦略

```python
import time

async def monitor_vector_store_health(store: VectorStore) -> dict:
    """ベクターDB健全性チェック"""
    start = time.monotonic()

    # テストクエリで応答時間計測
    test_embedding = [0.0] * 1536
    results = await store.search(test_embedding, limit=1)

    latency_ms = (time.monotonic() - start) * 1000

    return {
        "status": "healthy" if latency_ms < 100 else "degraded",
        "latency_ms": round(latency_ms, 2),
        "result_count": len(results),
    }

# 定期的なインデックス再構築（pgvector）
REINDEX_SQL = """
-- バキューム後にインデックス再構築
VACUUM ANALYZE embeddings;
REINDEX INDEX CONCURRENTLY embeddings_embedding_idx;
"""
```

ベクターDBの選択に正解はないが、まずpgvectorでシンプルに始めてスケール要件が明確になってからQdrantへ移行するアプローチが実践的だ。

---

この記事の内容は、Claude Code完全攻略ガイド（全7章）の一部。CLAUDE.md設計、Hooks実践、MCPセットアップ、マルチエージェント構成まで全7章にまとめた完全版はnoteで公開している。

https://note.com/myougatheaxo/n/ncf4f814d6b68

*みょうが (@myougaTheAxo) ― ウーパールーパーのVTuber。AIツールの実践的な使い方を発信中。*
