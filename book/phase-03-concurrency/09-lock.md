---
tags:
  - Java
  - Lock
  - Concurrency
---

# Lock (Java-level Locking)

> Phase: Phase 3 — Concurrency
> Chapter slug: `lock`

## Metadata

```yaml
Chapter: Lock (Java-level Locking)
Phase: Phase 3 — Concurrency
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 55%
Prerequisites:
  - Chapter 05 — synchronized
  - Chapter 08 — notify()
Used Later:
  - ReentrantLock (Chapter 10) — implementation chính của interface Lock học ở đây
  - ReadWriteLock (Chapter 11) — một biến thể chuyên biệt của cùng interface Lock
Estimated Reading: 20 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 05](05-synchronized.md) — `synchronized` giải quyết trọn vẹn bài toán đếm
> mà `volatile` không giải quyết được.

Nhưng thử tình huống này: một thread đang chờ giành một lock `synchronized` bị **giữ quá lâu**
(có thể do bug ở thread khác). Bạn muốn "huỷ" việc chờ đó — gọi `interrupt()` lên thread đang
chờ. Chuyện gì xảy ra?

```
Da goi interrupt() len waiter1 dang BLOCKED, trang thai: BLOCKED
waiter1 gianh duoc lock (khong bao gio interrupt duoc trong luc BLOCKED)
```

**Không có gì xảy ra cả** — thread vẫn `BLOCKED` (Chapter 02), tín hiệu `interrupt()` bị hoàn
toàn phớt lờ, thread chỉ có thể tiếp tục khi **thực sự giành được lock**, dù bao lâu đi nữa.
Đây là một giới hạn cố hữu của `synchronized` — không có cách nào "bỏ cuộc" khi đang chờ. Thử
đúng kịch bản đó với `Lock.lockInterruptibly()`:

```
waiter2 BI NGAT thanh cong trong luc cho lockInterruptibly()!
```

Chapter này giới thiệu `java.util.concurrent.locks.Lock` — một interface **tường minh** hoá
đúng ý tưởng "loại trừ lẫn nhau" của `synchronized`, nhưng khắc phục các giới hạn cố hữu như
trên.

## Interview Question (Central)

> `Lock` (interface, `java.util.concurrent.locks`) khác `synchronized` như thế nào? Nó khắc
> phục những giới hạn cụ thể nào của `synchronized`?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Kể tên 3 giới hạn cố hữu của `synchronized` mà `Lock` khắc phục: không thể ngắt
      (interrupt) khi đang chờ, không thể "thử giành lock rồi bỏ cuộc" (`tryLock`), phạm vi
      khoá bắt buộc theo cấu trúc khối lệnh (không thể giành ở một method, nhả ở method khác)
- [ ] Tự tay chứng minh thread `BLOCKED` trên `synchronized` không thể bị `interrupt()`, trong
      khi thread chờ `lockInterruptibly()` thì có thể
- [ ] Biết cấu trúc bắt buộc khi dùng `Lock`: luôn `unlock()` trong khối `finally`

## Prerequisites

- Chapter 05 — hiểu `synchronized` đảm bảo mutual exclusion.
- Chapter 08 — hiểu `wait()`/`notify()`, để so sánh với `Condition` (sẽ gặp ở Chapter 10) sau
  này.

## Used Later

- **ReentrantLock** (Chapter 10) — implementation chính, phổ biến nhất của interface `Lock`
  học ở đây.
- **ReadWriteLock** (Chapter 11) — một biến thể chuyên biệt, tách riêng lock cho đọc và ghi.

## Problem

`synchronized` (Chapter 05) là một cơ chế khoá **cứng nhắc** theo nhiều nghĩa: phạm vi khoá
luôn gắn chặt với cấu trúc khối lệnh (`{ }`) — không thể giành lock ở một chỗ, nhả nó ở một chỗ
hoàn toàn khác trong code; một thread `BLOCKED` chờ `synchronized` không có cách nào "bỏ cuộc"
(không `tryLock`, không thể `interrupt`); và chỉ có **một** chiến lược chờ duy nhất (không thể
tuỳ chỉnh fairness — thứ tự ai giành lock trước). Nhiều bài toán thực tế cần sự linh hoạt hơn
thế.

## Concept

`java.util.concurrent.locks.Lock` là một **interface**, biểu diễn tường minh khái niệm "một
khoá loại trừ lẫn nhau" — tách rời hoàn toàn khỏi cú pháp `synchronized` (không còn là từ khoá
ngôn ngữ, mà là một object thông thường với các method `lock()`, `unlock()`, `tryLock()`,
`lockInterruptibly()`).

## Why?

Tách "khoá" thành một **object tường minh** (thay vì cú pháp ẩn định gắn với `synchronized`)
cho phép linh hoạt hơn nhiều: giành lock ở method này, nhả ở method khác (nếu thiết kế yêu
cầu — dù hiếm khi nên làm); thử giành lock với `tryLock()` và **bỏ cuộc ngay** nếu không giành
được, thay vì `BLOCKED` vô thời hạn; và quan trọng nhất — cho phép **ngắt** (interrupt) một
thread đang chờ giành lock, một khả năng `synchronized` hoàn toàn không có.

## How?

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

Lock lock = new ReentrantLock(); // implementation chính, Chapter 10

lock.lock();
try {
    // ... vùng găng (critical section)
} finally {
    lock.unlock(); // BẮT BUỘC trong finally — Lock KHÔNG tự nhả như synchronized
}
```

