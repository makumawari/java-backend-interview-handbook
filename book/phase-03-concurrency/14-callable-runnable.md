---
tags:
  - Java
  - Callable
  - Runnable
  - Concurrency
---

# Callable vs Runnable

> Phase: Phase 3 — Concurrency
> Chapter slug: `callable-runnable`

## Metadata

```yaml
Chapter: Callable vs Runnable
Phase: Phase 3 — Concurrency
Difficulty: ★★
Importance: ★★★
Interview Frequency: 55%
Prerequisites:
  - Chapter 01 — Thread
Used Later:
  - Chapter 15 — Thread Pool (ExecutorService.submit nhận cả Runnable và Callable)
  - Chapter 18 — CompletableFuture
Estimated Reading: 12 phút
Estimated Practice: 10 phút
```

## Story

> Xem lại: [Chapter 01](01-thread.md) — `Runnable` có đúng một phương thức `run()`, trả về
> `void`, và **không được phép** khai báo checked exception.

Điều đó ổn với các tác vụ "chạy xong là xong" (ví dụ log một dòng, gửi một thông báo). Nhưng
nếu tác vụ chạy trên thread khác cần **trả về một kết quả tính toán** — ví dụ gọi một API và
cần lấy dữ liệu trả về — thì sao? Và nếu tác vụ đó có thể ném ra một checked exception (ví dụ
`IOException` khi gọi API), làm sao khai báo nó khi `run()` không cho phép?

## Interview Question (Central)

> `Callable` khác `Runnable` như thế nào? Vì sao cần cả hai thay vì chỉ dùng một interface duy
> nhất?

## Objectives

- [ ] Phân biệt chính xác `Runnable` (không trả về giá trị, không ném checked exception) và
      `Callable<V>` (trả về `V`, được phép ném checked exception)
- [ ] Tự tay lấy được kết quả trả về từ `Callable` qua `Future.get()`
- [ ] Hiểu vì sao exception ném ra trong `Callable` được bọc trong `ExecutionException` khi gọi
      `future.get()`

## Prerequisites

- Chapter 01 — hiểu `Runnable` cơ bản, cách chạy một task trên thread riêng.

## Used Later

- **Chapter 15 (Thread Pool)** — `ExecutorService.submit()` nhận cả `Runnable` lẫn `Callable`.
- **Chapter 18 (CompletableFuture)** — mô hình "task trả về kết quả bất đồng bộ" là nền tảng
  của `CompletableFuture`.

## Problem

`Runnable.run()` không trả về giá trị (`void`) và không được khai báo checked exception (chữ
ký `void run()` không có `throws`). Với các tác vụ chạy trên thread khác nhưng cần **kết quả**
(ví dụ tính toán, gọi API) hoặc có thể **ném lỗi có kiểm tra** (checked exception), `Runnable`
không đủ biểu đạt — buộc phải dùng biến ngoài (không thread-safe nếu không cẩn thận) để "lấy"
kết quả, hoặc bọc checked exception thành unchecked để vượt qua giới hạn chữ ký.

## Concept

`Callable<V>` (gói `java.util.concurrent`) là interface tương tự `Runnable` nhưng phương thức
`call()` **trả về giá trị kiểu `V`** và **được phép ném bất kỳ `Exception` nào** (kể cả
checked). Khi nộp (`submit`) một `Callable` cho `ExecutorService`, kết quả trả về là một
`Future<V>` — đại diện cho kết quả **sẽ có trong tương lai**, lấy ra bằng `future.get()`
(blocking cho tới khi task xong).

## Why?

