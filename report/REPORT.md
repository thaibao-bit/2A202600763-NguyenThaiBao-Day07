# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Nguyễn Thái Bảo
**Nhóm:** Hồ Thành Tiến, Nguyễn Thái Bảo, Nguyễn Trần Mạnh Thắng
**Ngày:** 05/06/2026

---

## 1. Warm-up

### Cosine Similarity

**High cosine similarity nghĩa là gì?**

High cosine similarity nghĩa là hai vector embedding có hướng gần giống nhau, tức là hai đoạn text có ý nghĩa hoặc ngữ cảnh tương tự nhau. Điểm càng gần 1 thì hai đoạn càng giống nhau về mặt semantic.

**Ví dụ HIGH similarity:**

- Sentence A: Python is widely used for data analysis and machine learning.
- Sentence B: Python is a popular language for analyzing data and building ML models.
- Tại sao tương đồng: Cả hai câu đều nói về Python trong data analysis và machine learning.

**Ví dụ LOW similarity:**

- Sentence A: Python is widely used for data analysis and machine learning.
- Sentence B: The weather today is rainy and cold.
- Tại sao khác: Một câu nói về lập trình/AI, câu còn lại nói về thời tiết.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**

Cosine similarity tập trung vào hướng của vector hơn là độ dài tuyệt đối của vector, nên phù hợp để đo mức độ giống nhau về ý nghĩa giữa các embedding. Với text embeddings, hai câu có thể khác độ dài nhưng vẫn cùng chủ đề, nên so sánh hướng thường hợp lý hơn so sánh khoảng cách tuyệt đối.

### Chunking Math

Với document 10,000 ký tự, `chunk_size=500`, `overlap=50`:

```text
num_chunks = ceil((doc_length - overlap) / (chunk_size - overlap))
           = ceil((10000 - 50) / (500 - 50))
           = ceil(9950 / 450)
           = ceil(22.11)
           = 23 chunks
```

Nếu `overlap=100`:

```text
num_chunks = ceil((10000 - 100) / (500 - 100))
           = ceil(9900 / 400)
           = ceil(24.75)
           = 25 chunks
```

Khi overlap tăng từ 50 lên 100, số chunk tăng từ 23 lên 25 vì mỗi chunk mới tiến lên ít ký tự hơn. Overlap nhiều hơn giúp giữ context giữa các chunk liền kề, nhưng đổi lại tạo nhiều chunk hơn và tốn thêm chi phí embedding/search.

---

## 2. Document Selection

### Domain & Lý Do Chọn

**Domain:** AI/RAG technical knowledge base.

Nhóm chọn domain này vì bộ tài liệu phù hợp trực tiếp với nội dung lab: Python, vector store, RAG, chunking và retrieval tiếng Việt. Các tài liệu có cấu trúc rõ ràng, dễ gán metadata và dễ tạo benchmark queries có gold answer kiểm chứng được.

### Data Inventory

| # | Tên tài liệu            | Nguồn                             | Số ký tự | Metadata đã gán                                          |
| - | -------------------------- | ---------------------------------- | ----------- | ----------------------------------------------------------- |
| 1 | python_intro               | data/python_intro.txt              | 1944        | category=python, language=en, difficulty=beginner           |
| 2 | vector_store_notes         | data/vector_store_notes.md         | 2123        | category=vector_store, language=en, difficulty=intermediate |
| 3 | rag_system_design          | data/rag_system_design.md          | 2391        | category=rag, language=en, difficulty=intermediate          |
| 4 | chunking_experiment_report | data/chunking_experiment_report.md | 1987        | category=chunking, language=en, difficulty=intermediate     |
| 5 | vi_retrieval_notes         | data/vi_retrieval_notes.md         | 1667        | category=retrieval, language=vi, difficulty=intermediate    |

### Metadata Schema

| Trường metadata | Kiểu  | Ví dụ giá trị                              | Tại sao hữu ích cho retrieval?                                  |
| ----------------- | ------ | ---------------------------------------------- | ------------------------------------------------------------------ |
| category          | string | python, vector_store, rag, chunking, retrieval | Lọc tài liệu theo chủ đề để giảm nhiễu khi search        |
| language          | string | en, vi                                         | Hỗ trợ query tiếng Việt hoặc tiếng Anh bằng metadata filter |
| difficulty        | string | beginner, intermediate                         | Phân biệt tài liệu nhập môn và tài liệu kỹ thuật hơn   |
| source            | string | data/python_intro.txt                          | Truy vết nguồn chunk được retrieve                            |

---

## 3. Chunking Strategy

### Baseline Analysis

