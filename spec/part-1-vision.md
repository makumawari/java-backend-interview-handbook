# Java Backend Engineering Handbook

## Project Specification

Version: 1.1

> **v1.1 (2026-07-23)**: sửa các mâu thuẫn/gap phát hiện khi scaffold repo lần đầu — xem
> [`spec/CHANGELOG.md`](CHANGELOG.md) để biết chi tiết từng thay đổi.

---

# PART 1 — Tầm nhìn, mục tiêu và triết lý

---

# 1. Tầm nhìn (Vision)

Mục tiêu của dự án này là xây dựng một bộ tài liệu Java Backend bằng tiếng Việt có chất lượng tương đương các đầu sách kỹ thuật nổi tiếng như:

* Effective Java
* Spring in Action
* Java Concurrency in Practice
* Designing Data-Intensive Applications
* Head First Design Patterns

Tuy nhiên, handbook này không thay thế các cuốn sách trên mà đóng vai trò là **bản đồ kiến thức** kết nối chúng lại với nhau.

Sau khi hoàn thành, người đọc có thể:

* Học Java từ nền tảng đến nâng cao.
* Ôn phỏng vấn từ Junior đến Senior.
* Tra cứu khi đi làm.
* Debug các vấn đề Production.
* Hiểu cách Spring, Hibernate và JVM hoạt động bên trong.
* Xây dựng tư duy của một Java Backend Engineer.

---

# 2. Đối tượng độc giả

Handbook hướng đến bốn nhóm đối tượng.

## Nhóm 1 — Người mới

Ví dụ:

* Sinh viên.
* Fresher.
* Người chuyển ngành.

Mục tiêu:

Hiểu Java từ đầu.

---

## Nhóm 2 — Junior

Khoảng:

0–2 năm kinh nghiệm.

Mục tiêu:

* Đi phỏng vấn.
* Làm việc được với Spring Boot.

---

## Nhóm 3 — Middle

Khoảng:

2–5 năm.

Mục tiêu:

* Hiểu bản chất.
* Debug.
* Thiết kế hệ thống.

---

## Nhóm 4 — Senior

Mục tiêu:

Tra cứu.

Đọc source code.

Production.

---

# 3. Mục tiêu của handbook

Không phải:

```
Java Interview Questions
```

Không phải:

```
Java Cheat Sheet
```

Không phải:

```
Java Tutorial
```

Mà là

```
Java Backend Engineering Handbook
```

---

# 4. Triết lý

Có ba nguyên tắc lớn.

---

## Nguyên tắc 1

**Hiểu bản chất trước khi học API.**

Ví dụ.

Không học

```
HashMap.put()
```

trước.

Mà học

```
Hash Function

↓

Bucket

↓

Collision

↓

Resize
```

Sau đó mới học API.

---

## Nguyên tắc 2

**Không học thuộc lòng.**

Ví dụ.

Không hỏi

```
String immutable là gì?
```

Mà hỏi

```
Nếu String mutable thì chuyện gì xảy ra?
```

---

## Nguyên tắc 3

**Production First.**

Ví dụ.

Không học

```
@Transactional
```

một cách lý thuyết.

Mà học

```
Transaction không rollback

↓

Debug

↓

Root Cause

↓

Fix
```

---

# 5. Sáu câu hỏi mỗi chapter bắt buộc phải trả lời

Đây là quy tắc quan trọng nhất.

Mỗi chapter đều phải trả lời đầy đủ sáu câu hỏi.

## What?

Nó là gì?

---

## Why?

Tại sao nó tồn tại?

---

## How?

Nó hoạt động như thế nào?

---

## When?

Khi nào nên dùng?

Khi nào không nên dùng?

---

## What if?

Nếu dùng sai thì sao?

Nếu bỏ nó đi thì sao?

Nếu thay bằng công nghệ khác thì sao?

---

## Production?

Trong Production gặp vấn đề gì?

Debug như thế nào?

---

# 6. Tiêu chí chất lượng

Một chapter chỉ được coi là hoàn thành khi đạt đủ các tiêu chí sau:

* Giải thích được bản chất.
* Có ví dụ Java chạy được.
* Có minh họa trực quan.
* Có lưu ý về production (nếu phù hợp).
* Có checklist debug (nếu phù hợp).
* Có câu hỏi ôn tập.
* Có câu hỏi phỏng vấn theo cấp độ.
* Có bài tập.
* Có phần tóm tắt cuối chương.
* Có liên kết tới các chapter liên quan.

---

# 7. Triết lý trình bày

Thay vì:

```
Definition

↓

Explanation
```

Handbook sẽ sử dụng mô hình:

```
Problem

↓

Question

↓

Think

↓

Concept

↓

Visualization

↓

Example

↓

Production

↓

Summary
```

Điều này giúp người đọc hiểu được **tại sao** trước khi biết **là gì**.

---

# 8. Phạm vi kiến thức

Handbook sẽ bao phủ:

* Java Core.
* JVM.
* Collections.
* Concurrency.
* Modern Java.
* Spring.
* Spring Boot.
* Spring Data JPA.
* Hibernate.
* SQL.
* Redis.
* Kafka.
* Docker.
* Kubernetes.
* Production.
* System Design.

---

# 9. Mục tiêu cuối cùng

Sau khi đọc hết handbook, người đọc nên có khả năng:

* Trả lời phần lớn câu hỏi phỏng vấn Java Backend.
* Đọc hiểu source code của OpenJDK, Spring và Hibernate ở mức cơ bản.
* Debug các lỗi phổ biến trong môi trường Production.
* Thiết kế và xây dựng một ứng dụng Spring Boot hoàn chỉnh.
* Có nền tảng để phát triển lên Tech Lead hoặc Solution Architect.

---

Đây là **Part 1** của Specification.

