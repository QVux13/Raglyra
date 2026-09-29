# Raglyra — Lo trinh thiet ke

> **Status:** REVIEWED
> **Muc tieu:** Quy dinh thu tu thiet ke de moi buoc deu dua tren phan da hieu va da review.

## Giai doan 1 — Hieu toan he thong

Doc va review:

~~~text
00-tong-quan/kien-truc-tong-the.md
~~~

Can hieu:
- Raglyra co 2 pipeline lon nao;
- Query Pipeline va Knowledge Pipeline gap nhau o dau;
- cac khoi Conversation, Planner, Retrieval, Evidence, Answering co vai tro gi.

## Giai doan 2 — Hieu hai luong chinh

### Query Pipeline — luong hoi-dap

~~~text
01-luong-hoi-dap/luong-xu-ly-cau-hoi.md
~~~

Can hieu tu:
User Message → Conversation → QueryPlan → Retrieval → Evidence → Answer.

### Knowledge Pipeline — luong nap tri thuc

~~~text
02-luong-nap-tri-thuc/luong-nap-va-danh-chi-muc.md
~~~

Can hieu tu:
Source → Parse/OCR → Chunk/Structured Record → Index → Active Knowledge.

## Giai doan 3 — Di sau tung chuc nang

Sau khi 2 flow da REVIEWED, moi viet chi tiet component ngay trong folder flow cua no.

Vi du Query Pipeline:

~~~text
01-luong-hoi-dap/
├── conversation-engine.md
├── query-understanding.md
├── routing-query-plan.md
├── retrieval-engine.md
└── evidence-answering.md
~~~

Knowledge Pipeline:

~~~text
02-luong-nap-tri-thuc/
├── source-versioning.md
├── parser-canonical-artifact.md
├── chunking.md
└── indexing-generation.md
~~~

## Giai doan 4 — Nen tang chung

~~~text
03-nen-tang-chung/
~~~

Thiet ke:
- multi-tenant/security;
- provider abstraction;
- evaluation/observability;
- configuration.

Nhung phan nay bao quanh nhieu flow, nen khong tach thanh hang loat folder rieng.

## Giai doan 5 — Thiet ke de code

~~~text
04-trien-khai/
~~~

Thu tu:

~~~text
Data Model / ERD
→ API
→ UI
→ Infrastructure
→ Testing
→ Migration
→ Task cho AI/dev
~~~

## Nguyen tac gate

Khong di tiep chi vi da viet xong file.

Moi phan can:
1. doc duoc;
2. hieu duoc;
3. review duoc;
4. chot cac design decision quan trong.

Neu flow con mo ho thi khong thiet ke DB/API de tranh phai dap lai sau.