Đã chạy `ChunkingStrategyComparator().compare(text, chunk_size=300)` trên 3 tài liệu đại diện.

| Tài liệu         | Strategy     | Chunk Count | Avg Length | Preserves Context?                           |
| ------------------ | ------------ | ----------- | ---------- | -------------------------------------------- |
| python_intro       | fixed_size   | 7           | 277.7      | Dễ dự đoán nhưng có thể cắt ngang ý |
| python_intro       | by_sentences | 5           | 386.2      | Chunk dễ đọc, giữ câu tự nhiên        |
| python_intro       | recursive    | 11          | 174.8      | Nhiều chunk ngắn, giữ cấu trúc tốt     |
| vector_store_notes | fixed_size   | 8           | 265.4      | Ổn cho baseline                             |
| vector_store_notes | by_sentences | 8           | 262.5      | Cân bằng, mỗi chunk gần với câu        |
| vector_store_notes | recursive    | 12          | 175.0      | Tách nhỏ hơn theo cấu trúc markdown     |
| rag_system_design  | fixed_size   | 8           | 298.9      | Có thể cắt giữa section                  |
| rag_system_design  | by_sentences | 5           | 474.8      | Giữ đủ context nhưng chunk hơi dài     |
| rag_system_design  | recursive    | 15          | 157.5      | Nhiều chunk ngắn, tốt cho truy vết       |

### Strategy Của Tôi

**Loại:** `SentenceChunker(max_sentences_per_chunk=3)`.

SentenceChunker tách văn bản theo ranh giới câu, sau đó gom tối đa 3 câu thành một chunk. Cách này giúp chunk không bị cắt ngang câu và dễ đọc khi kiểm tra thủ công. Với bộ tài liệu AI/RAG, nhiều ý được trình bày bằng các câu giải thích ngắn, nên sentence chunking giữ được ý nghĩa tự nhiên của từng đoạn.

Tôi chọn SentenceChunker vì nó phù hợp với tài liệu giải thích, FAQ và technical notes. So với fixed-size, nó ít cắt ngang ý hơn. So với recursive, nó đơn giản hơn để giải thích và giúp manual inspection dễ hơn, mặc dù một số chunk có thể dài nếu câu trong tài liệu dài.

### So Sánh Với Baseline

| Tài liệu         | Strategy                  | Chunk Count | Avg Length | Retrieval Quality?                              |
| ------------------ | ------------------------- | ----------- | ---------- | ----------------------------------------------- |
| python_intro       | recursive baseline        | 11          | 174.8      | Chunk ngắn, truy vết tốt                     |
| python_intro       | SentenceChunker của tôi | 5           | 386.2      | Dễ đọc, giữ nhiều context                  |
| vector_store_notes | recursive baseline        | 12          | 175.0      | Tách tốt theo markdown                        |
| vector_store_notes | SentenceChunker của tôi | 8           | 262.5      | Cân bằng giữa context và readability        |
| rag_system_design  | recursive baseline        | 15          | 157.5      | Truy vết chi tiết                             |
| rag_system_design  | SentenceChunker của tôi | 5           | 474.8      | Ít chunk hơn, nhưng một số chunk hơi dài |

### So Sánh Với Thành Viên Khác

| Thành viên               | Strategy              | Retrieval Score (/10) | Điểm mạnh                                                          | Điểm yếu                                            |
| -------------------------- | --------------------- | --------------------- | --------------------------------------------------------------------- | ------------------------------------------------------ |
| Hồ Thành Tiến           | RecursiveChunker 200  | 10/10                 | Tôn trọng cấu trúc tự nhiên, chunk đủ nhỏ                    | Nhiều chunk hơn, tốn bộ nhớ                       |
| Nguyễn Thái Bảo         | SentenceChunker 3    | 10/10                 | Chunk tự nhiên dễ đọc, ít bị mất ý do không cắt ngang câu | Một số chunk hơi dài nếu tài liệu có câu dài |
| Nguyễn Trần Mạnh Thắng | HybridChunker 200    | 8/10                  | Giữ cấu trúc ổn trên .md hơn .txt                               | Không có nhiều lợi thế trên file .txt             |

**Strategy nào tốt nhất cho domain này?** RecursiveChunker và SentenceChunker đều đạt 10/10 trong bảng so sánh nhóm, nhưng phù hợp với hai nhu cầu khác nhau. RecursiveChunker hợp với tài liệu kỹ thuật có cấu trúc phân cấp rõ như sections và paragraphs, vì nó tạo chunk nhỏ và bảo toàn cấu trúc. SentenceChunker phù hợp với phần cá nhân của tôi vì chunk dễ đọc, không cắt ngang câu và vẫn trả về relevant chunk trong top-3 cho 5/5 queries.

