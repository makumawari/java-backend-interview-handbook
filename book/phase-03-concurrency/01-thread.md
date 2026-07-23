---
tags:
  - Java
  - Thread
  - Concurrency
---

# Thread

> Phase: Phase 3 — Concurrency
> Chapter slug: `thread`

## Metadata

```yaml
Chapter: Thread
Phase: Phase 3 — Concurrency
Difficulty: ★★★
Importance: ★★★★★
Interview Frequency: 80%
Prerequisites:
  - Phase 1, Chapter 04 — Runtime Data Areas (mỗi thread có JVM Stack riêng)
  - Phase 1, Chapter 05 — Execution Engine
Used Later:
  - Toàn bộ Phase 3 — mọi chapter còn lại xây dựng trên khái niệm Thread ở đây
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Phase 1, Chapter 04](../phase-01-foundation/04-runtime-data-areas.md) — mỗi
> thread có JVM Stack **riêng**, PC Register **riêng**, nhưng dùng chung một Heap.

Suốt Phase 1 và Phase 2, mọi ví dụ code đều ngầm định chạy trên **một** thread duy nhất —
`main`. Từ Phase 3, giả định đó không còn nữa. Thử đoạn code sau, một lỗi cực kỳ phổ biến với
người mới:

```java
Thread t = new Thread(() -> System.out.println("Chay tren: " + Thread.currentThread().getName()));
t.run();  // ⚠️ gọi run() TRỰC TIẾP, không phải start()
```

Kết quả: `Chay tren: main` — **không** có thread mới nào được tạo ra cả! Gọi `run()` trực tiếp
chỉ là một lời gọi method bình thường, chạy ngay trên thread hiện tại, y hệt gọi bất kỳ method
nào khác. Phải gọi `start()` mới thực sự tạo một luồng thực thi mới. Sự khác biệt tưởng nhỏ này
là điểm khởi đầu bắt buộc phải nắm chắc trước khi học bất kỳ khái niệm concurrency nào khác.

## Interview Question (Central)

> Điều gì thực sự xảy ra khi bạn gọi `thread.start()`? Vì sao gọi trực tiếp `thread.run()`
> không tạo ra luồng thực thi mới?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Tạo Thread bằng hai cách: kế thừa `Thread` và implement `Runnable`, biết khi nào dùng
      cách nào
- [ ] Tự tay chứng minh sự khác biệt giữa `start()` (tạo thread mới) và `run()` (chạy trên
      thread hiện tại)
- [ ] Biết mỗi `Thread` có JVM Stack riêng (liên hệ Phase 1, Chapter 04) — đây là lý do các
      thread độc lập với nhau về biến cục bộ
- [ ] Dùng đúng `join()` để chờ một thread hoàn thành

## Prerequisites

- Phase 1, Chapter 04 — biết mỗi thread có Stack riêng, dùng chung Heap.
- Phase 1, Chapter 05 — biết Execution Engine thực thi bytecode; mỗi thread có luồng thực thi
  bytecode độc lập.

## Used Later

Toàn bộ 18 chapter còn lại của Phase 3 xây dựng trực tiếp trên khái niệm `Thread` học ở đây —
từ vòng đời (Chapter 02), tới đồng bộ hoá (Chapter 05-11), tới các mô hình quản lý thread hiện
đại (Thread Pool, Chapter 15; Virtual Thread, Chapter 19).

## Problem

Một chương trình chỉ chạy trên một thread duy nhất chỉ có thể làm **một việc tại một thời
điểm** — nếu việc đó chậm (đọc file, gọi API, tính toán nặng), toàn bộ chương trình phải chờ,
dù CPU hiện đại có nhiều lõi có thể xử lý song song. Cần một cơ chế để chạy nhiều đoạn code
**đồng thời**, tận dụng phần cứng đa nhân.

## Concept

`Thread` là đơn vị thực thi độc lập nhỏ nhất trong JVM. Một chương trình Java luôn có ít nhất
một thread (`main`), và có thể tạo thêm các thread khác — mỗi thread chạy code **song song**
(hoặc xen kẽ, tuỳ số lõi CPU thực tế) với các thread khác, có JVM Stack riêng (Phase 1, Chapter
04) nhưng dùng chung Heap — đây chính là nguồn gốc của mọi vấn đề concurrency sẽ học trong suốt
Phase 3: dữ liệu trên Heap có thể bị nhiều thread cùng truy cập.

## Why?

Nếu không có Thread: một web server chỉ xử lý được **một** request tại một thời điểm — request
thứ hai phải chờ request đầu xử lý xong hoàn toàn, dù CPU có 16 lõi đang rảnh rỗi. Thread cho
phép JVM giao cho hệ điều hành phân bổ nhiều luồng thực thi lên nhiều lõi CPU thực tế, xử lý
nhiều việc thực sự song song.

## How?

Hai cách tạo `Thread`:

```java
// Cách 1: kế thừa Thread, override run()
class MyThread extends Thread {
    public void run() { System.out.println("Chay trong MyThread"); }
}
new MyThread().start();

