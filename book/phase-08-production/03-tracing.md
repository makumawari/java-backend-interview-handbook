---
tags:
  - Tracing
  - Micrometer Tracing
  - Observability
---

# Tracing: Distributed Tracing với Micrometer Tracing

> Phase: Phase 8 — Production
> Chapter slug: `tracing`

## Metadata

```yaml
Chapter: Tracing
Phase: Phase 8 — Production
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 45%
Prerequisites:
  - Phase 8, Chapter 01 — Logging
  - Phase 8, Chapter 02 — Metrics
Used Later:
  - Chapter 06 — Kafka
  - Chapter 07 — Messaging
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

Một endpoint `/api/tracing-demo/order` gọi sang một endpoint khác `/api/tracing-demo/payment`
qua HTTP, dùng `RestTemplate` tự khởi tạo bằng `new RestTemplate()`:

```java
private final RestTemplate restTemplate = new RestTemplate(); // tu khoi tao
```

Log thật của hai request (cùng một luồng xử lý nghiệp vụ, nhưng khác thread):

```
[traceId=6a6313f833522927d32a06c255bae710] [correlationId=57e02d11] Bat dau xu ly don hang (span goc)
[traceId=6a6313f8ec95914f7bb8bb1c30d0df4d] [correlationId=8dcf5b0b] Dang xu ly thanh toan (span con, goi tu order)
```

Hai `traceId` **khác nhau hoàn toàn**! Về mặt nghiệp vụ, đây là **một** luồng xử lý duy nhất (tạo
đơn hàng → thanh toán), nhưng hệ thống tracing ghi nhận thành **hai trace tách biệt**, không có
liên kết nào giữa chúng. Nguyên nhân: `new RestTemplate()` tự khởi tạo **không được Spring Boot
tự động instrument** — nó không đính kèm bất kỳ header truyền ngữ cảnh trace nào vào request đi
ra. Đổi sang `RestTemplateBuilder` (được Spring Boot tự động cấu hình sẵn instrumentation khi có
`micrometer-tracing-bridge-brave` trên classpath) sửa triệt để vấn đề — xem Deep Dive.

## Interview Question (Central)

> Trong một hệ thống microservices, làm sao để biết một request cụ thể đã đi qua những service
> nào và service nào là nguyên nhân gây chậm? Vì sao chỉ dùng correlation ID (Chapter 01) là chưa
> đủ?

## Objectives

- [ ] Phân biệt đúng `traceId` (xuyên suốt toàn bộ luồng xử lý, có thể qua nhiều service) và
      `spanId` (một đoạn xử lý cụ thể trong luồng đó)
- [ ] Tự tay chứng minh bằng thực nghiệm: một cuộc gọi HTTP nội bộ không được instrument đúng cách
      sẽ phá vỡ chuỗi trace, và cách khắc phục
- [ ] Đọc hiểu định dạng header W3C `traceparent` mà Micrometer Tracing dùng để truyền ngữ cảnh
      trace giữa các service

## Prerequisites

- Phase 8, Chapter 01 — correlation ID gắn vào MDC, nền tảng để so sánh với `traceId`.
- Phase 8, Chapter 02 — Micrometer là hạ tầng chung cho cả metrics và tracing.

## Used Later

- **Chapter 06 (Kafka)**, **Chapter 07 (Messaging)** — trace context cần được truyền tay qua
  message header khi giao tiếp bất đồng bộ, không tự động như HTTP.

## Problem

`correlationId` (Chapter 01) là một chuỗi **tự đặt**, chỉ có ý nghĩa nếu chính ứng dụng gán và
truyền tay nó đi khắp nơi — bao gồm cả sang service khác. Trong một hệ thống nhiều service, mỗi
service có thể tự sinh `correlationId` riêng cho request nó nhận, dẫn tới log của cùng một luồng
xử lý nghiệp vụ nhưng bị gắn nhiều correlation ID khác nhau ở mỗi service, không thể nối lại thành
một bức tranh hoàn chỉnh nếu không có cơ chế **chuẩn hoá và tự động lan truyền**.

## Concept

**Distributed tracing** giải quyết đúng vấn đề đó bằng hai khái niệm chuẩn hoá:

- **Trace** — đại diện cho **toàn bộ hành trình** của một request, xuyên suốt mọi service nó đi
  qua, định danh bằng một `traceId` duy nhất.
- **Span** — đại diện cho **một đơn vị công việc cụ thể** trong hành trình đó (ví dụ: "Controller
  A xử lý", "gọi HTTP sang Service B", "query database"), mỗi span có `spanId` riêng nhưng dùng
  chung `traceId` của toàn bộ trace, và biết span cha của mình (parent span).

Micrometer Tracing (dùng Brave hoặc OpenTelemetry làm bridge) tự động tạo span cho các thao tác
phổ biến (nhận HTTP request, gọi HTTP request đi, query JDBC...) và **tự động truyền `traceId`**
sang service tiếp theo qua HTTP header — miễn là client HTTP được dùng đúng cách (xem Deep Dive).

## Why?

Vì `correlationId` tự đặt không có cơ chế lan truyền chuẩn hoá, hai service độc lập không có cách
nào biết chúng đang xử lý "cùng một request logic" trừ khi tự thống nhất một giao thức truyền tay.
Distributed tracing giải quyết vấn đề này ở tầng hạ tầng: `traceId` được sinh một lần ở điểm vào
hệ thống (edge service) và **tự động đính kèm** vào mọi request đi ra tiếp theo thông qua
instrumentation chuẩn của framework — không cần code nghiệp vụ tự truyền tay như `correlationId`.

## How?

```java
@RestController
public class TracingDemoController {
    private final RestTemplate restTemplate;

