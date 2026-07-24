---
tags:
  - Logging
  - MDC
  - Observability
---

# Logging: Structured Logging và MDC

> Phase: Phase 8 — Production
> Chapter slug: `logging`

## Metadata

```yaml
Chapter: Logging
Phase: Phase 8 — Production
Difficulty: ★★★
Importance: ★★★★★
Interview Frequency: 75%
Prerequisites:
  - Phase 3, Chapter 01 — Thread
  - Phase 3, Chapter 15 — Thread Pool
Used Later:
  - Chapter 02 — Metrics
  - Chapter 03 — Tracing
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: Phase 3, Chapter 15 (Thread Pool) — thread trong pool được **tái sử dụng** cho nhiều
> tác vụ khác nhau, không bị huỷ sau mỗi tác vụ. Điều này tạo ra một cạm bẫy nguy hiểm với
> logging: nếu dữ liệu "gắn vào thread hiện tại" không được dọn dẹp, nó có thể **rò rỉ** sang
> tác vụ tiếp theo dùng lại đúng thread đó.

```java
// "Request 1": dat MDC, KHONG don dep
pool.submit(() -> {
    MDC.put("correlationId", "request-1-id");
    log.info("Request 1 dang xu ly...");
    // QUEN MDC.clear()
});
// "Request 2": KHONG dat MDC gi ca
pool.submit(() -> {
    log.info("Request 2, MDC hien tai: {}", MDC.get("correlationId"));
});
```

Kết quả thật:

```
[correlationId=request-1-id] Request 1 dang xu ly, thread: pool-2-thread-1
[correlationId=request-1-id] Request 2 dang xu ly, thread: pool-2-thread-1, MDC hien tai: request-1-id
```

"Request 2" **chưa bao giờ** đặt `correlationId` — nhưng log của nó vẫn hiện
`request-1-id`, vì cả hai chạy trên **cùng một thread** (`pool-2-thread-1`) và không ai dọn dẹp.

## Interview Question (Central)

> MDC (Mapped Diagnostic Context) là gì? Vì sao quên dọn dẹp MDC có thể gây ra log sai lệch
> nghiêm trọng trong một ứng dụng dùng thread pool?

## Objectives

- [ ] Dùng thành thạo MDC để gắn correlation ID vào mọi dòng log của một request
- [ ] Tự tay chứng minh bằng thực nghiệm: hai request xử lý bởi hai thread khác nhau có
      correlation ID tách biệt hoàn toàn đúng
- [ ] Tự tay tái hiện lỗi rò rỉ MDC khi tái sử dụng thread không dọn dẹp, và chứng minh
      `try/finally` khắc phục triệt để

## Prerequisites

- Phase 3, Chapter 01 — hiểu thread cơ bản.
- Phase 3, Chapter 15 — hiểu thread pool, nền tảng của cạm bẫy rò rỉ MDC.

## Used Later

- **Chapter 02 (Metrics)**, **Chapter 03 (Tracing)** — logging, metrics, tracing là "ba trụ cột"
  của observability, correlation ID (chapter này) chính là sợi chỉ nối liền cả ba.

## Problem

Một ứng dụng backend xử lý hàng nghìn request đồng thời — log của các request khác nhau **trộn
lẫn** vào cùng một luồng output (console/file). Nếu không có cách nào "gắn nhãn" mỗi dòng log
thuộc về request nào, việc debug một lỗi cụ thể (ví dụ "request của khách hàng X bị lỗi 500") trở
thành việc tìm kim đáy bể giữa hàng nghìn dòng log không liên quan.

## Concept

**MDC (Mapped Diagnostic Context)** là một `Map` **gắn theo từng thread** (dựa trên
`ThreadLocal`, Phase 3) — mọi giá trị đặt vào MDC (ví dụ `correlationId`) tự động xuất hiện
trong **mọi** dòng log tiếp theo được ghi **trên cùng thread đó**, cho tới khi bị xoá. Trong một
ứng dụng web, gắn một **correlation ID** (mã định danh duy nhất) vào MDC ngay khi request bắt
đầu cho phép lọc/gom nhóm toàn bộ log liên quan tới đúng một request cụ thể.

## Why?

Vì MDC dựa trên `ThreadLocal` (Phase 3), nó tận dụng đúng mô hình "mỗi request một thread" phổ
biến của Spring MVC (Phase 5, Chapter 09) — không cần truyền correlation ID qua tham số của mọi
phương thức, chỉ cần đặt một lần vào MDC lúc bắt đầu request, mọi lời gọi `log.info(...)` sau đó
(dù ở tầng Controller, Service, hay Repository) tự động "thấy" được giá trị đó. Nhưng chính vì
dựa trên thread, và thread trong thread pool **được tái sử dụng** (Phase 3, Chapter 15), MDC
**bắt buộc** phải được dọn dẹp tường minh khi request kết thúc — nếu không, giá trị cũ "sống
sót" sang tác vụ tiếp theo dùng lại đúng thread đó, gây log sai lệch nghiêm trọng.

## How?

```java
@Component
class CorrelationIdFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain) {
        String correlationId = UUID.randomUUID().toString().substring(0, 8);
        MDC.put("correlationId", correlationId);
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.remove("correlationId"); // BAT BUOC
        }
    }
}
```

```xml
<!-- logback-spring.xml -->
<pattern>%d{HH:mm:ss.SSS} [%thread] %-5level [correlationId=%X{correlationId}] %logger - %msg%n</pattern>
```

## Visualization

```
KHONG don dep MDC (LOI):

  Thread pool-2-thread-1:
    Task 1: MDC.put("id", "req-1") ──► log (thay "req-1") ──► [QUEN xoa]
    Task 2: (khong dat gi)         ──► log (VAN thay "req-1" - RO RI!)

