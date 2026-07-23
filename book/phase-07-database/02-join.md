---
tags:
  - SQL
  - JOIN
  - PostgreSQL
---

# JOIN: INNER, LEFT, và cạm bẫy WHERE vs ON

> Phase: Phase 7 — Database
> Chapter slug: `join`

## Metadata

```yaml
Chapter: JOIN
Phase: Phase 7 — Database
Difficulty: ★★★
Importance: ★★★★★
Interview Frequency: 90%
Prerequisites:
  - Chapter 01 — SQL
Used Later:
  - Chapter 04 — Execution Plan
  - Chapter 09 — Optimization
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: [Chapter 01](01-sql.md) — `NULL = NULL` không bao giờ là `TRUE`. Điều này gây ra một
> trong những cạm bẫy JOIN phổ biến và nguy hiểm nhất: đặt điều kiện lọc **sai chỗ** khiến
> `LEFT JOIN` âm thầm biến thành `INNER JOIN` mà không có bất kỳ lỗi cú pháp nào.

```sql
-- Dieu kien loc trong ON: giu nguyen tinh chat LEFT JOIN
SELECT c.name, o.id AS order_id, o.status
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id AND o.status = 'SHIPPED';
```

```
     name     | order_id | status
--------------+----------+---------
 Le Van C     |          |            <- VAN CON, du khong co order SHIPPED
 Nguyen Van A |        2 | SHIPPED
 Pham Thi D   |          |            <- VAN CON
 Tran Thi B   |          |            <- VAN CON, du order cua ho la NEW
```

```sql
-- CUNG dieu kien do, nhung dat trong WHERE
SELECT c.name, o.id AS order_id, o.status
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'SHIPPED';
```

```
     name     | order_id | status
--------------+----------+---------
 Nguyen Van A |        2 | SHIPPED
