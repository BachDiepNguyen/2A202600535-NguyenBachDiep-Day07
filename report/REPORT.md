# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Nguyễn Bách Điệp  
**Mã học viên:** 2A202600535  
**Lớp:** E403  
**Nhóm:** F1 - nhóm 1 người  
**Ngày:** 05/06/2026

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**  
Hai text chunks có high cosine similarity nghĩa là vector embedding của chúng gần cùng hướng trong vector space. Về mặt sản phẩm, chúng thường đang nói về cùng chủ đề, cùng intent, hoặc có thể dùng thay nhau làm context cho cùng một câu hỏi.

**Ví dụ HIGH similarity:**
- Sentence A: `A vector store keeps embeddings and metadata for semantic search.`
- Sentence B: `A vector database stores embedding vectors with source metadata.`
- Tại sao tương đồng: Cả hai đều nói về vector store, embeddings, metadata và semantic search.

**Ví dụ LOW similarity:**
- Sentence A: `Vector databases store embeddings for retrieval.`
- Sentence B: `A cake recipe lists flour, eggs, and sugar.`
- Tại sao khác: Một câu nói về retrieval infrastructure, câu còn lại nói về nấu ăn.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**  
Cosine similarity đo góc giữa hai vector nên tập trung vào hướng/nghĩa hơn là độ lớn vector. Với text embeddings, hai câu cùng nghĩa nên gần nhau về hướng ngay cả khi magnitude khác nhau; vì vậy cosine thường ổn định hơn Euclidean distance cho semantic search.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**

Formula:

```text
num_chunks = ceil((doc_length - overlap) / (chunk_size - overlap))
           = ceil((10000 - 50) / (500 - 50))
           = ceil(9950 / 450)
           = 23
```

**Đáp án:** 23 chunks.

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**

```text
num_chunks = ceil((10000 - 100) / (500 - 100))
           = ceil(9900 / 400)
           = 25
```

Overlap tăng làm step nhỏ hơn nên số chunk tăng từ 23 lên 25. Muốn overlap nhiều hơn khi thông tin quan trọng nằm ở ranh giới giữa hai chunk; overlap giúp giữ ngữ cảnh nhưng cũng làm tăng storage và noise nếu dùng quá nhiều.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** Internal Knowledge Assistant / RAG Data Foundations.

