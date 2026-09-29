# Raglyra — System Architecture

> **Status:** DRAFT
> **Mục tiêu:** Mô tả toàn bộ kiến trúc Raglyra ở mức hệ thống: các khối chính, hai luồng lớn, ranh giới trách nhiệm và cách các khối kết nối với nhau.
> **Phạm vi:** Kiến trúc logic. Chưa chốt DB schema, API cụ thể, class Python, provider/model hay deployment topology cuối cùng.

---

## 1. Raglyra giải quyết bài toán gì?

Raglyra là nền tảng **RAG — Retrieval-Augmented Generation**.

**Retrieval-Augmented Generation** = cách xây hệ thống AI mà LLM không tự trả lời hoàn toàn bằng kiến thức đã học, mà trước tiên phải tìm dữ liệu liên quan từ nguồn của hệ thống, sau đó dùng dữ liệu đó để tạo câu trả lời.

Luồng tư duy:

~~~text
User hỏi
    ↓
Hiểu user đang hỏi gì
    ↓
Xác định nguồn nào có dữ liệu phù hợp
    ↓
Tìm dữ liệu
    ↓
Chọn bằng chứng đủ tốt
    ↓
LLM hoặc code tạo câu trả lời
    ↓
Gắn citation
~~~

**Citation** = trích dẫn để biết câu trả lời dựa trên tài liệu/source nào.

**Grounded Answer** = câu trả lời có căn cứ từ bằng chứng đã được hệ thống chọn, thay vì LLM tự tưởng tượng.

## 2. Raglyra có hai pipeline lớn

### 2.1 Knowledge Pipeline

Đây là luồng chuẩn bị knowledge trước khi user hỏi.

~~~text
Source
→ Parse / OCR
→ Chuẩn hóa cấu trúc
→ Chia knowledge
→ Tạo index
→ Sẵn sàng retrieval
~~~

### 2.2 Query Pipeline

Đây là luồng xử lý khi user chat.

~~~text
User Message
→ Hiểu context hội thoại
→ Hiểu ý định
→ Lập kế hoạch
→ Tìm data/evidence
→ Kiểm tra bằng chứng
→ Tạo answer
→ Citation
~~~

Hai pipeline gặp nhau ở Retrieval: Knowledge Pipeline tạo dữ liệu searchable, Query Pipeline dùng dữ liệu đó để trả lời.

## 3. Sơ đồ kiến trúc tổng thể

~~~mermaid
flowchart LR
    U[User / Client] --> API[RAG API]
    API --> SEC[Security Context]
    SEC --> CONV[Conversation Engine]
    CONV --> QU[Query Understanding]
    QU --> PLAN[Query Planner]
    PLAN --> EXEC[Plan Executor]
    EXEC --> STRUCT[Structured Sources]
    EXEC --> RET[Retrieval Engine]
    RET --> EV[Evidence Layer]
    STRUCT --> EV
    EV --> ANS[Answer Engine]
    ANS --> CIT[Citation Builder]
    CIT --> OUT[Response]

    SRC[Knowledge Sources] --> ING[Ingestion Engine]
    ING --> META[(Metadata DB)]
    ING --> OBJ[Object Storage]
    ING --> VEC[Vector Store]
    ING --> LEX[Lexical Index]
    RET --> VEC
    RET --> LEX
    RET --> META
~~~

## 4. RAG API

**RAG API** = cổng backend nhận request từ UI/widget/service khác.

Nhiệm vụ chính:
- nhận request;
- resolve authentication/security context;
- validate request cơ bản;
- gọi application flow;
- trả response.

Không nên để API route tự search vector, tự build prompt hoặc tự gọi nhiều provider.

## 5. Security Context

**Security Context** = thông tin xác định request đang thuộc user/tenant nào và được quyền truy cập phạm vi nào.

Ví dụ logic:

~~~text
tenant_id
workspace_id
actor_id
assistant_id
permissions
allowed_dataset_ids
request_id
~~~

Nguyên tắc: Security phải được xác định trước retrieval.

Không được search toàn database rồi mới filter tenant.

## 6. Conversation Engine

**Conversation Engine** = khối hiểu ngữ cảnh nhiều lượt chat.

Ví dụ:

~~~text
User: Tôi muốn xem VPS Basic.
Bot: ...
User: Còn giá?
~~~

Câu cuối không đủ nghĩa nếu xử lý độc lập.

Conversation Engine cần biết:

~~~text
active_entity = VPS Basic
~~~

Nhiệm vụ:
- giữ recent conversation context;
- giữ active entities;
- quản lý pending clarification;
- giữ user constraints;
- dùng summary khi conversation dài;
- resolve các từ tham chiếu như 'nó', 'gói kia', 'còn giá?'.

