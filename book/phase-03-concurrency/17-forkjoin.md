---
tags:
  - Java
  - ForkJoin
  - Concurrency
---

# Fork/Join Framework

> Phase: Phase 3 — Concurrency
> Chapter slug: `forkjoin`

## Metadata

```yaml
Chapter: ForkJoin
Phase: Phase 3 — Concurrency
Difficulty: ★★★★
Importance: ★★★
Interview Frequency: 40%
Prerequisites:
  - Chapter 16 — Executor
Used Later:
  - Stream API song song (parallelStream, Phase 4) — dùng chung ForkJoinPool.commonPool() bên dưới
Estimated Reading: 18 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: [Chapter 16](16-executor.md) — `ExecutorService` xử lý tốt các tác vụ **độc lập,
> phẳng** (mỗi tác vụ không phụ thuộc kết quả của tác vụ khác). Nhưng với bài toán **chia để
> trị** (chia nhỏ đệ quy, ví dụ tính tổng mảng 10 triệu phần tử bằng cách chia đôi liên tục),
> cần một cơ chế khác: mỗi tác vụ con lại có thể tự chia nhỏ tiếp, và kết quả cha phải **chờ
> gộp** kết quả từ các con.

Thử nghiệm thực tế với hai loại khối lượng công việc khác nhau trên cùng một `ForkJoinPool`:

```
Tuan tu (phep cong don gian, memory-bound): 26ms
ForkJoin (10 CPU, phep cong don gian): 79ms   <- CHAM HON tuan tu!

Tuan tu (CPU-nang moi phan tu): 91ms
ForkJoin (10 CPU, CPU-nang moi phan tu): 25ms  <- NHANH HON tuan tu 3.6 lan
```

Cùng dùng Fork/Join, nhưng với công việc **quá đơn giản** (chỉ cộng số nguyên), chạy song song
lại **chậm hơn** chạy tuần tự — vì chi phí chia nhỏ tác vụ và điều phối giữa các thread lớn hơn
cả công việc thực sự. Chỉ khi công việc mỗi phần tử đủ **nặng về CPU**, Fork/Join mới thực sự
mang lại lợi ích rõ rệt.

## Interview Question (Central)

> Fork/Join Framework hoạt động như thế nào? Vì sao nó phù hợp cho bài toán chia để trị hơn là
> `ExecutorService` thông thường?

## Objectives

- [ ] Giải thích mô hình `RecursiveTask`/`RecursiveAction`, `fork()`/`join()`
- [ ] Tự tay đo được: Fork/Join có thể **chậm hơn** tuần tự với công việc quá đơn giản, nhưng
      **nhanh hơn** đáng kể (3.6 lần trên 10 CPU) với công việc thực sự nặng CPU
- [ ] Hiểu khái niệm work-stealing — lý do Fork/Join hiệu quả hơn thread pool thường cho bài
      toán đệ quy chia nhỏ

## Prerequisites

- Chapter 16 — hiểu `ExecutorService`, vì `ForkJoinPool` cũng implement interface này.

## Used Later

- **Stream API song song** (`parallelStream()`, Phase 4) — dùng chung
  `ForkJoinPool.commonPool()` bên dưới để chia nhỏ và xử lý song song các phần tử của stream.

## Problem

Với bài toán chia để trị (chia đệ quy thành các bài toán con nhỏ hơn, rồi gộp kết quả),
`ExecutorService` thông thường không tối ưu: nếu một task cha nộp các task con vào cùng hàng đợi
rồi **chờ** (`Future.get()`) kết quả, task cha chiếm một thread nhưng lại ở trạng thái chờ
không làm gì — lãng phí. Ngoài ra, một số thread trong pool có thể "hết việc" trong khi thread
khác vẫn còn rất nhiều task con đang chờ xử lý, gây mất cân bằng tải.

## Concept

**Fork/Join Framework** (`ForkJoinPool`, `RecursiveTask<V>`/`RecursiveAction`) là mô hình
chuyên biệt cho bài toán chia để trị: một tác vụ tự quyết định **chia nhỏ** (`fork()`) thành các
tác vụ con chạy song song, sau đó **gộp** (`join()`) kết quả các con lại. Bên dưới,
`ForkJoinPool` dùng kỹ thuật **work-stealing**: mỗi thread có hàng đợi task riêng, khi một
thread hết việc, nó "trộm" (steal) task từ hàng đợi của thread khác đang bận — giữ mọi thread
luôn bận rộn, cân bằng tải tự động.

## Why?

Work-stealing giải quyết đúng vấn đề mất cân bằng tải của mô hình hàng đợi chung: thay vì tất
cả thread tranh nhau một hàng đợi duy nhất (điểm nghẽn tiềm tàng), mỗi thread có hàng đợi riêng
(giảm tranh chấp) — chỉ khi thực sự hết việc mới cần "mượn" từ thread khác. Việc task cha
`fork()` con rồi tự `compute()` một phần thay vì chỉ chờ (thay vì gọi `get()` như
`ExecutorService` thông thường) giúp task cha **cũng làm việc** thay vì ngồi chờ không — tận
dụng tối đa số thread có sẵn.

## How?

```java
class SumTask extends RecursiveTask<Long> {
    protected Long compute() {
        if (kichThuocDuNho) {
            return tinhTrucTiep();
        }
        SumTask left = new SumTask(...);
        SumTask right = new SumTask(...);
        left.fork();               // giao cho thread khac chay bat dong bo
        long r = right.compute();  // tu minh tinh phan con lai (khong ngoi cho khong)
        long l = left.join();      // cho ket qua cua left
        return l + r;
    }
}
ForkJoinPool pool = new ForkJoinPool();
long result = pool.invoke(new SumTask(...));
```

## Visualization

```
Chia doi de tri (vi du tong mang 8 phan tu):

                    SumTask(0,8)
                   /            \
             fork()              compute() [chinh thread nay lam]
                /                      \
        SumTask(0,4)              SumTask(4,8)
       /         \                /         \
  SumTask(0,2) SumTask(2,4)  SumTask(4,6) SumTask(6,8)
      │             │            │            │
   [tinh]        [tinh]       [tinh]       [tinh]
      └────join()──┘             └────join()──┘
            └──────────gop ket qua──────────┘

