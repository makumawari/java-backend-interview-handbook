---
tags:
  - Metrics
  - Micrometer
  - Observability
---

# Metrics: Micrometer và Actuator

> Phase: Phase 8 — Production
> Chapter slug: `metrics`

## Metadata

```yaml
Chapter: Metrics
Phase: Phase 8 — Production
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 55%
Prerequisites:
  - Phase 8, Chapter 01 — Logging
  - Phase 6, Chapter 04 — Multi-Datasource (HikariCP xuất hiện trong metrics)
Used Later:
  - Chapter 03 — Tracing
  - Chapter 04 — HikariCP
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

Một `Gauge` được đăng ký để theo dõi độ dài hàng đợi xử lý đơn hàng:

```java
public MetricsDemoController(MeterRegistry registry) {
    AtomicInteger queueSize = new AtomicInteger(0); // bien local!
    registry.gauge("demo.orders.queue.size", queueSize);
}
```

Gọi API tạo đơn hàng vài lần rồi kiểm tra giá trị Gauge thật qua Actuator:

```bash
curl -s http://localhost:8080/actuator/metrics/demo.orders.queue.size
```

Kết quả thật:

```json
{
    "name": "demo.orders.queue.size",
    "measurements": [ { "statistic": "VALUE", "value": "NaN" } ],
    "availableTags": []
}
```

`NaN` — dù `AtomicInteger` khởi tạo bằng `0`, chưa từng bị set về giá trị bất thường nào.
`Counter` và `Timer` đăng ký cùng constructor này hoạt động hoàn toàn bình thường. Vấn đề nằm ở
chỗ `queueSize` chỉ là **biến local**, không được gán vào field của class — và `Gauge` của
Micrometer **không giữ tham chiếu mạnh (strong reference)** tới state object nó theo dõi.

## Interview Question (Central)

> Sự khác biệt giữa Counter, Gauge, và Timer trong Micrometer là gì? Vì sao một `Gauge` được
> đăng ký đúng cú pháp vẫn có thể âm thầm trả về `NaN` trong production?

## Objectives

- [ ] Phân biệt đúng ba loại metric cơ bản của Micrometer: Counter, Gauge, Timer — và biết khi
      nào dùng loại nào
- [ ] Đọc hiểu metric tự động (auto-instrumented) mà Spring Boot Actuator cung cấp sẵn, đặc biệt
      `http.server.requests`
- [ ] Tự tay tái hiện lỗi Gauge trả về `NaN` do state object bị Garbage Collector thu hồi, và biết
      cách khắc phục
- [ ] Đọc hiểu định dạng Prometheus exposition format mà Micrometer xuất ra

## Prerequisites

- Phase 8, Chapter 01 — correlation ID và mối quan hệ giữa logging/metrics/tracing.
- Phase 6, Chapter 04 — HikariCP connection pool, nguồn gốc của nhóm metric `hikaricp.*`.

## Used Later

- **Chapter 03 (Tracing)** — metrics trả lời "cái gì/bao nhiêu", tracing trả lời "ở đâu/tại sao
  chậm"; cả hai cùng dùng chung hạ tầng Micrometer.
- **Chapter 04 (HikariCP)** — debug connection pool dựa trực tiếp vào nhóm metric
  `hikaricp.connections.*` đã thấy ở đây.

## Problem

Log (Chapter 01) trả lời câu hỏi "chuyện gì đã xảy ra với **một** request cụ thể". Nhưng khi cần
trả lời "hệ thống đang khoẻ không **nói chung**" — có bao nhiêu request/giây, độ trễ trung bình là
bao nhiêu, connection pool có đang cạn kiệt không — đọc từng dòng log là bất khả thi ở quy mô
production. Cần một cơ chế tổng hợp **số liệu theo thời gian** (metrics) mà không cần đọc log chi
tiết.

## Concept

**Micrometer** là facade đo lường (giống vai trò SLF4J với logging) mà Spring Boot Actuator dùng
để thu thập số liệu, hỗ trợ nhiều backend export (Prometheus, Datadog, CloudWatch...). Ba loại
metric cốt lõi:

- **Counter** — số đếm chỉ **tăng**, không bao giờ giảm (ví dụ: tổng số đơn hàng đã tạo).
- **Gauge** — giá trị **tức thời** tại thời điểm đọc, có thể tăng/giảm (ví dụ: số connection đang
  active, độ dài hàng đợi hiện tại).
- **Timer** — đo **số lần xảy ra + tổng thời gian + giá trị lớn nhất** của một thao tác có thời
  lượng (ví dụ: thời gian xử lý một đơn hàng).

Spring Boot Actuator tự động đăng ký hàng chục metric (`jvm.*`, `http.server.requests`,
`hikaricp.*`, `executor.*`, `tomcat.*`) mà không cần viết thêm code.

## Why?

Vì Counter/Gauge/Timer có ngữ nghĩa toán học khác nhau, việc chọn sai loại tạo ra số liệu vô
nghĩa: dùng Counter cho "số connection đang active" sẽ chỉ tăng mãi (sai — connection có thể đóng
lại), dùng Gauge cho "tổng số request" sẽ mất dữ liệu khi giá trị bị ghi đè giữa hai lần đọc (sai
— cần cộng dồn). Actuator tự động instrument các thành phần hạ tầng (Tomcat, HikariCP, JVM) vì
những con số này **luôn cần thiết** ở mọi ứng dụng Spring Boot, không có lý do gì bắt mỗi team tự
viết lại.

## How?

```java
@RestController
public class MetricsDemoController {
    private final Counter orderCreatedCounter;
    private final Timer processingTimer;