// Cách 2: implement Runnable, truyền vào Thread (KHUYẾN NGHỊ — xem Engineering Insight)
Runnable task = () -> System.out.println("Chay trong Runnable");
new Thread(task).start();
```

`start()` yêu cầu JVM (thông qua hệ điều hành) tạo một **luồng thực thi mới thật sự** — cấp
Stack riêng, đăng ký với bộ lập lịch (scheduler) của hệ điều hành — rồi luồng mới đó mới gọi
`run()`. Gọi `run()` trực tiếp bỏ qua toàn bộ bước này, chỉ là một lời gọi method bình thường.

## Visualization

```
thread.start()
      │
      ▼
JVM yêu cầu HỆ ĐIỀU HÀNH tạo một OS thread mới
      │
      ▼
OS thread mới được cấp Stack riêng (Phase 1, Chapter 04)
      │
      ▼
Bộ lập lịch OS đăng ký thread mới, sẽ được CPU thực thi khi tới lượt
      │
      ▼
Thread MỚI (không phải thread gọi start()) thực thi run()


thread.run()  (gọi trực tiếp)
      │
      ▼
CHỈ LÀ MỘT LỜI GỌI METHOD BÌNH THƯỜNG
      │
      ▼
Chạy NGAY trên thread hiện tại — KHÔNG có thread mới nào được tạo
```

## Example

Chứng minh trực tiếp sự khác biệt `start()` vs `run()`:

```java
public class ThreadDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("Main thread: " + Thread.currentThread().getName());

        Thread t1 = new Thread(() -> {
            System.out.println("t1 chay tren thread: " + Thread.currentThread().getName());
        });
        t1.start();
        t1.join();

        System.out.println("--- goi run() truc tiep (KHONG start()) ---");
        Thread t2 = new Thread(() -> {
            System.out.println("t2 chay tren thread: " + Thread.currentThread().getName());
        });
        t2.run();
    }
}
```

Kết quả thật (JDK 17):

```
Main thread: main
t1 chay tren thread: Thread-0
--- goi run() truc tiep (KHONG start()) ---
t2 chay tren thread: main
```

`t1` (gọi `start()`) chạy trên một thread thật sự mới (`Thread-0`). `t2` (gọi `run()` trực
tiếp) chạy ngay trên `main` — xác nhận chính xác điều đã dự đoán ở Story.

## Deep Dive

**`join()` làm gì chính xác?** `t1.join()` khiến thread gọi nó (ở đây là `main`) **tạm dừng**
cho tới khi `t1` kết thúc hoàn toàn (chuyển sang trạng thái `TERMINATED`, xem
[Chapter 02](02-thread-lifecycle.md)). Nếu không gọi `join()`, `main` có thể kết thúc **trước**
`t1` kịp in ra bất cứ gì — chương trình Java thoát khi **mọi** thread non-daemon (mặc định) đã
kết thúc, không phải chỉ riêng `main`, nhưng thứ tự output giữa các thread không được đảm bảo
nếu không có `join()` hoặc cơ chế đồng bộ khác.

## Engineering Insight

**Vì sao nên implement `Runnable` thay vì kế thừa `Thread` (như Cách 2 ở phần How?)?** Vì Java
không hỗ trợ đa kế thừa class (Phase 1, Chapter 08) — nếu class của bạn đã kế thừa `Thread`, nó
**không thể** kế thừa thêm bất kỳ class nào khác. `Runnable` chỉ là một interface mô tả "một
việc cần làm" (`run()`), tách biệt hoàn toàn khỏi khái niệm "là một Thread" — bạn có thể để một
class vừa kế thừa một class nghiệp vụ khác, vừa implement `Runnable`, rồi truyền nó cho
`Thread` hoặc (thực tế hơn nhiều — xem Chapter 15-16) cho một `ExecutorService`. Đây cũng chính
là ứng dụng của nguyên tắc composition-over-inheritance đã học ở
[Phase 1, Chapter 08](../phase-01-foundation/08-oop-fundamentals.md): `Thread` **có một**
`Runnable` để chạy, thay vì class của bạn **là một** `Thread`.

## Historical Note

`Thread` tồn tại từ Java 1.0 (1996) — là một trong những API lâu đời nhất của Java, phản ánh
Java được thiết kế "sẵn sàng cho đa luồng" ngay từ đầu (khác nhiều ngôn ngữ cùng thời). Qua gần
30 năm, API `Thread` cơ bản gần như không đổi — điều thay đổi là các công cụ **quản lý** thread
xây trên nền nó: `ExecutorService` (Java 5, Chapter 15-16), `CompletableFuture` (Java 8,
Chapter 18), và gần đây nhất, Virtual Thread (Java 21, Chapter 19) — thay đổi triệt để nhất
trong toàn bộ lịch sử threading của Java.

## Myth vs Reality

- **Myth:** "Gọi `run()` hay `start()` đều tạo ra một thread mới, chỉ khác cách viết."
  **Reality:** Xem Example — chỉ `start()` tạo thread mới. `run()` là lời gọi method bình
  thường.

- **Myth:** "Không gọi `join()` thì chương trình sẽ chờ tất cả thread daemon lẫn non-daemon
  trước khi thoát."
  **Reality:** JVM chỉ chờ các thread **non-daemon** (mặc định của thread tự tạo). Thread
  daemon (đặt bằng `setDaemon(true)`) bị JVM **ngắt ngang** khi mọi thread non-daemon đã xong,
  không đợi chúng hoàn thành.

## Common Mistakes

- **Gọi `run()` thay vì `start()`** — lỗi cú pháp hợp lệ (không báo lỗi biên dịch, không
  exception) nhưng sai hoàn toàn về ý định, rất khó phát hiện nếu không biết rõ sự khác biệt.
- **Kế thừa `Thread` một cách không cần thiết** khi chỉ cần mô tả "một việc cần làm" — nên dùng
  `Runnable` (xem Engineering Insight).
- **Quên `join()`** rồi ngạc nhiên vì `main` kết thúc trước khi thread con kịp in kết quả.

## Best Practices

- Ưu tiên `Runnable` (hoặc `Callable`, Chapter 14) hơn kế thừa `Thread` trực tiếp.
- Trong thực tế production, hiếm khi tự tạo `Thread` trực tiếp — hầu hết dùng
  `ExecutorService` (Chapter 15-16) để quản lý vòng đời thread một cách có kiểm soát.
- Luôn `join()` (hoặc dùng cơ chế đồng bộ tương đương) nếu cần đảm bảo một thread đã hoàn
  thành trước khi tiếp tục.

## Production Notes

Chapter này thuần nền tảng — các vấn đề production cụ thể liên quan tới Thread (race condition,
deadlock, resource exhaustion khi tạo quá nhiều thread) sẽ được bàn chi tiết ở các chapter tiếp
theo khi có đủ ngữ cảnh kỹ thuật (Chapter 03 trở đi cho race condition, Chapter 15 cho vấn đề
tạo quá nhiều thread).

## Debug Checklist

- [ ] Code trong `Runnable`/override `run()` không chạy song song như mong đợi? → kiểm tra có
      đang gọi `run()` trực tiếp thay vì `start()` không.
- [ ] `main` kết thúc trước khi thread con in xong kết quả? → thiếu `join()`.

## Source Code Walkthrough

`Thread.start()` (native method, hiện thực bằng C++ trong HotSpot) gọi xuống hệ điều hành để
tạo một OS thread thật sự (`pthread_create` trên Linux/macOS, tương đương trên Windows) —
JVM Thread và OS thread có quan hệ 1-1 trong mô hình threading truyền thống của Java (khác
Virtual Thread, Chapter 19, nơi quan hệ này không còn 1-1). Đây là lý do tạo quá nhiều
`Thread` truyền thống tốn kém — mỗi cái đều kéo theo chi phí thật của hệ điều hành, không chỉ
là một object Java nhẹ nhàng trên Heap.

## Summary

`Thread` là đơn vị thực thi độc lập nhỏ nhất trong JVM, có JVM Stack riêng (Phase 1, Chapter
04) nhưng dùng chung Heap với các thread khác — nguồn gốc của mọi vấn đề concurrency. `start()`
tạo một luồng thực thi mới thật sự (thông qua OS thread); gọi `run()` trực tiếp chỉ là một lời
gọi method bình thường trên thread hiện tại — đã chứng minh bằng thực nghiệm. Ưu tiên
`Runnable` hơn kế thừa `Thread` trực tiếp, vì Java không hỗ trợ đa kế thừa. `join()` chờ một
thread hoàn thành trước khi tiếp tục.

## Interview Questions

**Junior**

- Có mấy cách tạo Thread trong Java? Kể tên.
- `start()` và `run()` khác nhau như thế nào?

**Mid**

- Vì sao nên implement `Runnable` thay vì kế thừa `Thread` trực tiếp?
- `join()` làm gì? Điều gì xảy ra nếu không gọi nó?

**Senior**

- Giải thích ở tầng hệ điều hành điều gì xảy ra khi gọi `thread.start()`.
- So sánh mô hình 1-1 giữa JVM Thread và OS Thread (truyền thống) với mô hình Virtual Thread
  (Chapter 19, xem trước nếu đã biết) — vì sao sự khác biệt này quan trọng cho khả năng mở
  rộng?

## Exercises

- [ ] Chạy lại đúng ví dụ `ThreadDemo` ở trên, xác nhận kết quả giống hệt.
- [ ] Tạo 5 Thread, mỗi thread in ra tên của chính nó — chạy nhiều lần, quan sát thứ tự output
      có luôn giống nhau không (gợi ý: không, vì bộ lập lịch OS không đảm bảo thứ tự).
- [ ] Viết một Thread không gọi `join()`, quan sát chương trình có thể kết thúc trước khi
      thread đó kịp in kết quả hay không.

## Cheat Sheet

| | `start()` | `run()` |
| --- | --- | --- |
| Tạo thread mới? | Có | Không |
| Chạy trên thread nào? | Thread mới | Thread hiện tại (gọi nó) |

```java
// Cách khuyến nghị
Runnable task = () -> { /* việc cần làm */ };
Thread t = new Thread(task);
t.start();
t.join(); // chờ hoàn thành
```

## References

- Java SE API Documentation — `java.lang.Thread`, `java.lang.Runnable`.
- Java Concurrency in Practice (Brian Goetz) — Chapter 1: Introduction.