Ba khả năng mà `synchronized` không có:

```java
if (lock.tryLock()) {              // thử giành NGAY, không chờ nếu không được
    try { /* ... */ } finally { lock.unlock(); }
} else {
    // làm việc khác thay vì chờ
}

try {
    lock.lockInterruptibly();      // chờ giành lock, NHƯNG có thể bị interrupt()
    try { /* ... */ } finally { lock.unlock(); }
} catch (InterruptedException e) { /* xử lý việc bị ngắt */ }
```

## Visualization

```
synchronized (lock) { ... }        Lock (tường minh)
        │                                  │
   Phạm vi khoá = khối lệnh          lock()/unlock() TÁCH RỜI,
   (cố định theo cú pháp)             linh hoạt hơn (nhưng RỦI RO
        │                              hơn nếu quên unlock())
   Thread BLOCKED chờ:                Thread chờ lockInterruptibly():
   KHÔNG interrupt() được             CÓ THỂ interrupt() được
        │                                  │
   KHÔNG có tryLock()                 CÓ tryLock() — thử rồi bỏ cuộc
```

## Example

Chứng minh trực tiếp hai giới hạn/khả năng đối lập:

```java
import java.util.concurrent.locks.ReentrantLock;

public class InterruptibleLockDemo {
    static final Object syncLock = new Object();
    static final ReentrantLock explicitLock = new ReentrantLock();

    public static void main(String[] args) throws InterruptedException {
        // Phần 1: thread BLOCKED trên synchronized KHÔNG THỂ bị ngắt
        Thread holder1 = new Thread(() -> {
            synchronized (syncLock) {
                try { Thread.sleep(2000); } catch (InterruptedException e) {}
            }
        });
        holder1.start();
        Thread.sleep(100);

        Thread waiter1 = new Thread(() -> {
            synchronized (syncLock) { System.out.println("waiter1 gianh duoc lock"); }
        });
        waiter1.start();
        Thread.sleep(100);
        waiter1.interrupt(); // KHÔNG CÓ TÁC DỤNG
        System.out.println("Da goi interrupt(), trang thai: " + waiter1.getState());
        waiter1.join();

        // Phần 2: thread chờ lockInterruptibly() CÓ THỂ bị ngắt
        explicitLock.lock();
        Thread waiter2 = new Thread(() -> {
            try {
                explicitLock.lockInterruptibly();
            } catch (InterruptedException e) {
                System.out.println("waiter2 BI NGAT thanh cong!");
            }
        });
        waiter2.start();
        Thread.sleep(200);
        waiter2.interrupt(); // CÓ TÁC DỤNG
        waiter2.join();
        explicitLock.unlock();
    }
}
```

