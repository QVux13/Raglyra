# Raglyra — Query & Answer Flow

> **Status:** DRAFT  
> **Mục tiêu:** Mô tả đầy đủ luồng khi user gửi một câu hỏi.

## 1. Luồng chính

~~~text
User Message
→ Security Context
→ Conversation Context
→ Reference Resolution
→ Standalone Query
→ Query Understanding
→ Query Plan
→ Execution
→ Evidence
→ Evidence Gate
→ Answer
→ Citation
→ Conversation State Update
~~~

## 2. Conversation Context

**Conversation Context** = ngữ cảnh cuộc trò chuyện hiện tại.

Ví dụ:

~~~text
User: Tôi muốn xem VPS Basic.
Bot: ...
User: Còn giá?
~~~

Hệ thống cần nhớ:

~~~text
active entity = VPS Basic
~~~

để hiểu câu "Còn giá?".

## 3. Reference Resolution

**Reference Resolution** = giải nghĩa các từ/câu đang tham chiếu về lượt chat trước.

Ví dụ:

~~~text
"Còn giá?"
→ "Giá của VPS Basic?"
~~~

~~~text
"Nó có hoàn tiền không?"
→ "VPS Basic có hoàn tiền không?"
~~~

Nếu không thể xác định chắc chắn thì phải **CLARIFY** — hỏi lại user, không đoán.

## 4. Standalone Query

**Standalone Query** = câu hỏi đầy đủ nghĩa, có thể mang đi xử lý mà không cần đọc lại toàn history.

Ví dụ:

~~~text
Original:
"Còn giá?"

Standalone:
"Giá hiện tại của VPS Basic là bao nhiêu?"
~~~

## 5. Query Understanding

Hệ thống phân tích query thành dữ liệu có cấu trúc:

~~~text
task
topic
entity
freshness
constraints
ambiguity
multi-intent
~~~

Ví dụ:

~~~text
Task = STRUCTURED_LOOKUP
Topic = PRICING
Entity = VPS Basic
Freshness = CURRENT
~~~

## 6. Query Plan

**QueryPlan** = kế hoạch thực thi.

Các loại chính:

~~~text
DIRECT
CLARIFY
STRUCTURED_SOURCE
EXACT_SEARCH
RAG
MULTI_PLAN
NO_ANSWER
~~~

### DIRECT

Không cần search, ví dụ greeting.

### CLARIFY

Hỏi lại vì thiếu hoặc mơ hồ.

### STRUCTURED_SOURCE

Gọi nguồn dữ liệu có cấu trúc như API/SQL.

### EXACT_SEARCH

Tìm chính xác mã/SKU/document code.

### RAG

Tìm knowledge rồi dùng evidence tạo answer.

### MULTI_PLAN

Một query chứa nhiều việc.

Ví dụ:

~~~text
"VPS Basic giá bao nhiêu và có hoàn tiền không?"
~~~

có thể chạy:

~~~text
Plan A → Pricing API
Plan B → Policy RAG
~~~

## 7. Retrieval

Nếu plan là RAG:

~~~text
Exact / Lexical / Dense
→ Candidate
→ Fusion
→ Reranker
→ Evidence
~~~

**Lexical Search** = tìm theo từ khóa/toàn văn.

**Dense Search** = tìm theo ý nghĩa bằng embedding vector.

**Fusion** = gộp nhiều bảng xếp hạng.

**Reranker** = mô hình xếp hạng lại các kết quả để chọn kết quả liên quan hơn.

## 8. Evidence Gate

**Evidence Gate** = bước kiểm tra bằng chứng có đủ để trả lời hay chưa.

Có thể trả:

~~~text
ANSWERABLE
PARTIAL
NO_ANSWER
NEEDS_CLARIFICATION
~~~

LLM không được tự bổ sung phần thiếu.

## 9. Answer Strategy

### Deterministic

Code tạo answer trực tiếp từ typed data.

### Extractive

Lấy gần trực tiếp nội dung evidence.

### Grounded Synthesis

LLM tổng hợp nhiều evidence nhưng phải giữ đúng facts.

## 10. Citation

Citation được dựng từ provenance của Evidence.

**Provenance** = nguồn gốc dữ liệu, ví dụ file/version/page/section/row/URL.

LLM không tự bịa citation.

## 11. Conversation State Update

Sau lượt chat, hệ thống cập nhật các trạng thái như:

~~~text
active entity
current topic/task
pending clarification
user constraints
~~~

Không biến unsupported LLM output thành fact trong conversation state.
