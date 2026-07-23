---
tags:
  - SQL
  - MVCC
  - PostgreSQL
---

# MVCC (Multi-Version Concurrency Control)

> Phase: Phase 7 — Database
> Chapter slug: `mvcc`

## Metadata

```yaml
Chapter: MVCC
Phase: Phase 7 — Database
Difficulty: ★★★★★
Importance: ★★★★★
Interview Frequency: 70%
Prerequisites:
  - Chapter 06 — Isolation
  - Chapter 07 — Lock
Used Later:
  - Chapter 09 — Optimization
Estimated Reading: 18 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 07](07-lock.md) — một `UPDATE` giữ row lock khiến một `SELECT ... FOR
> UPDATE` khác **thực sự bị block** (~3.2 giây chờ đợi). Nhưng thử điều tương tự với một
> `SELECT` **thông thường** (không `FOR UPDATE`) — điều gì xảy ra?

```sql
-- Session A: UPDATE (giu row lock) roi pg_sleep(2)
BEGIN;
UPDATE customers SET city = 'Version3_Uncommitted' WHERE email = 'mvcc@example.com';
SELECT pg_sleep(2);

-- Session B: SELECT THUONG, chay GAN NHU dong thoi
SELECT clock_timestamp();
SELECT city FROM customers WHERE email = 'mvcc@example.com';
SELECT clock_timestamp();
```

Kết quả thật (Session B):

```
        clock_timestamp
-------------------------------
 2026-07-23 15:56:11.432749+00
   city
----------
 Version2
        clock_timestamp
-------------------------------
 2026-07-23 15:56:11.844004+00
```

Chênh lệch chỉ **~0.4 giây** — Session B **không hề bị block**, dù Session A đang giữ row lock
và `UPDATE` chưa commit. `SELECT` trả về ngay giá trị **cũ đã commit** (`'Version2'`), không
phải giá trị mới chưa commit.

## Interview Question (Central)

> MVCC là gì? Vì sao nó cho phép "readers không bao giờ block writers, writers không bao giờ
> block readers"?

## Objectives

- [ ] Hiểu MVCC là cơ chế lưu **nhiều phiên bản** của cùng một dòng dữ liệu, thay vì khoá để bảo
      vệ một bản duy nhất
- [ ] Tự tay chứng minh bằng thực nghiệm: `SELECT` thông thường không bị block bởi `UPDATE`
      đang giữ row lock — trái ngược với `SELECT ... FOR UPDATE` (Chapter 07)
- [ ] Tự tay quan sát các cột hệ thống `xmin`/`xmax` — bằng chứng trực tiếp rằng `UPDATE` tạo
      ra một **dòng vật lý mới**, không sửa tại chỗ

## Prerequisites

- Chapter 06 — hiểu Isolation Level, MVCC chính là cơ chế PostgreSQL dùng để hiện thực hoá phần
  lớn các mức isolation.
- Chapter 07 — hiểu row lock, để đối chiếu trực tiếp: lock bảo vệ **ghi**, MVCC loại bỏ nhu cầu
  lock cho **đọc**.

## Used Later

- **Chapter 09 (Optimization)** — hiểu MVCC giúp giải thích các hiện tượng như "bảng phình to"
  (bloat) cần `VACUUM` định kỳ.

## Problem

Với cơ chế khoá truyền thống (lock-based concurrency control), để đảm bảo một reader không đọc
phải dữ liệu "nửa vời" trong lúc writer đang sửa, cách đơn giản nhất là **khoá** dòng đó khi
writer đang thao tác, buộc reader phải **chờ**. Nhưng điều này có nghĩa **một writer duy nhất**
có thể chặn đứng **hàng loạt reader** — với ứng dụng có tỉ lệ đọc cao hơn ghi rất nhiều (phổ biến
trong thực tế), đây là một điểm nghẽn hiệu năng nghiêm trọng.

## Concept

**MVCC (Multi-Version Concurrency Control)** giải quyết vấn đề này bằng cách **không** sửa dữ
liệu tại chỗ — mỗi `UPDATE` tạo ra một **dòng vật lý hoàn toàn mới** (phiên bản mới), giữ nguyên
dòng cũ (đánh dấu "đã cũ", không xoá ngay). Mỗi dòng vật lý mang hai cột hệ thống ẩn: **`xmin`**
(ID của transaction đã **tạo ra** phiên bản này) và **`xmax`** (ID của transaction đã **thay
thế/xoá** phiên bản này, `0` nếu vẫn là phiên bản mới nhất). Một transaction, khi đọc dữ liệu,
chỉ nhìn thấy phiên bản **phù hợp với snapshot của chính nó** (dựa trên `xmin`/`xmax` so với
transaction ID hiện tại) — hoàn toàn không cần khoá để đọc đúng phiên bản.

## Why?

Vì mỗi phiên bản dữ liệu là **bất biến** sau khi tạo (không ai sửa một dòng vật lý đã tồn tại,
chỉ tạo phiên bản mới), một reader có thể **luôn** đọc phiên bản phù hợp với mình mà **không cần
khoá gì cả** — không có rủi ro đọc phải dữ liệu "nửa vời" (writer không sửa tại chỗ, chỉ thêm
phiên bản mới), và không cần chờ writer "xong việc". Writer cũng không cần chờ reader "đọc xong"
vì reader đang đọc phiên bản **cũ, riêng biệt** không bị writer động tới. Đây chính là lý do
"readers không block writers, writers không block readers" — cả hai hoạt động trên các **phiên
bản dữ liệu tách biệt**, chỉ **writer-writer** (hai bên cùng cố tạo phiên bản mới cho cùng một
dòng logic) mới thực sự cần khoá (đúng nội dung đã học ở Chapter 07).

## How?

```sql
-- Xem cot he thong xmin/xmax
SELECT xmin, xmax, * FROM customers WHERE id = 1;