Kết quả thật (JDK 17):

```
Da goi interrupt() len waiter1 dang BLOCKED, trang thai: BLOCKED
waiter1 gianh duoc lock (khong bao gio interrupt duoc trong luc BLOCKED)
waiter2 BI NGAT thanh cong trong luc cho lockInterruptibly()!
```

`waiter1` phớt lờ hoàn toàn tín hiệu `interrupt()`, tiếp tục `BLOCKED` cho tới khi thực sự
giành được lock. `waiter2` phản ứng ngay với `interrupt()`, thoát khỏi việc chờ bằng
`InterruptedException` — khả năng hoàn toàn không có với `synchronized`.

## Deep Dive

**Vì sao `unlock()` bắt buộc phải nằm trong `finally`, trong khi `synchronized` không cần lo
lắng điều này?** Vì `synchronized` là cú pháp **ngôn ngữ** — trình biên dịch tự động chèn
`monitorexit` (Phase 1... à, Phase 3 Chapter 05 — Source Code Walkthrough) vào mọi đường thoát
khỏi khối, kể cả khi có exception, không thể quên. `Lock` chỉ là một **object thông thường** —
`unlock()` là một lời gọi method bình thường, hoàn toàn không có gì tự động đảm bảo nó được gọi
nếu code giữa `lock()` và `unlock()` ném exception. Quên `unlock()` trong `finally` khiến lock
đó **không bao giờ được nhả** — mọi thread khác chờ nó sẽ `BLOCKED`/chờ mãi mãi, một lỗi
nghiêm trọng hơn hẳn bất kỳ vấn đề nào `synchronized` có thể gây ra, chính là cái giá phải trả
cho sự linh hoạt của `Lock`.

## Engineering Insight

**Đánh đổi cốt lõi giữa `synchronized` và `Lock` là gì — vì sao không phải lúc nào cũng nên
dùng `Lock` "vì nó mạnh hơn"?** `synchronized` **an toàn hơn theo mặc định** (không thể quên
nhả lock, cú pháp gọn, dễ đọc) nhưng **kém linh hoạt hơn**. `Lock` **linh hoạt hơn** (tryLock,
interrupt, nhiều `Condition` — Chapter 10) nhưng **rủi ro hơn** (phải tự kỷ luật gọi
`unlock()` đúng cách, dễ viết sai hơn nếu không cẩn thận). Đây là đánh đổi kinh điển giữa
"an toàn mặc định, ít linh hoạt" và "linh hoạt tối đa, cần kỷ luật cao" — xuất hiện lặp đi lặp
lại trong thiết kế API (một biến thể khác của cùng nguyên tắc đã thấy ở
[Phase 1 Chapter 11](../phase-01-foundation/11-immutable.md): API càng cho phép làm nhiều thứ,
càng dễ dùng sai nếu không cẩn thận). Nguyên tắc thực dụng: mặc định `synchronized` cho phần
lớn trường hợp đơn giản; chỉ chuyển sang `Lock` khi thực sự cần một trong các khả năng nó cung
cấp mà `synchronized` không có (interrupt, tryLock, nhiều Condition).

## Historical Note

`java.util.concurrent.locks.Lock` ra đời cùng đợt JSR 166 (Java 5, 2004) — cùng thời điểm với
`ExecutorService` (Chapter 15-16), `Atomic` (Chapter 12), và toàn bộ `java.util.concurrent` —
một nỗ lực tổng thể của Doug Lea và cộng sự nhằm cung cấp các công cụ concurrency cấp cao, tinh
vi hơn hẳn so với bộ công cụ nguyên thuỷ (`synchronized`, `wait`/`notify`) đã tồn tại từ Java
1.0 nhưng bộc lộ nhiều giới hạn qua gần một thập kỷ sử dụng thực tế.

## Myth vs Reality

- **Myth:** "`Lock` luôn tốt hơn `synchronized`, nên dùng `Lock` cho mọi trường hợp."
  **Reality:** `Lock` linh hoạt hơn nhưng rủi ro hơn (dễ quên `unlock()`) — xem Engineering
  Insight. `synchronized` vẫn là lựa chọn tốt, đơn giản, an toàn hơn cho phần lớn trường hợp
  không cần các khả năng đặc biệt của `Lock`.

