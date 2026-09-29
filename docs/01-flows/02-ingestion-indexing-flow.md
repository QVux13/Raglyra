# Raglyra — Ingestion & Indexing Flow

> **Status:** DRAFT  
> **Mục tiêu:** Mô tả dữ liệu được biến thành knowledge có thể retrieval như thế nào.

## 1. Luồng chính

~~~text
Source
→ Source Item
→ Source Version
→ Parse / OCR
→ Canonical Artifact
→ Region Segmentation
→ Chunk / Structured Record
→ Quality Gate
→ Embedding / Lexical Index
→ Index Generation
→ Activate
~~~

## 2. Source

**Source** = nguồn dữ liệu logic.

Ví dụ:
- uploaded files;
- website;
- Google Drive;
- API;
- database snapshot.

## 3. Source Item

**Source Item** = một item cụ thể bên trong Source.

Ví dụ website là Source, từng URL/page là Source Item.

## 4. Source Version

**Source Version** = snapshot bất biến của một Source Item tại một thời điểm.

**Immutable** = không sửa đè; nội dung đổi thì tạo version mới.

## 5. Parser / OCR

**Parser** = bộ phân tích file và cấu trúc.

**OCR — Optical Character Recognition** = nhận dạng chữ từ ảnh/PDF scan.

Không OCR mọi file/image nếu không cần.

## 6. Canonical Artifact

**Canonical Artifact** = format chuẩn nội bộ sau khi parser xử lý.

Nó có thể giữ:
- heading;
- paragraph;
- list;
- table;
- image;
- caption;
- page/sheet/slide;
- hierarchy;
- source location.

## 7. Region Segmentation

**Region Segmentation** = chia tài liệu theo vùng nội dung.

Ví dụ một PDF có:
- narrative text;
- pricing table;
- FAQ;
- chart.

Mỗi region có thể dùng strategy khác.

## 8. Chunking

**Chunking** = chia knowledge thành các đơn vị nhỏ để retrieval.

Nguyên tắc:

~~~text
semantic/structure boundary
trước
token size
~~~

Không áp một chunk size cố định cho mọi source.

## 9. Knowledge Unit

**KnowledgeUnit** = đơn vị knowledge dùng để search.

Có thể nhỏ hơn **ContextUnit** — vùng lớn hơn dùng khi cần mở rộng context.

## 10. Structured Record

Dữ liệu bảng/record có type rõ có thể lưu thành **StructuredRecord** thay vì ép thành text chunk.

Ví dụ:
- pricing row;
- product specification;
- FAQ record.

## 11. Embedding

**Embedding** = biến text thành vector số biểu diễn ý nghĩa.

Embedding chỉ áp trên representation phù hợp để semantic search.

## 12. Vector Store

Vector backend đi sau interface:

~~~text
VectorStore
├── PgVectorStore
└── QdrantVectorStore
~~~

Core ingestion không phụ thuộc backend cụ thể.

## 13. Lexical Index

**Lexical Index** = chỉ mục tìm theo từ khóa/toàn văn, ví dụ PostgreSQL FTS hoặc BM25 engine.

Raglyra có thể dùng cả lexical + vector retrieval.

## 14. Index Generation

**Index Generation** = một bộ index hoàn chỉnh ứng với version/profile nhất định.

Flow an toàn:

~~~text
build generation mới
→ validate
→ activate atomically
→ generation cũ giữ để rollback
~~~

**Atomic activation** = chuyển trạng thái active một cách nhất quán, tránh vector mới nhưng metadata cũ.

## 15. Quality Gate

Phải kiểm:
- content coverage;
- empty/broken chunks;
- token overflow;
- lost table header;
- duplicate ratio;
- OCR quality;
- provenance completeness;
- index count consistency.

Không silently drop dữ liệu.
