---
tags:
  - Java
  - VirtualThread
  - Concurrency
---

# Virtual Thread

> Phase: Phase 3 — Concurrency
> Chapter slug: `virtual-thread`

## Metadata

```yaml
Chapter: Virtual Thread
Phase: Phase 3 — Concurrency
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 55%
Prerequisites:
  - Chapter 01 — Thread
  - Chapter 15 — Thread Pool
  - Chapter 16 — Executor
Used Later:
  - Kiến trúc web server hiệu năng cao (Phase 5, Spring Boot 3+ hỗ trợ Virtual Thread)
Estimated Reading: 18 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: [Chapter 15](15-thread-pool.md) — thread hệ điều hành (platform thread) tốn bộ nhớ
> và thời gian tạo, buộc phải giới hạn số lượng qua thread pool, tái sử dụng thay vì tạo mới
> liên tục.

Nhưng giới hạn số lượng thread cũng đồng nghĩa giới hạn số tác vụ **chạy đồng thời được**. Với
mô hình "mỗi request một thread" phổ biến trong web server truyền thống, nếu mỗi request chờ
I/O (gọi API, query DB) mất 100ms, và pool chỉ có 200 thread, tối đa bao nhiêu request có thể xử
lý cùng lúc? Đo thử thực tế trên JDK 26 (Temurin):

```
10000 task tren PLATFORM thread pool (200 thread): 10000 xong, 5200ms
10000 task tren VIRTUAL thread (1 moi task): 10000 xong, 152ms
```

Cùng 10.000 tác vụ, mỗi tác vụ chờ 100ms (mô phỏng I/O) — dùng 200 platform thread mất **5200ms**
(vì phải xử lý theo từng đợt 200), trong khi tạo **10.000 virtual thread riêng** (một cho mỗi
tác vụ) chỉ mất **152ms** — nhanh hơn khoảng 34 lần.

## Interview Question (Central)

> Virtual Thread là gì? Nó khác Platform Thread (thread hệ điều hành truyền thống) như thế nào,
> và vì sao nó giúp xử lý được số lượng tác vụ I/O-bound lớn hơn nhiều?

## Objectives

- [ ] Phân biệt Platform Thread (ánh xạ 1-1 với thread hệ điều hành) và Virtual Thread (do JVM
      quản lý, nhẹ hơn nhiều)
- [ ] Tự tay đo được: 10.000 tác vụ I/O-bound chạy nhanh hơn ~34 lần với virtual thread so với
      platform thread pool 200 thread
- [ ] Tự tay chạy được 100.000 virtual thread đồng thời — điều gần như bất khả thi với platform
      thread

## Prerequisites

- Chapter 01 — hiểu thread hệ điều hành truyền thống (platform thread).
- Chapter 15, 16 — hiểu thread pool và lý do phải giới hạn số lượng platform thread.

## Used Later

- **Kiến trúc web server hiệu năng cao** (Phase 5) — Spring Boot 3.2+ hỗ trợ chạy trên Virtual
  Thread, thay đổi đáng kể cách suy nghĩ về giới hạn concurrency của một service.

## Problem

Platform thread ánh xạ **1-1** với thread hệ điều hành — mỗi platform thread tốn bộ nhớ stack
(mặc định ~1MB) và việc tạo/chuyển ngữ cảnh (context switch) do hệ điều hành quản lý, có chi phí
đáng kể. Với ứng dụng I/O-bound (phần lớn thời gian mỗi request chờ mạng/disk, không dùng CPU),
mô hình "mỗi request một platform thread" nhanh chóng chạm giới hạn: hàng chục nghìn thread đồng
thời có thể tiêu tốn hàng chục GB bộ nhớ chỉ riêng cho stack, và hệ điều hành gặp khó khăn quản
lý lịch trình cho quá nhiều thread.

## Concept

**Virtual Thread** (JEP 444, chính thức từ Java 21) là thread do **JVM quản lý** (không phải hệ
điều hành), rất nhẹ (bộ nhớ ban đầu chỉ vài trăm byte thay vì ~1MB), và có thể tạo **hàng triệu**
virtual thread trong một JVM. Nhiều virtual thread chia sẻ một số lượng nhỏ **carrier thread**
(platform thread thực sự chạy trên CPU) — khi một virtual thread bị block (ví dụ chờ I/O), JVM
tự động "gỡ" nó khỏi carrier thread, giải phóng carrier thread đó để chạy virtual thread khác,
rồi gắn lại khi I/O hoàn thành.

## Why?

Platform thread bị giới hạn bởi tài nguyên hệ điều hành (bộ nhớ, số lượng thread scheduler có
thể quản lý hiệu quả). Virtual thread giải quyết đúng giới hạn này bằng cách tách "đơn vị công
việc logic" (virtual thread) khỏi "tài nguyên thực thi" (carrier thread) — với ứng dụng
I/O-bound (đa số ứng dụng backend: chờ DB, chờ API, chờ file), phần lớn thời gian virtual thread
ở trạng thái "chờ", không cần chiếm CPU thực sự. Cho phép hàng chục nghìn virtual thread cùng
tồn tại (chỉ vài trăm chờ CPU tại một thời điểm bất kỳ) khai thác đúng thực tế này, thay vì lãng
phí tài nguyên hệ điều hành cho các platform thread phần lớn thời gian không làm gì ngoài chờ.

## How?

```java
Thread vt = Thread.ofVirtual().start(() -> {
    // logic tac vu, viet y het nhu voi platform thread
});