Không phải nhiệm vụ của Conversation Engine:
- vector search;
- pricing lookup;
- tạo answer business.

## 7. Query Understanding

**Query Understanding** = bước phân tích câu hỏi để hiểu user muốn hệ thống làm gì.

Ví dụ:

~~~text
"VPS Basic một tháng bao nhiêu?"
~~~

Output logic:

~~~text
task = STRUCTURED_LOOKUP
topic = PRICING
entity = VPS Basic
freshness = CURRENT
~~~

**Task** = loại công việc cần làm.

**Topic** = chủ đề nghiệp vụ.

**Freshness** = mức độ mới của dữ liệu mà query yêu cầu.

## 8. Query Planner

**Query Planner** = bộ biến kết quả Query Understanding thành kế hoạch có thể thực thi.

Output là **QueryPlan**.

**QueryPlan** = object mô tả hệ thống phải làm gì, lấy dữ liệu ở đâu, trong scope nào và theo strategy nào.

Ví dụ:

~~~text
plan_type = STRUCTURED_SOURCE
source_capability = CURRENT_PRICING
entity = VPS Basic
~~~

hoặc:

~~~text
plan_type = RAG
dataset = policy
retrieval_profile = HYBRID
~~~

Planner quyết định **WHAT/WHERE** — làm gì và dùng nguồn nào.

Retriever quyết định **HOW** — tìm như thế nào.

## 9. Plan Executor

**Plan Executor** = khối thực thi QueryPlan.

~~~text
DIRECT
→ không retrieval

STRUCTURED_SOURCE
→ gọi API/DB structured

EXACT_SEARCH
→ exact/lexical lookup

RAG
→ Retrieval Engine

MULTI_PLAN
→ chạy nhiều subplan
~~~

Executor không tự classify query lại.

## 10. Structured Sources

**Structured Source** = nguồn dữ liệu có cấu trúc rõ và thường là nguồn chính xác cho dữ liệu exact/current.

Ví dụ:
- Pricing API;
- Inventory API;
- SQL table;
- business service.

Ví dụ user hỏi giá realtime thì Pricing API thường phù hợp hơn vector document, vì vector snapshot có thể cũ.

## 11. Retrieval Engine

**Retrieval** = quá trình tìm dữ liệu liên quan từ knowledge đã được index.

Có thể gồm:

### Exact Search
Tìm chính xác mã/identifier.

### Lexical Search
**Lexical Search** = tìm theo chữ/từ khóa/toàn văn, ví dụ PostgreSQL FTS hoặc BM25.

### Dense Search
**Dense Search** = semantic search bằng embedding vector.

**Embedding** = biểu diễn text thành vector số để đo độ gần về nghĩa.

### Hybrid Search
**Hybrid Search** = kết hợp lexical + dense để tận dụng cả exact terms và semantic meaning.

## 12. Candidate

**Candidate** = kết quả tạm thời lấy từ retrieval lane.

Candidate chưa phải evidence cuối cùng.

## 13. Fusion

**Fusion** = gộp nhiều bảng xếp hạng thành một ranking chung.

Baseline có thể dùng **RRF — Reciprocal Rank Fusion**.

RRF gộp dựa trên thứ hạng thay vì cộng trực tiếp các score khác thang đo.

## 14. Reranker

**Reranker** = mô hình xếp hạng lại một tập candidate nhỏ để tăng precision.

~~~text
50 candidate
→ reranker
→ top 8
~~~

**Precision** = tỷ lệ kết quả được chọn thật sự liên quan.

**Recall** = khả năng tìm đủ các bằng chứng có liên quan.

## 15. Evidence Layer

**Evidence** = candidate đã được chọn làm căn cứ trả lời.

Evidence phải giữ:

~~~text
content
source
source version
page/section/row nếu có
freshness
authority
quality
retrieval trace
~~~

**Provenance** = thông tin nguồn gốc của evidence.

## 16. Evidence Gate

**Evidence Gate** = bước quyết định bằng chứng có đủ để trả lời hay không.

Possible outputs:

~~~text
ANSWERABLE
PARTIAL
NO_ANSWER
NEEDS_CLARIFICATION
~~~

Ví dụ user hỏi so sánh A và B nhưng chỉ có evidence cho A thì phải trả PARTIAL, không cho LLM tưởng tượng B.

## 17. Answer Engine

Answer Engine chọn strategy phù hợp:

### Deterministic
Code format trực tiếp từ typed data.

### Extractive
Lấy gần trực tiếp câu trả lời từ evidence.

### Grounded Synthesis
LLM tổng hợp nhiều evidence nhưng phải bị giới hạn bởi evidence đã chọn.

## 18. Citation Builder