Nhóm F1 có 1 thành viên nên em dùng toàn bộ bộ tài liệu mẫu có sẵn trong repo. Domain này phù hợp trực tiếp với nội dung Day 7 vì tài liệu nói về Python, vector store, RAG design, support knowledge assistant, chunking experiment và retrieval tiếng Việt. Bộ dữ liệu đủ nhỏ để kiểm tra thủ công nhưng đủ đa dạng để test metadata filtering, chunking strategy và grounding quality.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | `python_intro.txt` | Repo `data/` | 1,944 | `category=python`, `language=en`, `document_type=intro`, `source=python_intro.txt` |
| 2 | `vector_store_notes.md` | Repo `data/` | 2,123 | `category=vector_store`, `language=en`, `document_type=notes`, `source=vector_store_notes.md` |
| 3 | `rag_system_design.md` | Repo `data/` | 2,391 | `category=rag`, `language=en`, `document_type=design`, `source=rag_system_design.md` |
| 4 | `customer_support_playbook.txt` | Repo `data/` | 1,692 | `category=support`, `language=en`, `document_type=playbook`, `source=customer_support_playbook.txt` |
| 5 | `chunking_experiment_report.md` | Repo `data/` | 1,987 | `category=chunking`, `language=en`, `document_type=experiment`, `source=chunking_experiment_report.md` |
| 6 | `vi_retrieval_notes.md` | Repo `data/` | 1,667 | `category=retrieval`, `language=vi`, `document_type=notes`, `source=vi_retrieval_notes.md` |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| `source` | string | `vector_store_notes.md` | Truy vết nguồn và hiển thị/cite chunk chính xác. |
| `category` | string | `support`, `rag`, `chunking` | Cho phép filter trước search để tránh lẫn domain. |
| `language` | string | `en`, `vi` | Hữu ích cho query tiếng Việt hoặc song ngữ. |
| `document_type` | string | `notes`, `design`, `playbook` | Giúp phân biệt notes, design doc, playbook, experiment report. |
| `chunk_index` | int | `0`, `1`, `2` | Debug chunk nào được retrieve và tái hiện kết quả. |
| `strategy` | string | `recursive_650` | Ghi lại strategy dùng khi index để so sánh/eval. |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` với `chunk_size=650` trên 3 tài liệu đại diện:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|-----------|----------|-------------|------------|-------------------|
| `vector_store_notes.md` | FixedSizeChunker (`fixed_size`) | 4 | 579.5 | Trung bình; có thể cắt giữa paragraph. |
| `vector_store_notes.md` | SentenceChunker (`by_sentences`) | 10 | 210.2 | Tốt ở mức câu nhưng hơi nhỏ. |
| `vector_store_notes.md` | RecursiveChunker (`recursive`) | 4 | 529.0 | Tốt; ưu tiên paragraph/line trước. |
| `rag_system_design.md` | FixedSizeChunker (`fixed_size`) | 4 | 646.5 | Chunk dài, có nguy cơ gộp nhiều ý. |
| `rag_system_design.md` | SentenceChunker (`by_sentences`) | 7 | 338.9 | Dễ đọc nhưng tách section design hơi vụn. |
| `rag_system_design.md` | RecursiveChunker (`recursive`) | 6 | 396.7 | Cân bằng giữa section và kích thước. |
| `chunking_experiment_report.md` | FixedSizeChunker (`fixed_size`) | 4 | 545.5 | Dự đoán được count nhưng boundary cơ học. |
| `chunking_experiment_report.md` | SentenceChunker (`by_sentences`) | 7 | 281.4 | Coherent theo câu nhưng context ngắn. |
| `chunking_experiment_report.md` | RecursiveChunker (`recursive`) | 5 | 395.6 | Giữ ý theo section tốt nhất. |

### Strategy Của Tôi

**Loại:** `RecursiveChunker(chunk_size=650)`.

**Mô tả cách hoạt động:**  
Strategy này thử tách theo separator lớn đến nhỏ: paragraph (`\n\n`), line (`\n`), câu (`. `), space, rồi mới fallback fixed-size nếu không còn separator phù hợp. Khi một piece vượt `chunk_size`, algorithm đệ quy xuống separator nhỏ hơn. Khi các piece nhỏ hơn giới hạn, strategy gom lại để tạo chunk không quá dài.

**Tại sao tôi chọn strategy này cho domain nhóm?**  
Bộ tài liệu là mixed markdown/text: có heading, paragraph, bullet-like sections và cả tiếng Việt. Recursive chunking hợp vì nó tận dụng structure tự nhiên nếu có, nhưng vẫn fallback được khi gặp đoạn dài. Điều này bám đúng slide Day 7: bắt đầu bằng section/structure, tối ưu bằng eval thay vì cắt bừa theo số ký tự.

**Code snippet (core idea):**

```python
class RecursiveChunker:
    DEFAULT_SEPARATORS = ["\n\n", "\n", ". ", " ", ""]

    def chunk(self, text: str) -> list[str]:
        if not text:
            return []
        return [chunk for chunk in self._split(text.strip(), self.separators) if chunk]
