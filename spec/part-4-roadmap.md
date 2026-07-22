# Java Backend Engineering Handbook

## Project Specification

# PART 4 — Roadmap & Knowledge Architecture

> **Mục tiêu**
>
> Thiết kế toàn bộ handbook giống như thiết kế kiến trúc của một hệ thống phần mềm.
>
> Kiến thức sẽ được chia thành nhiều Phase, mỗi Phase gồm nhiều Chapter.
>
> Mỗi Chapter có dependency rõ ràng và được đánh dấu mức độ quan trọng.

---

# 1. Nguyên tắc xây dựng Roadmap

Roadmap **không được xây dựng theo tên package Java**.

Ví dụ KHÔNG nên:

```
Collections

↓

Thread

↓

JVM
```

Người học sẽ không hiểu vì sao học theo thứ tự đó.

---

Roadmap phải dựa trên

## Knowledge Dependency

Ví dụ.

```
Object

↓

equals()

↓

hashCode()

↓

HashMap

↓

ConcurrentHashMap
```

Không thể học HashMap trước.

---

Một ví dụ khác.

```
Memory

↓

Object

↓

GC

↓

Reference

↓

WeakHashMap
```

---

Một ví dụ nữa.

```
Thread

↓

Memory Visibility

↓

volatile

↓

synchronized

↓

Lock

↓

Thread Pool

↓

CompletableFuture
```

---

# 2. Quy tắc chia Phase

Một Phase phải đại diện cho **một nhóm kiến thức lớn**.

Ví dụ.

```
Java Foundation

Collections

Concurrency

Spring

Persistence

Production

System Design
```

Không chia theo số lượng chapter.

---

# 3. Độ ưu tiên của Chapter

Mỗi Chapter sẽ có hai loại đánh giá.

## 3.1. Importance

| Level | Ý nghĩa                    |
| ----- | -------------------------- |
| ⭐⭐⭐⭐⭐ | Bắt buộc phải hiểu rất sâu |
| ⭐⭐⭐⭐  | Quan trọng                 |
| ⭐⭐⭐   | Cần biết                   |
| ⭐⭐    | Nên biết                   |
| ⭐     | Tham khảo                  |

Ví dụ.

| Chapter     | Importance |
| ----------- | ---------- |
| JVM         | ⭐⭐⭐⭐⭐      |
| HashMap     | ⭐⭐⭐⭐⭐      |
| Transaction | ⭐⭐⭐⭐⭐      |
| Reflection  | ⭐⭐⭐        |
| Enum        | ⭐⭐         |

---

## 3.2. Interview Frequency

Bao nhiêu % khả năng xuất hiện.

Ví dụ.

| Chủ đề     | Frequency |
| ---------- | --------- |
| HashMap    | 95%       |
| Thread     | 90%       |
| String     | 90%       |
| JPA        | 95%       |
| Reflection | 30%       |

---

## 3.3. Ghi chú về tên trùng lặp có chủ đích (bổ sung v1.1)

> Ba khái niệm — **Transaction**, **Cache/Caching**, **Lock** — mỗi khái niệm xuất hiện ở
> **nhiều Phase khác nhau** (ví dụ Transaction ở cả Phase 5, 6, 7). Đây **không phải lỗi
> đánh máy** mà là ba chapter khác nhau nhìn cùng một khái niệm từ ba góc độ khác nhau
> (Spring / JPA / Database). Từ v1.1, mỗi lần trùng tên sẽ được viết rõ ràng dạng
> `Tên (Góc nhìn)` ngay trong roadmap để không còn mơ hồ khi đọc dependency graph. Khi viết
> nội dung, mỗi chapter phải mở đầu bằng một dòng `> Xem thêm góc nhìn khác:` trỏ sang các
> chapter cùng tên còn lại (đúng tinh thần Part 3 §9).

---

# 4. Kiến trúc toàn bộ Handbook

```
Java Backend Engineering Handbook

├── Phase 1 Foundation
│
├── Phase 2 Collections
│
├── Phase 3 Concurrency
│
├── Phase 4 Modern Java
│
├── Phase 5 Spring
│
├── Phase 6 Persistence
│
├── Phase 7 Database
│
├── Phase 8 Production
│
├── Phase 9 Architecture
│
└── Phase 10 Interview
```

---

# 5. Phase 1 — Java Foundation

Đây là nền móng.

Không được bỏ qua.

## Chapter

> **v1.1**: chèn thêm `08 OOP Fundamentals` và `17 Enum` (trước đây bị thiếu dù được
> nhắc tới như ví dụ độ sâu ở Part 3 §11 và §3.1 bên dưới) — các chapter phía sau dịch số.

