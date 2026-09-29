# Raglyra — System Architecture

> **Status:** DRAFT  
> **Mục tiêu:** Xác định kiến trúc tổng thể của Raglyra trước khi đi sâu từng component.

## 1. Raglyra là gì?

Raglyra là nền tảng **RAG — Retrieval-Augmented Generation**.

**Retrieval-Augmented Generation** = mô hình trả lời bằng cách:
1. tìm dữ liệu liên quan từ knowledge source;
2. lấy dữ liệu đó làm bằng chứng;
3. mới dùng LLM để tạo câu trả lời.

Raglyra không được thiết kế theo kiểu:

~~~text
User Query
→ gửi thẳng vào LLM
→ hy vọng LLM trả đúng
~~~

mà theo:

~~~text
User Query
→ hiểu ngữ cảnh
→ quyết định cần nguồn nào
→ tìm bằng chứng
→ kiểm tra bằng chứng
→ tạo câu trả lời grounded
→ citation
~~~

**Grounded** = câu trả lời có căn cứ từ dữ liệu đã tìm được.

**Citation** = trích dẫn nguồn của thông tin dùng để trả lời.

## 2. Hai luồng lớn

Raglyra có hai pipeline độc lập nhưng liên kết với nhau.

### 2.1. Knowledge Pipeline

Dùng để đưa dữ liệu vào hệ thống:

~~~text
Source
→ Parse / OCR
→ Chuẩn hóa
→ Chunk / Structured Record
→ Index
→ Available for Retrieval
~~~

### 2.2. Query Pipeline

Dùng khi user hỏi:

~~~text
User Message
→ Conversation Context
→ Query Understanding
→ Routing / Planning
→ Retrieval hoặc Structured Source
→ Evidence
→ Answer
→ Citation
~~~

## 3. Kiến trúc tổng thể

~~~mermaid
flowchart LR
    U[User / Client] --> API[RAG API]

    API --> C[Conversation Engine]
    C --> Q[Query Understanding]
    Q --> P[Query Planner]

    P --> X[Plan Executor]

    X --> S[Structured Sources]
    X --> R[Retrieval Engine]

    R --> E[Evidence Layer]
    S --> E

    E --> A[Answer Engine]
    A --> O[Answer + Citation]

    K[Knowledge Sources] --> I[Ingestion Engine]
    I --> M[(Metadata DB)]
    I --> V[Vector Store]
    I --> L[Lexical Index]
    I --> B[Object Storage]

    R --> V
    R --> L
    R --> M
~~~

## 4. Các khối chính

### Conversation Engine

**Conversation Engine** = bộ xử lý ngữ cảnh hội thoại.

Nó phải hiểu các câu kiểu:

~~~text
"còn giá?"
"gói kia thì sao?"
"nó có hoàn tiền không?"
~~~

dựa trên những gì user vừa nói trước đó.

### Query Understanding

**Query Understanding** = hiểu user muốn làm gì.

Ví dụ xác định:

~~~text
Task: STRUCTURED_LOOKUP
Topic: PRICING
Entity: VPS Basic
Freshness: CURRENT
~~~

### Query Planner

**Planner** = bộ lập kế hoạch xử lý.

Nó tạo **QueryPlan** — kế hoạch máy có thể thực thi, ví dụ:

~~~text
STRUCTURED_SOURCE
→ Pricing API
~~~

hoặc:

~~~text
RAG
→ Policy Dataset
→ Hybrid Retrieval
~~~

### Retrieval Engine

**Retrieval** = truy xuất dữ liệu liên quan.

Có thể gồm:

- Exact Search — tìm chính xác mã/tên.
- Lexical Search — tìm theo từ khóa/toàn văn.
- Dense Search — tìm theo ý nghĩa bằng vector.
- Hybrid Search — kết hợp nhiều kiểu tìm kiếm.

### Evidence Layer

**Evidence** = bằng chứng đã được chọn để làm căn cứ cho câu trả lời.

Không phải mọi kết quả search đều là Evidence.

### Answer Engine

Chọn cách trả lời:

- deterministic — code format trực tiếp;
- extractive — lấy gần trực tiếp từ evidence;
- grounded synthesis — LLM tổng hợp nhưng không được vượt evidence.

## 5. Infrastructure abstraction

Core không phụ thuộc vendor cụ thể.

~~~text
VectorStore
├── PgVectorStore
└── QdrantVectorStore

EmbeddingProvider
LLMProvider
RerankerProvider
OCRProvider
ObjectStorage
LexicalStore
~~~

**Abstraction** = lớp giao diện chung giúp thay implementation mà không đổi business flow.

Ví dụ có thể đổi:

~~~text
VECTOR_STORE_PROVIDER=pgvector
~~~

thành:

~~~text
VECTOR_STORE_PROVIDER=qdrant
~~~

mà không viết lại Query Pipeline.

## 6. Nguyên tắc kiến trúc

1. Conversation là first-class — hội thoại không được xử lý như từng câu độc lập.
2. Retrieval không tự quyết định intent.
3. Router không phụ thuộc vector database.
4. LLM không tự chọn nguồn dữ liệu runtime.
5. Evidence phải có provenance.
6. No-match không đồng nghĩa dữ liệu không tồn tại.
7. Structured/current data ưu tiên source chính xác hơn vector snapshot.
8. Multi-tenant isolation phải xuyên toàn pipeline.
9. Evaluation là một phần của architecture, không phải làm sau cùng.
10. Source code cũ không quyết định kiến trúc Raglyra.
