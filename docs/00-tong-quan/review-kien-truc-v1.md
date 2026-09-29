# Raglyra — Review Kien truc v1

> **Status:** REVIEWED
> **Muc tieu:** Ghi lai ket qua self-review bo kien truc Raglyra sau khi doi chieu voi cac pattern hien tai tu RAGFlow, Haystack, R2R, LlamaIndex, Microsoft GraphRAG va Docling.
> **Luu y:** File nay la review report. Implementation phai dua vao cac file design LATEST, khong dua vao file review nay.

---

## 1. Ket luan ngan

Sau vong review, kien truc Raglyra **du dieu kien de di tiep xuong Component Design**, voi cac dieu kien da duoc sua trong bo v2:

1. Tach AllowedScope va Policy Scope Validation.
2. Them Capability Registry va Source Health.
3. Them Source Resolver rieng.
4. Them QueryPlan Validation.
5. Gioi han query decomposition bang execution budget.
6. Them Source Sync / change detection / idempotency.
7. Them Embedding Profile Compatibility.
8. Them Active Generation Manifest.
9. Giu GraphRAG/knowledge graph/agentic retrieval la extension, khong baseline bat buoc.

---

## 2. Pattern tham khao — RAGFlow

RAGFlow hien co cac pattern dang chu y:

- document understanding truoc retrieval;
- ingestion pipeline tach Parser / Transformer / Chunker / Indexer;
- full-text + vector hybrid retrieval;
- reranking;
- metadata filtering;
- grounded citation;
- agentic retrieval cho query phuc tap;
- incremental data ingestion/synchronization.

Raglyra hoc:

~~~text
structure-aware ingestion
hybrid retrieval
reranker
metadata/scope filters
citation/provenance
bounded complex-query planning
~~~

Raglyra khong copy:

~~~text
agentic retrieval mac dinh cho moi query
knowledge compilation/graph bat buoc
storage/search backend cu the
~~~

Ly do: can benchmark tung ky thuat truoc khi dua vao baseline.

---

## 3. Pattern tham khao — Haystack

Haystack nhan manh:

- modular pipeline;
- component interface ro;
- routing/retrieval/memory/generation minh bach;
- branch/loop co kiem soat;
- provider/vendor agnostic;
- evaluation va observability.

Raglyra hoc:

~~~text
component boundaries
typed input/output contracts
provider abstractions
traceable pipeline
explicit routing
~~~

Day cung la ly do Raglyra khong de mot PublicChatService hoac Router khong lo om tat ca logic.

---

## 4. Pattern tham khao — R2R

R2R co:

- multimodal ingestion;
- hybrid semantic + keyword search;
- Reciprocal Rank Fusion;
- knowledge graph extension;
- REST API;
- user/access management;
- agentic RAG.

Raglyra hoc:

~~~text
hybrid retrieval
rank fusion
document management
access boundary
API-friendly architecture
~~~

Nhung access/security cua Raglyra phai tu dat invariant rieng va test cross-tenant, khong mac dinh tin implementation ben ngoai.

---

## 5. Pattern tham khao — Docling

Docling cho thay mot huong ingest tot:

~~~text
document structure
→ hierarchy-aware chunking
→ tokenizer-aware split/merge
→ metadata/headings/captions preserved
~~~

Raglyra da align o:

- Canonical Artifact;
- block/hierarchy;
- region-level planning;
- structure-first chunking;
- final embedding_text token validation;
- provenance.

Raglyra khong khoa parser vao Docling. Docling la mot adapter candidate.

---

## 6. Pattern tham khao — Microsoft GraphRAG

GraphRAG tach ro:

~~~text
Indexing Pipeline
khac
Query Engine
~~~

va co nhieu query mode cho cac loai bai toan khac nhau.

Raglyra hoc:

- ingestion/query separation;
- index artifacts/versioning;
- query strategy phai phu hop bai toan.

Raglyra khong dua graph extraction/community summary vao baseline V1 vi chi phi va complexity cao, va khong phai dataset nao cung can.

---

## 7. Pattern tham khao — LlamaIndex

LlamaIndex tach core va integrations cho:

- connectors;
- indices;
- retrievers;
- query engines;
- reranking;
- LLM/embedding/vector integrations.

Raglyra hoc pattern provider/backend abstraction va extension point.

Khong phu thuoc LlamaIndex framework trong core architecture.

---

## 8. Gap phat hien trong Raglyra v1

### Gap 1 — Scope Guard dat sai thu tu

v1:

~~~text
Standalone Query
→ Intent & Scope Guard
→ Query Understanding
~~~

Van de: chua Query Understanding thi chua biet task/topic/freshness de validate policy.

v2:

~~~text
Security Context
→ AllowedScope
→ Query Understanding
→ Entity Resolution
→ Capability Requirement
→ Policy Scope Validation
~~~

### Gap 2 — Capability chua co runtime registry

Da them Capability Registry + availability/source binding.

### Gap 3 — Thieu Plan Validation

Da them QueryPlan Validator truoc execute.

### Gap 4 — Complex query co nguy co agent loop

Da chot bounded decomposition voi budget.

### Gap 5 — Ingestion thieu incremental sync/idempotency

Da them sync cursor, content hash, upstream delete va idempotent job semantics.

### Gap 6 — Dense search cross-profile chua du ro

Da them Embedding Profile Compatibility.

### Gap 7 — Index generation co the tron artifact version

Da them Active Generation Manifest.

---

## 9. Nhung gi chua thiet ke xong

Kien truc high-level on, nhung chua du de code.

Can tiep tuc chi tiet:

~~~text
Conversation Engine
Routing / QueryPlan
Retrieval Engine
Evidence / Answering
Parser / Chunking components
Provider Abstraction
Evaluation / Observability
Data Model / ERD
API
UI
Deployment
Tasks
~~~

---

## 10. Readiness Decision

~~~text
High-level Architecture: REVIEWED
Query Flow v2: REVIEWED
Intent & Scope v2: REVIEWED
Ingestion v2: REVIEWED
Security Scope v2: REVIEWED
~~~

Du dieu kien bat dau component design.

Thu tu tiep theo:

~~~text
1. Conversation Engine
2. Routing / QueryPlan
3. Retrieval Engine
4. Evidence / Answering
5. Ingestion components
6. Provider abstraction
7. Evaluation / Observability
~~~