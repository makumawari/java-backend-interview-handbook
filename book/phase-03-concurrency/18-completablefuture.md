---
tags:
  - Java
  - CompletableFuture
  - Concurrency
---

# CompletableFuture

> Phase: Phase 3 — Concurrency
> Chapter slug: `completablefuture`

## Metadata

```yaml
Chapter: CompletableFuture
Phase: Phase 3 — Concurrency
Difficulty: ★★★★
Importance: ★★★★★
Interview Frequency: 70%
Prerequisites:
  - Chapter 14 — Callable vs Runnable
  - Chapter 16 — Executor
Used Later:
  - Gọi API bất đồng bộ, tổng hợp nhiều nguồn dữ liệu song song (Phase 5, Phase 8)
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

## Story

> Xem lại: [Chapter 14](14-callable-runnable.md) — `Future.get()` là **blocking**: thread gọi
> nó phải ngồi chờ tới khi task xong, không thể "làm gì tiếp theo với kết quả" mà không chặn
> đứng thread hiện tại.

Với các luồng xử lý thực tế — gọi API A, lấy kết quả rồi gọi API B dùng kết quả đó, rồi kết hợp
với kết quả của một API C chạy song song — nếu chỉ dùng `Future.get()` thuần, code sẽ phải
lồng nhau hoặc chờ tuần tự, mất hết lợi ích bất đồng bộ. `CompletableFuture` cho phép **nối
chuỗi** các bước xử lý mà không cần block:

```
Buoc 1 (thread: ForkJoinPool.commonPool-worker-1)
Ket qua: 20
Ket hop 2 future: 12
Da bat loi: loi!
```

## Interview Question (Central)

> `CompletableFuture` khác `Future` như thế nào? Vì sao nó cho phép nối chuỗi (chaining) các
> thao tác bất đồng bộ mà `Future` thông thường không làm được?

## Objectives

- [ ] Phân biệt `Future` (chỉ chờ blocking) và `CompletableFuture` (nối chuỗi non-blocking)
- [ ] Tự tay chạy được `thenApply`, `thenCombine`, `exceptionally` với kết quả thật
- [ ] Hiểu cơ chế xử lý lỗi trong chuỗi bất đồng bộ qua `exceptionally`/`handle`

## Prerequisites

- Chapter 14 — hiểu `Future`, hạn chế blocking của `get()`.
- Chapter 16 — hiểu `ExecutorService`, vì `CompletableFuture` có thể nhận `Executor` tuỳ chỉnh.

## Used Later

- **Gọi API bất đồng bộ, tổng hợp nhiều nguồn dữ liệu song song** (Phase 5, Phase 8) —
  `CompletableFuture` là công cụ chuẩn để gọi nhiều service song song rồi gộp kết quả trong
  kiến trúc microservice.

## Problem

`Future.get()` (Chapter 14) chỉ có một cách để lấy kết quả: **chờ blocking**. Không có cách nào
để nói "khi task này xong, tự động chạy tiếp bước sau với kết quả đó" mà không phải tự viết
callback thủ công hoặc block một thread chỉ để chờ. Với nhiều bước phụ thuộc lẫn nhau hoặc cần
chạy song song rồi gộp kết quả, `Future` thuần trở nên cồng kềnh và kém hiệu quả (mỗi lần chờ
chiếm một thread không làm gì).

## Concept

`CompletableFuture<T>` (Java 8) mở rộng `Future<T>`, bổ sung khả năng **nối chuỗi** các bước xử
lý bất đồng bộ: `thenApply` (biến đổi kết quả), `thenCombine` (kết hợp hai future), `exceptionally`/`handle` (xử lý lỗi) — tất cả **không cần block** thread hiện tại chờ đợi, các bước tiếp theo
tự động chạy khi bước trước hoàn thành.

## Why?

Mô hình lập trình bất đồng bộ kiểu "callback thủ công" (truyền vào một hàm để gọi khi xong) dễ
dẫn tới "callback hell" (nhiều callback lồng nhau, khó đọc, khó xử lý lỗi nhất quán).
`CompletableFuture` chuẩn hoá mẫu hình này thành các phương thức nối chuỗi rõ ràng, đọc gần như
tuần tự dù thực thi bất đồng bộ — đồng thời tích hợp sẵn cơ chế lan truyền lỗi qua toàn bộ chuỗi
(giống `try-catch` nhưng cho luồng bất đồng bộ), giải quyết đúng vấn đề "callback hell" từng
gặp phải với các API bất đồng bộ kiểu cũ.

## How?

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> tinhToan())   // chay bat dong bo
    .thenApply(x -> x * 2)           // bien doi ket qua, khong block
    .exceptionally(ex -> "gia tri mac dinh neu loi"); // xu ly loi trong chuoi

String result = future.get(); // chi block O DAY, khi thuc su can ket qua cuoi
```