Work-stealing: Thread A het viec trong hang doi rieng cua no
                    → "trom" task tu hang doi cua Thread B dang ban
                    → khong co thread nao ranh trong khi thread khac qua tai
```

## Example

**Trường hợp 1 — công việc đơn giản (phép cộng số nguyên), memory-bound:**

```java
int n = 100_000_000;
int[] arr = new int[n]; // moi phan tu = 1

// Tuan tu
long seqSum = 0;
for (int i = 0; i < n; i++) seqSum += arr[i];

// ForkJoin (SumTask nhu tren, threshold=1000)
ForkJoinPool pool = new ForkJoinPool();
long parSum = pool.invoke(new SumTask(arr, 0, n));
```

Kết quả thật (JDK 17):

```
Tuan tu: 100000000, 26ms
ForkJoin (10 CPU): 100000000, 79ms
```

**Trường hợp 2 — công việc nặng CPU mỗi phần tử:**

```java
static long heavyWork(int x) {
    long sum = 0;
    for (int i = 0; i < 200; i++) sum += (x * i) % 97;
    return sum;
}
// SumTask.compute() goi heavyWork(arr[i]) thay vi cong truc tiep
```

Kết quả thật (JDK 17):

```
Tuan tu (CPU-nang moi phan tu): 9455991192, 91ms
ForkJoin (10 CPU): 9455991192, 25ms
```

Đối lập rõ rệt: với phép cộng đơn giản (memory-bound, gần như không tốn CPU tính toán),
Fork/Join (79ms) **chậm hơn** tuần tự (26ms) — chi phí chia nhỏ task, điều phối giữa các thread,
và work-stealing lớn hơn chính công việc cần làm. Với công việc nặng CPU thực sự
(`heavyWork` — vòng lặp 200 bước mỗi phần tử), Fork/Join (25ms) **nhanh hơn** tuần tự (91ms)
khoảng 3.6 lần trên máy 10 CPU — lợi ích song song hoá vượt xa chi phí điều phối.

## Deep Dive

**Vì sao ngưỡng (threshold) chia nhỏ (`THRESHOLD = 1000` hoặc `5000` trong ví dụ) quan trọng?**
Nếu chia quá nhỏ (threshold quá thấp), số lượng task con tạo ra quá nhiều, chi phí tạo object
`SumTask` + điều phối `fork()`/`join()` cho từng task con nhỏ vượt xa lợi ích song song — đúng
hiện tượng đã thấy ở Trường hợp 1 khi công việc mỗi phần tử quá nhẹ. Nếu chia quá to (threshold
quá cao), số task con quá ít, không đủ để phân phối đều cho tất cả thread trong pool, một số
thread rảnh trong khi thread khác vẫn đang tính toán — không tận dụng hết khả năng song song.
Việc chọn threshold hợp lý (đủ nhỏ để chia đều cho các thread, đủ lớn để chi phí điều phối
không đáng kể so với công việc thực) là một quyết định cân bằng phải dựa trên đo lường thực tế.

## Engineering Insight

**Vì sao Fork/Join dùng riêng một pool (`ForkJoinPool`) thay vì tái sử dụng
`ThreadPoolExecutor` thông thường?** `ThreadPoolExecutor` dùng một hàng đợi tác vụ **dùng
chung** cho tất cả thread — phù hợp cho tác vụ độc lập, phẳng nhưng không tối ưu cho đệ quy chia
nhỏ (một thread `fork()` ra task con rồi `join()` chờ có thể bị **block** trong khi thread khác
rảnh, gây lãng phí, thậm chí có nguy cơ deadlock nếu tất cả thread đều đang chờ join task con
mà không còn thread nào rảnh để thực thi các task con đó). `ForkJoinPool` giải quyết bằng cơ chế
đặc biệt gọi là **join-helping**: khi một thread gọi `join()` chờ một task con chưa xong, thay
vì block hoàn toàn, nó có thể tự "giúp" thực thi các task khác đang chờ trong hàng đợi (kể cả
task của chính nó chưa được thread khác nhận) — giữ CPU luôn bận rộn thay vì ngồi chờ không.

## Historical Note

Fork/Join Framework được thêm vào Java 7 (2011, JSR 166y, cũng do Doug Lea dẫn dắt), muộn hơn
đáng kể so với phần còn lại của `java.util.concurrent` (Java 5, 2004) — vì mô hình work-stealing
đòi hỏi thiết kế và kiểm định phức tạp hơn nhiều so với thread pool thông thường. Java 8 (2014)
tận dụng lại chính `ForkJoinPool.commonPool()` này làm nền tảng cho `Stream.parallelStream()`.

## Myth vs Reality

- **Myth:** "Fork/Join (hoặc song song hoá nói chung) luôn nhanh hơn xử lý tuần tự."
  **Reality:** Đã chứng minh trực tiếp ở Example — với công việc quá đơn giản (memory-bound),
  Fork/Join chậm hơn tuần tự do chi phí điều phối vượt quá lợi ích song song.

- **Myth:** "`parallelStream()` luôn là lựa chọn tốt để tăng tốc xử lý collection."
  **Reality:** Vì `parallelStream()` dùng chung `ForkJoinPool.commonPool()` với cùng nguyên lý
  Fork/Join, nó chịu đúng giới hạn đã nêu — chỉ hiệu quả khi công việc mỗi phần tử đủ nặng CPU
  và tập dữ liệu đủ lớn.

## Common Mistakes

- **Dùng Fork/Join (hoặc `parallelStream`) cho công việc đơn giản mà không đo lường trước** —
  có thể làm chậm hơn thay vì nhanh hơn, đúng như Example đã chứng minh.
- **Chọn threshold quá nhỏ hoặc quá lớn mà không thử nghiệm** — ảnh hưởng trực tiếp tới hiệu
  năng, không có một con số "đúng" chung cho mọi bài toán.
- **Gọi blocking I/O bên trong `compute()`** — làm nghẽn thread trong `ForkJoinPool.commonPool()` (một pool dùng chung cho toàn JVM, kể cả `parallelStream`), có thể ảnh hưởng tới các
  phần khác của ứng dụng đang dùng chung pool này.

## Best Practices

- Chỉ dùng Fork/Join khi công việc mỗi phần tử đủ nặng CPU và có thể chia nhỏ đệ quy tự nhiên —
  đo lường thực tế trước khi quyết định, không giả định "song song luôn nhanh hơn".
- Tránh gọi blocking I/O bên trong task Fork/Join, vì `commonPool()` dùng chung có số thread
  giới hạn (mặc định bằng số lõi CPU trừ 1).
- Với công việc cần blocking I/O trong Fork/Join, cân nhắc `ForkJoinPool.ManagedBlocker` (kỹ
  thuật nâng cao, nằm ngoài phạm vi chapter nền tảng này) để tránh làm cạn kiệt pool dùng chung.

## Debug Checklist

- [ ] Fork/Join hoặc `parallelStream()` chậm hơn phiên bản tuần tự? → đo lại xem công việc mỗi
      phần tử có đủ nặng CPU không, cân nhắc threshold chia nhỏ.
- [ ] Nhiều phần không liên quan của ứng dụng cùng chậm lại khi có một tác vụ Fork/Join nặng? →
      kiểm tra có đang dùng chung `ForkJoinPool.commonPool()` và có blocking I/O bên trong task
      không.

## Source Code Walkthrough

`ForkJoinTask.fork()` (rút gọn từ OpenJDK) đẩy task hiện tại vào hàng đợi **riêng của thread
gọi** (không phải hàng đợi dùng chung):

```java
public final ForkJoinTask<V> fork() {
    Thread t;
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
        ((ForkJoinWorkerThread) t).workQueue.push(this);
    else
        ForkJoinPool.common.externalPush(this);
    return this;
}
```

Việc mỗi thread có `workQueue` riêng (thay vì một hàng đợi dùng chung cho cả pool) chính là nền
tảng kỹ thuật cho phép work-stealing hoạt động — thread khác chỉ "trộm" task khi hàng đợi riêng
của chính nó đã cạn.

## Summary

Fork/Join Framework chuyên biệt cho bài toán chia để trị: `fork()` giao task con cho thread
khác, `join()` chờ và gộp kết quả, cơ chế work-stealing giữ mọi thread luôn bận rộn. Đã chứng
minh bằng thực nghiệm hai chiều: với công việc đơn giản, Fork/Join **chậm hơn** tuần tự (79ms
vs 26ms) do chi phí điều phối; với công việc nặng CPU thực sự, Fork/Join **nhanh hơn** đáng kể
(25ms vs 91ms, ~3.6 lần trên 10 CPU). `parallelStream()` (Phase 4) dùng chung cơ chế và giới hạn
này qua `ForkJoinPool.commonPool()`.

## Interview Questions

**Mid**

- Fork/Join Framework hoạt động như thế nào? Vì sao nó phù hợp cho bài toán chia để trị hơn là
  `ExecutorService` thông thường?
- Work-stealing là gì? Nó giải quyết vấn đề gì?

**Senior**

- Giải thích vì sao Fork/Join có thể chậm hơn xử lý tuần tự trong một số trường hợp. Cho ví dụ
  cụ thể.
- `parallelStream()` dùng chung `ForkJoinPool.commonPool()` — điều này có rủi ro gì nếu code
  gọi blocking I/O bên trong stream song song?

## Exercises

- [ ] Chạy lại cả hai ví dụ (`ForkJoinCompare` và `ForkJoinCompare2`) ở trên, xác nhận Fork/Join
      chậm hơn với công việc đơn giản nhưng nhanh hơn với công việc nặng CPU.
- [ ] Thử đổi `THRESHOLD` trong `SumTask` ở Trường hợp 2 (ví dụ 500, 50000), đo lại thời gian,
      quan sát ảnh hưởng của ngưỡng chia nhỏ.
- [ ] Viết một `RecursiveAction` (không trả về giá trị) để sắp xếp song song một mảng lớn bằng
      merge sort đệ quy.

## Cheat Sheet

| | `RecursiveTask<V>` | `RecursiveAction` |
| --- | --- | --- |
| Trả về giá trị | Có (`V`) | Không (`void`) |
| Dùng khi | Cần gộp kết quả (tổng, max, ...) | Chỉ cần thực hiện hành động (sort tại chỗ, ...) |

**Điều kiện nên dùng Fork/Join:** bài toán chia để trị tự nhiên + công việc mỗi phần tử đủ nặng
CPU. Không đúng điều kiện này → xử lý tuần tự có thể nhanh hơn.

## References

- Java SE API Documentation — `java.util.concurrent.ForkJoinPool`, `RecursiveTask`,
  `RecursiveAction`.
- Doug Lea — "A Java Fork/Join Framework" (2000, nền tảng lý thuyết cho JSR 166y).
