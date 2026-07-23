---
tags:
  - Java
  - Executor
  - Concurrency
---

# Executor Framework

> Phase: Phase 3 — Concurrency
> Chapter slug: `executor`

## Metadata

```yaml
Chapter: Executor
Phase: Phase 3 — Concurrency
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 65%
Prerequisites:
  - Chapter 15 — Thread Pool
Used Later:
  - Chapter 17 — ForkJoin
  - Chapter 18 — CompletableFuture
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: [Chapter 15](15-thread-pool.md) — Thread Pool giải quyết vấn đề hiệu năng tạo/huỷ
> thread. Nhưng khi tắt một ứng dụng (deploy phiên bản mới, restart service), có một câu hỏi
> khác: nếu một tác vụ đang chạy dở trong pool, dừng ứng dụng như thế nào cho **đúng đắn** —
> chờ nó xong, hay ngắt nó ngay lập tức?

Thử nghiệm thực tế với `shutdownNow()` trên một task đang chạy dở (`Thread.sleep(2000)`):

```
Task dai bat dau
Task dai BI INTERRUPT boi shutdownNow()
shutdownNow() tra ve 0 task dang cho trong hang doi (chua chay)
```

`shutdownNow()` **ngắt** (interrupt) task đang chạy ngay lập tức, khác hẳn `shutdown()` — vốn
để task đang chạy hoàn thành bình thường, chỉ từ chối nhận task mới. `ExecutorService` là API
chuẩn hoá toàn bộ vòng đời quản lý thread pool này.

## Interview Question (Central)

> `shutdown()` khác `shutdownNow()` như thế nào? Khi nào nên dùng cái nào?

## Objectives

- [ ] Phân biệt các loại pool có sẵn: `newFixedThreadPool`, `newCachedThreadPool`,
      `newSingleThreadExecutor`, `newScheduledThreadPool`
- [ ] Tự tay chứng minh sự khác biệt hành vi giữa `shutdown()` và `shutdownNow()`
- [ ] Hiểu phân cấp interface `Executor` → `ExecutorService` → `ScheduledExecutorService`

## Prerequisites

- Chapter 15 — hiểu khái niệm Thread Pool, `ThreadPoolExecutor`.

## Used Later

- **Chapter 17 (ForkJoin)** — `ForkJoinPool` cũng implement `ExecutorService`.
- **Chapter 18 (CompletableFuture)** — có thể nhận một `Executor` tuỳ chỉnh để chạy các bước
  bất đồng bộ.

## Problem

Quản lý vòng đời một thread pool không chỉ là "tạo và nộp tác vụ" — còn cần một cách chuẩn để
**dừng** pool đúng đắn: chờ tác vụ đang chạy hoàn thành hay ngắt ngay lập tức? Xử lý ra sao với
các tác vụ còn đang **chờ trong hàng đợi**, chưa kịp chạy? Nếu mỗi đội tự nghĩ ra quy ước riêng,
code sẽ thiếu nhất quán và dễ rò rỉ tài nguyên (thread không bao giờ dừng).

## Concept

**`Executor`** là interface gốc, đơn giản nhất (chỉ có `execute(Runnable)`).
**`ExecutorService`** mở rộng thêm quản lý vòng đời (`submit`, `shutdown`, `shutdownNow`,
`awaitTermination`) và trả về `Future` cho kết quả. **`ScheduledExecutorService`** mở rộng
tiếp, hỗ trợ lên lịch chạy tác vụ sau một khoảng thời gian hoặc lặp định kỳ.

## Why?

Tách các interface theo mức độ khả năng (Executor đơn giản → ExecutorService đầy đủ vòng đời →
ScheduledExecutorService có lịch trình) theo đúng nguyên tắc **Interface Segregation** — code
chỉ cần `execute()` đơn giản không bị buộc phải biết về `shutdown()`/`Future`. Việc chuẩn hoá
`shutdown()` vs `shutdownNow()` thành hai phương thức tách biệt (thay vì một tham số boolean mơ
hồ) giúp code gọi rõ ràng về ý định: dừng "êm" hay dừng "ngay lập tức".

## How?

```java
ExecutorService pool = Executors.newFixedThreadPool(4);
pool.execute(() -> { /* khong can ket qua */ });
Future<Integer> f = pool.submit(() -> 42); // can ket qua

pool.shutdown();                          // KHONG nhan task moi, cho task dang chay xong
List<Runnable> pending = pool.shutdownNow(); // NGAT task dang chay, tra ve task con cho trong hang doi
```

## Visualization

```
Executor
   │  execute(Runnable)
   ▼
ExecutorService (mo rong Executor)
   │  submit(Callable/Runnable) -> Future
   │  shutdown() / shutdownNow() / awaitTermination()
   ▼
ScheduledExecutorService (mo rong ExecutorService)
      schedule(task, delay)
      scheduleAtFixedRate(task, initialDelay, period)
```

## Example

```java
import java.util.concurrent.*;
import java.util.List;

