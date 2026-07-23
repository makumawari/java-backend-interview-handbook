---
tags:
  - SQL
  - Optimization
  - PostgreSQL
---

# Query Optimization: Covering Index và Keyset Pagination

> Phase: Phase 7 — Database
> Chapter slug: `optimization`

## Metadata

```yaml
Chapter: Optimization
Phase: Phase 7 — Database
Difficulty: ★★★★
Importance: ★★★★★
Interview Frequency: 80%
Prerequisites:
  - Chapter 03 — Index
  - Chapter 04 — Execution Plan
  - Chapter 08 — MVCC
Used Later:
  - Phase 8 — Kiến trúc hệ thống (thiết kế API phân trang hiệu quả)
Estimated Reading: 18 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: [Chapter 03](03-index.md) — index tăng tốc tìm kiếm. Nhưng một API phân trang kinh
> điển — "lấy trang thứ 100,000" (`OFFSET 1000000 LIMIT 10`) — vẫn có thể chậm **dù đã có
> index đầy đủ**. Vì sao?

```sql
EXPLAIN ANALYZE SELECT id FROM big_orders ORDER BY id LIMIT 10 OFFSET 1000000;
```

```
Limit (actual time=109.794..109.795 rows=10 loops=1)
  ->  Index Only Scan using big_orders_pkey on big_orders
        (actual time=0.704..94.074 rows=1000010 loops=1)
Execution Time: 109.817 ms
```

Đổi sang **keyset pagination** (dùng `WHERE id > giá_trị_cuối_trang_trước` thay vì `OFFSET`):

```sql
EXPLAIN ANALYZE SELECT id FROM big_orders WHERE id > 1000000 ORDER BY id LIMIT 10;
```

```
Limit (actual time=0.019..0.019 rows=10 loops=1)
  ->  Index Only Scan using big_orders_pkey on big_orders
        (actual time=0.018..0.019 rows=10 loops=1)
Execution Time: 0.031 ms
```

Cùng lấy đúng 10 dòng — **109.817ms xuống còn 0.031ms**, nhanh hơn khoảng **3,500 lần**, chỉ nhờ
đổi cách viết điều kiện phân trang.

## Interview Question (Central)

> Vì sao `OFFSET` với giá trị lớn lại chậm, dù đã có index đầy đủ? Đề xuất cách tối ưu.

## Objectives

- [ ] Hiểu vì sao `OFFSET` buộc database phải **đếm và bỏ qua** từng dòng, bất kể có index
- [ ] Tự tay chứng minh bằng thực nghiệm: keyset pagination (dùng điều kiện `WHERE`) nhanh hơn
      `OFFSET` hàng nghìn lần với trang xa
- [ ] Hiểu **Covering Index** và **Index Only Scan** — khi nào database có thể trả lời truy vấn
      hoàn toàn từ index, không cần đọc bảng gốc

## Prerequisites

- Chapter 03 — hiểu Index B-tree cơ bản.
- Chapter 04 — hiểu cách đọc `EXPLAIN ANALYZE`.
- Chapter 08 — hiểu MVCC, nền tảng để hiểu vì sao `Index Only Scan` đôi khi vẫn cần
  "Heap Fetches".

## Used Later

- **Phase 8 (Kiến trúc hệ thống)** — thiết kế API phân trang hiệu quả cho hệ thống có lượng dữ
  liệu lớn là một quyết định kiến trúc quan trọng, trực tiếp áp dụng kiến thức chapter này.

## Problem

Phân trang kiểu `LIMIT n OFFSET m` là cách viết trực quan, phổ biến nhất ("lấy trang thứ X") —
nhưng với `m` (offset) lớn, dù có index hỗ trợ `ORDER BY`, database vẫn phải **duyệt qua và đếm**
đủ `m` dòng đầu tiên (dùng index để duyệt theo đúng thứ tự, nhưng vẫn phải đi qua từng dòng một
để đếm) trước khi mới bắt đầu trả về `n` dòng thực sự cần — càng vào các trang sau, chi phí "đếm
bỏ qua" càng tăng, dù kết quả cuối cùng chỉ có `n` dòng.

## Concept

**Keyset Pagination** (còn gọi cursor-based pagination) thay `OFFSET m` bằng điều kiện
`WHERE cột_sắp_xếp > giá_trị_cuối_cùng_của_trang_trước` — index có thể **nhảy thẳng** tới đúng
vị trí đó (`O(log n)`, giống tra cứu bình thường đã học ở Chapter 03), không cần đếm qua từng
dòng phía trước. **Covering Index** là một index chứa **đủ mọi cột** một truy vấn cần — cho phép
database trả lời hoàn toàn từ chính index đó (**Index Only Scan**), không cần truy cập thêm vào
bảng dữ liệu gốc (**heap**) để lấy các cột còn thiếu.

## Why?

Bản chất khác biệt giữa `OFFSET` và keyset pagination nằm ở **loại thao tác** trên B-tree:
`OFFSET` yêu cầu "đi qua đúng m dòng rồi dừng" — về bản chất là một phép **duyệt tuần tự có đếm**
trên cấu trúc index, độ phức tạp tỉ lệ thuận với `m`. Điều kiện `WHERE cột > giá_trị` là một phép
**tra cứu trực tiếp** (tương đương tìm kiếm nhị phân trong cây, Chapter 03) — độ phức tạp
`O(log n)`, hoàn toàn không phụ thuộc vào việc đây là trang thứ mấy. Covering Index tận dụng
thực tế: nếu index đã chứa đủ dữ liệu cần trả về, việc quay lại đọc bảng gốc (một bước I/O bổ
sung, thường ngẫu nhiên — đã học ở Chapter 03) là hoàn toàn không cần thiết.

## How?

```sql
-- OFFSET (cham voi trang xa)
SELECT id FROM big_orders ORDER BY id LIMIT 10 OFFSET 1000000;