## Visualization

```
supplyAsync() ──► thenApply() ──► thenApply() ──► get()
   [thread pool]    [khong block]   [khong block]   [block, chi khi can ket qua cuoi]

thenCombine (ket hop 2 future doc lap):

  future1 ──┐
            ├──► thenCombine(combiner) ──► ket qua gop
  future2 ──┘

exceptionally (xu ly loi trong chuoi):

  supplyAsync() [nem loi] ──► exceptionally(ex -> giaTriMacDinh) ──► get() [khong nem loi nua]
```

## Example

```java
import java.util.concurrent.CompletableFuture;

public class CompletableFutureDemo {
    public static void main(String[] args) throws Exception {
        CompletableFuture<String> future = CompletableFuture
            .supplyAsync(() -> {
                System.out.println("Buoc 1 (thread: " + Thread.currentThread().getName() + ")");
                return 10;
            })
            .thenApply(x -> x * 2)
            .thenApply(x -> "Ket qua: " + x);

        System.out.println(future.get());

        CompletableFuture<Integer> f1 = CompletableFuture.supplyAsync(() -> 5);
        CompletableFuture<Integer> f2 = CompletableFuture.supplyAsync(() -> 7);
        CompletableFuture<Integer> combined = f1.thenCombine(f2, Integer::sum);
        System.out.println("Ket hop 2 future: " + combined.get());

        CompletableFuture<String> risky = CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("loi!");
        });
        CompletableFuture<String> withError = risky.exceptionally(ex -> "Da bat loi: " + ex.getCause().getMessage());
        System.out.println(withError.get());
    }
}
```

Kết quả thật (JDK 17):

```
Buoc 1 (thread: ForkJoinPool.commonPool-worker-1)
Ket qua: 20
Ket hop 2 future: 12
Da bat loi: loi!
```

`supplyAsync` mặc định chạy trên `ForkJoinPool.commonPool()` (đúng pool đã học ở Chapter 17) —
không phải thread chính. Chuỗi `thenApply(x*2).thenApply("Ket qua: "+x)` cho ra `"Ket qua: 20"`
(10 → 20 → chuỗi) mà không cần gọi `get()` giữa các bước. `thenCombine` gộp đúng `5 + 7 = 12`
từ hai future độc lập. `exceptionally` bắt được lỗi ném ra trong `supplyAsync`, trả về giá trị
mặc định `"Da bat loi: loi!"` thay vì để `get()` ném `ExecutionException`.

## Deep Dive

**Vì sao `ex.getCause().getMessage()` trong `exceptionally`, chứ không phải `ex.getMessage()`
trực tiếp?** Giống hệt cơ chế bọc `ExecutionException` đã học ở Chapter 14: lỗi xảy ra bên trong
`supplyAsync` (chạy trên thread pool worker) được `CompletableFuture` bọc lại thành
`CompletionException` khi truyền tới `exceptionally` — lỗi gốc (`RuntimeException("loi!")`) nằm
ở `getCause()`. Đây là mẫu hình lặp lại xuyên suốt: bất cứ khi nào exception phải "vượt biên
giới thread" (từ worker thread sang thread gọi), JDK bọc nó trong một lớp exception trung gian
để phân biệt rõ "lỗi được chuyển tiếp" với lỗi xảy ra trực tiếp tại điểm gọi.

## Engineering Insight

