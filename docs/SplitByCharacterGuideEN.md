# Complete Guide to Using `split_by_character`

This document explains how to set custom delimiters when working directly with the Python API or the REST service, and
illustrates workflows that rely on Docker shared volumes or a lightweight proxy service.

## 1. Controlling Text Chunking from Python

The `LightRAG` dataclass exposes a `chunking_func` field that defaults to `chunking_by_token_size`. This helper respects
`split_by_character`, `split_by_character_only`, and token-size parameters in order.【F:lightrag/lightrag.py†L71-L116】【F:lightrag/lightrag.py†L1090-L1134】

### 1.1 Synchronous `LightRAG.insert`
```python
from lightrag.lightrag import LightRAG
from lightrag.operate import chunking_by_token_size

# You can also swap in a custom chunking function
rag = LightRAG(
    chunking_func=chunking_by_token_size,
    chunk_token_size=512,
    chunk_overlap_token_size=64,
)

tracking_id = rag.insert(
    input="""
### Heading
Paragraph 1.
### Heading
Paragraph 2.
""",
    split_by_character="###",
    split_by_character_only=True,
    file_paths="notes.md",
)
print("Sync task track ID:", tracking_id)
```
The code first splits the document on `###`; any fragment that still exceeds the token limit is subdivided again. Calling
`insert` spins up an event loop, delegates to `ainsert`, and returns the generated tracking identifier.【F:lightrag/lightrag.py†L1073-L1134】

### 1.2 Asynchronous `LightRAG.ainsert`
```python
import asyncio
from lightrag.lightrag import LightRAG

async def main() -> None:
    rag = LightRAG()
    track_id = await rag.ainsert(
        input=[
            "Paragraph A###Paragraph B",
            "Paragraph C",
        ],
        split_by_character="###",
        split_by_character_only=False,
        ids=["doc-a", "doc-b"],
        file_paths=["doc_a.txt", "doc_b.txt"],
    )
    print("Async task track ID:", track_id)

asyncio.run(main())
```
This snippet shows how to ingest multiple documents at once while attaching custom metadata:

* `input` accepts either a single string or a list of strings. When a list is provided, each entry represents one document.
  The first string (`"Paragraph A###Paragraph B"`) demonstrates that a single document can contain the `###` delimiter,
  while the second string (`"Paragraph C"`) is treated as a separate document. Because `split_by_character_only=False`,
  the pipeline will split at the custom delimiter and still fall back to token-based chunking whenever a chunk is larger
  than the configured size.【F:lightrag/lightrag.py†L71-L116】【F:lightrag/lightrag.py†L1090-L1134】Mixed inputs such as
  `input="One doc"` or `input=["A", "B", "C"]` are normalized internally so you can combine single strings and
  multiple documents.【F:lightrag/lightrag.py†L1243-L1248】
* `ids` and `file_paths` must align with the number of documents in `input`. Setting `ids=["doc-a", "doc-b"]`
  overrides the auto-generated identifiers, making downstream inspection or debugging easier. Likewise,
  `file_paths=["doc_a.txt", "doc_b.txt"]` records file origins inside the document status store. When you ingest a single
  document you can pass `ids="doc-a"` or `file_paths="doc_a.txt"`; both values are normalized to match the input
  list.【F:lightrag/lightrag.py†L1090-L1134】【F:lightrag/lightrag.py†L1243-L1315】【F:lightrag/lightrag.py†L1473-L1548】
* `ainsert` returns a `track_id` after ensuring one is available. It queues every document via
  `apipeline_enqueue_documents`, passing the raw text plus identifiers and file hints; the documents are then processed by
  `apipeline_process_enqueue_documents`, which performs chunking, vectorization, and knowledge-graph updates using the
  supplied parameters.【F:lightrag/lightrag.py†L1094-L1134】【F:lightrag/lightrag.py†L1560-L1603】

The resulting `track_id` helps correlate log entries or check job progress. To review the status later, call
`await rag.aget_docs_by_track_id(track_id)` to obtain a mapping of document IDs to `DocProcessingStatus` records, including
file paths, errors, and the generated chunk list.【F:lightrag/lightrag.py†L3606-L3635】【F:lightrag/kg/json_doc_status_impl.py†L126-L169】

### 1.3 What Happens Behind `ainsert`

`rag.ainsert` does far more than splitting text into separate files; it feeds each document into the full indexing
pipeline. For every document the pipeline:

* Updates `doc_status` with the chunk list, file metadata, and timestamps, enabling reliable audits and follow-up
  processing.【F:lightrag/lightrag.py†L1820-L1833】
* Calls `chunks_vdb.upsert` and `text_chunks.upsert` to compute embeddings and persist chunks inside the vector database and
  the raw text store. Embedding calculations occur inside `chunks_vdb.upsert`, so no extra step is required to trigger
  vector generation.【F:lightrag/lightrag.py†L1834-L1845】【F:lightrag/kg/nano_vector_db_impl.py†L91-L133】
* Runs `_process_extract_entities` to extract entities and relations from each chunk and write them into the knowledge graph
  storage layer.【F:lightrag/lightrag.py†L1852-L1859】【F:lightrag/operate.py†L2740-L2839】

