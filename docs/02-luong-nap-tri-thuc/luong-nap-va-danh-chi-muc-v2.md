# Raglyra — Ingestion & Indexing Flow v2

> **Version:** v2 — self-contained, LATEST

> **Status:** REVIEWED
> **Mục tiêu:** Mô tả chi tiết cách một nguồn dữ liệu được biến thành knowledge có thể retrieval.
> **Vị trí trong hệ thống:** Đây là Knowledge Pipeline. File này chuẩn bị dữ liệu cho Query Pipeline.

---

## 1. Vì sao cần Ingestion?

LLM không nên đọc lại toàn bộ PDF/website mỗi lần user hỏi.

Ta cần xử lý trước:

~~~text
Dữ liệu thô
→ hiểu cấu trúc
→ chia thành đơn vị hợp lý
→ tạo index
→ retrieval nhanh và có provenance
~~~

**Ingestion** = quá trình đưa dữ liệu vào hệ thống knowledge theo một pipeline có kiểm soát.

## 2. Luồng tổng thể

~~~mermaid
flowchart TD
    A[Source] --> SYNC[Source Sync / Upload]
    SYNC --> B[Discover Source Items]
    B --> CH{Content changed?}
    CH -->|No| NOOP[No-op / Metadata Sync]
    CH -->|Yes| C[Create SourceVersion]
    C --> D[Store Raw Source]
    D --> E[Source Analysis]
    E --> F[Parser / OCR]
    F --> G[Canonical Artifact]
    G --> H[Region Segmentation]
    H --> I[Region Planner]
    I --> J[Knowledge Units]
    I --> K[Structured Records]
    J --> L[Quality Gate]
    K --> L
    L --> M[Embedding Pipeline]
    L --> N[Lexical Index]
    L --> O[Exact / Structured Index]
    M --> P[Vector Store]
    N --> Q[Index Generation]
    O --> Q
    P --> Q
    Q --> R[Generation Validation]
    R --> S[Atomic Activation]
~~~

## 3. Source

**Source** = nguồn knowledge logic.

Ví dụ:

~~~text
uploaded document set
website
Drive folder
database snapshot
API connector
manual dataset
~~~

Source không nhất thiết là một file.

## 4. Source Item

**Source Item** = item cụ thể bên trong Source.

Ví dụ:

~~~text
Website Source
├── /
├── /pricing
├── /refund
└── /product/vps-basic
~~~

Mỗi page là một Source Item.

## 5. Source Version

**SourceVersion** = snapshot bất biến của Source Item tại một thời điểm.

**Immutable** = version đã tạo thì không sửa nội dung; dữ liệu mới tạo version mới.

Mục đích:
- audit;
- reparse;
- rechunk;
- reembed;
- rollback;
- citation chính xác.

## 6. Source Synchronization va Idempotency

### Source Synchronization

**Source Synchronization** = cơ chế đồng bộ source có nhiều item hoặc thay đổi theo thời gian.

Ví dụ:
- website có nhiều page;
- Google Drive folder;
- database snapshot;
- API connector;
- sitemap.

Source Adapter cần có thể theo dõi:

~~~text
external_source_item_id
sync_cursor
etag / content_hash
modified_at
last_successful_sync
deleted_upstream
~~~

**Sync Cursor** = mốc giúp lần chạy sau biết đã đồng bộ tới đâu, tránh đọc lại toàn bộ source nếu không cần.

### Content change detection

Một Source Item chỉ cần tạo SourceVersion mới khi content hoặc metadata có ý nghĩa thay đổi.

Ví dụ:

~~~text
external_item_id = page-123
content_hash = abc
~~~

Lần sync sau vẫn cùng hash:

~~~text
→ không reparse
→ không rechunk
→ không reembed
~~~

### Idempotency

**Idempotency** = chạy lại cùng một operation nhiều lần nhưng không tạo kết quả duplicate ngoài ý muốn.

Job identity nên dựa trên logical source item + source version + processing profile.

Ví dụ retry worker sau timeout không được tạo hai SourceVersion hay hai IndexGeneration tương đương.

### Upstream delete

Nếu item biến mất ở source ngoài:

~~~text
mark source item superseded/revoked
→ stop retrieval visibility
→ cleanup derived data sau
~~~

Không silently giữ data cũ active mãi.

## 7. Source Adapter

**Source Adapter** = interface chuẩn để Raglyra làm việc với nhiều loại nguồn mà ingestion core không phụ thuộc vendor.

Logical responsibilities:

~~~text
discover(sync_cursor)
fetch(source_item)
get_metadata(source_item)
health()
~~~

Ví dụ adapter:
- FileUploadSourceAdapter;
- WebSourceAdapter;
- DriveSourceAdapter;
- DatabaseSnapshotAdapter.

Live structured source có thể có adapter riêng cho query-time lookup và không bắt buộc đi qua indexing pipeline.

## 8. Source Modes

Không phải source nào cũng ingestion giống nhau.

Baseline:

~~~text
INDEXED_KNOWLEDGE
SNAPSHOT_STRUCTURED
LIVE_STRUCTURED
HYBRID
~~~

### Indexed Knowledge
Ví dụ PDF policy, technical manual, website article.

Đi qua parser/chunk/index.

### Snapshot Structured
Ví dụ CSV/XLSX/DB snapshot.

Tạo StructuredRecord và có thể semantic projection.

### Live Structured
Ví dụ Pricing API realtime.

Không bắt buộc ingestion/vector hóa. QueryPlan có thể gọi trực tiếp.

### Hybrid
Có cả live source-of-truth và knowledge mô tả/indexed.

## 9. Store Raw Source

Raw source cần lưu để có thể chạy lại pipeline.

Có thể lưu trong Object Storage.

**Object Storage** = storage phù hợp file/blob như S3/MinIO/local dev.

## 10. Source Analysis

Trước parser có thể detect:

~~~text
file type
MIME
text layer
scan ratio
table density
language
page count
sheet/slide count
source metadata
~~~

Mục tiêu: chọn parser/OCR strategy phù hợp.

## 11. Parser

**Parser** = bộ phân tích content và cấu trúc của source.

Ví dụ PDF parser không chỉ lấy text mà còn cố giữ:

~~~text
heading
paragraph
list
table
page
reading order
~~~

Parser phải nằm sau abstraction để có thể thay implementation sau này.

## 12. OCR

**OCR — Optical Character Recognition** = nhận dạng chữ từ ảnh hoặc PDF scan.

Không OCR mọi page mặc định.

Chỉ OCR khi:
- không có text layer;
- text layer chất lượng thấp;
- image chứa text quan trọng.

OCR output nên giữ confidence khi provider hỗ trợ.

## 13. Canonical Artifact

**Canonical Artifact** = representation chuẩn nội bộ sau parse.

Mục tiêu:
> downstream không cần biết file ban đầu được parse bằng thư viện nào.

Có thể chứa:

~~~text
pages/sheets/slides
blocks
hierarchy
tables
visuals
captions
reading order
provenance
~~~

## 14. Block

**Block** = đơn vị cấu trúc do parser nhận diện.

Ví dụ:

~~~text
Heading
Paragraph
List
Table
Figure
Caption
Code block
FAQ pair
~~~

Block chưa nhất thiết là retrieval chunk.

## 15. Provenance

**Provenance** = thông tin nguồn gốc dữ liệu.

Một block/unit cần truy ngược được:

~~~text
Source
→ SourceVersion
→ Page/Sheet/Slide
→ Region
→ Block
→ KnowledgeUnit
~~~

Metadata provenance có thể gồm:

~~~text
page
section
bounding box
sheet row/column
DOM path
reading order
~~~

## 16. Region Segmentation

**Region Segmentation** = chia artifact thành các vùng semantic/structural khác nhau.

Ví dụ PDF:

~~~text
Region 1 = narrative
Region 2 = pricing table
Region 3 = FAQ
Region 4 = chart
~~~

Lý do:
> một document không nên chỉ có một chunking strategy.

## 17. Region Types

Baseline:

~~~text
NARRATIVE
HIERARCHICAL_TEXT
FAQ
TABLE
STRUCTURED_RECORDS
LIST
CODE
VISUAL
CAPTION
OTHER
~~~

## 18. Region Planner

**Region Planner** = chọn strategy xử lý cho từng region.