-- INSERT: tao phien ban DAU TIEN
INSERT INTO customers (...) VALUES (...); -- xmin = transaction hien tai, xmax = 0

-- UPDATE: tao phien ban MOI, KHONG sua tai cho
UPDATE customers SET city = '...' WHERE id = 1;
-- -> dong CU: xmax = transaction hien tai (danh dau "da cu")
-- -> dong MOI: xmin = transaction hien tai, xmax = 0
```

## Visualization

```
UPDATE trong MVCC (KHONG sua tai cho):

  Truoc UPDATE:
    Dong vat ly #1: xmin=771, xmax=0    <- phien ban HIEN TAI, city='Version1'

  UPDATE ... SET city='Version2':
    Dong vat ly #1: xmin=771, xmax=772  <- danh dau DA CU (bi thay the boi transaction 772)
    Dong vat ly #2: xmin=772, xmax=0    <- phien ban MOI, city='Version2'  <-- HIEN TAI

Reader vs Writer (KHONG block nhau):

  Writer (transaction 773): UPDATE, giu row lock, CHUA commit
       │  (tao phien ban #3, xmax cua #2 = 773 nhung CHUA "chinh thuc" vi chua commit)
       │
  Reader (transaction 774): SELECT thuong
       │  KHONG can khoa gi ca - doc THANG phien ban #2 (xmax=0 truoc khi 773 commit)
       ▼
  Reader TRA VE NGAY, thay du lieu CU (dung, vi 773 chua commit)
```

## Example

**Cột hệ thống `xmin`/`xmax` — bằng chứng versioning thật:**

```sql
INSERT INTO customers (name, email, city) VALUES ('MVCC Demo', 'mvcc@example.com', 'Version1');
SELECT xmin, xmax, city FROM customers WHERE email = 'mvcc@example.com';
```

```
 xmin | xmax |   city
------+------+----------
  771 |    0 | Version1
```

```sql
BEGIN;
UPDATE customers SET city = 'Version2' WHERE email = 'mvcc@example.com';
SELECT xmin, xmax, city FROM customers WHERE email = 'mvcc@example.com';
COMMIT;
```

```
 xmin | xmax |   city
------+------+----------
  772 |    0 | Version2
```

`xmin` đổi từ `771` sang `772` — xác nhận đây là một **dòng vật lý hoàn toàn khác** (không phải
cùng dòng bị sửa tại chỗ), do transaction `772` (chính là transaction chạy `UPDATE`) tạo ra.

**Reader không bị block bởi Writer đang giữ lock:**

```sql
-- Session A: giu row lock, CHUA commit, dang "ngu" 2 giay
BEGIN;
UPDATE customers SET city = 'Version3_Uncommitted' WHERE email = 'mvcc@example.com';
SELECT pg_sleep(2);

-- Session B: SELECT thuong
SELECT clock_timestamp();
SELECT city FROM customers WHERE email = 'mvcc@example.com';
SELECT clock_timestamp();
```

Kết quả thật:

```
2026-07-23 15:56:11.432749+00
   city
----------
 Version2
2026-07-23 15:56:11.844004+00
```

Chỉ **~0.4 giây** — không phải **~2+ giây** như khi dùng `SELECT ... FOR UPDATE` (Chapter 07).
`SELECT` thông thường hoàn toàn **không** bị ảnh hưởng bởi `UPDATE` đang giữ lock, trả về ngay
phiên bản **đã commit gần nhất** (`'Version2'`) — không phải giá trị `'Version3_Uncommitted'`
(chưa commit, đúng nguyên tắc không Dirty Read đã học ở Chapter 06).

## Deep Dive

**Nếu `UPDATE` không xoá ngay dòng cũ, ai chịu trách nhiệm dọn dẹp các phiên bản cũ đó, và điều
gì xảy ra nếu không dọn dẹp?** PostgreSQL dùng tiến trình nền **`VACUUM`** (thường tự động qua
**`autovacuum`**) để quét và **giải phóng** không gian chiếm bởi các phiên bản dữ liệu cũ **không
còn transaction nào cần tới nữa** (không có transaction đang chạy nào còn "nhìn thấy" phiên bản
đó trong snapshot của nó). Nếu `autovacuum` không hoạt động hiệu quả (cấu hình sai, hoặc có
transaction chạy **rất lâu** giữ snapshot cũ, ngăn không cho dọn dẹp các phiên bản mà transaction
đó vẫn có thể cần), số lượng phiên bản cũ tích luỹ ngày càng nhiều — hiện tượng gọi là **table
bloat** (bảng phình to) — làm chậm cả truy vấn (phải quét qua nhiều phiên bản chết hơn) lẫn tốn
thêm dung lượng đĩa đáng kể so với dữ liệu thực sự cần thiết.

## Engineering Insight

**Vì sao hiểu MVCC quan trọng khi debug hiện tượng "bảng ngày càng chậm dù dữ liệu không tăng
nhiều"?** Đây là hệ quả trực tiếp của table bloat đã nêu ở Deep Dive: một bảng có tần suất
`UPDATE` cao (ví dụ bảng đếm số lượt xem, cập nhật liên tục) tích luỹ **rất nhiều** phiên bản cũ
theo thời gian nếu `autovacuum` không theo kịp tốc độ ghi — số dòng "thực sự tồn tại" (theo
`COUNT(*)`) có thể không đổi nhiều, nhưng **kích thước vật lý** của bảng trên đĩa tăng liên tục
(chứa cả phiên bản sống lẫn phiên bản chết chưa được dọn), khiến `Seq Scan` (Chapter 03) phải đọc
nhiều dữ liệu hơn hẳn mức cần thiết. Chẩn đoán đúng vấn đề này đòi hỏi kiểm tra thống kê bloat
(qua các view hệ thống hoặc extension chuyên dụng như `pgstattuple`), không chỉ nhìn vào số
lượng dòng logic — một trong những lý do hiểu cơ chế MVCC bên dưới là kiến thức thực tế quan
trọng, không chỉ lý thuyết suông.

## Historical Note

MVCC không phải phát minh riêng của PostgreSQL — khái niệm này xuất hiện từ nghiên cứu những năm
1970]  (song song với các khái niệm nền tảng khác đã gặp xuyên suốt Phase 7), nhưng PostgreSQL là
một trong những hệ quản trị database đầu tiên hiện thực hoá MVCC làm cơ chế **mặc định và duy
nhất** ngay từ những phiên bản đầu (khác với nhiều database khác ban đầu dùng lock-based
concurrency control truyền thống, chỉ bổ sung MVCC sau này — ví dụ MySQL/InnoDB, Oracle). Đây là
một trong những đặc trưng kiến trúc cốt lõi phân biệt PostgreSQL với nhiều hệ quản trị database
khác.

## Myth vs Reality

- **Myth:** "`UPDATE` sửa trực tiếp dữ liệu tại vị trí cũ trên đĩa, giống như gán lại giá trị
  một biến trong bộ nhớ."
  **Reality:** Đã chứng minh bằng thực nghiệm qua `xmin`/`xmax` — `UPDATE` tạo ra một **dòng vật
  lý hoàn toàn mới**, dòng cũ chỉ được đánh dấu "đã cũ", không sửa tại chỗ.

- **Myth:** "Một `SELECT` luôn phải chờ nếu có `UPDATE` khác đang thao tác trên cùng dòng dữ
  liệu."
  **Reality:** Đã chứng minh bằng thực nghiệm ngược lại — `SELECT` thông thường không bao giờ bị
  block bởi `UPDATE` (chỉ `SELECT ... FOR UPDATE` mới bị, vì nó chủ động xin lock — Chapter 07).

## Common Mistakes

- **Không hiểu vì sao bảng "phình to" dù số dòng logic không tăng nhiều** — bỏ lỡ nguyên nhân
  gốc rễ là table bloat từ các phiên bản cũ chưa được `VACUUM`.
- **Giữ transaction mở rất lâu** (ví dụ quên `COMMIT`/`ROLLBACK`) — ngăn `autovacuum` dọn dẹp
  các phiên bản cũ mà transaction đó về lý thuyết vẫn có thể cần, gây bloat nghiêm trọng hơn.
- **Nhầm lẫn MVCC với Optimistic Lock** (Phase 6, Chapter 13) — dù cùng liên quan tới việc không
  khoá khi đọc, MVCC là cơ chế versioning ở tầng lưu trữ vật lý; Optimistic Lock (`@Version`) là
  một kỹ thuật ứng dụng xây trên nền tảng đó để kiểm soát xung đột ghi ở tầng logic nghiệp vụ.

## Debug Checklist

- [ ] Bảng chậm đi theo thời gian dù số dòng không tăng nhiều? → nghi ngờ table bloat, kiểm tra
      `autovacuum` có hoạt động đúng không, cân nhắc `VACUUM` thủ công.
- [ ] Một transaction chạy rất lâu, nghi ngờ ảnh hưởng hiệu năng toàn hệ thống? → transaction dài
      giữ snapshot cũ có thể ngăn `autovacuum` dọn dẹp các phiên bản dữ liệu liên quan.
- [ ] Cần xác nhận `UPDATE` có thực sự tạo phiên bản mới không? → kiểm tra `xmin` thay đổi trước/
      sau `UPDATE` bằng `SELECT xmin, xmax, * FROM table WHERE ...`.

## Summary

MVCC lưu **nhiều phiên bản** của cùng một dòng dữ liệu thay vì khoá để bảo vệ một bản duy nhất —
mỗi `UPDATE` tạo dòng vật lý mới (`xmin` mới), đánh dấu dòng cũ "đã cũ" (`xmax`), không sửa tại
chỗ. Đã chứng minh bằng thực nghiệm: `xmin` đổi từ `771` sang `772` sau `UPDATE`, xác nhận
versioning thật; `SELECT` thông thường **không** bị block bởi `UPDATE` đang giữ lock (~0.4 giây,
so với ~3.2 giây của `SELECT ... FOR UPDATE` ở Chapter 07) — readers luôn đọc được phiên bản phù
hợp với snapshot riêng của mình mà không cần chờ đợi. Đánh đổi của MVCC là cần dọn dẹp định kỳ
(`VACUUM`/`autovacuum`) các phiên bản cũ không còn cần thiết — nếu không, gây "table bloat" (bảng
phình to), một nguyên nhân phổ biến của hiện tượng "bảng chậm dần theo thời gian" trong thực tế
production.

## Interview Questions

- MVCC là gì? Vì sao nó cho phép "readers không bao giờ block writers, writers không bao giờ
  block readers"?

**Senior**

- Giải thích cơ chế `xmin`/`xmax` và vì sao `UPDATE` không sửa dữ liệu tại chỗ.
- Table bloat là gì? Nó liên quan thế nào tới MVCC, và cách phát hiện/khắc phục?

## Exercises

- [ ] Chạy lại các câu lệnh ở Example trên một PostgreSQL thật, xác nhận `xmin`/`xmax` và hành
      vi blocking (hoặc không blocking) đúng như mô tả.
- [ ] Chạy nhiều `UPDATE` liên tiếp trên cùng một dòng, quan sát `xmin` thay đổi mỗi lần; dùng
      `SELECT pg_relation_size('customers')` trước/sau để quan sát kích thước bảng tăng lên.
- [ ] Chạy `VACUUM customers;` thủ công sau đó, so sánh kích thước bảng trước và sau.

## Cheat Sheet

| | Lock-based (truyền thống) | MVCC (PostgreSQL) |
| --- | --- | --- |
| Đọc trong lúc ghi | Reader phải chờ writer | Reader đọc phiên bản cũ, không chờ |
| `UPDATE` | Sửa tại chỗ | Tạo dòng vật lý mới |
| Cần dọn dẹp? | Không | Có (`VACUUM`, tránh table bloat) |
| Cột hệ thống | N/A | `xmin` (tạo bởi transaction nào), `xmax` (thay thế bởi transaction nào) |

## References

- PostgreSQL Documentation — Concurrency Control (MVCC), Routine Vacuuming.
