---
tags:
  - SQL
  - Lock
  - Deadlock
  - PostgreSQL
---

# Lock (Database-level Locking) và Deadlock

> Phase: Phase 7 — Database
> Chapter slug: `lock`

## Metadata

```yaml
Chapter: Lock (Database-level Locking)
Phase: Phase 7 — Database
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 75%
Prerequisites:
  - Chapter 06 — Isolation
  - Phase 6, Chapter 14 — Pessimistic Lock
  - Phase 3, Chapter 09-10 — Lock, ReentrantLock
Used Later:
  - Chapter 08 — MVCC
  - Chapter 09 — Optimization
Estimated Reading: 18 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: Phase 6, Chapter 14 — `SELECT ... FOR UPDATE` (qua JPA) khiến một thread thực sự bị
> block. Chapter này đi xuống tầng SQL thuần bên dưới, và khai thác thêm một hiện tượng nguy
> hiểm hơn: **deadlock** — hai giao dịch cùng chờ đợi lẫn nhau, mãi mãi không ai nhường ai.

Cho hai session cùng khoá **theo thứ tự ngược nhau**: Session A khoá dòng 1 rồi xin dòng 2;
Session B khoá dòng 2 rồi xin dòng 1 — cùng lúc:

```sql
-- Session A: BEGIN; SELECT ... WHERE id=1 FOR UPDATE; SELECT ... WHERE id=2 FOR UPDATE;
-- Session B: BEGIN; SELECT ... WHERE id=2 FOR UPDATE; SELECT ... WHERE id=1 FOR UPDATE;
```

Kết quả thật (PostgreSQL 16.14) — Session A:

```
ERROR:  deadlock detected
DETAIL:  Process 220 waits for ShareLock on transaction 767; blocked by process 221.
Process 221 waits for ShareLock on transaction 766; blocked by process 220.
HINT:  See server log for query details.
```

PostgreSQL **tự phát hiện** vòng chờ đợi tuần hoàn này và chủ động huỷ **một trong hai**
transaction (Session A) — Session B tiếp tục bình thường sau đó.

## Interview Question (Central)

> Row-level lock khác table-level lock như thế nào? Deadlock xảy ra khi nào, và database phát
> hiện/xử lý nó ra sao?

## Objectives

- [ ] Phân biệt row-level lock (`SELECT ... FOR UPDATE`) và table-level lock (`LOCK TABLE`)
- [ ] Tự tay chứng minh bằng thực nghiệm (đo timestamp thật): một session giữ row lock khiến
      session khác thực sự bị **block**, không phải lỗi hay bỏ qua
- [ ] Tự tay tái hiện một **deadlock thật**, quan sát PostgreSQL tự phát hiện và huỷ một
      transaction để phá vỡ vòng chờ đợi

## Prerequisites

- Chapter 06 — hiểu Isolation Level, vì lock là một trong những cơ chế hiện thực hoá nó.
- Phase 6, Chapter 14 — đã học Pessimistic Lock qua JPA, chapter này đi xuống đúng cơ chế SQL
  thuần bên dưới lớp trừu tượng đó.
- Phase 3, Chapter 09-10 — liên hệ với khái niệm `Lock`/deadlock ở tầng JVM, cùng nguyên lý
  nhưng khác tầng.

## Used Later

- **Chapter 08 (MVCC)** — MVCC giảm thiểu nhu cầu dùng lock cho việc **đọc** dữ liệu, chỉ lock
  vẫn cần thiết cho **ghi**.
- **Chapter 09 (Optimization)** — lock contention (tranh chấp khoá) là một nguyên nhân phổ biến
  của hiệu năng kém dưới tải cao.

## Problem

Khi nhiều transaction cùng cần **sửa đổi** cùng một dòng dữ liệu, cần một cơ chế đảm bảo chỉ
đúng một transaction thao tác tại một thời điểm — nếu không, hai `UPDATE` chạy chồng chéo có thể
dẫn tới kết quả sai (mất một trong hai thay đổi). Nhưng cơ chế khoá đó, nếu không cẩn thận, có
thể dẫn tới tình huống **hai transaction chờ đợi lẫn nhau vô thời hạn** — không transaction nào
có thể tiến triển, hệ thống "đứng hình" cục bộ.

## Concept

**Row-level lock** (`SELECT ... FOR UPDATE`) khoá **chỉ những dòng cụ thể** được truy vấn —
transaction khác vẫn tự do thao tác trên các dòng khác của cùng bảng. **Table-level lock**
(`LOCK TABLE`) khoá **toàn bộ bảng** — hiếm khi cần thiết trong ứng dụng thông thường, chủ yếu
dùng cho các thao tác quản trị (DDL, bulk operation). **Deadlock** xảy ra khi hai (hoặc nhiều)
transaction tạo thành một **vòng chờ đợi tuần hoàn** (A chờ khoá B đang giữ, B chờ khoá A đang
giữ) — PostgreSQL chạy một tiến trình **tự động phát hiện deadlock** định kỳ, khi phát hiện vòng
chờ đợi, nó chọn một transaction (thường là transaction "trẻ nhất" hoặc theo tiêu chí nội bộ) để
**huỷ bỏ**, phá vỡ vòng chờ đợi, cho phép các transaction còn lại tiếp tục.

## Why?

Row-level lock (thay vì table-level mặc định) tối ưu cho khả năng chạy song song — hầu hết giao
dịch chỉ cần thao tác trên một vài dòng cụ thể, khoá toàn bộ bảng cho mọi thao tác sẽ triệt tiêu
gần như toàn bộ lợi ích của việc có nhiều transaction đồng thời. Cơ chế tự động phát hiện deadlock
là **bắt buộc** vì bản thân database không có cách nào ngăn deadlock xảy ra một cách tiên nghiệm
(việc lập trình viên vô tình khoá theo thứ tự khác nhau ở các phần code khác nhau gần như không
thể loại trừ hoàn toàn) — thay vào đó, hệ thống chấp nhận deadlock **có thể** xảy ra, và tập
trung vào việc **phát hiện và phục hồi nhanh** (huỷ một transaction, transaction đó có thể thử
lại) thay vì cố ngăn chặn tuyệt đối.

## How?

```sql
-- Row-level lock
BEGIN;
SELECT * FROM customers WHERE id = 1 FOR UPDATE; -- CHI khoa dong id=1
COMMIT; -- nha khoa

