---
tags:
  - Java
  - wait
  - Concurrency
---

# wait()

> Phase: Phase 3 — Concurrency
> Chapter slug: `wait`

## Metadata

```yaml
Chapter: wait()
Phase: Phase 3 — Concurrency
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 60%
Prerequisites:
  - Chapter 06 — Monitor
Used Later:
  - notify() (Chapter 08) — luôn đi cùng cặp với wait(), một chapter tách làm hai để không dồn quá nhiều khái niệm
  - Executor, Thread Pool (Chapter 15-16) — về sau hầu như luôn dùng công cụ cấp cao hơn thay vì tự viết wait/notify
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 06](06-monitor.md) — mỗi Monitor có một **wait set**, nơi thread "chủ động
> nhường lại lock" chờ được đánh thức.

Thử gọi `wait()` mà quên mất một điều kiện tiên quyết:

```java
static final Object lock = new Object();
lock.wait(); // gọi TRỰC TIẾP, không synchronized(lock) trước
```

Kết quả:

```
IllegalMonitorStateException - current thread is not owner
```

JVM từ chối thẳng thừng — `wait()` **bắt buộc** phải được gọi từ bên trong một khối
`synchronized` trên **chính object** mà `wait()` được gọi lên. Đây không phải một ràng buộc tuỳ
tiện: nó phản ánh đúng bản chất của `wait()` — "nhường lại lock **tôi đang giữ**" chỉ có ý
nghĩa nếu bạn thực sự đang giữ lock đó. Chapter này giải thích `wait()` hoạt động ra sao, và xây
dựng một bài toán kinh điển — producer-consumer — hoàn toàn từ `wait()`/`notify()` thô, trước
khi Phase 3 giới thiệu các công cụ cấp cao hơn thay thế nó trong thực tế.

## Interview Question (Central)

> `wait()` làm gì chính xác? Vì sao nó bắt buộc phải được gọi từ bên trong khối
> `synchronized`?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích chính xác `wait()` làm gì: nhả lock, chuyển thread vào wait set
      (`WAITING`/`TIMED_WAITING`, Chapter 02), chờ được đánh thức
- [ ] Tự tay xác nhận `IllegalMonitorStateException` khi gọi `wait()` sai cách
- [ ] Xây dựng hoàn chỉnh bài toán producer-consumer bằng `wait()`
- [ ] Biết vì sao `wait()` luôn phải đặt trong vòng lặp `while`, không phải `if`

## Prerequisites

- Chapter 06 — hiểu Monitor, entry set, wait set.

## Used Later

- **notify()** (Chapter 08) — hoàn thiện nốt nửa còn lại của cơ chế: cách đánh thức một thread
  đang trong wait set.
- **Executor, Thread Pool** (Chapter 15-16) — trong thực tế hiện đại, hiếm khi tự viết
  `wait()`/`notify()` thô — các công cụ cấp cao hơn (`BlockingQueue`, `CountDownLatch`) che
  giấu độ phức tạp này, nhưng hiểu nguyên lý gốc vẫn cần thiết để dùng đúng các công cụ đó.

## Problem

`synchronized` (Chapter 05) đảm bảo loại trừ lẫn nhau, nhưng không giải quyết được bài toán:
"thread A cần **chờ một điều kiện cụ thể** trở thành đúng, do thread B tạo ra, trước khi tiếp
tục" (ví dụ: consumer chờ có dữ liệu trong buffer trước khi lấy ra). Nếu A giữ lock trong khi
chờ (ví dụ vòng lặp kiểm tra liên tục — busy-wait), nó sẽ **chặn đứng** B — B không bao giờ
giành được lock để tạo ra điều kiện mà A đang chờ, gây deadlock tự tạo hoặc lãng phí CPU
nghiêm trọng.

## Concept

`wait()` (kế thừa từ `Object`, Phase 1 Chapter 09 — không phải `Thread`, xem lại
[Chapter 06 — Deep Dive](06-monitor.md)) giải quyết đúng vấn đề trên: khi một thread đang giữ
lock của một Monitor gọi `wait()`, nó **nhả lock ngay lập tức** (để thread khác có cơ hội giành
lock, tạo ra điều kiện mong đợi) **và** tạm dừng chính nó, chuyển vào wait set của Monitor đó
(trạng thái `WAITING`/`TIMED_WAITING`, Chapter 02) — cho tới khi có thread khác gọi
`notify()`/`notifyAll()` (Chapter 08) trên **cùng object**.

## Why?

Nếu không có `wait()`: cách duy nhất để "chờ một điều kiện" là **busy-wait** — vòng lặp liên
tục kiểm tra điều kiện (`while (!ready) { }`), tốn 100% một lõi CPU chỉ để "chờ", và tệ hơn nếu
đang giữ lock trong lúc chờ (chặn đứng luôn thread cần tạo ra điều kiện đó). `wait()` cho phép
thread **thực sự ngừng chạy** (không tốn CPU) trong lúc chờ, đồng thời **nhường lại lock** cho
thread khác — giải quyết cả hai vấn đề cùng lúc.

## How?

```java
synchronized (lock) {
    while (!conditionMet) {  // LUÔN dùng while, KHÔNG dùng if — xem Deep Dive
        lock.wait();          // nhả lock, chờ, khi được đánh thức GIÀNH LẠI lock rồi mới return
    }
    // ... điều kiện đã đúng, xử lý tiếp
}
```

## Visualization

```
Thread (Consumer)                          Monitor của "lock"
      │
