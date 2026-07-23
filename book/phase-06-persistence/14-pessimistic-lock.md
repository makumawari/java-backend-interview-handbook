---
tags:
  - JPA
  - PessimisticLock
  - Hibernate
---

# Pessimistic Lock

> Phase: Phase 6 — Persistence
> Chapter slug: `pessimistic-lock`

## Metadata

```yaml
Chapter: Pessimistic Lock
Phase: Phase 6 — Persistence
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 60%
Prerequisites:
  - Chapter 13 — Optimistic Lock
  - Phase 3, Chapter 09-10 — Lock, ReentrantLock
Used Later:
  - Thiết kế hệ thống giao dịch tài chính, tồn kho (Phase 8)
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 13](13-optimistic-lock.md) — Optimistic Lock "lạc quan", chỉ kiểm tra xung
> đột **tại thời điểm ghi**, không ngăn ai đọc/sửa song song. Có tình huống nào cần ngăn hoàn
> toàn việc "hai người cùng sửa một bản ghi" — **chặn đứng** người thứ hai lại, giống hệt
> `synchronized`/`ReentrantLock` (Phase 3) nhưng ở tầng database?

Cho hai thread cùng cố giành khoá `PESSIMISTIC_WRITE` trên **cùng một** `Order`:

```
[ThreadA] Xin PESSIMISTIC_WRITE lock tren Order...
Hibernate: select ... from orders where id=? for update
[ThreadA] DA CO lock, gia lap xu ly ton 500ms...
[ThreadB] Xin PESSIMISTIC_WRITE lock tren CUNG Order (se bi CHAN)...
Hibernate: select ... from orders where id=? for update
[ThreadA] Da COMMIT, NHA lock luc 533ms
[ThreadB] CUOI CUNG cung co lock luc 535ms
```

`ThreadB` gọi `SELECT ... FOR UPDATE` nhưng **không** trả về ngay — nó **treo (block)** đúng tới
khi `ThreadA` commit (nhả khoá) ở mốc 533ms, rồi mới nhận được khoá ở mốc 535ms.

## Interview Question (Central)

> Pessimistic Lock khác Optimistic Lock như thế nào? Khi nào nên dùng Pessimistic Lock thay vì
> Optimistic Lock?

## Objectives

- [ ] Hiểu Pessimistic Lock dùng cơ chế khoá thật của database (`SELECT ... FOR UPDATE`), chặn
      đứng giao dịch khác thay vì chỉ kiểm tra sau
- [ ] Tự tay chứng minh bằng thực nghiệm: một thread giữ khoá khiến thread khác **thực sự bị
      block**, chỉ tiếp tục sau khi khoá được nhả
- [ ] Biết cách chọn giữa Optimistic Lock (Chapter 13) và Pessimistic Lock dựa trên mức độ xung
      đột dự kiến

## Prerequisites

- Chapter 13 — hiểu Optimistic Lock, để so sánh trực tiếp triết lý đối lập.
- Phase 3, Chapter 09-10 — hiểu `Lock`/`ReentrantLock`, khái niệm khoá độc quyền ở tầng JVM, để
  liên hệ với khoá độc quyền ở tầng database.

## Used Later

- **Thiết kế hệ thống giao dịch tài chính, tồn kho** (Phase 8) — Pessimistic Lock là công cụ tiêu
  chuẩn cho các thao tác không thể chấp nhận bất kỳ xung đột nào (trừ tiền, giữ chỗ tồn kho giới
  hạn).

## Problem

Optimistic Lock (Chapter 13) chỉ phát hiện xung đột **sau khi đã xảy ra** (tại thời điểm ghi) —
với tình huống xung đột **xảy ra thường xuyên** (nhiều giao dịch cùng tranh chấp một bản ghi có
tài nguyên giới hạn, ví dụ "chỉ còn 1 vé cuối cùng"), chiến lược "lạc quan rồi retry" trở nên kém
hiệu quả — tỉ lệ thất bại/retry cao, trải nghiệm người dùng kém, và có nguy cơ **starvation**
(một giao dịch liên tục thua trong cuộc đua, không bao giờ thành công) nếu không có gì đảm bảo
công bằng.

## Concept

**Pessimistic Lock** (`LockModeType.PESSIMISTIC_WRITE`) yêu cầu database khoá **thật sự** (dùng
cơ chế native của database, ví dụ `SELECT ... FOR UPDATE`) ngay tại thời điểm đọc — bất kỳ giao
dịch nào khác cố gắng đọc/sửa cùng bản ghi (với cùng loại khoá) sẽ bị **treo (block)** cho tới khi
giao dịch đang giữ khoá **commit hoặc rollback**, giải phóng khoá đó.

## Why?

Pessimistic Lock đánh đổi ngược lại Optimistic Lock: chấp nhận **chi phí chờ đợi** (một giao dịch
phải đợi giao dịch khác xong hoàn toàn) để đổi lấy **đảm bảo tuyệt đối** không có xung đột ghi xảy
ra — không cần retry, không có "lost update" nào có cơ hội xảy ra vì việc truy cập đồng thời bị
ngăn chặn hoàn toàn ngay từ đầu, không phải phát hiện sau. Đây là lựa chọn đúng khi **biết trước**
xung đột sẽ xảy ra thường xuyên, và chi phí một giao dịch phải chờ (thay vì retry nhiều lần) là
chấp nhận được hoặc thậm chí ưu việt hơn.

## How?

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select o from Order o where o.id = :id")
    Optional<Order> findByIdForUpdate(Long id);
}

// Giao dich A:
Order o = orderRepo.findByIdForUpdate(id); // SELECT ... FOR UPDATE, GIANH khoa
// ... xu ly nghiep vu ...
// commit() -> NHA khoa

// Giao dich B (chay GAN NHU dong thoi voi A):
Order o2 = orderRepo.findByIdForUpdate(id); // BI CHAN, cho toi khi A nha khoa
```