Input:
- region type;
- hierarchy quality;
- source profile;
- embedding profile;
- token statistics.

Output:

~~~text
RegionPlan
~~~

Ví dụ:

~~~text
FAQ
→ FAQ_ATOMIC

TABLE
→ STRUCTURED + ROW_SERIALIZATION

HIERARCHICAL_TEXT
→ PARENT_CHILD
~~~

## 19. Chunking

**Chunking** = chia content thành KnowledgeUnit.

Nguyên tắc:

~~~text
structure/semantic boundary
trước
token boundary
~~~

Không dùng universal fixed 500 token cho mọi loại dữ liệu.

## 20. Boundary precedence

Một strategy phổ biến:

~~~text
document hierarchy
→ section/subsection
→ paragraph
→ list group
→ sentence
→ token fallback
~~~

Token là giới hạn cuối, không phải điểm cắt đầu tiên.

## 21. KnowledgeUnit

**KnowledgeUnit** = đơn vị retrieval/search.

Ví dụ:

~~~text
Khoản 2 của điều 10
FAQ question + answer
một row pricing có context header
một paragraph group
~~~

## 22. ContextUnit

**ContextUnit** = đơn vị context lớn hơn có thể được mở rộng sau khi retrieval trúng KnowledgeUnit.

Ví dụ:

~~~text
Điều 10 = ContextUnit
Khoản 1/2/3 = KnowledgeUnit
~~~

Retriever tìm child.

Context Builder có thể lấy parent sau.

## 23. Parent-Child Retrieval

**Parent-Child Retrieval** = index child nhỏ để tăng precision, nhưng khi trúng thì expand parent để có đủ ngữ cảnh.

Chỉ dùng nếu hierarchy đáng tin.

## 24. Hierarchy Quality

Parser hierarchy có thể sai.

Có thể phân:

~~~text
HIGH
MEDIUM
LOW
~~~

Nếu LOW:

~~~text
fallback general structure-aware chunking
~~~

Không ép parent-child.

## 25. FAQ Atomic

Question + Answer nên đi cùng một KnowledgeUnit.

Nếu answer quá dài:

~~~text
Question + Answer Part 1
Question + Answer Part 2
~~~

Question/context được lặp để mỗi chunk vẫn tự hiểu.

## 26. Table Strategy

Không flatten cả bảng thành một text dài.

Giữ hai representation:

~~~text
StructuredTable
+
RetrievalSerialization
~~~

### StructuredTable

Giữ:

~~~text
columns
rows
types
units
coordinates
merged cells
~~~

### RetrievalSerialization

Biến row/entity group thành text dễ search, ví dụ:

~~~text
Product: VPS Basic
CPU: 2 vCPU
RAM: 4 GB
Price: 199000 VND
~~~

## 27. StructuredRecord

**StructuredRecord** = record typed dùng cho lookup/exact retrieval.

Ví dụ:

~~~text
product_id
price
currency
stock
effective_at
~~~

Nếu dữ liệu này là current/live thì source-of-truth có thể là API/DB trực tiếp, không phải vector.

## 28. Visual Region

Image có thể là:

~~~text
decorative
text-heavy
chart
diagram
screenshot
meaningful image
~~~

Strategy:

~~~text
decorative → ignore/metadata
text-heavy → OCR
chart/diagram → caption + optional vision extraction
~~~

Không OCR mọi image.

## 29. Text Representations

Một KnowledgeUnit có thể có nhiều representation:

~~~text
raw_text
display_text
embedding_text
rerank_text
structured_value
~~~

### raw_text
Gần source gốc.

### display_text
Dùng citation/UI.

### embedding_text
Text tối ưu semantic embedding, có thể thêm heading/context.

### rerank_text
Text tối ưu cho reranker.

## 30. Embedding Text Contextualization

Có thể thêm:

~~~text
heading breadcrumb
table header
entity identity
local section context
~~~

nhưng phải tránh **semantic dilution**.

**Semantic Dilution** = thêm quá nhiều context không liên quan làm vector mất trọng tâm.

## 31. Token Budget

Token validation phải chạy trên final embedding_text, không chỉ raw_text.

Profile có thể có:

~~~text
target_tokens
hard_max_tokens
context_prefix_budget
minimum_useful_tokens
~~~

Không có universal chunk size.

## 32. Overlap

**Overlap** = lặp một phần text giữa hai chunk kế nhau.

Không nên mặc định cao.

Parent-child thường overlap thấp hoặc 0.

Flat narrative có thể dùng small overlap nếu benchmark chứng minh có lợi.

## 33. Metadata

Mỗi unit/record cần metadata như:

~~~text
tenant
workspace
dataset
source
source version
region
page/sheet/slide
section path
language
ACL
sensitivity
effective date
provenance
~~~

## 34. Metadata phạm vi truy cập

Ngay từ Ingestion, mỗi artifact có khả năng được retrieval phải mang metadata phạm vi.

Tối thiểu:

~~~text
tenant_id
workspace_id
dataset_id
source_id
source_version_id
visibility
ACL / sensitivity
lifecycle_status
~~~

**ACL — Access Control List** = quy tắc xác định ai hoặc nhóm nào được phép truy cập dữ liệu.

Mục tiêu:

> Retrieval có thể filter đúng ngay tại VectorStore/LexicalStore/Database, thay vì search rộng rồi mới lọc bằng Python.

Ví dụ một KnowledgeUnit của Tenant A phải được index cùng:

~~~text
tenant_id = A
dataset_id = policy
source_id = policy-001
visibility = ACTIVE
~~~

Khi Tenant B query, backend filter phải loại unit này trước ranking.

### Scope inheritance

Phạm vi phải được kế thừa theo chuỗi:

~~~text
Source
→ SourceVersion
→ Region
→ KnowledgeUnit / StructuredRecord
→ Vector/Lexical index point
~~~

Nếu source bị revoke:

~~~text
source lifecycle = REVOKED
→ unit/index visibility bị chặn
→ physical cleanup có thể chạy sau
~~~

Không để vector point trở thành dữ liệu "mồ côi" không còn biết thuộc source/tenant nào.

Chi tiết security contract nằm tại:

~~~text
docs/03-nen-tang-chung/pham-vi-du-lieu-va-bao-mat-v2.md
~~~

## 35. AI Enrichment

Optional:

~~~text
summary
keywords
topics
generated questions
entity hints
~~~

Nếu model tạo, phải đánh dấu:

~~~text
MODEL_DERIVED
~~~

Không nhầm với source fact.

## 36. Quality Gate

Kiểm unit/artifact trước index.

Severity có thể:

~~~text
BLOCKING
WARNING
INFO
~~~

Blocking examples:
- lost provenance;
- token overflow;
- broken table schema;
- index count mismatch.

## 37. Coverage Validation

Phải biết content nào:

~~~text
handled
ignored intentionally
lost unexpectedly
~~~

Metrics:

~~~text
text coverage
table coverage
visual coverage
empty units
duplicate ratio
~~~

Không silently drop content.

## 38. Embedding Pipeline

~~~text
KnowledgeUnit
→ contextualize embedding_text
→ token validation
→ batch
→ EmbeddingProvider
→ vector
→ VectorStore upsert
~~~

**EmbeddingProvider** = abstraction cho model embedding.

## 39. VectorStore

Interface logic:

~~~text
create_generation
upsert
search
delete_generation
activate_generation
health
~~~

Adapters:

~~~text
PgVectorStore
QdrantVectorStore
~~~

Ingestion core không biết backend implementation.

## 40. Lexical Index

Lexical Index dùng cho keyword/full-text retrieval.

Có thể implementation:

~~~text
PostgreSQL FTS
BM25 engine
sparse search
~~~

Vector và lexical là hai index logical khác nhau.

## 41. Embedding Profile Compatibility

**Embedding Profile** = cấu hình xác định embedding provider/model/dimension/tokenizer/serializer version dùng cho một generation.

Ví dụ:

~~~text
profile_id = embed-v1
provider = managed-provider
model = embedding-model-x
dimension = 1536
serializer_version = 2
~~~

Dense search chỉ được so sánh trực tiếp vector trong cùng embedding space/profile tương thích.

Nếu một QueryPlan cần search nhiều dataset dùng embedding profile khác nhau:

~~~text
group theo embedding profile
→ embed query theo từng profile
→ search riêng
→ fuse bằng rank
~~~

Không so raw cosine score giữa hai embedding model khác nhau.

Đây là lý do IndexGeneration phải bind chính xác EmbeddingProfile.

## 42. Version Layers

Không dùng một document_version cho mọi thứ.

Target logical layers:

~~~text
SourceVersion
ParsedArtifactVersion
KnowledgeUnitSetVersion
StructuredRecordSetVersion
EmbeddingGeneration
LexicalGeneration
IndexGeneration
ActiveGeneration
~~~

## 43. Vì sao cần nhiều version layer?

Đổi parser:

~~~text
rebuild ParsedArtifact trở xuống
~~~

Đổi chunking:

~~~text
giữ SourceVersion
→ rebuild KnowledgeUnitSet trở xuống
~~~

Đổi embedding model:

~~~text
giữ chunk
→ rebuild embedding/vector generation
~~~

Đổi pgvector sang Qdrant:

~~~text
không cần parse/chunk lại nếu unit compatible
→ build vector generation ở backend mới
~~~

## 44. Index Generation

**IndexGeneration** = bộ index hoàn chỉnh ứng với một tập source/profile/version.

Không build đè active index.

Luồng:

~~~text
build new generation
→ validate
→ activate
→ old generation remains rollback candidate
~~~

## 45. Active Generation Manifest

**Active Generation Manifest** = pointer logic tới toàn bộ generation đang active và tương thích với nhau.

Có thể giữ:

~~~text
source / source-version set
knowledge-unit-set version
structured-record-set version
embedding profile
vector generation
lexical generation
metadata generation
validation status
capabilities exposed
~~~

Query Pipeline chỉ dùng generation được manifest đánh dấu ACTIVE + VALIDATED.

Mục tiêu tránh tình trạng:

~~~text
vector v2
lexical v1
metadata v3
~~~

bị trộn ngoài chủ ý.

## 46. Atomic Activation

**Atomic Activation** = chuyển bộ generation mới thành active một cách nhất quán.

Mục tiêu tránh:

~~~text
vector v2
+
metadata v1
~~~

trong cùng request.

## 47. Checkpoint / Resume

Job ingestion phải resume được.

Stages ví dụ:

~~~text
stored
parsed
region planned
units built
embedding batches done
indexes built
validated
activated
~~~

Nếu batch embedding 7/10 fail thì retry batch 7, không chạy lại toàn PDF.

## 48. Retry / Backpressure

Cần hỗ trợ:

~~~text
timeout
rate limit
exponential backoff
partial retry
max retry
provider health
backpressure
~~~

**Backpressure** = cơ chế giảm tốc độ nhận việc khi downstream đang quá tải.

## 49. Lifecycle

Processing state:

~~~text
RECEIVED
STORED
PARSING
PLANNING
BUILDING
VALIDATING
INDEXING
ACTIVATING
READY
FAILED_RETRYABLE
FAILED_FINAL
~~~

Knowledge lifecycle:

~~~text
ACTIVE
SUPERSEDED
ARCHIVED
REVOKED
DELETED
~~~

Hai state machine khác nhau.

## 50. Revocation

Source bị revoke:

~~~text
stop retrieval immediately
→ invalidate active visibility
→ cleanup physical data later
~~~

Không chờ vector delete xong mới chặn query.

## 51. Duplicate Handling

Exact duplicate có thể dùng content hash.

Near duplicate không nên auto-delete vì provenance/permission có thể khác.

## 52. Chunking Evaluation

Chunking phải benchmark bằng downstream retrieval.

Metrics:

~~~text
Recall@K
MRR
nDCG
Context Precision
Citation Coverage
Duplicate Hit Ratio
Latency
Index Size
Cost
~~~

Không chọn chunk size chỉ vì nhìn chunk đẹp.

## 53. Case — Policy PDF

~~~text
policy.pdf
→ SourceVersion
→ parser
→ headings/paragraphs
→ hierarchical regions
→ child KnowledgeUnits
→ embeddings
→ lexical index
→ generation
→ activate
~~~

User query sau này retrieve child, expand parent section và cite page/section.

