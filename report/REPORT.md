# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Bùi Văn Tuân
**Nhóm:** C1006  
**Ngày:** 2026-06-05

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**  
Hai text chunks có cosine similarity cao nghĩa là vector embedding của chúng gần cùng hướng, tức là nội dung có liên quan về mặt ngữ nghĩa. Trong retrieval, chunk có điểm cao thường là ứng viên tốt để đưa vào context cho LLM.

**Ví dụ HIGH similarity:**

- Sentence A: Python is used for data analysis and machine learning.
- Sentence B: Data scientists use Python to clean data and train models.
- Tại sao tương đồng: Cả hai đều nói về Python trong data science/machine learning.

**Ví dụ LOW similarity:**

- Sentence A: Vector databases store embeddings for similarity search.
- Sentence B: The customer changed their billing address yesterday.
- Tại sao khác: Một câu nói về vector store, câu còn lại nói về thông tin billing của khách hàng.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**  
Cosine similarity tập trung vào hướng của vector hơn là độ lớn tuyệt đối, nên phù hợp để đo mức liên quan ngữ nghĩa giữa embeddings. Với embeddings đã normalize, dot product/cosine similarity cũng dễ tính và thường cho ranking ổn định.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**  
Formula: `ceil((doc_length - overlap) / (chunk_size - overlap))`  
Tính: `ceil((10000 - 50) / (500 - 50)) = ceil(9950 / 450) = ceil(22.11) = 23`  
Đáp án: **23 chunks**

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**  
Tính: `ceil((10000 - 100) / (500 - 100)) = ceil(9900 / 400) = 25`, nên số chunks tăng từ 23 lên 25. Overlap nhiều hơn giúp giữ context qua ranh giới chunk, nhưng tốn thêm storage và thời gian search.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** Knowledge assistant cho data foundations, retrieval, RAG và customer support.

Nhóm chọn domain này vì các tài liệu có nội dung rõ ràng, gần với mục tiêu lab và dễ tạo benchmark queries có gold answers. Bộ tài liệu cũng có cả tiếng Anh và tiếng Việt, giúp kiểm tra metadata như `language`, `category`, và tác động của filtering.

### Data Inventory

| #   | Tên tài liệu                    | Nguồn                                | Số ký tự | Metadata đã gán                                                                 |
| --- | ------------------------------- | ------------------------------------ | -------- | ------------------------------------------------------------------------------- |
| 1   | `python_intro.txt`              | `data/python_intro.txt`              | 1944     | `category=programming`, `language=en`, `source=data/python_intro.txt`           |
| 2   | `vector_store_notes.md`         | `data/vector_store_notes.md`         | 2123     | `category=vector_store`, `language=en`, `source=data/vector_store_notes.md`     |
| 3   | `rag_system_design.md`          | `data/rag_system_design.md`          | 2391     | `category=rag`, `language=en`, `source=data/rag_system_design.md`               |
| 4   | `customer_support_playbook.txt` | `data/customer_support_playbook.txt` | 1692     | `category=support`, `language=en`, `source=data/customer_support_playbook.txt`  |
| 5   | `chunking_experiment_report.md` | `data/chunking_experiment_report.md` | 1987     | `category=chunking`, `language=en`, `source=data/chunking_experiment_report.md` |
| 6   | `vi_retrieval_notes.md`         | `data/vi_retrieval_notes.md`         | 1667     | `category=retrieval`, `language=vi`, `source=data/vi_retrieval_notes.md`        |

### Metadata Schema

