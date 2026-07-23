---
tags:
  - Spring
  - ExceptionHandler
---

# Xử lý lỗi tập trung với @RestControllerAdvice

> Phase: Phase 5 — Spring
> Chapter slug: `exception-handler`

## Metadata

```yaml
Chapter: Exception Handler
Phase: Phase 5 — Spring
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 70%
Prerequisites:
  - Chapter 09 — REST
  - Chapter 10 — Validation
Used Later:
  - Chapter 13 — Transaction (exception quyết định rollback)
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 10](10-validation.md) — `MethodArgumentNotValidException` (khi `@Valid`
> thất bại) đã tự động trả về response lỗi có cấu trúc rõ ràng. Nhưng đó là nhờ đã có sẵn một
> `@ExceptionHandler` xử lý riêng cho nó. Điều gì xảy ra với một exception **không ai xử lý**?

Gọi một endpoint cố tình ném ra một exception không có `@ExceptionHandler` nào bắt:

```
=== Exception KHONG duoc @ExceptionHandler nao xu ly (default Spring Boot error) ===
{"timestamp":"2026-07-23T08:43:38.925+00:00","status":500,"error":"Internal Server Error","path":"/api/boom"}
HTTP_STATUS:500
```

So sánh với một exception nghiệp vụ **có** handler xử lý riêng (`ProductNotFoundException`, đã
gặp ở Chapter 09):

```
{"timestamp":"2026-07-23T08:07:59.566161Z","status":404,"error":"Not Found","message":"Khong tim thay san pham voi id=999"}
HTTP_STATUS:404
```

Cùng là lỗi, nhưng một cái trả về `500` chung chung, không rõ nguyên nhân thật; cái còn lại trả
về đúng `404` với thông điệp cụ thể ("không tìm thấy sản phẩm với id=999") — sự khác biệt hoàn
toàn nằm ở việc có `@ExceptionHandler` xử lý riêng hay không.

## Interview Question (Central)

> `@ExceptionHandler`/`@RestControllerAdvice` hoạt động như thế nào? Nếu không xử lý exception
> tường minh, Spring Boot trả về response gì mặc định?

## Objectives

- [ ] Dùng thành thạo `@RestControllerAdvice` + `@ExceptionHandler` để xử lý lỗi tập trung
- [ ] Tự tay chứng minh bằng thực nghiệm khác biệt giữa response mặc định (`500`, chung chung)
      và response từ handler tường minh (status code + message đúng ngữ cảnh)
- [ ] Hiểu vì sao xử lý lỗi tập trung (một nơi duy nhất) tốt hơn xử lý rải rác trong từng
      controller

## Prerequisites

- Chapter 09 — hiểu REST controller cơ bản, nơi exception có thể xảy ra.
- Chapter 10 — đã thấy một ví dụ `@ExceptionHandler` cụ thể cho lỗi validation.

## Used Later

- **Chapter 13 (Transaction)** — loại exception (checked/unchecked) quyết định transaction có tự
  động rollback hay không, liên quan trực tiếp tới cách thiết kế exception hierarchy.

## Problem

Nếu không có cơ chế xử lý lỗi tập trung, mỗi controller method phải tự bọc `try-catch` để trả về
response lỗi đúng định dạng (status code phù hợp, thông điệp rõ ràng) — dẫn tới code lặp lại ở
khắp nơi, và dễ **quên** xử lý một loại exception nào đó, khiến nó "lọt" ra ngoài dưới dạng lỗi
`500` chung chung, không có thông tin hữu ích cho client (và có nguy cơ lộ thông tin nhạy cảm
qua stack trace nếu không cấu hình cẩn thận).

## Concept

**`@RestControllerAdvice`** đánh dấu một class là nơi xử lý lỗi **tập trung cho toàn bộ ứng
dụng** (không gắn với một controller cụ thể). Bên trong nó, mỗi phương thức đánh dấu
**`@ExceptionHandler(ExceptionType.class)`** sẽ được gọi tự động **bất cứ khi nào** một exception
kiểu đó (hoặc kiểu con của nó) được ném ra từ bất kỳ controller nào trong ứng dụng — DispatcherServlet chặn exception trước khi nó lan ra client, tìm handler phù hợp, và gọi nó để tạo response
lỗi theo đúng ý muốn.

## Why?

Tách xử lý lỗi ra khỏi từng controller (gom vào một nơi duy nhất) áp dụng đúng nguyên tắc DRY
(Don't Repeat Yourself) và Separation of Concerns: controller chỉ cần tập trung viết logic
"đường đi thành công" (happy path), ném ra exception mang ý nghĩa nghiệp vụ rõ ràng khi có lỗi
(`ProductNotFoundException`, không phải tự dàn dựng response lỗi tại chỗ) — còn việc "biến
exception đó thành response HTTP đúng chuẩn" là trách nhiệm của một nơi duy nhất, dễ kiểm soát,
dễ đảm bảo mọi lỗi đều được xử lý nhất quán trên toàn ứng dụng.

## How?

```java
@RestControllerAdvice // xu ly loi cho TOAN BO ung dung, khong rieng 1 controller
class GlobalExceptionHandler {
    @ExceptionHandler(ProductNotFoundException.class)
    ResponseEntity<Map<String,Object>> handleNotFound(ProductNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(Map.of("message", ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ResponseEntity<Map<String,Object>> handleValidation(MethodArgumentNotValidException ex) {
        // ... (Chapter 10)
    }
}
```

## Visualization

```
Exception KHONG co handler:

  Controller nem RuntimeException ──► DispatcherServlet khong tim thay @ExceptionHandler khop
       ──► forward noi bo toi "/error" ──► BasicErrorController (mac dinh cua Spring Boot)
       ──► {"status":500,"error":"Internal Server Error","path":"..."}  (CHUNG CHUNG, mat thong tin)

Exception CO handler:

  Controller nem ProductNotFoundException ──► DispatcherServlet TIM THAY @ExceptionHandler khop
       ──► goi handleNotFound(ex) ──► {"status":404,"message":"Khong tim thay san pham voi id=999"}
                                        (RO RANG, dung ngu canh nghiep vu)
```

## Example

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<Map<String, Object>> handleNotFound(ProductNotFoundException ex) {
        Map<String, Object> body = new LinkedHashMap<>();
        body.put("timestamp", Instant.now().toString());
        body.put("status", 404);
        body.put("error", "Not Found");
        body.put("message", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(body);
    }
}

// Mot endpoint KHAC, cố tinh nem loi khong co handler rieng
@GetMapping("/api/boom")
String boom() { throw new IllegalStateException("loi khong ai xu ly ca!"); }
```

Kết quả thật (Spring Boot 3.3.4):

```
=== /api/boom (IllegalStateException, KHONG co handler) ===
{"timestamp":"2026-07-23T08:43:38.925+00:00","status":500,"error":"Internal Server Error","path":"/api/boom"}
HTTP_STATUS:500

=== /api/products/999 (ProductNotFoundException, CO handler) ===
{"timestamp":"2026-07-23T08:07:59.566161Z","status":404,"error":"Not Found","message":"Khong tim thay san pham voi id=999"}
HTTP_STATUS:404
```

Response mặc định (`/api/boom`) chỉ có `"error":"Internal Server Error"` — hoàn toàn không có
thông tin về nguyên nhân thật (`"loi khong ai xu ly ca!"` bị **giấu đi**, không xuất hiện trong
response, dù nó có trong log server). Response từ handler tường minh (`/api/products/999`) có
đúng status `404` và message mô tả chính xác vấn đề — sự khác biệt hoàn toàn do có/không có
`@ExceptionHandler` xử lý riêng cho loại exception đó.

## Deep Dive

**Vì sao message thật (`"loi khong ai xu ly ca!"`) không xuất hiện trong response `500` mặc
định, dù nó chắc chắn có trong log server?** Đây là hành vi **bảo mật có chủ đích** của Spring
Boot: `BasicErrorController` (class xử lý lỗi mặc định) không lộ chi tiết exception nội bộ ra
response cho client theo mặc định — vì thông điệp exception có thể vô tình tiết lộ thông tin
nhạy cảm (cấu trúc database, đường dẫn file nội bộ, logic nghiệp vụ) cho bất kỳ ai gọi API, kể
cả kẻ tấn công đang dò tìm lỗ hổng. Có thể bật `server.error.include-message=always` trong cấu
hình để hiện thông điệp exception ra response, nhưng đây là điều cần cân nhắc kỹ — chỉ nên bật
trong môi trường phát triển, không nên bật mặc định ở production.

## Engineering Insight

**Thiết kế exception hierarchy hợp lý ảnh hưởng thế nào tới `@ExceptionHandler`?** Vì
`@ExceptionHandler(X.class)` bắt cả **subtype** của `X` (tính đa hình, Phase 1 Chapter 08), thiết
kế một exception hierarchy có ý nghĩa (ví dụ `BusinessException` là lớp cha chung cho mọi lỗi
nghiệp vụ, `ProductNotFoundException`/`InsufficientStockException` kế thừa từ nó) cho phép viết
**một** handler chung xử lý toàn bộ nhóm lỗi nghiệp vụ (`@ExceptionHandler(BusinessException.class)`), đồng thời vẫn có thể viết handler **riêng biệt, cụ thể hơn** cho một exception con nếu
cần xử lý khác biệt — Spring tự động chọn handler khớp **cụ thể nhất** (specific nhất) khi có
nhiều handler cùng áp dụng được cho một exception. Đây là lý do các dự án lớn thường xây dựng
một exception hierarchy rõ ràng ngay từ đầu, thay vì dùng `RuntimeException` trần trụi cho mọi
lỗi nghiệp vụ.

## Historical Note

`@ExceptionHandler` có từ Spring 3.0 (2009), ban đầu chỉ áp dụng **trong phạm vi một
controller** (đặt trực tiếp trong class controller đó). `@ControllerAdvice` (Spring 3.2, 2013)
mở rộng phạm vi áp dụng ra **toàn ứng dụng** — giải quyết đúng vấn đề lặp lại code xử lý lỗi
giữa nhiều controller khác nhau. `@RestControllerAdvice` (Spring 4.3, 2016) là tổ hợp
`@ControllerAdvice` + `@ResponseBody`, cùng logic với `@RestController` (Chapter 09) đã học.

## Myth vs Reality

- **Myth:** "Không xử lý exception tường minh nghĩa là ứng dụng sẽ crash hoặc không phản hồi."
  **Reality:** Đã chứng minh — Spring Boot luôn có một response mặc định (`500 Internal Server
  Error`) cho mọi exception không được xử lý riêng, ứng dụng không hề "crash", chỉ là response
  không có thông tin hữu ích cho client.

- **Myth:** "Thông điệp exception (`ex.getMessage()`) luôn xuất hiện trong response lỗi mặc
  định, tiện cho việc debug từ phía client."
  **Reality:** Xem Deep Dive — đây là hành vi bảo mật có chủ đích, mặc định **ẩn** thông điệp
  chi tiết khỏi response.

## Common Mistakes

- **Không có `@RestControllerAdvice` nào trong ứng dụng, để mọi lỗi rơi vào response `500` mặc
  định** — client nhận được thông tin vô nghĩa, khó debug từ phía client, và không phân biệt
  được lỗi do client (4xx) hay do server (5xx).
- **Bật `server.error.include-message=always` ở production** — có nguy cơ lộ thông tin nhạy cảm
  qua thông điệp exception cho bất kỳ client nào gọi API.
- **Viết `try-catch` xử lý lỗi rải rác trong từng controller thay vì tập trung vào
  `@RestControllerAdvice`** — lặp lại code, dễ thiếu nhất quán giữa các endpoint.

## Best Practices

- Luôn có ít nhất một `@RestControllerAdvice` xử lý các loại exception nghiệp vụ phổ biến, và
  một handler "catch-all" cho `Exception.class` để đảm bảo mọi lỗi đều có response nhất quán
  (dù là lỗi không lường trước).
- Thiết kế exception hierarchy có ý nghĩa cho lỗi nghiệp vụ, tận dụng tính đa hình để viết handler
  gọn gàng hơn.
- Không lộ thông tin nhạy cảm (message exception chi tiết, stack trace) trong response ở
  production; log đầy đủ chi tiết ở phía server để debug nội bộ.

## Debug Checklist

- [ ] Client nhận `500 Internal Server Error` không rõ nguyên nhân? → kiểm tra log server để
      thấy exception thật, cân nhắc thêm `@ExceptionHandler` riêng cho loại exception đó.
- [ ] `@ExceptionHandler` không được gọi dù đúng loại exception? → kiểm tra exception có được
      ném ra **trong luồng xử lý của Spring MVC** không (ví dụ exception ném trong một thread
      riêng, Phase 3, sẽ không được `@ExceptionHandler` bắt vì nó không nằm trong luồng request
      gốc).
- [ ] Cần ẩn/hiện thông điệp lỗi chi tiết tuỳ môi trường? → cấu hình
      `server.error.include-message` khác nhau giữa profile development và production.

## Summary

`@RestControllerAdvice` + `@ExceptionHandler` xử lý lỗi tập trung cho toàn ứng dụng — mỗi
handler tự động được gọi khi đúng loại exception (hoặc subtype) được ném ra từ bất kỳ
controller nào. Đã chứng minh bằng thực nghiệm đối chiếu trực tiếp: exception không có handler
riêng trả về `500` chung chung, ẩn hoàn toàn thông điệp thật (hành vi bảo mật có chủ đích); exception có handler trả về đúng status code và message rõ ràng theo ngữ cảnh nghiệp vụ. Thiết kế
exception hierarchy hợp lý cho phép viết handler chung cho một nhóm lỗi, đồng thời vẫn xử lý
riêng được các trường hợp cụ thể cần khác biệt.

## Interview Questions

**Mid**

- `@ExceptionHandler`/`@RestControllerAdvice` hoạt động như thế nào?
- Nếu không xử lý exception tường minh, Spring Boot trả về response gì mặc định?

**Senior**

- Vì sao thông điệp exception mặc định không xuất hiện trong response `500`? Khi nào nên/không
  nên bật hiển thị nó?
- Thiết kế exception hierarchy ảnh hưởng thế nào tới việc viết `@ExceptionHandler` hiệu quả?

## Exercises

- [ ] Chạy lại ví dụ trên, so sánh trực tiếp response của `/api/boom` (không handler) và
      `/api/products/999` (có handler).
- [ ] Thêm một `@ExceptionHandler(Exception.class)` "catch-all" ở cuối `GlobalExceptionHandler`,
      xác nhận nó chỉ được gọi khi không có handler cụ thể hơn nào khớp.
- [ ] Thiết kế một exception hierarchy đơn giản (`BusinessException` là cha,
      `ProductNotFoundException`/`InsufficientStockException` là con), viết một handler chung
      cho `BusinessException` và quan sát nó bắt được cả hai loại con.

## Cheat Sheet

| | Không có handler | Có `@ExceptionHandler` |
| --- | --- | --- |
| Status code | Luôn `500` | Tuỳ chỉnh theo ngữ cảnh (`404`, `400`, ...) |
| Message trả về client | Ẩn (chỉ "Internal Server Error") | Tuỳ chỉnh, rõ ràng |
| Xử lý | `BasicErrorController` mặc định | Method do bạn viết trong `@RestControllerAdvice` |
| Phạm vi | Toàn ứng dụng (fallback) | Tuỳ khai báo loại exception |

## References

- Spring Framework Documentation — Exception Handling: https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-exceptionhandler.html
- Spring Boot Documentation — Error Handling.
