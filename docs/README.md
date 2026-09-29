# Raglyra — Bản đồ tài liệu

> **Branch thiết kế:** `planning`
>
> Mục tiêu của cây `docs/`: nhìn tên folder và tên file là biết **nó đại diện cho chức năng nào của hệ thống RAG**.

## 1. Cây tài liệu hiện tại

~~~text
docs/
├── 00-tong-quan/
│   ├── kien-truc-tong-the.md
│   ├── lo-trinh-thiet-ke.md
│   └── quy-tac-viet-tai-lieu.md
│
├── 01-luong-hoi-dap/
│   ├── luong-xu-ly-cau-hoi.md
│   └── intent-va-pham-vi-cau-hoi.md
│
├── 02-luong-nap-tri-thuc/
│   └── luong-nap-va-danh-chi-muc.md
│
├── 03-nen-tang-chung/
│   ├── README.md
│   └── pham-vi-du-lieu-va-bao-mat.md
│
└── 04-trien-khai/
    └── README.md
~~~

## 2. 00-tong-quan — Overall / Tổng quan

Không phải chức năng runtime.

Đây là **bản đồ toàn dự án**.

- `kien-truc-tong-the.md`: toàn bộ Raglyra gồm những khối nào và hai pipeline lớn chạy ra sao.
- `lo-trinh-thiet-ke.md`: thứ tự nên thiết kế từ kiến trúc tới code.
- `quy-tac-viet-tai-lieu.md`: quy chuẩn bắt buộc để mọi doc đủ chi tiết và dễ học.

Đọc folder này trước.

## 3. 01-luong-hoi-dap — Query Pipeline / Luồng xử lý câu hỏi

Đại diện cho **chức năng khi user chat với Raglyra**.

~~~text
User Message
→ Conversation
→ Intent / Scope
→ Query Understanding
→ QueryPlan
→ Retrieval / Structured Source
→ Evidence
→ Answer
→ Citation
~~~

### luong-xu-ly-cau-hoi.md

File end-to-end.

Giải thích toàn bộ từ lúc user gửi message tới lúc nhận answer.

### intent-va-pham-vi-cau-hoi.md

File chuyên sâu về:
- Intent = user muốn hệ thống làm loại việc gì.
- Topic = user đang hỏi chủ đề nào.
- Capability = hệ thống cần khả năng nào để xử lý.
- Source = nguồn dữ liệu cụ thể.
- Scope = phạm vi dữ liệu/chức năng được phép.

File này đặc biệt quan trọng để tránh query mơ hồ hoặc prompt injection làm hệ thống search sang dữ liệu khác ngoài phạm vi.

## 4. 02-luong-nap-tri-thuc — Knowledge Pipeline / Luồng nạp tri thức

Đại diện cho **chức năng đưa dữ liệu vào RAG**.

~~~text
Source
→ Parse / OCR
→ Canonical Artifact
→ Region
→ Chunk / Structured Record
→ Embedding / Index
→ Active Knowledge
~~~

### luong-nap-va-danh-chi-muc.md

Giải thích toàn bộ:
- file/web/API snapshot vào hệ thống thế nào;
- parser/OCR;
- chunking;
- KnowledgeUnit;
- StructuredRecord;
- vector/lexical index;
- versioning;
- scope/ACL metadata;
- atomic activation.

## 5. 03-nen-tang-chung — Platform Foundation / Nền tảng dùng chung

Không phải một bước đơn lẻ trong flow.

Nó bao quanh cả Query Pipeline và Knowledge Pipeline.

### pham-vi-du-lieu-va-bao-mat.md

Giải thích:
- Multi-tenant = nhiều khách hàng dùng chung platform nhưng dữ liệu phải cách ly.
- Security Scope = request được phép xem gì.
- Assistant/Dataset/Source Scope.
- ACL.
- backend filtering.
- revocation.
- prompt injection boundary.
- cache scope.
- citation privacy.

Sau này folder này còn có provider abstraction, evaluation/observability và configuration.

## 6. 04-trien-khai — Implementation Design / Thiết kế để code

Chỉ dùng sau khi architecture + flow + component đủ rõ.

Sẽ chứa:
- Data Model / ERD.
- API.
- UI.
- Infrastructure.
- Testing.
- Migration.
- Task cho AI/dev.

## 7. Thứ tự nên đọc

~~~text
00-tong-quan/kien-truc-tong-the.md
        ↓
01-luong-hoi-dap/luong-xu-ly-cau-hoi.md
        ↓
01-luong-hoi-dap/intent-va-pham-vi-cau-hoi.md
        ↓
02-luong-nap-tri-thuc/luong-nap-va-danh-chi-muc.md
        ↓
03-nen-tang-chung/pham-vi-du-lieu-va-bao-mat.md
~~~

Sau khi năm file này REVIEWED mới đi sâu component.

## 8. Quy tắc version

Trong cây docs chỉ giữ **một file hiện tại cho mỗi chủ đề**.

Không tạo:

~~~text
file-v0.1.md
file-v0.2.md
file-self-review.md
~~~

Khi cập nhật:
- sửa file hiện tại;
- Git commit/history giữ bản cũ;
- cây docs luôn chỉ hiển thị bản mới nhất.

## 9. Trạng thái tài liệu

- **DRAFT** = đang thiết kế.
- **REVIEWED** = đã review kỹ, dùng được làm dependency.
- **APPROVED** = đã chốt để implementation.

## 10. Nguyên tắc kiến trúc

Kiến trúc target quyết định code.

Legacy/source code cũ chỉ dùng để:
- tái sử dụng implementation tốt;
- tham khảo behavior;
- rút ngắn thời gian code.

Không để code cũ ép kiến trúc mới đi theo lỗi cũ.
