---
layout: default
title: "How We Built a Serverless Vector Database on FAISS and Cut Our AWS Bill by 70%"
date: 2023-08-15
description: "A deep dive into building an on-demand vector search solution using FAISS, Apache Arrow, and Kubernetes that reduced our infrastructure costs from $3,000/month to $200/month."
tags: [vector-database, faiss, apache-arrow, aws, kubernetes, python, infrastructure, cost-optimization]
categories: [engineering, infrastructure]
---

# How We Built a Serverless Vector Database on FAISS and Cut Our AWS Bill by 70%

When your vector database costs more than the rest of your infrastructure combined, it's time to rethink your architecture. Here's how we built an on-demand vector search solution using FAISS, Apache Arrow, S3, and Kubernetes—and what we learned along the way.

## The Problem: $3,000/Month for Similarity Search

We had two products that needed vector similarity search:

1. **Clustering Visualization Tool** — Users select data points, and we return semantically similar items from the dataset.
2. **Document Search (RAG Pipeline)** — Ingest PDFs on a schedule, index them, and provide relevant context to LLMs at query time.

Our initial choice was **Milvus**, a popular open-source vector database. It worked well functionally, but the economics didn't.

For ~6 million embeddings, Milvus required:

- **50 GB RAM**
- **16 vCPUs**
- **24/7 uptime**

Monthly cost: **~$3,000** — roughly **70% of our entire production AWS bill**.

The kicker? Neither product needed real-time, always-on search. Our clustering tool was used during analysis sessions. The RAG pipeline ran on schedules. We were paying for 24/7 availability when we needed maybe 4-6 hours of actual compute per day.

## The Insight: Decouple Compute from Storage

We were already working with **KServe** for ML model serving, which follows a serverless pattern: load model artifacts from object storage, serve inference requests, scale to zero when idle.

Vector search has a similar structure:

| Component | Characteristic |
|-----------|----------------|
| **Indexing** | Batch process, runs periodically |
| **Index artifacts** | Static files, can live in cold storage |
| **Search** | Compute-intensive, requires index in RAM |

The realization: **we don't need a database—we need on-demand compute with persistent storage.**

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Control Plane                            │
├─────────────────────────────────────────────────────────────────┤
│  Init Setup  │  Check Status  │  Search Query  │  Cleanup       │
└──────┬───────┴───────┬────────┴───────┬────────┴───────┬────────┘
       │               │                │                │
       ▼               ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  Search Pod │  │  Search Pod │  │  Search Pod │   ...        │
│  │  (Arrow +   │  │  (Arrow +   │  │  (Arrow +   │              │
│  │  FAISS +    │  │  FAISS +    │  │  FAISS +    │              │
│  │  HTTP API)  │  │  HTTP API)  │  │  HTTP API)  │              │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
└─────────┼────────────────┼────────────────┼─────────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│                         AWS S3                                  │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐          │
│  │ index.faiss  │  │ metadata.arrow│  │  config.json │          │
│  └──────────────┘  └───────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

## Deep Dive: The Components

### 1. Indexing Pipeline

Indexing runs as a batch job, completely decoupled from search. We experimented with several FAISS index types:

| Index Type | Use Case | Trade-off |
|------------|----------|-----------|
| **Flat (IndexFlatL2)** | < 100K vectors | Exact search, no compression, slow at scale |
| **IVF (IndexIVFFlat)** | 100K - 10M vectors | Approximate search, configurable accuracy vs. speed |
| **IVF + PQ** | > 10M vectors | Adds product quantization for memory efficiency |

For our 6M embedding dataset, **IVF with 1024 centroids** gave us the best balance of recall and latency.

**Index size optimization** was critical. Raw embeddings (768-dim, float32) for 6M vectors = ~18 GB. With quantization, we reduced this to ~6 GB—small enough to load quickly from S3.

```python
# Simplified indexing logic
import faiss
import numpy as np

def build_index(embeddings: np.ndarray, n_centroids: int = 1024):
    dim = embeddings.shape[1]

    # IVF index with L2 distance
    quantizer = faiss.IndexFlatL2(dim)
    index = faiss.IndexIVFFlat(quantizer, dim, n_centroids)

    # Train on a sample, then add all vectors
    index.train(embeddings)
    index.add(embeddings)

    return index

# Save to disk, then upload to S3
faiss.write_index(index, "index.faiss")
```

### 2. Why Apache Arrow Over Pandas/Dask

For on-demand infrastructure, **cold start time is everything**. We evaluated three options for in-memory metadata storage:

| Approach | Load Time (3GB metadata) | Memory Footprint | Cold Start Impact |
|----------|-------------------------|------------------|-------------------|
| **Pandas DataFrame** | ~45 seconds | ~4.5 GB | Unacceptable |
| **Dask DataFrame** | ~30 seconds | ~4.0 GB | Still too slow |
| **Arrow Table (memory-mapped)** | ~3 seconds | ~1.2 GB | ✅ Acceptable |

The difference comes down to how data is loaded:

**Pandas/Dask approach:**
```
S3 → Download → Deserialize CSV/Pickle → Allocate memory → Build DataFrame
```

**Arrow approach:**
```
S3 → Download IPC file → Memory-map → Ready
```

Arrow's IPC (Inter-Process Communication) format stores data in a binary columnar layout that's **identical to its in-memory representation**. There's no parsing, no type inference, no memory copying—just map the file and go.

```python
import pyarrow as pa

# Traditional Pandas - slow, memory-hungry
# df = pd.read_parquet("s3://bucket/metadata.parquet")  # 45 seconds

# Arrow with memory mapping - near-instant
source = pa.memory_map("/tmp/metadata.arrow", 'r')
metadata_table = pa.ipc.open_file(source).read_all()  # 3 seconds
```

#### Memory Management with jemalloc

Arrow uses **jemalloc** (or mimalloc) as its memory allocator, which provides better fragmentation handling than the system allocator. The key configuration that saved us:

```python
import pyarrow as pa

def compact_memory():
    """
    Release fragmented memory back to OS.
    Critical for long-running search pods.
    """
    pool = pa.default_memory_pool()
    pool.release_unused()

    return {
        "bytes_allocated": pool.bytes_allocated(),
        "max_memory": pool.max_memory()
    }
```

For even more aggressive memory management, we configured jemalloc's decay settings via environment variable:

```bash
# In pod startup script
export MALLOC_CONF="background_thread:true,dirty_decay_ms:5000,muzzy_decay_ms:5000"
```

- `dirty_decay_ms` — How quickly jemalloc returns "dirty" (recently freed) pages to the OS
- `muzzy_decay_ms` — How quickly it reclaims "muzzy" pages (marked with `MADV_FREE`)
- `background_thread:true` — Enables background purging so search threads aren't blocked

#### The Result

| Metric | Pandas | Arrow |
|--------|--------|-------|
| Load time (3GB) | 45s | 3s |
| Peak memory | 9GB (2x during load) | 3.2GB |
| Memory after GC | 4.5GB | 1.2GB (with `release_unused`) |
| Pod spin-up time | 90s | 35s |

The 60% reduction in cold start time made the on-demand architecture viable. Users could tolerate a 35-second wait; 90 seconds felt broken.

### 3. FAISS to Arrow Table Lookup

We kept the mapping dead simple: **FAISS vector IDs = Arrow table row indices**.

During indexing, we build the FAISS index with incremental IDs (0, 1, 2, ..., N), and the Arrow metadata table is ordered identically. When FAISS returns search results, the indices directly map to row positions.

```python
import faiss
import pyarrow as pa
import numpy as np

class VectorSearchService:
    def __init__(self):
        self.index: faiss.Index = None
        self.metadata_table: pa.Table = None

    def load(self, index_path: str, metadata_path: str):
        # Load FAISS index
        self.index = faiss.read_index(index_path)
        self.index.nprobe = 32

        # Load Arrow table via memory mapping
        source = pa.memory_map(metadata_path, 'r')
        self.metadata_table = pa.ipc.open_file(source).read_all()

    def search(self, query_vector: np.ndarray, k: int = 10):
        # FAISS search returns indices (0, 1, 2, ...)
        distances, indices = self.index.search(
            query_vector.reshape(1, -1).astype('float32'), k
        )

        # Direct positional lookup - indices ARE row numbers
        # Arrow's take() is O(k), not O(n)
        result_indices = indices[0].tolist()
        results = self.metadata_table.take(result_indices)

        return {
            "distances": distances[0].tolist(),
            "results": results.to_pylist()
        }
```

**Why this works well:**

| Operation | Complexity | Notes |
|-----------|------------|-------|
| FAISS search | O(log n) | IVF index with nprobe centroids |
| Arrow `take()` | O(k) | Direct memory offset calculation |
| Total lookup | O(log n + k) | No joins, no hash lookups |

Arrow's `take()` is essentially pointer arithmetic—given row index `i`, it computes the memory offset directly from the schema. For our 6M row table, retrieving 10 results takes **<0.1ms** after FAISS returns.

