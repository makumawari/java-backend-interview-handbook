---
tags:
  - Spring
  - AOP
---

# Aspect-Oriented Programming (AOP) và Self-Invocation

> Phase: Phase 5 — Spring
> Chapter slug: `aop`

## Metadata

```yaml
Chapter: AOP
Phase: Phase 5 — Spring
Difficulty: ★★★★
Importance: ★★★★★
Interview Frequency: 80%
Prerequisites:
  - Chapter 05 — Bean Lifecycle
  - Chapter 06 — Configuration
Used Later:
  - Chapter 13 — Transaction
  - Chapter 14 — Async
Estimated Reading: 20 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: [Chapter 05](05-bean-lifecycle.md) — bean nhận qua `@Autowired` có thể là một
> **proxy**, không phải instance gốc, tạo ra bởi `BeanPostProcessor` tại bước
> `postProcessAfterInitialization`. AOP chính là ứng dụng thực tế của cơ chế đó — nhưng proxy
> chỉ "chặn" được lời gọi **từ bên ngoài**. Điều gì xảy ra khi một method tự gọi một method khác
> **trong cùng class**?

```java
@Component
class OrderService {
    void placeOrder() {
        this.confirmOrder(); // goi qua 'this'
    }
    void confirmOrder() { ... }
}
```

Với một `@Aspect` ghi log trước/sau mỗi method của `OrderService`, gọi trực tiếp
`confirmOrder()` từ bên ngoài và gọi `placeOrder()` (tự gọi `confirmOrder()` bên trong) cho ra
kết quả khác nhau:

```
=== Goi TRUC TIEP confirmOrder() TU BEN NGOAI (qua proxy) ===
  [AOP] TRUOC: confirmOrder
confirmOrder() dang chay
  [AOP] SAU: confirmOrder

=== Goi placeOrder() (se tu goi confirmOrder() qua 'this' o BEN TRONG) ===
  [AOP] TRUOC: placeOrder
placeOrder() dang chay, tu goi confirmOrder() qua 'this'...
confirmOrder() dang chay
  [AOP] SAU: placeOrder
```

Khi gọi từ bên ngoài, `confirmOrder()` có đủ cả `[AOP] TRUOC`/`[AOP] SAU`. Nhưng khi
`placeOrder()` tự gọi `confirmOrder()` bên trong, **không hề** có dòng `[AOP]` nào cho
`confirmOrder()` — dù cùng một method, cùng một aspect.

## Interview Question (Central)

> AOP trong Spring hoạt động dựa trên cơ chế proxy như thế nào? Vì sao gọi một method có AOP từ
> bên trong cùng class (self-invocation) lại không kích hoạt advice?

## Objectives

- [ ] Giải thích các khái niệm cốt lõi của AOP: Aspect, Advice, Pointcut, Join Point
- [ ] Tự tay chứng minh bằng thực nghiệm: self-invocation (`this.method()`) không kích hoạt AOP
      advice, trong khi gọi từ bên ngoài thì có
- [ ] Hiểu đây chính là "self-invocation limitation" — nền tảng để hiểu đúng các cạm bẫy tương
      tự ở `@Transactional` (Chapter 13) và `@Async` (Chapter 14)

## Prerequisites

- Chapter 05 — hiểu `BeanPostProcessor` là nơi proxy được tạo.
- Chapter 06 — hiểu CGLIB proxy, cùng kỹ thuật nền tảng AOP sử dụng.

## Used Later

- **Chapter 13 (Transaction)**, **Chapter 14 (Async)** — cả hai đều xây trên AOP, chịu chung giới
  hạn self-invocation đã học ở chapter này.

## Problem

Nhiều mối quan tâm (concern) trong một ứng dụng thực tế **cắt ngang** (cross-cutting) qua nhiều
class/method khác nhau, không thuộc về logic nghiệp vụ cốt lõi của bất kỳ class nào riêng lẻ:
logging, đo thời gian thực thi, kiểm tra quyền truy cập, quản lý transaction (Chapter 13). Nếu
viết logic này lặp lại thủ công trong từng method cần nó, code nghiệp vụ bị "nhiễu" bởi logic hạ
tầng không liên quan, và dễ bỏ sót khi thêm method mới.

## Concept

**AOP (Aspect-Oriented Programming)** cho phép tách các "cross-cutting concern" ra thành một
**Aspect** riêng biệt, rồi khai báo **Advice** (logic cần chèn thêm, ví dụ log trước/sau) áp
dụng vào các **Join Point** (điểm trong luồng thực thi, thường là lời gọi method) khớp với một
**Pointcut** (biểu thức mô tả "áp dụng cho những method nào"). Spring AOP hiện thực hoá cơ chế
này bằng **proxy** (JDK dynamic proxy hoặc CGLIB, đã gặp ở Chapter 06) — bọc quanh bean gốc, chặn
lời gọi method để chèn logic advice trước khi (hoặc sau khi) gọi method thật.

## Why?

Vì AOP dựa trên proxy — một object **bọc bên ngoài** bean gốc — nó chỉ có thể chặn được các lời
gọi method đi **qua chính proxy đó**, tức là các lời gọi **từ bên ngoài** class (ai đó cầm
reference tới proxy, gọi `service.confirmOrder()`). Khi một method bên trong class tự gọi
`this.confirmOrder()`, lời gọi đó đi **thẳng** tới instance gốc (chính `this` bên trong method
đang chạy — không phải là proxy!), hoàn toàn **bỏ qua** proxy bọc bên ngoài — đây là giới hạn cố
hữu, không phải lỗi cấu hình, của mọi kỹ thuật AOP dựa trên proxy.

## How?

```java
@Aspect
@Component
class LoggingAspect {
    @Around("execution(* com.handbook.demo.OrderService.*(..))")
    Object logAround(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("TRUOC: " + pjp.getSignature().getName());
        Object result = pjp.proceed(); // goi method that
        System.out.println("SAU: " + pjp.getSignature().getName());
        return result;
    }
}

