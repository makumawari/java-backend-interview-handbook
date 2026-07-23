---
tags:
  - Spring
  - Async
---

# @Async

> Phase: Phase 5 — Spring
> Chapter slug: `async`

## Metadata

```yaml
Chapter: Async
Phase: Phase 5 — Spring
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 65%
Prerequisites:
  - Chapter 12 — AOP
  - Phase 3, Chapter 15 — Thread Pool
Used Later:
  - Chapter 15 — Scheduler
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 12](12-aop.md), [Chapter 13](13-transaction.md) — cả hai đều là AOP advice,
> đều chịu giới hạn self-invocation. `@Async` cũng không ngoại lệ. Interview Question điển hình
> cho tính năng này: **"`@Async` đã bật, nhưng method vẫn chạy đồng bộ. Vì sao?"**

Gọi trực tiếp một method `@Async` từ bên ngoài, rồi thử gọi nó **gián tiếp** qua một method khác
trong cùng class (self-invocation) — in ra tên thread thực thi:

```
=== Goi TRUC TIEP sendEmailAsync() qua proxy ===
Main test thread: main
  sendEmailAsync(alice@example.com) chay tren thread: task-1

=== Goi placeOrderNoAsync() -> tu goi sendEmailAsync() qua 'this' ===
Main test thread: main
placeOrderNoAsync chay tren thread: main
  sendEmailAsync(bob@example.com) chay tren thread: main
```

Gọi trực tiếp: `sendEmailAsync` chạy trên thread `task-1` — khác hẳn thread `main` gọi nó, đúng
bản chất bất đồng bộ. Gọi qua self-invocation: `sendEmailAsync` chạy **ngay trên thread `main`**
— hoàn toàn đồng bộ, dù annotation `@Async` vẫn còn nguyên trên method đó.

## Interview Question (Central)

> "@Async" is enabled, but the method still executes synchronously. What could be wrong?

## Objectives

- [ ] Hiểu `@Async` cũng là AOP advice, chịu chung giới hạn self-invocation đã học ở Chapter 12
- [ ] Tự tay chứng minh bằng thực nghiệm: gọi trực tiếp `@Async` method chạy trên thread pool
      riêng; self-invocation khiến nó chạy đồng bộ trên chính thread gọi
- [ ] Biết các nguyên nhân khác khiến `@Async` "không hoạt động": thiếu `@EnableAsync`, gọi
      trong cùng thread do cấu hình `SimpleAsyncTaskExecutor` không đúng

## Prerequisites

- Chapter 12 — bắt buộc hiểu AOP proxy và self-invocation trước chapter này.
- Phase 3, Chapter 15 — hiểu Thread Pool, vì `@Async` mặc định chạy task trên một thread pool
  quản lý bởi Spring.

## Used Later

- **Chapter 15 (Scheduler)** — `@Scheduled` có thể kết hợp `@Async` để không chặn thread lập lịch
  chính khi task chạy lâu.

## Problem

Gọi một tác vụ tốn thời gian (gửi email, gọi API bên ngoài) theo cách đồng bộ khiến thread hiện
tại (ví dụ thread xử lý HTTP request) bị **chặn đứng** chờ tác vụ đó hoàn thành, dù bản thân tác
vụ không ảnh hưởng tới kết quả trả về ngay lập tức cho client. `@Async` cho phép "giao" tác vụ đó
cho một thread khác xử lý, để thread hiện tại tiếp tục ngay — nhưng cũng như mọi tính năng dựa
trên AOP proxy khác, nó có thể bị vô hiệu hoá một cách âm thầm nếu gọi sai cách.

## Concept

**`@Async`** đánh dấu một method sẽ được thực thi **bất đồng bộ** — khi được gọi (từ bên ngoài,
qua proxy), Spring không chạy thân method trên thread gọi, mà **giao** nó cho một thread khác
(lấy từ thread pool, mặc định là `SimpleAsyncTaskExecutor` — tạo thread mới mỗi lần, nên khuyến
nghị cấu hình `ThreadPoolTaskExecutor` riêng cho production, Phase 3 Chapter 15) — thread gọi
nhận lại quyền điều khiển **ngay lập tức**, không chờ tác vụ đó hoàn thành.

## Why?

`@Async` cũng hiện thực hoá qua AOP proxy — hoàn toàn cùng cơ chế với `@Transactional` (Chapter
13) và AOP tuỳ chỉnh (Chapter 12): proxy chặn lời gọi từ bên ngoài, giao task cho executor thay
vì chạy trực tiếp. Vì vậy nó **kế thừa chính xác** giới hạn self-invocation: khi gọi qua `this`
từ bên trong cùng class, lời gọi đi thẳng tới instance gốc, bỏ qua hoàn toàn lớp logic "giao cho
thread khác" của proxy — method chạy **ngay tại chỗ, đồng bộ**, y hệt như annotation `@Async`
không hề tồn tại.

## How?

```java
@EnableAsync // BAT BUOC, dat tren mot @Configuration class
class AsyncConfig {}

