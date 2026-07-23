---
tags:
  - SQL
  - ExecutionPlan
  - PostgreSQL
---

# Execution Plan (EXPLAIN ANALYZE)

> Phase: Phase 7 — Database
> Chapter slug: `execution-plan`

## Metadata

```yaml
Chapter: Execution Plan
Phase: Phase 7 — Database
Difficulty: ★★★★
Importance: ★★★★★
Interview Frequency: 80%
Prerequisites:
  - Chapter 01 — SQL
  - Chapter 02 — JOIN
  - Chapter 03 — Index
Used Later:
  - Chapter 09 — Optimization
Estimated Reading: 18 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: [Chapter 01](01-sql.md) — SQL là khai báo, engine tự quyết định "làm thế nào". Câu
> hỏi tự nhiên tiếp theo: làm sao **nhìn thấy** được engine đã quyết định làm thế nào?

```sql
EXPLAIN SELECT * FROM big_orders WHERE customer_email = 'user12345@example.com';
```

```
Index Scan using idx_big_orders_email on big_orders  (cost=0.43..20.40 rows=4 width=33)
```

```sql
EXPLAIN ANALYZE SELECT * FROM big_orders WHERE customer_email = 'user12345@example.com';
```

```
Index Scan using idx_big_orders_email on big_orders (cost=0.43..20.40 rows=4 width=33)
                                                      (actual time=0.139..0.218 rows=4 loops=1)
Planning Time: 0.196 ms
Execution Time: 0.232 ms
```

Cùng một câu lệnh cơ bản, nhưng `EXPLAIN` (không có `ANALYZE`) chỉ hiện **ước lượng**
(`cost`), trong khi `EXPLAIN ANALYZE` thực sự **chạy câu truy vấn** và báo cáo số liệu **thật**
(`actual time`, `rows`, `loops`).

## Interview Question (Central)

> `EXPLAIN` và `EXPLAIN ANALYZE` khác nhau như thế nào? Làm sao đọc hiểu một execution plan để
> chẩn đoán truy vấn chậm?

## Objectives

- [ ] Phân biệt `EXPLAIN` (chỉ ước lượng, không chạy) và `EXPLAIN ANALYZE` (chạy thật, báo cáo
      số liệu thật)
- [ ] Đọc hiểu các loại node phổ biến: `Seq Scan`, `Index Scan`, `Nested Loop`, `Hash Join`
- [ ] Tự tay chứng minh bằng thực nghiệm: planner chọn thuật toán JOIN khác nhau tuỳ ngữ cảnh
      (kích thước bảng, selectivity điều kiện)

## Prerequisites

- Chapter 01 — hiểu SQL khai báo, tiền đề để hiểu vì sao cần công cụ riêng để "nhìn thấy" cách
  thực thi.
- Chapter 02 — hiểu JOIN, đối tượng chính các thuật toán trong execution plan xử lý.
- Chapter 03 — hiểu Index, một trong những yếu tố chính ảnh hưởng lựa chọn của planner.

## Used Later

- **Chapter 09 (Optimization)** — `EXPLAIN ANALYZE` là công cụ chẩn đoán trung tâm cho mọi kỹ
  thuật tối ưu truy vấn.

## Problem

Vì SQL là khai báo (Chapter 01), cùng một câu lệnh có thể được thực thi theo **nhiều cách khác
nhau** (dùng index hay không, thuật toán join nào, thứ tự join các bảng ra sao) — nếu không có
công cụ nhìn thấy được engine **thực sự** chọn cách nào, việc chẩn đoán một truy vấn chậm chỉ có
thể dựa vào phỏng đoán, không có căn cứ cụ thể.

## Concept

**`EXPLAIN`** hiển thị **kế hoạch thực thi ước lượng** (estimated plan) mà planner sẽ dùng cho
một câu truy vấn — dựa trên thống kê phân bố dữ liệu (số dòng, số giá trị phân biệt) mà database
tự thu thập, **không thực sự chạy câu truy vấn**. **`EXPLAIN ANALYZE`** thực sự **chạy** câu
truy vấn, đồng thời báo cáo cả kế hoạch ước lượng **và** số liệu thực tế (thời gian thật, số
dòng thật ở mỗi bước) — cho phép so sánh trực tiếp "engine nghĩ nó sẽ làm gì" với "nó thực sự đã
làm gì", từ đó phát hiện các trường hợp planner ước lượng sai (thường là dấu hiệu thống kê đã
cũ, cần chạy lại `ANALYZE` để cập nhật).

## Why?

Tách `EXPLAIN` (chỉ ước lượng) khỏi `EXPLAIN ANALYZE` (chạy thật) cho phép kiểm tra kế hoạch thực
thi **mà không phải trả giá** thực sự chạy câu truy vấn — hữu ích khi câu truy vấn có khả năng
chạy rất lâu hoặc có tác dụng phụ (`INSERT`/`UPDATE`/`DELETE`, dù `EXPLAIN ANALYZE` cho các câu
lệnh này **thực sự thay đổi dữ liệu**, cần cẩn trọng — thường bọc trong transaction rồi
`ROLLBACK` khi chỉ muốn xem plan). Việc phân biệt rõ giữa "ước lượng" và "thực tế" cũng là công
cụ chẩn đoán trực tiếp: nếu số dòng **ước lượng** (`rows=` trong phần `cost`) sai lệch lớn so
với số dòng **thực tế** (`rows=` trong phần `actual`), đó là dấu hiệu rõ ràng planner đang quyết
định dựa trên thống kê không chính xác.

## How?

```sql
EXPLAIN SELECT ...;          -- CHI uoc luong, KHONG chay that
EXPLAIN ANALYZE SELECT ...;  -- CHAY THAT, bao cao ca uoc luong lan thuc te
```

## Visualization

```
EXPLAIN (uoc luong):

  Index Scan using idx_big_orders_email on big_orders
    (cost=0.43..20.40 rows=4 width=33)
     │       │           │
     │       │           └── planner UOC LUONG se tra ve 4 dong
     │       └── chi phi UOC LUONG (don vi tuong doi, khong phai ms)
     └── chi phi khoi dong UOC LUONG