- **Myth:** "Có thể ngắt (interrupt) một thread đang chờ bất kỳ loại khoá nào trong Java."
  **Reality:** Xem Example — thread `BLOCKED` trên `synchronized` **không thể** bị ngắt.
  Chỉ `lockInterruptibly()` của `Lock` mới cho phép điều này.

## Common Mistakes

- **Quên `unlock()` trong `finally`** — lock bị giữ vĩnh viễn nếu có exception xảy ra giữa
  `lock()` và `unlock()`, gây "deadlock" thực chất cho mọi thread khác chờ lock đó.
- **Gọi `unlock()` mà chưa từng `lock()` thành công** (ví dụ đặt `unlock()` sai vị trí, ngoài
  khối try tương ứng) — ném `IllegalMonitorStateException` tương tự lỗi đã gặp ở Chapter 07.
- **Dùng `Lock` khi `synchronized` là đủ** — thêm độ phức tạp và rủi ro không cần thiết.

## Best Practices

- Luôn viết theo đúng khuôn mẫu: `lock.lock(); try { ... } finally { lock.unlock(); }` —
  không có ngoại lệ.
- Chỉ chuyển từ `synchronized` sang `Lock` khi thực sự cần: `tryLock()`, `lockInterruptibly()`,
  hoặc nhiều `Condition` trên cùng một lock (Chapter 10).
- Không gọi `lock()` bên trong khối `try` — nếu `lock()` tự nó ném exception (hiếm nhưng có
  thể), `finally` sẽ cố `unlock()` một lock chưa từng giành được, gây lỗi.

## Production Notes

**Vấn đề:** một service dùng `ReentrantLock` để bảo vệ một tài nguyên dùng chung, thỉnh thoảng
"treo" hoàn toàn, mọi request liên quan tới tài nguyên đó không bao giờ hoàn thành.

- **Triệu chứng:** service không crash, nhưng một nhóm chức năng cụ thể (liên quan tới tài
  nguyên được bảo vệ bởi lock đó) hoàn toàn ngừng phản hồi.
- **Root cause:** một đường thực thi hiếm gặp (exception path) trong code giữa `lock()` và
  `unlock()` không được bọc đúng bằng `try-finally` — khi exception đó xảy ra, `unlock()` không
  bao giờ được gọi, lock bị giữ vĩnh viễn.
- **Debug:** thread dump (Chapter 02) cho thấy nhiều thread chờ cùng một lock, nhưng **không
  thread nào** đang thực sự giữ nó theo cách "đang xử lý bình thường" (thread giữ lock có thể
  đã kết thúc do exception, không giải phóng lock).
- **Solution:** sửa lại đúng khuôn mẫu `lock(); try { } finally { unlock(); }`, rà soát toàn bộ
  các nơi dùng `Lock` trong codebase để đảm bảo tuân thủ.
- **Prevention:** static analysis (nhiều công cụ có rule cảnh báo thiếu `finally` quanh
  `Lock.unlock()`); code review nghiêm ngặt cho mọi đoạn code dùng `Lock` tường minh.

## Debug Checklist

- [ ] Hệ thống "treo" liên quan tới một tài nguyên cụ thể, không crash? → nghi ngờ lock bị giữ
      vĩnh viễn do thiếu `finally`, kiểm tra thread dump.
- [ ] `IllegalMonitorStateException` khi gọi `unlock()`? → kiểm tra có đang gọi `unlock()` mà
      chưa từng `lock()` thành công không (vị trí đặt sai so với khối try).
- [ ] Cần huỷ một thao tác chờ lock quá lâu? → xác nhận đang dùng `Lock.lockInterruptibly()`,
      không phải `synchronized` (không hỗ trợ).

## Source Code Walkthrough

