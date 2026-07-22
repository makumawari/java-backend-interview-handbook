# Java Backend Engineering Handbook

## Project Specification

# PART 5 — Knowledge Graph & Competency Matrix

---

# 1. Mục tiêu

Handbook không chỉ để đọc từ đầu đến cuối.

Mà còn phải hỗ trợ

* tra cứu
* ôn phỏng vấn
* debug production
* lập roadmap học

Nghĩa là mỗi chapter đều phải biết

* mình phụ thuộc vào chapter nào
* chapter nào sẽ dùng mình
* mình giải quyết vấn đề gì

---

# 2. Mỗi Chapter phải có Metadata

Đây là điều bắt buộc.

Ví dụ.

```yaml
Chapter: HashMap

Difficulty: ★★★★★

Importance: ★★★★★

Interview Frequency: 95%

Prerequisites:
    - Object
    - equals()
    - hashCode()

Used Later:
    - ConcurrentHashMap
    - Spring Cache
    - Hibernate
    - Redis

Production:
    - Duplicate Key
    - Collision
    - Memory Leak

Estimated Reading:
    45 phút

Estimated Practice:
    2 giờ
```

Đây là metadata của chapter.

Không phải nội dung.

---

# 3. Dependency Graph

Ví dụ.

```text
Object
    │
    ▼
equals()
    │
    ▼
hashCode()
    │
    ▼
HashMap
    │
    ▼
ConcurrentHashMap
    │
    ▼
Spring Cache
```

Điều này rất quan trọng.

Ví dụ.

Người đọc hỏi

"Tôi muốn học ConcurrentHashMap."

Handbook sẽ biết.

```text
Bạn còn thiếu

equals()

↓

hashCode()

↓

HashMap
```

---

# 4. Knowledge Graph

Đây là phần mở rộng.

Ví dụ.

```text
Thread

├── synchronized

├── volatile

├── Lock

├── CAS

├── Atomic

├── Thread Pool

└── CompletableFuture
```

Toàn bộ handbook sẽ tạo thành một đồ thị.

Không phải một danh sách.

---

# 5. Competency Matrix

Mỗi chapter sẽ giúp tăng một hoặc nhiều năng lực.

Ví dụ.

## HashMap

| Năng lực    | Level |
| ----------- | ----- |
| Java Core   | ★★★★★ |
| Debug       | ★★★   |
| Performance | ★★★★  |
| Interview   | ★★★★★ |

---

Ví dụ.

## Transaction

| Năng lực   | Level |
| ---------- | ----- |
| Spring     | ★★★★★ |
| Database   | ★★★★★ |
| Production | ★★★★★ |
| Interview  | ★★★★★ |

---

# 6. Interview Mapping

Đây là phần rất hay.

Ví dụ.

Một câu hỏi.

```text
Tại sao phải override
equals()

và

hashCode()?
```

Handbook sẽ biết.

Nó liên quan tới.

```text
Object

↓

equals()

↓

hashCode()

↓

HashMap

↓

HashSet

↓

JPA Entity

↓

Hibernate
```

Như vậy.

Một câu hỏi.

Có thể ôn được nhiều chapter.

---

# 7. Production Mapping

Ví dụ.

Production Issue.

```text
Duplicate Record
```

Handbook sẽ gợi ý.

```text
Transaction

↓

Isolation

↓

Lock

↓

Unique Index

↓

Optimistic Lock

↓

Pessimistic Lock
```

---

Ví dụ.

```text
API chậm
```

Sẽ dẫn tới.

```text
Index

↓

Execution Plan

↓

N+1

↓

Lazy Loading

↓

Cache

↓

Connection Pool
```

---

# 8. Debug Mapping

Ví dụ.

```text
OutOfMemoryError
```

Sẽ liên kết tới.

```text
JVM Memory

↓

GC

↓

Heap

↓

Reference

↓

Memory Leak
```

---

Ví dụ.

```text
StackOverflowError
```

↓

```text
Call Stack

↓

Recursion
```

---

# 9. Coding Mapping

Ví dụ.

Một bài tập.

```text
LRU Cache
```

Cần.

```text
HashMap

+

LinkedList
```

---

Ví dụ.

```text
Producer Consumer
```

Cần.

```text
Thread

↓

BlockingQueue
```

---

# 10. System Design Mapping

Ví dụ.

```text
Inventory Service
```

Sẽ liên quan.

```text
Transaction

↓

Lock

↓

Kafka

↓

Redis

↓

Cache

↓

Database
```

---

# 11. OpenJDK Mapping

Đây là phần mình rất muốn có.

Ví dụ.

HashMap.

```text
HashMap

↓

putVal()

↓

resize()

↓

treeifyBin()
```

---

Ví dụ.

Thread.

```text
Thread.start()

↓

JVM

↓

OS Thread
```

---

Ví dụ.

String.

```text
String.java
```

↓

```text
coder

↓

value

↓

hash
```

---

# 12. Spring Source Mapping

Ví dụ.

```text
@Transactional

↓

TransactionInterceptor

↓

PlatformTransactionManager
```

---

Ví dụ.

```text
@RestController

↓

DispatcherServlet

↓

HandlerAdapter

↓

Controller
```

---

# 13. Hibernate Mapping

Ví dụ.

```text
save()

↓

PersistenceContext

↓

ActionQueue

↓

Flush
```

---

Ví dụ.

```text
Dirty Checking

↓

EntityEntry

↓

Snapshot
```

---

# 14. SQL Mapping

Ví dụ.

```text
N+1
```

↓

```text
JOIN

↓

Fetch Join

↓

Execution Plan
```

---

# 15. Production Case Mapping

Ví dụ.

```text
Connection Pool Exhausted
```

↓

