---
tags:
  - SQL
  - Database
  - PostgreSQL
---

# SQL: Ngôn ngữ Khai báo (Declarative)

> Phase: Phase 7 — Database
> Chapter slug: `sql`

## Metadata

```yaml
Chapter: SQL
Phase: Phase 7 — Database
Difficulty: ★★
Importance: ★★★★★
Interview Frequency: 90%
Prerequisites:
  - Phase 1, Chapter 08 — OOP Fundamentals (tư duy khai báo vs tư duy mệnh lệnh)
Used Later:
  - Chapter 02 — JOIN
  - Chapter 03 — Index
  - Chapter 04 — Execution Plan
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Viết một câu `SELECT` với alias, rồi thử **dùng chính alias đó** ngay trong mệnh đề `WHERE`
> của cùng câu lệnh — điều tưởng như hiển nhiên lại thất bại.

```sql
SELECT name, price, price * 1.1 AS price_with_tax
FROM products
WHERE price_with_tax > 100;
```

Kết quả thật (PostgreSQL 16.14):

```
ERROR:  column "price_with_tax" does not exist
LINE 2: ...price * 1.1 AS price_with_tax FROM products WHERE price_with...
```

Nhưng đổi `WHERE` thành `ORDER BY` với **đúng alias đó**, lại hoạt động hoàn hảo:

```sql
SELECT name, price, price * 1.1 AS price_with_tax FROM products ORDER BY price_with_tax;
```

```
  name  |  price  | price_with_tax
--------+---------+----------------
 Mouse  |   25.00 |         27.500
 Chair  |  150.00 |        165.000
 Desk   |  300.00 |        330.000
 Laptop | 1200.00 |       1320.000
```

Cùng một alias, cùng một câu lệnh, nhưng dùng được ở `ORDER BY` mà không dùng được ở `WHERE` —
đây không phải lỗi ngẫu nhiên, mà là bằng chứng trực tiếp cho một quy tắc nền tảng của SQL.

## Interview Question (Central)

> SQL là ngôn ngữ khai báo (declarative) — điều này có nghĩa gì, và nó khác gì so với việc viết
> Java (mệnh lệnh, imperative) mà bạn đã quen thuộc?

## Objectives

- [ ] Giải thích chính xác "khai báo" (declarative) nghĩa là gì, đối lập với "mệnh lệnh"
      (imperative) đã quen thuộc từ Java
- [ ] Tự tay chứng minh bằng thực nghiệm: thứ tự **viết** một câu SQL (`SELECT ... FROM ...
      WHERE ...`) khác với thứ tự **thực thi logic** thật sự của nó
- [ ] Hiểu đúng ngữ nghĩa `NULL` trong SQL — vì sao `NULL = NULL` không bao giờ là `TRUE`

## Prerequisites

- Phase 1, Chapter 08 — có nền tảng OOP/tư duy mệnh lệnh của Java để đối chiếu với tư duy khai
  báo của SQL.

## Used Later

- **Chapter 02 (JOIN)**, **Chapter 03 (Index)**, **Chapter 04 (Execution Plan)** — toàn bộ Phase
  7 xây trên nền tảng hiểu đúng bản chất khai báo của SQL: bạn nói "tôi muốn gì", database engine
  tự quyết định "làm thế nào" (đúng chủ đề của Execution Plan).

## Problem

Lập trình viên đến từ nền tảng Java (mệnh lệnh — imperative: viết ra **từng bước** máy tính phải
làm, theo đúng thứ tự) thường vô thức áp dụng tư duy đó khi đọc SQL — giả định câu lệnh thực thi
đúng theo thứ tự các từ khoá xuất hiện trên màn hình (`SELECT` trước, rồi `FROM`, rồi `WHERE`).
Giả định sai này dẫn tới những lỗi khó hiểu (như ví dụ ở Story) và ngăn cản việc dự đoán đúng
hành vi của các câu truy vấn phức tạp hơn.

## Concept

**SQL (Structured Query Language)** là ngôn ngữ **khai báo** (declarative) — bạn mô tả **kết
quả mong muốn** ("tôi muốn những dòng thoả điều kiện X, nhóm theo Y, sắp xếp theo Z"), không mô
tả **các bước thực hiện** để đạt được kết quả đó (không có vòng lặp, không có biến trung gian
tường minh). Database engine (PostgreSQL, MySQL, ...) tự chịu trách nhiệm chuyển đổi mô tả khai
báo đó thành một **kế hoạch thực thi** cụ thể (Execution Plan, Chapter 04) — quyết định "làm thế
nào" hoàn toàn thuộc về engine, không phải người viết SQL.

## Why?

Tách "mô tả kết quả mong muốn" khỏi "cách thực hiện" mang lại hai lợi ích cốt lõi: (1) người viết
SQL không cần biết chi tiết cấu trúc dữ liệu vật lý (có index nào, dữ liệu sắp xếp ra sao) — chỉ
cần mô tả đúng logic nghiệp vụ; (2) database engine có toàn quyền **tối ưu hoá** cách thực thi
(chọn dùng index nào, thứ tự join nào hiệu quả nhất) mà không cần người viết SQL sửa lại câu
lệnh — hoàn toàn khác với Java, nơi thay đổi thuật toán đòi hỏi sửa trực tiếp code. Đây là lý do
cùng một câu SQL có thể chạy nhanh hoặc chậm tuỳ vào index/dữ liệu bên dưới, dù bản thân câu lệnh
không đổi (Chapter 03-04 sẽ khai thác sâu điều này).

## How?

```sql
-- Khai bao: "toi muon ten khach hang co don hang trang thai NEW"
SELECT DISTINCT c.name
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'NEW';
-- KHONG can noi database phai duyet bang nao truoc, dung thuat toan join nao
```

## Visualization

```
Thu tu VIET (tren man hinh):        Thu tu THUC THI LOGIC (that su):

  SELECT ...        (1)                FROM / JOIN         (1)
  FROM ...           (2)                WHERE               (2)
  WHERE ...          (3)                GROUP BY            (3)
  GROUP BY ...       (4)                HAVING              (4)
  HAVING ...         (5)                SELECT (tinh alias) (5)
  ORDER BY ...       (6)                ORDER BY            (6)

  => WHERE (buoc 2) chua "biet" alias duoc tinh o SELECT (buoc 5) -> LOI
  => ORDER BY (buoc 6) chay SAU SELECT (buoc 5) -> DUNG duoc alias
