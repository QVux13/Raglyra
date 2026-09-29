# Raglyra — Ban do tai lieu

> Branch thiet ke: `planning`
>
> Muc tieu cua cay `docs/`: nhin ten folder la biet **phan nao cua he thong RAG** dang duoc mo ta.

## 1. Cau truc don gian

~~~text
docs/
├── 00-tong-quan/
├── 01-luong-hoi-dap/
├── 02-luong-nap-tri-thuc/
├── 03-nen-tang-chung/
└── 04-trien-khai/
~~~

## 2. Y nghia tung folder

### 00-tong-quan — Overall / Tong quan he thong

Day la noi de hieu Raglyra tu tren xuong.

No tra loi:
- Raglyra la gi?
- RAG trong du an nay chay theo kien truc nao?
- Co bao nhieu luong lon?
- Tai lieu nao nen doc truoc/sau?
- Quy tac viet tai lieu la gi?

Khong di sau vao tung class/API/DB.

### 01-luong-hoi-dap — Query Pipeline / Luong xu ly khi user hoi

Folder nay dai dien cho **chuc nang runtime khi user chat**.

No bao phu:

~~~text
User Message
→ Conversation Context
→ Query Understanding
→ Routing / QueryPlan
→ Retrieval / Structured Source
→ Evidence
→ Answer
→ Citation
~~~

Moi file chi tiet ve conversation, routing, retrieval, answering sau nay se nam trong folder nay de ban khong phai nhay qua nhieu folder.

### 02-luong-nap-tri-thuc — Knowledge Pipeline / Luong dua du lieu vao RAG

Folder nay dai dien cho **chuc nang nap va chuan bi knowledge**.

No bao phu:

~~~text
File / Web / API / Data Source
→ Parse / OCR
→ Chuan hoa
→ Chunk / Structured Record
→ Embedding / Index
→ San sang cho Retrieval
~~~

Moi file chi tiet ve parser, chunking, indexing, versioning sau nay se nam o day.

### 03-nen-tang-chung — Platform Foundation / Nen tang dung chung

Folder nay chua cac chuc nang **bao quanh ca Query Pipeline va Knowledge Pipeline**.

Vi du:
- Multi-tenant = mot he thong phuc vu nhieu khach hang nhung phai cach ly data.
- Security = quyen truy cap, scope, token, secret.
- Provider Abstraction = lop trung gian de doi OpenAI/Gemini/Qwen/Qdrant/pgvector ma core it bi anh huong.
- Evaluation = do chat luong RAG.
- Observability = theo doi he thong dang chay ra sao.

Day khong phai mot buoc duy nhat trong flow; no la nen tang dung chung cho toan bo he thong.

### 04-trien-khai — Implementation Design / Thiet ke de bien kien truc thanh code

Folder nay chi duoc di sau sau khi 3 phan tren da ro.

No se chua:
- Data Model / ERD = thiet ke bang DB va quan he.
- API = request/response/sequence tung endpoint.
- UI = man hinh va flow thao tac.
- Infrastructure = PostgreSQL, Qdrant/pgvector, Redis, Object Storage, deployment.
- Testing = test component, integration, performance.
- Migration = tan dung/chuyen tu code cu neu can.
- Tasks = task chi tiet de AI/dev code.

## 3. Cach doc tai lieu

Thu tu:

~~~text
00-tong-quan/kien-truc-tong-the.md
        ↓
01-luong-hoi-dap/luong-xu-ly-cau-hoi.md
        ↓
02-luong-nap-tri-thuc/luong-nap-va-danh-chi-muc.md
        ↓
03-nen-tang-chung/
        ↓
04-trien-khai/
~~~

## 4. Quy tac version

Trong cay docs chi giu **mot file hien tai cho moi chu de**.

Khong tao:
- file-v0.1.md
- file-v0.2.md
- file-self-review.md

Khi cap nhat:
- sua chinh file hien tai;
- Git commit/history giu lich su cu;
- cay docs luon chi hien ban moi nhat.

## 5. Quy tac trang thai

Moi file co the co:
- DRAFT = dang thiet ke.
- REVIEWED = da review, co the lam dependency cho phan sau.
- APPROVED = da chot de implementation.

## 6. Nguyen tac quan trong

Kien truc RAG target quyet dinh code.

Source code cu chi duoc xem la nguon de:
- tai su dung ham/module tot;
- rut ngan implementation;
- tham khao behavior hien tai.

Khong de legacy code ep kien truc moi di theo loi cu.