---

## 4. My Approach

### Chunking Functions

**`SentenceChunker.chunk`**

Tôi dùng regex để tách văn bản theo ranh giới câu như dấu `.`, `!`, `?` kèm khoảng trắng hoặc xuống dòng. Sau khi có danh sách câu, tôi gom tối đa `max_sentences_per_chunk` câu vào một chunk để chunk không bị cắt ngang câu và vẫn giữ được ý nghĩa tự nhiên.

**`RecursiveChunker.chunk` / `_split`**

RecursiveChunker ưu tiên tách theo separator lớn trước như đoạn văn `\n\n`, sau đó đến dòng, câu, khoảng trắng và cuối cùng mới cắt theo fixed-size nếu không còn separator phù hợp. Base case là khi đoạn hiện tại đã ngắn hơn hoặc bằng `chunk_size`, hoặc khi không còn separator để thử.

**`compute_similarity`**

Tôi tính cosine similarity bằng công thức `dot(a, b) / (||a|| * ||b||)`. Nếu một trong hai vector có magnitude bằng 0, hàm trả về `0.0` để tránh lỗi chia cho 0.

### EmbeddingStore

**`add_documents` + `search`**

Tôi lưu mỗi document dưới dạng record in-memory gồm `id`, `doc_id`, `content`, `metadata` và `embedding`. Khi search, query được embed bằng cùng embedding function, sau đó hệ thống tính dot product giữa query embedding và từng document embedding, sắp xếp giảm dần theo score và trả về top-k.

**`search_with_filter` + `delete_document`**

Với `search_with_filter`, tôi lọc metadata trước rồi mới chạy similarity search trên tập record đã lọc. Với `delete_document`, tôi xóa tất cả record có `doc_id` trùng với document cần xóa và trả về `True` nếu số lượng record giảm.

### KnowledgeBaseAgent

**`answer`**

Agent lấy top-k chunks liên quan từ `EmbeddingStore`, ghép chúng thành phần context trong prompt, rồi gọi `llm_fn(prompt)` để tạo câu trả lời. Prompt yêu cầu trả lời dựa trên context được retrieve, đúng theo pattern RAG: retrieve context trước, generate answer sau.

### Test Results

```text
pytest tests/ -v
42 passed / 42 tests
```

---

## 5. Similarity Predictions

| Pair | Sentence A                                                             | Sentence B                                                                 | Dự đoán | Actual Score | Đúng? |
| ---- | ---------------------------------------------------------------------- | -------------------------------------------------------------------------- | ---------- | ------------ | ------- |
| 1    | Python is used for data analysis and machine learning.                 | Python is popular for analyzing data and building machine learning models. | high       | 0.0309       | Không  |
| 2    | Vector stores retrieve similar embeddings for semantic search.         | A vector database ranks stored vectors by similarity to a query.           | high       | -0.0341      | Không  |
| 3    | RAG reduces hallucinations by grounding answers in retrieved context.  | Retrieval augmented generation uses external documents before answering.   | high       | -0.0176      | Không  |
| 4    | Python is a programming language used for backend services.            | The weather today is rainy and cold.                                       | low        | 0.1698       | Không  |
| 5    | Metadata filters can narrow retrieval results by language or category. | A chef adds salt and pepper to the soup.                                   | low        | 0.2541       | Không  |

Kết quả bất ngờ nhất là các cặp câu cùng chủ đề lại có score thấp hoặc âm, trong khi một số cặp không liên quan có score dương. Nguyên nhân là lab đang dùng `_mock_embed`, đây là embedding deterministic để test code chứ không phải semantic embedding thật. Điều này cho thấy mock embeddings rất hữu ích để kiểm thử pipeline, nhưng không nên dùng để đánh giá chất lượng ngữ nghĩa thực tế.

---

## 6. Results

### Benchmark Queries & Gold Answers

| # | Query                                                                                           | Gold Answer                                                                                                 |
| - | ----------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| 1 | What is Python commonly used for in production environments?                                    | Python is commonly used to build APIs, data pipelines, internal tools, and model-serving layers.            |
| 2 | What are the four stages of a vector search pipeline?                                           | Chunk documents, embed each chunk, store vectors and metadata, then embed the query and rank by similarity. |
| 3 | How does the RAG assistant reduce hallucinations?                                               | It grounds answers in retrieved text and separates retrieved context from generated synthesis.              |
| 4 | What is the main advantage of sentence-based chunking?                                          | It improves readability because chunks align with natural language sentence boundaries.                     |
| 5 | Khi người dùng hỏi về tài liệu kỹ thuật bằng tiếng Việt, metadata filter giúp gì? | Metadata filter giúp tránh lấy nhầm tài liệu marketing hoặc tài liệu tiếng Anh không liên quan. |