```
01

Một chương trình Java chạy như thế nào?

↓

02

JDK JRE JVM

↓

03

Class Loader

↓

04

Runtime Data Areas

↓

05

Execution Engine

↓

06

JIT Compiler

↓

07

Garbage Collection

↓

08

OOP Fundamentals (Polymorphism, Encapsulation, Inheritance, Abstraction,
Overloading vs Overriding, final, super)

↓

09

Object

↓

10

String

↓

11

Immutable

↓

12

equals()

↓

13

hashCode()

↓

14

Object Lifecycle

↓

15

Exception

↓

16

Generics

↓

17

Enum

↓

18

Annotation

↓

19

Reflection

↓

20

IO

↓

21

NIO

↓

22

Module System
```

---

# 6. Phase 2 — Collections

> **v1.1**: đánh số toàn bộ Phase (trước đây chỉ Phase 1 có số) và chèn thêm
> `14 Comparable vs Comparator` (gap lộ ra từ câu hỏi phỏng vấn không map được vào chapter
> nào).

```
01 Collection Framework

↓

02 List

↓

03 ArrayList

↓

04 LinkedList

↓

05 Set

↓

06 HashSet

↓

07 Map

↓

08 HashMap

↓

09 LinkedHashMap

↓

10 TreeMap

↓

11 ConcurrentHashMap

↓

12 PriorityQueue

↓

13 Deque
```

Sau đó.

```
14 Comparable vs Comparator

↓

15 Collections Utility

↓

16 Immutable Collection

↓

17 CopyOnWrite

↓

18 WeakHashMap

↓

19 IdentityHashMap
```

---

# 7. Phase 3 — Concurrency

> **v1.1**: đánh số toàn bộ Phase, chèn thêm `14 Callable vs Runnable` (gap lộ ra từ câu
> hỏi phỏng vấn không map được), và đổi tên `09 Lock` thành `09 Lock (Java-level Locking)`
> để phân biệt với `Lock (Database-level Locking)` ở Phase 7 (xem §3.3).

```
01 Thread

↓

02 Thread Lifecycle

↓

03 Memory Visibility

↓

04 volatile

↓

05 synchronized

↓

06 Monitor

↓

07 wait()

↓

08 notify()

↓

09 Lock (Java-level Locking)

↓

10 ReentrantLock

↓

11 ReadWriteLock

↓

12 Atomic

↓

13 CAS

↓

14 Callable vs Runnable

↓

15 Thread Pool

↓

16 Executor

↓

17 ForkJoin

↓

18 CompletableFuture

↓

19 Virtual Thread
```

---

# 8. Phase 4 — Modern Java

```
01 Lambda

↓

02 Functional Interface

↓

03 Method Reference

↓

04 Stream

↓

05 Collector

↓

06 Optional

↓

07 Record

↓

08 Sealed Class

↓

09 Pattern Matching
```

---

# 9. Phase 5 — Spring

> **v1.1**: đánh số toàn bộ Phase; đổi tên `13 Transaction` → `13 Transaction (Spring
> Transaction Management)` và `16 Cache` → `16 Cache (Spring Cache Abstraction)` để phân
> biệt với các chapter cùng tên ở Phase 6, 7, 9 (xem §3.3).

```
01 Spring Core

↓

02 IoC

↓

03 DI

↓

04 Bean

↓

05 Bean Lifecycle

↓

06 Configuration

↓

07 Auto Configuration

↓

08 Spring Boot

↓

09 REST

↓

10 Validation

↓

11 Exception Handler

↓

12 AOP

↓

13 Transaction (Spring Transaction Management)

↓

14 Async

↓

15 Scheduler

↓

16 Cache (Spring Cache Abstraction)

↓

17 Security
```

---

# 10. Phase 6 — Persistence

> **v1.1**: đánh số toàn bộ Phase; chèn thêm `04 Multi-Datasource Configuration` (gap lộ ra
> từ câu hỏi phỏng vấn không map được); đổi tên `12 Transaction` → `12 Transaction (JPA
> Transaction Boundary)` để phân biệt với Phase 5, 7 (xem §3.3).

```
01 JPA

↓

02 Entity

↓

03 Repository

↓

04 Multi-Datasource Configuration

↓

05 Persistence Context

↓

06 Dirty Checking

↓

07 Entity State

↓

08 Lazy Loading

↓

09 Fetch Join

↓

10 N+1

↓

11 Cascade

↓

12 Transaction (JPA Transaction Boundary)

↓

13 Optimistic Lock

↓

14 Pessimistic Lock
```

