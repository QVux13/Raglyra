# Raglyra — System Architecture v2

> **Version:** v2 — self-contained, thay thế v1 làm bản LATEST

> **Status:** REVIEWED
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
    API --> SEC[Security Context Resolver]
    SEC --> ALLOW[Allowed Scope Resolver]
    ALLOW --> CONV[Conversation Engine]
    CONV --> QU[Query Understanding]
    QU --> ER[Entity Resolver]
    ER --> REQ[Capability Requirement Builder]
    REQ --> POL[Policy Scope Validator]
    POL --> CAP[Capability Registry / Resolver]
    CAP --> SRCRES[Source Resolver]
    SRCRES --> PLAN[Query Planner]
    PLAN --> PVAL[QueryPlan Validator]
    PVAL --> EXEC[Plan Executor]

    EXEC --> STRUCT[Structured Sources]
    EXEC --> RET[Retrieval Engine]

    RET --> EV[Evidence Layer]
    STRUCT --> EV
    EV --> GATE[Evidence Gate]
    GATE --> ANS[Answer Engine]
    ANS --> CIT[Citation Builder]
    CIT --> OUT[Response]

    SRC[Knowledge Sources] --> ING[Ingestion Engine]
    ING --> META[(Metadata DB)]
    ING --> OBJ[Object Storage]
    ING --> VEC[Vector Store]
    ING --> LEX[Lexical Index]
    ING --> CAPREG[Capability Registry]

    RET --> VEC
    RET --> LEX
    RET --> META
    CAPREG --> CAP
~~~

Điểm quan trọng của v2 là tách **phạm vi được phép trước khi hiểu query** và **kiểm tra policy sau khi đã hiểu query**:

~~~text
Security Context
→ Allowed Scope
→ Conversation
→ Query Understanding
→ Entity Resolution
→ Capability Requirement
→ Policy Scope Validation
→ Capability / Source Resolution
→ QueryPlan
→ Plan Validation
→ Execute
~~~

Lý do: trước khi Query Understanding chạy, hệ thống chỉ có thể biết **user/assistant được phép nhìn thấy gì**. Sau khi Query Understanding chạy, hệ thống mới biết query đang hỏi **Task/Topic/Entity/Freshness nào** để đối chiếu với allowed scope.

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

## 6. Hai lớp kiểm soát Scope

Ở v1, khái niệm `Intent & Scope Guard` đứng trước Query Understanding, điều này gây vòng phụ thuộc: chưa hiểu intent/topic thì chưa thể kiểm policy theo intent/topic.

v2 tách thành hai lớp.

### 6.1 Allowed Scope Resolver — xác định "được phép nhìn thấy gì"

**Allowed Scope Resolver** = khối tạo phạm vi tài nguyên tối đa mà request có thể sử dụng dựa trên dữ liệu đáng tin cậy.

Input:

~~~text
verified identity/session
assistant configuration
tenant/workspace binding
permissions/ACL
resource active state
~~~

Output:

~~~text
AllowedScope
├── tenant
├── workspace
├── assistant
├── datasets
├── sources
├── capabilities
└── permissions
~~~

Khối này **không cần hiểu câu hỏi**.

### 6.2 Policy Scope Validator — xác định "query này có được dùng những thứ đó không"

Sau Query Understanding + Entity Resolution, hệ thống đã biết:

~~~text
task
topic
entities
freshness
constraints
required capability
~~~

**Policy Scope Validator** kiểm:

~~~text
required capability ∈ allowed capabilities?
topic ∈ assistant domain?
entity/source ∈ allowed resource scope?
freshness có source hợp lệ?
action có được phép?
~~~

Possible result:

~~~text
ALLOWED
CLARIFY
OUT_OF_SCOPE
UNSUPPORTED
FORBIDDEN
~~~

Nguyên tắc:

~~~text
AllowedScope giới hạn trần quyền
PolicyScopeValidation không bao giờ được mở rộng trần đó
~~~

