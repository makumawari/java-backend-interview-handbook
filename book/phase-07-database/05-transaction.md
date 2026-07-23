---
tags:
  - SQL
  - Transaction
  - ACID
  - PostgreSQL
---

# Transaction (Database ACID Transaction)

> Phase: Phase 7 — Database
> Chapter slug: `transaction`

## Metadata

```yaml
Chapter: Transaction (Database ACID Transaction)
Phase: Phase 7 — Database
Difficulty: ★★★
Importance: ★★★★★
Interview Frequency: 90%
Prerequisites:
  - Chapter 01 — SQL
  - Phase 6, Chapter 12 — Transaction (JPA Transaction Boundary)
Used Later:
  - Chapter 06 — Isolation
  - Chapter 07 — Lock
  - Chapter 08 — MVCC
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: Phase 6, Chapter 12 — flush/commit ở tầng JPA cuối cùng đều "nói chuyện" với một
> **transaction database thật**. Đây chính là tầng nền tảng nhất: `BEGIN`/`COMMIT`/`ROLLBACK`
> ở SQL thuần, không qua bất kỳ lớp trừu tượng nào.

Cố tình chèn hai khách hàng trong cùng một transaction, người thứ hai vi phạm ràng buộc `UNIQUE`
trên email:

```sql
BEGIN;
INSERT INTO customers (name, email, city) VALUES ('Test Atomicity', 'atomic@example.com', 'Hue');
INSERT INTO customers (name, email, city) VALUES ('Duplicate Email', 'atomic@example.com', 'Hue');
COMMIT;
```

Kết quả thật:

```
BEGIN
INSERT 0 1
ERROR:  duplicate key value violates unique constraint "customers_email_key"
```

Kiểm tra lại — dòng **đầu tiên** (đã `INSERT` thành công trước khi câu thứ hai lỗi):

```sql
SELECT * FROM customers WHERE name = 'Test Atomicity';
```

```
 id | name | email | city
----+------+-------+------
(0 rows)
```

Dòng đầu tiên **cũng biến mất** — dù chính nó không hề vi phạm ràng buộc nào. Cả transaction bị
huỷ **toàn bộ**, không phải chỉ câu lệnh lỗi.

## Interview Question (Central)

> ACID là gì? Giải thích từng chữ cái, và chứng minh bằng ví dụ cụ thể tính chất Atomicity hoạt
> động như thế nào.

## Objectives

- [ ] Giải thích chính xác 4 tính chất ACID: Atomicity, Consistency, Isolation, Durability
- [ ] Tự tay chứng minh bằng thực nghiệm: một câu lệnh lỗi trong transaction khiến **toàn bộ**
      transaction bị huỷ, kể cả các câu lệnh đã thành công trước đó
- [ ] Tự tay chứng minh `ROLLBACK` tường minh hoàn tác hoàn toàn thay đổi trong transaction

## Prerequisites

- Chapter 01 — hiểu SQL cơ bản.
- Phase 6, Chapter 12 — đã học flush/commit ở tầng JPA, chapter này đi xuống tầng database thuần
  bên dưới lớp trừu tượng đó.

## Used Later

- **Chapter 06 (Isolation)**, **Chapter 07 (Lock)**, **Chapter 08 (MVCC)** — cả ba đều là cơ chế
  cụ thể hiện thực hoá tính chất **Isolation** của ACID.

## Problem

Một thao tác nghiệp vụ thường cần **nhiều** câu lệnh SQL liên quan tới nhau (trừ tiền tài khoản
A, cộng tiền tài khoản B — giống ví dụ đã gặp ở Phase 5, Chapter 13) — nếu không có cơ chế đảm
bảo "tất cả cùng thành công hoặc tất cả cùng thất bại", một lỗi giữa chừng (mất kết nối, vi phạm
ràng buộc, crash hệ thống) có thể để lại dữ liệu ở trạng thái **nửa vời**, không nhất quán.

## Concept

**ACID** là bốn tính chất một transaction database phải đảm bảo: **Atomicity** (tính nguyên tử
— mọi câu lệnh trong transaction cùng thành công hoặc cùng thất bại, không có trạng thái "nửa
vời"), **Consistency** (tính nhất quán — transaction luôn đưa database từ một trạng thái hợp lệ
sang một trạng thái hợp lệ khác, không vi phạm ràng buộc/constraint), **Isolation** (tính cô lập
— các transaction chạy đồng thời không ảnh hưởng lẫn nhau theo cách không mong muốn, chi tiết ở
Chapter 06-08), **Durability** (tính bền vững — một khi đã `COMMIT`, thay đổi tồn tại vĩnh viễn,
sống sót qua crash hệ thống).

## Why?

Bốn tính chất này giải quyết đúng bốn khía cạnh khác nhau của vấn đề "dữ liệu nửa vời": Atomicity
đảm bảo không có kết quả "một phần" của một đơn vị công việc logic; Consistency đảm bảo mọi
ràng buộc nghiệp vụ (khoá ngoại, unique, check constraint) luôn được tôn trọng dù có bao nhiêu
bước trung gian; Isolation đảm bảo nhiều giao dịch đồng thời không "giẫm chân" lên nhau (đã thấy
hệ quả cụ thể ở Phase 6, Chapter 13 — Optimistic Lock); Durability đảm bảo một khi hệ thống đã
xác nhận "thành công" với người dùng, thông tin đó không thể biến mất bất ngờ (kể cả khi mất
điện ngay sau đó).

## How?

```sql
BEGIN; -- bat dau transaction