    public MetricsDemoController(MeterRegistry registry) {
        this.orderCreatedCounter = Counter.builder("demo.orders.created")
            .description("So don hang da tao")
            .tag("channel", "web")
            .register(registry);
        this.processingTimer = Timer.builder("demo.orders.processing.time")
            .description("Thoi gian xu ly don hang")
            .register(registry);
    }

    @GetMapping("/api/metrics-demo/create-order")
    public String createOrder() {
        orderCreatedCounter.increment();
        processingTimer.record(() -> { try { Thread.sleep(50); } catch (InterruptedException ignored) {} });
        return "order created, counter = " + orderCreatedCounter.count();
    }
}
```

Đọc metric qua Actuator: `GET /actuator/metrics/{tên metric}`, hoặc toàn bộ ở định dạng Prometheus
qua `GET /actuator/prometheus`.

## Visualization

```
Counter (chi tang):        0 -> 1 -> 2 -> 3 -> 3 -> 4       (khong bao gio giam)

Gauge (tuc thoi):          5 -> 3 -> 8 -> 1 -> 0             (len xuong tu do)

Timer (moi lan goi ghi 2 con so):
  goi 1: count=1, total=0.050s, max=0.050s
  goi 2: count=2, total=0.098s, max=0.050s
  goi 3: count=3, total=0.178s, max=0.060s   <- max cap nhat khi co gia tri lon hon