```
┌─────────────────────────────────────────────────────────────┐
│                      FAISS Index                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Vector 0 │ Vector 1 │ Vector 2 │ ... │ Vector N    │    │
│  └─────────────────────────────────────────────────────────┘    │
│       ↓           ↓          ↓                 ↓            │
│    ID: 0       ID: 1      ID: 2            ID: N           │
└─────────────────────────────────────────────────────────────┘
                              ↓
                    Search returns [42, 1337, 7, ...]
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    Arrow Table                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Row 0  │ Row 1  │ Row 2  │ ... │ Row N             │    │
│  │ {meta} │ {meta} │ {meta} │     │ {meta}            │    │
│  └─────────────────────────────────────────────────────────┘    │
│       ↑                                                     │
│    table.take([42, 1337, 7, ...])                          │
│    → Direct positional access, zero scanning               │
└─────────────────────────────────────────────────────────────┘
```

**Indexing Pipeline (ensuring alignment):**

```python
import pyarrow as pa
import faiss
import numpy as np

def build_aligned_index_and_metadata(documents: list[dict], embed_model):
    """
    Build FAISS index and Arrow table with aligned row IDs.
    """
    embeddings = []
    metadata_rows = []

    for idx, doc in enumerate(documents):
        # Embedding model
        embedding = embed_model.encode(doc["text"])
        embeddings.append(embedding)

        # Metadata (idx is implicit - it's the row position)
        metadata_rows.append({
            "doc_id": doc["id"],
            "title": doc["title"],
            "text_chunk": doc["text"][:500],
            "source_url": doc["url"],
        })

    # Build FAISS index - IDs are 0, 1, 2, ...
    embeddings_np = np.array(embeddings, dtype='float32')
    index = faiss.IndexFlatL2(embeddings_np.shape[1])
    index.add(embeddings_np)  # Adds with sequential IDs

    # Build Arrow table - row order matches FAISS IDs
    metadata_table = pa.Table.from_pylist(metadata_rows)

    # Save both
    faiss.write_index(index, "index.faiss")

    with pa.OSFile("metadata.arrow", 'wb') as f:
        writer = pa.ipc.new_file(f, metadata_table.schema)
        writer.write_table(metadata_table)
        writer.close()

    return index, metadata_table
```

This alignment is maintained through the entire pipeline—any re-indexing rebuilds both artifacts together, ensuring they stay in sync.

### 4. On-Demand Search Service

The search service is a lightweight HTTP server that:

1. Downloads the index and Arrow metadata from S3 on startup
2. Loads them into RAM (with memory mapping for metadata)
3. Exposes a `/search` endpoint
4. Runs background memory compaction

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager
import asyncio
import pyarrow as pa
import numpy as np

search_service = VectorSearchService()

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: Load index and metadata
    download_from_s3("s3://bucket/index.faiss", "/tmp/index.faiss")
    download_from_s3("s3://bucket/metadata.arrow", "/tmp/metadata.arrow")

    search_service.load("/tmp/index.faiss", "/tmp/metadata.arrow")

    # Start background compaction
    compaction_task = asyncio.create_task(
        background_compaction_loop()
    )

    # Warmup queries
    await warmup_index(search_service)

    yield

    # Shutdown
    compaction_task.cancel()

app = FastAPI(lifespan=lifespan)

async def background_compaction_loop(interval_seconds: int = 300):
    """Periodically release fragmented memory."""
    while True:
        await asyncio.sleep(interval_seconds)
        pool = pa.default_memory_pool()
        pool.release_unused()

@app.post("/search")
async def search(query_vector: list[float], k: int = 10):
    return search_service.search(np.array(query_vector), k)

@app.get("/health")
async def health():
    return {"status": "healthy", "index_size": search_service.index.ntotal}