synchronized(lock) { ─────────────────▶  GIÀNH LOCK
   while (buffer rỗng) {
       lock.wait() ─────────────────────▶  NHẢ LOCK, vào WAIT SET (WAITING)
                                                    │
                            (Producer giành lock,   │
                             thêm dữ liệu, notify()) │
                                                    ▼
       [được đánh thức] ◀───────────────────  quay lại ENTRY SET,
                                                TRANH GIÀNH LẠI lock
       [giành lại lock, kiểm tra lại điều kiện while]
   }
   // xử lý dữ liệu
}
```

## Example

**`IllegalMonitorStateException` khi gọi `wait()` sai cách:**

```java
public class WaitWithoutLock {
    static final Object lock = new Object();
    public static void main(String[] args) throws InterruptedException {
        try {
            lock.wait(); // GOI wait() MA KHONG synchronized(lock) truoc
        } catch (IllegalMonitorStateException e) {
            System.out.println("Loi: " + e.getClass().getSimpleName() + " - " + e.getMessage());
        }
    }
}
```

Kết quả thật (JDK 17):

```
Loi: IllegalMonitorStateException - current thread is not owner
```

**Producer-Consumer hoàn chỉnh** — bài toán kinh điển giải quyết trọn vẹn bằng `wait()`:

```java
import java.util.LinkedList;
import java.util.Queue;

public class ProducerConsumer {
    static final Queue<Integer> buffer = new LinkedList<>();
    static final int CAPACITY = 3;
    static final Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        Thread producer = new Thread(() -> {
            for (int i = 1; i <= 6; i++) {
                synchronized (lock) {
                    while (buffer.size() == CAPACITY) {
                        System.out.println("Producer: buffer day, wait()...");
                        try { lock.wait(); } catch (InterruptedException e) {}
                    }
                    buffer.add(i);
                    System.out.println("Producer: them " + i + ", buffer=" + buffer);
                    lock.notifyAll();
                }
            }
        });

        Thread consumer = new Thread(() -> {
            for (int i = 1; i <= 6; i++) {
                synchronized (lock) {
                    while (buffer.isEmpty()) {
                        System.out.println("Consumer: buffer rong, wait()...");
                        try { lock.wait(); } catch (InterruptedException e) {}
                    }
                    int val = buffer.poll();
                    System.out.println("Consumer: lay " + val + ", buffer=" + buffer);
                    lock.notifyAll();
                }
            }
        });