**`thenApply` vs `thenApplyAsync` — khác biệt tưởng nhỏ nhưng ảnh hưởng lớn tới hiệu năng.**
`thenApply(fn)` (không hậu tố `Async`) thực thi hàm `fn` **trên chính thread vừa hoàn thành
bước trước** — nếu bước trước đã xong ngay khi gọi `thenApply` (ví dụ giá trị đã có sẵn), `fn`
chạy đồng bộ luôn trên thread gọi. `thenApplyAsync(fn)` luôn đảm bảo `fn` chạy trên một thread
khác lấy từ pool (`ForkJoinPool.commonPool()` mặc định, hoặc `Executor` tuỳ chỉnh nếu truyền
vào). Với chuỗi có nhiều bước và bước nào đó tốn CPU đáng kể, lẫn lộn giữa hai biến thể này có
thể dẫn tới việc vô tình chạy công việc nặng trên một thread "nhạy cảm" (ví dụ thread xử lý
event loop trong một số framework) thay vì thread pool chuyên dụng — gây nghẽn không mong muốn.

## Historical Note

`CompletableFuture` ra đời ở Java 8 (2014), cùng đợt với Stream API và Lambda — phản ánh xu
hướng lập trình hàm (functional programming) lan rộng vào JDK giai đoạn này. Trước đó (Java 5-7),
lập trình bất đồng bộ nối chuỗi phải dựa vào các thư viện bên thứ ba (ví dụ Guava's
`ListenableFuture`) hoặc tự viết callback thủ công.

## Myth vs Reality

- **Myth:** "`CompletableFuture` luôn chạy bất đồng bộ hoàn toàn, không bao giờ block thread
  chính."
  **Reality:** Chỉ các bước trung gian (`thenApply`, `thenCombine`, ...) là non-blocking. Gọi
  `get()` để lấy kết quả cuối cùng vẫn **block** — đây là lý do trong ứng dụng thực tế, `get()`
  thường chỉ được gọi ở "điểm cuối" của luồng xử lý (ví dụ trả response HTTP), không gọi giữa
  chừng.

- **Myth:** "`exceptionally` xử lý lỗi giống hệt `try-catch` thông thường."
  **Reality:** Xem Deep Dive — lỗi bị bọc trong `CompletionException`, cần `getCause()` để lấy
  lỗi gốc, khác với `try-catch` bắt trực tiếp exception ném ra.

## Common Mistakes

- **Gọi `future.get()` ngay sau khi tạo `CompletableFuture`** — mất hết lợi ích bất đồng bộ,
  quay lại hành vi blocking như `Future` thường.
- **Nhầm lẫn `thenApply` và `thenApplyAsync`** — xem Engineering Insight, có thể vô tình chạy
  công việc nặng trên thread không phù hợp.
- **Không xử lý lỗi trong chuỗi (`exceptionally`/`handle`)** — lỗi không được xử lý sẽ lan
  truyền tới bước cuối, và `get()` sẽ ném `ExecutionException` bọc lỗi gốc, dễ bị bỏ sót nếu
  không có `try-catch` bao quanh `get()`.

## Best Practices

- Chỉ gọi `get()`/`join()` ở điểm thực sự cần kết quả cuối cùng (thường là cuối luồng xử lý),
  không gọi giữa chừng.
- Dùng `exceptionally` hoặc `handle` để xử lý lỗi tường minh trong chuỗi, tránh để lỗi "rơi
  xuống" tới `get()` mà không được xử lý đúng ngữ cảnh.
- Cân nhắc truyền `Executor` tuỳ chỉnh cho các biến thể `*Async` khi công việc nặng CPU, tránh
  chiếm dụng `ForkJoinPool.commonPool()` dùng chung toàn JVM (đã nhắc ở Chapter 17).

## Production Notes

**Vấn đề:** một service gọi 3 API bên ngoài song song bằng `CompletableFuture`, gộp kết quả
bằng `thenCombine`. Một API thỉnh thoảng lỗi (timeout mạng), khiến toàn bộ response trả lỗi 500
cho người dùng dù 2 API còn lại đã trả về thành công.

- **Triệu chứng:** tỉ lệ lỗi 500 tăng theo đúng tỉ lệ lỗi của một API phụ thuộc, dù 2 API còn
  lại vẫn hoạt động bình thường.
- **Root cause:** không có `exceptionally`/`handle` nào được đặt cho future gọi API hay lỗi —
  lỗi lan thẳng qua `thenCombine` rồi tới `get()`, làm hỏng toàn bộ chuỗi dù chỉ một phần dữ
  liệu bị thiếu.
