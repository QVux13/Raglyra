# Raglyra — Document Writing Standard

> **Status:** APPROVED
> **Mục tiêu:** Quy định cách viết mọi tài liệu thiết kế trong Raglyra để người đọc có thể học, review và kiểm soát hệ thống mà không cần đọc source code.

## 1. Nguyên tắc

Mỗi file thiết kế phải tự trả lời được:

1. File này mô tả phần nào của hệ thống?
2. Phần đó nằm ở đâu trong luồng tổng thể?
3. Nó nhận input gì?
4. Nó xử lý gì?
5. Nó trả output gì?
6. Nó gọi component nào khác?
7. Vì sao component này tồn tại?
8. Nếu bỏ component này thì lỗi gì xảy ra?
9. Có những case thực tế nào?
10. Có những quyết định nào đã chốt?
11. Có những gì chưa chốt?
12. Bước tiếp theo là gì?

## 2. Technical Terms

Mỗi thuật ngữ technical khi xuất hiện lần đầu phải được giải thích ngay bằng tiếng Việt.

Ví dụ:

**Reranker** = mô hình xếp hạng lại một tập candidate nhỏ để tăng độ chính xác trước khi chọn evidence.

Không được viết một chuỗi thuật ngữ mà không giải thích.

## 3. Cấu trúc bắt buộc cho file kiến trúc/flow

~~~text
1. Mục tiêu
2. Phạm vi
3. Vị trí trong hệ thống
4. Sơ đồ tổng thể
5. Input
6. Output
7. Giải thích từng bước/component
8. Technical terms
9. Case thực tế
10. Error/edge cases
11. Trách nhiệm component
12. Những gì không thuộc file này
13. Design Decisions
14. Open Questions
15. Review Checklist
16. Next Step
~~~

## 4. Cấu trúc cho Component Design

Mỗi component cần:

~~~text
Purpose
Responsibilities
Non-responsibilities
Inputs
Outputs
Domain Objects
Sequence
Decision Rules
Errors
Configuration
Observability
Security Considerations
Tests
Dependencies
Extension Points
~~~

## 5. Cấu trúc cho API Design

Mỗi API cần:

~~~text
Use case
Method / Path
Authorization
Request
Response
Validation
Sequence
Data read/write
Errors
Idempotency
Pagination
Observability
Tests
~~~

## 6. Cấu trúc cho Task Implementation

Task chỉ được tạo khi architecture/component/API liên quan đã APPROVED.

Task phải có:

~~~text
Objective
Why
Dependencies
Exact scope
Files/modules
Input/output contract
Allowed changes
Forbidden changes
Tests
Acceptance criteria
Migration impact
Rollback
~~~

## 7. Không được làm

Không được viết tài liệu kiểu:

~~~text
Router → Retriever → LLM
~~~

rồi kết thúc.

Phải giải thích Router là gì, Retriever nhận gì, tại sao cần bước đó, output nào được truyền tiếp và case nào minh họạ.

Không dùng source code cũ làm lý do để thay đổi architecture target.

Không tách một chủ đề thành quá nhiều file version gây mất kiểm soát; dùng Git history cho lịch sử.