# Java Backend Engineering Handbook

## Project Specification

# PART 3 — Quy chuẩn toàn bộ Handbook

> **Mục tiêu của Part 3**
>
> Part 2 định nghĩa **một chapter phải viết như thế nào**.
>
> Part 3 định nghĩa **toàn bộ cuốn sách phải thống nhất ra sao**.
>
> Đây là "Coding Convention" của handbook.

---

# 1. Triết lý thiết kế của handbook

Handbook không được viết theo kiểu "tổng hợp kiến thức".

Nó phải được viết giống như một **hệ thống (System)**.

```
Foundation
      │
      ▼
Java Core
      │
      ▼
Collections
      │
      ▼
Concurrency
      │
      ▼
Spring
      │
      ▼
Hibernate
      │
      ▼
Production
      │
      ▼
System Design
```

Điều này có nghĩa:

* Chapter sau luôn kế thừa chapter trước.
* Không dạy lại kiến thức đã học.
* Luôn tham chiếu ngược nếu cần.

Ví dụ:

```
Chapter 18 - HashMap

↓

Không giải thích lại

equals()

↓

Tham chiếu

Chapter 11
```

---

# 2. Quy tắc đặt tên Chapter

Không dùng tên quá chung chung.

❌ Không nên

```
Collections
```

✅ Nên

```
HashMap hoạt động như thế nào?
```

---

❌ Không nên

```
Thread
```

✅ Nên

```
Điều gì xảy ra khi bạn tạo một Thread?
```

---

Tên chapter nên trả lời một câu hỏi.

Người đọc sẽ tò mò hơn.

---

## 2.1. Slug (roadmap) vs Title (nội dung)

> **Bổ sung v1.1** — làm rõ để không mâu thuẫn với Part 4.
>
> Quy tắc "tên chapter phải là câu hỏi" ở trên áp dụng cho **tiêu đề hiển thị (H1) bên
> trong file nội dung của chapter**, ví dụ H1 thật sự có thể là "Tại sao HashMap nhanh?".
>
> Roadmap ở Part 4 và mọi bảng phụ thuộc (Part 4 §3, Part 5 §2-4) dùng **danh từ ngắn**
> ("HashMap", "Lock", "Transaction") làm **slug/định danh** — phục vụ việc vẽ dependency
> graph, tra cứu, đặt tên file. Slug không cần và không nên viết dạng câu hỏi.
>
> Vậy: mỗi chapter có hai tên — slug ngắn (dùng trong roadmap, tên file, metadata) và
> title dạng câu hỏi (dùng làm H1 khi viết nội dung thật). Không được lẫn lộn hai vai trò
> này khi review.

---

# 3. Quy chuẩn Markdown

## Heading

Chỉ sử dụng:

```text
#

##

###

####
```

Không dùng Heading cấp 5 hoặc 6.

---

## Danh sách

Ưu tiên

```markdown
- Item A
- Item B
```

Không lạm dụng đánh số.

---

## Table

Chỉ dùng khi cần so sánh.

Ví dụ.

| ArrayList | LinkedList |
| --------- | ---------- |
| O(1) get  | O(n) get   |

---

## Callout

Có bốn loại chuẩn.

### Note

> **Note**
>
> Thông tin bổ sung.

---

### Tip

> **Tip**
>
> Kinh nghiệm thực tế.

---

### Warning

> **Warning**
>
> Điều dễ gây bug.

---

### Production

> **Production**
>
> Những điều chỉ gặp khi hệ thống chạy thật.

---

# 4. Quy chuẩn Diagram

Ưu tiên theo thứ tự.

```
ASCII

↓

Mermaid

↓

Code

↓

Ảnh
```

Không chèn ảnh nếu có thể diễn đạt bằng Mermaid hoặc ASCII.

Lý do:

* Dễ chỉnh sửa.
* Diff trên Git.
* Render trên GitHub.
* Render trên GitBook.

---

# 5. Quy chuẩn Mermaid

Chỉ sử dụng các loại sau:

| Loại            | Mục đích         |
| --------------- | ---------------- |
| flowchart       | Flow             |
| sequenceDiagram | Request/Response |
| classDiagram    | OOP              |
| stateDiagram    | State            |
| erDiagram       | Database         |
| journey         | User Flow        |
| gitGraph        | Git              |
| mindmap         | Tổng hợp         |

Không dùng Mermaid chỉ để trang trí.

---

# 6. Quy chuẩn ASCII Diagram

Flow đơn giản luôn dùng ASCII.

Ví dụ.

```
Client

↓

Controller

↓

Service

↓

Repository

↓

Database
```

---

Ví dụ HashMap.

```
hashCode()

↓

Bucket

↓

Node

↓

Value
```

