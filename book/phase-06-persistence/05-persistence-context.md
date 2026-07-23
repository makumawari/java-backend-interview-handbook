---
tags:
  - JPA
  - PersistenceContext
  - Hibernate
---

# Persistence Context (First-Level Cache)

> Phase: Phase 6 — Persistence
> Chapter slug: `persistence-context`

## Metadata

```yaml
Chapter: Persistence Context
Phase: Phase 6 — Persistence
Difficulty: ★★★★
Importance: ★★★★★
Interview Frequency: 75%
Prerequisites:
  - Chapter 01 — JPA
  - Chapter 02 — Entity
Used Later:
  - Chapter 06 — Dirty Checking
  - Chapter 07 — Entity State
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 01](01-jpa.md) — `em.getDelegate()` cho thấy `EntityManager` được backing
> bởi `org.hibernate.internal.SessionImpl`. Nhưng "Session" đó thực chất **giữ trạng thái** gì
> bên trong nó?

Gọi `em.find(Customer.class, id)` **hai lần** với cùng `id`, trong cùng một transaction, đếm số
câu SQL thật sự chạy:

```
=== TRONG CUNG mot transaction/persistence context ===
Hibernate: select c1_0.id,c1_0.name from customer c1_0 where c1_0.id=?
Lan 1: em.find() -> co SELECT that
Lan 2: em.find() -> 
first == second (CUNG mot object trong bo nho): true
```

Chỉ **một** câu `SELECT` cho hai lần gọi `find()` — và `first == second` là `true`: cùng một
object trong bộ nhớ, không phải hai object khác nhau có cùng dữ liệu.

## Interview Question (Central)

> Persistence Context là gì? Vì sao gọi `EntityManager.find()` hai lần với cùng ID trong cùng
> một transaction chỉ sinh ra một câu SQL?

## Objectives

- [ ] Hiểu Persistence Context là "bộ nhớ đệm cấp một" (first-level cache) gắn liền với vòng đời
      một `EntityManager`/transaction
- [ ] Tự tay chứng minh bằng thực nghiệm: `find()` lặp lại trong cùng transaction không sinh
      thêm SQL, trả về cùng object reference
- [ ] Tự tay chứng minh Persistence Context **không** chia sẻ giữa hai transaction khác nhau

## Prerequisites

- Chapter 01 — hiểu `EntityManager` được backing bởi `SessionImpl` của Hibernate.
- Chapter 02 — hiểu Entity, đối tượng được Persistence Context theo dõi.

## Used Later

- **Chapter 06 (Dirty Checking)** — cơ chế tự động phát hiện thay đổi entity dựa trực tiếp trên
  dữ liệu được Persistence Context lưu giữ.
- **Chapter 07 (Entity State)** — các trạng thái Managed/Detached của entity được định nghĩa
  chính xác dựa trên việc nó có nằm trong Persistence Context hay không.

## Problem

Nếu mỗi lần gọi `find()`/truy vấn cùng một entity đều tạo một object Java **mới hoàn toàn** (dù
cùng dữ liệu), sẽ nảy sinh hai vấn đề: (1) lãng phí — truy vấn lại database cho dữ liệu đã có
sẵn trong cùng một luồng xử lý; (2) nguy hiểm hơn — nếu code sửa đổi object A (một trong số nhiều
object đại diện cùng một dòng dữ liệu), object B (đại diện cùng dòng đó nhưng là instance khác)
sẽ **không** phản ánh thay đổi đó, dẫn tới trạng thái không nhất quán trong bộ nhớ dù cùng đại
diện một dòng database.

## Concept

**Persistence Context** (còn gọi là **first-level cache** — bộ nhớ đệm cấp một) là một vùng nhớ
gắn liền với vòng đời của một `EntityManager` (thường tương ứng một transaction) — nó **theo
dõi** mọi entity đã được load hoặc lưu trong phạm vi đó, đảm bảo với cùng một khoá chính (ID),
**luôn** trả về **đúng một** object Java duy nhất (identity map — "bản đồ đồng nhất") thay vì
tạo instance mới mỗi lần truy vấn.

## Why?

Đảm bảo "cùng một hàng dữ liệu = cùng một object Java" trong phạm vi một transaction giải quyết
trực tiếp vấn đề đã nêu: không cần truy vấn lại database cho dữ liệu đã load (tối ưu hiệu năng),
và mọi thay đổi trên entity đó — bất kể thực hiện thông qua tham chiếu nào lấy được trong cùng
transaction — đều nhất quán, vì thực chất chỉ có **một** object duy nhất tồn tại trong bộ nhớ.
Đây cũng chính là nền tảng cho phép Dirty Checking (Chapter 06) hoạt động: Hibernate chỉ cần so
sánh trạng thái hiện tại của entity trong Persistence Context với "ảnh chụp" (snapshot) lúc mới
load, để biết cần `UPDATE` gì khi transaction kết thúc.

## How?

```java
@PersistenceContext
EntityManager em; // moi transaction gan voi MOT Persistence Context rieng