@Configuration
@EnableAspectJAutoProxy // BAT BUOC de kich hoat AOP
class Config { ... }
```

## Visualization

```
Goi TU BEN NGOAI (qua proxy):

  Client ──► Proxy.confirmOrder() ──► [AOP TRUOC] ──► OrderService that.confirmOrder() ──► [AOP SAU]
             (proxy CHAN duoc, chen advice)

Goi TU BEN TRONG (self-invocation, KHONG qua proxy):

  Client ──► Proxy.placeOrder() ──► [AOP TRUOC placeOrder]
                                        │
                                   OrderService THAT.placeOrder() dang chay
                                        │
                                   this.confirmOrder()  <- 'this' la INSTANCE GOC, KHONG PHAI proxy!
                                        │
                                   OrderService THAT.confirmOrder() chay TRUC TIEP
                                        │           (KHONG co [AOP TRUOC/SAU confirmOrder] nao!)
                                   quay lai placeOrder()
                                        │
                                   [AOP SAU placeOrder]
```

## Example

```java
@Aspect
@Component
static class LoggingAspect {
    @Around("execution(* com.handbook.demo.AopSelfInvocationTest.OrderService.*(..))")
    Object logAround(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("  [AOP] TRUOC: " + pjp.getSignature().getName());
        Object result = pjp.proceed();
        System.out.println("  [AOP] SAU: " + pjp.getSignature().getName());
        return result;
    }
}

@Component
static class OrderService {
    void placeOrder() {
        System.out.println("placeOrder() dang chay, tu goi confirmOrder() qua 'this'...");
        this.confirmOrder(); // goi qua 'this' - KHONG di qua proxy!
    }
    void confirmOrder() { System.out.println("confirmOrder() dang chay"); }
}
```

Kết quả thật (Spring Framework 6.1.13, `@EnableAspectJAutoProxy`):

```
Class thuc te cua bean: ...OrderService$$SpringCGLIB$$0

=== Goi TRUC TIEP confirmOrder() TU BEN NGOAI (qua proxy) ===
  [AOP] TRUOC: confirmOrder
confirmOrder() dang chay
  [AOP] SAU: confirmOrder

=== Goi placeOrder() (se tu goi confirmOrder() qua 'this' o BEN TRONG) ===
  [AOP] TRUOC: placeOrder
placeOrder() dang chay, tu goi confirmOrder() qua 'this'...
confirmOrder() dang chay
  [AOP] SAU: placeOrder
