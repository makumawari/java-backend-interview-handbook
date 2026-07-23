---
tags:
  - Java
  - Thread
  - Concurrency
  - Lifecycle
---

# Thread Lifecycle

> Phase: Phase 3 — Concurrency
> Chapter slug: `thread-lifecycle`

## Metadata

```yaml
Chapter: Thread Lifecycle
Phase: Phase 3 — Concurrency
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 65%
Prerequisites:
  - Chapter 01 — Thread
Used Later:
  - synchronized (Chapter 05) — trạng thái BLOCKED gắn liền với việc chờ monitor lock
  - wait()/notify() (Chapter 07-08) — trạng thái WAITING/TIMED_WAITING gắn liền với hai method này
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 01](01-thread.md) — bạn đã tạo Thread, gọi `start()`, gọi `join()`.

Một `Thread` không chỉ đơn giản "đang chạy" hoặc "đã xong" — nó trải qua nhiều trạng thái rất
cụ thể, và JVM cho bạn xem trực tiếp trạng thái đó qua `Thread.getState()`. Chạy đoạn code sau,
bạn sẽ thấy chính xác 4 trạng thái khác nhau chỉ trong một chương trình nhỏ — mỗi trạng thái
ứng với một tình huống rất đặc trưng mà bạn sẽ liên tục gặp lại xuyên suốt Phase 3.

## Interview Question (Central)

> Một Thread trong Java có những trạng thái nào? Sự khác biệt giữa `BLOCKED` và `WAITING` là
> gì?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Kể tên đủ 6 trạng thái của `Thread.State`: NEW, RUNNABLE, BLOCKED, WAITING,
      TIMED_WAITING, TERMINATED
- [ ] Tự tay tái hiện được từng trạng thái bằng code cụ thể, không chỉ học thuộc tên
- [ ] Phân biệt chính xác BLOCKED (chờ monitor lock) với WAITING/TIMED_WAITING (chủ động gọi
      `wait()`/`join()`/`sleep()`)
- [ ] Biết `RUNNABLE` trong Java **không** phân biệt "đang thực sự chạy trên CPU" và "sẵn sàng
      chạy, đang chờ tới lượt"

## Prerequisites

- Chapter 01 — đã biết `start()` tạo thread mới, `join()` chờ hoàn thành.

## Used Later

- **synchronized** (Chapter 05) — trạng thái `BLOCKED` xảy ra chính xác khi một thread chờ giữ
  được monitor lock của `synchronized`.
- **wait()/notify()** (Chapter 07-08) — trạng thái `WAITING`/`TIMED_WAITING` gắn liền trực
  tiếp với hai method này.

## Problem

Khi một chương trình đa luồng "treo" hoặc chạy chậm bất thường, câu hỏi đầu tiên cần trả lời là
"từng thread đang ở trạng thái gì — đang thực sự làm việc, hay đang bị chặn chờ thứ gì đó?".
Không có một mô hình trạng thái rõ ràng, không thể chẩn đoán chính xác vấn đề nằm ở đâu.

## Concept

`Thread.State` (enum — Phase 1, Chapter 17) định nghĩa 6 trạng thái:

- **NEW** — đã tạo (`new Thread(...)`), chưa gọi `start()`.
- **RUNNABLE** — đã `start()`, đang chạy **hoặc** đang chờ tới lượt CPU (JVM không phân biệt
  hai trường hợp này thành hai trạng thái riêng).
- **BLOCKED** — đang chờ giành được một monitor lock (`synchronized`, Chapter 05) mà thread
  khác đang giữ.
- **WAITING** — đang chờ vô thời hạn, do chủ động gọi `wait()` (không tham số),
  `join()` (không tham số), hoặc `LockSupport.park()`.
- **TIMED_WAITING** — giống `WAITING` nhưng có giới hạn thời gian (`sleep(ms)`,
  `wait(ms)`, `join(ms)`).
- **TERMINATED** — đã chạy xong `run()` (hoặc kết thúc do exception không bắt được).

## Why?

Nếu chỉ có "đang chạy" và "đã xong" (hai trạng thái): không thể phân biệt một thread bị
**tạm dừng có chủ đích** (đang `sleep()`, đang chờ dữ liệu qua `wait()`) với một thread bị
**chặn ngoài ý muốn** (đang `BLOCKED` chờ lock, có thể là dấu hiệu deadlock hoặc tranh chấp tài
nguyên nghiêm trọng). Sự phân biệt rõ ràng giữa `BLOCKED` và `WAITING`/`TIMED_WAITING` chính là
công cụ chẩn đoán quan trọng nhất khi debug vấn đề đa luồng (xem Production Notes).

## How?

```
NEW ──start()──▶ RUNNABLE ◀────────────┐
                     │                  │
       chờ lock      │      wait()/join() hết hạn/
       (synchronized) │      notify()/interrupt()
                     ▼                  │
                  BLOCKED           WAITING/TIMED_WAITING
                     │                  ▲
                     └── giành được lock ┘
                     │
                     ▼ (run() kết thúc)
                TERMINATED