Chi tiết nằm trong:
`docs/01-luong-hoi-dap/intent-va-pham-vi-cau-hoi-v2.md`.

## 7. Conversation Engine

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

## 8. Query Understanding

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

## 9. Capability Registry, Source Resolver và Query Planner

### Capability Registry

**Capability Registry** = danh mục runtime cho biết hệ thống hiện có những khả năng nào và khả năng đó được cung cấp bởi source nào.

Ví dụ:

~~~text
CURRENT_PRICING
├── source = pricing-api
├── status = AVAILABLE
├── freshness = CURRENT
├── authority = AUTHORITATIVE
└── supported topics = PRICING

POLICY_KNOWLEDGE
├── source = policy-dataset generation-12
├── status = AVAILABLE
├── freshness = STABLE
└── supported topics = REFUND_POLICY
~~~

Status baseline:

~~~text
AVAILABLE
DEGRADED
UNAVAILABLE
DISABLED
BUILDING
~~~

**Source Health** = trạng thái hoạt động hiện tại của source. Source có cấu hình nhưng đang lỗi không được coi như AVAILABLE.

### Source Resolver

**Source Resolver** = chọn source cụ thể từ các capability candidate đã:
1. nằm trong AllowedScope;
2. đúng task/topic/entity/freshness;
3. đang AVAILABLE hoặc DEGRADED theo policy;
4. đáp ứng authority requirement.

LLM không được tự chọn runtime source ID.

### Query Planner

**Query Planner** nhận semantic understanding + source binding đã hợp lệ để tạo QueryPlan.

Planner quyết định **WHAT / WHERE**:
- cần làm gì;
- dùng source/dataset nào;
- có cần chia subplan không.

Retriever quyết định **HOW** search.

### QueryPlan Validator

**QueryPlan Validator** = bước cuối trước execution.

Nó kiểm:
- plan type hợp lệ;
- required entity có đủ;
- source binding vẫn hợp lệ;
- scope không bị mở rộng;
- source/capability đang usable;
- execution budget hợp lệ;
- retrieval/answer profile reference tồn tại.

Plan invalid phải fail closed.

## 10. Plan Executor và bounded decomposition

**Plan Executor** = khối thực thi QueryPlan đã validate.

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

### Bounded decomposition

**Query Decomposition** = tách câu hỏi phức tạp thành một số câu hỏi nhỏ hơn.

Ví dụ:

~~~text
"Giá VPS Basic hiện tại và chính sách hoàn tiền của gói này?"
~~~

có thể tách:

~~~text
Subplan A → CURRENT_PRICING
Subplan B → REFUND_POLICY
~~~

Raglyra V1/V2 kiến trúc **không dùng autonomous recursive agent** làm mặc định.

Mọi decomposition phải bị giới hạn:

~~~text
max_subplans
max_depth
max_source_calls
max_retrieval_calls
max_llm_calls
time_budget
~~~

Mỗi subplan vẫn dùng cùng AllowedScope; decomposition không được phép mở dataset/source mới.

## 11. Structured Sources

**Structured Source** = nguồn dữ liệu có cấu trúc rõ và thường là nguồn chính xác cho dữ liệu exact/current.

Ví dụ:
- Pricing API;
- Inventory API;
- SQL table;
- business service.

Ví dụ user hỏi giá realtime thì Pricing API thường phù hợp hơn vector document, vì vector snapshot có thể cũ.

## 12. Retrieval Engine

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

## 13. Candidate

**Candidate** = kết quả tạm thời lấy từ retrieval lane.

Candidate chưa phải evidence cuối cùng.

## 14. Fusion

**Fusion** = gộp nhiều bảng xếp hạng thành một ranking chung.

Baseline có thể dùng **RRF — Reciprocal Rank Fusion**.

RRF gộp dựa trên thứ hạng thay vì cộng trực tiếp các score khác thang đo.

