---
tags:
  - SQL
  - Index
  - PostgreSQL
---

# Index (B-tree) và Selectivity

> Phase: Phase 7 — Database
> Chapter slug: `index`

## Metadata

```yaml
Chapter: Index
Phase: Phase 7 — Database
Difficulty: ★★★★
Importance: ★★★★★
Interview Frequency: 90%
Prerequisites:
  - Chapter 01 — SQL
  - Phase 2, Chapter 10 — TreeMap
Used Later:
  - Chapter 04 — Execution Plan
  - Chapter 09 — Optimization
Estimated Reading: 18 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: Phase 2, Chapter 10 (TreeMap) — cấu trúc cây cân bằng cho phép tìm kiếm
> `O(log n)` thay vì duyệt tuần tự `O(n)`. Index database áp dụng **chính xác nguyên lý đó** cho
> dữ liệu trên đĩa.

Tìm một dòng cụ thể trong bảng **2 triệu dòng**, trước và sau khi tạo index:

```sql
EXPLAIN ANALYZE SELECT * FROM big_orders WHERE customer_email = 'user12345@example.com';
```

**Không có index:**

```
Parallel Seq Scan on big_orders (actual time=58.251..126.525 rows=1 loops=3)
  Rows Removed by Filter: 666665
Execution Time: 131.365 ms
```

**Sau khi `CREATE INDEX idx_big_orders_email ON big_orders(customer_email)`:**

```
Index Scan using idx_big_orders_email on big_orders (actual time=0.015..0.023 rows=4 loops=1)
Execution Time: 0.038 ms
```

Cùng dữ liệu, cùng câu truy vấn — **131.365ms xuống còn 0.038ms**, nhanh hơn khoảng **3,400
lần**, chỉ nhờ một câu `CREATE INDEX` duy nhất.

## Interview Question (Central)

> Index trong database là gì? Nó hoạt động dựa trên cấu trúc dữ liệu nào, và vì sao không phải
> lúc nào cũng nên tạo index cho mọi cột?

## Objectives

- [ ] Hiểu Index B-tree là cấu trúc dữ liệu cây cân bằng, giống `TreeMap` (Phase 2) nhưng tối ưu
      cho việc đọc/ghi trên đĩa
- [ ] Tự tay chứng minh bằng thực nghiệm tốc độ tăng vọt (hàng nghìn lần) khi truy vấn có index
      phù hợp trên bảng lớn
- [ ] Tự tay chứng minh bằng thực nghiệm: có index không đảm bảo được dùng — planner **bỏ qua**
      index khi **selectivity** thấp (điều kiện khớp quá nhiều dòng)

## Prerequisites

- Chapter 01 — hiểu SQL là khai báo, engine tự quyết định cách thực thi (bao gồm việc có dùng
  index hay không).
- Phase 2, Chapter 10 — hiểu cấu trúc cây B-tree/Red-Black tree qua `TreeMap`, nền tảng lý
  thuyết của Index B-tree.

## Used Later

- **Chapter 04 (Execution Plan)** — đọc hiểu chi tiết `EXPLAIN ANALYZE`, công cụ đã dùng xuyên
  suốt chapter này.
- **Chapter 09 (Optimization)** — thiết kế index đúng là kỹ thuật tối ưu hiệu năng quan trọng
  nhất trong thực tế.

## Problem

Không có index, tìm một dòng cụ thể trong bảng buộc database phải **duyệt tuần tự** (Seq Scan)
qua **mọi** dòng, kiểm tra từng dòng có khớp điều kiện hay không — độ phức tạp `O(n)`, với bảng
càng lớn, thời gian tìm kiếm càng tăng tuyến tính. Với bảng hàng triệu dòng (phổ biến trong thực
tế production), điều này khiến ngay cả một truy vấn tìm-một-dòng đơn giản cũng trở nên chậm
không thể chấp nhận được.

## Concept

**Index** (mặc định là **B-tree** trong hầu hết database) là một cấu trúc dữ liệu **riêng biệt**
với bảng dữ liệu gốc, lưu trữ giá trị của (các) cột được đánh index cùng con trỏ trỏ tới vị trí
dòng dữ liệu thật — được tổ chức theo cây cân bằng, cho phép tìm kiếm với độ phức tạp
`O(log n)` thay vì `O(n)`. Database engine tự động **quyết định** có dùng index hay không dựa
trên **selectivity** (độ chọn lọc — tỉ lệ số dòng khớp điều kiện trên tổng số dòng): điều kiện
càng **chọn lọc cao** (khớp ít dòng), lợi ích dùng index càng lớn.

## Why?

B-tree được chọn làm cấu trúc index mặc định vì nó tối ưu cho **I/O đĩa**: mỗi node của B-tree
tương ứng một trang dữ liệu (page) trên đĩa, cây được thiết kế **thấp và rộng** (mỗi node chứa
nhiều key, không phải cây nhị phân đơn giản) để giảm thiểu số lần đọc đĩa cần thiết khi tìm kiếm
(mỗi lần đọc đĩa tốn kém hơn nhiều so với so sánh trong bộ nhớ). Việc engine tự quyết định dùng
hay không dùng index (dựa trên selectivity) phản ánh đúng bản chất khai báo của SQL (Chapter 01)
— index chỉ là một **công cụ có sẵn**, không phải một **chỉ thị bắt buộc**; nếu dùng index thực
sự chậm hơn (selectivity thấp, phải nhảy ngẫu nhiên qua quá nhiều trang đĩa so với đọc tuần tự),
engine hoàn toàn có quyền bỏ qua nó.

## How?

```sql
CREATE INDEX idx_big_orders_email ON big_orders(customer_email);
-- Sau do, query WHERE customer_email = '...' co the dung index nay
EXPLAIN ANALYZE SELECT * FROM big_orders WHERE customer_email = 'user12345@example.com';
```

## Visualization

```
Khong co Index (Seq Scan):              Co Index (Index Scan), selectivity CAO:

  Duyet TUAN TU qua het 2 trieu dong       B-tree: [tang goc]
  ──► ──► ──► ──► ... ──► TIM THAY              /    |    \
  (kiem tra TUNG dong mot)                  [nhanh] [nhanh] [nhanh]
                                                        │
  O(n) = 2,000,000 buoc                          [la: tro toi dong that]
                                              O(log n) ~ 21 buoc (log2 2,000,000)