- **Debug:** log lỗi tại điểm `get()` cho thấy `ExecutionException` bọc đúng lỗi timeout của
  API thứ 3.
- **Solution:** thêm `exceptionally` ngay sau future gọi API dễ lỗi, trả về giá trị mặc định
  (hoặc `Optional.empty()`) thay vì để lỗi lan truyền — cho phép phần response còn lại vẫn trả
  về đầy đủ, chỉ thiếu phần dữ liệu không quan trọng.
- **Prevention:** với mọi future gọi service ngoài trong một chuỗi tổng hợp nhiều nguồn, luôn
  cân nhắc future đó có "bắt buộc phải thành công" hay "có thể thiếu, dùng giá trị mặc định" —
  và xử lý lỗi tường minh theo đúng mức độ quan trọng đó.

## Debug Checklist

- [ ] `get()` ném `ExecutionException` không rõ nguyên nhân? → gọi `getCause()`, kiểm tra có
      `exceptionally`/`handle` xử lý đúng ngữ cảnh trong chuỗi không.
- [ ] Chuỗi `CompletableFuture` chạy chậm hơn mong đợi? → kiểm tra có đang lẫn lộn
      `thenApply`/`thenApplyAsync`, hoặc có `get()` gọi sớm giữa chừng làm mất tính bất đồng bộ
      không.
- [ ] Một phần lỗi từ nguồn phụ làm hỏng toàn bộ kết quả tổng hợp? → thêm `exceptionally` cho
      từng future thành phần thay vì chỉ xử lý lỗi ở bước cuối cùng.

## Summary

`CompletableFuture` mở rộng `Future` với khả năng nối chuỗi bất đồng bộ (`thenApply`,
`thenCombine`) và xử lý lỗi tích hợp (`exceptionally`) — đã chứng minh bằng thực nghiệm: chuỗi
biến đổi cho ra đúng `"Ket qua: 20"`, gộp hai future độc lập cho `12`, và lỗi ném trong
`supplyAsync` được `exceptionally` bắt đúng (qua `getCause()`, vì lỗi bị bọc trong
`CompletionException` khi vượt biên giới thread). Chỉ nên `get()`/`join()` ở điểm thực sự cần
kết quả cuối, và cần xử lý lỗi tường minh cho từng future thành phần khi tổng hợp nhiều nguồn.

## Interview Questions

**Mid**

- `CompletableFuture` khác `Future` như thế nào?
- `thenApply` khác `thenApplyAsync` như thế nào?

**Senior**

- Giải thích cách `CompletableFuture` xử lý và lan truyền lỗi trong một chuỗi nhiều bước. Vì
  sao `getCause()` cần thiết trong `exceptionally`?
- Một service tổng hợp kết quả từ 3 `CompletableFuture` song song, một trong số đó thỉnh thoảng
  lỗi làm hỏng toàn bộ response. Đề xuất cách xử lý.

## Exercises

- [ ] Chạy lại `CompletableFutureDemo` ở trên, xác nhận kết quả giống mô tả.
- [ ] Viết một chuỗi gọi 2 "API giả lập" (dùng `Thread.sleep` mô phỏng độ trễ) song song bằng
      `supplyAsync`, gộp bằng `thenCombine`, đo thời gian tổng so với gọi tuần tự.
- [ ] Thử `allOf()`/`anyOf()` với 3 `CompletableFuture`, quan sát khác biệt hành vi (chờ tất cả
      xong vs chờ cái đầu tiên xong).

## Cheat Sheet

| Phương thức | Mục đích |
| --- | --- |
| `supplyAsync(Supplier)` | Chạy bất đồng bộ, trả về giá trị |
| `thenApply(Function)` | Biến đổi kết quả, non-blocking |
| `thenCombine(other, fn)` | Gộp kết quả 2 future độc lập |
| `exceptionally(fn)` | Xử lý lỗi, trả về giá trị mặc định |
| `handle(fn)` | Xử lý cả kết quả lẫn lỗi trong một hàm |
| `get()` / `join()` | Block, chỉ dùng khi thực sự cần kết quả cuối |

## References

- Java SE API Documentation — `java.util.concurrent.CompletableFuture`.
- Java Concurrency in Practice — không có chương riêng (CompletableFuture ra đời sau sách),
  tham khảo trực tiếp Javadoc chính thức của JDK 8+.