ASCII phải đọc được ngay cả khi copy sang PDF hoặc Notion.

---

# 7. Quy chuẩn Code

## Java Version

Ưu tiên

```
Java 21
```

Nếu tính năng chỉ có từ Java 17 thì ghi rõ.

Nếu code tương thích Java 8 thì cũng ghi chú.

---

## Spring Version

```
Spring Boot 3.x
```

Không dùng XML configuration trừ khi giải thích lịch sử.

---

## Naming

Ví dụ xuyên suốt.

```
User

Order

Product

Cart

Payment

Inventory
```

Không dùng

```
Animal

Dog

Cat

Student
```

trừ khi đang dạy OOP.

---

## Code Style

Theo convention của Google hoặc OpenJDK:

* Tên biến rõ nghĩa.
* Không viết tắt khó hiểu.
* Không dùng magic number.
* Có comment khi logic phức tạp.

---

# 8. Quy chuẩn ví dụ

Toàn bộ handbook sử dụng **một domain xuyên suốt**.

## Domain chính

E-Commerce.

Bao gồm:

```
User

↓

Product

↓

Cart

↓

Order

↓

Payment

↓

Inventory

↓

Notification
```

Ví dụ HashMap.

```
Map<Long, Product>
```

Ví dụ Transaction.

```
OrderService
```

Ví dụ Lock.

```
InventoryService
```

Lợi ích:

Người đọc không phải học lại context ở mỗi chapter.

---

# 9. Quy chuẩn liên kết giữa các chapter

Nếu chapter A sử dụng kiến thức của chapter B:

Không giải thích lại.

Thay vào đó:

> **Xem lại:** Chapter X – Tên chương.

Ví dụ:

```
HashMap

↓

hashCode()

↓

Tham chiếu

Chapter Object
```

Điều này giúp handbook không bị lặp.

---

# 10. Quy chuẩn đánh số

Heading:

```
1

1.1

1.2

2

2.1
```

Hình:

```
Hình 2.1

Hình 2.2
```

Code:

```
Listing 3.1

Listing 3.2
```

Bảng:

```
Bảng 5.1
```

Điều này giúp dễ tham chiếu trong PDF hoặc GitBook.

---

# 11. Quy chuẩn độ sâu

Không phải chapter nào cũng dài như nhau.

Phân loại:

| Loại       | Độ sâu |
| ---------- | ------ |
| Foundation | ★★★    |
| Quan trọng | ★★★★★  |
| Tham khảo  | ★★     |

Ví dụ:

| Chủ đề      | Độ sâu |
| ----------- | ------ |
| JVM         | ★★★★★  |
| HashMap     | ★★★★★  |
| Thread      | ★★★★★  |
| Transaction | ★★★★★  |
| Annotation  | ★★★    |
| Enum        | ★★     |

Chương quan trọng sẽ có nhiều ví dụ, deep dive và production case hơn.

---

# 12. Quy chuẩn chất lượng ví dụ

Mỗi ví dụ nên trả lời được:

* Mục đích của đoạn code?
* Kết quả mong đợi là gì?
* Điều gì sẽ xảy ra nếu sửa một dòng?
* Có lỗi phổ biến nào không?

Không chỉ đưa code mà không giải thích.

---

# 13. Quy chuẩn Production Case

Mỗi production case nên có cấu trúc thống nhất:

```text
Bối cảnh

↓

Triệu chứng

↓

Nguyên nhân gốc

↓

Quá trình điều tra

↓

Cách khắc phục

↓

Cách phòng tránh
```

Nếu có thể, bổ sung log, stack trace hoặc metric minh họa.

---

# 14. Quy chuẩn về nguồn tham khảo

Nguồn ưu tiên:

1. Java Language Specification (JLS).
2. JVM Specification.
3. OpenJDK Source.
4. Spring Official Documentation.
5. Hibernate Official Documentation.
6. PostgreSQL Documentation.
7. Oracle Documentation.
8. Effective Java.
9. Java Concurrency in Practice.
10. Designing Data-Intensive Applications.

Nguồn thứ cấp (blog, Stack Overflow...) chỉ dùng để tham khảo, không dùng làm cơ sở cho các khẳng định kỹ thuật.

---

# 15. Quy chuẩn cập nhật

Công nghệ thay đổi theo thời gian.

Vì vậy:

* Ghi rõ phiên bản Java/Spring áp dụng.
* Nếu hành vi thay đổi giữa các phiên bản, phải chỉ rõ sự khác biệt.
* Đánh dấu những nội dung đã lỗi thời (deprecated) và giới thiệu cách tiếp cận hiện đại.

Ví dụ:

* Java 8 vs Java 21.
* Spring Boot 2.x vs 3.x.
* `javax.*` vs `jakarta.*`.

---

# Kết thúc Part 3