@Service
class NotificationService {
    @Async
    void sendEmailAsync(String to) {
        // chay tren thread KHAC, khong chan thread goi
    }
}
```

## Visualization

```
Goi TU BEN NGOAI qua proxy:

  Thread "main" ──► Proxy.sendEmailAsync() ──► GIAO cho thread pool
       │                                              │
  Thread "main" TIEP TUC NGAY                    Thread "task-1" chay sendEmailAsync() that
  (khong cho doi)

Self-invocation (goi qua 'this'):

  Thread "main" ──► Proxy.placeOrderNoAsync() ──► instance GOC dang chay
                                                        │
                                                   this.sendEmailAsync() <- KHONG QUA PROXY!
                                                        │
                                                   chay TRUC TIEP tren CHINH thread "main"
                                                   (dong bo, KHONG giao cho thread nao khac)
```

## Example

```java
@Service
public class NotificationService {
    @Async
    public void sendEmailAsync(String to) {
        System.out.println("  sendEmailAsync(" + to + ") chay tren thread: " + Thread.currentThread().getName());
    }

    public void placeOrderNoAsync(String to) {
        System.out.println("placeOrderNoAsync chay tren thread: " + Thread.currentThread().getName());
        this.sendEmailAsync(to); // SELF-INVOCATION
    }
}
```

Kết quả thật (Spring Boot 3.3.4):

```
=== Goi TRUC TIEP sendEmailAsync() qua proxy ===
Main test thread: main
  sendEmailAsync(alice@example.com) chay tren thread: task-1

=== Goi placeOrderNoAsync() -> tu goi sendEmailAsync() qua 'this' ===
Main test thread: main
placeOrderNoAsync chay tren thread: main
  sendEmailAsync(bob@example.com) chay tren thread: main