Citation Builder nhận provenance của Evidence và tạo citation an toàn.

LLM không được tự bịa source URL.

## 19. Ingestion Engine

**Ingestion** = quá trình đưa dữ liệu vào hệ thống knowledge.

~~~text
Source
→ SourceVersion
→ Parser/OCR
→ Canonical Artifact
→ Region Segmentation
→ KnowledgeUnit / StructuredRecord
→ Index
~~~

Chi tiết nằm ở `docs/01-flows/02-ingestion-indexing-flow.md`.

## 20. Storage Layer

### Metadata DB
Lưu tenant, dataset, source, source version, ingestion job, index generation, conversation metadata, profiles và trace/evaluation metadata.

### Object Storage
**Object Storage** = nơi lưu file/blob như PDF gốc, parsed artifacts và snapshots.

### Vector Store
Lưu/search embedding.

~~~text
VectorStore
├── PgVectorStore
└── QdrantVectorStore
~~~

### Lexical Index
Dùng cho keyword/full-text retrieval.

## 21. Evaluation & Observability

**Observability** = khả năng nhìn hệ thống đang chạy như thế nào.

**Evaluation** = đo hệ thống có trả đúng hay không.

Ví dụ metrics:

~~~text
Reference Resolution Accuracy
Recall@K
MRR
Groundedness
Citation Accuracy
No-answer Accuracy
~~~

## 22. Infrastructure Abstraction

Các backend/provider phải nằm sau interface:

~~~text
VectorStore
EmbeddingProvider
LLMProvider
RerankerProvider
OCRProvider
ObjectStorage
LexicalStore
QueueBackend
~~~

**Abstraction** = hợp đồng chung mà nhiều implementation có thể tuân theo.

Mục tiêu: đổi vendor dễ, test dễ và core không coupling provider.

## 23. Ví dụ end-to-end — policy query

~~~text
User: Nếu tôi không dùng VPS nữa thì có được hoàn tiền không?
↓
Security Scope
↓
Conversation Context
↓
Query Understanding
task = KNOWLEDGE_QA
topic = REFUND_POLICY
↓
Query Planner
plan = RAG
dataset = policy
↓
Retrieval
lexical + dense
↓
Fusion
↓
Reranker
↓
Evidence
↓
Evidence Gate
↓
Grounded Synthesis
↓
Citation
↓
Response
~~~

## 24. Ví dụ end-to-end — current price

~~~text
User: VPS Basic một tháng bao nhiêu?
↓
Query Understanding
STRUCTURED_LOOKUP + PRICING + CURRENT
↓
Planner
STRUCTURED_SOURCE
↓
Pricing API/DB
↓
typed result
↓
deterministic formatter
↓
response
~~~

Không cần vector search.

## 25. Error Philosophy

Nếu scope không chắc thì **fail closed**.

**Fail Closed** = khi không chắc quyền/phạm vi thì từ chối an toàn thay vì mở rộng quyền.

Nếu evidence thiếu:

~~~text
NO_ANSWER / PARTIAL / CLARIFY
~~~

không ép LLM trả lời.

## 26. Những gì file này chưa chốt

- DB schema;
- exact API;
- exact Python package layout;
- default vector backend;
- embedding model;
- reranker model;
- parser/OCR provider;
- deployment topology;
- thresholds;
- cache;
- retry/timeouts.

## 27. Design Decisions

1. Raglyra có Knowledge Pipeline và Query Pipeline tách rõ.
2. Conversation được xử lý trước Query Understanding.
3. Query Planner tạo QueryPlan.
4. Planner quyết định WHAT/WHERE; Retrieval quyết định HOW.
5. Current structured data không mặc định lấy từ vector snapshot.
6. Candidate khác Evidence.
7. Evidence Gate đứng trước generation.
8. Citation build từ provenance.
9. Core độc lập pgvector/Qdrant/provider cụ thể.
10. Security scope đi xuyên pipeline.
11. Evaluation/Observability là core capability.

## 28. Review Checklist

- [ ] Hiểu hai pipeline lớn.
- [ ] Đồng ý Conversation đứng trước Query Understanding.
- [ ] Đồng ý QueryPlan là contract execution.
- [ ] Đồng ý current structured source không nhất thiết đi RAG.
- [ ] Đồng ý Retrieval trả Evidence, không answer.
- [ ] Đồng ý Evidence Gate.
- [ ] Đồng ý provider/backend abstraction.
- [ ] Đồng ý Evaluation là first-class.

## 29. Next Step

Đọc tiếp:

~~~text
01-query-answer-flow.md
↓
02-ingestion-indexing-flow.md
~~~

Sau khi hai flow được REVIEWED mới đi vào Component Design.