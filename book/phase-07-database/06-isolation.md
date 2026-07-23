---
tags:
  - SQL
  - Isolation
  - PostgreSQL
---

# Isolation Levels

> Phase: Phase 7 — Database
> Chapter slug: `isolation`

## Metadata

```yaml
Chapter: Isolation
Phase: Phase 7 — Database
Difficulty: ★★★★★
Importance: ★★★★★
Interview Frequency: 85%
Prerequisites:
  - Chapter 05 — Transaction (Database ACID Transaction)
  - Phase 6, Chapter 13 — Optimistic Lock
Used Later:
  - Chapter 07 — Lock
  - Chapter 08 — MVCC
Estimated Reading: 20 phút
Estimated Practice: 18 phút
```

## Story

> Xem lại: [Chapter 05](05-transaction.md) — chữ "I" trong ACID là Isolation, nhưng "cô lập ở
> mức nào" hoá ra có **nhiều mức độ khác nhau**, mỗi mức cho phép hoặc ngăn chặn những "hiện
> tượng bất thường" (anomaly) khác nhau khi nhiều transaction chạy đồng thời.

Ở mức **READ COMMITTED** (mặc định của PostgreSQL), đọc cùng một dòng **hai lần** trong cùng
một transaction, có một transaction khác `UPDATE` + `COMMIT` ở giữa:

```
BEGIN
-- LAN DOC 1
   city
-------------------
 DIRTY_UNCOMMITTED
-- (giao dich khac UPDATE + COMMIT o day)
-- LAN DOC 2, CUNG transaction
   city
--------------
 HCMC_CHANGED
```

**Hai lần đọc trong cùng một transaction cho ra hai giá trị khác nhau.** Đổi sang
**REPEATABLE READ**, lặp lại chính xác kịch bản đó:

```
-- LAN DOC 1
   city
----------
 BASELINE
-- (giao dich khac UPDATE + COMMIT o day)
-- LAN DOC 2, CUNG transaction
   city
----------
 BASELINE
```

Cùng kịch bản, cùng thao tác của transaction khác — nhưng lần này, **cả hai lần đọc đều cho
cùng một giá trị**.

## Interview Question (Central)

> Có bao nhiêu mức Isolation Level trong SQL chuẩn? Phân biệt Dirty Read, Non-repeatable Read,
> và Phantom Read — mỗi mức isolation ngăn chặn được hiện tượng nào?

## Objectives

- [ ] Phân biệt 4 mức isolation chuẩn: `READ UNCOMMITTED`, `READ COMMITTED`,
      `REPEATABLE READ`, `SERIALIZABLE`
- [ ] Tự tay chứng minh bằng thực nghiệm cả ba hiện tượng: Dirty Read (PostgreSQL không bao giờ
      cho phép), Non-repeatable Read (xảy ra ở READ COMMITTED, bị chặn ở REPEATABLE READ),
      Phantom Read (bị chặn ở REPEATABLE READ của PostgreSQL — mạnh hơn chuẩn SQL)
- [ ] Tự tay tái hiện một xung đột `SERIALIZABLE` thật, nhận lỗi serialization failure

## Prerequisites

- Chapter 05 — hiểu ACID, đặc biệt tính chất Isolation là tiền đề của chapter này.
- Phase 6, Chapter 13 — đã thấy hệ quả cụ thể của xung đột giao dịch đồng thời qua Optimistic
  Lock, chapter này giải thích cơ chế tổng quát hơn đứng sau nó.

## Used Later

- **Chapter 07 (Lock)**, **Chapter 08 (MVCC)** — cả hai đều là cơ chế cụ thể PostgreSQL dùng để
  hiện thực hoá các mức isolation đã học ở chapter này.

## Problem

"Cô lập hoàn toàn" (mỗi transaction chạy như thể nó là transaction duy nhất trên hệ thống) là lý
tưởng nhưng **cực kỳ đắt đỏ** về hiệu năng — cần khoá gần như mọi thứ, ngăn hoàn toàn khả năng
chạy song song. Ngược lại, "không cô lập gì cả" (mọi transaction thấy ngay thay đổi chưa commit
của nhau) nhanh nhưng dẫn tới dữ liệu sai lệch nghiêm trọng. SQL chuẩn định nghĩa **nhiều mức
trung gian**, mỗi mức đánh đổi khác nhau giữa hiệu năng và mức độ "nhìn thấy" thay đổi của các
transaction khác.