-- Keyset pagination (nhanh, khong phu thuoc "trang thu may")
SELECT id FROM big_orders WHERE id > 1000000 ORDER BY id LIMIT 10;

-- Covering index: index chi can chua dung cot query can
CREATE INDEX idx_email_only ON big_orders(customer_email);
SELECT customer_email FROM big_orders WHERE customer_email = '...'; -- Index Only Scan
```

## Visualization

```
OFFSET 1,000,000:

  B-tree: [bat dau] ──duyet qua 1,000,000 dong (dem, bo qua)──► [lay 10 dong tiep theo]
  Chi phi ti le voi OFFSET, du dung index

Keyset (WHERE id > 1000000):

  B-tree: [tim TRUC TIEP vi tri id=1000000] ──O(log n)──► [lay 10 dong tiep theo NGAY]
  Chi phi KHONG doi, bat ke "trang thu may"

Covering Index / Index Only Scan:

  Truy van CHI can cot da co san trong INDEX:
    Index ──► TRA VE TRUC TIEP, KHONG can vao bang goc (Heap Fetches: 0)

  Truy van can THEM cot KHONG co trong index:
    Index ──► tim vi tri ──► PHAI vao Heap doc them cot con thieu (Index Scan, khong phai Index Only)
```

## Example

**Keyset pagination vs OFFSET:**

```sql
EXPLAIN ANALYZE SELECT id FROM big_orders ORDER BY id LIMIT 10 OFFSET 1000000;
```

```
Limit (actual time=109.794..109.795 rows=10 loops=1)
  ->  Index Only Scan using big_orders_pkey on big_orders
        (actual time=0.704..94.074 rows=1000010 loops=1)
Execution Time: 109.817 ms
```

```sql
EXPLAIN ANALYZE SELECT id FROM big_orders WHERE id > 1000000 ORDER BY id LIMIT 10;
```

```
Limit (actual time=0.019..0.019 rows=10 loops=1)
  ->  Index Only Scan using big_orders_pkey on big_orders
        Index Cond: (id > 1000000)
        (actual time=0.018..0.019 rows=10 loops=1)
Execution Time: 0.031 ms
```

Cả hai đều dùng `Index Only Scan` trên cùng một index — nhưng `OFFSET` phải **đi qua thực sự**
`1,000,010` dòng (`rows=1000010` trong node con) trước khi `Limit` cắt lấy 10 dòng cuối, còn
`WHERE id > 1000000` **nhảy thẳng** tới đúng vị trí, chỉ xử lý đúng 10 dòng cần thiết.

**Covering Index / Index Only Scan:**

```sql
VACUUM ANALYZE big_orders; -- can thiet de Visibility Map cap nhat, Index Only Scan moi tranh Heap Fetch
EXPLAIN ANALYZE SELECT customer_email FROM big_orders WHERE customer_email = 'user99999@example.com';
```

```
Index Only Scan using idx_big_orders_email on big_orders (actual time=0.012..0.013 rows=4 loops=1)
  Heap Fetches: 0
