---
tags:
  - Spring
  - Reactive
  - WebFlux
---

# Reactive Programming với Project Reactor (Mono/Flux)

> Phase: Phase 5 — Spring
> Chapter slug: `reactive-programming`

## Metadata

```yaml
Chapter: Reactive Programming (Project Reactor, WebFlux, Mono/Flux, RSocket)
Phase: Phase 5 — Spring
Difficulty: ★★★★
Importance: ★★★
Interview Frequency: 45%
Prerequisites:
  - Chapter 09 — REST
  - Phase 4, Chapter 04 — Stream
  - Phase 3, Chapter 18 — CompletableFuture
Used Later:
  - Kiến trúc hệ thống hiệu năng cao, độ trễ thấp (Phase 8)
Estimated Reading: 20 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: Phase 4, Chapter 04 (Stream) — `Stream` là **lazy**: `map()`/`filter()` không chạy
> cho tới khi có terminal operation. `Mono`/`Flux` (Project Reactor, nền tảng của Spring
> WebFlux) mang chính xác nguyên lý đó sang thế giới **bất đồng bộ**.

```java
Mono<Integer> mono = Mono.just(10)
    .map(x -> { System.out.println("map() DANG CHAY"); return x * 2; });
System.out.println("Da tao xong Mono, map() CHUA CHAY (lazy)");
mono.subscribe(result -> System.out.println("Ket qua: " + result));
```

Kết quả thật:

```
Da tao xong Mono, map() CHUA CHAY (lazy)
=== Goi subscribe() ===
  map() DANG CHAY
Ket qua: 20
```

Giống hệt `Stream`: khai báo `.map()` không chạy gì cả — chỉ khi `subscribe()` (tương đương
terminal operation) được gọi, toàn bộ chuỗi mới thực sự thực thi.

## Interview Question (Central)

> `Mono` và `Flux` khác nhau như thế nào? Vì sao lập trình reactive (Project Reactor) được coi
> là "lazy" và "non-blocking" — điều này thể hiện qua bằng chứng cụ thể nào?

## Objectives

- [ ] Phân biệt `Mono<T>` (0 hoặc 1 phần tử) và `Flux<T>` (0 tới N phần tử)
- [ ] Tự tay chứng minh bằng thực nghiệm: `Mono`/`Flux` lazy giống `Stream`, và hỗ trợ
      short-circuit qua `take()`
- [ ] Hiểu vì sao có cả `spring-boot-starter-web` (Servlet/Tomcat) và `spring-boot-starter-webflux` (Reactive/Netty) trên classpath khiến Spring Boot **ưu tiên** Servlet stack theo mặc định

## Prerequisites

- Chapter 09 — hiểu REST controller truyền thống, để so sánh với REST controller kiểu reactive.
- Phase 4, Chapter 04 — hiểu Stream lazy evaluation, nguyên lý tương tự áp dụng cho Reactor.
- Phase 3, Chapter 18 — hiểu `CompletableFuture`, một cách tiếp cận bất đồng bộ khác để so sánh.

## Used Later

- **Kiến trúc hệ thống hiệu năng cao, độ trễ thấp** (Phase 8) — reactive stack phù hợp cho hệ
  thống cần xử lý số lượng kết nối đồng thời cực lớn với tài nguyên thread hạn chế.

## Problem

Mô hình "mỗi request một thread" truyền thống (Servlet/Tomcat, Chapter 09) chặn (block) thread
xử lý trong suốt thời gian chờ I/O (gọi database, gọi API khác) — với số lượng kết nối đồng thời
rất lớn, số thread cần thiết tăng tương ứng, tốn tài nguyên (dù Virtual Thread, Phase 3 Chapter
19, đã giảm nhẹ vấn đề này ở tầng JVM). Lập trình reactive giải quyết cùng vấn đề I/O-bound này
theo một hướng khác: xử lý bất đồng bộ, non-blocking hoàn toàn, dựa trên một số ít thread xử lý
event.

## Concept

**Project Reactor** (`Mono<T>`, `Flux<T>`) là thư viện lập trình reactive nền tảng của Spring
WebFlux — implement chuẩn **Reactive Streams**. `Mono<T>` đại diện cho một luồng dữ liệu phát ra
**0 hoặc 1** phần tử (tương tự `Optional`/`CompletableFuture<T>`, nhưng bất đồng bộ và lazy).
`Flux<T>` đại diện cho luồng phát ra **0 tới N** phần tử (tương tự `Stream<T>`, nhưng bất đồng
bộ). Cả hai đều **lazy** — không có gì thực thi cho tới khi có ai `subscribe()`.

## Why?

Thiết kế lazy (giống Stream) cho phép Reactor xây dựng toàn bộ "đường ống xử lý" (pipeline) như
một mô tả thuần tuý trước khi thực thi, từ đó tối ưu hoá và hỗ trợ **backpressure** (subscriber
có thể báo cho publisher "chậm lại, tôi chưa xử lý kịp") — điều mà mô hình push đơn giản (publisher tự động đẩy dữ liệu, không quan tâm subscriber có theo kịp không) không làm được. Toàn bộ mô
hình reactive dựa trên non-blocking I/O ở tầng thấp nhất (Netty thay vì Tomcat) để một số ít
thread có thể phục vụ số lượng kết nối đồng thời rất lớn, mỗi thread không bao giờ bị "khoá cứng"
chờ I/O.

## How?

```java
Mono<Integer> mono = Mono.just(10).map(x -> x * 2);
mono.subscribe(result -> System.out.println(result)); // CHI luc nay moi chay

