---
tags:
  - Java
  - ThreadPool
  - Concurrency
---

# Thread Pool

> Phase: Phase 3 — Concurrency
> Chapter slug: `thread-pool`

## Metadata

```yaml
Chapter: Thread Pool
Phase: Phase 3 — Concurrency
Difficulty: ★★★
Importance: ★★★★★
Interview Frequency: 80%
Prerequisites:
  - Chapter 01 — Thread
  - Chapter 14 — Callable vs Runnable
Used Later:
  - Chapter 16 — Executor
  - Web server request handling (Phase 5, Tomcat thread pool)
Estimated Reading: 18 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: [Chapter 01](01-thread.md) — mỗi `new Thread()` tạo một thread hệ điều hành thật,
> tốn bộ nhớ (mặc định ~1MB stack) và thời gian để hệ điều hành khởi tạo.

Điều gì xảy ra nếu một service web nhận 10.000 request/giây, và mỗi request tạo một
`new Thread()` riêng để xử lý? Đo thử thực tế:

```
Tao 10000 Thread MOI: 229ms
Dung Thread Pool (8 thread) tai su dung: 11ms
```

Tạo 10.000 thread mới chậm hơn **20 lần** so với tái sử dụng một pool chỉ 8 thread cho cùng
10.000 tác vụ. Và đó còn chưa kể tới rủi ro **cạn kiệt tài nguyên hệ thống** nếu số thread tạo
ra không kiểm soát được (mỗi thread hệ điều hành tốn bộ nhớ stack, quá nhiều thread có thể làm
sập cả server).

## Interview Question (Central)

> Thread Pool là gì? Tại sao không nên tạo thread bằng `new Thread()` liên tục?

## Objectives

- [ ] Giải thích chính xác lý do hiệu năng và tài nguyên khiến Thread Pool tốt hơn tạo thread
      thủ công
- [ ] Tự tay đo được chênh lệch thực tế: tạo 10.000 thread mới (229ms) vs tái sử dụng pool 8
      thread (11ms)
- [ ] Hiểu các tham số cấu hình cốt lõi: corePoolSize, maximumPoolSize, hàng đợi (queue),
      keepAliveTime, RejectedExecutionHandler

## Prerequisites

- Chapter 01 — hiểu chi phí tạo một `Thread` (bộ nhớ, thời gian hệ điều hành cấp phát).
- Chapter 14 — hiểu `Runnable`/`Callable`, thứ được nộp (`submit`) vào thread pool.

## Used Later

- **Chapter 16 (Executor)** — `ExecutorService` là API tiêu chuẩn để làm việc với Thread Pool.
- **Web server request handling** (Phase 5) — Tomcat (server nhúng của Spring Boot) dùng một
  thread pool để xử lý các request HTTP đến, đúng mẫu hình "tái sử dụng thread thay vì tạo mới
  mỗi request".

## Problem

Tạo một `Thread` mới không miễn phí: hệ điều hành phải cấp phát bộ nhớ stack (mặc định khoảng
512KB-1MB tuỳ nền tảng), đăng ký thread với scheduler, và giải phóng lại khi thread kết thúc.
Với khối lượng công việc lớn, ngắn hạn, lặp đi lặp lại (ví dụ xử lý hàng nghìn request mỗi
giây), tạo-huỷ thread liên tục gây lãng phí đáng kể — đã đo được chênh lệch **20 lần** ở Story.
Ngoài ra, số lượng thread không kiểm soát có thể làm cạn kiệt bộ nhớ hoặc khiến hệ điều hành quá
tải context-switching.

## Concept

**Thread Pool** là một tập hợp thread được tạo sẵn và **tái sử dụng** để thực thi nhiều tác vụ
liên tiếp, thay vì tạo-huỷ thread cho từng tác vụ. Tác vụ được đưa vào một **hàng đợi
(queue)**; các thread trong pool lần lượt lấy tác vụ ra khỏi hàng đợi và thực thi, sau đó quay
lại lấy tác vụ tiếp theo — không bị huỷ đi.

## Why?

Chi phí tạo thread là **cố định và tốn kém** (bộ nhớ + thời gian hệ điều hành xử lý), trong khi
nhiều tác vụ thực tế rất **ngắn** (vài mili giây). Nếu chi phí tạo/huỷ thread lớn hơn nhiều so
với thời gian thực thi tác vụ, hiệu năng tổng thể bị chi phối bởi overhead thay vì công việc
thực sự. Thread Pool khấu hao chi phí tạo thread **một lần** cho nhiều tác vụ, đồng thời giới
hạn số thread tối đa để bảo vệ tài nguyên hệ thống khỏi bị quá tải.

## How?

```java
ExecutorService pool = Executors.newFixedThreadPool(8); // pool co dinh 8 thread
pool.submit(() -> { /* tac vu */ });
pool.shutdown(); // khong nhan tac vu moi, cho tac vu dang chay xong
```

## Visualization

```
Khong dung Thread Pool (moi tac vu 1 thread MOI):

  Task 1 ──► new Thread ──► chay ──► huy      (tao/huy lien tuc, TON KEM)
  Task 2 ──► new Thread ──► chay ──► huy
  Task 3 ──► new Thread ──► chay ──► huy