```

Cùng logic lọc, cùng dữ liệu — nhưng chỉ khác **vị trí** đặt điều kiện, kết quả thay đổi từ **4
dòng** xuống còn **1 dòng**. Không hề có lỗi cú pháp nào cảnh báo.

## Interview Question (Central)

> `INNER JOIN` và `LEFT JOIN` khác nhau như thế nào? Vì sao đặt điều kiện lọc trong `WHERE` thay
> vì `ON` có thể âm thầm biến `LEFT JOIN` thành `INNER JOIN`?

## Objectives

- [ ] Dùng thành thạo `INNER JOIN`/`LEFT JOIN`, hiểu chính xác sự khác biệt về tập kết quả
- [ ] Tự tay chứng minh bằng thực nghiệm: `INNER JOIN` loại bỏ customer không có order, `LEFT
      JOIN` giữ lại (điền `NULL`)
- [ ] Tự tay tái hiện cạm bẫy "điều kiện lọc trong `WHERE` vô tình huỷ tính chất `LEFT JOIN`",
      liên hệ trực tiếp tới ngữ nghĩa `NULL` đã học ở Chapter 01

## Prerequisites

- Chapter 01 — bắt buộc phải hiểu ngữ nghĩa `NULL`/logic ba trị trước chapter này, vì nó chính
  là nguyên nhân kỹ thuật của cạm bẫy `WHERE` vs `ON`.

## Used Later

- **Chapter 04 (Execution Plan)** — cách database engine chọn thuật toán (nested loop, hash
  join, merge join) để thực thi một JOIN.
- **Chapter 09 (Optimization)** — JOIN không đúng index là một trong những nguyên nhân phổ biến
  nhất của truy vấn chậm.

## Problem

Dữ liệu quan hệ (relational) tách thông tin thành nhiều bảng (Phase 6 đã thấy `Customer`/`Order`
tách biệt) — để lấy được thông tin **kết hợp** từ nhiều bảng (ví dụ "tên khách hàng kèm đơn hàng
của họ"), cần một cơ chế **nối** (join) các bảng lại theo một điều kiện liên kết (thường là khoá
ngoại). Nhưng "nối" có nhiều ngữ nghĩa khác nhau: chỉ giữ lại các cặp **khớp** được cả hai bên
(`INNER JOIN`), hay giữ lại **toàn bộ** một bên kể cả khi không tìm được cặp khớp
(`LEFT JOIN`)?

## Concept

**`INNER JOIN`** chỉ trả về các dòng có **cặp khớp** ở cả hai bảng theo điều kiện join — nếu một
`Customer` không có `Order` nào khớp, `Customer` đó **biến mất hoàn toàn** khỏi kết quả.
**`LEFT JOIN`** (hay `LEFT OUTER JOIN`) giữ lại **toàn bộ** dòng từ bảng bên trái (`customers`),
kể cả khi không tìm được dòng khớp ở bảng bên phải (`orders`) — với các dòng không khớp, mọi cột
đến từ bảng bên phải được điền `NULL`.

## Why?

Sự khác biệt giữa `INNER` và `LEFT JOIN` phản ánh trực tiếp hai nhu cầu nghiệp vụ khác nhau:
"chỉ quan tâm các cặp có quan hệ thực sự tồn tại" (INNER — ví dụ báo cáo doanh thu, chỉ cần đơn
hàng thật) so với "cần đầy đủ danh sách một bên, có thông tin liên kết thì tốt, không có cũng
không sao" (LEFT — ví dụ danh sách toàn bộ khách hàng, kèm đơn hàng gần nhất nếu có). Vấn đề
`WHERE` vs `ON` xảy ra vì hai mệnh đề này có **thời điểm áp dụng khác nhau** trong ngữ nghĩa
`LEFT JOIN`: điều kiện trong `ON` được áp dụng **trong lúc join** (quyết định dòng nào khớp,
trước khi điền `NULL` cho phần không khớp), còn điều kiện trong `WHERE` được áp dụng **sau khi**
`LEFT JOIN` đã hoàn tất (kể cả với các dòng đã bị điền `NULL`) — và vì `NULL = 'SHIPPED'` luôn
là `UNKNOWN` (Chapter 01), những dòng đã bị điền `NULL` bị `WHERE` loại bỏ tiếp, xoá sạch đúng
phần lợi ích mà `LEFT JOIN` mang lại.

## How?

```sql
-- INNER JOIN: CHI giu cap KHOP
SELECT c.name, o.status FROM customers c
INNER JOIN orders o ON o.customer_id = c.id;

-- LEFT JOIN: giu TOAN BO customers, dien NULL neu khong khop
SELECT c.name, o.status FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id;

-- Loc THEO DIEU KIEN cua bang phai ma VAN giu tinh chat LEFT JOIN: dat trong ON
SELECT c.name, o.status FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id AND o.status = 'SHIPPED';
```

## Visualization

```
INNER JOIN:                       LEFT JOIN:

  customers    orders               customers    orders
  ┌────┐       ┌────┐               ┌────┐       ┌────┐
  │ A  │──────>│ o1 │               │ A  │──────>│ o1 │
  │ A  │──────>│ o2 │               │ A  │──────>│ o2 │
  │ B  │──────>│ o3 │               │ B  │──────>│ o3 │
  │ C  │   X   (khong co)           │ C  │──────>│NULL│  <- VAN CO, dien NULL
  │ D  │   X   (khong co)           │ D  │──────>│NULL│  <- VAN CO, dien NULL
  └────┘                            └────┘

Ket qua: 3 dong (C, D bien mat)   Ket qua: 5 dong (C, D con, cot orders = NULL)

LEFT JOIN + loc trong ON:          LEFT JOIN + loc trong WHERE:

  ON giu C,D (dien NULL)            WHERE chay SAU khi da LEFT JOIN
  vi dieu kien CHI anh huong        WHERE loai dong co orders.status=NULL
  viec KHOP voi orders, khong       (vi NULL = 'SHIPPED' la UNKNOWN)
  loai bo chinh dong C, D           => C, D, VA CA cac Order NEW bi loai
  => 4 dong (van co C, D)           => 1 dong (thanh INNER JOIN tren thuc te)