`java.util.concurrent.locks.Lock` chỉ là một **interface** — không có hiện thực riêng. Mọi
hiện thực thực tế (`ReentrantLock`, `ReentrantReadWriteLock`, Chapter 10-11) đều xây trên một
lớp nền tảng chung tên `AbstractQueuedSynchronizer` (AQS) — một framework nội bộ (do Doug Lea
thiết kế) cung cấp sẵn cơ chế hàng đợi thread chờ (dựa trên CAS, Chapter 13, không phải
Monitor của `synchronized`) mà các lock cụ thể chỉ cần định nghĩa "trạng thái giành được lock
nghĩa là gì" lên trên đó — một kiến trúc rất được đánh giá cao, sẽ gặp lại chi tiết hơn ở
Chapter 10.

## Summary

`java.util.concurrent.locks.Lock` (Java 5+) là interface tường minh hoá khái niệm loại trừ lẫn
nhau, khắc phục ba giới hạn cố hữu của `synchronized`: cho phép `tryLock()` (thử rồi bỏ cuộc),
`lockInterruptibly()` (có thể bị ngắt khi đang chờ — đã chứng minh `synchronized` không làm
được điều này), và phạm vi khoá linh hoạt hơn (không bắt buộc theo cấu trúc khối lệnh). Đổi
lại, `Lock` đòi hỏi kỷ luật cao hơn — `unlock()` phải luôn nằm trong `finally`, không có gì tự
động đảm bảo như `synchronized`. Mặc định vẫn nên chọn `synchronized` cho trường hợp đơn giản;
chỉ chuyển sang `Lock` khi thực sự cần một trong các khả năng đặc biệt của nó.

## Interview Questions

**Junior**

- `Lock` (interface) khác `synchronized` như thế nào?
- Vì sao phải gọi `unlock()` trong khối `finally`?

**Mid**

- `tryLock()` làm gì? Cho một use case cụ thể cần dùng nó thay vì `lock()` thông thường.
- Chứng minh (bằng lý luận hoặc code) rằng thread `BLOCKED` trên `synchronized` không thể bị
  `interrupt()`.

**Senior**

- Phân tích đánh đổi giữa `synchronized` (an toàn mặc định, kém linh hoạt) và `Lock` (linh
  hoạt, cần kỷ luật cao) — khi nào bạn sẽ chọn cái nào trong một dự án thực tế?
- Một service "treo" do lock bị giữ vĩnh viễn sau một exception hiếm gặp. Trình bày quy trình
  điều tra và giải pháp phòng ngừa lâu dài.

## Exercises

- [ ] Chạy lại đúng ví dụ `InterruptibleLockDemo` ở trên, xác nhận kết quả giống hệt.
- [ ] Viết một đoạn code cố tình quên `unlock()` trong `finally` (chỉ gọi `unlock()` ở cuối
      khối try bình thường), rồi cho một exception xảy ra giữa chừng — quan sát lock bị giữ
      vĩnh viễn, một thread khác chờ nó `BLOCKED` mãi mãi.
- [ ] Viết một đoạn code dùng `tryLock(500, TimeUnit.MILLISECONDS)` (`tryLock` có timeout) để
      "thử giành lock trong tối đa 500ms rồi bỏ cuộc" — xác nhận hành vi đúng khi lock đang bị
      giữ bởi thread khác lâu hơn 500ms.

## Cheat Sheet

| Khả năng | `synchronized` | `Lock` |
| --- | --- | --- |
| Mutual exclusion | Có | Có |
| Tự động nhả lock | Có (trình biên dịch đảm bảo) | Không (phải tự `finally`) |
| `tryLock()` (thử rồi bỏ cuộc) | Không | Có |
| Ngắt được khi đang chờ | Không | Có (`lockInterruptibly()`) |
| Nhiều điều kiện chờ trên 1 lock | Không (1 wait set) | Có (`Condition`, Chapter 10) |

```java
lock.lock();
try { /* vùng găng */ }
finally { lock.unlock(); } // BẮT BUỘC
```

## References

- Java SE API Documentation — `java.util.concurrent.locks.Lock`.
- JSR 166: Concurrency Utilities.
- Java Concurrency in Practice (Brian Goetz) — Chapter 13: Explicit Locks.