## Concept

**Isolation Level** quy định một transaction "nhìn thấy" thay đổi của các transaction đồng thời
khác ở mức độ nào, được định nghĩa qua việc **ngăn chặn hay cho phép** ba hiện tượng bất thường:
**Dirty Read** (đọc dữ liệu **chưa commit** của transaction khác), **Non-repeatable Read** (đọc
cùng một dòng hai lần trong cùng transaction, nhận hai giá trị khác nhau vì transaction khác đã
`UPDATE` + commit ở giữa), **Phantom Read** (chạy cùng một truy vấn tổng hợp/đếm hai lần, nhận
số lượng dòng khác nhau vì transaction khác đã `INSERT` mới ở giữa). Bốn mức chuẩn SQL, theo thứ
tự cô lập tăng dần: `READ UNCOMMITTED` → `READ COMMITTED` → `REPEATABLE READ` → `SERIALIZABLE`.

## Why?

Việc chia nhỏ thành nhiều mức isolation cho phép lập trình viên **lựa chọn đánh đổi phù hợp** với
từng nhu cầu cụ thể, thay vì buộc phải chọn giữa hai cực đoan (không cô lập gì / cô lập tuyệt
đối, tốn hiệu năng nhất). Với nghiệp vụ chỉ cần đọc dữ liệu "gần đúng thời điểm hiện tại", READ
COMMITTED (mặc định, rẻ nhất) là đủ. Với nghiệp vụ cần **tính toán nhất quán** dựa trên nhiều lần
đọc trong cùng giao dịch (ví dụ tính tổng rồi kiểm tra điều kiện dựa trên tổng đó), REPEATABLE
READ đảm bảo không có dữ liệu "thay đổi giữa chừng" làm sai kết quả. SERIALIZABLE, mức mạnh nhất,
dành cho các trường hợp cần đảm bảo tuyệt đối không có bất kỳ hiện tượng bất thường nào, chấp
nhận đổi lấy khả năng bị **từ chối** (phải retry) khi có xung đột thực sự.

## How?

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;   -- MAC DINH cua PostgreSQL
BEGIN ISOLATION LEVEL REPEATABLE READ;  -- ngan non-repeatable read (VA phantom read tren PG)
BEGIN ISOLATION LEVEL SERIALIZABLE;     -- manh nhat, phat hien xung dot, co the tu choi commit
```

## Visualization

```
READ COMMITTED (mac dinh PostgreSQL):

  Transaction A:  doc lan 1 ──► [B update+commit o giua] ──► doc lan 2
                     "cu"                                      "MOI" (thay doi!)
  => Non-repeatable Read XAY RA

REPEATABLE READ:

  Transaction A:  doc lan 1 ──► [B update+commit o giua] ──► doc lan 2
                     "cu"                                      "cu" (KHONG doi!)
  => Non-repeatable Read BI NGAN (A "dong bang" tai thoi diem BEGIN)

SERIALIZABLE:

  Transaction A, B: ca hai CUNG doc, CUNG INSERT du lieu lien quan
       │
       A commit TRUOC ──► THANH CONG
       B commit SAU ──► PHAT HIEN xung dot doc/ghi ──► TU CHOI (serialization failure)
```

## Example

**Dirty Read — PostgreSQL không bao giờ cho phép (kể cả READ COMMITTED):**

```sql
-- Session B: UPDATE nhung CHUA COMMIT
BEGIN;
UPDATE customers SET city = 'DIRTY_UNCOMMITTED' WHERE name = 'Nguyen Van A';

-- Session A (READ COMMITTED mac dinh): doc CUNG luc
SELECT city FROM customers WHERE name = 'Nguyen Van A';
```

Kết quả thật:

```
 city
-------
 Hanoi
