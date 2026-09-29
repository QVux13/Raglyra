# Raglyra — Query & Answer Flow

> **Status:** DRAFT
> **Mục tiêu:** Mô tả chi tiết toàn bộ luồng từ lúc user gửi một message cho tới khi Raglyra trả response.
> **Vị trí trong hệ thống:** Đây là Query Pipeline. File này không mô tả ingestion/chunking chi tiết.

---

## 1. Luồng tổng thể

~~~mermaid
flowchart TD
    A[User Message] --> B[Resolve Security Context]
    B --> C[Load Conversation Context]
    C --> D[Resolve Reference / Clarification]
    D --> E[Build Standalone Query]
    E --> F[Query Understanding]
    F --> G[Entity Resolution]
    G --> H[Capability Resolution]
    H --> I[Build QueryPlan]

    I -->|DIRECT| J[Direct Response]
    I -->|CLARIFY| K[Clarification]
    I -->|STRUCTURED| L[Structured Source]
    I -->|EXACT| M[Exact Search]
    I -->|RAG| N[Retrieval Engine]
    I -->|MULTI| O[SubPlans]

    N --> P[Candidate Retrieval]
    P --> Q[Fusion]
    Q --> R[Reranker]
    R --> S[Evidence Selection]

    L --> T[Evidence / Typed Result]
    M --> T
    S --> T
    O --> T

    T --> U[Evidence Gate]
    U --> V[Answer Strategy]
    V --> W[Grounding Validation]
    W --> X[Citation Builder]
    X --> Y[Response]
    Y --> Z[Conversation State Update]
~~~

## 2. Input của flow

Input tối thiểu:

~~~text
message
conversation/session reference
assistant/bot reference
authentication/session context
optional client metadata
~~~

Raw message chưa chắc đủ nghĩa và không được mặc định đưa thẳng vào vector search.

## 3. Bước 1 — Resolve Security Context

Xác định:

~~~text
tenant
workspace
assistant/bot
actor/session
permissions
allowed datasets/sources
request_id
~~~

Mục tiêu là biết request được phép nhìn thấy dữ liệu nào trước khi routing/retrieval.

Nếu scope không hợp lệ thì dừng. Không fallback sang global data.

## 4. Bước 2 — Load Conversation Context

**Conversation Context** = ngữ cảnh cuộc trò chuyện hiện tại.

Có thể gồm:

~~~text
recent_turns
active_entities
current_topic
current_task
user_constraints
pending_clarification
rolling_summary
~~~

Ví dụ:

~~~text
User: Tôi muốn xem VPS Basic.
Bot: ...
User: Còn giá?
~~~

Context cần nhớ:

~~~text
active_entity = VPS Basic
~~~

## 5. Recent Turns, Summary và Structured State khác nhau

### Recent Turns
Các lượt chat gần nhất, dùng để hiểu câu chữ và tham chiếu.

### Summary
Bản tóm tắt conversation dài để tiết kiệm token.

### Structured State
Dữ liệu có cấu trúc như active_entity, pending_clarification, constraints.

Structured State đáng tin hơn việc parse summary lại mỗi lượt.

Summary không phải authoritative business evidence.

## 6. Bước 3 — Reference Resolution

**Reference Resolution** = giải nghĩa các từ/câu phụ thuộc lịch sử.

Ví dụ:

~~~text
"Còn giá?"
→ "Giá của VPS Basic?"

"Nó có hoàn tiền không?"
→ "VPS Basic có hoàn tiền không?"
~~~

Nếu có nhiều cách hiểu hợp lý thì phải CLARIFY — hỏi lại user, không đoán bừa.

## 7. Pending Clarification

Ví dụ hệ thống vừa hỏi:

~~~text
Bạn hỏi VPS Basic hay Hosting Basic?
~~~

User trả:

~~~text
VPS
~~~

Lượt này phải được thử resolve như câu trả lời cho clarification đang pending trước khi routing như một query mới.

## 8. Bước 4 — Standalone Query

**Standalone Query** = câu hỏi đủ nghĩa để xử lý độc lập.