Execution Time: 0.035 ms
```

```sql
EXPLAIN ANALYZE SELECT * FROM big_orders WHERE customer_email = 'user99999@example.com';
```

```
Index Scan using idx_big_orders_email on big_orders (actual time=0.011..0.013 rows=4 loops=1)
Execution Time: 0.027 ms
```

Truy vấn chỉ cần `customer_email` (**đúng cột đã có trong index**) đạt `Heap Fetches: 0` — trả
lời hoàn toàn từ index, không cần đọc bảng gốc. Truy vấn `SELECT *` (cần thêm `amount`, không có
trong index) buộc phải dùng `Index Scan` thường (không phải Index **Only** Scan) — tra index để
tìm vị trí, rồi **vẫn phải** ghé qua bảng gốc lấy nốt cột còn thiếu.

## Deep Dive

**Vì sao ví dụ `Index Only Scan` ban đầu (trước `VACUUM`) vẫn báo `Heap Fetches: 4` dù đúng tên
gọi là "Only"?** Đây là một chi tiết implementation quan trọng của PostgreSQL liên quan trực
tiếp tới MVCC (Chapter 08): mỗi dòng vật lý có thể có nhiều phiên bản (`xmin`/`xmax`), và bản
thân index **không lưu thông tin "phiên bản này có visible với transaction hiện tại hay
không"** — chỉ bảng dữ liệu gốc (heap) mới biết chắc điều đó. PostgreSQL duy trì một cấu trúc
phụ trợ gọi là **Visibility Map** (bản đồ khả kiến) — đánh dấu các trang dữ liệu mà **mọi**
transaction đang hoạt động đều nhìn thấy giống nhau (không có phiên bản đang tranh chấp) — chỉ
với các trang đã được đánh dấu trong Visibility Map, PostgreSQL mới dám bỏ qua hoàn toàn việc ghé
heap. `VACUUM` (đã nhắc ở Chapter 08) chính là bước cập nhật Visibility Map đó — đây là lý do sau
khi chạy `VACUUM ANALYZE`, `Heap Fetches` giảm từ `4` xuống `0`.

## Engineering Insight

**Trong thực tế, khi nào keyset pagination không áp dụng được, và cần cân nhắc gì?** Keyset
pagination yêu cầu điều kiện sắp xếp phải dựa trên một cột (hoặc tổ hợp cột) có **thứ tự tuyến
tính rõ ràng và duy nhất** (thường là khoá chính tăng dần) — nó **không** hỗ trợ trực tiếp việc
"nhảy tới trang bất kỳ" (ví dụ "đi thẳng tới trang 500" mà không cần biết giá trị cuối trang
499) — chỉ hỗ trợ tốt mô hình "trang tiếp theo/trang trước" (giống hầu hết infinite-scroll hiện
đại, không giống UI phân trang truyền thống với số trang bấm được). Với UI **thực sự cần** nhảy
tới số trang cụ thể (ít phổ biến hơn trong thiết kế hiện đại, nhưng vẫn tồn tại ở một số hệ thống
quản trị nội bộ), có thể cần kết hợp cả hai chiến lược: `OFFSET` cho các trang gần đầu (chi phí
còn chấp nhận được), hoặc chấp nhận đánh đổi UX (chỉ cho phép "trang tiếp theo") để đổi lấy hiệu
năng nhất quán bất kể dữ liệu lớn tới đâu.

## Historical Note

Vấn đề hiệu năng của `OFFSET` với giá trị lớn là một trong những "cạm bẫy kinh điển" được nhận
diện rộng rãi trong cộng đồng database từ rất sớm — nhiều thư viện/framework ORM hiện đại (bao
gồm Spring Data JPA, Phase 6) cung cấp sẵn hỗ trợ cho keyset pagination (`Slice`, cursor-based
API) như một lựa chọn thay thế tường minh cho `Pageable` truyền thống dựa trên offset, phản ánh
mức độ phổ biến và tầm quan trọng thực tế của vấn đề này.

## Myth vs Reality

- **Myth:** "Đã có index cho cột `ORDER BY` là đủ để phân trang luôn nhanh, bất kể `OFFSET` lớn
  tới đâu."
  **Reality:** Đã chứng minh bằng thực nghiệm — index giúp `ORDER BY` nhanh, nhưng `OFFSET` vẫn
  phải duyệt qua đúng số dòng bị bỏ qua, độc lập với việc có index hay không.

- **Myth:** "`Index Only Scan` trong tên gọi nghĩa là luôn luôn không cần đọc bảng gốc."
  **Reality:** Đã chứng minh bằng thực nghiệm — vẫn có thể có `Heap Fetches > 0` nếu Visibility
  Map chưa cập nhật (cần `VACUUM`).

## Common Mistakes

- **Dùng `OFFSET` cho phân trang không giới hạn (ví dụ cho phép người dùng cào dữ liệu tới trang
  rất xa)** — hiệu năng suy giảm nghiêm trọng, có thể trở thành lỗ hổng DoS tiềm ẩn.
- **Viết `SELECT *` khi chỉ cần vài cột đã có sẵn trong index** — bỏ lỡ cơ hội `Index Only Scan`,
  buộc phải ghé heap không cần thiết.
- **Không chạy `VACUUM`/`autovacuum` định kỳ** — không chỉ gây table bloat (Chapter 08), còn làm
  giảm hiệu quả của `Index Only Scan` do Visibility Map không cập nhật.

## Best Practices

- Ưu tiên keyset pagination (`WHERE cột > giá_trị_cuối`) thay vì `OFFSET` cho API có khả năng
  phân trang sâu hoặc dữ liệu lớn.
- Thiết kế index bao phủ (`covering index`) cho các truy vấn đọc thường xuyên, chỉ cần vài cột
  cụ thể — tận dụng `Index Only Scan`.
- Đảm bảo `autovacuum` hoạt động đúng, đặc biệt cho bảng có tần suất ghi cao, để `Index Only
  Scan` phát huy hết hiệu quả.
- Chỉ `SELECT` đúng những cột thực sự cần, tránh `SELECT *` khi có thể tận dụng covering index.

## Debug Checklist

- [ ] Phân trang chậm dần khi người dùng cào tới các trang xa? → nghi ngờ hàng đầu là `OFFSET`
      lớn, cân nhắc chuyển sang keyset pagination.
- [ ] `EXPLAIN ANALYZE` báo `Index Only Scan` nhưng vẫn chậm? → kiểm tra `Heap Fetches` — nếu
      lớn hơn 0, chạy `VACUUM ANALYZE` để cập nhật Visibility Map.
- [ ] Không chắc một truy vấn có tận dụng được covering index không? → kiểm tra mọi cột trong
      `SELECT`/`WHERE` có đều nằm trong index đang dùng không.

## Summary

`OFFSET` với giá trị lớn chậm vì database phải duyệt và đếm qua đúng số dòng bị bỏ qua, bất kể
có index — đã chứng minh bằng thực nghiệm: `OFFSET 1000000` tốn 109.817ms, trong khi keyset
pagination (`WHERE id > 1000000`) cho cùng kết quả chỉ tốn 0.031ms (~3,500 lần nhanh hơn), vì nó
tra cứu trực tiếp (`O(log n)`) thay vì duyệt tuần tự có đếm. Covering Index cho phép
`Index Only Scan` — trả lời truy vấn hoàn toàn từ index, không cần đọc bảng gốc (`Heap Fetches:
0`) — nhưng chỉ đạt hiệu quả này khi Visibility Map đã cập nhật (`VACUUM`, liên quan trực tiếp
tới MVCC, Chapter 08); nếu chưa, `Index Only Scan` vẫn phải ghé heap (`Heap Fetches > 0`) dù tên
gọi có "Only".

## Interview Questions

- Vì sao `OFFSET` với giá trị lớn lại chậm, dù đã có index đầy đủ? Đề xuất cách tối ưu.

**Senior**

- Covering Index là gì? Khi nào `Index Only Scan` thực sự tránh được việc đọc bảng gốc?
- Keyset pagination có nhược điểm gì so với `OFFSET` truyền thống? Khi nào không áp dụng được?

## Exercises

- [ ] Chạy lại các câu lệnh ở Example trên một PostgreSQL thật với bảng đủ lớn, xác nhận kết quả
      tương tự.
- [ ] Đo thời gian `OFFSET` ở nhiều mức độ khác nhau (10,000 / 100,000 / 1,000,000), vẽ biểu đồ
      thời gian tăng tuyến tính theo offset.
- [ ] Tạo một composite covering index `(customer_email) INCLUDE (amount)`, chạy lại
      `SELECT customer_email, amount FROM ... WHERE customer_email = ...`, xác nhận đạt được
      `Index Only Scan` với `Heap Fetches: 0`.

## Cheat Sheet

| Kỹ thuật | Vấn đề giải quyết | Cách dùng |
| --- | --- | --- |
| Keyset Pagination | `OFFSET` lớn chậm | `WHERE cột_sort > giá_trị_cuối_trang_trước ORDER BY cột_sort LIMIT n` |
| Covering Index | Truy vấn phải ghé heap không cần thiết | Index chứa đủ mọi cột truy vấn cần |
| `VACUUM ANALYZE` | Visibility Map chưa cập nhật, `Index Only Scan` vẫn cần Heap Fetches | Chạy định kỳ (thường qua `autovacuum`) |

## References

- PostgreSQL Documentation — Index-Only Scans and Covering Indexes.
- Markus Winand — "SQL Performance Explained" (chương về pagination, keyset vs offset).