```

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|-----------|----------|-------------|------------|--------------------|
| Mixed docs | FixedSizeChunker | 4 chunks/doc điển hình | 545-646 | Dễ implement nhưng có thể cắt giữa ý. |
| Mixed docs | SentenceChunker | 7-10 chunks/doc điển hình | 210-339 | Rất readable nhưng nhiều chunk nhỏ, dễ thiếu context. |
| Mixed docs | **RecursiveChunker của tôi** | 4-6 chunks/doc điển hình | 396-529 | Cân bằng nhất: context đủ dài, boundary hợp lý, top-3 hit 5/5 benchmark. |

### So Sánh Với Thành Viên Khác

Nhóm F1 có 1 thành viên, nên em so sánh 3 strategy variants như 3 baseline nội bộ thay cho so sánh giữa nhiều thành viên.

| Thành viên / Variant | Strategy | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Tôi | RecursiveChunker `chunk_size=650` | 10/10 | Giữ paragraph/section tốt, top-3 hit 5/5. | Phức tạp hơn fixed-size, cần giải thích recursion. |
| Variant A | FixedSizeChunker | 7/10 ước lượng | Đơn giản, count ổn định. | Cắt giữa câu/ý, context đôi khi nhiễu. |
| Variant B | SentenceChunker | 8/10 ước lượng | Chunk rất dễ đọc, không cắt câu. | Một số answer cần nhiều câu/section nên chunk hơi ngắn. |

**Strategy nào tốt nhất cho domain này? Tại sao?**  
Recursive chunking tốt nhất cho bộ tài liệu này vì data có cấu trúc paragraph/heading nhưng không đồng nhất. Nó giữ được ý trọn vẹn hơn fixed-size và ít vụn hơn sentence-based.

---

## 4. My Approach — Cá nhân (10 điểm)

### Chunking Functions

**`SentenceChunker.chunk` — approach:**  
Em dùng regex `(?<=[.!?])\s+|\n+` để tách sau dấu câu hoặc newline, sau đó strip whitespace và gom tối đa `max_sentences_per_chunk` câu vào một chunk. Edge cases: text rỗng trả `[]`, nếu không detect được sentence thì trả text đã strip.

**`RecursiveChunker.chunk` / `_split` — approach:**  
Algorithm dùng danh sách separator theo ưu tiên lớn đến nhỏ. Base case là text rỗng hoặc text đã ngắn hơn `chunk_size`. Nếu một piece vẫn quá dài, `_split` gọi lại chính nó với separator nhỏ hơn; nếu hết separator thì fallback sang `FixedSizeChunker` không overlap.

### EmbeddingStore

**`add_documents` + `search` — approach:**  
Mỗi `Document` được chuẩn hóa thành record gồm `id`, `doc_id`, `content`, `metadata`, `embedding`, và `index`. `add_documents` embed từng document/chunk và lưu in-memory. `search` embed query rồi tính dot product với từng record, sort score giảm dần và trả top-k.

**`search_with_filter` + `delete_document` — approach:**  
`search_with_filter` lọc metadata trước rồi mới similarity search, đúng yêu cầu lab vì filter sau search có thể bỏ mất candidate tốt. `delete_document` xóa tất cả records có `doc_id` trùng trong metadata hoặc field record, và trả `True/False` theo việc có xóa được gì không.

### KnowledgeBaseAgent

**`answer` — approach:**  
Agent gọi `store.search(question, top_k)`, format retrieved chunks thành prompt có source và score, rồi gọi `llm_fn(prompt)`. Prompt yêu cầu chỉ trả lời từ retrieved context và nói rõ nếu context không đủ. Đây là RAG pattern tối thiểu: retrieve → inject → generate.

### Test Results

```text
pytest tests/ -v
============================= test session starts =============================
collected 42 items
...
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_true_for_existing_doc PASSED [100%]
============================= 42 passed in 0.34s ==============================
```

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

Embedding dùng trong lab là mock lexical hashing deterministic, không gọi API ngoài. Em dự đoán trước bằng mức độ overlap chủ đề/từ khóa, sau đó đo bằng `compute_similarity(_mock_embed(a), _mock_embed(b))`.

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | Vector stores keep embeddings and metadata for semantic search. | A vector database stores embedding vectors with source metadata. | high | 0.378 | Đúng |
| 2 | Python is used for data analysis and machine learning. | Teams use Python for machine learning and data workflows. | high | 0.704 | Đúng |
| 3 | Chunk overlap preserves context between neighboring chunks. | Overlap helps keep meaning at chunk boundaries. | high | 0.298 | Đúng một phần |
| 4 | Vector databases store embeddings for retrieval. | A cake recipe lists flour, eggs, and sugar. | low | -0.333 | Đúng |
| 5 | Customer support articles need escalation criteria. | Brown bears live in northern forests. | low | 0.000 | Đúng |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**  
Pair 3 có score chỉ 0.298 dù về nghĩa khá gần. Lý do là mock embedder hiện dựa nhiều vào token overlap/hashing, nên nó chưa hiểu paraphrase sâu như embedding model thật. Điều này nhắc rằng embedding backend ảnh hưởng retrieval quality, nhưng lab vẫn dùng mock để deterministic và tập trung vào pipeline.

---

## 6. Results — Cá nhân (10 điểm)

Chạy 5 benchmark queries trên implementation cá nhân với strategy `RecursiveChunker(chunk_size=650)`. Một số query dùng metadata filter để minh họa filter trước retrieval.

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | What are the four stages of a vector search pipeline? | Chunk documents, embed chunks, store vector+metadata, embed query and rank by similarity. |
| 2 | Why can metadata filtering improve retrieval precision for a support assistant? | Metadata separates customer-facing, support-only, and engineering-only docs, preventing wrong-audience context. |
| 3 | What should a RAG assistant do when retrieved evidence is weak or contradictory? | It should say evidence is insufficient instead of pretending the answer is complete. |
| 4 | Which chunking strategy gave the best balance for mixed technical documentation? | Recursive chunking gave the best balance by splitting larger structures first then falling back to smaller separators. |
| 5 | Metadata giúp retrieval tiếng Việt tránh lấy nhầm tài liệu như thế nào? | Metadata như language/category giúp filter đúng tài liệu tiếng Việt hoặc đúng phòng ban, tránh lấy nhầm tài liệu marketing/English/khác domain. |

### Kết Quả Của Tôi

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | What are the four stages of a vector search pipeline? | `vector_store_notes.md`: vector store notes + typical workflow | 0.433 | Yes | Vector search pipeline gồm chunk, embed, store metadata/vector, embed query và rank. |
| 2 | Why can metadata filtering improve retrieval precision for a support assistant? | `customer_support_playbook.txt`: retrieval insufficient/escalation | 0.362 | Yes | Support assistant nên dùng metadata để phân biệt customer-facing/internal/engineering docs và escalate khi thiếu tài liệu. |
| 3 | What should a RAG assistant do when retrieved evidence is weak or contradictory? | `rag_system_design.md`: assistant grounded in internal docs | 0.471 | Yes | Nếu context yếu hoặc mâu thuẫn, assistant phải nói rõ không đủ bằng chứng thay vì bịa. |
| 4 | Which chunking strategy gave the best balance for mixed technical documentation? | `chunking_experiment_report.md`: recursive strong default | 0.515 | Yes | Recursive chunking là default mạnh cho mixed technical docs vì giữ context tốt và vẫn kiểm soát size. |
| 5 | Metadata giúp retrieval tiếng Việt tránh lấy nhầm tài liệu như thế nào? | `vi_retrieval_notes.md`: failure cases and metadata notes | 0.218 | Yes | Filter `language=vi` giúp search trong tài liệu tiếng Việt, tránh lẫn tài liệu khác ngôn ngữ/domain. |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 5 / 5

### Notes On Filters

- Query 2 dùng `metadata_filter={"category": "support"}`.
- Query 3 dùng `metadata_filter={"category": "rag"}`.
- Query 4 dùng `metadata_filter={"category": "chunking"}`.
- Query 5 dùng `metadata_filter={"language": "vi"}`.

Filter giúp giảm noise rõ rệt vì dataset có nhiều tài liệu cùng nói về retrieval nhưng khác mục đích.

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**  
Nhóm F1 chỉ có 1 thành viên, nên em học bằng cách so sánh các strategy variants với nhau. Điểm quan trọng là cùng một data nhưng boundary chunk khác nhau tạo ra retrieval behavior khác nhau; fixed-size dễ chạy nhưng không phải lúc nào cũng tốt cho grounding.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**  
Điểm em sẽ quan sát khi nghe demo nhóm khác là metadata schema của họ. Nếu họ chọn metadata sát workflow hơn, ví dụ `audience`, `freshness`, `access_level`, retrieval sẽ dễ filter đúng hơn so với chỉ có `source`.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**  
Em sẽ thêm vài tài liệu nhiễu cùng keyword nhưng khác category để test metadata filtering khó hơn. Hiện dataset sạch nên top-3 hit tốt; trong production cần failure cases mạnh hơn như stale docs, duplicate docs, hoặc query mơ hồ.

### Failure Analysis

**Failure case:** Query tiếng Việt `Metadata giúp retrieval tiếng Việt tránh lấy nhầm tài liệu như thế nào?` có top-1 đúng source nhưng score chỉ 0.218, thấp hơn các query tiếng Anh.  
**Tại sao:** Mock embedder dựa nhiều vào token overlap và không có multilingual semantic understanding như embedding model thật. Ngoài ra query tiếng Việt có thể dùng cách diễn đạt khác với tài liệu.  
**Cải thiện:** Dùng OpenAI/local multilingual embedding thật cho production, thêm metadata filter `language=vi`, và viết thêm eval queries tiếng Việt để đo riêng retrieval quality song ngữ.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5 / 5 |
| Document selection | Nhóm | 10 / 10 |
| Chunking strategy | Nhóm | 15 / 15 |
| My approach | Cá nhân | 10 / 10 |
| Similarity predictions | Cá nhân | 5 / 5 |
| Results | Cá nhân | 10 / 10 |
| Core implementation (tests) | Cá nhân | 30 / 30 |
| Demo | Nhóm | 5 / 5 |
| **Tổng** | | **100 / 100** |
