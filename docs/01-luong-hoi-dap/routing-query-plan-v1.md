# Raglyra — Routing & QueryPlan v1

> **Status:** DRAFT
> **Thuộc chức năng:** 01-luong-hoi-dap — Query Pipeline
> **Vị trí trong flow:** Sau Conversation Engine / StandaloneQuery, trước Plan Executor / Retrieval.
> **Mục tiêu:** Biến query đã được giải nghĩa thành một kế hoạch thực thi rõ ràng, đúng capability, đúng source, đúng scope và không tự mở rộng dữ liệu.

---

## 1. Routing là gì?

**Routing** = quá trình quyết định query cần được xử lý theo con đường nào.

Ví dụ:

~~~text
Xin chào
→ DIRECT

Giá VPS Basic hiện tại?
→ STRUCTURED_SOURCE

ERR_SSL_298 là gì?
→ EXACT_SEARCH

Chính sách hoàn tiền?
→ RAG
~~~

Routing không trực tiếp tìm dữ liệu; nó tạo **QueryPlan**.

## 2. QueryPlan là gì?

**QueryPlan** = hợp đồng thực thi mà downstream component có thể chạy mà không cần đoán lại ý định.

Ví dụ:

~~~text
plan_type = STRUCTURED_SOURCE
task = STRUCTURED_LOOKUP
topic = PRICING
entity = product_vps_basic
freshness = CURRENT
source_binding = pricing-api
~~~

## 3. Vị trí trong flow

~~~text
ResolvedTurn + StandaloneQuery
↓
Query Understanding
↓
Entity Resolution
↓
Capability Requirement
↓
Policy Scope Validation
↓
Capability Registry
↓
Source Resolver
↓
Query Planner
↓
QueryPlan Validator
↓
Plan Executor
~~~

## 4. Input

Logical input:

~~~text
AllowedScope
ResolvedTurn
StandaloneQuery
DomainTaxonomy
CapabilityRegistry snapshot
RoutingPolicy/Profile
request_id
~~~

## 5. Output

Output duy nhất:

~~~text
Validated QueryPlan
~~~

Hoặc terminal decision:

~~~text
CLARIFY
OUT_OF_SCOPE
UNSUPPORTED
FORBIDDEN
NO_ANSWER
~~~

## 6. Query Understanding

**Query Understanding** = phân tích semantics của query.

Structured output:

~~~text
task_type
interaction_type
topic
entity_mentions[]
freshness
time_range
constraints
ambiguity
multi_task flag
answer_shape
~~~

Không free-text output nếu có thể.

## 7. Query Understanding không được làm gì?

Không được:

~~~text
chọn tenant
chọn runtime source ID
mở dataset
quyết định permission
chọn vector collection
tự gọi tool
tự quyết định top_k
~~~

Nó chỉ hiểu semantics.

## 8. Task Type

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
UNSUPPORTED
~~~

## 9. Interaction Type

~~~text
NORMAL_QUERY
CLARIFICATION_RESPONSE
~~~

Interaction Type không phải business Task Type.

## 10. Topic

**Topic** = chủ đề nghiệp vụ được cấu hình.

Ví dụ:

~~~text
PRICING
REFUND_POLICY
PRODUCT
TECHNICAL
ACCOUNT
~~~

Topic không được là free-form vô hạn do LLM tự phát minh.

## 11. Freshness

~~~text
CURRENT
RECENT
STABLE
HISTORICAL
UNKNOWN
~~~

Freshness là routing constraint thật.

Ví dụ CURRENT_PRICE không được tự fallback sang tài liệu cũ nếu policy không cho phép.

## 12. Entity Extraction

**Entity Extraction** = phát hiện mention trong text.

Ví dụ:

~~~text
"VPS Basic giá bao nhiêu?"
→ raw mention = VPS Basic
~~~

## 13. Entity Resolution

**Entity Resolution** = map raw mention về canonical entity.