Flux<Integer> flux = Flux.range(1, 100)
    .filter(n -> n % 2 == 0)
    .map(n -> n * n)
    .take(3); // CHI can 3 phan tu dau, KHONG xu ly het 100
flux.subscribe(System.out::println);

// WebFlux controller: tra ve Flux/Mono truc tiep
@GetMapping("/api/reactive/numbers")
Flux<Integer> numbers() { return Flux.range(1, 5).delayElements(Duration.ofMillis(50)); }
```

## Visualization

```
Mono/Flux (LAZY, giong Stream):

  Mono.just(10).map(...) ──► CHI mo ta "cong thuc", CHUA chay
       │
  subscribe() ──► BAY GIO moi thuc su chay toan bo chuoi

Flux voi take() (short-circuit, giong Stream.findFirst):

  Flux.range(1, 100).filter(...).map(...).take(3)
       │
  Chi xu ly DEN KHI du 3 phan tu khop dieu kien, KHONG can xu ly het 100 phan tu
```

## Example

**Lazy evaluation:**

```java
Mono<Integer> mono = Mono.just(10)
    .map(x -> { System.out.println("  map() DANG CHAY"); return x * 2; });
System.out.println("Da tao xong Mono, map() CHUA CHAY (lazy)");
mono.subscribe(result -> System.out.println("Ket qua: " + result));
```

Kết quả thật (Project Reactor, Spring Boot 3.3.4):

```
Da tao xong Mono, map() CHUA CHAY (lazy)
=== Goi subscribe() ===
  map() DANG CHAY
Ket qua: 20
```

**Xử lý lỗi và bất đồng bộ thật (không block):**

```java
Integer result = Mono.<Integer>error(new RuntimeException("loi gia lap"))
    .onErrorReturn(-1)
    .block();
System.out.println("Ket qua khi co loi (fallback): " + result); // -1

