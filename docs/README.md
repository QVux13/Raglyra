# Raglyra — Bản đồ tài liệu

> **Branch thiết kế:** `planning`
>
> Mục tiêu: nhìn tên folder và version là biết **phần nào của RAG đang được mô tả** và **bản nào là LATEST**.

## 1. Cây tài liệu chính

~~~text
docs/
├── 00-tong-quan/
│   ├── kien-truc-tong-the-v1.md
│   ├── kien-truc-tong-the-v2.md              ← LATEST
│   ├── review-kien-truc-v1.md
│   ├── lo-trinh-thiet-ke.md
│   └── quy-tac-viet-tai-lieu.md
│
├── 01-luong-hoi-dap/
│   ├── luong-xu-ly-cau-hoi-v1.md
│   ├── luong-xu-ly-cau-hoi-v2.md             ← LATEST
│   ├── intent-va-pham-vi-cau-hoi-v1.md
│   ├── intent-va-pham-vi-cau-hoi-v2.md       ← LATEST
│   ├── conversation-engine-v1.md              ← component detail
│   └── routing-query-plan-v1.md               ← component detail
│
├── 02-luong-nap-tri-thuc/
│   ├── luong-nap-va-danh-chi-muc-v1.md
│   └── luong-nap-va-danh-chi-muc-v2.md        ← LATEST
│
├── 03-nen-tang-chung/
│   ├── pham-vi-du-lieu-va-bao-mat-v1.md
│   ├── pham-vi-du-lieu-va-bao-mat-v2.md      ← LATEST
│   └── README.md
│
└── 04-trien-khai/
    └── README.md
~~~

## 2. Ý nghĩa từng folder

### 00-tong-quan — Overall / Tổng quan

Không phải chức năng runtime.

Dùng để hiểu:
- Raglyra là gì;
- toàn hệ thống có những khối nào;
- Query Pipeline và Knowledge Pipeline nối nhau ra sao;
- bản thiết kế nào đang là latest;
- thứ tự đọc/tổ chức docs.

### 01-luong-hoi-dap — Query Pipeline / Luồng xử lý câu hỏi

Đại diện cho toàn bộ runtime khi user chat:

~~~text
User Message
→ Conversation
→ Query Understanding
→ Entity Resolution
→ Capability / Scope Validation
→ QueryPlan
→ Retrieval / Structured Source
→ Evidence
→ Answer
→ Citation
~~~

### 02-luong-nap-tri-thuc — Knowledge Pipeline / Luồng nạp tri thức

Đại diện cho cách đưa dữ liệu vào RAG:

~~~text
Source
→ Sync / Version
→ Parse / OCR
→ Canonical Artifact
→ Region / Chunk
→ KnowledgeUnit / StructuredRecord
→ Embedding / Lexical Index
→ Active Generation
~~~

### 03-nen-tang-chung — Platform Foundation / Nền tảng dùng chung

Bao quanh cả hai pipeline:
- multi-tenant;
- security;
- provider abstraction;
- evaluation;
- observability;
- configuration.

### 04-trien-khai — Implementation Design / Thiết kế để code

Chỉ đi xuống sau khi component design đủ rõ:
- Data Model / ERD;
- API;
- UI;
- Infrastructure;
- Testing;
- Migration;
- Implementation Tasks.

## 3. LATEST hiện tại

~~~text
System Architecture       → kien-truc-tong-the-v2.md
Query Flow                → luong-xu-ly-cau-hoi-v2.md
Intent / Scope            → intent-va-pham-vi-cau-hoi-v2.md
Ingestion / Indexing      → luong-nap-va-danh-chi-muc-v2.md
Data Scope / Security     → pham-vi-du-lieu-va-bao-mat-v2.md
~~~

Implementation và review mới phải dùng LATEST, trừ khi đang so sánh lịch sử.

## 4. Quy tắc version

Version mới phải **self-contained**.

Ví dụ `v2` phải chứa:
- toàn bộ nội dung v1 còn hiệu lực;
- nội dung sửa đổi đặt đúng vị trí;
- không yêu cầu người đọc mở v1 để ghép logic.

Version cũ được freeze để so sánh lịch sử.

## 5. Thứ tự nên đọc

~~~text
00-tong-quan/kien-truc-tong-the-v2.md
        ↓
01-luong-hoi-dap/luong-xu-ly-cau-hoi-v2.md
        ↓
01-luong-hoi-dap/intent-va-pham-vi-cau-hoi-v2.md
        ↓
02-luong-nap-tri-thuc/luong-nap-va-danh-chi-muc-v2.md
        ↓
03-nen-tang-chung/pham-vi-du-lieu-va-bao-mat-v2.md
        ↓
01-luong-hoi-dap/conversation-engine-v1.md
        ↓
01-luong-hoi-dap/routing-query-plan-v1.md
~~~

## 6. Trạng thái tài liệu

- **DRAFT** = đang thiết kế.
- **REVIEWED** = đã self-review đủ để làm dependency.
- **APPROVED** = đã được chốt để implementation.

## 7. Nguyên tắc kiến trúc

Kien trúc target quyết định code.

Legacy/source code cũ chỉ dùng để:
- tái sử dụng implementation tốt;
- tham khảo behavior;
- rút ngắn thời gian code.

Không để code cũ ép kiến trúc mới đi theo lỗi cũ.