    public TracingDemoController(RestTemplateBuilder builder) {
        this.restTemplate = builder.build(); // QUAN TRONG: dung Builder, khong new RestTemplate()
    }

    @GetMapping("/api/tracing-demo/order")
    public String createOrder() {
        log.info("Bat dau xu ly don hang (span goc)");
        String result = restTemplate.getForObject(
            "http://localhost:8080/api/tracing-demo/payment", String.class);
        return "order-done + " + result;
    }
}
```

```properties
management.tracing.sampling.probability=1.0
```

```xml
<!-- logback-spring.xml -->
<pattern>... [traceId=%X{traceId},spanId=%X{spanId}] ...%msg%n</pattern>
```

## Visualization

```
Trace (traceId=6a631464...):
  Span goc  (spanId=b84ab4cf..) : GET /api/tracing-demo/order      [http-nio-8080-exec-1]
    └─ Span con (spanId=a0bb57..): GET /api/tracing-demo/payment   [http-nio-8080-exec-2]
                                    (spanId cha = b84ab4cf..)

Header HTTP truyen di kem request con:
  traceparent: 00-6a631464...-a0bb57...-01
               │  │            │         └ flags (sampled)
               │  │            └ spanId cua span nay
               │  └ traceId (GIONG span cha)
               └ version