## Visualization

```
ThreadA:  findByIdForUpdate() ──► SELECT ... FOR UPDATE ──► GIANH khoa
              │
         xu ly (500ms)
              │
         commit() ──► NHA khoa

ThreadB:                    findByIdForUpdate() ──► SELECT ... FOR UPDATE
                                    │
                              KHONG TRA VE NGAY, bi CHAN o day
                                    │
                              (doi ThreadA nha khoa)
                                    │
                              ThreadA commit ──► ThreadB MOI duoc tiep tuc, GIANH khoa
```

## Example

```java
Thread threadA = new Thread(() -> {
    TransactionStatus txA = txManager.getTransaction(new DefaultTransactionDefinition());
    orderRepo.findByIdForUpdate(orderId); // SELECT ... FOR UPDATE
    threadAHasLock.countDown();
    Thread.sleep(500);
    txManager.commit(txA); // NHA lock o day
});

Thread threadB = new Thread(() -> {
    threadAHasLock.await();
    TransactionStatus txB = txManager.getTransaction(new DefaultTransactionDefinition());
    orderRepo.findByIdForUpdate(orderId); // SE BI CHAN toi khi ThreadA commit
    txManager.commit(txB);
});
```

Kết quả thật (Hibernate ORM 6.5.3.Final, H2):

```
[ThreadA] Xin PESSIMISTIC_WRITE lock tren Order...
Hibernate: select o1_0.id,... from orders o1_0 where o1_0.id=? for update
[ThreadA] DA CO lock, gia lap xu ly ton 500ms...
[ThreadB] Xin PESSIMISTIC_WRITE lock tren CUNG Order (se bi CHAN)...
Hibernate: select o1_0.id,... from orders o1_0 where o1_0.id=? for update
[ThreadA] Da COMMIT, NHA lock luc 533ms
[ThreadB] CUOI CUNG cung co lock luc 535ms
ThreadB co PHAI CHO ThreadA nha lock khong? true
```

Cả hai câu SQL đều có `for update` — cú pháp khoá độc quyền chuẩn SQL. `ThreadB` gọi
`findByIdForUpdate()` **ngay sau khi** `ThreadA` đã giành khoá, nhưng dòng log
`"CUOI CUNG cung co lock"` chỉ xuất hiện **sau** khi `ThreadA` commit (533ms so với 535ms — gần
như ngay lập tức sau khi khoá được nhả) — xác nhận `ThreadB` thực sự bị **treo** (không phải
polling hay retry), JVM/JDBC driver tự động chờ database mở khoá.

## Deep Dive