// Hoac dung ExecutorService chuyen dung
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> { /* tac vu */ });
} // tu dong shutdown khi ra khoi try-with-resources (tu Java 19+)
```

## Visualization

```
Platform Thread (1-1 voi OS thread):

  Virtual "cong viec" ──► Platform Thread ──► OS Thread ──► CPU
  (gioi han boi so luong OS thread ho tro duoc, ~vai nghin la da nang)

Virtual Thread (M virtual thread : N carrier thread, M >> N):

  VT1 ──┐
  VT2 ──┤   chi nhung VT dang THUC SU chay (khong cho I/O)
  VT3 ──┼──► [Carrier Thread 1][Carrier Thread 2]...  ← so luong nho, = platform thread
  ...   │        │ VT bi block I/O → JVM "go" khoi carrier, carrier ranh chay VT khac
  VT100000┘       │ I/O xong → JVM "gan lai" VT vao mot carrier ranh
```

## Example

**100.000 virtual thread đồng thời:**

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class VirtualThreadDemo {
    public static void main(String[] args) throws Exception {
        int n = 100_000;
        AtomicInteger completed = new AtomicInteger(0);
        long start = System.currentTimeMillis();
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < n; i++) {
                executor.submit(() -> {
                    try { Thread.sleep(100); } catch (InterruptedException e) {}
                    completed.incrementAndGet();
                });
            }
        }
        long elapsed = System.currentTimeMillis() - start;
        System.out.println(n + " virtual thread (moi thread sleep 100ms) hoan thanh: " + completed.get() + ", mat " + elapsed + "ms");
    }
}
```

Kết quả thật (Temurin 26):

```
100000 virtual thread (moi thread sleep 100ms) hoan thanh: 100000, mat 478ms
```

100.000 virtual thread, mỗi cái `sleep(100ms)`, hoàn thành **tất cả** chỉ trong 478ms — nếu
chạy tuần tự sẽ mất 100.000 × 100ms = 10.000 giây (~2.7 giờ). Với platform thread, tạo 100.000
thread thật gần như chắc chắn gây lỗi tài nguyên hệ điều hành trên hầu hết máy thông thường.

**So sánh trực tiếp platform thread pool vs virtual thread (10.000 tác vụ):**

```java
try (ExecutorService platformExec = Executors.newFixedThreadPool(200)) {
    for (int i = 0; i < 10_000; i++)
        platformExec.submit(() -> { try { Thread.sleep(100); } catch (InterruptedException e) {} });
}
// vs
try (ExecutorService virtualExec = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10_000; i++)
        virtualExec.submit(() -> { try { Thread.sleep(100); } catch (InterruptedException e) {} });
}
```

Kết quả thật (Temurin 26):

```
10000 task tren PLATFORM thread pool (200 thread): 10000 xong, 5200ms
10000 task tren VIRTUAL thread (1 moi task): 10000 xong, 152ms
```

Với 200 platform thread, 10.000 tác vụ phải xử lý theo từng đợt ~200 (10000/200 = 50 đợt ×
100ms ≈ 5000ms, khớp với 5200ms đo được). Với virtual thread, toàn bộ 10.000 tác vụ được "gắn"
vào carrier thread và chạy gần như đồng thời — chỉ mất 152ms, đúng bằng khoảng thời gian
`sleep(100ms)` cộng thêm overhead khởi tạo, không bị nhân lên theo số đợt.

