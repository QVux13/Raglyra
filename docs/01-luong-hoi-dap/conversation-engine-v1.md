# Raglyra — Conversation Engine v1

> **Status:** DRAFT
> **Thuộc chức năng:** 01-luong-hoi-dap — Query Pipeline
> **Vị trí trong flow:** Sau Security/AllowedScope, trước Query Understanding.
> **Mục tiêu:** Biến một message có thể phụ thuộc lịch sử thành một lượt chat đã được hiểu đủ ngữ cảnh để Query Understanding xử lý ổn định.

---

## 1. Conversation Engine là gì?

**Conversation Engine** = khối chịu trách nhiệm hiểu ngữ cảnh nhiều lượt hội thoại.

Ví dụ:

~~~text
User: Tôi đang xem VPS Basic.
Bot: ...
User: Còn giá?
~~~

Message cuối không đủ nghĩa nếu xử lý độc lập.

Conversation Engine phải hiểu:

~~~text
active entity = VPS Basic
~~~

và tạo semantic input đầy đủ cho bước sau.

Conversation Engine không phải Retrieval Engine và không có quyền tự search knowledge.

## 2. Vì sao component này bắt buộc?

Nếu bỏ Conversation Engine:

~~~text
"Còn giá?"
→ vector search
~~~

retrieval có thể tìm sai sản phẩm hoặc sai context.

Nguyên tắc:

~~~text
hiểu đang nói về cái gì
trước
tìm dữ liệu về cái đó
~~~

## 3. Vị trí trong Query Pipeline

~~~text
User Message
↓
Security Context / AllowedScope
↓
Conversation Engine
├── Load ConversationContext
├── Resolve Pending Clarification
├── Resolve Explicit Correction
├── Resolve Reference / Ellipsis
└── Build Standalone Query
↓
ResolvedTurn + StandaloneQuery
↓
Query Understanding
~~~

## 4. Input

Logical input:

~~~text
conversation_id
current_message
actor/session reference
assistant_id
AllowedScope reference
request_id
~~~

Optional metadata có thể gồm locale, channel, timestamp và attachment reference.

## 5. ConversationContext

**ConversationContext** = typed object chứa phần context cần thiết của conversation.

Baseline:

~~~text
ConversationContext
├── conversation_id
├── recent_turns
├── rolling_summary
├── interaction_state
├── pending_clarification
└── state_version
~~~

Typed contract giúp mọi component dùng cùng cách hiểu về conversation.

## 6. Recent Turns

**Recent Turns** = cửa sổ nhỏ các lượt chat gần nhất.

Dùng để hiểu:
- đại từ;
- câu rút gọn;
- correction;
- continuity ngắn hạn.

Không gửi toàn bộ history vô hạn vào mọi LLM call.

## 7. Rolling Summary

**Rolling Summary** = bản tóm tắt conversation dài để giảm token.

Có thể giữ:
- user goal;
- entities đã bàn;
- constraints;
- unresolved issues.

Summary không phải authoritative business evidence.

## 8. Interaction State

**Interaction State** = trạng thái có cấu trúc của hội thoại.

Baseline:

~~~text
active_entities[]
current_task
current_topic
user_goal
constraints{}
pending_clarification
last_resolved_turn
~~~

Interaction State không phải Knowledge Base.

Ví dụ:

~~~text
User: Tôi chọn VPS Pro
→ active entity = VPS Pro
~~~

Nhưng retrieval không tìm thấy giá thì không được ghi thành fact rằng VPS Pro không có giá.

## 9. State Provenance

**Provenance** = nguồn gốc khiến state có giá trị hiện tại.

Ví dụ:

~~~text
value = VPS Basic
source_turn_id = turn_12
resolution_method = EXPLICIT_MENTION
explicit_or_inferred = EXPLICIT
~~~

Giúp debug, correction và evaluation.

## 10. Explicit Correction

User correction phải có precedence cao.

Ví dụ:

~~~text
User: Tôi đang hỏi VPS Basic.
...
User: Không, ý tôi là Hosting Basic.
~~~

Result:

~~~text
active entity = Hosting Basic
~~~

Precedence:

~~~text
1. explicit correction hiện tại
2. explicit entity hiện tại
3. pending clarification resolution
4. structured active state
5. recent-turn inference
6. rolling-summary inference
~~~

## 11. Pending Clarification

**Pending Clarification** = state cho câu hỏi làm rõ mà assistant đang chờ user trả lời.

Logical state:

~~~text
clarification_id
type
missing_slots
candidate_values
created_turn_id
status
~~~

Status:

~~~text
PENDING
RESOLVED
SUPERSEDED
EXPIRED
~~~

## 12. Clarification Resolution

Turn mới phải được thử map vào clarification PENDING trước query routing bình thường.

Ví dụ:

~~~text
System: VPS Basic hay Hosting Basic?
User: VPS
~~~

Result:

~~~text
clarification = RESOLVED
entity = VPS Basic
~~~

## 13. Reference Resolution

**Reference Resolution** = giải nghĩa các từ phụ thuộc context.

Ví dụ:

~~~text
nó
gói đó
cái đầu tiên
còn giá?
thế còn hoàn tiền?
~~~

Output trước tiên là semantics, không nhất thiết là một câu rewrite.

## 14. Ellipsis

**Ellipsis** = câu bị lược bớt thành phần nhưng người nói vẫn hiểu nhờ context.

Ví dụ:

~~~text
Còn giá?
Thế Pro?
Bao lâu?
~~~

Nếu có nhiều interpretation hợp lý thì phải CLARIFY.

## 15. ResolvedTurn