**Vì sao Pessimistic Lock có nguy cơ deadlock cao hơn hẳn Optimistic Lock, và cách phòng tránh?**
Vì Pessimistic Lock giữ khoá **thật** trong suốt thời gian giao dịch xử lý (không chỉ tại thời
điểm ghi như Optimistic Lock), nếu hai giao dịch cùng cần khoá **nhiều** bản ghi theo **thứ tự
khác nhau** (giao dịch A khoá X rồi cố khoá Y; giao dịch B khoá Y rồi cố khoá X), cả hai có thể
**chờ đợi vòng tròn** lẫn nhau vô thời hạn — deadlock kinh điển, giống hệt nguyên lý đã học ở
Phase 3 (dù ở đây xảy ra ở tầng database, không phải JVM lock). Database hiện đại thường tự phát
hiện deadlock (timeout hoặc phân tích đồ thị chờ đợi) và chủ động huỷ một trong hai giao dịch
(ném exception) để phá vỡ deadlock — nhưng phòng tránh tốt nhất vẫn là **kỷ luật thứ tự khoá**
nhất quán (luôn khoá theo cùng một thứ tự, ví dụ theo ID tăng dần) xuyên suốt toàn bộ ứng dụng.

## Engineering Insight

**Quyết định chọn Optimistic hay Pessimistic Lock dựa trên tiêu chí cụ thể nào, không phải cảm
tính?** Câu hỏi cốt lõi: **tần suất xung đột thực tế dự kiến là bao nhiêu?** Nếu xung đột **hiếm**
(ví dụ một trang cá nhân người dùng tự sửa, xác suất hai request cùng sửa gần như bằng 0) —
Optimistic Lock tối ưu hơn nhiều (không có chi phí chờ đợi khi không có xung đột thực sự). Nếu
xung đột **thường xuyên và có thể dự đoán** (ví dụ đặt vé số lượng giới hạn trong sự kiện flash
sale, nhiều request chắc chắn cạnh tranh cùng một tài nguyên trong khoảng thời gian ngắn) —
Pessimistic Lock tránh được chi phí retry lặp lại nhiều lần của Optimistic Lock dưới tải cao,
dù phải trả giá bằng thời gian chờ đợi tuần tự. Kinh nghiệm thực tế: đo lường (hoặc ước lượng cẩn
thận) tần suất xung đột thực tế của từng luồng nghiệp vụ cụ thể, không áp dụng một chiến lược
duy nhất cho toàn bộ ứng dụng một cách máy móc.

## Historical Note

`SELECT ... FOR UPDATE` là cú pháp SQL chuẩn tồn tại từ rất lâu trong hầu hết hệ quản trị
database quan hệ (Oracle, PostgreSQL, MySQL, SQL Server đều hỗ trợ, dù chi tiết ngữ nghĩa khoá có
thể khác biệt nhỏ giữa các hệ). JPA/Hibernate chuẩn hoá việc kích hoạt cú pháp này qua
`LockModeType` (`PESSIMISTIC_READ`, `PESSIMISTIC_WRITE`, `PESSIMISTIC_FORCE_INCREMENT`) — trừu
tượng hoá sự khác biệt cú pháp cụ thể giữa các database đằng sau một API Java thống nhất.

## Myth vs Reality

- **Myth:** "Pessimistic Lock luôn an toàn hơn Optimistic Lock, nên dùng làm mặc định."
  **Reality:** Xem Engineering Insight — Pessimistic Lock có chi phí chờ đợi và nguy cơ deadlock
  cao hơn; chỉ nên dùng khi xung đột thực sự thường xuyên, không phải mặc định cho mọi trường
  hợp.

- **Myth:** "`findByIdForUpdate()` gọi hai lần liên tiếp từ hai thread sẽ gây lỗi ngay lập tức
  nếu bản ghi đang bị khoá."
  **Reality:** Đã chứng minh bằng thực nghiệm — thread thứ hai **không** gặp lỗi, nó **chờ**
  (block) cho tới khi khoá được nhả, rồi tiếp tục bình thường (trừ khi cấu hình timeout khoá cụ
  thể).

## Common Mistakes

- **Dùng Pessimistic Lock cho mọi thao tác đọc/sửa "để an toàn"** — gây chi phí hiệu năng không
  cần thiết khi xung đột thực tế hiếm khi xảy ra.
- **Khoá nhiều bản ghi theo thứ tự không nhất quán giữa các luồng nghiệp vụ khác nhau** — nguy
  cơ deadlock cao, xem Deep Dive.