Customer first = em.find(Customer.class, id);  // SELECT that, luu vao Persistence Context
Customer second = em.find(Customer.class, id); // KHONG SELECT lai, tra ve TU Persistence Context
// first == second luon la true, trong CUNG mot transaction
```

## Visualization

```
Transaction 1 (mot Persistence Context):

  em.find(Customer.class, 1) ──► CHUA co trong PC ──► SELECT that ──► luu vao PC ──► tra ve object A
  em.find(Customer.class, 1) ──► DA co trong PC (id=1) ──► KHONG SELECT ──► tra ve CHINH object A

Transaction 2 (Persistence Context KHAC, moi):

  em.find(Customer.class, 1) ──► PC nay CHUA co gi (moi, rieng biet voi Transaction 1)
                                ──► SELECT that ──► luu vao PC nay ──► tra ve object B (KHAC object A)
```

## Example

```java
TransactionStatus tx = txManager.getTransaction(new DefaultTransactionDefinition());
Customer first = em.find(Customer.class, id);
Customer second = em.find(Customer.class, id);
System.out.println("first == second: " + (first == second));
txManager.commit(tx);
```

Kết quả thật (Hibernate ORM 6.5.3.Final):

```
=== TRONG CUNG mot transaction/persistence context ===
Hibernate: select c1_0.id,c1_0.name from customer c1_0 where c1_0.id=?
Lan 1: em.find() -> co SELECT that
Lan 2: em.find() -> 
first == second (CUNG mot object trong bo nho): true
```

**So sánh với hai transaction khác nhau:**

```java
TransactionStatus tx1 = txManager.getTransaction(new DefaultTransactionDefinition());
Customer inTx1 = em.find(Customer.class, id);
txManager.commit(tx1);