Ví dụ:

~~~text
VPS Basic
→ product_vps_basic
~~~

Resolution precedence:

~~~text
exact ID
→ exact alias
→ normalized name
→ lexical candidates
→ semantic entity candidates
→ conversation state tie-break
→ CLARIFY
~~~

Không dùng full knowledge retrieval để đoán entity mơ hồ.

## 14. Capability Requirement

**Capability Requirement** = mô tả khả năng cần có để xử lý query.

Ví dụ:

~~~text
STRUCTURED_LOOKUP + PRICING + CURRENT
→ CURRENT_PRICING

KNOWLEDGE_QA + REFUND_POLICY + STABLE
→ POLICY_KNOWLEDGE
~~~

Capability khác Source.

## 15. AllowedScope

AllowedScope đã được Security layer tạo trước.

Routing chỉ được chọn trong:

~~~text
allowed datasets
allowed sources
allowed capabilities
allowed actions
~~~

Routing không có quyền mở rộng AllowedScope.

## 16. Policy Scope Validation

Kiểm:

~~~text
topic có thuộc assistant domain?
capability requirement có được phép?
entity có nằm trong resource scope?
action có được phép?
freshness có candidate source hợp lệ?
~~~

Possible result:

~~~text
ALLOWED
CLARIFY
OUT_OF_SCOPE
UNSUPPORTED
FORBIDDEN
~~~

## 17. Capability Registry

**Capability Registry** = runtime registry cho biết capability nào đang tồn tại và source nào cung cấp.

Logical entry:

~~~text
capability_id
supported_task_types
supported_topics
supported_entity_types
freshness_classes
source_bindings[]
authority
availability_status
profile/version
~~~

## 18. Availability Status

~~~text
AVAILABLE
DEGRADED
UNAVAILABLE
DISABLED
BUILDING
~~~

AVAILABLE mới là normal candidate.

DEGRADED chỉ dùng nếu policy cho phép.

UNAVAILABLE/DISABLED/BUILDING không được chọn cho production query bình thường.

## 19. Authority

**Authority** = mức độ source được coi là nguồn chính thức cho một loại fact.

Ví dụ:

~~~text
current price → Pricing API = AUTHORITATIVE
refund terms → approved Policy Document = AUTHORITATIVE
blog summary → SECONDARY
~~~

Authority không thay relevance; source vẫn phải đúng topic/entity.

## 20. Source Resolver

**Source Resolver** = chọn concrete source binding từ capability candidates đã được authorize.

Precedence:

~~~text
1. explicit user source nếu được phép
2. assistant-bound source
3. scope/authorization
4. capability match
5. freshness
6. authority
7. entity/topic compatibility
8. semantic dataset rank nếu cần
9. explicit fallback policy
~~~

## 21. Semantic Dataset Routing

Chỉ dùng khi có nhiều dataset đã authorized và metadata/rules chưa đủ để chọn.

Semantic ranking:

~~~text
query/topic representation
vs
dataset/capability descriptors
~~~

Semantic score không phải permission.

## 22. QueryPlan Types

~~~text
DIRECT
CLARIFY
STRUCTURED_SOURCE
EXACT_SEARCH
RAG
MULTI_PLAN
NO_ANSWER
~~~

## 23. QueryPlan Common Fields

Logical schema:

~~~text
plan_id
plan_type
scope_reference
authorization_revision
task_type
topic
resolved_entities[]
freshness
constraints
source_bindings[]
retrieval_profile_id
answer_profile_id
fallback_policy_id
execution_budget
reason_codes[]
taxonomy_version
routing_policy_version
query_understanding_profile_version
~~~

Không chứa secret hoặc connection string.

## 24. DIRECT Plan

Dùng cho greeting/acknowledgement/static help.

Không retrieval.

## 25. CLARIFY Plan

Logical fields:

~~~text
clarification_type
missing_slots
candidate_values
reason_code
~~~

