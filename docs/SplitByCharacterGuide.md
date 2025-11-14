# 使用 `split_by_character` 的完整教學

本文說明如何在程式端與 API 層指定特殊分隔符，並示範以 Docker 分享卷或外掛小型 API 服務的整合方式。

## 1. 在 Python 端控制文本切分

`LightRAG` 的 `chunking_func` 欄位預設使用 `chunking_by_token_size`，會依序處理 `split_by_character`、`split_by_character_only` 與 `chunk_token_size` 相關參數。【F:lightrag/lightrag.py†L71-L116】【F:lightrag/lightrag.py†L1090-L1134】

### 1.1 `LightRAG.insert` 同步範例
```python
from lightrag.lightrag import LightRAG
from lightrag.operate import chunking_by_token_size

# 也可以改成自訂 chunking_func
rag = LightRAG(
    chunking_func=chunking_by_token_size,
    chunk_token_size=512,
    chunk_overlap_token_size=64,
)

tracking_id = rag.insert(
    input="""
### 題目
第 1 段。
### 題目
第 2 段。
""",
    split_by_character="###",
    split_by_character_only=True,
    file_paths="notes.md",
)
print("同步任務 Track ID:", tracking_id)
```
上述程式會先以 `###` 切出兩個段落，若段落仍超過 token 上限才會再次切分。呼叫 `insert` 會自動建立事件迴圈並委派至 `ainsert`，最後回傳追蹤編號。【F:lightrag/lightrag.py†L1073-L1134】

### 1.2 `LightRAG.ainsert` 非同步範例
```python
import asyncio
from lightrag.lightrag import LightRAG

async def main() -> None:
    rag = LightRAG()
    track_id = await rag.ainsert(
        input=[
            "段落 A###段落 B",
            "段落 C",
        ],
        split_by_character="###",
        split_by_character_only=False,
        ids=["doc-a", "doc-b"],
        file_paths=["doc_a.txt", "doc_b.txt"],
    )
    print("非同步任務 Track ID:", track_id)

asyncio.run(main())
```
這段程式碼示範一次插入多個文件並附帶額外資訊的完整流程：

* `input` 可為字串或字串列表。傳入列表時，每個元素被視為獨立文件。第一筆 `"段落 A###段落 B"` 展示單一文件可由 `###` 拆成多段；第二筆 `"段落 C"` 是另一份文件。由於 `split_by_character_only=False`，系統除了遇到 `###` 會切段外，若有段落仍超過 token 上限，會再依 token 數拆分。【F:lightrag/lightrag.py†L71-L116】【F:lightrag/lightrag.py†L1090-L1134】如果要混合單一字串與多個文件，也可以傳入 `input="一份文件"` 或 `input=["A", "B", "C"]` 兩種形態，系統會自動標準化為列表後處理。【F:lightrag/lightrag.py†L1243-L1248】
* `ids` 與 `file_paths` 的長度需與 `input` 對齊。`ids=["doc-a", "doc-b"]` 會覆寫自動生成的文件 ID，讓後續查詢或除錯時更容易定位來源；`file_paths=["doc_a.txt", "doc_b.txt"]` 會寫入文件狀態的 `file_path` 欄位，便於還原實際檔案或整合現有檔案管理系統。若僅插入單一文件，也可以傳 `ids="doc-a"` 或 `file_paths="doc_a.txt"`，在內部會轉成與 `input` 對應的列表。【F:lightrag/lightrag.py†L1090-L1134】【F:lightrag/lightrag.py†L1243-L1315】【F:lightrag/lightrag.py†L1473-L1548】
* `ainsert` 取得輸入後會回傳 `track_id`。流程包含：若未指定，先建立新的追蹤編號；呼叫 `apipeline_enqueue_documents` 將每筆原文（連同 `id`、`file_path` 等資訊）排入待處理佇列；最後由 `apipeline_process_enqueue_documents` 依指定參數進行分段、向量化及知識圖譜更新。【F:lightrag/lightrag.py†L1094-L1134】【F:lightrag/lightrag.py†L1560-L1603】

印出的 `track_id` 可用來查閱背景任務的執行狀態或與日誌進行對照。

若要追蹤處理進度，可在稍後呼叫 `await rag.aget_docs_by_track_id(track_id)` 取得所有同批文件的狀態與產出段落，回傳值以文件 ID 為鍵、`DocProcessingStatus` 為內容，會連同 `file_path`、錯誤訊息等資訊一併提供。【F:lightrag/lightrag.py†L3606-L3635】【F:lightrag/kg/json_doc_status_impl.py†L126-L169】

### 1.3 `ainsert` 背後的處理流程

`rag.ainsert` 不只是把原始文字拆成多個檔案；它會把文件排入完整的索引管線。每份文件會先被切成 chunk，然後平行地：

* 更新 `doc_status`，記錄 chunk 清單、來源檔案與處理時間戳記，方便後續追蹤。【F:lightrag/lightrag.py†L1820-L1833】
* 呼叫 `chunks_vdb.upsert` 與 `text_chunks.upsert`，把 chunk 內容與中繼資料儲存到向量資料庫與文字儲存層。`chunks_vdb` 在 `upsert` 時會批次執行 embedding 計算，再將結果寫入儲存檔案，因此不需要額外手動觸發向量化。【F:lightrag/lightrag.py†L1834-L1845】【F:lightrag/kg/nano_vector_db_impl.py†L91-L133】
* 等待上述動作完成後，再執行 `_process_extract_entities`，透過 LLM 從 chunk 中擷取實體與關係並寫回知識圖譜儲存體，完成整個索引流程。【F:lightrag/lightrag.py†L1852-L1859】【F:lightrag/operate.py†L2740-L2839】