```

Session A **không hề** thấy `'DIRTY_UNCOMMITTED'` — vẫn thấy giá trị đã commit trước đó
(`'Hanoi'`), dù Session B đã `UPDATE` (chưa commit). Đây là bằng chứng: PostgreSQL **không bao
giờ** cho phép Dirty Read, kể cả ở mức thấp nhất.

**Non-repeatable Read — xảy ra ở READ COMMITTED, bị chặn ở REPEATABLE READ:**

```sql
-- Session A (READ COMMITTED)
BEGIN;
SELECT city FROM customers WHERE name = 'Nguyen Van A'; -- lan 1
-- (Session B: UPDATE + COMMIT o giua)
SELECT city FROM customers WHERE name = 'Nguyen Van A'; -- lan 2, CUNG transaction A
COMMIT;
```

Kết quả thật: lần 1 `'DIRTY_UNCOMMITTED'`, lần 2 `'HCMC_CHANGED'` — **khác nhau** trong cùng một
transaction.

```sql
-- Session A (REPEATABLE READ) - LAP LAI dung kich ban do
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT city FROM customers WHERE name = 'Nguyen Van A'; -- lan 1: BASELINE
-- (Session B: UPDATE + COMMIT o giua)
SELECT city FROM customers WHERE name = 'Nguyen Van A'; -- lan 2: VAN LA BASELINE
COMMIT;
```

**Phantom Read — bị chặn ở REPEATABLE READ của PostgreSQL:**

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM customers WHERE city = 'Hanoi'; -- lan 1: 1
-- (Session B: INSERT customer moi voi city='Hanoi' + COMMIT o giua)
SELECT count(*) FROM customers WHERE city = 'Hanoi'; -- lan 2: VAN LA 1
COMMIT;
```

**SERIALIZABLE — phát hiện xung đột, từ chối một transaction:**

```sql
-- Ca A va B cung BEGIN ISOLATION LEVEL SERIALIZABLE
-- Ca hai cung SELECT count(*) WHERE city='Da Nang', roi cung INSERT them 1 dong voi city='Da Nang'
-- A commit TRUOC:
COMMIT; -- THANH CONG

-- B commit SAU:
COMMIT;
```

Kết quả thật cho Session B:

```
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
HINT:  The transaction might succeed if retried.
```

## Deep Dive

**Vì sao `REPEATABLE READ` của PostgreSQL ngăn được **cả** Phantom Read, trong khi chuẩn SQL chỉ
yêu cầu nó ngăn Non-repeatable Read?** Chuẩn SQL định nghĩa các mức isolation dựa trên hiện
tượng **tối thiểu** cần ngăn chặn — `REPEATABLE READ` theo chuẩn chỉ **bắt buộc** ngăn
Non-repeatable Read, để ngỏ khả năng Phantom Read vẫn xảy ra. Nhưng PostgreSQL hiện thực hoá
`REPEATABLE READ` bằng cơ chế **Snapshot Isolation** (dựa trên MVCC, Chapter 08) — toàn bộ
transaction nhìn thấy một "ảnh chụp" (snapshot) nhất quán của dữ liệu tại đúng thời điểm `BEGIN`,
không chỉ riêng từng dòng đã đọc — cơ chế này **tự động** ngăn luôn cả Phantom Read như một hệ
quả tất yếu (không phải nỗ lực bổ sung riêng). Đây là lý do PostgreSQL's `REPEATABLE READ` mạnh
hơn yêu cầu tối thiểu của chuẩn SQL — một chi tiết implementation-specific quan trọng cần biết
khi làm việc với các database khác (một số database khác, ví dụ MySQL/InnoDB, cũng dùng cơ chế
tương tự, nhưng không phải mọi database đều đảm bảo điều này ở cùng mức isolation).

## Engineering Insight