EXPLAIN ANALYZE (uoc luong + THUC TE):

  ... (actual time=0.139..0.218 rows=4 loops=1)
              │            │        │       │
              │            │        │       └── so lan node nay CHAY (quan trong voi vong lap ben trong JOIN)
              │            │        └── so dong THAT SU tra ve (so sanh voi uoc luong o tren)
              │            └── thoi gian THAT SU ket thuc (ms)
              └── thoi gian THAT SU bat dau tra ve dong dau tien (ms)
```

## Example

**Hash Join — cho hai bảng nhỏ:**

```sql
EXPLAIN ANALYZE
SELECT c.name, o.status FROM customers c JOIN orders o ON o.customer_id = c.id;
```

```
Hash Join (cost=12.93..33.07 rows=800 width=276) (actual time=0.059..0.061 rows=3 loops=1)
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o (actual time=0.019..0.019 rows=3 loops=1)
  ->  Hash (actual time=0.033..0.034 rows=4 loops=1)
        ->  Seq Scan on customers c (actual time=0.022..0.023 rows=4 loops=1)
```

**Nested Loop + Index Scan — khi một bên đã lọc còn rất ít dòng:**

```sql
EXPLAIN ANALYZE
SELECT bo.* FROM customers c
JOIN big_orders bo ON bo.customer_email = c.email
WHERE c.id = 1;
```

```
Nested Loop (actual time=0.080..0.081 rows=0 loops=1)
  ->  Index Scan using customers_pkey on customers c (actual time=0.017..0.017 rows=1 loops=1)
        Index Cond: (id = 1)
  ->  Index Scan using idx_big_orders_email on big_orders bo (actual time=0.061..0.061 rows=0 loops=1)
        Index Cond: ((customer_email)::text = (c.email)::text)