```

## Example

**Bằng chứng thứ tự thực thi logic qua lỗi alias:**

```sql
SELECT name, price, price * 1.1 AS price_with_tax
FROM products
WHERE price_with_tax > 100;
```

Kết quả thật:

```
ERROR:  column "price_with_tax" does not exist
```

```sql
SELECT name, price, price * 1.1 AS price_with_tax FROM products ORDER BY price_with_tax;
```

```
  name  |  price  | price_with_tax
--------+---------+----------------
 Mouse  |   25.00 |         27.500
 Chair  |  150.00 |        165.000
 Desk   |  300.00 |        330.000
 Laptop | 1200.00 |       1320.000
```

**Bằng chứng ngữ nghĩa `NULL`:**

```sql
SELECT 1 WHERE NULL = NULL;
```

```
 ?column?
----------
(0 rows)
```

```sql
SELECT 1 WHERE NULL IS NULL;
```

```
 ?column?
----------
        1
(1 row)
```

`WHERE` chỉ "biết" các cột thật sự tồn tại trong bảng nguồn (`price`), **chưa biết** alias
`price_with_tax` — vì theo thứ tự thực thi logic, `WHERE` (bước 2) chạy **trước** `SELECT` (bước
5, nơi alias được tính). `ORDER BY` (bước 6) chạy **sau** `SELECT`, nên dùng alias được bình
thường. Với `NULL`: `NULL = NULL` trả về `0 rows` — không phải vì so sánh sai, mà vì trong SQL,
so sánh bất kỳ giá trị nào với `NULL` (kể cả `NULL` với `NULL`) cho kết quả `UNKNOWN` (không phải
`TRUE` hay `FALSE`) — chỉ `IS NULL`/`IS NOT NULL` mới kiểm tra đúng ngữ nghĩa "có phải NULL hay
không".

## Deep Dive

**Vì sao `NULL = NULL` không phải là `TRUE` — điều này phản ánh triết lý gì của SQL?** SQL dùng
logic ba trị (three-valued logic): mỗi biểu thức boolean có thể là `TRUE`, `FALSE`, hoặc
`UNKNOWN` — khác với Java (chỉ có `true`/`false`). `NULL` trong SQL mang ý nghĩa "giá trị không
xác định/không áp dụng được" (không phải "giá trị rỗng" như `""` hay `0`) — so sánh hai giá trị
**đều không xác định** với nhau **không thể** kết luận chúng "bằng nhau" một cách hợp lý (làm
sao biết hai điều "không xác định" có giống nhau hay không?), nên kết quả là `UNKNOWN`, không
phải `TRUE`. Mọi mệnh đề `WHERE`/`JOIN ON` chỉ giữ lại dòng khi biểu thức là `TRUE` — `UNKNOWN`
bị loại bỏ y hệt `FALSE`, đây là lý do `WHERE NULL = NULL` không trả về dòng nào.

## Engineering Insight

**Vì sao hiểu "SQL là khai báo" quan trọng khi làm việc với JPA/Hibernate (Phase 6)?** Toàn bộ
lớp trừu tượng ORM (Phase 6) — derived query method (Chapter 03), JPQL, `@Query` — cuối cùng đều
sinh ra SQL khai báo, rồi **giao phó** cho database engine quyết định cách thực thi. Điều này
giải thích trực tiếp một hiện tượng đã gặp xuyên suốt Phase 6: cùng một JPQL, nhưng thời gian
thực thi thay đổi hoàn toàn tuỳ vào **index** có tồn tại hay không (Chapter 03), **execution
plan** engine chọn (Chapter 04) — không phải do code Java thay đổi. Lập trình viên quen tư duy
mệnh lệnh của Java dễ có xu hướng "ép" SQL hoạt động theo đúng thứ tự viết ra (ví dụ cố tối ưu
hiệu năng bằng cách sắp xếp lại mệnh đề `WHERE`) — nhưng vì SQL là khai báo, database engine có
toàn quyền **sắp xếp lại** thứ tự thực thi thật sự miễn đảm bảo đúng kết quả logic, khiến việc
"tối ưu" bằng cách viết lại thứ tự các mệnh đề gần như luôn vô ích.

## Historical Note

SQL ra đời tại IBM đầu thập niên 1970 (Donald Chamberlin, Raymond Boyce), dựa trên nền tảng lý
thuyết Đại số Quan hệ (Relational Algebra, Edgar F. Codd, 1970) — một mô hình toán học thuần
khai báo, mô tả các phép biến đổi tập hợp (selection, projection, join) mà không quy định thuật
toán cụ thể. Đây là lý do SQL, dù ra đời hơn 50 năm, vẫn giữ nguyên bản chất khai báo xuyên suốt
— nó được thiết kế theo đúng lý thuyết toán học đó ngay từ đầu, không phải một quyết định thiết
kế tuỳ tiện.

## Myth vs Reality

- **Myth:** "Câu SQL thực thi đúng theo thứ tự các từ khoá xuất hiện trên màn hình
  (`SELECT` → `FROM` → `WHERE`)."
  **Reality:** Đã chứng minh bằng thực nghiệm — thứ tự thực thi logic thật sự là `FROM` →
  `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY`, hoàn toàn khác thứ tự viết.

- **Myth:** "`NULL = NULL` trả về `TRUE`, vì hai giá trị `NULL` thì 'giống nhau'."
  **Reality:** Đã chứng minh bằng thực nghiệm — SQL dùng logic ba trị, so sánh với `NULL` luôn
  cho `UNKNOWN`, bị loại khỏi kết quả `WHERE` giống như `FALSE`.

## Common Mistakes

- **Dùng alias của `SELECT` trong mệnh đề `WHERE`** — lỗi cú pháp phổ biến với người mới, do
  chưa hiểu thứ tự thực thi logic (dùng `HAVING` nếu cần lọc theo giá trị đã tính toán/tổng hợp,
  hoặc lặp lại biểu thức gốc trong `WHERE`).
- **Dùng `WHERE column = NULL` thay vì `WHERE column IS NULL`** — luôn trả về `0 rows` một cách
  âm thầm (không có lỗi cú pháp), dễ gây bug khó phát hiện.
- **Cố "tối ưu" SQL bằng cách sắp xếp lại thứ tự các điều kiện trong `WHERE`** — vô ích, vì SQL
  là khai báo, engine tự quyết định thứ tự đánh giá thật sự (Chapter 04).

## Best Practices

- Luôn nhớ thứ tự thực thi logic (`FROM`→`WHERE`→`GROUP BY`→`HAVING`→`SELECT`→`ORDER BY`) khi
  debug lỗi liên quan tới alias hoặc phạm vi truy cập cột.
- Luôn dùng `IS NULL`/`IS NOT NULL` để kiểm tra `NULL`, không bao giờ dùng `= NULL`/`<> NULL`.
- Tin tưởng database engine tự tối ưu cách thực thi — tập trung viết SQL đúng logic nghiệp vụ,
  dùng Execution Plan (Chapter 04) để chẩn đoán hiệu năng thay vì đoán mò cách viết lại câu lệnh.

## Debug Checklist

- [ ] Lỗi "column ... does not exist" dù chắc chắn đã khai báo alias đó? → kiểm tra alias có
      đang được dùng trong `WHERE`/`GROUP BY`/`HAVING` (chạy trước `SELECT`) thay vì `ORDER BY`
      (chạy sau `SELECT`).
- [ ] Câu `WHERE column = NULL` luôn trả về rỗng dù dữ liệu chắc chắn có `NULL`? → đổi thành
      `WHERE column IS NULL`.

## Summary

SQL là ngôn ngữ khai báo — mô tả **kết quả mong muốn**, không mô tả **các bước thực hiện**;
database engine tự quyết định cách thực thi. Đã chứng minh bằng thực nghiệm: thứ tự thực thi
logic thật sự (`FROM`→`WHERE`→...→`SELECT`→`ORDER BY`) khác với thứ tự viết trên màn hình — dùng
alias trong `WHERE` gây lỗi vì `WHERE` chạy trước `SELECT`, nhưng dùng được trong `ORDER BY` vì
nó chạy sau. SQL dùng logic ba trị — `NULL = NULL` luôn là `UNKNOWN` (bị loại khỏi kết quả), chỉ
`IS NULL` mới kiểm tra đúng ngữ nghĩa. Hiểu bản chất khai báo này là nền tảng cho toàn bộ Phase
7 — đặc biệt Execution Plan (Chapter 04), nơi engine quyết định "làm thế nào" hoàn toàn độc lập
với cách câu SQL được viết ra.

## Interview Questions

- SQL là ngôn ngữ khai báo (declarative) — điều này có nghĩa gì, và nó khác gì so với việc viết
  Java?

**Mid**

- Thứ tự thực thi logic thật sự của một câu SQL là gì? Vì sao alias trong `SELECT` không dùng
  được trong `WHERE`?
- Vì sao `NULL = NULL` không trả về `TRUE`?

## Exercises

- [ ] Chạy lại các câu lệnh ở Example trên một PostgreSQL thật, xác nhận kết quả đúng như mô tả.
- [ ] Viết một câu SQL dùng `HAVING` (thay vì cố dùng alias trong `WHERE`) để lọc theo một giá
      trị tổng hợp (`SUM`/`COUNT`), xác nhận nó hoạt động đúng.
- [ ] Thử `SELECT NULL <> NULL`, `SELECT NULL OR TRUE`, `SELECT NULL AND FALSE` — dự đoán kết
      quả trước khi chạy, đối chiếu với logic ba trị đã học.

## Cheat Sheet

| Bước | Mệnh đề | Alias `SELECT` khả dụng? |
| --- | --- | --- |
| 1 | `FROM`/`JOIN` | Không |
| 2 | `WHERE` | Không |
| 3 | `GROUP BY` | Không (một số DB cho phép, không chuẩn) |
| 4 | `HAVING` | Không (tương tự) |
| 5 | `SELECT` | (chính là nơi tính alias) |
| 6 | `ORDER BY` | **Có** |

**Logic ba trị:** `TRUE` / `FALSE` / `UNKNOWN` — so sánh với `NULL` luôn cho `UNKNOWN`, bị loại
khỏi `WHERE` giống `FALSE`.

## References

- PostgreSQL Documentation — The SQL Language, Value Expressions.
- Edgar F. Codd — "A Relational Model of Data for Large Shared Data Banks" (1970).