TransactionStatus tx2 = txManager.getTransaction(new DefaultTransactionDefinition());
Customer inTx2 = em.find(Customer.class, id);
txManager.commit(tx2);
System.out.println("inTx1 == inTx2: " + (inTx1 == inTx2));
```

Kết quả thật:

```
=== O HAI transaction/persistence context KHAC NHAU ===
Hibernate: select c1_0.id,c1_0.name from customer c1_0 where c1_0.id=?
Hibernate: select c1_0.id,c1_0.name from customer c1_0 where c1_0.id=?
inTx1 == inTx2 (KHAC transaction, KHAC persistence context): false
```

Trong **cùng** một transaction, gọi `find()` hai lần chỉ sinh **một** câu `SELECT`, và trả về
đúng cùng một object (`first == second`). Ở **hai transaction khác nhau**, mỗi transaction có
Persistence Context riêng — mỗi cái đều phải `SELECT` lại từ đầu (2 câu SQL cho 2 transaction),
và `inTx1 == inTx2` là `false` dù cả hai đại diện cùng một dòng dữ liệu (cùng `id`) — vì chúng
là hai object Java khác nhau, thuộc hai Persistence Context tách biệt.

## Deep Dive

**Persistence Context "biến mất" khi nào, và điều gì xảy ra với các entity nó đang theo dõi?**
Khi transaction kết thúc (commit hoặc rollback) — hoặc khi `EntityManager` bị đóng — Persistence
Context gắn với nó cũng kết thúc theo. Mọi entity đang được theo dõi trong đó chuyển sang trạng
thái **Detached** (Chapter 07) — vẫn là object Java hợp lệ, vẫn giữ dữ liệu, nhưng **không còn**
được Hibernate theo dõi thay đổi nữa (Dirty Checking, Chapter 06, ngừng hoạt động với nó). Đây
là lý do một entity lấy ra từ một request HTTP trước đó, giữ lại và dùng ở một request sau (hoặc
một thread khác), dù vẫn "trông có vẻ hợp lệ" về mặt Java, không còn nằm trong Persistence
Context nào — sửa đổi trên nó sẽ **không** tự động được lưu xuống database.

## Engineering Insight

**Vì sao Persistence Context "identity map" (cùng ID → cùng object) chỉ hoạt động trong phạm vi
MỘT transaction, không mở rộng ra toàn ứng dụng?** Nếu Persistence Context là **dùng chung toàn
ứng dụng** (một cache lớn cho mọi request), sẽ nảy sinh vấn đề nghiêm trọng: hai transaction chạy
song song (Phase 3) cùng sửa một entity sẽ **thấy thay đổi của nhau ngay lập tức**, dù mỗi
transaction lẽ ra phải "cô lập" (isolation, đặc tính ACID) khỏi những transaction khác chưa
commit — vi phạm trực tiếp nguyên lý isolation, và một transaction bị rollback có thể để lại
"dấu vết" thay đổi tạm thời trong bộ nhớ dùng chung, ảnh hưởng tới các transaction khác đang đọc
cùng entity. Việc giới hạn Persistence Context trong phạm vi một transaction (short-lived, gắn
với `EntityManager` cụ thể) là quyết định thiết kế **bắt buộc** để giữ đúng tính đúng đắn của mô
hình transaction — đây khác biệt căn bản với **second-level cache** (Hibernate hỗ trợ riêng,
dùng chung toàn ứng dụng, cần cơ chế đồng bộ hoá phức tạp hơn nhiều để tránh đúng vấn đề vừa nêu).

## Historical Note

Khái niệm "identity map" trong Persistence Context không phải phát minh riêng của Hibernate —
đây là một mẫu hình thiết kế (design pattern) kinh điển được Martin Fowler mô tả trong "Patterns
of Enterprise Application Architecture" (2002), áp dụng rộng rãi trong hầu hết ORM framework
trên nhiều ngôn ngữ (không chỉ Java) để đảm bảo tính nhất quán khi nhiều phần code cùng truy cập
một bản ghi dữ liệu trong cùng một đơn vị công việc (unit of work).

## Myth vs Reality

- **Myth:** "`em.find()` luôn truy vấn database mỗi lần được gọi."
  **Reality:** Đã chứng minh bằng thực nghiệm — trong cùng Persistence Context, lần gọi thứ hai
  trở đi với cùng ID hoàn toàn không sinh SQL, trả về từ bộ nhớ.

- **Myth:** "Hai entity có cùng ID, lấy ra ở hai thời điểm khác nhau trong ứng dụng, luôn là
  cùng một object Java."
  **Reality:** Chỉ đúng khi cùng một Persistence Context (cùng transaction) — đã chứng minh
  ngược lại khi khác transaction.

## Common Mistakes

- **Giữ lại entity sau khi transaction đã kết thúc, kỳ vọng sửa đổi tự động lưu xuống database**
  — entity đã Detached (xem Deep Dive), cần gọi `merge()` hoặc mở transaction mới để cập nhật.
- **Ngộ nhận `==` giữa hai entity ở hai transaction khác nhau, dùng nó để so sánh "cùng bản ghi
  hay không"** — nên dùng `equals()` dựa trên ID (thường override tường minh) thay vì `==` khi
  so sánh entity giữa các transaction khác nhau.
- **Load một lượng lớn entity trong một transaction dài, không cần dùng tới hầu hết chúng nữa**
  — Persistence Context giữ tham chiếu tất cả entity đã load, có thể gây tốn bộ nhớ nếu
  transaction xử lý khối lượng dữ liệu lớn không cần thiết.

## Debug Checklist

- [ ] Sửa entity nhưng thay đổi không được lưu xuống database? → kiểm tra entity có đang ở
      trạng thái Detached không (transaction đã kết thúc, Chapter 07).
- [ ] Hai entity cùng ID nhưng `==` trả về `false`? → bình thường nếu chúng thuộc hai
      Persistence Context/transaction khác nhau — dùng `equals()` dựa trên ID để so sánh đúng
      ngữ nghĩa.
- [ ] Nghi ngờ có quá nhiều entity bị giữ trong bộ nhớ suốt một transaction dài? → cân nhắc gọi
      `em.clear()` định kỳ để giải phóng Persistence Context (cẩn thận: mất luôn khả năng dirty
      checking cho các entity đã clear).

## Summary

Persistence Context (first-level cache) là vùng nhớ gắn liền với vòng đời một
`EntityManager`/transaction, đảm bảo "cùng ID → cùng object Java" (identity map) trong phạm vi
đó. Đã chứng minh bằng thực nghiệm: gọi `find()` hai lần cùng ID trong cùng transaction chỉ sinh
một `SELECT`, trả về cùng object reference (`first == second` là `true`); ở hai transaction khác
nhau, mỗi transaction có Persistence Context riêng, mỗi cái tự `SELECT` lại và tạo object khác
nhau (`inTx1 == inTx2` là `false`). Khi transaction kết thúc, Persistence Context biến mất, mọi
entity chuyển sang trạng thái Detached (Chapter 07) — không còn được theo dõi thay đổi tự động.
Giới hạn phạm vi một transaction là quyết định thiết kế bắt buộc để giữ đúng tính isolation.

## Interview Questions

**Mid**

- Persistence Context là gì?
- Vì sao gọi `EntityManager.find()` hai lần với cùng ID trong cùng transaction chỉ sinh ra một
  câu SQL?

**Senior**

- Vì sao Persistence Context chỉ hoạt động trong phạm vi một transaction, không mở rộng ra toàn
  ứng dụng? Điều gì sẽ xảy ra nếu nó dùng chung?
- Persistence Context khác second-level cache của Hibernate như thế nào?

## Exercises

- [ ] Chạy lại `PersistenceContextTest` ở trên, xác nhận số câu SQL và kết quả so sánh reference
      đúng như mô tả.
- [ ] Gọi `em.clear()` giữa hai lần `find()` trong cùng transaction, quan sát câu `SELECT` thứ
      hai xuất hiện trở lại (Persistence Context đã bị xoá).
- [ ] Tìm hiểu second-level cache của Hibernate (`@Cacheable`, cấu hình region), so sánh phạm vi
      hoạt động với first-level cache đã học ở chapter này.

## Cheat Sheet

| | Cùng transaction | Khác transaction |
| --- | --- | --- |
| `find()` cùng ID lần 2 | Không SELECT lại | SELECT lại (Persistence Context riêng) |
| `==` giữa hai lần lấy | `true` | `false` |
| Phạm vi | Một `EntityManager`/transaction | N/A — mỗi transaction độc lập |

## References

- Jakarta Persistence Specification — Persistence Context.
- Martin Fowler — "Patterns of Enterprise Application Architecture", Identity Map pattern (2002).
- Hibernate ORM Documentation — Persistence Context.