CLARIFY dùng khi user có thể bổ sung thông tin.

Source lỗi không phải CLARIFY.

## 26. STRUCTURED_SOURCE Plan

Dùng cho typed/current data.

Ví dụ:

~~~text
source_binding = pricing-api
capability = CURRENT_PRICING
entity = product_vps_basic
freshness = CURRENT
~~~

## 27. EXACT_SEARCH Plan

Dùng cho:

~~~text
SKU
error code
document number
contract code
exact clause
~~~

Exact-first; RAG fallback chỉ nếu policy cho phép.

## 28. RAG Plan

Chứa:

~~~text
selected dataset/source bindings
metadata filters
retrieval_profile_id
required coverage
freshness/authority policy
~~~

Không nhét chi tiết backend như Qdrant collection name nếu có thể.

## 29. MULTI_PLAN

**MULTI_PLAN** = plan chứa nhiều subplan hữu hạn.

Ví dụ:

~~~text
Giá VPS Basic và chính sách hoàn tiền?
~~~

~~~text
Subplan A → STRUCTURED_SOURCE / CURRENT_PRICING
Subplan B → RAG / REFUND_POLICY
~~~

## 30. Bounded Decomposition

Không dùng recursive autonomous planner mặc định.

Execution budget:

~~~text
max_subplans
max_depth
max_source_calls
max_retrieval_calls
max_llm_calls
time_budget
~~~

Mọi subplan kế thừa cùng AllowedScope.

## 31. Dependency giữa subplan

Subplan có thể:

~~~text
PARALLEL
SEQUENTIAL
~~~

V1 ưu tiên independent/parallel subplans.

Dependent multi-hop chỉ hỗ trợ khi input/output mapping rõ và depth nhỏ.

## 32. Fallback Policy

Fallback phải explicit.

Không:

~~~text
pricing API fail
→ search all tenant docs
~~~

Có thể:

~~~text
pricing API fail
→ stale snapshot
~~~

chỉ khi policy đã cấu hình và response bắt buộc có freshness warning.

## 33. NO_ANSWER

Reason codes:

~~~text
NO_CAPABILITY
NO_ALLOWED_SOURCE
SOURCE_UNAVAILABLE
UNSUPPORTED_FRESHNESS
OUT_OF_SCOPE
FORBIDDEN
PLAN_INVALID
~~~

Không ép LLM trả lời.

## 34. QueryPlan Validator

Check trước execute:

~~~text
scope reference hợp lệ?
authorization/resource revision còn hợp lệ?
required entity đủ?
source binding đúng?
capability usable?
freshness compatible?
profile reference tồn tại?
execution budget hợp lệ?
~~~

Fail closed.

## 35. TOCTOU / Revalidation

**TOCTOU** = quyền/resource có thể thay đổi giữa lúc plan và execute.

Executor phải critical re-check:

~~~text
assistant active?
source active?
binding revision?
resource revoked?
capability status?
~~~

## 36. Reason Codes

Ví dụ:

~~~text
RULE_GREETING
MODEL_KNOWLEDGE_QA
ENTITY_AMBIGUOUS
CAPABILITY_CURRENT_PRICING
SOURCE_AUTHORITATIVE_SELECTED
SOURCE_UNAVAILABLE
FALLBACK_TO_SNAPSHOT
NO_ALLOWED_SOURCE
~~~

Reason code giúp trace decision, không phải user-facing text bắt buộc.

## 37. Routing Trace

Trace:

~~~text
original/standalone query refs
query understanding output
entity resolution
capability requirement
policy validation
candidate capabilities
candidate sources
selected source
routing policy
QueryPlan
plan validation
fallback/degradation
latency
profile/model versions
~~~

Không log secret.

## 38. Failure Modes

### Query Understanding schema invalid
Retry bounded hoặc deterministic fallback; nếu không thì NO_ANSWER/CLARIFY.