**"Serialization failure" ở mức `SERIALIZABLE` không phải lỗi — nó là hành vi **thiết kế đúng
đắn**. Vì sao, và ứng dụng cần xử lý nó như thế nào?** SERIALIZABLE đảm bảo kết quả cuối cùng
**tương đương** với việc chạy các transaction đó **tuần tự** (lần lượt, không chồng chéo) — nếu
hai transaction chạy đồng thời có khả năng dẫn tới kết quả **khác** so với chạy tuần tự theo bất
kỳ thứ tự nào, PostgreSQL **buộc phải** huỷ một trong hai để bảo toàn đúng đảm bảo đó (đã chứng
minh bằng thực nghiệm: cả A và B cùng đếm rồi cùng thêm dòng mới — nếu cả hai cùng thành công,
kết quả sẽ khác với chạy tuần tự đúng đắn). Ứng dụng dùng `SERIALIZABLE` **bắt buộc** phải có
logic **retry** khi gặp lỗi `40001` (serialization_failure) — đây không phải một trường hợp
ngoại lệ hiếm gặp cần lo lắng, mà là một phần **thiết kế bình thường** của việc dùng mức
isolation này, giống hệt cách Optimistic Lock (Phase 6, Chapter 13) cần chiến lược retry cho
`OptimisticLockException`.

## Historical Note

Bốn mức isolation chuẩn được định nghĩa chính thức trong SQL-92 (1992), dựa trên nghiên cứu lý
thuyết về concurrency control từ những năm 1970-1980. Serializable Snapshot Isolation (SSI) —
thuật toán cụ thể PostgreSQL dùng để hiện thực hoá mức `SERIALIZABLE` hiệu quả (tránh phải khoá
nặng nề như các cách tiếp cận truyền thống) — chỉ được thêm vào PostgreSQL từ phiên bản 9.1
(2011), dựa trên nghiên cứu học thuật của Michael Cahill (2008) — một minh chứng cho việc lý
thuyết database vẫn tiếp tục phát triển và được áp dụng vào các hệ thống production hiện đại.

## Myth vs Reality

- **Myth:** "Dirty Read có thể xảy ra ở mức `READ COMMITTED` vì tên gọi 'committed' chỉ áp dụng
  cho một số trường hợp."
  **Reality:** Đã chứng minh bằng thực nghiệm — PostgreSQL **không bao giờ** cho phép Dirty
  Read, kể cả ở `READ UNCOMMITTED` (PostgreSQL coi `READ UNCOMMITTED` tương đương
  `READ COMMITTED` trong thực thi, dù chấp nhận cú pháp khai báo mức đó).

- **Myth:** "`REPEATABLE READ` chỉ ngăn Non-repeatable Read, Phantom Read vẫn có thể xảy ra bất
  kể database nào."
  **Reality:** Đã chứng minh bằng thực nghiệm ngược lại trên PostgreSQL — Snapshot Isolation
  ngăn cả hai hiện tượng cùng lúc, mạnh hơn yêu cầu tối thiểu của chuẩn SQL.

## Common Mistakes

- **Dùng `SERIALIZABLE` cho mọi transaction "để an toàn tuyệt đối" mà không có chiến lược
  retry** — ứng dụng sẽ crash/lỗi thường xuyên khi gặp serialization failure thay vì tự phục
  hồi.
- **Nhầm lẫn READ COMMITTED của PostgreSQL với hành vi cho phép Dirty Read** — đã chứng minh sai,
  PostgreSQL không bao giờ cho Dirty Read dù ở mức nào.
- **Không nhận ra Non-repeatable Read đang ảnh hưởng logic nghiệp vụ** (ví dụ tính tổng hai lần,
  nhận hai kết quả khác nhau trong cùng transaction) — cần REPEATABLE READ hoặc thiết kế lại
  logic chỉ đọc một lần.

## Best Practices

- Mặc định dùng `READ COMMITTED` cho phần lớn nghiệp vụ thông thường — cân bằng tốt giữa hiệu
  năng và mức độ nhất quán.
- Dùng `REPEATABLE READ` khi logic nghiệp vụ cần đọc dữ liệu nhất quán nhiều lần trong cùng một
  transaction (ví dụ tính toán phức tạp dựa trên nhiều truy vấn).
- Chỉ dùng `SERIALIZABLE` khi thực sự cần đảm bảo tuyệt đối, và **luôn** có chiến lược retry cho
  lỗi `40001`.
- Không nhầm lẫn isolation level với locking (Chapter 07) — isolation level là **chính sách**,
  cơ chế bên dưới (MVCC, lock) là **cách hiện thực hoá** chính sách đó (Chapter 08).