```

## Example

```sql
SELECT c.name, o.id AS order_id, o.status
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id
ORDER BY c.name;
```

Kết quả thật (PostgreSQL 16.14):

```
     name     | order_id | status
--------------+----------+---------
 Nguyen Van A |        1 | NEW
 Nguyen Van A |        2 | SHIPPED
 Tran Thi B   |        3 | NEW
(3 rows)
```

```sql
SELECT c.name, o.id AS order_id, o.status
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
ORDER BY c.name;
```

```
     name     | order_id | status
--------------+----------+---------
 Le Van C     |          |
 Nguyen Van A |        1 | NEW
 Nguyen Van A |        2 | SHIPPED
 Pham Thi D   |          |
 Tran Thi B   |        3 | NEW
(5 rows)
```

`Le Van C` và `Pham Thi D` (không có `Order` nào) **biến mất** với `INNER JOIN` (chỉ 3 dòng), 
nhưng **vẫn xuất hiện** với `LEFT JOIN` (5 dòng, với `order_id`/`status` là `NULL`).

**Cạm bẫy `WHERE` vs `ON`** — xem đầy đủ ở Story: đặt `o.status = 'SHIPPED'` trong `ON` giữ
nguyên 4 khách hàng (chỉ khác giá trị `order_id`/`status` tuỳ có đơn SHIPPED hay không); đặt
cùng điều kiện đó trong `WHERE` xoá sạch mọi khách hàng không có đơn SHIPPED — kể cả những khách
hàng **có** đơn hàng (chỉ là đơn NEW, không phải SHIPPED) cũng bị loại, đúng như một `INNER
JOIN` thông thường.

## Deep Dive

**Giải thích chi tiết cơ chế khiến `WHERE` "vô tình" huỷ `LEFT JOIN`:** `LEFT JOIN` được PostgreSQL
xử lý theo hai bước logic: (1) tìm mọi cặp khớp theo điều kiện `ON`; (2) với mỗi dòng bên trái
**không** tìm được cặp khớp nào, tạo thêm một dòng "giả", điền `NULL` cho mọi cột bên phải. Kết
quả trung gian sau bước này đã bao gồm cả các dòng "giả" đó. `WHERE` (nếu có) chạy **sau cùng**,
lọc trên **toàn bộ** kết quả trung gian đó — bao gồm cả các dòng "giả" vừa tạo. Vì các dòng giả
có `o.status = NULL`, và `NULL = 'SHIPPED'` cho `UNKNOWN` (Chapter 01), `WHERE` loại bỏ toàn bộ
các dòng giả đó — vô tình xoá sạch đúng phần dữ liệu mà `LEFT JOIN` được thiết kế để giữ lại.
Ngược lại, điều kiện trong `ON` chỉ ảnh hưởng **bước (1)** (quyết định "cặp nào được coi là khớp"),
hoàn toàn không ảnh hưởng bước (2) (dòng nào được giữ lại dù không khớp) — đây là lý do đặt điều
kiện trong `ON` giữ nguyên được tính chất `LEFT JOIN`.

## Engineering Insight

**Trong thực tế, lỗi "WHERE vô tình huỷ LEFT JOIN" nguy hiểm tới mức nào, và vì sao nó thường
không bị phát hiện qua test thông thường?** Đây là một trong những lỗi SQL **âm thầm nhất** —
không có exception, không có cảnh báo cú pháp, kết quả trả về vẫn là dữ liệu "hợp lệ" (chỉ là
thiếu một phần). Nó đặc biệt nguy hiểm trong các báo cáo/dashboard nghiệp vụ: một truy vấn "danh
sách toàn bộ khách hàng kèm đơn hàng gần nhất (nếu có)" viết sai kiểu này sẽ **âm thầm loại bỏ**
mọi khách hàng chưa từng mua hàng — một sai sót nghiêm trọng về mặt phân tích kinh doanh (ví dụ
đội marketing muốn nhắm tới đúng nhóm khách hàng "chưa từng mua" sẽ nhận báo cáo trống rỗng, dẫn
tới kết luận sai hoàn toàn) nhưng lại trông hoàn toàn "bình thường" khi xem qua kết quả (vẫn là
một bảng dữ liệu có vẻ hợp lý, không có lỗi rõ ràng). Đây là lý do luôn cần kiểm tra: khi thêm
điều kiện lọc liên quan tới bảng bên phải của một `LEFT JOIN`, cần tự hỏi "tôi có đang cố tình
loại bỏ những dòng không khớp không? Nếu không, điều kiện này phải nằm trong `ON`".

## Historical Note

Cú pháp `JOIN` tường minh (`INNER JOIN`/`LEFT JOIN ... ON ...`) chuẩn hoá trong SQL-92 (1992) —
trước đó, join thường viết theo kiểu "implicit join" cũ hơn
(`SELECT ... FROM a, b WHERE a.id = b.a_id`, liệt kê nhiều bảng cách nhau dấu phẩy trong `FROM`,
điều kiện join đặt luôn trong `WHERE`). Cú pháp cũ này chính là nguồn gốc lịch sử của thói quen
"đặt mọi điều kiện trong `WHERE`" — thói quen đó vô hại với `INNER JOIN` implicit (không có khái
niệm "giữ lại dòng không khớp"), nhưng trở thành cạm bẫy nguy hiểm khi áp dụng nhầm sang
`LEFT JOIN` hiện đại.

## Myth vs Reality

- **Myth:** "Đặt điều kiện lọc trong `ON` hay `WHERE` chỉ là khác biệt về phong cách viết, không
  ảnh hưởng kết quả."
  **Reality:** Đã chứng minh bằng thực nghiệm — với `LEFT JOIN`, vị trí đặt điều kiện thay đổi
  hoàn toàn tập kết quả trả về (4 dòng so với 1 dòng).

- **Myth:** "`LEFT JOIN` luôn chậm hơn `INNER JOIN` vì phải xử lý thêm trường hợp không khớp."
  **Reality:** Hiệu năng phụ thuộc chủ yếu vào Execution Plan (Chapter 04) và index (Chapter
  03), không phải bản chất `LEFT`/`INNER` — cả hai đều có thể nhanh hoặc chậm tuỳ dữ liệu và
  index.

## Common Mistakes

- **Đặt điều kiện lọc liên quan tới bảng bên phải của `LEFT JOIN` vào `WHERE` thay vì `ON`** —
  cạm bẫy trung tâm của chapter này, âm thầm biến `LEFT JOIN` thành `INNER JOIN`.
- **Dùng `INNER JOIN` khi thực sự cần giữ lại toàn bộ một bên** — mất dữ liệu không mong muốn mà
  không có cảnh báo.
- **Không kiểm tra kỹ số dòng kết quả khi đổi từ `INNER` sang `LEFT JOIN` (hoặc ngược lại)** —
  nên luôn xác nhận số dòng thay đổi đúng như kỳ vọng.

## Best Practices

- Khi lọc theo điều kiện của bảng bên phải trong `LEFT JOIN` mà vẫn muốn giữ nguyên tính chất
  "giữ lại dòng không khớp", luôn đặt điều kiện đó trong `ON`.
- Khi thực sự muốn loại bỏ hoàn toàn các dòng không khớp (bất kể lý do), có thể đặt trong
  `WHERE` — nhưng nên nhận thức rõ ràng đây là hành vi tương đương `INNER JOIN`, cân nhắc dùng
  hẳn `INNER JOIN` cho rõ ràng thay vì `LEFT JOIN` + `WHERE`.
- Luôn viết cú pháp `JOIN` tường minh (`INNER JOIN`/`LEFT JOIN ... ON ...`), tránh cú pháp
  implicit join cũ (liệt kê bảng cách nhau dấu phẩy).

## Debug Checklist

- [ ] `LEFT JOIN` nhưng kết quả thiếu các dòng lẽ ra phải có (dòng không khớp)? → kiểm tra có
      điều kiện nào liên quan tới bảng bên phải đang nằm trong `WHERE` thay vì `ON`.
- [ ] Không chắc một truy vấn `LEFT JOIN` có đang hoạt động đúng ý đồ không? → đếm số dòng kết
      quả, so sánh với số dòng dự kiến ở bảng bên trái (phải luôn ≥ số dòng đó nếu là LEFT JOIN
      đúng).

## Summary

`INNER JOIN` chỉ giữ lại cặp khớp giữa hai bảng; `LEFT JOIN` giữ toàn bộ bảng bên trái, điền
`NULL` cho phần không khớp bên phải. Đã chứng minh bằng thực nghiệm: `INNER JOIN` loại 2 khách
hàng không có đơn hàng (còn 3 dòng), `LEFT JOIN` giữ lại cả hai (5 dòng, `NULL` ở cột order).
Cạm bẫy quan trọng nhất: đặt điều kiện lọc bảng bên phải trong `WHERE` (thay vì `ON`) khiến
`WHERE` loại luôn các dòng đã bị điền `NULL` (vì `NULL = giá_trị` luôn là `UNKNOWN`, Chapter 01)
— âm thầm biến `LEFT JOIN` thành `INNER JOIN` mà không có lỗi cú pháp nào cảnh báo. Đã chứng
minh trực tiếp: cùng điều kiện, đặt trong `ON` giữ 4 dòng, đặt trong `WHERE` chỉ còn 1 dòng.

## Interview Questions

- `INNER JOIN` và `LEFT JOIN` khác nhau như thế nào?
- Vì sao đặt điều kiện lọc trong `WHERE` thay vì `ON` có thể âm thầm biến `LEFT JOIN` thành
  `INNER JOIN`?

**Senior**

- Giải thích cơ chế hai bước của `LEFT JOIN` (tìm khớp, điền NULL cho không khớp) và cách
  `WHERE`/`ON` tương tác khác nhau với từng bước.
- Vì sao lỗi "WHERE vô tình huỷ LEFT JOIN" đặc biệt nguy hiểm trong báo cáo nghiệp vụ, dù không
  gây lỗi cú pháp?

## Exercises

- [ ] Chạy lại các câu lệnh ở Example trên một PostgreSQL thật, xác nhận số dòng đúng như mô tả.
- [ ] Viết một câu `RIGHT JOIN` tương đương một `LEFT JOIN` đã có (đổi vị trí bảng), xác nhận
      kết quả giống hệt nhau.
- [ ] Cố tình viết một truy vấn `LEFT JOIN` với điều kiện lọc sai vị trí (`WHERE`), rồi sửa lại
      đúng vị trí (`ON`), so sánh số dòng kết quả trước và sau khi sửa.

## Cheat Sheet

| | `INNER JOIN` | `LEFT JOIN` |
| --- | --- | --- |
| Dòng không khớp bên trái | Loại bỏ | Giữ lại, điền `NULL` |
| Điều kiện lọc bảng phải trong `ON` | Không ảnh hưởng tính "giữ toàn bộ" (N/A cho INNER) | Giữ nguyên tính chất LEFT JOIN |
| Điều kiện lọc bảng phải trong `WHERE` | Tương đương `ON` (N/A) | **Vô tình biến thành INNER JOIN** |

## References

- PostgreSQL Documentation — Joins Between Tables.
- SQL-92 Standard — Joined Table syntax.