INSERT INTO customers (...) VALUES (...);
INSERT INTO orders (...) VALUES (...);
-- neu CA HAI thanh cong:
COMMIT; -- xac nhan VINH VIEN

-- neu CO LOI, hoac muon huy bo:
ROLLBACK; -- HOAN TAC TOAN BO, ve dung trang thai truoc BEGIN
```

## Visualization

```
Atomicity - mot cau loi, CA transaction bi huy:

  BEGIN
    INSERT 'Test Atomicity'  ──► THANH CONG (nhung CHUA vinh vien)
    INSERT 'Duplicate Email' ──► LOI (vi pham UNIQUE)
  COMMIT (that ra khong con y nghia, transaction DA BI HUY tu luc loi)
       │
       ▼
  Ca hai INSERT deu KHONG con ton tai - "Test Atomicity" cung bien mat
  (du chinh no khong loi gi ca)

ROLLBACK tuong minh:

  BEGIN
    INSERT 'Se Bi Huy'  ──► trong TRANSACTION, co the THAY duoc (count=1)
  ROLLBACK
       │
       ▼
  Tu MOT session/query MOI: 'Se Bi Huy' KHONG TON TAI (count=0)
```

## Example

**Atomicity — lỗi giữa chừng huỷ toàn bộ:**

```sql
BEGIN;
INSERT INTO customers (name, email, city) VALUES ('Test Atomicity', 'atomic@example.com', 'Hue');
INSERT INTO customers (name, email, city) VALUES ('Duplicate Email', 'atomic@example.com', 'Hue');
COMMIT;
```

Kết quả thật (PostgreSQL 16.14):

```
BEGIN
INSERT 0 1
ERROR:  duplicate key value violates unique constraint "customers_email_key"
DETAIL:  Key (email)=(atomic@example.com) already exists.
```

```sql
SELECT * FROM customers WHERE name = 'Test Atomicity';
```

```
 id | name | email | city
----+------+-------+------
(0 rows)
```

**`ROLLBACK` tường minh:**

```sql
BEGIN;
INSERT INTO customers (name, email, city) VALUES ('Se Bi Huy', 'huy@example.com', 'Hue');
SELECT count(*) AS trong_transaction FROM customers WHERE name = 'Se Bi Huy';
ROLLBACK;
```

```
BEGIN
INSERT 0 1
 trong_transaction
-------------------
                 1
ROLLBACK
```

Kiểm tra lại từ một session/kết nối mới:

```sql
SELECT count(*) AS sau_rollback FROM customers WHERE name = 'Se Bi Huy';
```

```
 sau_rollback
--------------
            0