## Deep Dive

**Cơ chế "gỡ và gắn lại" (unmount/mount) hoạt động thế nào khi virtual thread gọi
`Thread.sleep()` hoặc I/O blocking?** Khi một virtual thread gọi một thao tác blocking mà JDK đã
được điều chỉnh để nhận biết (`Thread.sleep`, hầu hết API I/O chuẩn trong `java.io`/`java.net`
từ Java 21), JVM **không** để carrier thread bị block theo kiểu truyền thống (như platform
thread sẽ làm) — thay vào đó, trạng thái của virtual thread (continuation) được lưu lại, virtual
thread bị "gỡ" (unmount) khỏi carrier thread, và carrier thread đó được giải phóng để chạy
virtual thread khác đang chờ. Khi thao tác blocking hoàn thành (ví dụ hết thời gian sleep, hoặc
dữ liệu I/O đã sẵn sàng), virtual thread được "gắn lại" (mount) vào một carrier thread rảnh
(không nhất thiết là carrier ban đầu) để tiếp tục chạy.

## Engineering Insight

**Virtual Thread có giải quyết được bài toán CPU-bound không?** Không. Virtual Thread chỉ giải
quyết đúng vấn đề I/O-bound (nhiều tác vụ phần lớn thời gian chờ, không chiếm CPU) — với tác vụ
CPU-bound (tính toán nặng, không có điểm chờ I/O nào), số carrier thread thực tế vẫn giới hạn
bởi số lõi CPU vật lý, tạo thêm virtual thread cho công việc CPU-bound không mang lại lợi ích gì
(vẫn phải xếp hàng chờ CPU, y hệt platform thread). Đây cũng là lý do quan trọng: **không nên
thay thế `ForkJoinPool`/công việc CPU-bound** (Chapter 17) bằng virtual thread — chúng phục vụ
hai mục đích khác nhau hoàn toàn (I/O-bound scale-out vs CPU-bound song song hoá tính toán).

Một cạm bẫy khác: **`synchronized` block trong virtual thread**. Trước Java 24, nếu một virtual
thread bị block bên trong một `synchronized` block/method, nó **không** được "gỡ" khỏi carrier
thread (khác với `Thread.sleep`/I/O thông thường) — hiện tượng gọi là "pinning", có thể làm
giảm hẳn lợi ích scale-out nếu code cũ dùng nhiều `synchronized` (thay vì `ReentrantLock`,
Chapter 05, 10) trong đường dẫn xử lý chính.

## Historical Note

Virtual Thread được đề xuất từ Project Loom (khởi động khoảng 2017-2018), trải qua nhiều JEP dự
thảo (JEP 425 preview ở Java 19, JEP 436 ở Java 20) trước khi chính thức ổn định ở **Java 21**
(JEP 444, tháng 9/2023). Đây là một trong những thay đổi nền tảng lớn nhất của JVM kể từ khi
Generics/Lambda ra đời, trực tiếp giải quyết hạn chế "mỗi request một platform thread" đã tồn
tại từ Java 1.0.

## Myth vs Reality

- **Myth:** "Virtual Thread thay thế hoàn toàn Platform Thread, nên dùng cho mọi trường hợp."
  **Reality:** Xem Engineering Insight — Virtual Thread không mang lại lợi ích cho tác vụ
  CPU-bound, và có cạm bẫy "pinning" với `synchronized` (trước Java 24).

- **Myth:** "Virtual Thread là một dạng coroutine/green thread giống các ngôn ngữ khác, hoàn
  toàn không liên quan tới thread hệ điều hành."
  **Reality:** Virtual Thread vẫn cần **carrier thread** (platform thread thật) để thực sự chạy
  trên CPU — nó là lớp trừu tượng nhẹ hơn nằm trên platform thread, không phải hoàn toàn độc lập
  khỏi khái niệm thread hệ điều hành.

## Common Mistakes

- **Tạo thread pool cố định (`newFixedThreadPool`) chứa virtual thread** — mất hết ý nghĩa;
  virtual thread được thiết kế để tạo mới cho mỗi tác vụ (`newVirtualThreadPerTaskExecutor`),
  không cần tái sử dụng như platform thread (Chapter 15) vì bản thân chúng đã rất rẻ để tạo.
- **Dùng virtual thread cho tác vụ CPU-bound, mong đợi tăng tốc** — xem Engineering Insight,
  không mang lại lợi ích, nên dùng `ForkJoinPool` (Chapter 17) cho trường hợp này.