```text
HikariCP

↓

Transaction

↓

Leak

↓

Slow Query
```

---

Ví dụ.

```text
Deadlock
```

↓

```text
Isolation

↓

Lock

↓

Transaction
```

---

# 16. Learning Path

Handbook phải hỗ trợ nhiều cách học.

Ví dụ.

## Roadmap Junior

```text
Foundation

↓

Collections

↓

Spring

↓

JPA

↓

SQL

↓

Interview
```

---

## Roadmap Performance

```text
JVM

↓

GC

↓

HashMap

↓

Cache

↓

SQL

↓

Redis
```

---

## Roadmap Backend

```text
Java

↓

Spring

↓

JPA

↓

Docker

↓

Production
```

---

## Roadmap Architect

```text
Foundation

↓

Concurrency

↓

JVM

↓

Spring

↓

Kafka

↓

DDD

↓

System Design
```

---

# 17. Chapter Tags

Mỗi chapter có tag.

Ví dụ.

```yaml
Tags:

Java

JVM

Performance

Interview

Production
```

Giúp tìm kiếm nhanh.

---

# 18. Estimated Time

Ví dụ.

```yaml
Reading

45 phút

Coding

2 giờ

Review

20 phút

Interview

15 phút
```

---

# 19. Difficulty

```text
★

Rất dễ

★★

Dễ

★★★

Trung bình

★★★★

Khó

★★★★★

Rất khó
```

---

# 20. Completion Criteria

Một chapter chỉ được đánh dấu hoàn thành khi:

```text
✓ Đã đọc

✓ Đã code

✓ Đã làm bài tập

✓ Đã trả lời interview

✓ Đã đọc source

✓ Đã hiểu production
```

---

# 21. Quan hệ giữa các cơ chế Mapping (tránh trùng lặp)

> **Bổ sung v1.1** — §6 (Interview Mapping), §7 (Production Mapping), §8 (Debug Mapping)
> và Cross-Reference Index (đề xuất bổ sung bên dưới) đều làm chung một việc: "đi từ một
> câu hỏi/triệu chứng tới danh sách chapter liên quan". Nếu không phân vai rõ, bốn cơ chế
> này sẽ trôi dạt (drift) và không đồng bộ khi sách phát triển. Phân vai như sau:

| Cơ chế | Nằm ở đâu | Vai trò | Chi tiết đến mức nào |
| --- | --- | --- | --- |
| Interview/Production/Debug Mapping (§6-8) | Bên trong **từng chapter** | Chapter tự khai bên trong nó dẫn tới/từ đâu | Chi tiết, gắn với 1 chapter cụ thể |
| Cross-Reference Index | File riêng ở cấp sách (`book/cross-reference-index.md`) | Bảng tra cứu nhanh cấp **toàn sách**, gộp nhiều mapping lại | Ngắn gọn, 1 dòng/câu hỏi |
| Production Playbook | File riêng ở cấp sách (`book/production-playbook.md`) | Sổ tay xử lý sự cố, đi từ **triệu chứng production** (không phải câu hỏi phỏng vấn) | Theo khung Bối cảnh→Triệu chứng→Nguyên nhân→Điều tra→Khắc phục→Phòng tránh (Part 3 §13) |
| Glossary | File riêng ở cấp sách (`book/glossary.md`) | Định nghĩa thuật ngữ 1 lần duy nhất | Không phải mapping, chỉ là định nghĩa + link |

Quy tắc cập nhật: khi viết nội dung một chapter và thêm Interview/Production Mapping bên
trong chapter đó, **phải đồng bộ tay** dòng tương ứng ở Cross-Reference Index hoặc
Production Playbook nếu có. Đây là index thủ công, không tự sinh — chấp nhận đánh đổi để
giữ đơn giản.

---

# Đề xuất bổ sung để handbook đạt chất lượng "10/10"

Sau khi hoàn thiện 5 phần Specification, mình thấy vẫn còn **ba mảnh ghép còn thiếu**. Đây là những thứ mình hiếm khi thấy trong các handbook khác nhưng lại rất hữu ích:

## 1. Glossary (Từ điển thuật ngữ)

Một chương riêng định nghĩa các thuật ngữ như:

* Heap, Stack, GC Root.
* IoC, DI, Bean.
* Dirty Checking, Persistence Context.
* CAS, Memory Barrier.
* MVCC, Isolation Level.

Mỗi thuật ngữ chỉ định nghĩa **một lần** và các chapter khác sẽ tham chiếu lại, tránh lặp nội dung.

---

## 2. Cross-Reference Index

Một chỉ mục cuối sách, ví dụ:

| Muốn hiểu                               | Đọc các chapter                       |
| --------------------------------------- | ------------------------------------- |
| Vì sao `@Transactional` không rollback? | Transaction, AOP, Proxy, Exception    |
| Vì sao HashMap mất dữ liệu?             | HashMap, equals/hashCode, Concurrency |
| API chậm                                | SQL, Index, Cache, HikariCP, JVM      |

Điều này biến handbook thành một tài liệu tra cứu rất mạnh.

---

## 3. Production Playbook

Một phần riêng tập hợp các tình huống thực tế:

* API chậm.
* Memory Leak.
* OutOfMemoryError.
* Deadlock.
* Duplicate Record.
* Connection Pool Exhausted.
* Cache Stale.
* N+1 Query.

Mỗi tình huống sẽ dẫn người đọc tới các chapter liên quan, giống như một "sổ tay xử lý sự cố".

Theo mình, nếu bổ sung ba phần này vào Specification thì handbook sẽ không chỉ là một cuốn sách học Java, mà sẽ trở thành **một hệ thống tri thức hoàn chỉnh** phục vụ học tập, phỏng vấn và công việc thực tế.