```

Dòng đầu tiên `INSERT` thành công (`INSERT 0 1`) — nhưng khi câu lệnh thứ hai vi phạm ràng buộc
`UNIQUE`, PostgreSQL **tự động huỷ toàn bộ transaction** — dòng đầu tiên, dù không hề có lỗi gì,
cũng biến mất hoàn toàn. Với `ROLLBACK` tường minh: bên trong transaction, dữ liệu **có thể thấy
được** (`count = 1`), nhưng sau `ROLLBACK`, kiểm tra từ một kết nối khác xác nhận dữ liệu
**không hề tồn tại** (`count = 0`) — đúng ngữ nghĩa "hoàn tác hoàn toàn".

## Deep Dive

**Vì sao PostgreSQL tự động huỷ toàn bộ transaction ngay khi gặp lỗi, thay vì chỉ báo lỗi và cho
phép tiếp tục các câu lệnh khác?** Đây là hành vi mặc định của PostgreSQL (khác với một số
database khác, ví dụ MySQL với engine InnoDB có thể cho phép transaction tiếp tục sau lỗi, tuỳ
cấu hình) — triết lý "fail fast, fail completely" (thất bại nhanh, thất bại triệt để) đảm bảo
không có khả năng transaction kết thúc ở một trạng thái mà lập trình viên **không** chủ động
kiểm soát (ví dụ quên kiểm tra kết quả câu lệnh thứ hai, tưởng nó cũng chạy thành công). Trong
PostgreSQL, sau khi một câu lệnh lỗi, toàn bộ transaction chuyển sang trạng thái "aborted" — mọi
câu lệnh tiếp theo (kể cả `COMMIT`) đều bị từ chối cho tới khi có `ROLLBACK` tường minh (hoặc
implicit rollback khi kết nối đóng) — đảm bảo không có "tai nạn" nào có thể vô tình commit một
phần dữ liệu.

## Engineering Insight

**Consistency (chữ C trong ACID) khác gì so với Isolation (chữ I) — hai khái niệm dễ gây nhầm lẫn
khi trả lời phỏng vấn?** Consistency liên quan tới **ràng buộc dữ liệu nội tại** (constraint,
foreign key, check, trigger) — đảm bảo mỗi transaction, dù thành công hay thất bại, không bao
giờ để database ở trạng thái vi phạm các ràng buộc đó (đã chứng minh trực tiếp: `UNIQUE
constraint` chặn đứng transaction vi phạm nó). Isolation liên quan tới **tương tác giữa nhiều
transaction chạy đồng thời** — một chủ đề hoàn toàn khác, về việc transaction A có "nhìn thấy"
thay đổi chưa commit của transaction B hay không (Chapter 06). Một câu trả lời phỏng vấn tốt cần
phân biệt rõ: Consistency là về **ràng buộc dữ liệu** (một transaction đơn lẻ, không liên quan
tới transaction khác), Isolation là về **cô lập giữa các transaction** (nhiều transaction cùng
lúc) — hai trục hoàn toàn độc lập của ACID, dễ bị gộp lẫn nếu không hiểu rõ.

## Historical Note

ACID (thuật ngữ, không phải khái niệm) được đặt tên bởi Andreas Reuter và Theo Härder năm 1983,
dựa trên các khái niệm nền tảng đã có từ nghiên cứu hệ thống database những năm 1970 (bao gồm cả
System R đã nhắc ở Chapter 04). Đây là một trong những bộ nguyên tắc có ảnh hưởng lâu dài nhất
trong lịch sử database — gần như mọi hệ quản trị database quan hệ hiện đại (PostgreSQL, MySQL,
Oracle, SQL Server) đều thiết kế xoay quanh việc đảm bảo bốn tính chất này.

## Myth vs Reality

- **Myth:** "Nếu một câu lệnh trong transaction lỗi, chỉ câu lệnh đó bị huỷ, các câu lệnh khác
  đã thành công trước đó vẫn được giữ nguyên."
  **Reality:** Đã chứng minh bằng thực nghiệm ngược lại — PostgreSQL huỷ **toàn bộ** transaction
  khi gặp lỗi, kể cả các câu lệnh đã thành công trước đó.

- **Myth:** "Consistency và Isolation trong ACID là hai tên gọi khác nhau cho cùng một khái
  niệm — 'dữ liệu không bị sai'."
  **Reality:** Xem Engineering Insight — Consistency là về ràng buộc dữ liệu nội tại; Isolation
  là về tương tác giữa các transaction chạy đồng thời, hai trục hoàn toàn khác nhau.

## Common Mistakes

- **Giả định câu lệnh SQL thành công trước đó trong transaction "an toàn" ngay cả khi câu sau
  lỗi** — đã chứng minh sai, cần luôn xử lý lỗi và `ROLLBACK` tường minh khi cần.
- **Không bọc nhiều câu lệnh liên quan trong cùng một transaction** — mất tính Atomicity, có
  nguy cơ dữ liệu nửa vời nếu có lỗi giữa chừng.
- **Nhầm lẫn Consistency với "dữ liệu đúng theo logic nghiệp vụ phức tạp"** — Consistency trong
  ACID chỉ đảm bảo ràng buộc **khai báo được** ở tầng database (constraint), không đảm bảo mọi
  logic nghiệp vụ phức tạp hơn (điều đó thuộc trách nhiệm tầng ứng dụng).

## Best Practices

- Luôn bọc các thao tác ghi liên quan tới nhau trong cùng một transaction, để tận dụng Atomicity.
- Định nghĩa ràng buộc (constraint) rõ ràng ở tầng database cho mọi quy tắc dữ liệu có thể diễn
  đạt được (không chỉ dựa vào validate ở tầng ứng dụng, như đã học ở Phase 5, Chapter 10).
- Xử lý lỗi tường minh trong mọi transaction đa câu lệnh — không giả định các câu lệnh trước đó
  "an toàn" chỉ vì chúng đã chạy mà không lỗi.

## Debug Checklist

- [ ] Dữ liệu "biến mất" sau một lỗi trong transaction, dù chính câu lệnh đó không lỗi? → bình
      thường, đúng tính chất Atomicity — kiểm tra lỗi ở câu lệnh khác trong cùng transaction.
- [ ] Transaction báo lỗi "current transaction is aborted, commands ignored until end of
      transaction block"? → một câu lệnh trước đó đã lỗi, cần `ROLLBACK` trước khi tiếp tục.
- [ ] Không chắc một ràng buộc nghiệp vụ có được đảm bảo ở tầng database không? → kiểm tra có
      `CONSTRAINT`/`UNIQUE`/`FOREIGN KEY`/`CHECK` tương ứng, hay chỉ dựa vào validate tầng ứng
      dụng.

## Summary

ACID gồm Atomicity (tất cả hoặc không gì cả), Consistency (luôn tôn trọng ràng buộc dữ liệu),
Isolation (transaction đồng thời không ảnh hưởng nhau ngoài ý muốn, Chapter 06-08), Durability
(đã commit thì tồn tại vĩnh viễn). Đã chứng minh bằng thực nghiệm: một `INSERT` vi phạm ràng buộc
`UNIQUE` khiến PostgreSQL huỷ **toàn bộ** transaction, kể cả `INSERT` trước đó đã thành công và
không hề vi phạm gì — đúng tính chất Atomicity "tất cả hoặc không gì cả". `ROLLBACK` tường minh
hoàn tác hoàn toàn thay đổi, xác nhận qua kiểm tra từ kết nối khác. Consistency (ràng buộc dữ
liệu nội tại) và Isolation (tương tác giữa các transaction đồng thời) là hai trục độc lập, dễ
nhầm lẫn khi trả lời phỏng vấn.

## Interview Questions

- ACID là gì? Giải thích từng chữ cái, và chứng minh bằng ví dụ cụ thể tính chất Atomicity hoạt
  động như thế nào.

**Mid**

- Consistency và Isolation trong ACID khác nhau như thế nào?
- Vì sao PostgreSQL huỷ toàn bộ transaction khi một câu lệnh lỗi, thay vì chỉ huỷ câu lệnh đó?

## Exercises

- [ ] Chạy lại các câu lệnh ở Example trên một PostgreSQL thật, xác nhận kết quả đúng như mô tả.
- [ ] Viết một transaction có 3 câu lệnh `INSERT` liên quan tới nhau, cố tình cho câu thứ ba lỗi,
      xác nhận cả 3 đều bị huỷ.
- [ ] Thử `SAVEPOINT` (điểm khôi phục cục bộ trong một transaction) để chỉ hoàn tác một phần
      transaction thay vì toàn bộ, tìm hiểu khi nào kỹ thuật này hữu ích.

## Cheat Sheet

| Chữ cái | Tính chất | Đảm bảo điều gì |
| --- | --- | --- |
| A | Atomicity | Tất cả câu lệnh cùng thành công, hoặc tất cả cùng thất bại |
| C | Consistency | Ràng buộc dữ liệu (constraint) luôn được tôn trọng |
| I | Isolation | Transaction đồng thời không ảnh hưởng nhau ngoài ý muốn (Chapter 06-08) |
| D | Durability | Đã `COMMIT` thì tồn tại vĩnh viễn, sống sót qua crash |

## References

- Härder, T.; Reuter, A. — "Principles of Transaction-Oriented Database Recovery" (1983, nguồn
  gốc thuật ngữ ACID).
- PostgreSQL Documentation — Transactions.
