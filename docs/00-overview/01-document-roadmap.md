# Raglyra — Documentation Roadmap

> **Status:** REVIEWED  
> **Mục tiêu:** Quy định thứ tự thiết kế để dự án dễ kiểm soát.

## Phase 1 — Overview

~~~text
00-system-architecture.md
01-document-roadmap.md
~~~

Trả lời:
- Raglyra gồm những khối nào?
- Thiết kế theo thứ tự nào?

## Phase 2 — Flows

~~~text
01-query-answer-flow.md
02-ingestion-indexing-flow.md
~~~

Trả lời:
- User hỏi thì hệ thống chạy từng bước nào?
- Dữ liệu được nạp vào RAG từng bước nào?

## Phase 3 — Components

Thiết kế chi tiết từng khối:

~~~text
conversation
query-understanding
routing-planning
retrieval
evidence
answering
ingestion
providers
security
evaluation
~~~

Mỗi component doc phải có:
- trách nhiệm;
- input;
- output;
- dữ liệu/domain object;
- sequence;
- error;
- config;
- test;
- extension point.

## Phase 4 — Data Model

Sau component contract mới thiết kế:

~~~text
DB schema
ERD
versioning
index generations
conversation state
trace/evaluation data
~~~

## Phase 5 — API

Sau DB + component:

~~~text
API convention
auth/security context
knowledge API
chat API
assistant/bot API
provider/config API
evaluation API
admin API
~~~

Mỗi API phải có:
- URL/method;
- request;
- response;
- permission;
- sequence;
- error code;
- idempotency nếu cần.

## Phase 6 — UI

Thiết kế:
- admin information architecture;
- knowledge management;
- ingestion monitor;
- assistant config;
- conversation debugger;
- evaluation dashboard;
- public widget.

## Phase 7 — Infrastructure

Thiết kế:
- PostgreSQL;
- pgvector/Qdrant;
- Redis/queue;
- object storage;
- provider configuration;
- deployment;
- observability;
- secrets.

## Phase 8 — Testing & Evaluation

Thiết kế:
- unit/integration tests;
- conversation golden set;
- routing golden set;
- retrieval evaluation;
- grounding/citation evaluation;
- security negative tests;
- benchmark.

## Phase 9 — Migration

Nếu tái sử dụng code cũ:
- inventory;
- mapping;
- data migration;
- re-index;
- cutover.

Target architecture quyết định migration, không làm ngược lại.

## Phase 10 — Tasks

Cuối cùng mới tạo task để AI/dev code.

Task phải cụ thể:

~~~text
TASK-001
Objective
Dependencies
Exact module/files
Input/Output contract
Allowed changes
Forbidden changes
Tests
Acceptance criteria
Rollback
~~~

Không tạo task kiểu "refactor RAG cho tốt hơn".