-- Table-level lock (hiem dung)
BEGIN;
LOCK TABLE customers IN ACCESS EXCLUSIVE MODE; -- khoa TOAN BO bang
COMMIT;
```

## Visualization

```
Row lock blocking (that su, do duoc bang timestamp):

  Session A: BEGIN; SELECT id=1 FOR UPDATE; ──► GIANH khoa row 1
                     pg_sleep(2)...
  Session B:              BEGIN; SELECT id=1 FOR UPDATE; ──► CHO (block that su)
                                                              │
  Session A: COMMIT (nha khoa) ─────────────────────────────►│
                                                              ▼
  Session B:                                          MOI duoc tiep tuc

Deadlock (vong cho doi tuan hoan):

  Session A: khoa row1 ──► xin row2 (dang bi B giu) ──► CHO B
  Session B: khoa row2 ──► xin row1 (dang bi A giu) ──► CHO A
                    │                                        │
                    └──────────── VONG TRON ────────────────┘
                                     │
                          PostgreSQL PHAT HIEN, HUY 1 ben (vd: A)
                                     │
                          B duoc tiep tuc binh thuong
```

## Example

**Row lock blocking — chứng minh bằng timestamp thật:**

```sql
-- Session A
BEGIN;
SELECT * FROM customers WHERE id = 1 FOR UPDATE;
SELECT pg_sleep(2); -- gia lap xu ly

-- Session B (chay GAN NHU dong thoi)
BEGIN;
SELECT clock_timestamp(); -- TRUOC khi xin khoa
SELECT * FROM customers WHERE id = 1 FOR UPDATE; -- se BI CHAN
SELECT clock_timestamp(); -- SAU khi gianh duoc khoa
```

Kết quả thật (Session B):

```
        clock_timestamp
-------------------------------
 2026-07-23 15:34:59.182459+00
...
        clock_timestamp
-------------------------------
 2026-07-23 15:35:02.414486+00
