# Raglyra — Documentation Map

> **Branch:** planning  
> **Nguyên tắc:** Mỗi chủ đề chỉ có **một file thiết kế chính**. Khi review, cập nhật chính file đó và dùng lịch sử Git để xem phiên bản cũ. Không tạo chuỗi file v0.1/v0.2/self-review gây khó kiểm soát.

## 1. Trạng thái tài liệu

Mỗi file dùng một trong ba trạng thái:

- **DRAFT** — đang thiết kế, chưa chốt.
- **REVIEWED** — đã tự review kỹ, có thể dùng làm dependency cho tài liệu sau.
- **APPROVED** — đã chốt để dùng làm căn cứ implementation.

## 2. Thứ tự tài liệu

~~~text
00-overview
Kiến trúc chung + roadmap
        ↓
01-flows
Luồng RAG end-to-end
        ↓
02-components
Thiết kế chi tiết từng khối
        ↓
03-data-model
DB / ERD / versioning
        ↓
04-api
API contract + sequence từng API
        ↓
05-ui
Màn hình / flow UI
        ↓
06-infrastructure
Provider / Vector DB / deployment / config
        ↓
07-testing-evaluation
Test, golden set, benchmark
        ↓
08-migration
Kế hoạch chuyển từ hệ thống cũ nếu cần
        ↓
09-tasks
Task MD cụ thể để AI/dev code
~~~

## 3. Nguyên tắc thiết kế

- Thiết kế theo chuẩn RAG trước, không bám source code cũ.
- Source code cũ chỉ được dùng để tái sử dụng implementation phù hợp.
- Không thiết kế DB/API trước khi contract giữa các component rõ.
- Không viết task implementation trước khi Architecture + Component + DB + API đủ rõ.
- Thuật ngữ technical phải được giải thích bằng tiếng Việt ngay trong tài liệu khi xuất hiện lần đầu.

## 4. Cấu trúc

~~~text
docs/
├── 00-overview/
├── 01-flows/
├── 02-components/
├── 03-data-model/
├── 04-api/
├── 05-ui/
├── 06-infrastructure/
├── 07-testing-evaluation/
├── 08-migration/
└── 09-tasks/
~~~