Thiết kế tách `Runnable` và `Callable` phản ánh đúng hai nhu cầu khác nhau: `Runnable` có từ
Java 1.0 (1996), thiết kế cho mô hình "fire-and-forget" (chạy xong không cần biết kết quả) —
ví dụ target của `Thread`. `Callable` ra đời cùng `java.util.concurrent` ở Java 5 (2004), khi
nhu cầu "chạy bất đồng bộ và lấy kết quả sau" trở nên phổ biến với `ExecutorService`. Giữ cả
hai interface (thay vì chỉ giữ `Callable` tổng quát hơn) là vì tương thích ngược — hàng triệu
dòng code Java 1.0 đã dùng `Runnable`, và với các tác vụ thực sự không cần trả kết quả,
`Runnable` vẫn đơn giản, gọn hơn.

## How?

```java
ExecutorService pool = Executors.newSingleThreadExecutor();

Callable<Integer> task = () -> {
    Thread.sleep(100);
    return 42; // co the return gia tri
};
Future<Integer> future = pool.submit(task);
Integer result = future.get(); // blocking cho toi khi task xong
```

## Visualization

```
Runnable:                          Callable<V>:

  run() -> void                      call() -> V
  KHONG duoc throws checked           duoc throws bat ky Exception nao

  submit(Runnable)                   submit(Callable<V>)
       │                                  │
       ▼                                  ▼
  Future<?>  (get() luon tra ve null)  Future<V> (get() tra ve V, hoac
                                              nem ExecutionException boc loi that)
```

## Example

```java
import java.util.concurrent.*;

public class CallableDemo {
    public static void main(String[] args) throws Exception {
        ExecutorService pool = Executors.newSingleThreadExecutor();

        Callable<Integer> task = () -> {
            Thread.sleep(100);
            return 42;
        };
        Future<Integer> future = pool.submit(task);
        System.out.println("Callable tra ve: " + future.get());

        Callable<Integer> failingTask = () -> { throw new RuntimeException("loi trong task"); };
        Future<Integer> future2 = pool.submit(failingTask);
        try {
            future2.get();
        } catch (ExecutionException e) {
            System.out.println("ExecutionException boc loi that: " + e.getCause().getMessage());
        }

        pool.shutdown();
    }
}
```

Kết quả thật (JDK 17):

```
Callable tra ve: 42
ExecutionException boc loi that: loi trong task
```

`future.get()` cho task thành công trả về đúng `42` — giá trị `Callable` đã `return`.
Với task ném lỗi, `future.get()` **không ném trực tiếp `RuntimeException` gốc** mà bọc nó
trong `ExecutionException` — lỗi thật nằm ở `e.getCause()`. Đây là điểm khác biệt quan trọng so
với gọi trực tiếp: exception xảy ra trên thread khác (thread pool worker) được "chuyển giao" về
thread gọi `get()` thông qua cơ chế bọc này.

## Deep Dive

**Vì sao exception phải bọc trong `ExecutionException` thay vì ném thẳng?** Exception xảy ra
trên **thread khác** (thread pool worker) — nó không thể tự động "nhảy" sang thread đang gọi
`future.get()` như một exception thông thường (exception chỉ lan truyền trong cùng một call
stack, một thread). `Future` phải **lưu lại** exception đó khi nó xảy ra trên worker thread,
rồi khi thread gọi `get()` đến lấy kết quả, `Future` mới "ném lại" nó — nhưng vì đây là một
hành động khác về bản chất (ném một exception đã xảy ra trước đó, trên thread khác, chứ không
phải exception vừa xảy ra ngay bây giờ), JDK chọn bọc nó trong `ExecutionException` để phân
biệt rõ ràng "đây là lỗi từ task, được chuyển tiếp qua Future" với lỗi xảy ra trực tiếp tại
điểm gọi `get()`.

## Historical Note

`Callable` được thêm vào Java 5 (2004) cùng toàn bộ gói `java.util.concurrent` (JSR 166).
Trước đó (Java 1.0-1.4), chỉ có `Runnable`, và các lập trình viên muốn "lấy kết quả" từ một
thread phải tự implement thủ công (biến instance dùng chung, kèm đồng bộ hoá thủ công) —
`Callable`/`Future` chuẩn hoá mẫu hình này thành API chính thức của JDK.