### Entity ambiguous
CLARIFY.

### No capability
NO_ANSWER / UNSUPPORTED.

### Capability exists nhưng source unavailable
Explicit fallback hoặc NO_ANSWER.

### Plan violates scope
FORBIDDEN / PLAN_INVALID.

### MultiPlan vượt budget
Reject/trim theo policy, không chạy tiếp vô hạn.

## 39. Security Considerations

Routing không được:
- suy tenant từ query;
- thêm source ngoài AllowedScope;
- biến semantic score thành permission;
- expose source secret;
- bypass resource revoke.

## 40. Tests bắt buộc

~~~text
greeting → DIRECT
current price → STRUCTURED_SOURCE
policy query → RAG
exact code → EXACT_SEARCH
ambiguous product → CLARIFY
out-of-scope → NO_ANSWER without retrieval
source unavailable → explicit fallback/no-answer
multi-task → bounded MULTI_PLAN
forbidden dataset → never selected
revoked source between plan/execute → fail closed
~~~

## 41. Golden Routing Dataset

**Golden Routing Dataset** = bộ query có expected routing đã human-review.

Mỗi case có thể chứa:

~~~text
query / ResolvedTurn
expected task
expected topic
expected entity
expected freshness
expected capability
expected plan type
allowed source
forbidden source
~~~

## 42. Metrics

~~~text
Task Accuracy
Topic Accuracy
Entity Resolution Accuracy
Freshness Accuracy
Capability Selection Accuracy
Source Selection Accuracy
Plan Accuracy
Clarification Precision
Unsafe Route Rate
Multi-task Detection Accuracy
Latency
Cost
~~~

Unsafe Route Rate phải cực thấp.

## 43. Non-responsibilities

Routing không chịu trách nhiệm:
- chunking;
- vector similarity implementation;
- fusion/reranker chi tiết;
- final answer generation;
- citation rendering;
- persistence schema cụ thể.

## 44. Dependencies

Phụ thuộc:
- Conversation Engine outputs;
- DomainTaxonomy;
- Entity Resolver;
- AllowedScope;
- Capability Registry;
- Routing Policy/Profile.

## 45. Extension Points

Future:
- semantic source router nâng cao;
- policy DSL;
- action/tool capability;
- graph query plan;
- learned router.

Extension không được phá scope invariant.

## 46. Design Decisions v1

1. Routing nhận ResolvedTurn/StandaloneQuery, không raw history.
2. Query Understanding không chọn runtime source.
3. Task/Interaction/Topic tách nhau.
4. Entity Resolution là explicit stage.
5. Capability Requirement khác Source.
6. Policy validation chỉ thu hẹp AllowedScope.
7. Capability Registry quản lý runtime availability.
8. Source Resolver tách khỏi classifier.
9. QueryPlan là single execution contract.
10. QueryPlan phải validate trước execute.
11. MultiPlan bounded.
12. Fallback explicit.
13. Semantic ranking không cấp permission.
14. Routing trace/reason codes first-class.

## 47. Open Questions

- Query Understanding provider/model nào?
- Topic taxonomy lưu DB/config thế nào?
- Capability Registry materialized hay computed/cache?
- Entity catalog/index ở đâu?
- Policy DSL có cần ngay V1 không?
- Semantic dataset routing threshold?
- Execution budget mặc định?

## 48. Review Checklist

- [ ] Task/Topic/Capability/Source đã tách rõ?
- [ ] AllowedScope có được giữ xuyên routing?
- [ ] LLM có chỗ nào tự chọn source không?
- [ ] Source unavailable có mở search rộng không?
- [ ] QueryPlan có validator không?
- [ ] MultiPlan có budget không?
- [ ] Reason codes/trace đủ debug không?
- [ ] Golden routing tests đủ edge cases không?

## 49. Next Step

Routing / QueryPlan REVIEWED thì đi tiếp Retrieval Engine.