- **Giữ khoá pessimistic quá lâu** (thực hiện thao tác I/O chậm, gọi API bên ngoài trong khi vẫn
  giữ khoá) — làm tăng thời gian chờ đợi của các giao dịch khác, giảm throughput tổng thể của hệ
  thống.

## Best Practices

- Chỉ dùng Pessimistic Lock khi đã xác định xung đột thực sự thường xuyên (đo lường hoặc phân
  tích nghiệp vụ cẩn thận), không dùng mặc định.
- Giữ khoá pessimistic trong thời gian **ngắn nhất có thể** — tránh thực hiện I/O chậm/gọi API
  bên ngoài trong khi đang giữ khoá.
- Áp dụng thứ tự khoá nhất quán xuyên suốt ứng dụng khi cần khoá nhiều bản ghi trong cùng giao
  dịch, để giảm nguy cơ deadlock.

## Debug Checklist

- [ ] Một giao dịch bị "treo" lâu bất thường? → kiểm tra có giao dịch khác đang giữ pessimistic
      lock trên cùng bản ghi, xử lý chậm hoặc chưa commit/rollback.
- [ ] Deadlock xuất hiện dưới tải cao? → kiểm tra thứ tự khoá nhiều bản ghi có nhất quán giữa
      các luồng nghiệp vụ khác nhau không.
- [ ] Cần xác nhận một câu truy vấn có thực sự dùng pessimistic lock không? → bật `show-sql`,
      tìm `for update` trong câu SQL sinh ra.

## Summary

Pessimistic Lock (`LockModeType.PESSIMISTIC_WRITE`, `SELECT ... FOR UPDATE`) khoá thật ở tầng
database ngay khi đọc, chặn đứng giao dịch khác cho tới khi khoá được nhả — khác với Optimistic
Lock (Chapter 13) chỉ kiểm tra xung đột tại thời điểm ghi. Đã chứng minh bằng thực nghiệm: thread
thứ hai gọi `findByIdForUpdate()` trên cùng bản ghi **thực sự bị block**, chỉ tiếp tục ngay sau
khi thread đầu tiên commit (nhả khoá) — không phải lỗi, không phải retry, mà là chờ đợi thật.
Pessimistic Lock phù hợp khi xung đột thường xuyên và có thể dự đoán trước; có nguy cơ deadlock
cao hơn Optimistic Lock nếu khoá nhiều bản ghi không theo thứ tự nhất quán. Lựa chọn giữa hai
chiến lược nên dựa trên tần suất xung đột thực tế của từng luồng nghiệp vụ, không áp dụng máy
móc một chiến lược duy nhất.

## Interview Questions

- Pessimistic Lock khác Optimistic Lock như thế nào? Khi nào nên dùng Pessimistic Lock thay vì
  Optimistic Lock?

**Senior**

- Vì sao Pessimistic Lock có nguy cơ deadlock cao hơn Optimistic Lock? Cách phòng tránh?
- Đề xuất tiêu chí cụ thể để quyết định chọn Optimistic hay Pessimistic Lock cho một luồng
  nghiệp vụ.

## Exercises

- [ ] Chạy lại `PessimisticLockTest` ở trên, xác nhận `ThreadB` thực sự bị block tới khi
      `ThreadA` commit.
- [ ] Mô phỏng deadlock: hai thread cùng khoá hai bản ghi theo thứ tự ngược nhau, quan sát
      exception database ném ra khi phát hiện deadlock.
- [ ] So sánh thời gian xử lý tổng thể giữa Optimistic Lock (với retry) và Pessimistic Lock cho
      cùng một kịch bản có tỉ lệ xung đột cao (ví dụ 10 thread cùng tranh chấp 1 bản ghi).

## Cheat Sheet

| | Optimistic Lock (Chapter 13) | Pessimistic Lock |
| --- | --- | --- |
| Cơ chế | Kiểm tra `version` tại thời điểm ghi | Khoá thật (`FOR UPDATE`) tại thời điểm đọc |
| Giao dịch khác | Không bị chặn, có thể đọc/sửa song song | Bị chặn (block) cho tới khi khoá được nhả |
| Phù hợp khi | Xung đột hiếm | Xung đột thường xuyên, có thể dự đoán |
| Rủi ro chính | Cần xử lý exception/retry | Deadlock, giảm throughput nếu giữ khoá lâu |

## References

- Jakarta Persistence Specification — `LockModeType`.
- Hibernate ORM Documentation — Pessimistic Locking.