```

## Example

**Danh sách toàn bộ metric Actuator tự động đăng ký (thật, Spring Boot 3.3.4):**

```json
{
  "names": [
    "executor.active", "executor.completed", "executor.pool.core", "executor.pool.max",
    "hikaricp.connections", "hikaricp.connections.active", "hikaricp.connections.idle",
    "hikaricp.connections.max", "hikaricp.connections.pending", "hikaricp.connections.usage",
    "http.server.requests", "http.server.requests.active",
    "jvm.memory.used", "jvm.gc.pause", "jvm.threads.live",
    "tomcat.sessions.active.current",
    "demo.orders.created", "demo.orders.processing.time", "demo.orders.queue.size"
  ]
}
```

`demo.orders.*` — ba metric tự viết — nằm chung danh sách với các metric hạ tầng tự động, không
cần cấu hình gì thêm ngoài `.register(registry)`.

**`http.server.requests` thật, sau nhiều request thật tới nhiều endpoint khác nhau:**

```bash
curl http://localhost:8080/api/log-demo -H "X-Correlation-Id: metrics-evidence-1"
curl http://localhost:8080/api/log-demo -H "X-Correlation-Id: metrics-evidence-2"
curl "http://localhost:8080/actuator/metrics/http.server.requests"
```

```json
{
    "name": "http.server.requests",
    "baseUnit": "seconds",
    "measurements": [
        { "statistic": "COUNT", "value": 10.0 },
        { "statistic": "TOTAL_TIME", "value": 0.288220583 },
        { "statistic": "MAX", "value": 0.020365292 }
    ],
    "availableTags": [
        { "tag": "method", "values": ["GET"] },
        { "tag": "uri", "values": [
            "/actuator/metrics/{requiredMetricName}", "/api/log-demo",
            "/api/metrics-demo/create-order", "/actuator/metrics", "/actuator/prometheus" ] },
        { "tag": "status", "values": ["200"] },
        { "tag": "outcome", "values": ["SUCCESS"] }
    ]
}
```

Lọc riêng theo một `uri` cụ thể bằng query param `tag`:

```bash
curl "http://localhost:8080/actuator/metrics/http.server.requests?tag=uri:/api/log-demo"
```

```json
{
    "measurements": [
        { "statistic": "COUNT", "value": 2.0 },
        { "statistic": "TOTAL_TIME", "value": 0.00547025 },
        { "statistic": "MAX", "value": 0.003293208 }
    ]
}
```

`http.server.requests` là **một Timer duy nhất** cho toàn bộ ứng dụng, với các **tag** (`uri`,
`method`, `status`, `outcome`) cho phép lọc/nhóm theo chiều bất kỳ — không phải một metric riêng
cho mỗi endpoint.

**Counter và Timer tự viết, dữ liệu thật sau 3 lần gọi:**

```bash
curl http://localhost:8080/api/metrics-demo/create-order   # x3
curl "http://localhost:8080/actuator/metrics/demo.orders.created"
curl "http://localhost:8080/actuator/metrics/demo.orders.processing.time"
```

```json
{
    "name": "demo.orders.created",
    "measurements": [ { "statistic": "COUNT", "value": 3.0 } ],
    "availableTags": [ { "tag": "channel", "values": ["web"] } ]
}
```

```json
{
    "name": "demo.orders.processing.time",
    "baseUnit": "seconds",
    "measurements": [
        { "statistic": "COUNT", "value": 3.0 },
        { "statistic": "TOTAL_TIME", "value": 0.177543707 },
        { "statistic": "MAX", "value": 0.060049625 }
    ]
}
```

`processingTimer.record(...)` bọc quanh `Thread.sleep(50)` — kết quả thật xác nhận đúng: 3 lần
gọi, tổng thời gian ~0.177s (~59ms/lần, hợp lý với `sleep(50)` cộng overhead), max ~0.06s.

**Cùng dữ liệu đó, ở định dạng Prometheus exposition (`/actuator/prometheus`):**

```
# HELP demo_orders_total So don hang da tao
# TYPE demo_orders_total counter
demo_orders_total{channel="web"} 3.0
# HELP demo_orders_processing_time_seconds Thoi gian xu ly don hang
# TYPE demo_orders_processing_time_seconds summary
demo_orders_processing_time_seconds_count 3
demo_orders_processing_time_seconds_sum 0.177543707
# TYPE demo_orders_processing_time_seconds_max gauge
demo_orders_processing_time_seconds_max 0.0
```

Chú ý: `demo.orders.created` (Counter) tự động đổi tên thành `demo_orders_total` — Micrometer
**tự thêm hậu tố `_total`** cho Counter theo đúng quy ước đặt tên của Prometheus.

**Lỗi Gauge trả về `NaN` — tái hiện thật (đã mô tả ở Story):**

```json
{
    "name": "demo.orders.queue.size",
    "measurements": [ { "statistic": "VALUE", "value": "NaN" } ]
}
```

## Deep Dive

**Vì sao Gauge trả về `NaN` dù `AtomicInteger` khởi tạo đúng bằng `0`?** `MeterRegistry.gauge(...)`
chỉ giữ một **weak reference** tới state object được truyền vào — đây là thiết kế **cố ý** của
Micrometer để Gauge không gây rò rỉ bộ nhớ (nếu Gauge giữ strong reference, object nó theo dõi sẽ
không bao giờ được GC dọn, kể cả khi phần còn lại của ứng dụng không còn dùng tới nó nữa). Trong
`MetricsDemoController`, `queueSize` là **biến local trong constructor**, không được gán vào field
— sau khi constructor kết thúc, không còn bất kỳ strong reference nào khác trỏ tới nó ngoài chính
Gauge (weak). Garbage Collector thu hồi nó ở lần GC tiếp theo, và khi Actuator đọc giá trị Gauge,
Micrometer phát hiện state object đã bị GC và trả về `NaN` thay vì ném exception. Đây không phải
lỗi lý thuyết — nó đã xảy ra thật trong chính demo ở Story.

## Engineering Insight

**Sửa lỗi Gauge `NaN` như thế nào, và tại sao cách sửa lại chính là "vi phạm" nguyên tắc thiết kế
ban đầu?** Cách sửa đúng là đảm bảo state object được giữ **strong reference** ở nơi khác — thường
là gán vào **field của class** đăng ký Gauge:

```java
private final AtomicInteger queueSize = new AtomicInteger(0); // FIELD, khong phai local

