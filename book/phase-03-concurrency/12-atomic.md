---
tags:
  - Java
  - Atomic
  - Concurrency
---

# Atomic (AtomicInteger, AtomicLong, AtomicReference)

> Phase: Phase 3 — Concurrency
> Chapter slug: `atomic`

## Metadata

```yaml
Chapter: Atomic
Phase: Phase 3 — Concurrency
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 60%
Prerequisites:
  - Chapter 04 — Volatile
  - Chapter 05 — Synchronized
Used Later:
  - Chapter 13 — CAS (cơ chế nền tảng bên trong Atomic)
  - Counter/metric trong ứng dụng production (Phase 8)
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 04](04-volatile.md) — `volatile` đảm bảo **visibility** (nhìn thấy giá trị
> mới nhất) nhưng KHÔNG đảm bảo **atomicity**. Bằng chứng thật đã đo được: `counter++` dưới 10
> thread dùng biến `volatile` cho ra `240104` thay vì `1000000` mong đợi — vì `++` thực chất là
> ba bước (đọc, cộng, ghi), và `volatile` không bảo vệ được chuỗi ba bước đó khỏi bị chen ngang.

Cách "truyền thống" để sửa lỗi này là `synchronized` (Chapter 05) — nhưng khoá độc quyền cho
một phép cộng đơn giản là hơi "nặng tay". Có cách nào **nguyên tử hoá đúng thao tác đọc-sửa-ghi
đó** mà không cần khoá độc quyền toàn phần không?

## Interview Question (Central)

> `AtomicInteger` giải quyết vấn đề gì mà `volatile` không giải quyết được? Vì sao
> `AtomicInteger` không cần `synchronized` mà vẫn đảm bảo đúng dưới nhiều thread?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích vì sao `counter++` không nguyên tử, kể cả khi biến là `volatile`
- [ ] Tự tay đo được `AtomicInteger.incrementAndGet()` cho kết quả đúng tuyệt đối dưới 10
      thread — khác hẳn kết quả sai của Chapter 04
- [ ] Biết `compareAndSet()` là nền tảng bên dưới mọi thao tác của lớp Atomic

## Prerequisites

- Chapter 04 — hiểu tại sao `volatile` không đủ cho `counter++`.
- Chapter 05 — hiểu `synchronized` là giải pháp "nặng" hơn cho cùng vấn đề.

## Used Later

- **Chapter 13 (CAS)** — cơ chế `compareAndSet` chính là nền tảng bên trong mọi lớp Atomic.
- **Counter/metric trong production** (Phase 8) — đếm request, đếm lỗi thường dùng
  `AtomicLong` thay vì `synchronized` vì hiệu năng cao hơn dưới tải lớn.

## Problem

`volatile` chỉ bảo đảm visibility, không bảo đảm atomicity cho thao tác đọc-sửa-ghi (đã chứng
minh ở Chapter 04: `240104` thay vì `1000000`). `synchronized` giải quyết đúng nhưng bắt buộc
độc quyền toàn phần — thread khác phải chờ, kể cả khi thao tác chỉ là một phép cộng cực nhanh.

## Concept

Lớp `Atomic*` (`AtomicInteger`, `AtomicLong`, `AtomicReference`, ...) trong gói
`java.util.concurrent.atomic` cung cấp các thao tác đọc-sửa-ghi **nguyên tử** (không thể bị chen
ngang) mà **không cần khoá** — dựa trên lệnh CPU cấp phần cứng gọi là **CAS**
(Compare-And-Swap, chi tiết ở Chapter 13).

## Why?

Khoá (`synchronized`/`ReentrantLock`) hoạt động bằng cách **chặn đứng** các thread khác — buộc
chúng chờ (blocking). Với thao tác cực đơn giản, cực nhanh như tăng một biến đếm, chi phí "đưa
thread vào trạng thái chờ, đánh thức lại sau" (context switch) có thể còn tốn kém hơn chính
bản thân thao tác. CAS cho phép nguyên tử hoá **mà không cần blocking** — thread không thành
công thì thử lại ngay lập tức (busy-retry, cực nhanh vì không cần hệ điều hành can thiệp) thay
vì bị đưa vào hàng chờ.

## How?

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();      // tuong duong ++counter, nhung NGUYEN TU
counter.get();                  // doc gia tri hien tai
counter.compareAndSet(10, 20);  // neu dang la 10, doi thanh 20; tra ve true/false
```