Dung Thread Pool (8 thread CO DINH, TAI SU DUNG):

  Task 1 ──┐
  Task 2 ──┼──► Hang doi (Queue) ──► [T1][T2]...[T8] lay task, chay, XONG thi
  Task 3 ──┘                          QUAY LAI lay task TIEP THEO (khong huy)
  ...
  Task N ──┘
```

## Example

```java
import java.util.concurrent.*;

public class ThreadPoolPerf {
    public static void main(String[] args) throws InterruptedException {
        int tasks = 10_000;

        long t1 = System.nanoTime();
        Thread[] threads = new Thread[tasks];
        for (int i = 0; i < tasks; i++) {
            threads[i] = new Thread(() -> { int x = 1 + 1; });
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        long elapsed1 = (System.nanoTime() - t1) / 1_000_000;
        System.out.println("Tao " + tasks + " Thread MOI: " + elapsed1 + "ms");

        long t2 = System.nanoTime();
        ExecutorService pool = Executors.newFixedThreadPool(8);
        CountDownLatch latch = new CountDownLatch(tasks);
        for (int i = 0; i < tasks; i++) {
            pool.submit(() -> { int x = 1 + 1; latch.countDown(); });
        }
        latch.await();
        long elapsed2 = (System.nanoTime() - t2) / 1_000_000;
        System.out.println("Dung Thread Pool (8 thread) tai su dung: " + elapsed2 + "ms");
        pool.shutdown();
    }
}
```

Kết quả thật (JDK 17):

```
Tao 10000 Thread MOI: 229ms
Dung Thread Pool (8 thread) tai su dung: 11ms
```

Cùng 10.000 tác vụ cực nhỏ (`1 + 1`), tạo thread mới cho mỗi tác vụ tốn **229ms**, trong khi tái
sử dụng pool 8 thread chỉ tốn **11ms** — nhanh hơn khoảng 20 lần. Chênh lệch này hoàn toàn nằm
ở chi phí tạo/huỷ thread (bản thân công việc `1 + 1` gần như không tốn thời gian), chứng minh
trực tiếp overhead thực sự của `new Thread()` lặp lại.

## Deep Dive

**Các tham số cấu hình cốt lõi của `ThreadPoolExecutor`** (lớp thực thi bên dưới mọi
`Executors.newXxxThreadPool()`):

- **`corePoolSize`**: số thread tối thiểu giữ lại kể cả khi rảnh.
- **`maximumPoolSize`**: số thread tối đa được tạo khi hàng đợi đầy.
- **`workQueue`**: hàng đợi giữ tác vụ chờ thread rảnh xử lý.
- **`keepAliveTime`**: thời gian một thread "thừa" (vượt corePoolSize) được giữ ở trạng thái
  rảnh trước khi bị huỷ.
- **`RejectedExecutionHandler`**: chính sách xử lý khi cả `maximumPoolSize` thread đều bận VÀ
  hàng đợi đã đầy (mặc định ném `RejectedExecutionException`).

Thứ tự xử lý một tác vụ mới: nếu số thread hiện tại < `corePoolSize` → tạo thread mới ngay. Nếu
đã đủ core nhưng hàng đợi còn chỗ → đưa vào hàng đợi chờ. Nếu hàng đợi đầy VÀ số thread <
`maximumPoolSize` → tạo thêm thread (vượt core, tới tối đa). Nếu tất cả đều đầy → áp dụng
`RejectedExecutionHandler`.

## Engineering Insight

**Vì sao `Executors.newFixedThreadPool()` là factory method tiện lợi nhưng bị khuyến cáo tránh
dùng trong production ở codebase nghiêm túc?** `newFixedThreadPool(n)` bên dưới dùng
`LinkedBlockingQueue` **không giới hạn kích thước** làm hàng đợi. Nếu tốc độ nộp tác vụ vượt
quá tốc độ xử lý trong thời gian dài, hàng đợi phình to vô hạn — có thể dẫn tới
`OutOfMemoryError` thay vì áp dụng `RejectedExecutionHandler` một cách có kiểm soát (vì hàng
đợi "không bao giờ đầy" nên logic từ chối task không bao giờ được kích hoạt). Đây là lý do
nhiều đội ngũ production tự tạo `ThreadPoolExecutor` trực tiếp với `ArrayBlockingQueue` (hàng
đợi có giới hạn kích thước rõ ràng) thay vì dùng các factory method tiện lợi của `Executors`.

## Historical Note

`Executors`/`ThreadPoolExecutor` ra đời cùng JSR 166 (Java 5, 2004). Trước đó, lập trình viên
phải tự viết cơ chế "pool" thủ công (danh sách thread + hàng đợi tác vụ tự đồng bộ hoá) —
`java.util.concurrent` chuẩn hoá và tối ưu hoá mẫu hình này thành một API chính thức, đã qua
kiểm định kỹ lưỡng của JDK.

## Myth vs Reality

- **Myth:** "Thread Pool càng nhiều thread càng xử lý nhanh hơn."
  **Reality:** Số thread tối ưu phụ thuộc vào bản chất tác vụ — CPU-bound (tính toán nặng) nên
  gần bằng số lõi CPU; I/O-bound (chờ mạng/disk) có thể lớn hơn nhiều vì thread phần lớn thời
  gian ở trạng thái chờ, không chiếm CPU. Quá nhiều thread gây context-switching overhead, có
  thể làm CHẬM hơn.

- **Myth:** "`Executors.newFixedThreadPool()` luôn là lựa chọn an toàn cho production."
  **Reality:** Xem Engineering Insight — hàng đợi không giới hạn kích thước tiềm ẩn rủi ro
  `OutOfMemoryError` dưới tải cao kéo dài.

## Common Mistakes

- **Tạo pool mới cho mỗi request thay vì dùng một pool dùng chung suốt vòng đời ứng dụng** —
  mất hoàn toàn lợi ích tái sử dụng thread.
- **Dùng `Executors.newFixedThreadPool()`/`newCachedThreadPool()` không cân nhắc rủi ro hàng
  đợi không giới hạn / số thread không giới hạn (`newCachedThreadPool` có `maximumPoolSize` =
  `Integer.MAX_VALUE`)**.
- **Quên gọi `shutdown()`** — pool không tự động dừng khi chương trình chính kết thúc logic
  (các non-daemon thread trong pool giữ JVM sống, chương trình không thoát được).

## Best Practices

- Với tác vụ CPU-bound (tính toán nặng), đặt số thread pool gần bằng số lõi CPU
  (`Runtime.getRuntime().availableProcessors()`).
- Với tác vụ I/O-bound (chờ mạng/disk), số thread có thể lớn hơn nhiều lần số lõi CPU, vì phần
  lớn thời gian thread ở trạng thái chờ (BLOCKED/WAITING), không chiếm CPU thực sự.
- Trong production, cân nhắc tự cấu hình `ThreadPoolExecutor` với hàng đợi có giới hạn kích
  thước rõ ràng, thay vì các factory method mặc định của `Executors`.
- Luôn gọi `shutdown()` (hoặc dùng try-with-resources với `ExecutorService` từ Java 19+) khi
  không còn cần pool nữa.

## Production Notes

**Vấn đề:** một service dùng `Executors.newFixedThreadPool(20)` để xử lý các tác vụ nền, dưới
đợt tải đột biến (traffic spike), bộ nhớ tăng liên tục và cuối cùng service bị crash với
`OutOfMemoryError`.

- **Triệu chứng:** bộ nhớ heap tăng dần trong đợt tải cao, không giảm lại kể cả khi tải giảm,
  cuối cùng OOM.
- **Root cause:** tốc độ nộp tác vụ vượt xa tốc độ 20 thread có thể xử lý; vì
  `newFixedThreadPool` dùng hàng đợi `LinkedBlockingQueue` không giới hạn, tác vụ tồn đọng ngày
  càng nhiều trong hàng đợi thay vì bị từ chối — mỗi tác vụ tồn đọng giữ một phần bộ nhớ, tích
  luỹ tới khi hết heap.
- **Debug:** heap dump cho thấy phần lớn bộ nhớ nằm trong hàng đợi nội bộ của
  `ThreadPoolExecutor`.
- **Solution:** thay bằng `ThreadPoolExecutor` tự cấu hình với `ArrayBlockingQueue` có giới hạn
  kích thước cụ thể, kèm `RejectedExecutionHandler` phù hợp (ví dụ trả lỗi ngay cho client thay
  vì để hàng đợi phình vô hạn).
- **Prevention:** với mọi thread pool xử lý tác vụ từ nguồn có khả năng đột biến tải (request từ
  bên ngoài), luôn giới hạn kích thước hàng đợi tường minh, không dựa vào giá trị mặc định của
  `Executors.newFixedThreadPool()`.

## Debug Checklist

- [ ] Bộ nhớ tăng liên tục dưới tải cao? → kiểm tra hàng đợi thread pool có giới hạn kích thước
      không.
- [ ] Chương trình không thoát được dù logic chính đã chạy xong? → kiểm tra đã gọi
      `pool.shutdown()` chưa.
- [ ] Thread pool xử lý chậm dù CPU không cao? → xem xét tác vụ có phải I/O-bound không — có
      thể cần tăng số thread thay vì giữ ở mức gần số lõi CPU.

## Source Code Walkthrough

Các factory method của `Executors` chỉ là lớp bọc mỏng quanh `ThreadPoolExecutor`:

```java
// Rut gon tu OpenJDK java.util.concurrent.Executors
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                   0L, TimeUnit.MILLISECONDS,
                                   new LinkedBlockingQueue<Runnable>()); // KHONG gioi han kich thuoc
}