public class ExecutorTypesDemo {
    public static void main(String[] args) throws Exception {
        ExecutorService fixed = Executors.newFixedThreadPool(2);
        for (int i = 0; i < 4; i++) {
            int id = i;
            fixed.submit(() -> System.out.println("Fixed task " + id + " chay tren " + Thread.currentThread().getName()));
        }
        fixed.shutdown();
        fixed.awaitTermination(5, TimeUnit.SECONDS);

        ExecutorService cached = Executors.newCachedThreadPool();
        for (int i = 0; i < 4; i++) {
            int id = i;
            cached.submit(() -> System.out.println("Cached task " + id + " chay tren " + Thread.currentThread().getName()));
        }
        cached.shutdown();
        cached.awaitTermination(5, TimeUnit.SECONDS);

        ExecutorService svc = Executors.newFixedThreadPool(1);
        svc.submit(() -> {
            try {
                System.out.println("Task dai bat dau");
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                System.out.println("Task dai BI INTERRUPT boi shutdownNow()");
            }
        });
        Thread.sleep(100);
        List<Runnable> pending = svc.shutdownNow();
        System.out.println("shutdownNow() tra ve " + pending.size() + " task dang cho trong hang doi (chua chay)");
        svc.awaitTermination(5, TimeUnit.SECONDS);
    }
}
```

Kết quả thật (JDK 17):

```
Fixed task 0 chay tren pool-1-thread-1
Fixed task 1 chay tren pool-1-thread-2
Fixed task 2 chay tren pool-1-thread-1
Fixed task 3 chay tren pool-1-thread-2
---
Cached task 0 chay tren pool-2-thread-1
Cached task 2 chay tren pool-2-thread-3
Cached task 3 chay tren pool-2-thread-4
Cached task 1 chay tren pool-2-thread-2
---
Task dai bat dau
Task dai BI INTERRUPT boi shutdownNow()
shutdownNow() tra ve 0 task dang cho trong hang doi (chua chay)
```

Với `FixedThreadPool(2)`: chỉ 2 thread thực (`pool-1-thread-1`, `pool-1-thread-2`) được **tái
sử dụng** cho cả 4 task. Với `CachedThreadPool`: **4 thread khác nhau** được tạo cho 4 task nộp
gần như đồng thời — đúng bản chất "tạo thread mới khi cần, không giới hạn". Với `shutdownNow()`:
task đang `Thread.sleep(2000)` bị **interrupt ngay lập tức** (in ra dòng "BI INTERRUPT"), khác
hẳn hành vi `shutdown()` (task được để chạy hết tự nhiên).

## Deep Dive

**`awaitTermination()` để làm gì, và vì sao cần gọi sau `shutdown()`/`shutdownNow()`?**
`shutdown()`/`shutdownNow()` **không blocking** — chúng chỉ báo hiệu "bắt đầu quá trình dừng"
rồi trả về ngay lập tức, các thread trong pool có thể vẫn đang chạy dở. `awaitTermination(timeout,
unit)` mới thực sự **chờ** (blocking, có giới hạn thời gian) cho tới khi tất cả thread trong
pool thực sự kết thúc, hoặc hết timeout. Thiếu bước `awaitTermination`, code có thể tưởng nhầm
pool đã dừng hẳn trong khi thực tế vẫn còn thread đang chạy — gây race condition khi logic sau
đó phụ thuộc vào giả định "mọi task đã xong".

## Engineering Insight

**Vì sao nhiều framework (Spring, Tomcat) quản lý riêng vòng đời `ExecutorService` thay vì để
JVM tự lo?** JVM chỉ tự thoát khi **tất cả non-daemon thread** kết thúc (Phase 1, Chapter 04 có
nhắc gián tiếp qua Runtime Data Areas). Thread trong `ExecutorService` mặc định là non-daemon —
nếu quên gọi `shutdown()`, chương trình **không bao giờ thoát được** dù logic chính đã hoàn
tất từ lâu, vì các thread pool vẫn "sống" chờ task mới vô thời hạn. Đây là lý do các framework
lớn luôn có một bước "graceful shutdown" tường minh: bắt tín hiệu dừng ứng dụng (SIGTERM), gọi
`shutdown()` cho mọi executor đã tạo, `awaitTermination` với timeout hợp lý, rồi mới thực sự
thoát tiến trình.

## Historical Note

`Executor`/`ExecutorService` ra đời cùng JSR 166 (Java 5, 2004). `ScheduledExecutorService`
cùng đợt, thay thế cho `java.util.Timer`/`TimerTask` cũ hơn (từ Java 1.3) — `Timer` chạy trên
**một thread duy nhất**, nếu một task ném exception không bắt, toàn bộ `Timer` bị dừng hoạt
động vĩnh viễn (lỗi thiết kế nổi tiếng); `ScheduledExecutorService` dùng thread pool, một task
lỗi không ảnh hưởng tới các task lên lịch khác.

## Myth vs Reality

- **Myth:** "`shutdown()` dừng thread pool ngay lập tức."
  **Reality:** Xem Example — `shutdown()` chỉ ngăn nhận task **mới**, các task đang chạy và
  đang chờ trong hàng đợi vẫn được hoàn thành bình thường. Chỉ `shutdownNow()` mới ngắt task
  đang chạy ngay lập tức.

- **Myth:** "Gọi `shutdown()` là đủ, không cần `awaitTermination()`."
  **Reality:** Xem Deep Dive — thiếu `awaitTermination` có thể khiến code tiếp tục chạy trong
  khi pool chưa thực sự dừng hẳn.

## Common Mistakes

- **Quên gọi `shutdown()`** — chương trình không thoát được vì thread pool giữ JVM sống.
- **Gọi `shutdown()` nhưng không `awaitTermination()`** — logic tiếp theo chạy trước khi pool
  thực sự dừng hẳn, dẫn tới giả định sai về trạng thái.
- **Dùng `shutdownNow()` khi thực sự chỉ cần dừng êm** — có thể làm gián đoạn task quan trọng
  giữa chừng, gây mất dữ liệu hoặc trạng thái không nhất quán nếu task không xử lý
  `InterruptedException` cẩn thận.

## Best Practices

- Ưu tiên `shutdown()` (dừng êm) trừ khi có lý do rõ ràng cần dừng ngay lập tức
  (`shutdownNow()`).
- Luôn gọi `awaitTermination(timeout, unit)` sau `shutdown()`/`shutdownNow()` để đảm bảo chờ
  đúng tới khi pool thực sự dừng, có timeout để tránh chờ vô hạn.
- Với tác vụ lên lịch định kỳ, dùng `ScheduledExecutorService` thay vì `Timer`/`TimerTask` cũ.

## Debug Checklist

- [ ] Chương trình không thoát dù logic chính đã xong? → kiểm tra đã gọi `shutdown()` cho mọi
      `ExecutorService` đã tạo chưa.
- [ ] Task bị dừng giữa chừng ngoài ý muốn? → kiểm tra có đang gọi nhầm `shutdownNow()` thay vì
      `shutdown()` không.
- [ ] Logic sau `shutdown()` chạy sai vì pool chưa thực sự dừng? → thêm `awaitTermination()`
      với timeout phù hợp.

## Summary

`Executor` → `ExecutorService` → `ScheduledExecutorService` là phân cấp interface chuẩn hoá
việc chạy và quản lý vòng đời thread pool. Đã chứng minh bằng thực nghiệm: `FixedThreadPool`
tái sử dụng đúng số thread cố định, `CachedThreadPool` tạo thread mới không giới hạn, và
`shutdownNow()` ngắt task đang chạy ngay lập tức (khác `shutdown()` — để task chạy hết tự
nhiên). Luôn kèm `awaitTermination()` sau khi gọi shutdown để đảm bảo chờ đúng tới khi pool
thực sự dừng hẳn.

## Interview Questions

**Junior**

- `Executor` và `ExecutorService` khác nhau như thế nào?

**Mid**

- `shutdown()` khác `shutdownNow()` như thế nào? Khi nào nên dùng cái nào?
- Vì sao cần gọi `awaitTermination()` sau `shutdown()`?

**Senior**

- Giải thích vì sao quên gọi `shutdown()` có thể khiến một chương trình Java không bao giờ
  thoát được, dù logic chính đã hoàn tất.
- So sánh `ScheduledExecutorService` với `Timer`/`TimerTask` cũ — vì sao `ScheduledExecutorService` được khuyến nghị thay thế?

## Exercises

- [ ] Chạy lại `ExecutorTypesDemo` ở trên, xác nhận `FixedThreadPool` tái sử dụng đúng 2
      thread trong khi `CachedThreadPool` tạo 4 thread khác nhau.
- [ ] Viết một ví dụ dùng `ScheduledExecutorService.scheduleAtFixedRate()` in ra thời gian mỗi
      giây, 5 lần, rồi tự `shutdown()`.
- [ ] Thử gọi `shutdown()` (không phải `shutdownNow()`) trên một pool có task đang
      `Thread.sleep(2000)`, quan sát task đó hoàn thành bình thường thay vì bị interrupt.

## Cheat Sheet

| | `shutdown()` | `shutdownNow()` |
| --- | --- | --- |
| Task mới | Từ chối | Từ chối |
| Task đang chạy | Chạy tiếp tới khi xong | Bị `interrupt()` ngay |
| Task đang chờ trong hàng đợi | Vẫn được chạy | Bị huỷ, trả về qua giá trị hàm |
| Trả về | `void` | `List<Runnable>` (task chưa chạy) |

## References

- Java SE API Documentation — `java.util.concurrent.Executor`, `ExecutorService`,
  `ScheduledExecutorService`.
- Java Concurrency in Practice (Brian Goetz) — Chapter 7: Cancellation and Shutdown.