```

Với `customers`/`orders` (cả hai bảng **nhỏ**), planner chọn **Hash Join**: xây một bảng băm
(hash table) từ bên nhỏ hơn (`customers`, 4 dòng), rồi duyệt qua bên kia (`orders`) để dò khớp —
hiệu quả khi cả hai bảng đủ nhỏ để vừa trong bộ nhớ. Với truy vấn join `big_orders` (2 triệu
dòng) nhưng đã **lọc trước** `customers` xuống còn đúng 1 dòng (`WHERE c.id = 1`), planner chọn
**Nested Loop**: với **mỗi** dòng từ bên ngoài (`customers`, chỉ 1 dòng), tra cứu trực tiếp qua
Index Scan vào `big_orders` — hiệu quả khi bên ngoài có rất ít dòng và bên trong có index tốt,
tránh phải xây hash table cho bảng 2 triệu dòng một cách không cần thiết.

## Deep Dive

**Vì sao planner đổi thuật toán JOIN tuỳ ngữ cảnh thay vì luôn dùng một thuật toán "tốt nhất" cố
định?** Không có thuật toán JOIN nào tốt nhất tuyệt đối — mỗi thuật toán có đặc tính đánh đổi
khác nhau: **Nested Loop** hiệu quả khi bên ngoài **rất nhỏ** và bên trong có index tốt (chi phí
tỉ lệ với `số dòng bên ngoài × chi phí tra cứu index`) nhưng **tệ** khi cả hai bảng đều lớn (chi
phí tăng theo tích số dòng). **Hash Join** hiệu quả khi có thể xây hash table vừa trong bộ nhớ
(thường từ bảng nhỏ hơn) nhưng cần bộ nhớ tỉ lệ với kích thước bảng đó, và không tận dụng được
index. **Merge Join** (chưa xuất hiện trong ví dụ) hiệu quả khi cả hai bên **đã được sắp xếp**
sẵn theo cột join (ví dụ cả hai đều có index B-tree trên cột đó). Planner dùng **cost-based
optimization**: ước lượng chi phí (dựa trên thống kê phân bố dữ liệu) của **từng** thuật toán khả
thi cho tình huống cụ thể, rồi chọn phương án có chi phí ước lượng **thấp nhất** — đây là lý do
cùng loại JOIN nhưng ngữ cảnh dữ liệu khác nhau (kích thước bảng, điều kiện lọc trước đó) có thể
cho ra kế hoạch thực thi hoàn toàn khác nhau.

## Engineering Insight

**Khi `actual rows` lệch lớn so với ước lượng trong `cost=...rows=...`, đây là dấu hiệu của vấn
đề gì, và ảnh hưởng ra sao tới toàn bộ execution plan?** Planner luôn phải **ước lượng trước**
số dòng ở mỗi bước (dựa trên thống kê từ `ANALYZE` gần nhất) để quyết định thuật toán JOIN phù
hợp cho **bước tiếp theo** — nếu thống kê đã **cũ** (dữ liệu thay đổi nhiều kể từ lần chạy
`ANALYZE` gần nhất mà chưa cập nhật lại), ước lượng có thể sai lệch nghiêm trọng, dẫn tới
planner chọn **sai thuật toán** cho các bước sau (ví dụ chọn Nested Loop tưởng bên ngoài chỉ có
vài dòng, nhưng thực tế có hàng nghìn dòng — khiến tổng chi phí tăng vọt so với dự kiến). Đây là
lý do khi chẩn đoán một truy vấn "đột nhiên chậm đi" dù không có gì thay đổi trong code, một
trong những nghi phạm đầu tiên cần kiểm tra là: thống kê bảng đã lâu chưa được cập nhật (chạy
`ANALYZE table_name;` thủ công, hoặc kiểm tra `autovacuum`/`autoanalyze` có đang hoạt động đúng
không) — không phải luôn là vấn đề thiếu index hay thiết kế truy vấn sai.

## Historical Note

Cost-based query optimization (thay vì rule-based — chỉ áp dụng quy tắc cố định không xét tới dữ
liệu thực tế) trở thành chuẩn mực từ System R (dự án nghiên cứu của IBM, cuối thập niên 1970) —
một trong những đóng góp kỹ thuật quan trọng nhất của dự án này, ảnh hưởng trực tiếp tới thiết kế
optimizer của hầu hết database quan hệ hiện đại, bao gồm PostgreSQL.

## Myth vs Reality

- **Myth:** "`EXPLAIN` và `EXPLAIN ANALYZE` cho kết quả giống hệt nhau, chỉ khác việc có thực
  thi câu lệnh hay không."
  **Reality:** Đã chứng minh bằng thực nghiệm — `EXPLAIN ANALYZE` bổ sung số liệu **thực tế**
  (`actual time`/`rows`/`loops`) mà `EXPLAIN` đơn thuần không có, cho phép so sánh trực tiếp ước
  lượng và thực tế.

- **Myth:** "Planner luôn chọn cùng một thuật toán JOIN cho cùng một cặp bảng, bất kể điều kiện
  lọc."
  **Reality:** Đã chứng minh bằng thực nghiệm — cùng liên quan tới `customers`, nhưng join với
  `orders` (nhỏ) dùng Hash Join, join với `big_orders` (lớn, đã lọc trước) dùng Nested Loop.

## Common Mistakes

- **Chỉ dùng `EXPLAIN` (không `ANALYZE`) rồi kết luận về hiệu năng thực tế** — chỉ là ước lượng,
  có thể sai lệch nếu thống kê cũ.
- **Chạy `EXPLAIN ANALYZE` cho câu `INSERT`/`UPDATE`/`DELETE` mà quên đây là thực thi thật, gây
  thay đổi dữ liệu thật** — nên bọc trong transaction rồi `ROLLBACK` nếu chỉ muốn xem plan.
- **Không kiểm tra chênh lệch giữa `rows` ước lượng và `rows` thực tế** — bỏ lỡ dấu hiệu quan
  trọng nhất để phát hiện thống kê đã cũ.

## Best Practices

- Luôn dùng `EXPLAIN ANALYZE` (không chỉ `EXPLAIN`) khi thực sự cần chẩn đoán hiệu năng, để có
  số liệu thực tế đối chiếu.
- Kiểm tra chênh lệch giữa `rows` ước lượng và thực tế ở mỗi node — chênh lệch lớn là dấu hiệu
  cần chạy lại `ANALYZE table_name;`.
- Đọc execution plan từ **trong ra ngoài** (node lá trước, node gốc sau) — các node lá thường là
  điểm bắt đầu chi phí, node gốc là kết quả tổng hợp cuối cùng.

## Debug Checklist

- [ ] Truy vấn chậm không rõ nguyên nhân? → chạy `EXPLAIN ANALYZE`, tìm node có `actual time`
      lớn nhất, kiểm tra loại node đó (`Seq Scan` trên bảng lớn thường là nghi phạm hàng đầu).
- [ ] `rows` ước lượng và thực tế chênh lệch lớn? → chạy `ANALYZE table_name;` để cập nhật thống
      kê, kiểm tra lại plan.
- [ ] Không chắc vì sao planner chọn Nested Loop thay vì Hash Join (hoặc ngược lại)? → kiểm tra
      kích thước thực tế của từng bảng/kết quả trung gian tại thời điểm đó, cost-based
      optimizer chọn dựa trên ước lượng chi phí cụ thể của tình huống đó.

## Summary

`EXPLAIN` hiển thị kế hoạch thực thi **ước lượng**, không chạy câu truy vấn; `EXPLAIN ANALYZE`
thực sự chạy và bổ sung số liệu **thực tế** (`actual time`/`rows`/`loops`) — đã chứng minh bằng
thực nghiệm sự khác biệt trực tiếp giữa hai lệnh. Planner dùng cost-based optimization, chọn
thuật toán JOIN (`Hash Join`, `Nested Loop`, `Merge Join`) tuỳ theo ước lượng chi phí cụ thể của
từng tình huống — đã chứng minh: hai bảng nhỏ dùng Hash Join, một bảng lớn đã lọc trước xuống rất
ít dòng dùng Nested Loop + Index Scan. Chênh lệch lớn giữa `rows` ước lượng và thực tế là dấu
hiệu thống kê đã cũ, cần chạy lại `ANALYZE` — thường là nguyên nhân của các truy vấn "đột nhiên
chậm đi" dù không có gì thay đổi trong code.

## Interview Questions

- `EXPLAIN` và `EXPLAIN ANALYZE` khác nhau như thế nào?
- Làm sao đọc hiểu một execution plan để chẩn đoán truy vấn chậm?

**Senior**

- Giải thích sự khác biệt giữa Nested Loop, Hash Join, Merge Join. Khi nào planner chọn thuật
  toán nào?
- Chênh lệch giữa `rows` ước lượng và thực tế trong execution plan cho biết điều gì? Cách khắc
  phục?

## Exercises

- [ ] Chạy lại các câu lệnh ở Example trên một PostgreSQL thật, xác nhận planner chọn đúng thuật
      toán như mô tả.
- [ ] Chạy `EXPLAIN ANALYZE` cho một JOIN giữa hai bảng lớn (không lọc trước xuống ít dòng),
      quan sát planner có chuyển sang Hash Join hay Merge Join thay vì Nested Loop không.
- [ ] Xoá thống kê bằng cách chèn thêm nhiều dữ liệu mới mà không chạy `ANALYZE`, quan sát chênh
      lệch giữa `rows` ước lượng và thực tế xuất hiện.

## Cheat Sheet

| Lệnh | Chạy thật? | Có số liệu thực tế? |
| --- | --- | --- |
| `EXPLAIN` | Không | Không (chỉ ước lượng) |
| `EXPLAIN ANALYZE` | Có | Có (`actual time`/`rows`/`loops`) |

| Thuật toán JOIN | Hiệu quả khi |
| --- | --- |
| Nested Loop | Bên ngoài rất nhỏ, bên trong có index tốt |
| Hash Join | Cả hai bảng đủ nhỏ để hash table vừa bộ nhớ |
| Merge Join | Cả hai bên đã sắp xếp sẵn theo cột join |

## References

- PostgreSQL Documentation — Using EXPLAIN.
- Selinger, P. et al. — "Access Path Selection in a Relational Database Management System"
  (System R, 1979).