public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE, // KHONG gioi han so thread
                                   60L, TimeUnit.SECONDS,
                                   new SynchronousQueue<Runnable>());
}
```

Nhìn thẳng vào tham số truyền cho `ThreadPoolExecutor` xác nhận đúng rủi ro đã nêu ở Engineering
Insight và Production Notes: `newFixedThreadPool` có `corePoolSize == maximumPoolSize` (số
thread cố định) nhưng hàng đợi không giới hạn; `newCachedThreadPool` có hàng đợi giới hạn chặt
(`SynchronousQueue`, gần như không giữ gì) nhưng số thread tối đa không giới hạn.

## Summary

Thread Pool tái sử dụng một tập thread cố định thay vì tạo-huỷ thread cho mỗi tác vụ — đã đo
được chênh lệch thực tế **20 lần** (229ms tạo mới vs 11ms tái sử dụng, 10.000 tác vụ). Các tham
số cốt lõi: `corePoolSize`, `maximumPoolSize`, hàng đợi, `keepAliveTime`,
`RejectedExecutionHandler` quyết định cách pool phản ứng khi tải tăng. Factory method tiện lợi
của `Executors` (`newFixedThreadPool`, `newCachedThreadPool`) có rủi ro tiềm ẩn (hàng đợi không
giới hạn, hoặc số thread không giới hạn) — production nghiêm túc thường tự cấu hình
`ThreadPoolExecutor` trực tiếp.

## Interview Questions

- Thread Pool là gì? Tại sao không nên tạo thread bằng new Thread() liên tục?

**Mid**

- Giải thích các tham số `corePoolSize`, `maximumPoolSize`, hàng đợi trong `ThreadPoolExecutor`.

**Senior**

- Vì sao `Executors.newFixedThreadPool()` bị khuyến cáo tránh dùng trong production nghiêm
  túc? Đề xuất cấu hình thay thế an toàn hơn.
- Với tác vụ CPU-bound và I/O-bound, số lượng thread pool tối ưu nên khác nhau như thế nào? Vì
  sao?

## Exercises

- [ ] Chạy lại `ThreadPoolPerf` ở trên, xác nhận chênh lệch thời gian tương tự trên máy bạn.
- [ ] Tự tạo một `ThreadPoolExecutor` với `ArrayBlockingQueue` giới hạn 10 phần tử, thử nộp 100
      tác vụ chạy chậm, quan sát `RejectedExecutionException` khi hàng đợi đầy.
- [ ] Đo thời gian xử lý một tác vụ I/O-bound giả lập (`Thread.sleep`) với pool 4 thread so với
      pool 50 thread, quan sát pool lớn hơn xử lý nhanh hơn dù CPU chỉ có vài lõi.

## Cheat Sheet

| Factory method | corePoolSize | maximumPoolSize | Hàng đợi | Rủi ro |
| --- | --- | --- | --- | --- |
| `newFixedThreadPool(n)` | n | n | Không giới hạn | OOM nếu tải cao kéo dài |
| `newCachedThreadPool()` | 0 | Không giới hạn | `SynchronousQueue` | Quá nhiều thread nếu tải cao |
| `newSingleThreadExecutor()` | 1 | 1 | Không giới hạn | OOM nếu tải cao kéo dài |
| Tự cấu hình `ThreadPoolExecutor` | Tuỳ chỉnh | Tuỳ chỉnh | `ArrayBlockingQueue` (giới hạn) | Kiểm soát được, khuyến nghị production |

## References

- Java SE API Documentation — `java.util.concurrent.ThreadPoolExecutor`, `Executors`.
- Java Concurrency in Practice (Brian Goetz) — Chapter 8: Applying Thread Pools.