## Visualization

```
volatile int counter++ (KHONG nguyen tu — 3 buoc rieng biet):

  Thread A: doc(5) ────────────► cong(6) ──► ghi(6)
  Thread B:       doc(5) ──► cong(6) ──► ghi(6)   ← ca hai doc CUNG gia tri 5, MAT 1 lan tang!

AtomicInteger.incrementAndGet() (NGUYEN TU qua CAS):

  Thread A: CAS(5→6) THANH CONG
  Thread B: CAS(5→6) THAT BAI (gia tri da la 6) → doc lai(6) → CAS(6→7) THANH CONG
```

## Example

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.*;

public class AtomicDemo {
    public static void main(String[] args) throws InterruptedException {
        AtomicInteger counter = new AtomicInteger(0);
        int threads = 10, increments = 100_000;

        ExecutorService pool = Executors.newFixedThreadPool(threads);
        for (int t = 0; t < threads; t++) {
            pool.submit(() -> {
                for (int i = 0; i < increments; i++) counter.incrementAndGet();
            });
        }
        pool.shutdown();
        pool.awaitTermination(30, TimeUnit.SECONDS);
        System.out.println("AtomicInteger: " + counter.get() + " (mong doi " + (threads * increments) + ")");

        AtomicInteger cas = new AtomicInteger(10);
        boolean success1 = cas.compareAndSet(10, 20);
        boolean success2 = cas.compareAndSet(10, 30);
        System.out.println("CAS(10->20) thanh cong: " + success1 + ", gia tri = " + cas.get());
        System.out.println("CAS(10->30) thanh cong: " + success2 + ", gia tri = " + cas.get());
    }
}
```

Kết quả thật (JDK 17):

```
AtomicInteger: 1000000 (mong doi 1000000)
CAS(10->20) thanh cong: true, gia tri = 20
CAS(10->30) thanh cong: false, gia tri = 20
```

So với `volatile int` ở Chapter 04 (`240104`/`1000000` — SAI), `AtomicInteger` cho ra chính xác
`1000000/1000000` dưới cùng điều kiện 10 thread × 100.000 lần tăng — không hề mất phép tăng nào.

`compareAndSet(10, 20)` thành công vì giá trị hiện tại **đúng là 10** → đổi thành 20. Ngay sau
đó `compareAndSet(10, 30)` **thất bại** (trả về `false`) vì giá trị hiện tại đã là 20 (không
còn là 10 nữa) — giá trị giữ nguyên 20, không bị đổi thành 30. Đây chính là bản chất của CAS:
chỉ thành công khi giá trị **chưa bị ai khác thay đổi** kể từ lần đọc gần nhất.

## Deep Dive

**Vì sao CAS không cần blocking mà vẫn đúng?** Lệnh CAS ở cấp CPU (`cmpxchg` trên x86) thực
hiện "so sánh và đổi" trong **một chu kỳ CPU duy nhất, không thể bị ngắt giữa chừng** — phần
cứng đảm bảo tính nguyên tử, không phải phần mềm. Khi nhiều thread cùng CAS vào một biến, chỉ
đúng một thread thành công tại một thời điểm; các thread còn lại nhận `false` và (do
`incrementAndGet()` được cài đặt bằng vòng lặp retry bên trong) tự động đọc lại giá trị mới
nhất rồi thử CAS lại — lặp đến khi thành công. Không ai bị "chặn đứng" chờ đợi kiểu blocking,
chỉ đơn giản là thử lại rất nhanh.

## Engineering Insight

**Atomic có luôn nhanh hơn `synchronized` trong mọi trường hợp?** Không. Dưới tranh chấp
(contention) **rất cao** — rất nhiều thread cùng CAS vào **một** biến duy nhất liên tục — tỉ lệ
CAS thất bại tăng vọt, mỗi thread phải retry nhiều lần, tổng công việc CPU lãng phí vào các lần
retry thất bại có thể vượt qua lợi ích "không blocking". Đây là lý do JDK 8 giới thiệu thêm
`LongAdder` — chia bộ đếm thành nhiều "ô" riêng biệt, mỗi thread cập nhật một ô ít tranh chấp
hơn, rồi cộng dồn khi cần đọc tổng — hiệu quả hơn `AtomicLong` khi có rất nhiều thread cùng ghi
vào một bộ đếm.

## Historical Note

Gói `java.util.concurrent.atomic` ra đời cùng JSR 166 (Java 5, 2004, do Doug Lea dẫn dắt — cũng
chính là tác giả của AQS đã nhắc ở Chapter 09-10). `LongAdder`/`DoubleAdder` bổ sung ở Java 8
(2014) để giải quyết đúng hạn chế "tranh chấp cao" của `AtomicLong` nêu trên.

## Myth vs Reality

- **Myth:** "Atomic classes dùng khoá bên trong, chỉ là khoá 'nhẹ' hơn `synchronized`."
  **Reality:** Không dùng khoá (lock) nào cả — dựa hoàn toàn vào lệnh CAS cấp phần cứng, đây
  gọi là kỹ thuật **lock-free** (không khoá), khác bản chất với khoá "nhẹ".

- **Myth:** "`AtomicInteger` luôn nhanh hơn `synchronized` cho mọi bộ đếm."
  **Reality:** Xem Engineering Insight — dưới tranh chấp cực cao, `LongAdder` còn nhanh hơn cả
  `AtomicLong`.

## Common Mistakes

- **Dùng `get()` rồi `set()` riêng biệt tưởng là nguyên tử** — `int x = counter.get(); counter.set(x + 1);` KHÔNG nguyên tử (y hệt lỗi `counter++`), phải dùng `incrementAndGet()` hoặc
  `updateAndGet()`.
- **Dùng `AtomicInteger` cho field không thực sự cần chia sẻ giữa nhiều thread** — nếu chỉ một
  thread truy cập, `int` thường là đủ, không cần overhead của Atomic.
- **Quên rằng `compareAndSet` có thể thất bại "hợp lệ"** — code gọi `compareAndSet` phải luôn
  kiểm tra giá trị trả về hoặc dùng trong vòng lặp retry, không được giả định luôn thành công.

## Best Practices

- Dùng `Atomic*` cho counter/flag đơn giản chia sẻ giữa nhiều thread thay vì `synchronized` khi
  thao tác đủ đơn giản để biểu diễn bằng CAS.
- Với bộ đếm bị tranh chấp cực cao (nhiều thread ghi liên tục), cân nhắc `LongAdder` thay vì
  `AtomicLong`.
- Dùng `updateAndGet(Function)`/`accumulateAndGet(...)` cho các phép cập nhật phức tạp hơn phép
  cộng đơn giản, vẫn giữ tính nguyên tử.

## Production Notes

**Vấn đề:** một service đếm số request xử lý mỗi giây bằng `AtomicLong` để xuất metric, dưới
tải cao (hàng chục nghìn request/giây, hàng trăm thread cùng ghi) CPU usage tăng bất thường dù
logic xử lý request không đổi.

- **Triệu chứng:** CPU cao hơn dự kiến khi tải tăng, đặc biệt phần liên quan tới cập nhật
  counter.
- **Root cause:** rất nhiều thread cùng CAS vào **một** biến `AtomicLong` duy nhất — tranh chấp
  cực cao khiến tỉ lệ CAS thất bại/retry tăng vọt, lãng phí chu kỳ CPU (đúng giới hạn đã phân
  tích ở Engineering Insight).
- **Debug:** profiling cho thấy phần lớn thời gian CPU nằm trong vòng lặp retry của
  `incrementAndGet()`.
- **Solution:** đổi sang `LongAdder` — giảm tranh chấp bằng cách chia bộ đếm thành nhiều ô nội
  bộ, mỗi thread ghi vào một ô khác nhau (giảm khả năng đụng độ), chỉ cộng dồn khi cần đọc
  `sum()`.
- **Prevention:** với counter chịu tải ghi rất cao từ nhiều thread, mặc định cân nhắc
  `LongAdder` ngay từ đầu thay vì `AtomicLong`.

## Debug Checklist

- [ ] Counter cho kết quả sai dưới nhiều thread? → kiểm tra có đang dùng `get()` + `set()` tách
      rời thay vì thao tác nguyên tử (`incrementAndGet()`) không.
- [ ] CPU cao bất thường liên quan tới cập nhật counter dưới tải lớn? → cân nhắc `LongAdder`
      thay vì `AtomicLong`/`AtomicInteger`.
- [ ] `compareAndSet` luôn thất bại? → xác nhận giá trị "expected" truyền vào có đúng là giá
      trị hiện tại tại thời điểm gọi không — giá trị có thể đã bị thread khác thay đổi.

## Source Code Walkthrough

`AtomicInteger.incrementAndGet()` (OpenJDK) về bản chất là một vòng lặp:

```java
// Rut gon tu OpenJDK
public final int incrementAndGet() {
    int prev, next;
    do {
        prev = get();
        next = prev + 1;
    } while (!compareAndSet(prev, next)); // that bai thi doc lai va thu lai
    return next;
}
```

Vòng lặp `do-while` với `compareAndSet` bên trong chính là mẫu hình "optimistic retry" chuẩn
của mọi thao tác lock-free trong gói `java.util.concurrent.atomic`.

## Summary

`Atomic*` cung cấp thao tác đọc-sửa-ghi nguyên tử **không cần khoá**, dựa trên lệnh CAS cấp
phần cứng — đã chứng minh bằng thực nghiệm: `AtomicInteger` cho kết quả đúng tuyệt đối
(`1000000/1000000`) dưới 10 thread, khác hẳn kết quả sai của `volatile int` (Chapter 04).
`compareAndSet` chỉ thành công khi giá trị chưa bị thay đổi kể từ lần đọc gần nhất — nền tảng
cho toàn bộ họ lớp Atomic. Dưới tranh chấp cực cao, `LongAdder` hiệu quả hơn `AtomicLong`.

## Interview Questions

**Junior**

- `AtomicInteger` khác `volatile int` như thế nào?

**Mid**

- Giải thích tại sao `compareAndSet` không cần khoá mà vẫn đảm bảo đúng dưới nhiều thread.
- Khi nào nên dùng `AtomicLong`, khi nào nên dùng `LongAdder`?

**Senior**

- Một counter dùng `AtomicLong` gây CPU cao dưới tải lớn. Giải thích nguyên nhân và cách khắc
  phục.

## Exercises

- [ ] Chạy lại `AtomicDemo` ở trên, xác nhận kết quả `1000000/1000000` trên máy bạn.
- [ ] Viết lại cùng bài toán dùng `synchronized`, so sánh thời gian chạy với `AtomicInteger`
      dưới cùng điều kiện (10 thread × 100.000 lần).
- [ ] Tìm hiểu `LongAdder`, viết benchmark so sánh với `AtomicLong` dưới 50 thread cùng ghi.

## Cheat Sheet

| | `volatile int` | `AtomicInteger` | `synchronized` |
| --- | --- | --- | --- |
| Visibility | Có | Có | Có |
| Atomicity cho `++` | Không | Có | Có |
| Cơ chế | Bộ nhớ | CAS (lock-free) | Khoá (blocking) |
| Tranh chấp cực cao | N/A | Suy giảm hiệu năng | Suy giảm hiệu năng |

## References

- Java SE API Documentation — `java.util.concurrent.atomic.AtomicInteger`, `LongAdder`.
- Java Concurrency in Practice (Brian Goetz) — Chapter 15: Atomic Variables and Nonblocking
  Synchronization.