```

Gọi trực tiếp `sendEmailAsync()` từ test (thread `main`): thân method thực thi trên thread
`task-1` — một thread hoàn toàn khác, xác nhận đúng hành vi bất đồng bộ. Gọi qua
`placeOrderNoAsync()` (tự gọi `sendEmailAsync()` bên trong qua `this`): thân method của
`sendEmailAsync` chạy **ngay trên thread `main`** — giống hệt thread của `placeOrderNoAsync` gọi
nó, chứng minh `@Async` đã bị bỏ qua hoàn toàn, method chạy đồng bộ như một lời gọi Java thông
thường.

## Deep Dive

**Vì sao lỗi self-invocation với `@Async` thường khó phát hiện hơn cả với `@Transactional`
(Chapter 13)?** Với `@Transactional`, hậu quả (dữ liệu không rollback) thường lộ ra tương đối rõ
qua kiểm tra dữ liệu sai lệch. Với `@Async`, hậu quả **chỉ là hiệu năng** — method vẫn chạy
**đúng logic**, cho **đúng kết quả**, chỉ là chạy đồng bộ thay vì bất đồng bộ. Ứng dụng vẫn hoạt
động "đúng" về mặt chức năng, chỉ **chậm hơn dự kiến** (thread gọi bị chặn chờ tác vụ hoàn thành
thay vì trả về ngay) — loại lỗi thuần về hiệu năng này cực kỳ khó phát hiện qua test chức năng
thông thường (mọi test case đều pass), chỉ lộ ra khi đo lường thời gian phản hồi thực tế hoặc
theo dõi tên thread thực thi.

## Engineering Insight

**`@Async` trả về `void` so với trả về `CompletableFuture<T>` (Phase 3, Chapter 18) — khác biệt
quan trọng nào cần lưu ý?** Method `@Async` trả về `void` là "fire-and-forget" hoàn toàn — code
gọi **không có cách nào** biết task đã hoàn thành hay thất bại (exception ném ra bên trong bị
"nuốt" mất, chỉ log lại nếu có cấu hình `AsyncUncaughtExceptionHandler` riêng, mặc định gần như
im lặng). Method `@Async` trả về `CompletableFuture<T>` cho phép code gọi **theo dõi** kết quả
(`thenApply`, `exceptionally`, đã học ở Phase 3 Chapter 18) khi cần thiết — kết hợp hai kiến thức
này (AOP self-invocation + CompletableFuture) là nền tảng để thiết kế đúng các luồng xử lý bất
đồng bộ phức tạp trong một ứng dụng Spring thực tế, tránh tình trạng "gọi async nhưng không biết
bao giờ xong, có lỗi hay không".

## Historical Note

`@Async` xuất hiện từ Spring 3.0 (2009), cùng thời điểm Java-based configuration
(`@Configuration`, Chapter 06) ra đời — phản ánh xu hướng chung của Spring giai đoạn này: chuyển
dần các tính năng vốn phải cấu hình XML phức tạp (task executor, thread pool) sang annotation đơn
giản hơn.

## Myth vs Reality

- **Myth:** "Chỉ cần đánh dấu `@Async` trên method là đảm bảo nó luôn chạy bất đồng bộ."
  **Reality:** Đã chứng minh bằng thực nghiệm — self-invocation khiến nó chạy hoàn toàn đồng bộ,
  không có bất kỳ cảnh báo nào.

- **Myth:** "Exception xảy ra bên trong method `@Async` sẽ làm crash ứng dụng hoặc lộ rõ lỗi
  ngay lập tức."
  **Reality:** Với method trả về `void`, exception bị "nuốt" âm thầm (chỉ log nếu có cấu hình
  riêng) — đây là một cạm bẫy khác cần lưu ý khi thiết kế method `@Async`.

## Common Mistakes

- **Gọi method `@Async` qua `this` từ bên trong cùng class** — lỗi self-invocation, giống hệt
  mẫu hình đã gặp ở `@Transactional` (Chapter 13).
- **Quên `@EnableAsync`** — annotation `@Async` hoàn toàn không có tác dụng nếu thiếu bước kích
  hoạt này ở một `@Configuration` class.
- **Không cấu hình `ThreadPoolTaskExecutor` riêng, để mặc định `SimpleAsyncTaskExecutor`** — mỗi
  lời gọi `@Async` tạo một thread mới hoàn toàn (không tái sử dụng), lặp lại đúng vấn đề hiệu
  năng đã học ở Phase 3, Chapter 15 (Thread Pool).

## Best Practices

- Tránh gọi method `@Async` qua `this` — tách ra bean riêng nếu cần gọi từ bên trong cùng class,
  đúng giải pháp đã học ở Chapter 12.
- Luôn cấu hình `ThreadPoolTaskExecutor` riêng cho `@Async` trong production, không dùng mặc
  định `SimpleAsyncTaskExecutor`.
- Cân nhắc trả về `CompletableFuture<T>` thay vì `void` khi code gọi cần biết kết quả/lỗi của
  tác vụ bất đồng bộ.
- Cấu hình `AsyncUncaughtExceptionHandler` để không bỏ lỡ exception xảy ra trong method `@Async`
  trả về `void`.

## Debug Checklist

- [ ] Method `@Async` vẫn chạy trên cùng thread với nơi gọi nó? → kiểm tra có đang gọi qua `this`
      (self-invocation) không; kiểm tra `@EnableAsync` đã được khai báo chưa.
- [ ] Không rõ một tác vụ `@Async` có lỗi hay không, không thấy log nào? → kiểm tra method có
      trả về `void` không (exception bị nuốt âm thầm); cân nhắc đổi sang
      `CompletableFuture<T>` hoặc cấu hình `AsyncUncaughtExceptionHandler`.
- [ ] Hiệu năng kém dù đã dùng `@Async`? → kiểm tra executor đang dùng có phải
      `SimpleAsyncTaskExecutor` mặc định (tạo thread mới mỗi lần) không.

## Summary

`@Async` là AOP advice (giống `@Transactional`, Chapter 13), giao method cho một thread khác
thực thi thay vì chạy trên thread gọi. Đã chứng minh bằng thực nghiệm: gọi trực tiếp từ bên
ngoài (qua proxy) chạy đúng trên thread pool riêng (`task-1`); self-invocation (`this.method()`)
khiến nó chạy đồng bộ trên chính thread gọi (`main`), dù annotation vẫn còn nguyên — đúng câu
trả lời cho câu hỏi phỏng vấn kinh điển "`@Async` đã bật nhưng vẫn chạy đồng bộ". Lỗi này khó
phát hiện hơn self-invocation của `@Transactional` vì kết quả logic vẫn đúng, chỉ chậm hơn dự
kiến. Method trả về `void` khiến exception bên trong bị nuốt âm thầm — nên cân nhắc
`CompletableFuture<T>` khi cần theo dõi kết quả/lỗi.

## Interview Questions

- "@Async" is enabled, but the method still executes synchronously. What could be wrong?

**Mid**

- Vì sao self-invocation khiến `@Async` không hoạt động?

**Senior**

- So sánh `@Async` trả về `void` và trả về `CompletableFuture<T>` — khác biệt quan trọng nào cần
  lưu ý khi thiết kế method bất đồng bộ trong Spring?

## Exercises

- [ ] Chạy lại `AsyncSelfInvocationTest` ở trên, xác nhận kết quả tên thread đúng như mô tả.
- [ ] Đổi `sendEmailAsync` để trả về `CompletableFuture<Void>`, viết code gọi
      `.thenRun()`/`.exceptionally()` để xử lý kết quả.
- [ ] Cấu hình một `ThreadPoolTaskExecutor` riêng cho `@Async`, xác nhận tên thread thực thi đổi
      theo tên pool đã cấu hình (thay vì `task-N` mặc định).

## Cheat Sheet

| Kịch bản | Thread thực thi |
| --- | --- |
| Gọi từ bên ngoài (qua proxy) | Thread khác (từ async executor) |
| Self-invocation (`this.method()`) | Cùng thread với nơi gọi (đồng bộ, `@Async` bị bỏ qua) |
| Thiếu `@EnableAsync` | Cùng thread (annotation hoàn toàn vô tác dụng) |

## References

- Spring Framework Documentation — Task Execution and Scheduling: https://docs.spring.io/spring-framework/reference/integration/scheduling.html
- Spring Framework Documentation — "Understanding AOP Proxies" (self-invocation, chung cho
  `@Async`/`@Transactional`/AOP tuỳ chỉnh).