## Myth vs Reality

- **Myth:** "`Runnable` đã lỗi thời, nên luôn dùng `Callable` cho mọi tác vụ."
  **Reality:** Với tác vụ thực sự không cần trả kết quả và không ném checked exception,
  `Runnable` đơn giản và rõ ràng hơn — không cần thêm kiểu generic `V` không dùng tới.

- **Myth:** "`future.get()` ném thẳng exception gốc từ task."
  **Reality:** Xem Example và Deep Dive — nó luôn bọc trong `ExecutionException`, phải gọi
  `getCause()` để lấy lỗi thật.

## Common Mistakes

- **Quên gọi `getCause()` khi bắt `ExecutionException`** — code chỉ log `e.getMessage()` sẽ mất
  thông tin lỗi thật (message của `ExecutionException` thường generic, không phải message của
  lỗi gốc).
- **Dùng `Callable` cho tác vụ không cần trả kết quả** — thêm độ phức tạp không cần thiết
  (kiểu generic, giá trị trả về giả `null`), `Runnable` là lựa chọn đúng cho trường hợp này.
- **Gọi `future.get()` không kèm timeout trên thread chính** — nếu task treo vô hạn,
  `future.get()` cũng chờ vô hạn theo; nên dùng `future.get(timeout, unit)` khi cần giới hạn
  thời gian chờ.

## Best Practices

- Dùng `Runnable` cho tác vụ "fire-and-forget", `Callable<V>` khi cần kết quả trả về hoặc cần
  ném checked exception.
- Luôn kiểm tra `getCause()` khi bắt `ExecutionException` để không mất thông tin lỗi gốc.
- Cân nhắc `future.get(timeout, TimeUnit)` thay vì `get()` không giới hạn khi có khả năng task
  bị treo.

## Debug Checklist

- [ ] `future.get()` bị treo mãi không trả về? → kiểm tra task có bị deadlock/vòng lặp vô hạn
      không; cân nhắc dùng `get(timeout, unit)`.
- [ ] Bắt được `ExecutionException` nhưng thông tin lỗi không rõ ràng? → gọi `getCause()` để
      lấy đúng exception gốc từ task.

## Summary

`Runnable` (`run()` → `void`, không throws checked) phù hợp cho tác vụ "fire-and-forget";
`Callable<V>` (`call()` → `V`, được throws bất kỳ `Exception`) phù hợp khi cần kết quả trả về.
Đã chứng minh bằng thực nghiệm: `future.get()` trả về đúng giá trị `Callable` return
(`42`), và exception từ task được bọc trong `ExecutionException` — lỗi thật nằm ở
`getCause()`, vì exception phải được "chuyển giao" từ thread pool worker sang thread gọi
`get()`.

## Interview Questions

- Callable vs Runnable.

**Mid**

- Vì sao `future.get()` ném `ExecutionException` thay vì ném thẳng exception gốc từ task?

## Exercises

- [ ] Chạy lại `CallableDemo` ở trên, xác nhận kết quả `42` và thông điệp lỗi đúng như mô tả.
- [ ] Thử gọi `future.get(50, TimeUnit.MILLISECONDS)` với một task `Thread.sleep(200)`, quan
      sát `TimeoutException` được ném ra.

## Cheat Sheet

| | `Runnable` | `Callable<V>` |
| --- | --- | --- |
| Phương thức | `run()` | `call()` |
| Giá trị trả về | Không (`void`) | Có (`V`) |
| Checked exception | Không được throws | Được throws bất kỳ `Exception` |
| `submit()` trả về | `Future<?>` | `Future<V>` |
| Từ Java | 1.0 | 5 |

## References

- Java SE API Documentation — `java.lang.Runnable`, `java.util.concurrent.Callable`,
  `java.util.concurrent.Future`.