## Debug Checklist

- [ ] Dữ liệu đọc hai lần trong cùng transaction cho kết quả khác nhau ngoài dự kiến? → kiểm tra
      isolation level đang dùng, cân nhắc `REPEATABLE READ` nếu cần nhất quán trong suốt
      transaction.
- [ ] Gặp lỗi `could not serialize access due to read/write dependencies`? → bình thường ở mức
      `SERIALIZABLE`, cần logic retry, không phải lỗi cần "sửa" theo nghĩa thông thường.
- [ ] Không chắc một database khác (MySQL, Oracle) có cùng đảm bảo Phantom Read ở `REPEATABLE
      READ` như PostgreSQL không? → kiểm tra tài liệu cụ thể của database đó, đây là chi tiết
      implementation-specific, không phải yêu cầu chuẩn SQL.

## Summary

Isolation Level quy định mức "nhìn thấy" thay đổi của transaction khác, định nghĩa qua việc ngăn
chặn Dirty Read, Non-repeatable Read, Phantom Read. Đã chứng minh bằng thực nghiệm trên
PostgreSQL: Dirty Read không bao giờ xảy ra (kể cả READ COMMITTED); Non-repeatable Read xảy ra ở
READ COMMITTED (hai lần đọc cùng transaction cho hai giá trị khác nhau) nhưng bị chặn hoàn toàn
ở REPEATABLE READ; Phantom Read cũng bị chặn ở REPEATABLE READ của PostgreSQL (mạnh hơn yêu cầu
tối thiểu của chuẩn SQL, nhờ cơ chế Snapshot Isolation dựa trên MVCC — Chapter 08); SERIALIZABLE
phát hiện xung đột đọc/ghi thực sự và chủ động từ chối một transaction với lỗi
`could not serialize access` — một hành vi thiết kế đúng đắn, không phải lỗi, đòi hỏi chiến lược
retry tương tự Optimistic Lock (Phase 6, Chapter 13).

## Interview Questions

- Có bao nhiêu mức Isolation Level trong SQL chuẩn? Phân biệt Dirty Read, Non-repeatable Read,
  và Phantom Read.

**Senior**

- Vì sao `REPEATABLE READ` của PostgreSQL ngăn được cả Phantom Read, trong khi chuẩn SQL chỉ
  yêu cầu nó ngăn Non-repeatable Read?
- Giải thích vì sao "serialization failure" ở mức `SERIALIZABLE` là hành vi thiết kế đúng đắn,
  không phải lỗi. Ứng dụng cần xử lý nó như thế nào?

## Exercises

- [ ] Chạy lại toàn bộ 4 kịch bản ở Example trên một PostgreSQL thật (dùng 2 session song song),
      xác nhận kết quả đúng như mô tả.
- [ ] Viết một transaction `SERIALIZABLE` với logic retry (bắt lỗi `40001`, thử lại tối đa 3
      lần), xác nhận cuối cùng thành công sau một vài lần thử.
- [ ] Tìm hiểu tài liệu isolation level của MySQL/InnoDB, so sánh hành vi Phantom Read ở mức
      `REPEATABLE READ` với PostgreSQL.

## Cheat Sheet

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read |
| --- | --- | --- | --- |
| `READ UNCOMMITTED` | Cho phép (theo chuẩn; PostgreSQL thực tế vẫn chặn) | Cho phép | Cho phép |
| `READ COMMITTED` (mặc định PG) | Chặn | Cho phép | Cho phép |
| `REPEATABLE READ` | Chặn | Chặn | Chặn (trên PostgreSQL, mạnh hơn chuẩn) |
| `SERIALIZABLE` | Chặn | Chặn | Chặn (đảm bảo tương đương chạy tuần tự) |

## References

- PostgreSQL Documentation — Transaction Isolation.
- Cahill, M.; Röhm, U.; Fekete, A. — "Serializable Isolation for Snapshot Databases" (2008,
  nền tảng SSI của PostgreSQL).
- ISO/IEC 9075 (SQL-92) — Transaction Isolation Levels.