As a result, `ainsert` covers chunking, embedding, vector persistence, and knowledge-graph enrichment in one call.

### 1.4 Example: Pre-split Markdown with `<<BREAK>>`

When you already have a Markdown file (`XYZ.md`) that uses `<<BREAK>>` as a custom separator, the following script performs
splitting, embedding, and storage in one go:

```python
import asyncio
from pathlib import Path
from lightrag.lightrag import LightRAG

async def main() -> None:
    rag = LightRAG()

    # Load the Markdown document that already contains delimiters
    source_path = Path("XYZ.md")
    raw_markdown = source_path.read_text(encoding="utf-8")

    track_id = await rag.ainsert(
        input=raw_markdown,
        split_by_character="<<BREAK>>",
        split_by_character_only=True,
        ids="xyz-md",
        file_paths=str(source_path),
    )
    print("Queued track ID:", track_id)

    # Optional: inspect the status for each document in this batch
    docs = await rag.aget_docs_by_track_id(track_id)
    for doc_id, status in docs.items():
        print(doc_id, status.status, len(status.chunks_list))

asyncio.run(main())
```

The pipeline splits the Markdown content by `<<BREAK>>` and hands each chunk to the same indexing stages described above.
`chunks_vdb.upsert` computes embeddings and stores them in the shared NanoVectorDB files while the raw chunk text and
metadata are written to `text_chunks` and `doc_status` respectively.【F:lightrag/lightrag.py†L1834-L1859】【F:lightrag/kg/nano_vector_db_impl.py†L91-L133】
Afterward `_process_extract_entities` enriches the knowledge graph, making the document searchable through semantic
retrieval and graph queries.【F:lightrag/operate.py†L2740-L2839】 Use `aget_docs_by_track_id` to review chunk identifiers,
processing status, or any reported errors.

## 2. Enabling Custom Delimiters in the REST API

Today the FastAPI routes accept text and file metadata but do not expose `split_by_character`, so the default service cannot
receive custom separators.【F:lightrag/api/routers/document_routes.py†L205-L240】【F:lightrag/api/routers/document_routes.py†L1547-L1570】
To add support without touching the core `lightrag/` package:

1. **Extend the request models.** Add `split_by_character` and `split_by_character_only` (defaulting to `None` and `False`)
   to `InsertTextRequest`, `InsertTextsRequest`, `PipelineIndexTextsRequest`, and any similar schemas.
2. **Propagate the parameters.**
   - Pass the new fields to `pipeline_index_texts` inside each affected route.
   - Update `pipeline_index_texts` so it forwards the values to
     `rag.apipeline_process_enqueue_documents(split_by_character, split_by_character_only)`.
3. **Follow the call chain.** Any background task that calls `pipeline_index_texts` (file uploads, folder scans, etc.)
   should forward the same parameters to keep behavior consistent across ingestion paths.

Once those steps are in place you can send the delimiter inside the HTTP JSON body without modifying the core package.

## 3. Working with Docker and Shared Volumes

If you launch LightRAG with the official `docker compose up` workflow, mount shared volumes to keep storage durable and
exchange files with the host machine:

```bash
docker compose up -d
# Or run manually
# docker run -d \
#   --name lightrag \
#   -p 3000:3000 -p 8000:8000 \
#   -v $PWD/rag_storage:/app/rag_storage \
#   -v $PWD/shared_docs:/data/shared_docs \
#   lightrag:latest
```

* `rag_storage`: stores indexes and caches so data survives container restarts.
* `shared_docs`: a host-visible folder where you can drop files that your pipeline or custom scripts will ingest.

To add a lightweight API without modifying the core codebase, create a small FastAPI or Flask app that shares the same
volume and calls `LightRAG` directly:

```python
# external_gateway.py
from fastapi import FastAPI
from pydantic import BaseModel
from lightrag.lightrag import LightRAG

app = FastAPI()
rag = LightRAG(working_dir="/app/rag_storage")

class InsertPayload(BaseModel):
    text: str
    separator: str | None = None
    strict: bool = False

@app.post("/custom/insert")
async def custom_insert(payload: InsertPayload) -> dict[str, str]:
    track_id = await rag.ainsert(
        input=payload.text,
        split_by_character=payload.separator,
        split_by_character_only=payload.strict,
    )
    return {"track_id": track_id}
```

Place the gateway inside a mounted volume or a companion image, then run it with
`uvicorn external_gateway:app --host 0.0.0.0 --port 8100` to proxy requests into the in-container `LightRAG` instance.
This approach keeps the main package untouched while providing custom ingestion logic.

## 4. Performance Considerations for the API

The FastAPI routes ultimately call `rag.apipeline_enqueue_documents` and `rag.apipeline_process_enqueue_documents`, the same
functions used when you invoke `insert` or `ainsert` from Python. The only extra cost comes from HTTP serialization and
network latency.【F:lightrag/api/routers/document_routes.py†L1547-L1570】【F:lightrag/lightrag.py†L1090-L1134】Given that LLM
inference dominates the runtime of chunk extraction and embedding, the overhead introduced by the API is negligible for
most workloads. For best performance keep the client and API on the same machine or within a low-latency network.