## 54. Case — XLSX Pricing Snapshot

~~~text
pricing.xlsx
→ StructuredArtifact
→ table
→ StructuredRecords
→ semantic projection optional
→ exact/structured index
→ vector projection optional
~~~

Nếu có live Pricing API, current-price query vẫn ưu tiên API.

## 55. Case — Scanned PDF

~~~text
scan.pdf
→ source analysis
→ no reliable text layer
→ OCR
→ canonical blocks
→ region/chunk
→ quality check OCR confidence
→ index
~~~

## 56. Component Responsibilities

| Component | Trách nhiệm |
|---|---|
| Source Adapter | discover/fetch source items |
| SourceVersion Manager | immutable snapshot/version |
| Parser | parse structure/content |
| OCRProvider | OCR khi cần |
| Canonical Builder | normalize artifact |
| Region Segmenter | chia vùng |
| Region Planner | chọn strategy |
| Chunker | tạo KnowledgeUnit |
| Structured Builder | tạo StructuredRecord |
| Quality Gate | validate |
| EmbeddingProvider | tạo vectors |
| VectorStore | lưu/search vectors |
| LexicalIndexer | tạo keyword index |
| Generation Manager | build/activate generations |
| Job Manager | checkpoint/retry |

## 57. Advanced Knowledge Structures — không phải baseline V1

Các kỹ thuật như:
- Knowledge Graph;
- GraphRAG;
- generated questions;
- document summaries;
- knowledge compilation;
- multi-vector/multimodal index;

có thể được thêm như **optional processing/index strategies** sau này.

Raglyra không đưa chúng vào baseline V1 vì:
1. tăng chi phí ingestion;
2. phức tạp version/evaluation;
3. không phải dataset nào cũng cần;
4. phải chứng minh retrieval uplift bằng evaluation trước.

Core architecture phải chừa extension point nhưng không phụ thuộc các kỹ thuật này.

## 58. File này không mô tả

- Query Understanding;
- Routing;
- Answer generation;
- DB schema cụ thể;
- API cụ thể;
- UI;
- provider/model cuối cùng.

## 59. Design Decisions

1. Source khác SourceItem và SourceVersion.
2. Source sync + change detection là first-class.
3. Ingestion job phải idempotent.
4. SourceVersion immutable.
5. SourceAdapter tách connector khỏi core.
6. Canonical Artifact không plain-text-only.
7. Mixed document xử lý theo region.
8. Chunking structure-first.
9. KnowledgeUnit khác ContextUnit.
10. StructuredRecord first-class.
11. Live structured data không bắt buộc vector hóa.
12. Provider/vector backend qua abstraction.
13. Scope/ACL metadata được kế thừa từ Source tới index point.
14. Embedding profile compatibility là invariant của dense retrieval.
15. Version layers tách.
16. ActiveGenerationManifest giữ generation tương thích.
17. Build generation mới rồi atomic activate.
18. Quality/Coverage bắt buộc.
19. Chunking phải evaluation.
20. Graph/advanced knowledge structures là optional extension, không baseline.

## 60. Open Questions

- default parser stack;
- OCR provider;
- exact CanonicalArtifact schema;
- region detection algorithm;
- default vector backend;
- default lexical backend;
- chunk profile presets;
- semantic chunking có cần V1;
- exact quality thresholds;
- source sync scheduler.

## 61. Review Checklist

- [ ] Hiểu Source/SourceItem/SourceVersion.
- [ ] Đồng ý Canonical Artifact.
- [ ] Đồng ý region-level processing.
- [ ] Đồng ý structure-first chunking.
- [ ] Đồng ý KnowledgeUnit vs ContextUnit.
- [ ] Đồng ý StructuredRecord.
- [ ] Đồng ý version layers.
- [ ] Đồng ý VectorStore abstraction.
- [ ] Đồng ý atomic generation activation.
- [ ] Đồng ý chunking evaluation.

## 62. Next Step

Sau khi Query Flow và Ingestion Flow đều REVIEWED:

~~~text
02-components/
→ thiết kế chi tiết từng component
~~~

Component đầu tiên nên là Conversation Engine vì nó là entry point quan trọng của Query Pipeline.