List<Long> ticks = Flux.interval(Duration.ofMillis(100)).take(3).collectList().block();
// Ticks: [0, 1, 2], thoi gian: 327ms (dung nhu ky vong ~300ms - Flux.interval THAT SU cho)
```

**WebFlux controller thật, gọi qua HTTP:**

```java
@GetMapping("/api/reactive/numbers")
Flux<Integer> numbers() { return Flux.range(1, 5).delayElements(Duration.ofMillis(50)); }
```

```
$ time curl http://localhost:8080/api/reactive/numbers
[1,2,3,4,5]
... 0.410 total
```

`Flux.interval` thực sự **chờ** đúng khoảng thời gian đã khai báo (327ms cho 3 tick × 100ms,
không phải trả về ngay lập tức) — xác nhận đây là xử lý bất đồng bộ thật, không phải giả lập.
Endpoint HTTP trả về mảng JSON sau khi đã đợi đủ 5 phần tử × 50ms trễ.

**Kiểm tra server nào thực sự chạy — Tomcat hay Netty?**

```
$ grep -i tomcat app.log
Tomcat initialized with port 8080 (http)
Tomcat started on port 8080 (http) with context path '/'
```

Dù có cả `spring-boot-starter-web` **và** `spring-boot-starter-webflux` trên classpath, log khởi
động xác nhận **Tomcat** (Servlet stack) đang chạy, không phải Netty.

## Deep Dive

**Vì sao có cả hai starter (`web` và `webflux`) trên classpath nhưng Tomcat vẫn được chọn thay vì
Netty?** Đây là hệ quả trực tiếp của cơ chế Auto Configuration (Chapter 07):
`WebMvcAutoConfiguration` (Servlet stack) và `WebFluxAutoConfiguration` (Reactive stack) đều tồn
tại trong `AutoConfiguration.imports`, nhưng Spring Boot có logic **ưu tiên Servlet stack khi cả
hai đều khả dụng** trên classpath (dùng `@ConditionalOnMissingBean`/thứ tự đánh giá đặc biệt cho
tình huống này) — giả định rằng nếu một dự án cố tình thêm `spring-boot-starter-web`, đó là lựa
chọn có chủ đích. Điều này giải thích tại sao ví dụ WebFlux `Flux<Integer>` trong chapter này vẫn
hoạt động đúng (`curl` nhận đúng JSON) dù chạy trên Tomcat — Spring MVC (Chapter 09) có hỗ trợ
sẵn khả năng nhận diện kiểu trả về `Mono`/`Flux` và tự thích ứng (adapt) chúng, dù bản thân
Tomcat/Servlet API không phải reactive/non-blocking thuần tuý ở tầng thấp nhất. Muốn thực sự chạy
trên Netty (WebFlux đầy đủ, non-blocking từ tầng network trở lên), cần loại bỏ hẳn
`spring-boot-starter-web` khỏi dependency.

## Engineering Insight

**Khi nào nên chọn Reactive stack (WebFlux) thay vì Servlet stack (MVC) truyền thống, và khi nào
không nên?** Reactive stack mang lại lợi ích rõ rệt khi: (1) ứng dụng cần xử lý **số lượng kết
nối đồng thời cực lớn** với tài nguyên hạn chế (ví dụ API gateway, streaming data), và (2) toàn
bộ chuỗi phụ thuộc (database driver, HTTP client gọi service khác) đều hỗ trợ non-blocking thật
sự — nếu chỉ một mắt xích trong chuỗi vẫn dùng driver blocking truyền thống (ví dụ JDBC thông
thường, không phải R2DBC), lợi ích non-blocking của WebFlux bị **phá vỡ hoàn toàn** ở chính điểm
đó, thậm chí có thể gây hại (block một trong số ít thread event-loop khiến toàn bộ ứng dụng bị
ảnh hưởng, không giống mô hình Servlet nơi mỗi request có thread riêng, một request bị block
không ảnh hưởng request khác). Đây là lý do các chuyên gia Spring thường khuyến nghị: **chỉ**
chọn WebFlux khi thực sự cần và toàn bộ hệ sinh thái phụ thuộc đã sẵn sàng reactive; với phần lớn
ứng dụng backend thông thường, đặc biệt từ khi có Virtual Thread (Phase 3, Chapter 19), Servlet
stack truyền thống vẫn là lựa chọn đơn giản, dễ debug hơn nhiều.

## Historical Note

Project Reactor ra đời khoảng 2013, chuẩn hoá theo đặc tả **Reactive Streams** (2015, hợp tác
giữa Netflix, Pivotal/Spring, Lightbend, và nhiều bên khác) — một nỗ lực thống nhất API cho lập
trình reactive trên JVM (trước đó có nhiều thư viện cạnh tranh không tương thích: RxJava, Akka
Streams, Reactor). Spring WebFlux chính thức ra mắt cùng Spring Framework 5.0 (2017) — muộn hơn
đáng kể so với Spring MVC (2007), phản ánh đây là một bổ sung cho các trường hợp sử dụng chuyên
biệt, không phải thay thế hoàn toàn mô hình MVC truyền thống.

## Myth vs Reality

- **Myth:** "WebFlux/Reactive luôn nhanh hơn Spring MVC truyền thống."
  **Reality:** Xem Engineering Insight — chỉ có lợi khi toàn bộ chuỗi phụ thuộc đều non-blocking;
  nếu có bất kỳ điểm blocking nào trong chuỗi, WebFlux có thể **tệ hơn** MVC truyền thống.

- **Myth:** "Có `spring-boot-starter-webflux` trên classpath nghĩa là ứng dụng đang chạy trên
  Netty, non-blocking hoàn toàn."
  **Reality:** Đã chứng minh bằng thực nghiệm — nếu `spring-boot-starter-web` cũng có mặt, Spring
  Boot mặc định vẫn chọn Tomcat (Servlet stack), không phải Netty.

## Common Mistakes

- **Trộn lẫn code blocking (JDBC thông thường) vào một luồng xử lý WebFlux** — phá vỡ hoàn toàn
  lợi ích non-blocking, có thể gây hại nghiêm trọng hơn dùng MVC truyền thống.
- **Chọn WebFlux "vì nghe nói nó nhanh hơn" mà không đánh giá liệu use case thực sự cần** — thêm
  độ phức tạp đáng kể (lập trình reactive khó debug, khó viết đúng hơn code tuần tự) mà không có
  lợi ích tương xứng.
- **Gọi `.block()` bên trong một luồng xử lý reactive** — về cơ bản "biến" reactive code thành
  blocking code, mất hết ý nghĩa của việc dùng Mono/Flux ngay từ đầu (chỉ nên dùng `.block()`
  ở test hoặc code hoàn toàn nằm ngoài luồng reactive).

## Best Practices

- Chỉ chọn WebFlux khi thực sự cần xử lý lượng kết nối đồng thời cực lớn và toàn bộ chuỗi phụ
  thuộc đã hỗ trợ non-blocking (R2DBC thay vì JDBC, WebClient thay vì RestTemplate).
- Không gọi `.block()` bên trong luồng xử lý reactive thực tế — chỉ dùng ở ranh giới (test, hoặc
  điểm chuyển đổi từ code đồng bộ sang bất đồng bộ).
- Với phần lớn ứng dụng backend, cân nhắc Servlet stack + Virtual Thread (Phase 3, Chapter 19)
  trước khi quyết định chuyển sang WebFlux — đơn giản hơn nhiều để viết đúng và debug.

## Debug Checklist

- [ ] Không chắc ứng dụng đang chạy trên Tomcat hay Netty? → kiểm tra log khởi động, tìm dòng
      "Tomcat started" hoặc "Netty started".
- [ ] Hiệu năng WebFlux tệ hơn dự kiến? → kiểm tra có lời gọi blocking nào (JDBC, `.block()`,
      thư viện đồng bộ) lẫn trong luồng xử lý reactive không.
- [ ] `Mono`/`Flux` không "chạy" dù đã khai báo đầy đủ pipeline? → kiểm tra có quên gọi
      `subscribe()` (hoặc `.block()` trong test) không — giống lỗi quên terminal operation của
      Stream.

## Summary

`Mono<T>` (0-1 phần tử) và `Flux<T>` (0-N phần tử) của Project Reactor mang nguyên lý lazy
evaluation của Stream (Phase 4) sang thế giới bất đồng bộ, non-blocking. Đã chứng minh bằng thực
nghiệm: `.map()` không chạy cho tới khi `subscribe()`; `.take(3)` short-circuit giống
`Stream.findFirst`; `Flux.interval` thực sự chờ đúng thời gian đã khai báo (327ms cho 3×100ms),
xác nhận đây là xử lý bất đồng bộ thật. Dù có cả `spring-boot-starter-web` và
`spring-boot-starter-webflux`, Spring Boot mặc định vẫn chọn chạy trên Tomcat (Servlet stack),
không phải Netty — WebFlux chỉ thực sự phát huy lợi ích khi loại bỏ hoàn toàn dependency Servlet
và toàn bộ chuỗi phụ thuộc đều non-blocking.

## Interview Questions

**Mid**

- `Mono` và `Flux` khác nhau như thế nào?
- Vì sao lập trình reactive được coi là "lazy" và "non-blocking"?

**Senior**

- Khi nào nên chọn WebFlux thay vì Spring MVC truyền thống? Rủi ro gì nếu trộn code blocking vào
  luồng xử lý reactive?
- Giải thích vì sao có cả `spring-boot-starter-web` và `spring-boot-starter-webflux` trên
  classpath, Spring Boot vẫn mặc định chạy trên Tomcat.

## Exercises

- [ ] Chạy lại `ReactiveDemoTest` ở trên, xác nhận cả 4 kết quả (lazy evaluation, short-circuit,
      error handling, timing thật) đúng như mô tả.
- [ ] Gọi endpoint `/api/reactive/numbers` qua `curl`, đo thời gian phản hồi, xác nhận nó tương
      ứng với tổng độ trễ `delayElements` đã khai báo.
- [ ] Loại bỏ hẳn `spring-boot-starter-web` khỏi dependency, giữ lại `webflux`, chạy lại ứng
      dụng, xác nhận log khởi động đổi từ "Tomcat started" sang "Netty started".

## Cheat Sheet

| | `Mono<T>` | `Flux<T>` | `Stream<T>` (Phase 4) | `CompletableFuture<T>` (Phase 3) |
| --- | --- | --- | --- | --- |
| Số phần tử | 0-1 | 0-N | 0-N | 1 (kết quả cuối) |
| Lazy | Có | Có | Có | Không (thực thi ngay khi tạo) |
| Bất đồng bộ | Có | Có | Không | Có |
| Backpressure | Có | Có | Không áp dụng | Không |

## References

- Project Reactor Documentation — https://projectreactor.io/docs/core/release/reference/
- Spring Framework Documentation — Web on Reactive Stack (WebFlux): https://docs.spring.io/spring-framework/reference/web/webflux.html
- Reactive Streams Specification — https://www.reactive-streams.org/