Co Index nhung selectivity THAP (is_paid ~50%):

  Dung index -> phai NHAY qua GAN 1 TRIEU vi tri ngau nhien tren dia
  So voi Seq Scan -> doc TUAN TU (nhanh hon vi khong "nhay lung tung")
  => PLANNER TU CHOI dung index, chon Seq Scan
```

## Example

**Selectivity cao — index thắng áp đảo:**

```sql
EXPLAIN ANALYZE SELECT * FROM big_orders WHERE customer_email = 'user12345@example.com';
```

Không có index (2 triệu dòng):

```
Parallel Seq Scan on big_orders (actual time=58.251..126.525 rows=1 loops=3)
  Filter: ((customer_email)::text = 'user12345@example.com'::text)
  Rows Removed by Filter: 666665
Execution Time: 131.365 ms
```

Sau khi tạo index:

```
Index Scan using idx_big_orders_email on big_orders (actual time=0.015..0.023 rows=4 loops=1)
  Index Cond: ((customer_email)::text = 'user12345@example.com'::text)
Execution Time: 0.038 ms
```

**Selectivity thấp — planner chủ động bỏ qua index:**

```sql
CREATE INDEX idx_big_orders_is_paid ON big_orders(is_paid);
EXPLAIN ANALYZE SELECT * FROM big_orders WHERE is_paid = true;
```

Kết quả thật (dù index đã tồn tại):

```
Seq Scan on big_orders (actual time=0.009..101.275 rows=999320 loops=1)
  Filter: is_paid
  Rows Removed by Filter: 1000680