因此，`ainsert` 會完成分段、embedding、向量儲存與知識圖譜更新，而非僅僅把文本拆成兩個檔案。

### 1.4 已預先插入分隔符的 Markdown 文件範例

若手邊已有一份 `XYZ.md`，內容使用 `<<BREAK>>` 作為段落分隔，依下列方式即可一次完成切分、embedding 與資料庫寫入：

```python
import asyncio
from pathlib import Path
from lightrag.lightrag import LightRAG

async def main() -> None:
    rag = LightRAG()

    # 讀取已含有分隔符的 Markdown 文件
    source_path = Path("XYZ.md")
    raw_markdown = source_path.read_text(encoding="utf-8")

    track_id = await rag.ainsert(
        input=raw_markdown,
        split_by_character="<<BREAK>>",
        split_by_character_only=True,
        ids="xyz-md",
        file_paths=str(source_path),
    )
    print("排程編號:", track_id)

    # 可選：查詢本次任務的每個文件狀態與 chunk 細節
    docs = await rag.aget_docs_by_track_id(track_id)
    for doc_id, status in docs.items():
        print(doc_id, status.status, len(status.chunks_list))

asyncio.run(main())
```

程式會把 `XYZ.md` 的段落依 `<<BREAK>>` 分割，並交由同一套索引管線處理。每個 chunk 在 `chunks_vdb.upsert` 時會自動計算 embedding、儲存在共享的 NanoVectorDB 檔案裡，同時寫入 `text_chunks` 與 `doc_status` 以便追蹤來源與後續查詢。【F:lightrag/lightrag.py†L1834-L1859】【F:lightrag/kg/nano_vector_db_impl.py†L91-L133】之後的 `_process_extract_entities` 也會針對這些 chunk 執行知識圖譜抽取，確保文件在資料庫中可供語意檢索與圖譜分析。【F:lightrag/operate.py†L2740-L2839】如果需要檢查成果，可以藉由 `aget_docs_by_track_id` 取得 chunk ID、處理狀態或錯誤訊息。

## 2. 讓 REST API 支援特殊分隔符

FastAPI 路由目前只接收文字與來源資訊，沒有暴露 `split_by_character`，因此透過現成 API 無法指定特殊分隔符。【F:lightrag/api/routers/document_routes.py†L205-L240】【F:lightrag/api/routers/document_routes.py†L1547-L1570】要在不更動核心模組的前提下加入支援，可依下列步驟修改 API 專案：

1. **擴充請求模型**：在 `InsertTextRequest`、`InsertTextsRequest`、`PipelineIndexTextsRequest` 等模型新增 `split_by_character` 與 `split_by_character_only` 欄位（預設 `None` 與 `False`）。
2. **向下傳遞參數**：
   - 在對應的路由函式中，將新欄位傳給 `pipeline_index_texts`。
   - 調整 `pipeline_index_texts` 讓它呼叫 `rag.apipeline_process_enqueue_documents(split_by_character, split_by_character_only)`。
3. **回溯既有背景任務**：任何呼叫 `pipeline_index_texts` 的背景任務（例如上傳檔案或掃描資料夾）都需傳遞一致的參數，以免資料來源行為不一致。

完成後即可透過 HTTP JSON Body 指定特殊分隔符，不須改動 `lightrag/` 目錄下的核心程式碼。

## 3. Docker + 分享卷的使用方式

若你以官方 `docker compose up` 啟動 LightRAG，可以利用 Docker Volume 與主機分享檔案：

```bash
docker compose up -d
# 或自行執行
# docker run -d \
#   --name lightrag \
#   -p 3000:3000 -p 8000:8000 \
#   -v $PWD/rag_storage:/app/rag_storage \
#   -v $PWD/shared_docs:/data/shared_docs \
#   lightrag:latest
```

* `rag_storage`：保留索引與快取資料，以避免容器重啟後遺失。
* `shared_docs`：與主機共用的資料夾，讓資料管線（或自訂腳本）可以把檔案放入，並由容器內的掃描程序或 API 索引。

若要外掛小型 API 而不改動核心程式，可額外撰寫一個 FastAPI / Flask 應用，掛載同一個共享卷並呼叫 `LightRAG`：

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
將此檔案放在掛載卷或額外的程式碼目錄中，配合 `uvicorn external_gateway:app --host 0.0.0.0 --port 8100` 即可對內部 `LightRAG` 進行調用，而不需修改核心套件。

## 4. API 調用的效能考量

FastAPI 路徑最終仍呼叫 `rag.apipeline_enqueue_documents` 與 `rag.apipeline_process_enqueue_documents`，與直接在程式中呼叫 `insert` / `ainsert` 是相同邏輯，因此主要額外成本只來自 HTTP 序列化與網路傳輸延遲。【F:lightrag/api/routers/document_routes.py†L1547-L1570】【F:lightrag/lightrag.py†L1090-L1134】對於以 LLM 執行抽取與向量化為主的工作流程，這類額外開銷通常遠小於模型推論時間，效能影響可以忽略。如果需要最佳化，可在 API 與客戶端間使用同一台主機或內部網路以降低往返時間。