```

Chênh lệch **~3.2 giây** giữa hai timestamp — khớp chính xác với thời gian Session A giữ khoá
(`pg_sleep(2)` cộng thêm thời gian tới khi `COMMIT`) — xác nhận Session B **thực sự bị block**,
không phải lỗi hay tự động bỏ qua.

**Deadlock thật:**

```sql
-- Session A: BEGIN; SELECT id=1 FOR UPDATE; -- roi SELECT id=2 FOR UPDATE
-- Session B: BEGIN; SELECT id=2 FOR UPDATE; -- roi SELECT id=1 FOR UPDATE
```

Kết quả thật (Session A):

```
ERROR:  deadlock detected
DETAIL:  Process 220 waits for ShareLock on transaction 767; blocked by process 221.
Process 221 waits for ShareLock on transaction 766; blocked by process 220.
HINT:  See server log for query details.
CONTEXT:  while locking tuple (0,2) in relation "customers"
```

Session B, sau khi Session A bị huỷ (giải phóng khoá), tiếp tục thành công bình thường và nhận
được dữ liệu dòng `id=1`.

## Deep Dive

**PostgreSQL "phát hiện" deadlock bằng cơ chế cụ thể nào, không phải chỉ dựa vào timeout đơn
thuần?** PostgreSQL duy trì một **đồ thị chờ đợi** (wait-for graph) nội bộ — mỗi node là một
transaction, mỗi cạnh biểu diễn "transaction này đang chờ khoá mà transaction kia đang giữ". Định
kỳ (mặc định mỗi 1 giây, cấu hình qua `deadlock_timeout`), khi một transaction phải chờ khoá quá
lâu, PostgreSQL **duyệt** đồ thị chờ đợi đó để tìm **chu trình** (cycle) — nếu tìm thấy (như
trong ví dụ: A chờ B, B chờ A, tạo thành chu trình khép kín), đó chính là deadlock, và
PostgreSQL chủ động huỷ một transaction trong chu trình đó (thông báo lỗi `deadlock detected`
ngay lập tức, không cần đợi thêm) để phá vỡ chu trình. Đây là lý do thông điệp lỗi trong ví dụ
liệt kê chính xác **process nào chờ transaction nào** — đó chính là các cạnh trong đồ thị chờ
đợi mà PostgreSQL vừa duyệt qua để phát hiện chu trình.

## Engineering Insight

**Chiến lược thực tế nào giúp tránh deadlock hoàn toàn, thay vì chỉ dựa vào cơ chế phát hiện +
huỷ bỏ của database?** Chiến lược hiệu quả nhất và đơn giản nhất: **quy ước thứ tự khoá nhất
quán** trên toàn bộ codebase — nếu **mọi** giao dịch trong ứng dụng luôn khoá các dòng theo cùng
một thứ tự cố định (ví dụ luôn theo `id` tăng dần, bất kể nghiệp vụ cụ thể là gì), một chu trình
chờ đợi tuần hoàn **về mặt toán học không thể hình thành** (vì mọi transaction đều "tiến" theo
cùng một hướng thứ tự, không ai có thể "quay ngược" để tạo vòng tròn). Đây chính là nguyên nhân
sâu xa của deadlock trong ví dụ: Session A khoá theo thứ tự `1 → 2`, Session B khoá theo thứ tự
`2 → 1` — ngược nhau. Nếu cả hai đều khoá theo đúng thứ tự `1 → 2`, deadlock ở ví dụ này sẽ
**không bao giờ** xảy ra (một trong hai chỉ đơn giản phải chờ, không có vòng tròn). Đây là lý do
kỷ luật thiết kế (thứ tự khoá nhất quán) quan trọng hơn việc dựa vào cơ chế phát hiện/retry của
database — phòng tránh tốt hơn nhiều so với chỉ xử lý hậu quả.

## Historical Note

Deadlock detection dựa trên wait-for graph là kỹ thuật kinh điển trong lý thuyết hệ điều hành và
database, có nguồn gốc từ nghiên cứu concurrency control những năm 1970 — cùng thời kỳ với các
khái niệm nền tảng khác đã gặp xuyên suốt Phase 7 (System R, ACID). Đây là một trong những vấn
đề lý thuyết máy tính "cổ điển" áp dụng gần như y nguyên xuyên suốt nhiều tầng hệ thống khác nhau
— từ deadlock giữa các thread trong một JVM (Phase 3) tới deadlock giữa các transaction trong
một database, cùng bản chất toán học (chu trình trong đồ thị chờ đợi), chỉ khác tầng áp dụng.

## Myth vs Reality

- **Myth:** "Row lock chỉ ảnh hưởng ai đó cố `UPDATE`/`DELETE` cùng dòng, không ảnh hưởng người
  chỉ `SELECT` thông thường."
  **Reality:** Đúng với `SELECT` thông thường (không lock), nhưng `SELECT ... FOR UPDATE` (đọc
  kèm khoá, Phase 6 Chapter 14) hoàn toàn bị block giống hệt `UPDATE` — đã chứng minh bằng thực
  nghiệm timestamp.

- **Myth:** "Deadlock là lỗi hệ thống nghiêm trọng, cần escalate ngay khi gặp."
  **Reality:** Đây là hành vi **thiết kế đúng đắn** của database — PostgreSQL chủ động phát hiện
  và xử lý, ứng dụng chỉ cần có logic retry cho transaction bị huỷ (tương tự serialization
  failure, Chapter 06).

## Common Mistakes

- **Khoá nhiều dòng theo thứ tự không nhất quán giữa các luồng nghiệp vụ khác nhau** — nguyên
  nhân trực tiếp và phổ biến nhất của deadlock, xem Engineering Insight.
- **Không có logic retry cho transaction bị huỷ do deadlock** — ứng dụng báo lỗi trực tiếp cho
  người dùng thay vì tự động thử lại.
- **Dùng `LOCK TABLE` khi chỉ cần row-level lock** — triệt tiêu khả năng chạy song song không
  cần thiết.

## Best Practices

- Luôn khoá nhiều dòng theo một thứ tự **nhất quán** (ví dụ theo khoá chính tăng dần) xuyên suốt
  toàn bộ ứng dụng.
- Ưu tiên row-level lock (`SELECT ... FOR UPDATE`), chỉ dùng table-level lock cho các thao tác
  quản trị đặc biệt.
- Có chiến lược retry rõ ràng cho transaction bị huỷ do deadlock (lỗi `40P01`).
- Giữ transaction giữ lock **càng ngắn càng tốt** — tránh thực hiện I/O chậm trong khi đang giữ
  khoá (giống bài học đã học ở Phase 6, Chapter 14).

## Debug Checklist

- [ ] Một truy vấn "treo" lâu bất thường? → kiểm tra có transaction khác đang giữ row lock trên
      cùng dòng dữ liệu, chưa commit/rollback.
- [ ] Gặp lỗi `deadlock detected`? → kiểm tra thứ tự khoá các dòng dữ liệu có nhất quán giữa các
      luồng nghiệp vụ khác nhau không; áp dụng thứ tự khoá cố định để phòng tránh.
- [ ] Cần biết ai đang giữ một khoá cụ thể? → truy vấn view hệ thống `pg_locks` kết hợp
      `pg_stat_activity` để xem chi tiết.

## Summary

Row-level lock (`SELECT ... FOR UPDATE`) khoá chỉ những dòng cụ thể; table-level lock
(`LOCK TABLE`) khoá toàn bộ bảng, hiếm khi cần thiết. Đã chứng minh bằng thực nghiệm (đo
timestamp thật): một session giữ row lock khiến session khác **thực sự bị block** (~3.2 giây
chênh lệch, khớp chính xác thời gian giữ khoá). Deadlock xảy ra khi các transaction tạo vòng chờ
đợi tuần hoàn (khoá theo thứ tự ngược nhau) — đã chứng minh bằng thực nghiệm: PostgreSQL tự động
phát hiện qua đồ thị chờ đợi, chủ động huỷ một transaction (`ERROR: deadlock detected` với chi
tiết process/transaction cụ thể), cho phép transaction còn lại tiếp tục. Chiến lược phòng tránh
hiệu quả nhất là quy ước thứ tự khoá nhất quán trên toàn ứng dụng — về mặt toán học loại bỏ khả
năng hình thành chu trình chờ đợi.

## Interview Questions

- Row-level lock khác table-level lock như thế nào?
- Deadlock xảy ra khi nào, và database phát hiện/xử lý nó ra sao?

**Senior**

- Giải thích cơ chế wait-for graph PostgreSQL dùng để phát hiện deadlock.
- Chiến lược hiệu quả nhất để phòng tránh deadlock trong ứng dụng là gì? Vì sao nó hiệu quả hơn
  việc chỉ dựa vào cơ chế phát hiện của database?

## Exercises

- [ ] Chạy lại hai kịch bản ở Example trên một PostgreSQL thật (2 session song song), xác nhận
      timestamp blocking và deadlock error đúng như mô tả.
- [ ] Sửa lại kịch bản deadlock để cả hai session khoá theo **cùng một thứ tự** (`1 → 2`), xác
      nhận deadlock không còn xảy ra (một session chỉ đơn giản chờ session kia xong).
- [ ] Truy vấn `pg_locks`/`pg_stat_activity` trong lúc một session đang giữ row lock, quan sát
      thông tin khoá chi tiết.

## Cheat Sheet

| | Row-level Lock | Table-level Lock |
| --- | --- | --- |
| Phạm vi | Chỉ dòng cụ thể | Toàn bộ bảng |
| Cú pháp | `SELECT ... FOR UPDATE` | `LOCK TABLE ... IN ... MODE` |
| Ảnh hưởng song song | Thấp (chỉ chặn giao dịch khác cần đúng dòng đó) | Cao (chặn mọi giao dịch khác trên bảng) |
| Dùng khi | Thao tác nghiệp vụ thông thường | Thao tác quản trị đặc biệt |

**Phòng tránh deadlock:** luôn khoá nhiều dòng theo thứ tự nhất quán trên toàn ứng dụng.

## References

- PostgreSQL Documentation — Explicit Locking, Deadlocks.