Execution Time: 115.512 ms
```

Với điều kiện chọn lọc **cao** (`customer_email = '...'`, chỉ khớp 4/2,000,000 dòng — selectivity
cực thấp về tỉ lệ, nghĩa là "rất chọn lọc"), index mang lại tốc độ **gấp ~3,400 lần**. Nhưng với
điều kiện chọn lọc **thấp** (`is_paid = true`, khớp gần **50%** tổng số dòng), planner **hoàn
toàn bỏ qua** index dù nó tồn tại (dòng `QUERY PLAN` không hề nhắc tới
`idx_big_orders_is_paid`) — tự chọn `Seq Scan`, vì với gần một nửa số dòng khớp điều kiện, đọc
tuần tự thực sự **nhanh hơn** việc nhảy ngẫu nhiên qua từng vị trí trong index.

## Deep Dive

**Vì sao "nhảy ngẫu nhiên qua index" lại chậm hơn "đọc tuần tự" khi selectivity thấp, dù về mặt
lý thuyết `O(log n)` luôn nhanh hơn `O(n)`?** Đây là điểm dễ hiểu lầm nhất về index: độ phức tạp
`O(log n)` chỉ đo **số lần tìm kiếm trong cây** để định vị **một** dòng — nhưng sau khi định vị,
engine còn phải **đọc dòng dữ liệu thật** từ bảng gốc (vì index thường chỉ lưu con trỏ, không
lưu toàn bộ dữ liệu dòng, trừ trường hợp index bao phủ — covering index). Nếu cần đọc **gần một
triệu** dòng dữ liệu thật (selectivity thấp), và các dòng đó nằm **rải rác không theo thứ tự**
trên đĩa (vì thứ tự trong index không nhất thiết khớp thứ tự vật lý trên đĩa của bảng gốc), mỗi
lần đọc là một **I/O ngẫu nhiên** riêng biệt (random I/O — chậm hơn đáng kể so với sequential
I/O trên hầu hết thiết bị lưu trữ, dù SSD hiện đại đã thu hẹp khoảng cách này). Với selectivity
đủ thấp, tổng chi phí của hàng loạt I/O ngẫu nhiên đó **vượt quá** chi phí đơn giản đọc tuần tự
toàn bộ bảng — đây chính xác là lý do planner (dựa trên thống kê phân bố dữ liệu nó tự thu thập,
`ANALYZE`) chủ động chọn `Seq Scan` thay vì `Index Scan` trong ví dụ `is_paid`.

## Engineering Insight

**Vì sao "cứ thêm index cho mọi cột hay dùng trong WHERE" là một chiến lược sai lầm trong thực
tế production, dù index tăng tốc đọc?** Mỗi index không phải "miễn phí" — nó phải được **cập
nhật đồng bộ** với mọi thao tác `INSERT`/`UPDATE`/`DELETE` trên bảng gốc (mỗi thay đổi dữ liệu
phải cập nhật cả cấu trúc B-tree tương ứng), làm **chậm** các thao tác ghi, và tốn thêm **dung
lượng đĩa** đáng kể (một index lớn có thể tốn dung lượng tương đương hoặc hơn cả bảng gốc). Với
bảng có tần suất ghi cao, việc thêm index không cân nhắc kỹ (đặc biệt cho các cột selectivity
thấp, nơi index hiếm khi thực sự được dùng như ví dụ `is_paid`) là đánh đổi **lỗ** — trả chi phí
ghi chậm hơn và tốn đĩa, đổi lấy lợi ích đọc gần như không đáng kể (vì planner sẽ bỏ qua index đó
phần lớn thời gian). Quyết định đúng đắn luôn dựa trên đo lường thực tế (selectivity của điều
kiện, tần suất truy vấn so với tần suất ghi), không phải nguyên tắc "càng nhiều index càng tốt".

## Historical Note

B-tree (Bayer & McCreight, 1972) được thiết kế ban đầu chính xác cho bài toán tổ chức dữ liệu
trên thiết bị lưu trữ có độ trễ truy cập cao (ổ đĩa từ thời kỳ đó) — ý tưởng cốt lõi "giảm số
lần truy cập đĩa bằng cách làm cây thấp và rộng" vẫn giữ nguyên giá trị dù công nghệ lưu trữ đã
thay đổi nhiều (SSD, NVMe) — chi phí truy cập ngẫu nhiên vẫn cao hơn truy cập tuần tự, dù khoảng
cách đã thu hẹp đáng kể so với ổ đĩa từ truyền thống.

## Myth vs Reality

- **Myth:** "Có index luôn nhanh hơn không có index, bất kể điều kiện truy vấn là gì."
  **Reality:** Đã chứng minh bằng thực nghiệm — với selectivity thấp (`is_paid = true`, khớp
  ~50% dữ liệu), planner chủ động bỏ qua index đã tồn tại, chọn Seq Scan vì thực sự nhanh hơn.

- **Myth:** "Nên tạo index cho mọi cột xuất hiện trong mệnh đề `WHERE`, càng nhiều càng an
  toàn."
  **Reality:** Xem Engineering Insight — mỗi index có chi phí ghi và dung lượng thật, index
  selectivity thấp hiếm khi được dùng nhưng vẫn phải trả chi phí duy trì.

## Common Mistakes

- **Tạo index cho cột có selectivity thấp (boolean, hoặc cột chỉ có vài giá trị phân biệt) kỳ
  vọng tăng tốc** — planner thường bỏ qua, lãng phí chi phí duy trì index.
- **Ngạc nhiên/hoang mang khi thấy `EXPLAIN` không dùng index dù đã tạo** — đây là hành vi đúng
  đắn của planner với selectivity thấp, không phải lỗi.
- **Không đo lường selectivity thực tế trước khi quyết định tạo index** — nên kiểm tra tỉ lệ
  giá trị phân biệt/tổng số dòng trước khi đầu tư vào index cho một cột.

## Best Practices

- Tạo index cho các cột có **selectivity cao** (nhiều giá trị phân biệt, mỗi điều kiện chỉ khớp
  một phần nhỏ dữ liệu) — điển hình: khoá ngoại, email, mã đơn hàng.
- Tránh tạo index cho cột selectivity thấp (boolean, trạng thái chỉ có vài giá trị) trừ khi có
  lý do cụ thể (ví dụ kết hợp composite index với cột khác).
- Luôn dùng `EXPLAIN ANALYZE` để xác nhận index thực sự được dùng và mang lại lợi ích, không chỉ
  dựa vào giả định.
- Cân nhắc chi phí ghi khi thêm index cho bảng có tần suất `INSERT`/`UPDATE` cao.

## Debug Checklist

- [ ] Truy vấn chậm dù đã tạo index đúng cột? → chạy `EXPLAIN ANALYZE`, kiểm tra planner có
      thực sự dùng index không — nếu không, kiểm tra selectivity của điều kiện.
- [ ] Không chắc một index có đang thực sự hữu ích không? → so sánh selectivity (tỉ lệ dòng khớp
      điều kiện), selectivity càng thấp thì index càng ít khả năng được dùng.
- [ ] Tốc độ ghi (`INSERT`/`UPDATE`) giảm sau khi thêm nhiều index? → bình thường, mỗi index
      thêm chi phí đồng bộ hoá cấu trúc — cân nhắc loại bỏ index ít selectivity/ít dùng.

## Summary

Index (B-tree) là cấu trúc cây cân bằng riêng biệt, cho phép tìm kiếm `O(log n)` thay vì `O(n)`
duyệt tuần tự — cùng nguyên lý với `TreeMap` (Phase 2). Đã chứng minh bằng thực nghiệm: với điều
kiện selectivity cao (khớp 4/2,000,000 dòng), index tăng tốc từ 131.365ms xuống 0.038ms (~3,400
lần). Nhưng index **không phải lúc nào cũng được dùng** — với selectivity thấp (khớp ~50% dữ
liệu), planner chủ động bỏ qua index đã tồn tại, chọn Seq Scan vì đọc tuần tự thực sự nhanh hơn
nhảy ngẫu nhiên qua index (random I/O tốn kém hơn sequential I/O). Mỗi index có chi phí thật (làm
chậm ghi, tốn dung lượng) — chỉ nên tạo cho cột có selectivity cao, xác nhận hiệu quả qua
`EXPLAIN ANALYZE` thay vì giả định.

## Interview Questions

- Index trong database là gì? Nó hoạt động dựa trên cấu trúc dữ liệu nào?
- Vì sao không phải lúc nào cũng nên tạo index cho mọi cột?

**Senior**

- Giải thích vì sao planner có thể bỏ qua một index đã tồn tại. Selectivity là gì, và nó ảnh
  hưởng quyết định của planner như thế nào?
- So sánh chi phí đọc (random I/O qua index) và chi phí đọc tuần tự (Seq Scan) — khi nào cái nào
  thắng?

## Exercises

- [ ] Chạy lại các câu lệnh ở Example trên một PostgreSQL thật với bảng đủ lớn (>1 triệu dòng),
      xác nhận kết quả tương tự.
- [ ] Thử tạo composite index (nhiều cột) cho `big_orders(is_paid, customer_email)`, chạy lại
      truy vấn kết hợp cả hai điều kiện, quan sát planner có dùng index không.
- [ ] Dùng `SELECT count(*), count(DISTINCT customer_email) FROM big_orders;` để tự tính
      selectivity thực tế của cột `customer_email`, so sánh với `is_paid`.

## Cheat Sheet

| Selectivity | Ví dụ | Planner có dùng index? |
| --- | --- | --- |
| Rất cao (gần như duy nhất) | Email, mã đơn hàng, khoá chính | Có, hiệu quả rõ rệt |
| Trung bình | Thành phố (vài chục giá trị) | Tuỳ tình huống, thường có |
| Thấp (~50%, ít giá trị phân biệt) | Boolean (`is_paid`), trạng thái chỉ 2-3 giá trị | Thường KHÔNG dùng |

**Chi phí index:** làm chậm `INSERT`/`UPDATE`/`DELETE`, tốn dung lượng đĩa — chỉ tạo khi lợi ích
đọc vượt trội chi phí này.

## References

- PostgreSQL Documentation — Indexes.
- Bayer, R.; McCreight, E. — "Organization and Maintenance of Large Ordered Indexes" (1972).