## 15. Reranker

**Reranker** = mô hình xếp hạng lại một tập candidate nhỏ để tăng precision.

~~~text
50 candidate
→ reranker
→ top 8
~~~

**Precision** = tỷ lệ kết quả được chọn thật sự liên quan.

**Recall** = khả năng tìm đủ các bằng chứng có liên quan.

## 16. Evidence Layer

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

## 17. Evidence Gate

**Evidence Gate** = bước quyết định bằng chứng có đủ để trả lời hay không.

Possible outputs:

~~~text
ANSWERABLE
PARTIAL
NO_ANSWER
NEEDS_CLARIFICATION
~~~

Ví dụ user hỏi so sánh A và B nhưng chỉ có evidence cho A thì phải trả PARTIAL, không cho LLM tưởng tượng B.

## 18. Answer Engine

Answer Engine chọn strategy phù hợp:

### Deterministic
Code format trực tiếp từ typed data.

### Extractive
Lấy gần trực tiếp câu trả lời từ evidence.

### Grounded Synthesis
LLM tổng hợp nhiều evidence nhưng phải bị giới hạn bởi evidence đã chọn.

## 19. Citation Builder

Citation Builder nhận provenance của Evidence và tạo citation an toàn.

LLM không được tự bịa source URL.

## 20. Ingestion Engine

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

Chi tiết nằm ở `docs/02-luong-nap-tri-thuc/luong-nap-va-danh-chi-muc-v2.md`.

## 21. Storage Layer

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

## 22. Evaluation & Observability

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

## 23. Infrastructure Abstraction

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

## 24. Ví dụ end-to-end — policy query

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

## 25. Ví dụ end-to-end — current price

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

## 26. Error Philosophy

Nếu scope không chắc thì **fail closed**.

**Fail Closed** = khi không chắc quyền/phạm vi thì từ chối an toàn thay vì mở rộng quyền.

Nếu evidence thiếu:

~~~text
NO_ANSWER / PARTIAL / CLARIFY
~~~

không ép LLM trả lời.

## 27. Những gì file này chưa chốt

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

## 28. Design Decisions

1. Raglyra có Knowledge Pipeline và Query Pipeline tách rõ.
2. Allowed Scope được resolve từ trusted context trước Query Understanding.
3. Conversation được xử lý trước Query Understanding.
4. Policy Scope Validation chạy sau Query Understanding + Entity Resolution.
5. Capability Registry tách capability khỏi runtime source.
6. Source Resolver chỉ chọn trong candidate đã authorized và available.
7. Query Planner tạo QueryPlan, QueryPlan Validator kiểm trước execute.
8. Intent không được tự mở source/capability ngoài scope.
9. Planner quyết định WHAT/WHERE; Retrieval quyết định HOW.
10. Complex query chỉ được bounded decomposition, không autonomous recursion mặc định.
11. Current structured data không mặc định lấy từ vector snapshot.
12. Candidate khác Evidence.
13. Evidence Gate đứng trước generation.
14. Citation build từ provenance.
15. Core độc lập pgvector/Qdrant/provider cụ thể.
16. Security scope đi xuyên pipeline.
17. Evaluation/Observability là core capability.

## 29. Review Checklist

- [ ] Hiểu hai pipeline lớn.
- [ ] Đồng ý Conversation đứng trước Query Understanding.
- [ ] Đồng ý QueryPlan là contract execution.
- [ ] Đồng ý current structured source không nhất thiết đi RAG.
- [ ] Đồng ý Retrieval trả Evidence, không answer.
- [ ] Đồng ý Evidence Gate.
- [ ] Đồng ý provider/backend abstraction.
- [ ] Đồng ý Evaluation là first-class.

## 30. Next Step

Đọc tiếp:

~~~text
01-query-answer-flow.md
↓
02-ingestion-indexing-flow.md
~~~

Sau khi hai flow được REVIEWED mới đi vào Component Design.