```

### 5. Lifecycle Management API

The control plane manages the full lifecycle:

**`POST /setup/init`** — Spins up Kubernetes pods, triggers index download, starts HTTP server.

```json
{
  "index_id": "product-embeddings-v2",
  "replicas": 2,
  "ttl_minutes": 60
}
```

**`GET /setup/status/{setup_id}`** — Returns readiness state. Clients poll this before sending queries.

```json
{
  "status": "ready",
  "pods_ready": 2,
  "pods_total": 2,
  "elapsed_seconds": 45
}
```

**`POST /search/{setup_id}`** — Proxies to the underlying FAISS pods with load balancing.

**`DELETE /setup/{setup_id}`** — Immediate teardown. Also triggered automatically when TTL expires.

### 6. Background Processes

- **Index Watcher**: Polls S3 for new index versions. When detected, performs a rolling update—spin up new pods with the new index, drain old pods, zero downtime.

- **TTL Enforcer**: Each setup has a time-to-live. A background job checks for expired setups and terminates them. This prevents forgotten setups from accumulating costs.

- **Memory Monitor**: Tracks Arrow memory pool stats across pods. Alerts if memory usage exceeds thresholds.

## The Trade-offs

Nothing is free. Here's what we gave up:

| Aspect | Always-On (Milvus) | On-Demand (Ours) |
|--------|-------------------|------------------|
| **Cold start** | None | 30-90 seconds |
| **Cost (our workload)** | ~$3,000/month | ~$200/month |
| **Operational complexity** | Managed by Milvus | Custom lifecycle management |
| **Real-time ingestion** | Supported | Batch only |
| **Multi-tenancy** | Built-in | Manual (separate setups) |

For our use cases, the 30-90 second cold start was acceptable. Users initiate an analysis session, we spin up the index, they work for 30-60 minutes, we tear it down. The cost savings were massive.

## Performance Characteristics

After optimization, here's what we observed:

| Metric | Value |
|--------|-------|
| Index load time (6M vectors, 6GB index + 1.2GB metadata) | ~35 seconds |
| p50 search latency | 12ms |
| p99 search latency | 45ms |
| Queries per second (single pod) | ~200 |
| Memory footprint (per pod) | ~8 GB |
| Monthly cost | **~$200** (down from $3,000) |

## Lessons Learned

### 1. Quantization matters more than you think

Our initial indexes were 18 GB. After applying scalar quantization, we got them down to 6 GB with negligible recall loss (<1%). This cut our load times by 3x and reduced memory requirements.

### 2. `nprobe` is your main tuning knob

With IVF indexes, `nprobe` controls how many centroids to search. Higher = better recall, slower search. We found `nprobe=32` (out of 1024 centroids) gave us 95%+ recall at <50ms p99.

### 3. TTL enforcement is non-negotiable

Early on, we had a bug where setups weren't being cleaned up. We woke up to a $400 bill for pods that had been running for days. Aggressive TTL enforcement and alerting are essential.

### 4. Pre-warm for predictable latency

The first few queries after index load are slow (CPU caches cold, memory pages not faulted in). We added a warmup step that runs 100 random queries before marking the setup as "ready."

### 5. Arrow's `release_unused()` prevents memory creep

Without explicit compaction, we observed memory usage growing ~10% per hour under sustained load—the allocator holds onto freed memory for potential reuse. Periodic `release_unused()` calls solved this completely.

### 6. Arrow over Pandas/Dask for cold start

The 15x faster load time (3s vs 45s) was the difference between a viable product and a frustrating one. Arrow's memory-mapped IPC format is purpose-built for this use case.

## Validation

When we built this in August 2023, serverless vector databases weren't really a thing. We wondered if we were over-engineering.

Then the market validated the approach:

- **Milvus** launched a serverless tier
- **Pinecone** introduced serverless pricing
- Startups like [Turbopuffer](https://turbopuffer.com/) and [Upstash Vector](https://upstash.com/) emerged with similar architectures

One of our enterprise customers had been planning to build this internally. After we walked them through our architecture and trade-offs, they adopted our solution instead. We shared our learnings on the pros and cons of this setup, and they were able to get up and running with our offering as part of their current product stack.

The usage and validation from a real customer is something I'm really proud of. When you work on something and see it being used by people, it makes the effort worth it.

## When Should You Consider This Approach?

**This architecture makes sense if:**

- ✅ Your workload is bursty or session-based
- ✅ You can tolerate 30-90 second cold starts
- ✅ You don't need real-time index updates
- ✅ Cost efficiency is a priority

**Stick with a managed solution if:**

- ❌ You need sub-second cold starts
- ❌ You have continuous, high-throughput query traffic
- ❌ You need real-time vector ingestion
- ❌ Operational simplicity outweighs cost savings

## Conclusion

Sometimes the best infrastructure decision is realizing you don't need infrastructure running 24/7. By treating vector search as an on-demand compute problem rather than a database problem, we cut costs by 93% while maintaining the performance our products needed.

The combination of FAISS for vector indexing and Apache Arrow for metadata management gave us the memory efficiency and fast cold starts we needed. Arrow's `release_unused()` was the unsung hero—without it, we'd have been fighting memory creep constantly.

We identified the problem with Milvus costs, designed, built, and integrated this solution into production **within two weeks**. The tight timeline forced us to keep things simple. That constraint turned out to be a feature—simple systems are easier to operate, debug, and extend.

This project also taught me to think bigger—not to constrain ideas to just being a feature within a company, but to consider how they might serve a broader audience. There are now companies building exactly this as their core product.

If you're facing similar challenges, I hope this writeup gives you a useful starting point.

---

*Feel free to reach out if you have questions or want to discuss vector search infrastructure.*

---

[← Back to Blog](/blog) ・ [Home](/) ・ [Contact](/contact)