```

Class thực tế của bean là `OrderService$$SpringCGLIB$$0` — xác nhận đây là CGLIB proxy, cùng cơ
chế đã học ở Chapter 06. Gọi `confirmOrder()` trực tiếp từ bên ngoài kích hoạt đủ cả
`[AOP] TRUOC`/`[AOP] SAU`. Nhưng bên trong `placeOrder()`, dòng `this.confirmOrder()` chạy
**hoàn toàn không có** log AOP nào bao quanh nó — chỉ có `"confirmOrder() dang chay"` (log của
chính method) xuất hiện, chứng minh trực tiếp advice đã bị bỏ qua cho lời gọi self-invocation
này.

## Deep Dive

**Vì sao `this` bên trong một method lại là instance gốc, không phải proxy — dù bean được lấy
ra từ container ban đầu chính là proxy?** Khi Spring tạo proxy (CGLIB subclass hoặc JDK dynamic
proxy) bọc quanh bean gốc, proxy đó **giữ một tham chiếu nội bộ** tới instance gốc, và mỗi lời
gọi method trên proxy sẽ **uỷ quyền** (delegate) cho instance gốc sau khi chạy advice. Nhưng khi
đoạn code **bên trong chính instance gốc** đó thực thi (ví dụ thân method `placeOrder()`), từ
khoá `this` trong ngữ cảnh đó luôn tham chiếu tới **chính instance đang chạy** — tức instance
gốc, không phải proxy bọc bên ngoài nó. Proxy hoàn toàn "vô hình" đối với chính bản thân object
nó đang bọc — đây là bản chất kỹ thuật cố hữu, không có cách nào "sửa" bằng cấu hình mà không
thay đổi cách viết code.

## Engineering Insight

**Có những cách nào để né tránh self-invocation limitation khi thực sự cần?** (1) **Tách method
cần advice ra một bean/class khác** — cách rõ ràng nhất: nếu `confirmOrder()` cần AOP hoạt động
khi được gọi từ `placeOrder()`, tách `confirmOrder()` sang một bean riêng, rồi `OrderService`
tiêm bean đó vào và gọi nó (lúc này là lời gọi từ bên ngoài, đi qua proxy đúng cách). (2) **Tự
lấy proxy của chính mình** qua `AopContext.currentProxy()` (yêu cầu bật
`exposeProxy = true`) rồi gọi qua proxy đó thay vì qua `this` — cách này hoạt động nhưng làm code
phụ thuộc chặt vào chi tiết triển khai AOP, thường không được khuyến khích vì làm giảm khả năng
đọc hiểu code. Cách (1) — tái cấu trúc tách method ra bean riêng — hầu như luôn là lựa chọn tốt
hơn, và thường **cũng chính là dấu hiệu** cho thấy `confirmOrder()` xứng đáng là một trách nhiệm
độc lập, không nên gộp chung với `placeOrder()` trong cùng class ngay từ đầu.

## Historical Note

Spring AOP (dựa trên proxy runtime, chỉ áp dụng cho bean do Spring quản lý) khác với AspectJ đầy
đủ (compile-time hoặc load-time weaving, áp dụng được cho **mọi** object, kể cả object không do
Spring tạo) — Spring AOP dùng lại cú pháp pointcut của AspectJ (`execution(...)`, đã thấy trong
Example) để quen thuộc và tương thích, nhưng cơ chế thực thi bên dưới hoàn toàn khác (proxy thay
vì chèn bytecode trực tiếp vào class file). Đây là lý do self-invocation limitation chỉ tồn tại
với Spring AOP — AspectJ weaving đầy đủ (ít dùng hơn trong dự án Spring Boot thông thường vì cần
cấu hình build phức tạp hơn) không gặp giới hạn này vì nó chèn logic trực tiếp vào bytecode của
class, không qua proxy bên ngoài.

## Myth vs Reality

- **Myth:** "AOP luôn hoạt động bất kể method được gọi từ đâu, miễn method đó khớp pointcut."
  **Reality:** Đã chứng minh bằng thực nghiệm — self-invocation (gọi qua `this` từ bên trong
  cùng class) hoàn toàn bỏ qua advice, dù method đó khớp pointcut một cách chính xác.

- **Myth:** "Self-invocation limitation là một lỗi/thiếu sót của Spring AOP, sẽ được sửa ở phiên
  bản sau."
  **Reality:** Đây là giới hạn **cố hữu** của kỹ thuật proxy-based AOP nói chung (không riêng
  Spring) — không phải lỗi, mà là hệ quả tất yếu của cách proxy hoạt động, đã tồn tại xuyên suốt
  lịch sử Spring AOP và sẽ luôn tồn tại với cách tiếp cận proxy.

## Common Mistakes

- **Gọi method có `@Transactional`/`@Async`/AOP tuỳ chỉnh qua `this` từ bên trong cùng class,
  kỳ vọng advice vẫn hoạt động** — đây là cạm bẫy phổ biến nhất liên quan tới AOP trong Spring,
  sẽ gặp lại chính xác vấn đề này ở Chapter 13-14.
- **Không hiểu vì sao class chứa method có AOP không được khai báo `final`** — giống lý do đã
  học ở Chapter 06, CGLIB cần tạo subclass.
- **Viết pointcut quá rộng** (ví dụ khớp mọi method của mọi class trong một package lớn) — có
  thể vô tình áp dụng advice cho các method không nên bị ảnh hưởng, gây tác dụng phụ khó lường.

## Best Practices

- Khi thiết kế class có method cần AOP (transaction, cache, async, ...), luôn nhớ self-invocation limitation — tách method thành bean riêng nếu cần gọi từ bên trong cùng class mà vẫn muốn
  advice hoạt động.
- Viết pointcut expression cụ thể, rõ ràng, tránh áp dụng advice nhầm phạm vi.
- Cân nhắc self-invocation limitation như một tín hiệu thiết kế: nếu gặp nó thường xuyên, có thể
  class đang gánh quá nhiều trách nhiệm, nên tách nhỏ (tương tự bài học về constructor quá dài ở
  Chapter 03).

## Debug Checklist

- [ ] Advice AOP (log, transaction, cache) không hoạt động dù đã đánh dấu đúng annotation? →
      kiểm tra method đó có đang được gọi qua `this` từ bên trong cùng class không (self-invocation).
- [ ] Cần xác nhận một bean có đang là AOP proxy không? → in `bean.getClass().getName()`, tìm
      hậu tố `$$SpringCGLIB$$` hoặc kiểm tra qua `AopUtils.isAopProxy(bean)`.

## Summary

Spring AOP dùng proxy (CGLIB/JDK dynamic proxy, Chapter 06) để chèn Advice vào các method khớp
Pointcut. Đã chứng minh bằng thực nghiệm: gọi method từ **bên ngoài** (qua proxy) kích hoạt đúng
advice, nhưng self-invocation (`this.method()` từ **bên trong** cùng class) hoàn toàn **bỏ qua**
advice — vì `this` bên trong instance gốc không phải là proxy bọc bên ngoài nó, cơ chế này vô
hình đối với chính bản thân object. Đây là giới hạn cố hữu của mọi kỹ thuật proxy-based AOP,
không phải lỗi cấu hình — cách khắc phục phổ biến nhất là tách method cần advice ra một bean
riêng. Kiến thức này là nền tảng bắt buộc trước khi học `@Transactional` (Chapter 13) và
`@Async` (Chapter 14), cả hai đều gặp đúng giới hạn này.

## Interview Questions

**Senior**

- AOP trong Spring hoạt động dựa trên cơ chế proxy như thế nào?
- Vì sao gọi một method có AOP từ bên trong cùng class (self-invocation) lại không kích hoạt
  advice? Giải thích bằng cơ chế kỹ thuật cụ thể.
- Có những cách nào để né tránh self-invocation limitation khi thực sự cần advice hoạt động?

## Exercises

- [ ] Chạy lại `AopSelfInvocationTest` ở trên, xác nhận kết quả đúng như mô tả.
- [ ] Tách `confirmOrder()` ra một bean `ConfirmationService` riêng, tiêm vào `OrderService`, xác
      nhận advice hoạt động đúng khi gọi qua bean riêng đó từ bên trong `placeOrder()`.
- [ ] Thử dùng `AopContext.currentProxy()` (kèm `exposeProxy = true`) để gọi
      `confirmOrder()` qua proxy từ bên trong `placeOrder()`, xác nhận advice hoạt động đúng
      theo cách này.

## Cheat Sheet

| Khái niệm | Ý nghĩa |
| --- | --- |
| Aspect | Class đóng gói một cross-cutting concern (`@Aspect`) |
| Advice | Logic thực thi tại join point (`@Before`, `@After`, `@Around`, ...) |
| Pointcut | Biểu thức mô tả tập hợp join point áp dụng advice (`execution(...)`) |
| Join Point | Một điểm cụ thể trong luồng thực thi (thường là lời gọi method) |
| Self-invocation | Gọi method qua `this` từ bên trong cùng class — **bỏ qua** AOP proxy |

## References

- Spring Framework Documentation — Aspect Oriented Programming with Spring: https://docs.spring.io/spring-framework/reference/core/aop.html
- Spring Framework Documentation — "Understanding AOP Proxies" (giải thích chính thức về
  self-invocation limitation).