public MetricsDemoController(MeterRegistry registry) {
    registry.gauge("demo.orders.queue.size", queueSize);
}
```

Giờ `queueSize` được giữ sống bởi chính instance của `MetricsDemoController` (một Spring bean,
sống suốt vòng đời ứng dụng), nên Gauge luôn đọc được giá trị thật. Bài học rộng hơn: **API dùng
weak reference luôn đặt gánh nặng "giữ đối tượng sống" lên người gọi** — một lỗi rất dễ mắc phải
vì code biên dịch hoàn toàn bình thường, không có warning, và chỉ lộ ra khi GC chạy (có thể vài
giây, vài phút sau khi ứng dụng khởi động), khiến việc debug trở nên khó hiểu nếu không biết trước
cơ chế này.

## Historical Note

Micrometer ra đời năm 2018, tách ra từ Spring Boot 1.x's metrics module (`spring-boot-actuator`
cũ dùng `CounterService`/`GaugeService` riêng, gắn chặt với Spring). Micrometer được thiết kế lại
theo mô hình facade giống SLF4J — cho phép cùng một bộ code instrument xuất ra nhiều backend khác
nhau (Prometheus, Datadog, New Relic, CloudWatch) chỉ bằng cách đổi dependency registry, không đổi
code nghiệp vụ.

## Myth vs Reality

- **Myth:** "Đăng ký một Gauge đúng cú pháp là đủ để nó luôn phản ánh giá trị thật."
  **Reality:** Đã chứng minh bằng thực nghiệm — Gauge dùng weak reference; nếu state object không
  được giữ sống ở nơi khác, nó bị GC thu hồi và Gauge âm thầm trả về `NaN`, không có exception hay
  warning nào.
- **Myth:** "`http.server.requests` là một metric riêng cho mỗi endpoint."
  **Reality:** Đã xác nhận — đó là **một** Timer duy nhất cho toàn ứng dụng, phân biệt theo
  endpoint bằng tag `uri`, lọc bằng query param `?tag=uri:...`.

## Common Mistakes

- **Đăng ký Gauge với state object là biến local, không gán vào field** — gây lỗi `NaN` sau khi
  GC chạy, như đã tái hiện ở Story/Deep Dive.
- **Dùng Counter cho giá trị có thể giảm** (ví dụ số connection đang mở) — ngữ nghĩa sai, nên dùng
  Gauge.
- **Tạo Counter/Timer mới trong mỗi lần gọi method thay vì đăng ký một lần** — mỗi lần
  `Counter.builder(...).register(registry)` được gọi lại sẽ trả về cùng một Counter đã đăng ký
  (Micrometer dedupe theo tên+tag), nhưng tạo lại builder liên tục trong hot path vẫn tốn overhead
  không cần thiết — nên đăng ký một lần và giữ tham chiếu (như `orderCreatedCounter` ở trên).

## Best Practices

- Luôn gán state object của Gauge vào **field**, không phải biến local — tránh lỗi `NaN` do GC.
- Đăng ký Counter/Timer/Gauge **một lần** (constructor hoặc `@PostConstruct`), giữ tham chiếu làm
  field, không tạo lại mỗi request.
- Dùng tag (`channel`, `status`...) để phân nhóm thay vì tạo nhiều metric riêng biệt cho từng biến
  thể — giữ số lượng "tên metric" gốc nhỏ, tận dụng khả năng lọc/nhóm theo tag của Prometheus.
- Đặt tên metric theo quy ước `noun.noun` (ví dụ `demo.orders.created`), để Micrometer tự dịch
  đúng sang định dạng backend (Prometheus tự thêm `_total` cho Counter).

## Debug Checklist

- [ ] Một Gauge trả về `NaN` hoặc giá trị không đổi bất thường? → nghi ngờ hàng đầu là state
      object bị GC do chỉ được giữ bởi weak reference — kiểm tra nó có phải field hay không.
- [ ] Cần biết độ trễ/tỉ lệ lỗi của một endpoint cụ thể? → dùng
      `/actuator/metrics/http.server.requests?tag=uri:...`, không cần tự viết Timer riêng.
- [ ] Cần xuất số liệu cho Prometheus scrape? → xác nhận dependency
      `micrometer-registry-prometheus` có trong classpath và endpoint `/actuator/prometheus` khả
      dụng.

## Summary

Micrometer cung cấp ba loại metric cốt lõi — Counter (chỉ tăng), Gauge (tức thời, lên xuống tự
do), Timer (đếm số lần + tổng thời gian + max) — và Spring Boot Actuator tự động đăng ký hàng chục
metric hạ tầng (`http.server.requests`, `hikaricp.*`, `jvm.*`) mà không cần cấu hình. Đã chứng
minh bằng thực nghiệm: `http.server.requests` là một Timer duy nhất phân biệt theo tag `uri`, không
phải nhiều metric riêng; Counter tự viết được Prometheus tự thêm hậu tố `_total`. Đã tái hiện một
lỗi thật: Gauge đăng ký với state object là biến local (không giữ strong reference ở nơi khác) bị
Garbage Collector thu hồi, khiến `/actuator/metrics/...` trả về `NaN` — sửa bằng cách gán state
object vào field của class.

## Interview Questions

**Mid**

- Sự khác biệt giữa Counter, Gauge, và Timer trong Micrometer là gì?
- Actuator tự động cung cấp những nhóm metric nào mà không cần viết thêm code?

**Senior**

- Vì sao một Gauge đăng ký đúng cú pháp vẫn có thể trả về `NaN`? Cách khắc phục?
- `http.server.requests` được tổ chức thành một metric hay nhiều metric? Làm sao lọc theo một
  endpoint cụ thể?

## Exercises

- [ ] Viết một Gauge với state object là biến local (không gán field), gọi `System.gc()` rồi kiểm
      tra `/actuator/metrics/...` — xác nhận tái hiện được `NaN`, sau đó sửa bằng cách gán field.
- [ ] Gọi `/actuator/metrics/http.server.requests` với `?tag=status:200` và `?tag=method:GET` để
      xác nhận cơ chế lọc theo tag hoạt động độc lập với `uri`.
- [ ] So sánh cùng một Counter đọc qua `/actuator/metrics/{name}` (JSON) và `/actuator/prometheus`
      (text) — xác nhận tên metric bị đổi (`demo.orders.created` → `demo_orders_total`).

## Cheat Sheet

| Loại metric | Có thể giảm? | Ví dụ | Endpoint đọc |
| --- | --- | --- | --- |
| Counter | Không, chỉ tăng | `demo.orders.created` | `/actuator/metrics/{name}` |
| Gauge | Có | `hikaricp.connections.active` | `/actuator/metrics/{name}` |
| Timer | N/A (đo thời lượng) | `http.server.requests` | `/actuator/metrics/{name}` |

| | Local variable | Field |
| --- | --- | --- |
| Gauge state object | Bị GC → `NaN` | Giữ sống → đúng giá trị |

## References

- Micrometer Documentation — Concepts (Counter, Gauge, Timer).
- Micrometer Documentation — Gauges (weak reference behavior).
- Spring Boot Actuator Documentation — Metrics.
- Prometheus Documentation — Exposition Formats.
