# 03-nen-tang-chung — Platform Foundation

> **Status:** DRAFT
> **Dịch nghĩa:** Nền tảng dùng chung cho toàn bộ Raglyra.

Folder này chứa những chức năng **không thuộc riêng Query Pipeline hay Knowledge Pipeline**, mà bao quanh cả hai.

## 1. File hiện tại

~~~text
pham-vi-du-lieu-va-bao-mat-v2.md
~~~

File này chốt:
- tenant isolation;
- assistant/dataset/source scope;
- ACL;
- backend filtering;
- revocation;
- prompt injection boundary;
- cache/citation privacy.

## 2. Các file sẽ thiết kế sau

~~~text
provider-abstraction.md
evaluation-observability.md
configuration.md
~~~

### Provider Abstraction

Lớp giao diện chung để thay:
- LLM provider;
- Embedding provider;
- Reranker;
- OCR;
- VectorStore.

### Evaluation

Đo Raglyra có trả đúng không.

### Observability

Theo dõi request chạy qua bước nào, chậm/sai ở đâu.

### Configuration

Quản lý profile, feature flags, thresholds và environment-level config.

## 3. Khi nào đi sâu folder này?

Sau khi Query Pipeline và Knowledge Pipeline đã REVIEWED ở mức flow.