        producer.start();
        consumer.start();
        producer.join();
        consumer.join();
    }
}
```

Kết quả thật (rút gọn, JDK 17):

```
Producer: them 1, buffer=[1]
Producer: them 2, buffer=[1, 2]
Producer: them 3, buffer=[1, 2, 3]
Producer: buffer day, wait()...
Consumer: lay 1, buffer=[2, 3]
Producer: them 4, buffer=[2, 3, 4]
...
```

Producer tự động dừng lại (`wait()`) khi buffer đầy, và tự động tiếp tục ngay khi Consumer lấy
bớt ra — hoàn toàn không có busy-wait, không lãng phí CPU chờ đợi.

## Deep Dive

**Vì sao luôn phải dùng `while (!condition) { wait(); }`, không được dùng `if (!condition) { wait(); }`?**
Vì có hiện tượng gọi là **spurious wakeup** (đánh thức giả) — JVM/hệ điều hành **được phép**
(theo đặc tả) đánh thức một thread đang `wait()` mà **không có** `notify()` nào thực sự xảy ra
tương ứng (một chi tiết hiện thực ở tầng thấp, hiếm nhưng hợp lệ). Nếu dùng `if`, thread có thể
"tưởng" điều kiện đã đúng (vì vừa được đánh thức) và tiếp tục xử lý dù thực tế điều kiện **vẫn
sai**. Ngoài ra, ngay cả khi không có spurious wakeup: với **nhiều** thread cùng chờ trên một
lock (như nhiều consumer), khi một consumer được đánh thức, một consumer **khác** có thể đã
"nhanh tay" giành lock trước và lấy mất dữ liệu — buộc thread vừa được đánh thức phải **kiểm
tra lại** điều kiện, không được giả định nó vẫn đúng. `while` giải quyết cả hai vấn đề bằng
cách luôn kiểm tra lại điều kiện **sau khi** được đánh thức, trước khi làm bất cứ điều gì khác.

## Engineering Insight

**Vì sao `wait()` "nhả lock rồi chờ" phải là một thao tác nguyên tử (atomic), không phải hai
bước riêng biệt?** Nếu "nhả lock" và "bắt đầu chờ" là hai bước tách rời (ví dụ nếu lập trình
viên tự viết `unlock(); pause();`): có một khoảng hở cực nhỏ giữa hai bước, nơi một thread khác
có thể `notify()` **trước khi** thread đầu tiên kịp thực sự vào trạng thái chờ — tín hiệu
`notify()` đó sẽ "trôi qua trong hư không", không ai nhận được, và thread đang chờ sẽ chờ **mãi
mãi** dù điều kiện đã đúng từ lâu (gọi là "missed signal" — tín hiệu bị bỏ lỡ). `wait()` được
thiết kế như một thao tác **nguyên tử duy nhất** ở tầng JVM chính xác để loại bỏ khoảng hở nguy
hiểm này — nhả lock và bắt đầu chờ xảy ra như một hành động không thể chia cắt, không có
khoảng trống nào cho một `notify()` "trôi mất".

## Historical Note

`wait()`/`notify()`/`notifyAll()` tồn tại từ Java 1.0 (1996) — là cơ chế điều phối
(coordination) đa luồng nguyên thuỷ đầu tiên của Java, đi kèm với `synchronized`. Java 5 (2004)
bổ sung `java.util.concurrent.locks.Condition` (gắn với `ReentrantLock`, Chapter 10) — một
phiên bản tường minh, linh hoạt hơn của cùng ý tưởng (cho phép nhiều "điều kiện chờ" khác nhau
trên cùng một lock, thay vì chỉ một wait set duy nhất như Monitor). Từ Java 5 trở đi, cộng đồng
Java dần chuyển sang dùng các cấu trúc cấp cao hơn hẳn (`BlockingQueue`, Chapter 15-16) cho
phần lớn bài toán producer-consumer thực tế, thay vì tự viết `wait()`/`notify()` thô như ở
Example — nhưng hiểu nguyên lý gốc vẫn là nền tảng bắt buộc.

## Myth vs Reality

- **Myth:** "`wait()` có thể gọi ở bất kỳ đâu, miễn là trên một object hợp lệ."
  **Reality:** Xem Example — bắt buộc phải gọi từ bên trong khối `synchronized` trên đúng
  object đó, nếu không ném `IllegalMonitorStateException` ngay lập tức.

- **Myth:** "Dùng `if (!condition) wait();` là đủ, vì logic có vẻ tương đương `while`."
  **Reality:** Xem Deep Dive về spurious wakeup và tranh chấp giữa nhiều thread — `if` có thể
  dẫn tới xử lý sai dù điều kiện thực tế chưa đúng.

## Common Mistakes

- **Gọi `wait()` ngoài khối `synchronized`** — `IllegalMonitorStateException` ngay lập tức.
- **Dùng `if` thay vì `while`** quanh `wait()` — lỗi tinh vi, có thể không lộ ra ngay (đặc biệt
  nếu chỉ có đúng một thread chờ, spurious wakeup lại hiếm) nhưng tiềm ẩn nguy cơ nghiêm trọng.
- **Quên bắt `InterruptedException`** đúng cách — `wait()` là checked exception (Phase 1,
  Chapter 15), việc nuốt exception này mà không xử lý gì có thể che giấu tín hiệu ngắt quan
  trọng.

## Best Practices

- Luôn bọc `wait()` trong vòng lặp `while` kiểm tra điều kiện, không bao giờ dùng `if`.
- Luôn gọi `wait()`/`notify()`/`notifyAll()` bên trong khối `synchronized` trên đúng object.
- Trong code production hiện đại, ưu tiên `BlockingQueue` (Chapter 15-16) hoặc các cấu trúc
  `java.util.concurrent` cấp cao thay vì tự viết `wait()`/`notify()` thô — dễ viết đúng hơn
  nhiều, ít cạm bẫy hơn.

## Production Notes

**Vấn đề:** một hệ thống dùng `wait()`/`notify()` tự viết cho hàng đợi xử lý task nội bộ,
thỉnh thoảng có task "kẹt" vô thời hạn dù rõ ràng có task mới liên tục được thêm vào.

- **Triệu chứng:** một số worker thread ở trạng thái `WAITING` mãi mãi (Chapter 02), dù hàng
  đợi có dữ liệu mới liên tục.
- **Root cause:** thường là dùng `notify()` (chỉ đánh thức **một** thread ngẫu nhiên trong wait
  set) thay vì `notifyAll()` khi có nhiều thread cùng chờ — thread bị đánh thức "nhầm" có thể
  không phải thread cần được đánh thức để xử lý đúng task đang chờ, dẫn tới tín hiệu bị "lãng
  phí" — chi tiết đầy đủ về `notify()` vs `notifyAll()` ở [Chapter 08](08-notify.md).
- **Debug:** thread dump (Chapter 02) cho thấy nhiều thread `WAITING` đồng thời, trong khi
  logic mong đợi ít nhất một trong số chúng phải được đánh thức.
- **Solution:** đổi `notify()` thành `notifyAll()` nếu có nhiều loại điều kiện chờ khác nhau
  trên cùng một lock — chi tiết đầy đủ ở Chapter 08.
- **Prevention:** mặc định dùng `notifyAll()` trừ khi có lý do rõ ràng và đã kiểm chứng kỹ để
  dùng `notify()` (chỉ an toàn khi mọi thread chờ đều tương đương nhau — hiếm khi đúng trong
  thực tế phức tạp).

## Debug Checklist

- [ ] `IllegalMonitorStateException` khi gọi `wait()`? → kiểm tra có đang gọi từ bên trong
      đúng khối `synchronized` trên đúng object không.
- [ ] Thread kẹt ở `WAITING` mãi mãi dù điều kiện đã đúng? → kiểm tra có `notify()`/
      `notifyAll()` tương ứng được gọi không, và nếu dùng `notify()`, xem có đang đánh thức
      nhầm thread không (Chapter 08).
- [ ] Nghi ngờ code dùng `if` thay vì `while` quanh `wait()`? → sửa ngay, đây luôn là lỗi tiềm
      ẩn dù chưa lộ ra.

## Source Code Walkthrough

`Object.wait()` là một `native` method (Phase 1, Chapter 09) — hiện thực thật nằm trong HotSpot
C++, thao tác trực tiếp lên cấu trúc `ObjectMonitor` (đã nhắc ở Chapter 06): nhả lock hiện tại
(giảm "recursion count" — vì reentrant, Chapter 05, một thread có thể giữ lock nhiều lớp; JVM
nhớ số lớp để trả lại đúng khi được đánh thức), thêm chính thread vào wait set, rồi chặn thread
ở tầng hệ điều hành cho tới khi được đánh thức hoặc hết thời gian chờ (nếu dùng `wait(ms)`).

## Summary

`wait()` (kế thừa từ `Object`) khiến thread hiện tại **nhả lock** đang giữ và tạm dừng
(`WAITING`/`TIMED_WAITING`, Chapter 02), chuyển vào wait set của Monitor (Chapter 06) — bắt
buộc phải gọi từ bên trong `synchronized` trên đúng object, nếu không ném
`IllegalMonitorStateException` (đã xác nhận thực nghiệm). Luôn đặt `wait()` trong vòng lặp
`while` kiểm tra điều kiện, không dùng `if`, vì spurious wakeup và tranh chấp giữa nhiều thread
chờ có thể khiến điều kiện chưa thực sự đúng dù vừa được đánh thức. Kết hợp `wait()` với
`notify()`/`notifyAll()` (Chapter 08) giải quyết trọn vẹn bài toán producer-consumer kinh điển
— đã xây dựng và chạy đúng bằng thực nghiệm.

## Interview Questions

**Junior**

- `wait()` làm gì? Nó có giải phóng lock đang giữ không?
- Điều gì xảy ra nếu gọi `wait()` mà không có `synchronized` bao quanh?

**Mid**

- Vì sao phải dùng `while` thay vì `if` quanh `wait()`?
- Cài đặt bài toán producer-consumer bằng `wait()`/`notify()`.

**Senior**

- Giải thích "spurious wakeup" và "missed signal" — vì sao thiết kế `wait()` là một thao tác
  nguyên tử (nhả lock + chờ) lại quan trọng để tránh missed signal?
- So sánh `wait()`/`notify()` (Monitor, nguyên thuỷ) với `Condition` (gắn với
  `ReentrantLock`, Chapter 10) — `Condition` giải quyết hạn chế gì của Monitor truyền thống?

## Exercises

- [ ] Chạy lại đúng ví dụ `WaitWithoutLock`, xác nhận `IllegalMonitorStateException`.
- [ ] Chạy lại đúng ví dụ `ProducerConsumer` ở trên, quan sát Producer tự dừng khi buffer đầy
      và tự tiếp tục khi Consumer lấy bớt.
- [ ] Cố tình đổi `while` thành `if` trong ví dụ `ProducerConsumer` với nhiều hơn 1 consumer —
      thử tái hiện tình huống một consumer bị lỗi (`NoSuchElementException` khi `poll()` trên
      buffer rỗng) do điều kiện không được kiểm tra lại sau khi thức dậy.

## Cheat Sheet

```java
synchronized (lock) {
    while (!conditionMet) {   // LUÔN while, không if
        lock.wait();          // nhả lock, chờ, giành lại lock khi được đánh thức
    }
    // xử lý khi điều kiện đã đúng
}
```

| Yêu cầu | Bắt buộc |
| --- | --- |
| Gọi trong `synchronized` trên đúng object | Có (nếu không: `IllegalMonitorStateException`) |
| Bọc trong `while`, không `if` | Có (spurious wakeup, tranh chấp nhiều thread) |
| Xử lý `InterruptedException` | Có (checked exception) |

## References

- Java SE API Documentation — `java.lang.Object#wait()`.
- Java Language Specification (JLS) — Chapter 17.2: Wait Sets and Notification.
- Java Concurrency in Practice (Brian Goetz) — Chapter 14: Building Custom Synchronizers.
