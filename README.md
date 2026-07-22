# Java Backend Engineering Handbook

Bộ tài liệu Java Backend bằng tiếng Việt, viết theo hướng "hiểu bản chất trước khi học API,
production-first" — xem đầy đủ triết lý ở [spec/part-1-vision.md](spec/part-1-vision.md).

## Cấu trúc repo

```
spec/    Specification gốc (Part 1-5) — KHÔNG được vi phạm khi viết nội dung
book/    Nội dung handbook, chia theo Phase → Chapter
```

## spec/ — nguồn chuẩn

| File | Nội dung |
| --- | --- |
| [part-1-vision.md](spec/part-1-vision.md) | Tầm nhìn, đối tượng độc giả, triết lý, phạm vi |
| [part-2-chapter-standard.md](spec/part-2-chapter-standard.md) | Template bắt buộc của **một chapter** |
| [part-3-book-wide-standard.md](spec/part-3-book-wide-standard.md) | Coding convention cho **toàn bộ sách** (markdown, diagram, domain ví dụ...) |
| [part-4-roadmap.md](spec/part-4-roadmap.md) | Roadmap 10 Phase, dependency giữa các chapter |
| [part-5-knowledge-graph.md](spec/part-5-knowledge-graph.md) | Metadata, Knowledge Graph, Interview/Production/Debug Mapping |
| [raw-interview-questions.md](spec/raw-interview-questions.md) | Danh sách câu hỏi phỏng vấn thô — đã được phân phối vào `book/` |
| [CHANGELOG.md](spec/CHANGELOG.md) | Lịch sử thay đổi specification (v1.0 → v1.1) |

## book/ — 10 Phase, 139 chapter

Cấu trúc thư mục đi theo đúng dependency graph ở Part 4 v1.2 (không theo package Java):

1. `phase-01-foundation` (22 chapter) — Java Core / JVM — có thêm OOP Fundamentals, Enum
2. `phase-02-collections` (19 chapter) — có thêm Comparable vs Comparator
3. `phase-03-concurrency` (19 chapter) — có thêm Callable vs Runnable
4. `phase-04-modern-java` (9 chapter)
5. `phase-05-spring` (18 chapter) — có thêm Reactive Programming (Project Reactor, WebFlux)
6. `phase-06-persistence` (14 chapter) — có thêm Multi-Datasource Configuration
7. `phase-07-database` (10 chapter) — có thêm NoSQL & Non-Relational Data
8. `phase-08-production` (12 chapter) — có thêm Messaging (JMS, RabbitMQ, Spring Integration)
9. `phase-09-architecture` (10 chapter) — có thêm Resilience Patterns
10. `phase-10-interview` (6 chapter + 1 file gap-log)

v1.2 thêm 3 chapter trên sau khi so sánh phạm vi với *Modern Java in Action* và
*Spring in Action* — chi tiết ở [spec/CHANGELOG.md](spec/CHANGELOG.md).

Mỗi file chapter đã được sinh sẵn theo đúng khung 20 mục của
[part-2-chapter-standard.md](spec/part-2-chapter-standard.md) (Story → Objectives →
Prerequisites → ... → Interview → Exercises → Cheat Sheet), phần nội dung còn để `TODO` —
viết dần theo tiêu chí hoàn thành ở Part 2 §24, mức độ bắt buộc từng mục theo Importance
(Part 2 §25).

Ba phần bổ sung được chính Part 5 đề xuất cũng đã scaffold sẵn:

- [book/glossary.md](book/glossary.md)
- [book/cross-reference-index.md](book/cross-reference-index.md)
- [book/production-playbook.md](book/production-playbook.md)

**Ba khái niệm cố ý trùng tên** ở nhiều Phase (Transaction, Cache/Caching, Lock) — xem Part 4
§3.3 để hiểu vì sao và cách phân biệt.

## Câu hỏi phỏng vấn đã map

48/49 dòng câu hỏi trong `spec/raw-interview-questions.md` đã được phân phối vào đúng
"Interview Questions" section của chapter tương ứng — **không còn câu nào bị bỏ sót**. 9 câu
từng không map được ở lần scaffold đầu (v1.0) đã có chapter riêng từ v1.1 — lịch sử ở
[book/phase-10-interview/00-resolved-gap-log.md](book/phase-10-interview/00-resolved-gap-log.md).

## Cách dùng

- Viết nội dung: mở file chapter trong `book/`, điền theo đúng thứ tự mục đã có sẵn,
  không tự ý đổi thứ tự (Part 2 §2).
- Chapter sau kế thừa chapter trước, không dạy lại — dùng `> Xem lại: ...` để tham chiếu
  ngược (Part 3 §9).
- Domain ví dụ xuyên suốt: **E-Commerce** (User, Product, Cart, Order, Payment, Inventory) —
  không dùng Animal/Dog/Student trừ khi đang dạy OOP thuần (Part 3 §8).