Ví dụ:

~~~text
Original:
Còn giá?

Standalone:
Giá hiện tại của VPS Basic là bao nhiêu?
~~~

Không nhất thiết gọi LLM để rewrite mọi query.

Rewrite phải preserve:
- entity;
- identifier/code;
- số;
- ngày;
- phủ định;
- hai phía so sánh;
- constraint user đã nói.

## 9. Bước 5 — Query Understanding

Query Understanding phân tích Standalone Query thành semantic fields.

Ví dụ:

~~~text
task = STRUCTURED_LOOKUP
topic = PRICING
entity mention = VPS Basic
freshness = CURRENT
ambiguity = false
~~~

## 10. Task Type

**Task Type** = loại công việc hệ thống cần làm.

Baseline:

~~~text
CONVERSATIONAL
KNOWLEDGE_QA
STRUCTURED_LOOKUP
EXACT_LOOKUP
COMPARE
SUMMARIZE
MULTI_HOP
OUT_OF_SCOPE
~~~

Task generic, không phụ thuộc một ngành cụ thể.

## 11. Topic

**Topic** = chủ đề nghiệp vụ.

Ví dụ:

~~~text
PRICING
REFUND_POLICY
PRODUCT
TECHNICAL
CONTRACT
~~~

Task và Topic phải tách.

## 12. Freshness

**Freshness** = mức độ mới của dữ liệu mà query yêu cầu.

~~~text
CURRENT
RECENT
STABLE
HISTORICAL
UNKNOWN
~~~

Ví dụ:

~~~text
"giá hiện tại"
→ CURRENT
~~~

Freshness ảnh hưởng trực tiếp việc chọn source.

## 13. Bước 6 — Entity Resolution

**Entity Extraction** = nhận ra text nào là entity.

**Entity Resolution** = map text đó về entity canonical trong hệ thống.

Ví dụ:

~~~text
"VPS Basic"
→ product_vps_basic
~~~

Resolution precedence có thể là:

~~~text
exact ID
→ alias
→ normalized name
→ lexical candidates
→ semantic candidates
→ clarify
~~~

## 14. Bước 7 — Capability Resolution

**Capability** = khả năng hệ thống có để xử lý một loại nhu cầu.

Ví dụ:

~~~text
CURRENT_PRICING
POLICY_KNOWLEDGE
PRODUCT_CATALOG
TECHNICAL_KNOWLEDGE
CURRENT_STOCK
~~~

Capability khác Source:

~~~text
Capability = cần khả năng gì
Source = implementation nào cung cấp khả năng đó
~~~

## 15. Source Binding

Ví dụ:

~~~text
CURRENT_PRICING
→ PricingServiceAdapter

POLICY_KNOWLEDGE
→ Policy Dataset Generation 12
~~~

LLM không tự chọn arbitrary source ID.

## 16. Bước 8 — QueryPlan

**QueryPlan** = kế hoạch máy có thể thực thi.

Plan types:

~~~text
DIRECT
CLARIFY
STRUCTURED_SOURCE
EXACT_SEARCH
RAG
MULTI_PLAN
NO_ANSWER
~~~

## 17. DIRECT

Dùng cho greeting/acknowledgement/static help.

Không retrieval.

## 18. CLARIFY

Dùng khi thiếu thông tin user có thể bổ sung.

Ví dụ:

~~~text
"giá bao nhiêu?"
~~~

nhưng không biết product nào.

Không vector search để đoán.

## 19. STRUCTURED_SOURCE

Dùng khi có source typed/current phù hợp.

Ví dụ:

~~~text
CURRENT_PRICING
CURRENT_STOCK
customer status
~~~

Structured result có thể trả trực tiếp bằng deterministic formatter.

## 20. EXACT_SEARCH

Dùng cho identifier:

~~~text
ERR_SSL_298
SKU-001
document number
contract code
~~~

Exact/lexical-first thường phù hợp hơn dense-first.

## 21. RAG Plan

Dùng khi cần retrieve knowledge.

Plan nên chứa:

~~~text
selected dataset/source bindings
filters
retrieval profile
coverage requirement
scope
~~~

Routing không nên nhét chi tiết dense_top_k/reranker model vào QueryPlan nếu các chi tiết đó thuộc RetrievalProfile.

## 22. MULTI_PLAN

Ví dụ:

~~~text
"VPS Basic giá bao nhiêu và có hoàn tiền không?"
~~~

Tạo:

~~~text
SubPlan A
→ STRUCTURED_SOURCE / CURRENT_PRICING

SubPlan B
→ RAG / REFUND_POLICY
~~~

MultiPlan phải có execution budget để tránh chạy vô hạn.

## 23. NO_ANSWER

Dùng khi:
- không có capability;
- source unavailable;
- permission không đủ;
- freshness không đáp ứng;
- query out-of-scope.

Không ép LLM bịa.

## 24. Bước 9 — Retrieval

Nếu plan là RAG:

~~~text
Prepared Query
├── Exact
├── Lexical
└── Dense
      ↓
Candidates
      ↓
Fusion
      ↓
Reranker
      ↓
Evidence Selection
~~~

## 25. Exact Retrieval

Tìm chính xác identifier, code, canonical name.

## 26. Lexical Retrieval

**Lexical Retrieval** = tìm theo từ khóa/toàn văn.

Mạnh với:
- mã;
- tên riêng;
- thuật ngữ;
- exact phrase;
- keyword hiếm.

## 27. Dense Retrieval

**Dense Retrieval** = semantic search bằng embedding vector.

Mạnh với paraphrase và các câu khác chữ nhưng cùng nghĩa.

## 28. Hybrid Retrieval

**Hybrid Retrieval** = kết hợp nhiều lane retrieval rồi fusion.

Không phải query nào cũng cần Hybrid.

## 29. Candidate

**Candidate** = kết quả retrieval tạm thời.

Logical fields có thể gồm:

~~~text
candidate_id
type
content
source reference
lane ranks
native scores
authority
freshness
provenance
~~~

## 30. Fusion

**Fusion** = gộp nhiều ranking.

Không cộng raw cosine + BM25 score tùy tiện.

RRF là baseline phù hợp để bắt đầu benchmark.

## 31. Reranker

**Reranker** = mô hình chấm lại candidate để tăng precision.

~~~text
40 candidates
→ reranker
→ top candidates
~~~

## 32. Evidence Selection

Không chỉ lấy top-K.

Cần xét:
- entity coverage;
- topic coverage;
- authority;
- freshness;
- quality;
- compare-side coverage;
- diversity nếu cần.

## 33. Evidence

**Evidence** = candidate đã được chọn làm căn cứ.

Evidence phải giữ provenance.

Ví dụ:

~~~text
E1
source = policy.pdf
version = v3
page = 12
section = Refund
~~~

## 34. Bước 10 — Evidence Gate

Kiểm:

~~~text
đúng entity?
đủ coverage?
đủ authority?
freshness phù hợp?
compare đủ hai phía?
no-match hay negative fact?
~~~

**No-match** = không tìm thấy evidence.

No-match không đồng nghĩa dữ liệu không tồn tại.

## 35. Bước 11 — Answer Strategy

### Deterministic
Code format typed data.

### Extractive
Lấy gần trực tiếp từ evidence.

### Grounded Synthesis
LLM tổng hợp evidence nhưng không được vượt evidence.

## 36. Bước 12 — Grounding Validation

Kiểm các factual claim quan trọng:
- số;
- ngày;
- code;
- entity;
- phủ định;
- claim có evidence hỗ trợ hay không.

Nếu critical claim unsupported thì fallback hoặc NO_ANSWER/PARTIAL.

## 37. Bước 13 — Citation

Citation được build từ Evidence provenance.

LLM có thể dùng Evidence ID nội bộ nhưng public citation do server tạo.

## 38. Bước 14 — Conversation State Update

Sau response cập nhật interaction state:

~~~text
active_entity
current_topic
pending clarification
user constraints
~~~

Không lấy unsupported LLM claim ghi thành state/fact.