```

## Visualization

```
Thread t = new Thread(...)          → NEW
t.start()                            → RUNNABLE (đang chạy hoặc chờ CPU)
   trong t.run(): synchronized(lock) → BLOCKED (nếu lock đang bị giữ)
   trong t.run(): lock.wait()        → WAITING
   trong t.run(): lock.wait(500)     → TIMED_WAITING
run() kết thúc                       → TERMINATED
```

## Example

Tái hiện đủ 4 trạng thái quan sát được (trừ NEW, RUNNABLE là hiển nhiên) trong một chương
trình:

```java
public class LifecycleStates {
    static final Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            synchronized (lock) {
                try { lock.wait(500); } catch (InterruptedException e) {}
            }
        });

        System.out.println("Truoc start(): " + t.getState());
        t.start();
        Thread.sleep(50);
        System.out.println("Sau start(), dang wait(): " + t.getState());

        Thread blocker = new Thread(() -> {
            synchronized (lock) {
                try { Thread.sleep(1000); } catch (InterruptedException e) {}
            }
        });
        blocker.start();
        Thread.sleep(50);

        Thread t2 = new Thread(() -> {
            synchronized (lock) { }
        });
        t2.start();
        Thread.sleep(50);
        System.out.println("t2 dang cho lock (BLOCKED): " + t2.getState());

        t.join();
        blocker.join();
        t2.join();
        System.out.println("Sau khi xong: " + t.getState());
    }
}
```

Kết quả thật (JDK 17):

```
Truoc start(): NEW
Sau start(), dang wait(): TIMED_WAITING
t2 dang cho lock (BLOCKED): BLOCKED
Sau khi xong: TERMINATED
```

Bốn trạng thái, bốn tình huống code cụ thể, khớp chính xác với lý thuyết ở phần Concept —
`t` gọi `lock.wait(500)` (có tham số thời gian) cho `TIMED_WAITING`; `t2` cố `synchronized`
trong khi `blocker` đang giữ lock cho `BLOCKED`.

## Deep Dive

**Vì sao Java gộp "đang thực sự chạy trên CPU" và "sẵn sàng chạy, đang xếp hàng chờ CPU" thành
chung một trạng thái `RUNNABLE`?** Vì việc CPU nào, lúc nào, thực thi thread nào là quyết định
của **bộ lập lịch hệ điều hành** (OS scheduler) — JVM không có (và không cần) khả năng biết
chính xác thời điểm một thread `RUNNABLE` thực sự đang chiếm CPU hay chỉ đang chờ tới lượt. Cả
hai đều là "sẵn sàng thực thi, không bị chặn bởi bất kỳ lý do logic nào (lock, wait...)" — đây
là mức độ chi tiết mà JVM quan tâm; chi tiết lập lịch CPU cụ thể hơn thuộc về tầng hệ điều hành,
nằm ngoài phạm vi mà `Thread.State` cần mô tả.

## Engineering Insight

**Vì sao phân biệt `BLOCKED` với `WAITING` lại là công cụ chẩn đoán quan trọng bậc nhất khi
debug đa luồng?** Vì hai trạng thái này chỉ ra hai **nguyên nhân gốc hoàn toàn khác nhau** khi
một thread "không làm gì cả": `BLOCKED` nghĩa là thread **muốn** chạy tiếp nhưng bị **chặn bởi
tranh chấp tài nguyên** (một thread khác đang giữ lock nó cần) — đây là dấu hiệu trực tiếp cần
điều tra tranh chấp lock, có thể dẫn tới deadlock. `WAITING`/`TIMED_WAITING` nghĩa là thread
**chủ động** chọn tạm dừng (đang chờ tín hiệu, chờ thời gian, chờ thread khác hoàn thành) — đây
thường là hành vi **có chủ đích**, không phải dấu hiệu sự cố. Khi lấy thread dump (một kỹ thuật
debug thực tế, sẽ hữu ích ở Phase 8) để chẩn đoán một hệ thống "treo", việc đầu tiên cần làm là
đếm xem có bao nhiêu thread đang `BLOCKED` — nếu có nhiều thread cùng `BLOCKED` chờ cùng một
lock, đó gần như chắc chắn là điểm nghẽn (bottleneck) cần điều tra ngay.

## Historical Note

`Thread.State` (enum tường minh, dễ đọc) được thêm vào Java 5 (2004) — trước đó, xác định
chính xác trạng thái một thread khó khăn hơn nhiều, chủ yếu dựa vào các method rời rạc như
`isAlive()`. Việc chuẩn hoá thành 6 trạng thái rõ ràng, có thể truy vấn trực tiếp qua
`getState()`, đi cùng đợt với `java.util.concurrent` (JSR 166, Executor/Future — Chapter
15-16) — một nỗ lực tổng thể hiện đại hoá công cụ làm việc với đa luồng trong Java.

## Myth vs Reality

- **Myth:** "`BLOCKED` và `WAITING` về cơ bản là một, đều nghĩa là 'thread đang không làm gì
  cả'."
  **Reality:** Khác nhau về **nguyên nhân gốc** — `BLOCKED` là bị chặn ngoài ý muốn bởi tranh
  chấp lock; `WAITING` là chủ động tạm dừng. Xem Engineering Insight về ý nghĩa chẩn đoán khác
  biệt hoàn toàn giữa hai điều này.

- **Myth:** "`RUNNABLE` nghĩa là thread chắc chắn đang thực thi trên CPU ngay lúc này."
  **Reality:** `RUNNABLE` chỉ nghĩa là "không bị chặn bởi lý do logic nào" — có thể đang thực
  sự chạy, hoặc đang xếp hàng chờ CPU, JVM không phân biệt hai trường hợp này.

## Common Mistakes

- **Dùng vòng lặp kiểm tra `getState()` liên tục (busy-wait/polling) để "chờ" một thread tới
  trạng thái mong muốn** — lãng phí CPU, nên dùng `join()` hoặc cơ chế đồng bộ đúng đắn
  (`wait()`/`notify()`, Chapter 07-08; hoặc `CountDownLatch`, Chapter 16).
- **Nhầm lẫn `Thread.State` (mức JVM) với trạng thái thật của OS thread bên dưới** — chúng liên
  quan nhưng không phải luôn ánh xạ 1-1 hoàn toàn (chi tiết OS-level nằm ngoài phạm vi
  `Thread.State`).

## Best Practices

- Dùng `getState()` chủ yếu cho mục đích quan sát/debug, không dùng nó để điều khiển logic
  chương trình (ví dụ vòng lặp chờ một trạng thái cụ thể).
- Khi debug hệ thống đa luồng "treo", luôn bắt đầu bằng cách phân loại thread theo trạng thái
  (đặc biệt đếm số thread `BLOCKED`) trước khi đi sâu vào từng thread cụ thể.

## Production Notes

**Vấn đề:** một service đột nhiên không phản hồi bất kỳ request nào, dù process vẫn "sống"
(không crash), CPU usage gần như bằng 0.

- **Triệu chứng:** ứng dụng "treo" hoàn toàn, không log lỗi nào mới, CPU thấp bất thường (nếu
  cao bất thường thay vào đó, đó là dấu hiệu khác — busy loop, không phải deadlock).
- **Root cause:** nghi ngờ hàng đầu là **deadlock** hoặc tranh chấp lock nghiêm trọng — nhiều
  thread đang `BLOCKED` chờ lock lẫn nhau (chi tiết đầy đủ về deadlock sẽ học ở Chapter 09-10),
  hoặc `WAITING` vô thời hạn chờ một tín hiệu không bao giờ tới.
- **Debug:** lấy **thread dump** (`jstack <pid>` hoặc `jcmd <pid> Thread.print`) — công cụ này
  liệt kê trạng thái **và** stack trace của mọi thread tại một thời điểm. Tìm các thread
  `BLOCKED`, xem chúng đang chờ lock nào, ai đang giữ lock đó.
- **Solution:** tuỳ nguyên nhân cụ thể tìm được qua thread dump — thường là sửa logic khoá
  (thứ tự giành lock không nhất quán giữa các thread, nguyên nhân kinh điển của deadlock).
- **Prevention:** thiết kế logic khoá nhất quán (luôn giành lock theo cùng một thứ tự ở mọi
  nơi trong codebase); giám sát định kỳ số lượng thread `BLOCKED` như một chỉ số sức khoẻ hệ
  thống.

## Debug Checklist

- [ ] Hệ thống "treo", CPU thấp? → lấy thread dump, đếm thread `BLOCKED`/`WAITING`, tìm điểm
      nghẽn.
- [ ] Nhiều thread cùng `BLOCKED` chờ cùng một lock? → điều tra ai đang giữ lock đó và tại sao
      giữ lâu, nghi ngờ deadlock nếu có chu trình chờ lẫn nhau.
- [ ] Thread ở `WAITING` mãi mãi không tiếp tục? → kiểm tra có `notify()`/`notifyAll()` tương
      ứng nào bị bỏ sót không (Chapter 07-08).

## Source Code Walkthrough

`Thread.getState()` không đơn thuần đọc một field — nó tổng hợp thông tin từ trạng thái nội bộ
của JVM (bao gồm việc thread có đang giữ/chờ monitor lock cụ thể nào không) tại đúng thời điểm
gọi. Đây là lý do trạng thái trả về là một "ảnh chụp tức thời" (snapshot) — có thể đã đổi ngay
sau khi bạn đọc được nó, đặc biệt với các thread `RUNNABLE` đang hoạt động tích cực.

## Summary

`Thread.State` có 6 trạng thái: NEW (đã tạo, chưa start), RUNNABLE (đang chạy hoặc chờ CPU —
JVM không phân biệt), BLOCKED (chờ giành monitor lock), WAITING/TIMED_WAITING (chủ động tạm
dừng, vô thời hạn hoặc có giới hạn), TERMINATED (đã xong). Phân biệt BLOCKED (bị chặn ngoài ý
muốn bởi tranh chấp lock) với WAITING (chủ động tạm dừng) là công cụ chẩn đoán quan trọng nhất
khi debug hệ thống đa luồng "treo" — thread dump (`jstack`) là công cụ thực tế để quan sát
trạng thái này trong production.

## Interview Questions

**Junior**

- Kể tên các trạng thái của một Thread trong Java.
- Trạng thái nào một Thread có ngay sau khi tạo (`new Thread(...)`), trước khi gọi `start()`?

**Mid**

- Phân biệt `BLOCKED` và `WAITING`. Cho ví dụ cụ thể tình huống nào dẫn tới mỗi trạng thái.
- `RUNNABLE` có nghĩa là thread chắc chắn đang chạy trên CPU không? Giải thích.

**Senior**

- Một hệ thống production "treo" hoàn toàn. Trình bày quy trình chẩn đoán, bắt đầu từ việc
  phân loại trạng thái thread.
- Giải thích vì sao Java không có trạng thái riêng biệt cho "đang thực sự chạy trên CPU" khác
  với "sẵn sàng chạy, đang chờ CPU" — đây là giới hạn thiết kế hay lựa chọn hợp lý?

## Exercises

- [ ] Chạy lại đúng ví dụ `LifecycleStates` ở trên, xác nhận cả 4 trạng thái quan sát được.
- [ ] Viết một chương trình tạo hai thread cố tình giành lock theo thứ tự ngược nhau (thread A
      giành lock1 rồi lock2; thread B giành lock2 rồi lock1) — quan sát cả hai bị `BLOCKED`
      mãi mãi (deadlock), lấy thread dump bằng `jstack` để xác nhận.
- [ ] Viết một chương trình có thread ở trạng thái `WAITING` (gọi `wait()` không tham số) mà
      không bao giờ được `notify()` — quan sát nó "treo" mãi mãi, khác với `BLOCKED` ở chỗ
      không có tranh chấp lock nào cả.

## Cheat Sheet

| Trạng thái | Khi nào |
| --- | --- |
| `NEW` | Đã tạo, chưa `start()` |
| `RUNNABLE` | Đang chạy hoặc chờ CPU |
| `BLOCKED` | Chờ giành monitor lock (`synchronized`) |
| `WAITING` | `wait()`/`join()` không tham số, chờ vô thời hạn |
| `TIMED_WAITING` | `sleep(ms)`/`wait(ms)`/`join(ms)`, chờ có giới hạn |
| `TERMINATED` | `run()` đã kết thúc |

**Công cụ debug:** `jstack <pid>` / `jcmd <pid> Thread.print` — thread dump, xem trạng thái +
stack trace mọi thread.

## References

- Java SE API Documentation — `java.lang.Thread.State`.
- Oracle: "The Java Tutorials — Thread States".