- **Dùng nhiều `synchronized` block dài trong code chạy trên virtual thread (trước Java 24)** —
  gây "pinning", giảm hẳn lợi ích scale-out; cân nhắc `ReentrantLock` thay thế.

## Best Practices

- Dùng `Executors.newVirtualThreadPerTaskExecutor()` cho khối lượng lớn tác vụ I/O-bound (gọi
  API, query DB, đọc file) — không cần giới hạn/tái sử dụng như platform thread pool.
- Vẫn dùng platform thread pool (`ThreadPoolExecutor`, Chapter 15) hoặc `ForkJoinPool` (Chapter
  17) cho tác vụ CPU-bound.
- Với code chạy trên Java 21-23, rà soát các đoạn `synchronized` trong đường dẫn xử lý chính
  nếu dự định chuyển sang virtual thread, cân nhắc đổi sang `ReentrantLock` để tránh pinning.

## Debug Checklist

- [ ] Chuyển sang virtual thread nhưng không thấy cải thiện hiệu năng? → xác nhận tác vụ có
      thực sự I/O-bound không — virtual thread không giúp gì cho tác vụ CPU-bound.
- [ ] Hiệu năng virtual thread thấp hơn mong đợi dưới tải cao (trước Java 24)? → kiểm tra code
      có dùng nhiều `synchronized` block gây pinning không.
- [ ] Cần tạo hàng chục nghìn tác vụ đồng thời chờ I/O? → cân nhắc virtual thread thay vì cố
      tăng kích thước platform thread pool.

## Summary

Virtual Thread (Java 21, JEP 444) là thread nhẹ do JVM quản lý, nhiều virtual thread chia sẻ một
số ít carrier thread (platform thread thật) — khi virtual thread block I/O, nó tự động được "gỡ"
khỏi carrier để carrier chạy việc khác, "gắn lại" khi I/O xong. Đã chứng minh bằng thực nghiệm:
10.000 tác vụ I/O-bound nhanh hơn ~34 lần với virtual thread (152ms) so với platform pool 200
thread (5200ms), và 100.000 virtual thread hoàn thành trong 478ms — điều gần như bất khả thi
với platform thread thật. Chỉ mang lại lợi ích cho tác vụ I/O-bound, không giúp gì cho CPU-bound,
và có cạm bẫy "pinning" với `synchronized` (trước Java 24).

## Interview Questions

**Mid**

- Virtual Thread là gì? Nó khác Platform Thread như thế nào?
- Vì sao Virtual Thread phù hợp cho ứng dụng I/O-bound nhưng không giúp ích cho ứng dụng
  CPU-bound?

**Senior**

- Giải thích cơ chế "carrier thread" và quá trình unmount/mount của Virtual Thread khi gặp thao
  tác blocking.
- Hiện tượng "pinning" với `synchronized` là gì? Nó ảnh hưởng thế nào tới lợi ích của Virtual
  Thread, và cách khắc phục?

## Exercises

- [ ] Chạy lại `VirtualThreadDemo` và `PlatformVsVirtual` ở trên (cần JDK 21+), xác nhận kết
      quả tương tự trên máy bạn.
- [ ] Thử viết một đoạn code có `synchronized` block dài chạy trên virtual thread, dùng
      `jcmd <pid> Thread.dump_to_file` để quan sát hiện tượng pinning (nếu chạy trên Java < 24).
- [ ] So sánh bộ nhớ sử dụng khi tạo 10.000 platform thread (`new Thread()`) so với 10.000
      virtual thread, dùng công cụ theo dõi bộ nhớ JVM (ví dụ `jconsole`).

## Cheat Sheet

| | Platform Thread | Virtual Thread |
| --- | --- | --- |
| Quản lý bởi | Hệ điều hành | JVM |
| Bộ nhớ ban đầu | ~1MB (stack) | Vài trăm byte |
| Số lượng thực tế tối đa | Vài nghìn | Hàng triệu |
| Phù hợp cho | CPU-bound | I/O-bound |
| Tạo qua | `new Thread()` | `Thread.ofVirtual()`, `newVirtualThreadPerTaskExecutor()` |
| Từ Java | 1.0 | 21 (JEP 444) |

## References

- JEP 444: Virtual Threads — https://openjdk.org/jeps/444
- Project Loom — https://wiki.openjdk.org/display/loom
- Oracle Java Documentation — Virtual Threads (Java 21+).