## 39. Case hoàn chỉnh — Follow-up Price

~~~text
User: VPS Basic có gì?
Bot: ...
User: Còn giá?
↓
ConversationContext
active product = VPS Basic
↓
Standalone = Giá hiện tại của VPS Basic là bao nhiêu?
↓
Query Understanding
STRUCTURED_LOOKUP + PRICING + CURRENT
↓
Capability CURRENT_PRICING
↓
STRUCTURED_SOURCE
↓
Pricing adapter
↓
typed result
↓
deterministic answer
~~~

## 40. Case hoàn chỉnh — Policy

~~~text
User: Nếu tôi không dùng VPS nữa thì có được hoàn tiền không?
↓
KNOWLEDGE_QA
↓
REFUND_POLICY
↓
POLICY_KNOWLEDGE
↓
RAG
↓
lexical + dense
↓
fusion
↓
reranker
↓
evidence
↓
grounded answer
↓
citation
~~~

## 41. Case hoàn chỉnh — Multi-intent

~~~text
User: VPS Basic giá bao nhiêu và nếu không hợp thì có hoàn tiền không?
↓
Task A + Task B
↓
MULTI_PLAN
├── pricing structured
└── refund RAG
↓
merge results
↓
grounded synthesis
↓
citations
~~~

## 42. Error / Edge Cases

### Conversation ambiguous
→ CLARIFY.

### Entity không resolve được
→ CLARIFY hoặc NO_ANSWER.

### Source unavailable
→ fallback chỉ khi policy cho phép.

### Retrieval empty
→ NO_MATCH, không bịa.

### Generation invalid
→ bounded fallback, không retry vô hạn.

## 43. Trách nhiệm component

| Component | Trách nhiệm |
|---|---|
| Security Resolver | resolve allowed scope |
| Conversation Engine | hiểu multi-turn context |
| Reference Resolver | resolve pronoun/ellipsis |
| Query Understanding | task/topic/freshness |
| Entity Resolver | canonical entity |
| Capability Resolver | tìm capability phù hợp |
| Query Planner | tạo QueryPlan |
| Plan Executor | execute plan |
| Retriever | tìm candidate |
| Fusion | gộp ranking |
| Reranker | xếp hạng lại |
| Evidence Selector | chọn evidence |
| Evidence Gate | quyết định đủ bằng chứng |
| Answer Engine | tạo answer |
| Grounding Validator | kiểm claim |
| Citation Builder | tạo citation |
| Conversation State Manager | cập nhật state |

## 44. File này không mô tả

- parser/chunking chi tiết;
- DB schema;
- API DTO;
- provider/model cụ thể;
- UI;
- deployment.

## 45. Design Decisions

1. Conversation resolution trước routing/retrieval.
2. Query Understanding trả semantic fields.
3. Entity Resolution tách riêng.
4. Capability khác Source.
5. QueryPlan là contract execution.
6. Current typed data ưu tiên structured source.
7. Retrieval không tự route lại.
8. Candidate khác Evidence.
9. Evidence Gate trước generation.
10. No-match khác negative fact.
11. Citation server-side.
12. Conversation state không phải Knowledge Base.

## 46. Open Questions

- exact ConversationState schema;
- Query Understanding model;
- Entity catalog implementation;
- Capability Registry storage;
- RetrievalProfile schema;
- reranker provider;
- thresholds;
- exact QueryPlan DTO;
- cache/retry/timeouts.

## 47. Review Checklist

- [ ] Hiểu flow từ message tới response.
- [ ] Đồng ý Conversation-first.
- [ ] Đồng ý QueryPlan.
- [ ] Đồng ý Capability abstraction.
- [ ] Đồng ý structured/current source.
- [ ] Đồng ý Retrieval lanes.
- [ ] Đồng ý Evidence Gate.
- [ ] Đồng ý citation/provenance.
- [ ] Đồng ý no-match semantics.

## 48. Next Step

Sau khi file này REVIEWED, review `02-ingestion-indexing-flow.md`, rồi mới bắt đầu Component Design.