CO don dep (try/finally):

  Thread pool-3-thread-1:
    Task 3: MDC.put("id", "req-3") ──► log (thay "req-3") ──► finally: MDC.clear()
    Task 4: (khong dat gi)         ──► log (thay null - DUNG)
```

## Example

**HTTP thật, hai request với correlation ID khác nhau, xử lý bởi hai thread khác nhau:**

```bash
curl -H "X-Correlation-Id: abc12345" http://localhost:8080/api/log-demo
curl -H "X-Correlation-Id: xyz98765" http://localhost:8080/api/log-demo
```

Log thật (Logback, Spring Boot 3.3.4):

```
[http-nio-8080-exec-2] INFO [correlationId=abc12345] LogDemoController - Bat dau xu ly request
[http-nio-8080-exec-2] INFO [correlationId=abc12345] LogDemoController - Dang chay logic nghiep vu
[http-nio-8080-exec-3] INFO [correlationId=xyz98765] LogDemoController - Bat dau xu ly request
[http-nio-8080-exec-3] INFO [correlationId=xyz98765] LogDemoController - Dang chay logic nghiep vu
```

Mỗi request giữ đúng correlation ID của mình xuyên suốt mọi dòng log, kể cả log phát sinh từ một
phương thức private khác (`doBusinessLogic()`) — không cần truyền tham số nào.

**Log DEBUG bị lọc đúng theo cấu hình:**

```java
log.debug("Day la log DEBUG - se KHONG hien thi vi root level la INFO");
```

Dòng này **hoàn toàn không xuất hiện** trong log thật — xác nhận cấu hình `<root level="INFO">`
lọc đúng theo mức độ đã khai báo.

**Rò rỉ MDC khi tái sử dụng thread không dọn dẹp:**

```java
ExecutorService pool = Executors.newFixedThreadPool(1); // CHI 1 thread -> DAM BAO tai su dung
pool.submit(() -> {
    MDC.put("correlationId", "request-1-id");
    log.info("Request 1 dang xu ly, thread: {}", Thread.currentThread().getName());
    // QUEN MDC.clear()
});
pool.submit(() -> {
    log.info("Request 2, thread: {}, MDC hien tai: {}", Thread.currentThread().getName(), MDC.get("correlationId"));
});
```

Kết quả thật:

```
[pool-2-thread-1] INFO [correlationId=request-1-id] Request 1 dang xu ly, thread: pool-2-thread-1
[pool-2-thread-1] INFO [correlationId=request-1-id] Request 2 dang xu ly, thread: pool-2-thread-1, MDC hien tai: request-1-id
```

"Request 2" chưa bao giờ gọi `MDC.put()` nhưng vẫn thấy `correlationId=request-1-id` — bằng
chứng trực tiếp về rò rỉ. Sửa bằng `try { ... } finally { MDC.clear(); }`:

```
[pool-3-thread-1] INFO [correlationId=request-3-id] Request 3 dang xu ly
[pool-3-thread-1] INFO [correlationId=] Request 4 dang xu ly, MDC hien tai: null
```

"Request 4" giờ đúng `null` — không còn rò rỉ.

## Deep Dive

**Vì sao `MDC.remove()`/`MDC.clear()` phải nằm trong `finally`, không phải chỉ cuối phương thức
(sau logic chính)?** Nếu logic chính ném exception (ví dụ lỗi xử lý request), luồng thực thi
**nhảy thẳng** ra khỏi phương thức, bỏ qua mọi câu lệnh sau logic chính mà không nằm trong
`finally` — MDC sẽ **không bao giờ** được dọn dẹp trong trường hợp lỗi, dẫn tới rò rỉ chính xác ở
những request **thất bại** (thường là những request quan trọng nhất cần debug đúng!). `finally`
đảm bảo dọn dẹp chạy **bất kể** phương thức kết thúc bình thường hay ném exception — đây là ứng
dụng trực tiếp của kiến thức exception handling (Phase 1, Chapter 15) vào một tình huống thực tế
cụ thể.

## Engineering Insight

**MDC (dựa trên `ThreadLocal`) gặp vấn đề gì khi kết hợp với lập trình bất đồng bộ
(`@Async`, Phase 5 Chapter 14, hoặc reactive, Phase 5 Chapter 18)?** Vì MDC gắn với **thread cụ
thể**, khi một tác vụ được giao cho thread khác xử lý (`@Async` chạy trên thread pool riêng,
reactive code có thể nhảy qua nhiều thread khác nhau trong cùng một luồng xử lý logic), MDC đặt
ở thread gốc **không tự động** theo sang thread mới — log trên thread mới sẽ **thiếu**
correlation ID, dù về mặt logic nghiệp vụ nó vẫn thuộc về cùng một request. Giải quyết vấn đề
này đòi hỏi kỹ thuật **truyền tay MDC context**: đọc `MDC.getCopyOfContextMap()` trên thread gốc
trước khi giao việc, rồi `MDC.setContextMap(...)` ngay đầu tác vụ trên thread mới (một số thư
viện, ví dụ `TaskDecorator` của Spring, tự động hoá việc này cho `@Async`) — nếu không xử lý,
log của các luồng xử lý bất đồng bộ/reactive sẽ mất khả năng theo dõi theo request, một vấn đề
thực tế phổ biến khi hệ thống chuyển từ đồng bộ sang bất đồng bộ.

## Historical Note

MDC là một tính năng của Log4j/Logback/SLF4J, xuất hiện từ khá sớm trong lịch sử logging Java
(Log4j 1.x, cuối thập niên 1990) — ý tưởng "gắn ngữ cảnh chẩn đoán theo thread" phản ánh đúng mô
hình xử lý phổ biến thời kỳ đó (mỗi request một thread, servlet container truyền thống). Sự phát
triển của lập trình bất đồng bộ/reactive (Phase 5, Chapter 14, 18) những năm gần đây bộc lộ rõ
hạn chế cố hữu của mô hình dựa trên `ThreadLocal` này, thúc đẩy các giải pháp mới như
context propagation trong OpenTelemetry (Chapter 03).

## Myth vs Reality

- **Myth:** "Chỉ cần dùng MDC là tự động có correlation ID đúng cho mọi log, không cần lo lắng
  gì thêm."
  **Reality:** Đã chứng minh bằng thực nghiệm — quên dọn dẹp khi tái sử dụng thread gây rò rỉ
  nghiêm trọng; MDC cũng không tự động "theo" sang thread khác trong code bất đồng bộ.

- **Myth:** "Log DEBUG luôn được ghi lại, chỉ là ẩn đi trên console."
  **Reality:** Đã chứng minh bằng thực nghiệm — với `root level="INFO"`, log DEBUG hoàn toàn
  không được tạo ra, không phải chỉ ẩn hiển thị.

## Common Mistakes

- **Đặt `MDC.remove()`/`clear()` ở cuối phương thức thay vì trong `finally`** — mất tác dụng dọn
  dẹp khi có exception xảy ra giữa chừng.
- **Dùng `MDC.clear()` khi chỉ nên `MDC.remove("key")` cụ thể** — `clear()` xoá **toàn bộ** MDC,
  có thể xoá nhầm dữ liệu context khác đang được dùng bởi phần code khác trên cùng thread.
- **Quên truyền MDC context khi giao việc cho `@Async`/thread khác** — log của tác vụ bất đồng bộ
  mất correlation ID, khó truy vết theo request gốc.

## Best Practices

- Luôn dùng `try { ... } finally { MDC.remove(...) hoặc clear(); }` khi làm việc trực tiếp với
  MDC.
- Dùng một `Filter`/`Interceptor` tập trung (như `CorrelationIdFilter`) để tự động gắn/dọn dẹp
  MDC cho mọi request, không rải rác logic này khắp các Controller.
- Với `@Async`/reactive, chủ động truyền tay MDC context hoặc dùng `TaskDecorator` để tránh mất
  correlation ID.
- Log theo định dạng **có cấu trúc** (structured logging — JSON) trong môi trường production để
  dễ dàng tìm kiếm/lọc bằng công cụ log aggregation (ELK, Loki).

## Debug Checklist

- [ ] Log của một request hiện correlation ID của request khác? → nghi ngờ hàng đầu là rò rỉ MDC
      do tái sử dụng thread không dọn dẹp — kiểm tra có `finally { MDC.remove/clear() }` chưa.
- [ ] Log DEBUG không xuất hiện dù đã thêm code log? → kiểm tra cấu hình mức log hiện tại
      (`root level`), có thể đang lọc DEBUG.
- [ ] Log của tác vụ `@Async` thiếu correlation ID? → cần truyền tay MDC context sang thread
      mới, xem Engineering Insight.

## Summary

MDC gắn dữ liệu ngữ cảnh (như correlation ID) vào **thread hiện tại** — tự động xuất hiện trong
mọi log ghi trên thread đó cho tới khi bị xoá. Đã chứng minh bằng thực nghiệm: hai request HTTP
xử lý bởi hai thread khác nhau giữ đúng correlation ID tách biệt hoàn toàn; nhưng khi hai tác vụ
chạy tuần tự trên **cùng một thread** (thread pool tái sử dụng, Phase 3 Chapter 15) mà không dọn
dẹp, tác vụ sau **kế thừa nhầm** giá trị MDC của tác vụ trước — sửa bằng `try/finally`. Log DEBUG
bị lọc hoàn toàn (không chỉ ẩn) khi mức log cấu hình cao hơn. MDC dựa trên `ThreadLocal` nên
không tự động "theo" sang thread khác trong code `@Async`/reactive, cần truyền tay context nếu
cần giữ correlation ID xuyên suốt luồng xử lý bất đồng bộ.

## Interview Questions

**Mid**

- MDC (Mapped Diagnostic Context) là gì?
- Vì sao quên dọn dẹp MDC có thể gây ra log sai lệch nghiêm trọng trong một ứng dụng dùng thread
  pool?

**Senior**

- MDC gặp vấn đề gì khi kết hợp với `@Async`/reactive? Cách khắc phục?
- Vì sao `MDC.remove()` phải nằm trong `finally`, không phải cuối phương thức?

## Exercises

- [ ] Chạy lại `MdcLeakTest` ở trên, xác nhận rò rỉ xảy ra khi không dọn dẹp và biến mất khi
      dùng `try/finally`.
- [ ] Gọi hai request HTTP thật liên tiếp với hai `X-Correlation-Id` khác nhau, xác nhận log
      tách biệt đúng theo từng request.
- [ ] Viết một `TaskDecorator` truyền MDC context sang thread `@Async`, xác nhận log của tác vụ
      bất đồng bộ giữ đúng correlation ID của request gốc.

## Cheat Sheet

| | Không dọn dẹp MDC | Có `try/finally` dọn dẹp |
| --- | --- | --- |
| Request/tác vụ sau trên cùng thread | Kế thừa nhầm giá trị cũ (rò rỉ) | Sạch, đúng giá trị mới hoặc `null` |
| An toàn khi có exception? | Không | Có (finally luôn chạy) |

## References

- SLF4J/Logback Documentation — Mapped Diagnostic Context (MDC).
- Spring Framework Documentation — `TaskDecorator`.