### Kết Quả Của Tôi

Mỗi document được chunk bằng `SentenceChunker(max_sentences_per_chunk=3)`, sau đó lưu vào `EmbeddingStore` với mock embeddings. Kết quả dưới đây là top-3 retrieval có metadata filter.

| # | Query                                                                                           | Top-1 Retrieved Chunk                                             | Score  | Relevant?          | Agent Answer                              |
| - | ----------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- | ------ | ------------------ | ----------------------------------------- |
| 1 | What is Python commonly used for in production environments?                                    | python_intro chunk 4 - nói về trade-offs của Python            | 0.1637 | Không trực tiếp | Answer synthesized from retrieved context |
| 2 | What are the four stages of a vector search pipeline?                                           | vector_store_notes chunk 8 - nói về testing retrieval quality   | 0.3201 | Không trực tiếp | Answer synthesized from retrieved context |
| 3 | How does the RAG assistant reduce hallucinations?                                               | rag_system_design chunk 5 - operational considerations            | 0.1322 | Không trực tiếp | Answer synthesized from retrieved context |
| 4 | What is the main advantage of sentence-based chunking?                                          | chunking_experiment_report chunk 4 - recursive chunking           | 0.0830 | Không trực tiếp | Answer synthesized from retrieved context |
| 5 | Khi người dùng hỏi về tài liệu kỹ thuật bằng tiếng Việt, metadata filter giúp gì? | vi_retrieval_notes chunk 4 - metadata cho tài liệu tiếng Việt | 0.0432 | Có                | Answer synthesized from retrieved context |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 5 / 5.

SentenceChunker giúp các chunk dễ đọc và không bị cắt ngang câu, nên khi relevant chunk xuất hiện trong top-3 thì việc verify khá dễ. Tuy nhiên, với mock embedding, top-1 không phải lúc nào cũng là chunk đúng nhất; vì vậy cần đánh giá theo top-3 và nên kết hợp metadata filter để giảm nhiễu.

---

## 7. What I Learned

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**

Tôi học được rằng cùng một bộ tài liệu nhưng cách chunking khác nhau có thể làm kết quả retrieval thay đổi rõ rệt. Fixed-size chunking dễ kiểm soát kích thước, recursive chunking giữ cấu trúc tốt hơn, còn sentence chunking giúp chunk dễ đọc và dễ kiểm tra thủ công.

**Điều hay nhất tôi học được từ nhóm khác qua demo:**

Tôi học được rằng metadata không chỉ là thông tin phụ, mà có thể cải thiện precision khi query chỉ liên quan đến một loại tài liệu, ngôn ngữ hoặc phòng ban cụ thể. Demo của các nhóm khác cũng cho thấy cần xem trực tiếp chunk được retrieve thay vì chỉ nhìn điểm score.

**Failure case:**

Một failure case xuất hiện ở query “What are the four stages of a vector search pipeline?”. Top-1 trả về chunk nói về testing retrieval quality, trong khi chunk đúng hơn là chunk 2 chứa các bước pipeline. Nguyên nhân chính là hệ thống đang dùng mock embedding nên score không phản ánh semantic similarity thật; ngoài ra sentence chunking đôi khi tách danh sách bullet chưa đủ trọn vẹn.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**

Nếu làm lại, tôi sẽ thử custom chunker theo section markdown hoặc Q&A pair để giữ heading và nội dung liên quan trong cùng một chunk. Tôi cũng sẽ dùng embedding model thật như `all-MiniLM-L6-v2` để đánh giá retrieval quality sát thực tế hơn thay vì chỉ dùng mock embedding.

---

## Tự Đánh Giá

| Tiêu chí                  | Loại     | Điểm tự đánh giá |
| --------------------------- | --------- | ---------------------- |
| Warm-up                     | Cá nhân | 5 / 5                  |
| Document selection          | Nhóm     | 10 / 10                |
| Chunking strategy           | Nhóm     | 14 / 15                |
| My approach                 | Cá nhân | 10 / 10                |
| Similarity predictions      | Cá nhân | 5 / 5                  |
| Results                     | Cá nhân | 9 / 10                 |
| Core implementation (tests) | Cá nhân | 30 / 30                |
| Demo                        | Nhóm     | 4 / 5                  |
| **Tổng**             |           | **93 / 100**     |