**ResolvedTurn** = artifact đầu ra chính của Conversation Engine.

Logical schema:

~~~text
turn_id
original_text
interaction_type
resolved_entities[]
resolved_references[]
user_constraints{}
corrections[]
ambiguity
needs_standalone_rewrite
~~~

## 16. Standalone Query Builder

**Standalone Query** = câu đủ nghĩa để bước sau xử lý mà không cần đọc toàn history.

Ví dụ:

~~~text
Original: Còn giá?
Resolved entity: VPS Basic
Standalone: Giá hiện tại của VPS Basic là bao nhiêu?
~~~

## 17. Khi nào không cần rewrite?

Nếu message đã standalone thì giữ nguyên.

Không gọi LLM chỉ để paraphrase.

## 18. Rewrite Preservation Rules

Nếu dùng LLM rewrite phải preserve:

~~~text
canonical entity
identifier / SKU / code
numbers
date/time
negation
comparison sides
user constraints
explicit source selection
~~~

## 19. Rewrite Validation

Sau rewrite phải kiểm entity, identifier, số, phủ định, thời gian và constraint.

Nếu fail:

~~~text
deterministic fallback
hoặc
CLARIFY
~~~

## 20. Interaction Type

Conversation Engine phân biệt:

~~~text
NORMAL_QUERY
CLARIFICATION_RESPONSE
~~~

Business Task Type thuộc Query Understanding, không thuộc Conversation Engine.

## 21. State Commit Timing

Không ghi mọi inferred candidate vào committed state ngay.

Có thể dùng:

~~~text
provisional state
→ validate
→ committed state
~~~

## 22. Conversation State Update sau Answer

Answer Engine trả outcome; Conversation State Manager quyết định transition.

Ví dụ:

~~~text
CLARIFY answer
→ pending clarification = PENDING

successful explicit entity query
→ active entity = resolved entity
~~~

## 23. Long Conversation

Target strategy:

~~~text
small recent window
+ structured state
+ rolling summary
~~~

Không gửi hàng trăm message vào mọi prompt.

## 24. State Expiry / Decay

Một số state cần expiry như pending clarification, compare set, temporary filters.

Rule có thể theo turn, time, explicit supersede hoặc conversation end.

## 25. Concurrent Requests

Nếu hai request cùng đọc state version 10 và cùng commit, có nguy cơ lost update.

Cần state_version cho **Optimistic Concurrency** = chỉ commit nếu version hiện tại vẫn đúng version đã đọc.

## 26. Security Considerations

Phải enforce:
- tenant/session isolation;
- assistant binding;
- conversation ownership;
- retention policy;
- PII policy.

Client không được đoán conversation ID để đọc conversation khác.

## 27. Observability

Trace mỗi turn:

~~~text
conversation_id
turn_id
state_version_before
pending_clarification_before
resolution_method
resolved_entities
standalone_query
rewrite_used
rewrite_validation
state_transition
state_version_after
latency
~~~

## 28. Failure Modes

### Conversation không tồn tại
Create mới nếu API cho phép, hoặc NOT_FOUND.

### Pending clarification hết hạn
Bỏ pending state và xử lý current message bình thường hoặc clarify mới.

### Summary generation fail
Recent turns + structured state vẫn phải hoạt động.

### Rewrite provider fail
Deterministic fallback hoặc dùng original + resolved semantics.

## 29. Tests bắt buộc

~~~text
single-turn standalone
pronoun resolution
ellipsis
explicit correction
ambiguity
pending clarification response
long conversation
state version conflict
rewrite preservation
~~~

## 30. Evaluation Metrics

~~~text
Reference Resolution Accuracy
Clarification Resolution Accuracy
Active Entity Accuracy
Correction Accuracy
Standalone Query Accuracy
State Transition Accuracy
Conversation Consistency
Rewrite Preservation Error Rate
~~~

## 31. Dependencies

Phụ thuộc:
- Conversation Store interface;
- optional Summary Provider;
- optional Rewrite Provider;
- Entity Resolver contract;
- Security/AllowedScope reference.

Không phụ thuộc VectorStore/Retriever/Reranker.

## 32. Extension Points

Future:
- cross-session memory;
- long-term user preferences;
- episodic memory;
- multimodal conversation state.

V1 chỉ conversation-scoped memory.

## 33. Design Decisions v1

1. ConversationContext là typed contract.
2. Interaction State khác Knowledge Base.
3. Explicit correction có precedence cao.
4. Pending clarification là state machine.
5. ResolvedTurn là first-class artifact.
6. Standalone rewrite chỉ khi cần.
7. Rewrite phải validate preservation.
8. Recent turns + structured state + summary cho conversation dài.
9. State có provenance.
10. State update hỗ trợ version/concurrency.
11. V1 không làm long-term cross-session memory.

## 34. Open Questions

- Conversation Store relational, JSON hay hybrid?
- Summary refresh theo token hay turn count?
- Entity state expiry policy?
- Rewrite provider/model nào?
- Có cần deterministic Vietnamese ellipsis rules trước LLM không?
- Conflict retry policy khi state_version mismatch?

## 35. Review Checklist

- [ ] Trách nhiệm component đã rõ?
- [ ] State có bị dùng như business knowledge không?
- [ ] Pending clarification xử lý trước query mới chưa?
- [ ] Explicit correction có ưu tiên cao chưa?
- [ ] Rewrite có preservation validation chưa?
- [ ] Có strategy long conversation chưa?
- [ ] Có concurrency/versioning chưa?
- [ ] Có golden multi-turn tests chưa?

## 36. Next Step

Conversation Engine REVIEWED thì đi tiếp Routing / QueryPlan.