| Trường metadata | Kiểu   | Ví dụ giá trị                | Tại sao hữu ích cho retrieval?                              |
| --------------- | ------ | ---------------------------- | ----------------------------------------------------------- |
| `category`      | string | `rag`, `support`, `chunking` | Cho phép filter theo domain khi query có chủ đề rõ.         |
| `language`      | string | `en`, `vi`                   | Giúp query tiếng Việt không bị trộn với tài liệu tiếng Anh. |
| `source`        | string | `data/rag_system_design.md`  | Giúp trace kết quả về tài liệu gốc.                         |
| `chunk_index`   | int    | `0`, `1`, `2`                | Giúp debug chunk nào trong tài liệu được retrieve.          |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` trên 3 tài liệu với `chunk_size=500`:

| Tài liệu                | Strategy                         | Chunk Count | Avg Length | Preserves Context?              |
| ----------------------- | -------------------------------- | ----------- | ---------- | ------------------------------- |
| `python_intro.txt`      | FixedSizeChunker (`fixed_size`)  | 4           | 486.0      | Trung bình, có thể cắt giữa câu |
| `python_intro.txt`      | SentenceChunker (`by_sentences`) | 5           | 387.0      | Tốt, giữ câu nguyên vẹn         |
| `python_intro.txt`      | RecursiveChunker (`recursive`)   | 5           | 387.0      | Tốt, ưu tiên boundary tự nhiên  |
| `vector_store_notes.md` | FixedSizeChunker (`fixed_size`)  | 5           | 424.6      | Trung bình                      |
| `vector_store_notes.md` | SentenceChunker (`by_sentences`) | 8           | 263.6      | Tốt nhưng hơi nhỏ               |
| `vector_store_notes.md` | RecursiveChunker (`recursive`)   | 7           | 301.4      | Tốt, cân bằng size/context      |
| `rag_system_design.md`  | FixedSizeChunker (`fixed_size`)  | 5           | 478.2      | Trung bình                      |
| `rag_system_design.md`  | SentenceChunker (`by_sentences`) | 5           | 441.4      | Tốt                             |
| `rag_system_design.md`  | RecursiveChunker (`recursive`)   | 7           | 339.7      | Tốt nhất cho markdown section   |

### Strategy Của Tôi

**Loại:** `RecursiveChunker(chunk_size=500)`

**Mô tả cách hoạt động:**  
Strategy này thử tách văn bản theo separator lớn trước như đoạn văn `\n\n`, sau đó mới dùng newline, câu, khoảng trắng và cuối cùng fallback sang fixed-size. Nếu một phần sau khi split vẫn quá dài, hàm `_split` tiếp tục recurse với separator tiếp theo. Cách này giúp chunk không quá dài nhưng vẫn giữ được cấu trúc tự nhiên của tài liệu markdown/text.

**Tại sao tôi chọn strategy này cho domain nhóm?**  
Tài liệu nhóm chủ yếu là markdown và notes có section/paragraph rõ ràng. `RecursiveChunker` phù hợp vì nó tận dụng boundary tự nhiên thay vì cắt cứng theo ký tự, đồng thời tạo chunk nhỏ vừa đủ để retrieval không bị nhiễu quá nhiều.

**Code snippet (nếu custom):**

```python
chunker = RecursiveChunker(chunk_size=500)
chunks = chunker.chunk(document_text)
```

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu   | Strategy              | Chunk Count | Avg Length | Retrieval Quality?                         |
| ---------- | --------------------- | ----------- | ---------- | ------------------------------------------ |
| All 6 docs | FixedSize baseline    | 29          | 446.7      | Dễ implement, nhưng đôi khi cắt giữa ý     |
| All 6 docs | Sentence baseline     | 25          | 457.9      | Dễ đọc, nhưng có chunk hơi dài khi câu dài |
| All 6 docs | **Recursive của tôi** | 34          | 345.4      | Cân bằng nhất giữa readability và context  |

### So Sánh Với Thành Viên Khác

| Thành viên | Strategy                           | Retrieval Score (/10) | Điểm mạnh                                         | Điểm yếu                     |
| ---------- | ---------------------------------- | --------------------- | ------------------------------------------------- | ---------------------------- |
| Tôi        | RecursiveChunker + metadata filter | 8                     | Chunk coherent, filter theo category/language tốt | Phụ thuộc metadata đúng      |
| Bạn Dũng   | FixedSizeChunker overlap 50        | 6                     | Đơn giản, chunk count ổn định                     | Dễ cắt giữa câu/section      |
| Bạn Thắng  | SentenceChunker 4 câu/chunk        | 7                     | Chunk dễ đọc, ít cắt câu                          | Một số chunk quá rộng chủ đề |

**Strategy nào tốt nhất cho domain này? Tại sao?**  
`RecursiveChunker` tốt nhất cho domain này vì tài liệu có paragraph và markdown sections rõ. Khi kết hợp với metadata filter, top-3 thường chứa chunk đúng tài liệu, trong khi search thuần bằng `_mock_embed` dễ nhiễu vì mock embedding không hiểu nghĩa thật.

---

## 4. My Approach — Cá nhân (10 điểm)

### Chunking Functions

**`SentenceChunker.chunk` — approach:**  
Tôi dùng regex để tách câu theo dấu `.`, `!`, `?` khi theo sau là whitespace hoặc hết chuỗi. Sau đó gom tối đa `max_sentences_per_chunk` câu vào một chunk và strip whitespace để output sạch hơn.

**`RecursiveChunker.chunk` / `_split` — approach:**  
Base case là text đã ngắn hơn `chunk_size` thì trả về ngay. Nếu còn quá dài, thuật toán thử separator theo thứ tự ưu tiên; phần nào vẫn quá dài sẽ recurse với separator tiếp theo, và cuối cùng fallback về `FixedSizeChunker`.

### EmbeddingStore

**`add_documents` + `search` — approach:**  
Mỗi `Document` được normalize thành record gồm `id`, `content`, `metadata`, và `embedding`. Search embed query bằng cùng embedding function, tính dot product với từng record, sort score giảm dần và trả về top-k.

**`search_with_filter` + `delete_document` — approach:**  
`search_with_filter` filter metadata trước rồi mới tính similarity để giảm nhiễu và tăng precision. `delete_document` xóa tất cả record có `metadata["doc_id"]` trùng với document id cần xóa.

### KnowledgeBaseAgent

**`answer` — approach:**  
Agent retrieve top-k chunks từ store, format chúng thành context blocks có source và content, rồi inject vào prompt. Prompt yêu cầu LLM trả lời chỉ dựa trên retrieved context để giữ grounding.

### Test Results

```text
============================= test session starts ==============================
platform darwin -- Python 3.11.11, pytest-9.0.3, pluggy-1.6.0
collected 42 items
tests/test_solution.py::TestProjectStructure::test_root_main_entrypoint_exists PASSED
tests/test_solution.py::TestProjectStructure::test_src_package_exists PASSED
tests/test_solution.py::TestClassBasedInterfaces::test_chunker_classes_exist PASSED
tests/test_solution.py::TestClassBasedInterfaces::test_mock_embedder_exists PASSED
tests/test_solution.py::TestFixedSizeChunker::* PASSED
tests/test_solution.py::TestSentenceChunker::* PASSED
tests/test_solution.py::TestRecursiveChunker::* PASSED
tests/test_solution.py::TestEmbeddingStore::* PASSED
tests/test_solution.py::TestKnowledgeBaseAgent::* PASSED
tests/test_solution.py::TestComputeSimilarity::* PASSED
tests/test_solution.py::TestCompareChunkingStrategies::* PASSED
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::* PASSED
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::* PASSED
============================== 42 passed in 0.02s ==============================
```

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

Các actual scores dưới đây dùng `_mock_embed` để đúng môi trường lab mặc định.

| Pair | Sentence A                                            | Sentence B                                               | Dự đoán | Actual Score | Đúng?    |
| ---- | ----------------------------------------------------- | -------------------------------------------------------- | ------- | ------------ | -------- |
| 1    | Python is a programming language.                     | Python is used to write software.                        | high    | 0.050        | Một phần |
| 2    | Vector stores retrieve similar embeddings.            | A database can search vectors by similarity.             | high    | -0.115       | Không    |
| 3    | Password reset instructions are in the support guide. | The customer support playbook explains account recovery. | high    | -0.236       | Không    |
| 4    | Vietnamese retrieval requires language metadata.      | Bánh mì is a popular Vietnamese food.                    | low     | -0.074       | Có       |
| 5    | Chunk overlap preserves context between chunks.       | The invoice was paid yesterday.                          | low     | 0.002        | Có       |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**  
Kết quả bất ngờ nhất là pair 2 và pair 3 có nghĩa khá gần nhưng score lại âm. Nguyên nhân là lab đang dùng `_mock_embed`, đây là embedding deterministic để test code chứ không phải semantic embedding thật, nên nó không biểu diễn nghĩa như OpenAI hoặc sentence-transformers.

---

## 6. Results — Cá nhân (10 điểm)

Chạy 5 benchmark queries của nhóm trên implementation cá nhân. Strategy dùng `RecursiveChunker(chunk_size=500)` và metadata filter khi query có domain/language rõ.

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| #   | Query                                                        | Gold Answer                                                                                               |
| --- | ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------- |
| 1   | What does a vector store keep and retrieve?                  | Vector store lưu embeddings và metadata/documents, rồi retrieve chunks gần query nhất theo similarity.    |
| 2   | What are the main steps in a RAG pipeline?                   | Ingest/chunk documents, embed and store chunks, retrieve top chunks, build prompt, call LLM.              |
| 3   | How should support articles be written for better retrieval? | Support articles nên rõ ràng, dùng tiêu đề cụ thể, tránh câu mơ hồ, có owner/date/scope/limitations.      |
| 4   | Why can overlap improve chunking results?                    | Overlap giữ context ở ranh giới chunk, giảm mất thông tin khi câu/ý bị chia giữa hai chunks.              |
| 5   | Khi nào metadata ngôn ngữ giúp retrieval tiếng Việt?         | Khi query/tài liệu có nhiều ngôn ngữ, `language=vi` giúp chỉ tìm trong tài liệu tiếng Việt và giảm nhiễu. |

### Kết Quả Của Tôi

| #   | Query                                                        | Top-1 Retrieved Chunk (tóm tắt)                                      | Score | Relevant?                                    | Agent Answer (tóm tắt)                                                         |
| --- | ------------------------------------------------------------ | -------------------------------------------------------------------- | ----- | -------------------------------------------- | ------------------------------------------------------------------------------ |
| 1   | What does a vector store keep and retrieve?                  | `vector_store_notes.md`, chunk về metadata/filter trong vector store | 0.024 | Có, nhưng chưa phải chunk tốt nhất           | Vector store lưu vector + metadata và dùng similarity/filter để retrieve.      |
| 2   | What are the main steps in a RAG pipeline?                   | `rag_system_design.md`, background của internal knowledge assistant  | 0.174 | Có                                           | RAG gồm ingestion/chunking, embedding/store, retrieval và prompt/LLM answer.   |
| 3   | How should support articles be written for better retrieval? | `customer_support_playbook.txt`, chunk về review failed queries      | 0.126 | Có một phần                                  | Nên viết bài support rõ ràng, tránh mơ hồ, review failed queries để cải thiện. |
| 4   | Why can overlap improve chunking results?                    | `chunking_experiment_report.md`, chunk về recursive chunking         | 0.158 | Có                                           | Overlap/context boundary giúp chunk giữ ý đầy đủ hơn.                          |
| 5   | Khi nào metadata ngôn ngữ giúp retrieval tiếng Việt?         | `vi_retrieval_notes.md`, chunk tiếng Việt về lỗi retrieval           | 0.120 | Có một phần; top-3 có chunk metadata tốt hơn | Dùng `language=vi` khi cần giới hạn retrieval vào tài liệu tiếng Việt.         |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 5 / 5

**Ghi chú về failure case:**  
Khi bỏ metadata filter và chỉ dùng `_mock_embed`, query về vector store có thể retrieve nhầm tài liệu tiếng Việt hoặc support. Điều này cho thấy metadata filter rất quan trọng trong lab này, đặc biệt khi embedding backend không phải semantic model thật.

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**  
Fixed-size chunking dễ kiểm soát số chunk và dễ debug, nhưng không nên dùng một mình cho markdown/notes có cấu trúc rõ. Sentence-based chunking giúp chunk dễ đọc hơn, nhưng nếu câu dài hoặc nhiều ý trong một câu thì chunk vẫn có thể quá rộng.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**  
Một điểm cần quan sát là nhóm nào thiết kế metadata tốt thường có retrieval ổn định hơn nhóm chỉ đổi model/chunk size. Retrieval quality phụ thuộc nhiều vào data strategy, không chỉ embedding.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**  
Tôi sẽ thêm metadata chi tiết hơn như `audience`, `section_title`, `last_updated`, và `public_internal`. Tôi cũng sẽ thử custom chunker theo markdown headings để mỗi chunk luôn có tiêu đề section, giúp agent có context rõ hơn khi trả lời.

---

## Tự Đánh Giá

| Tiêu chí                    | Loại    | Điểm tự đánh giá |
| --------------------------- | ------- | ---------------- |
| Warm-up                     | Cá nhân | 5 / 5            |
| Document selection          | Nhóm    | 9 / 10           |
| Chunking strategy           | Nhóm    | 13 / 15          |
| My approach                 | Cá nhân | 10 / 10          |
| Similarity predictions      | Cá nhân | 4 / 5            |
| Results                     | Cá nhân | 8 / 10           |
| Core implementation (tests) | Cá nhân | 30 / 30          |
| Demo                        | Nhóm    | 4 / 5            |
| **Tổng**                    |         | **83 / 100**     |