```

## Example

**Trace hoạt động đúng — dùng `RestTemplateBuilder` (auto-instrumented):**

```bash
curl http://localhost:8080/api/tracing-demo/order
```

Log thật:

```
[http-nio-8080-exec-1] [traceId=6a6314641ffd61ebff7aae9dfcbf612b,spanId=b84ab4cf79c922b9] [correlationId=0c8b88a2] Bat dau xu ly don hang (span goc)
[http-nio-8080-exec-2] [traceId=6a6314641ffd61ebff7aae9dfcbf612b,spanId=a0bb5750010d5fb6] [correlationId=dca210a7] Dang xu ly thanh toan (span con, goi tu order). Header tracing nhan duoc: [traceparent=00-6a6314641ffd61ebff7aae9dfcbf612b-fc3bc6de1fb53214-01]
[http-nio-8080-exec-1] [traceId=6a6314641ffd61ebff7aae9dfcbf612b,spanId=b84ab4cf79c922b9] [correlationId=0c8b88a2] Don hang hoan tat, ket qua thanh toan: payment-ok
```

Ba điều xác nhận được bằng dữ liệu thật:

1. **`traceId` giống hệt nhau** ở cả hai span (`6a6314641ffd61ebff7aae9dfcbf612b`) — dù chạy trên
   hai thread khác nhau (`exec-1` và `exec-2`), Micrometer Tracing tự nối chúng vào cùng một trace.
2. **`spanId` khác nhau** ở mỗi span — mỗi đơn vị xử lý có định danh riêng.
3. Request nội bộ mang theo header **`traceparent`** (chuẩn W3C Trace Context) với định dạng
   `00-{traceId}-{spanId của span cha}-{flags}` — chính là cơ chế Micrometer Tracing dùng để "báo"
   cho service nhận biết nó thuộc trace nào và span cha là ai.
4. **`correlationId` (Chapter 01) vẫn khác nhau** ở mỗi hop (`0c8b88a2` rồi `dca210a7`) — vì
   `CorrelationIdFilter` tự viết không biết gì về tracing, nó vẫn tạo correlation ID mới cho mỗi
   HTTP request độc lập. `traceId` mới là thứ thực sự nối liền hai request thành một luồng.

**Trace bị gãy — dùng `new RestTemplate()` tự khởi tạo (KHÔNG được instrument):**

```
[http-nio-8080-exec-1] [traceId=6a6313f833522927d32a06c255bae710] Bat dau xu ly don hang (span goc)
[http-nio-8080-exec-2] [traceId=6a6313f8ec95914f7bb8bb1c30d0df4d] Dang xu ly thanh toan (span con, goi tu order)
```

Hai `traceId` khác nhau — request sang `/payment` không mang theo bất kỳ header trace nào, nên
service nhận (dù là cùng một ứng dụng) tự sinh một trace **hoàn toàn mới**, không có liên kết nào
với trace gốc.

## Deep Dive

**Vì sao `new RestTemplate()` không được instrument, còn `RestTemplateBuilder` thì có?** Spring
Boot tự động cấu hình một `RestTemplateBuilder` bean (`RestTemplateAutoConfiguration`) — khi
`micrometer-tracing-bridge-brave` có mặt trên classpath, Spring Boot đăng ký thêm một
`ObservationRestTemplateCustomizer` áp dụng cho **mọi `RestTemplate` được tạo qua builder này**,
tự động thêm một `ClientHttpRequestInterceptor` chèn header `traceparent` vào mọi request đi ra.
Khi tự gọi `new RestTemplate()`, đối tượng này **không đi qua bất kỳ customizer nào** của Spring
Boot — nó là một `RestTemplate` "trần", không có interceptor nào được gắn, nên không có cách nào
để framework "chèn thêm" hành vi instrumentation vào sau khi đối tượng đã được tạo bằng `new`. Đây
chính là lý do tổng quát vì sao Spring Boot luôn khuyến nghị lấy các client HTTP/DB qua
auto-configured builder/bean thay vì tự `new` — không chỉ tracing, mà cả metrics
(`http.client.requests`, Chapter 02) cũng bị mất theo cùng cơ chế.

## Engineering Insight

**`traceparent` là gì, và vì sao nó là một chuẩn W3C chứ không phải thứ riêng của Spring?** Dữ
liệu thật ở Example cho thấy định dạng `00-6a6314641ffd61ebff7aae9dfcbf612b-fc3bc6de1fb53214-01`
gồm 4 phần: `version-traceId-parentSpanId-flags`. Đây là chuẩn **W3C Trace Context**
(`traceparent` header), được thiết kế để nhiều hệ thống tracing khác nhau (Zipkin, Jaeger, AWS
X-Ray, Datadog) có thể **liên thông** với nhau mà không cần biết implementation cụ thể của bên
kia — miễn cả hai đều hiểu định dạng header chuẩn này. Trước W3C Trace Context, hệ sinh thái Java
chủ yếu dùng định dạng **B3** (`X-B3-TraceId`, `X-B3-SpanId`) do Zipkin định nghĩa — Micrometer
Tracing (qua Brave) hỗ trợ cả hai, có thể cấu hình qua `management.tracing.propagation.type`,
nhưng mặc định ở Spring Boot 3.x là W3C.

## Historical Note

Distributed tracing xuất phát từ paper **Dapper** của Google (2010), mô tả hệ thống tracing nội bộ
cho hạ tầng quy mô lớn. Ý tưởng này ảnh hưởng trực tiếp tới Zipkin (Twitter, sinh ra định dạng B3)
và OpenTracing/OpenTelemetry sau này. Spring Cloud Sleuth (thế hệ trước của Micrometer Tracing,
đã ngừng phát triển từ Spring Boot 3) từng là lựa chọn mặc định cho tracing trong hệ sinh thái
Spring — Micrometer Tracing kế thừa vai trò đó nhưng tách biệt rõ ràng khỏi Spring Cloud, theo
đúng triết lý facade chung với Micrometer Metrics (Chapter 02).

## Myth vs Reality

- **Myth:** "Chỉ cần thêm dependency `micrometer-tracing-bridge-brave` là mọi HTTP call tự động
  được trace đúng."
  **Reality:** Đã chứng minh bằng thực nghiệm — chỉ các bean HTTP client được Spring Boot
  auto-configure (qua `RestTemplateBuilder`, `WebClient.Builder`) mới tự động instrument;
  `new RestTemplate()` tự khởi tạo hoàn toàn nằm ngoài cơ chế này.
- **Myth:** "correlationId (Chapter 01) và traceId là một khái niệm, chỉ khác tên."
  **Reality:** correlationId là chuỗi tự đặt theo request HTTP riêng lẻ (khác nhau mỗi hop, như đã
  thấy ở Example); traceId được framework tự sinh và tự lan truyền xuyên suốt nhiều service/hop.

## Common Mistakes

- **Tự khởi tạo HTTP client bằng `new RestTemplate()`/`new OkHttpClient()`** thay vì lấy qua bean
  auto-configured — mất cả tracing lẫn metrics tự động (`http.client.requests`).
- **Nhầm lẫn traceId với correlationId tự viết** — dẫn tới việc duy trì hai hệ thống định danh
  song song không cần thiết, trong khi traceId đã đủ để nối log/trace xuyên service.
- **Không đặt `management.tracing.sampling.probability`** — mặc định Spring Boot chỉ sample 10%
  request (`0.1`), khiến phần lớn trace "biến mất" trong môi trường dev/test khi cố gắng debug một
  request cụ thể.

## Best Practices

- Luôn lấy HTTP client qua bean auto-configured (`RestTemplateBuilder`, `WebClient.Builder`), để
  tận dụng tự động cả tracing và metrics.
- Đặt `management.tracing.sampling.probability=1.0` ở môi trường dev/staging để debug dễ dàng;
  giảm xuống một tỉ lệ nhỏ (ví dụ `0.1`) ở production để tránh overhead.
- Kết hợp `traceId` vào log pattern (như Chapter 01) để có thể nhảy từ một dòng log bất thường
  sang toàn bộ trace liên quan trên hệ thống tracing backend (Zipkin/Jaeger).
- Với giao tiếp bất đồng bộ (message queue, Chapter 06/07), chủ động đọc/ghi `traceparent` vào
  message header vì instrumentation tự động không bao phủ hết mọi loại giao thức.

## Debug Checklist

- [ ] Trace của một luồng xử lý nhiều service bị "gãy" thành nhiều trace rời rạc? → nghi ngờ hàng
      đầu là HTTP client không được auto-configure đúng cách — kiểm tra có `new RestTemplate()`
      hay tương đương ở đâu đó trong đường đi.
- [ ] Không thấy trace nào trong hệ thống tracing backend dù ứng dụng có traffic? → kiểm tra
      `management.tracing.sampling.probability`, mặc định chỉ 10%.
- [ ] `traceId` xuất hiện trong log nhưng rỗng (`traceId=`)? → request đó chưa đi qua một span nào
      (thường là log phát sinh trong giai đoạn khởi động ứng dụng, ngoài vòng đời request).

## Summary

Distributed tracing dùng `traceId` (xuyên suốt toàn bộ luồng xử lý qua nhiều service) và `spanId`
(một đơn vị công việc cụ thể) để giải quyết vấn đề mà `correlationId` tự viết (Chapter 01) không
làm được: liên kết log của nhiều service thành một bức tranh hoàn chỉnh, tự động, không cần truyền
tay. Đã chứng minh bằng thực nghiệm: hai HTTP call nội bộ dùng `RestTemplateBuilder` (auto-
instrumented) giữ đúng `traceId` xuyên suốt, mang theo header chuẩn W3C `traceparent`; cùng kịch
bản đó nhưng dùng `new RestTemplate()` tự khởi tạo làm gãy trace thành hai trace độc lập hoàn
toàn, vì đối tượng tự tạo không đi qua cơ chế auto-configuration của Spring Boot.

## Interview Questions

**Mid**

- Sự khác biệt giữa trace và span trong distributed tracing?
- Sự khác biệt giữa `correlationId` tự viết (Chapter 01) và `traceId` do Micrometer Tracing quản
  lý?

**Senior**

- Vì sao `new RestTemplate()` tự khởi tạo không được instrument tracing, dù dependency
  `micrometer-tracing-bridge-brave` đã có trên classpath? Cách khắc phục?
- `traceparent` header (W3C Trace Context) có cấu trúc như thế nào? So sánh với định dạng B3 cũ
  của Zipkin.

## Exercises

- [ ] Đổi `TracingDemoController` sang dùng `new RestTemplate()`, gọi lại `/api/tracing-demo/order`
      và xác nhận `traceId` của hai span khác nhau — tái hiện đúng lỗi ở Story.
- [ ] Đặt `management.tracing.sampling.probability=0.0`, gọi API, xác nhận `traceId`/`spanId`
      trong log trở thành rỗng do request không được sample.
- [ ] Thêm một endpoint thứ ba, gọi tuần tự order → payment → shipping, xác nhận cả ba span dùng
      chung một `traceId`.

## Cheat Sheet

| | correlationId (Chapter 01) | traceId (Chapter 03) |
| --- | --- | --- |
| Ai sinh ra? | Code tự viết (`CorrelationIdFilter`) | Micrometer Tracing tự động |
| Lan truyền sang service khác? | Chỉ khi tự truyền tay qua header | Tự động qua `traceparent` (nếu client được instrument) |
| Phạm vi | Một request HTTP đơn lẻ | Toàn bộ luồng xử lý, nhiều service/hop |

| HTTP client | Được instrument tự động? |
| --- | --- |
| `new RestTemplate()` | Không |
| `restTemplateBuilder.build()` | Có |
| `WebClient.builder().build()` (auto-configured builder) | Có |

## References

- Micrometer Tracing Documentation.
- W3C Trace Context Specification — `traceparent` header.
- Google Dapper Paper (2010) — nguồn gốc distributed tracing hiện đại.
- Spring Boot Documentation — `RestTemplateAutoConfiguration`, Observability.