---

# 11. Phase 7 — Database

> **v1.1**: đánh số toàn bộ Phase; đổi tên `05 Transaction` → `05 Transaction (Database
> ACID Transaction)` và `07 Lock` → `07 Lock (Database-level Locking)` để phân biệt với
> Phase 3, 5, 6 (xem §3.3).

```
01 SQL

↓

02 JOIN

↓

03 Index

↓

04 Execution Plan

↓

05 Transaction (Database ACID Transaction)

↓

06 Isolation

↓

07 Lock (Database-level Locking)

↓

08 MVCC

↓

09 Optimization
```

---

# 12. Phase 8 — Production

```
01 Logging

↓

02 Metrics

↓

03 Tracing

↓

04 HikariCP

↓

05 Redis

↓

06 Kafka

↓

07 Docker

↓

08 Kubernetes

↓

09 CI/CD

↓

10 JVM Tuning

↓

11 Performance Tuning
```

---

# 13. Phase 9 — Architecture

> **v1.1**: đánh số toàn bộ Phase; chèn thêm `07 Resilience Patterns` (Circuit Breaker,
> Retry, Bulkhead, Timeout — gap lộ ra từ câu hỏi phỏng vấn "slow external API kéo sập cả
> hệ thống" không map được vào chapter nào); đổi tên `08 Caching` → `08 Caching
> (Architecture-level Caching Strategy)` để phân biệt với Phase 5, 8 (xem §3.3).

```
01 Layered Architecture

↓

02 Hexagonal

↓

03 DDD

↓

04 CQRS

↓

05 Saga

↓

06 Outbox

↓

07 Resilience Patterns

↓

08 Caching (Architecture-level Caching Strategy)

↓

09 Event Driven

↓

10 Microservices
```

---

# 14. Phase 10 — Interview

Đây không phải kiến thức mới.

Đây là Phase tổng hợp.

Bao gồm.

```
01 Top 100 Java Questions

↓

02 Top 100 Spring Questions

↓

03 Production Questions

↓

04 Mock Interview

↓

05 Coding Test

↓

06 System Design
```

---

# 15. Kiến trúc Dependency giữa các Phase

```
Foundation
      │
      ▼
Collections
      │
      ▼
Concurrency
      │
      ▼
Modern Java
      │
      ▼
Spring
      │
      ▼
Persistence
      │
      ▼
Database
      │
      ▼
Production
      │
      ▼
Architecture
      │
      ▼
Interview
```

**Lưu ý quan trọng:** Mũi tên thể hiện **thứ tự học khuyến nghị**, không phải phụ thuộc tuyệt đối. Ví dụ, SQL cơ bản có thể học sớm hơn, nhưng các chủ đề như tối ưu truy vấn hay transaction nên được học sau khi đã nắm được JPA và cách ứng dụng sử dụng cơ sở dữ liệu.

---

# 16. Ma trận độ sâu của từng Phase

| Phase        | Mức độ |
| ------------ | ------ |
| Foundation   | ⭐⭐⭐⭐⭐  |
| Collections  | ⭐⭐⭐⭐⭐  |
| Concurrency  | ⭐⭐⭐⭐⭐  |
| Modern Java  | ⭐⭐⭐⭐   |
| Spring       | ⭐⭐⭐⭐⭐  |
| Persistence  | ⭐⭐⭐⭐⭐  |
| Database     | ⭐⭐⭐⭐   |
| Production   | ⭐⭐⭐⭐⭐  |
| Architecture | ⭐⭐⭐⭐   |
| Interview    | ⭐⭐⭐⭐   |

---

# 17. Ma trận đối tượng độc giả

| Phase        | Fresher | Junior | Middle | Senior    |
| ------------ | ------- | ------ | ------ | --------- |
| Foundation   | ✅       | ✅      | Ôn tập | Tham khảo |
| Collections  | ✅       | ✅      | ✅      | Ôn tập    |
| Concurrency  | Cơ bản  | ✅      | ✅      | ✅         |
| Modern Java  | Cơ bản  | ✅      | ✅      | ✅         |
| Spring       | ❌       | ✅      | ✅      | ✅         |
| Persistence  | ❌       | ✅      | ✅      | ✅         |
| Database     | Cơ bản  | ✅      | ✅      | ✅         |
| Production   | ❌       | Cơ bản | ✅      | ✅         |
| Architecture | ❌       | ❌      | Cơ bản | ✅         |
| Interview    | ✅       | ✅      | ✅      | ✅         |

---

# Kết thúc Part 4

