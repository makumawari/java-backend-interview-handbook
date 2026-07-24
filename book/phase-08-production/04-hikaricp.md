---
tags:
  - HikariCP
  - Connection Pool
  - Production
---

# HikariCP: Connection Pool Tuning và Debug

> Phase: Phase 8 — Production
> Chapter slug: `hikaricp`

## Metadata

```yaml
Chapter: HikariCP
Phase: Phase 8 — Production
Difficulty: ★★★★
Importance: ★★★★★
Interview Frequency: 70%
Prerequisites:
  - Phase 8, Chapter 02 — Metrics
  - Phase 6, Chapter 04 — Multi-Datasource
Used Later:
  - Chapter 11 — JVM Tuning
  - Chapter 12 — Performance Tuning
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

Một pool HikariCP cấu hình `maximum-pool-size=2`, `connection-timeout=1000`ms. Gửi 3 request
đồng thời, mỗi request giữ connection trong 5 giây:

```bash
for i in 1 2 3; do curl http://localhost:8080/api/hikari-demo/hold & done; wait
```

Kết quả thật:

```
HTTP:200 - held-and-released after 0ms wait
HTTP:200 - held-and-released after 0ms wait
HTTP:200 - FAILED after 1015ms: HikariDemoPool - Connection is not available,
           request timed out after 1010ms (total=2, active=2, idle=0, waiting=0)
```

Hai request đầu lấy connection ngay lập tức (`0ms wait`) vì pool có 2 connection. Request thứ ba
**thất bại** sau đúng 1010ms — bằng `connection-timeout` đã cấu hình — với thông báo lỗi kinh điển
`Connection is not available`, kèm theo trạng thái pool tại thời điểm timeout ngay trong message:
`total=2, active=2, idle=0, waiting=0`.

## Interview Question (Central)

> HikariCP starts throwing "Connection is not available" errors during peak traffic. What would
> you investigate?

## Objectives

- [ ] Tự tay tái hiện lỗi "Connection is not available" bằng cách cố ý làm cạn kiệt pool, đọc
      hiểu từng con số trong thông báo lỗi
- [ ] Dùng `leak-detection-threshold` để tự động phát hiện connection bị giữ quá lâu, kể cả khi nó
      cuối cùng vẫn được trả lại
- [ ] Biết chính xác các bước điều tra khi gặp lỗi này ở production, dựa trên metric
      `hikaricp.connections.*` (Chapter 02)

## Prerequisites

- Phase 8, Chapter 02 — nhóm metric `hikaricp.connections.*` đã thấy trong danh sách Actuator.
- Phase 6, Chapter 04 — cấu hình multi-datasource, nền tảng để hiểu vì sao mỗi `DataSource` có
  một `HikariPool` riêng.

## Used Later

- **Chapter 11 (JVM Tuning)**, **Chapter 12 (Performance Tuning)** — connection pool là một trong
  những "nghi phạm" hàng đầu khi điều tra độ trễ bất thường ở production.

## Problem

Một connection tới database là tài nguyên **đắt** (bắt tay TCP, xác thực, cấp phát bộ nhớ phía
DB) — tạo mới cho mỗi request là quá chậm. Connection pool (HikariCP) giải quyết việc này bằng
cách giữ sẵn một số connection đã mở, tái sử dụng giữa các request. Nhưng pool có **kích thước
giới hạn** — khi số request đồng thời cần connection vượt quá kích thước pool, các request sau
phải **chờ**, và nếu chờ quá lâu, bị từ chối hẳn.

## Concept

**HikariCP** là connection pool mặc định của Spring Boot từ 2.x trở đi (thay thế Tomcat JDBC
Pool), nổi tiếng vì hiệu năng cao và cấu hình tối giản. Các tham số cốt lõi:

- `maximum-pool-size` — số connection tối đa pool giữ.
- `connection-timeout` — thời gian tối đa một thread **chờ** để lấy được connection trước khi bị
  ném exception.
- `leak-detection-threshold` — nếu một connection bị giữ (không trả lại pool) lâu hơn ngưỡng này,
  HikariCP log một **cảnh báo** kèm stack trace nơi connection được lấy ra — dù connection đó cuối
  cùng có được trả lại hay không.

## Why?

Vì connection là tài nguyên hữu hạn, giá trị `maximum-pool-size` quá nhỏ khiến traffic cao gây
nghẽn cổ chai (request phải chờ), nhưng quá lớn lại có thể **áp đảo** chính database (mỗi
connection tốn bộ nhớ/thread phía DB server, và hầu hết database có giới hạn connection tối đa
riêng). `connection-timeout` tồn tại để **thất bại nhanh** thay vì để request treo vô thời hạn khi
pool cạn kiệt — một request lỗi rõ ràng sau 1 giây dễ xử lý hơn nhiều so với một request "treo"
không rõ nguyên nhân trong 30 giây.

## How?

```properties
app.datasource.hikaridemo.maximum-pool-size=2
app.datasource.hikaridemo.connection-timeout=1000
app.datasource.hikaridemo.leak-detection-threshold=2000
```

```java
@RestController
public class HikariDemoController {
    private final DataSource hikariDemoDataSource;

    @GetMapping("/api/hikari-demo/hold")
    public String holdConnection() throws Exception {
        long start = System.currentTimeMillis();
        try (Connection conn = hikariDemoDataSource.getConnection()) { // co the nem exception o day
            Thread.sleep(5000);
            return "held-and-released";
        }
    }
}
```

## Visualization

```
Pool (maximum-pool-size=2):

t=0ms    Request 1 -> lay connection A (active=1, idle=1)
t=0ms    Request 2 -> lay connection B (active=2, idle=0)
t=0ms    Request 3 -> KHONG con connection -> CHO...
t=1010ms Request 3 -> het connection-timeout -> EXCEPTION
                        "Connection is not available ... (total=2, active=2, idle=0, waiting=0)"
t=5000ms Request 1, 2 tra connection ve pool (active=0, idle=2)
```

## Example

**Lỗi cạn kiệt pool — dữ liệu thật, 3 request đồng thời, pool size 2:**

```
FAILED after 1015ms: HikariDemoPool - Connection is not available,
    request timed out after 1010ms (total=2, active=2, idle=0, waiting=0)
```

Đọc đúng các con số trong thông báo lỗi: `total=2` (tổng connection trong pool, bằng
`maximum-pool-size`), `active=2` (đang bị giữ, **bằng total** — pool cạn kiệt hoàn toàn),
`idle=0` (không còn connection rảnh), `waiting=0` (số thread khác đang chờ, tại đúng thời điểm
log dòng này thì request hiện tại đã hết hạn nên không còn "waiting" nữa).

**Leak detection — cùng lúc đó, dữ liệu thật:**

```
WARN c.zaxxer.hikari.pool.ProxyLeakTask - Connection leak detection triggered for conn11:
    url=jdbc:h2:mem:hikari_demo user= on thread http-nio-8080-exec-2, stack trace follows
java.lang.Exception: Apparent connection leak detected
    at com.zaxxer.hikari.HikariDataSource.getConnection(HikariDataSource.java:127)
    at com.handbook.demo.logging.HikariDemoController.holdConnection(HikariDemoController.java:24)
...
INFO c.zaxxer.hikari.pool.ProxyLeakTask - Previously reported leaked connection conn11:
    ... was returned to the pool (unleaked)
```

`leak-detection-threshold=2000`ms nhưng connection bị giữ 5000ms (`Thread.sleep(5000)`) — HikariCP
cảnh báo ở mốc 2000ms kèm **chính xác vị trí code** đã lấy connection ra (rất hữu ích khi debug
connection leak thật trong production, nơi code phức tạp hơn nhiều). Khi connection cuối cùng được
trả lại (đóng đúng cách qua `try-with-resources`), HikariCP tự log tiếp "was returned to the pool
(unleaked)" — xác nhận đây **không phải leak thật**, chỉ là giữ lâu.

## Deep Dive

**Vì sao thông báo lỗi có định dạng chính xác `(total=X, active=Y, idle=Z, waiting=W)` mà không
phải một exception message tối giản?** HikariCP cố tình in kèm **trạng thái pool tại thời điểm
timeout** ngay trong exception message — đây là quyết định thiết kế hướng tới khả năng debug: một
kỹ sư đọc log production, chỉ với một dòng log duy nhất, đã biết ngay được: pool có đang bị cạn
kiệt hoàn toàn không (`active` gần bằng `total`)? Có nhiều thread khác cũng đang bị nghẽn theo
không (`waiting` cao)? Điều này loại bỏ nhu cầu phải tra cứu riêng metric `hikaricp.connections.*`
(Chapter 02) chỉ để xác nhận "pool có đang full không" trong tình huống khẩn cấp.

## Engineering Insight

**Với câu hỏi phỏng vấn trung tâm — "Connection is not available" khi traffic cao — quy trình
điều tra thực tế theo đúng thứ tự ưu tiên là gì?**

1. **Đọc chính exception message** (như Deep Dive) — nếu `active ≈ total`, pool thực sự cạn kiệt;
   nếu `waiting` rất cao, vấn đề là **tốc độ giải phóng** connection chậm hơn tốc độ request đến.
2. **Kiểm tra `hikaricp.connections.usage`** (Timer, Chapter 02) — biết connection thường bị giữ
   bao lâu; nếu thời gian giữ tăng đột biến đúng lúc traffic cao, nghi ngờ query chậm hoặc
   transaction giữ connection quá lâu (N+1, Phase 6 Chapter 10), không phải pool quá nhỏ.
3. **Bật `leak-detection-threshold`** ở mức thấp (như demo: 2000ms) tạm thời — nếu thấy cảnh báo
   "Apparent connection leak" xuất hiện kèm stack trace **luôn trỏ về cùng một vị trí code**, đó
   là dấu hiệu leak thật (thiếu `close()`/`try-with-resources`), không phải do traffic cao đơn
   thuần.
4. **Chỉ tăng `maximum-pool-size` sau khi loại trừ (2) và (3)** — tăng pool size mà không giải
   quyết nguyên nhân gốc (query chậm, leak) chỉ trì hoãn vấn đề, đồng thời có nguy cơ làm quá tải
   chính database (mỗi connection tốn tài nguyên phía server, và hầu hết RDBMS giới hạn tổng số
   connection đồng thời).

## Historical Note

HikariCP ra đời năm 2013, được viết lại từ đầu với mục tiêu tối thiểu hoá overhead so với các
connection pool trước đó (Apache DBCP, C3P0, Tomcat JDBC Pool) — tên "Hikari" (光, tiếng Nhật
nghĩa là "ánh sáng") phản ánh triết lý thiết kế "nhẹ và nhanh". Spring Boot chuyển sang dùng
HikariCP làm connection pool mặc định từ Spring Boot 2.0 (2018), thay thế Tomcat JDBC Pool.

## Myth vs Reality

- **Myth:** "Gặp lỗi 'Connection is not available' thì cứ tăng `maximum-pool-size` là xong."
  **Reality:** Đã phân tích ở Engineering Insight — tăng pool size mà không kiểm tra
  `active`/`waiting` và không loại trừ connection leak có thể chỉ che giấu vấn đề gốc, thậm chí
  làm quá tải database.
- **Myth:** "`leak-detection-threshold` chỉ cảnh báo khi connection thực sự bị mất (không bao giờ
  trả lại)."
  **Reality:** Đã chứng minh bằng thực nghiệm — nó cảnh báo bất kỳ khi nào connection bị giữ lâu
  hơn ngưỡng, kể cả khi sau đó vẫn được trả lại đúng cách (log "was returned to the pool
  (unleaked)" xác nhận điều này).

## Common Mistakes

- **Đặt `maximum-pool-size` quá lớn "cho chắc"** — có thể làm database vượt quá giới hạn connection
  tối đa của chính nó khi nhiều instance ứng dụng cùng chạy.
- **Không set `leak-detection-threshold` ở môi trường staging/dev** — bỏ lỡ cơ hội phát hiện sớm
  connection leak trước khi lên production.
- **Giữ connection mở trong lúc gọi một service/API bên ngoài (network call) thay vì chỉ trong lúc
  chạy query** — kéo dài thời gian giữ connection không cần thiết, dễ gây cạn kiệt pool khi service
  bên ngoài chậm.

## Best Practices

- Luôn dùng `try-with-resources` khi làm việc trực tiếp với `Connection` — đảm bảo trả lại pool kể
  cả khi có exception.
- Đặt `connection-timeout` ở mức đủ ngắn (vài giây) để request thất bại nhanh, không "treo" vô
  thời hạn khi pool cạn kiệt.
- Theo dõi `hikaricp.connections.active` và `hikaricp.connections.pending` (Chapter 02) như một
  chỉ số sức khoẻ chính, đặt alert khi `active` áp sát `maximum-pool-size` trong thời gian dài.
- Công thức khởi điểm phổ biến cho `maximum-pool-size`: `connections = ((core_count * 2) + effective_spindle_count)` (khuyến nghị chính thức của HikariCP wiki) — cần benchmark thực tế, không đoán mò.

## Production Notes

**Problem:** API bắt đầu trả lỗi 500 hàng loạt trong giờ cao điểm, log đầy thông báo
"Connection is not available".

**Symptoms:** Latency tăng đột biến ngay trước khi lỗi xuất hiện; lỗi xảy ra theo cụm (burst),
không rải đều.

**Root Cause (một trong các khả năng, xác nhận theo Engineering Insight):** (a) pool quá nhỏ so
với traffic thực tế, (b) một query/transaction cụ thể trở nên chậm bất thường (thiếu index, lock
contention — Phase 7), giữ connection lâu hơn bình thường, hoặc (c) connection leak thật do thiếu
`close()` ở một nhánh code hiếm khi chạy (ví dụ nhánh xử lý exception).

**Debug:** Đọc exception message để lấy `active`/`waiting`; đối chiếu với
`hikaricp.connections.usage` (thời gian giữ trung bình) trước và trong lúc sự cố; bật tạm
`leak-detection-threshold` thấp nếu nghi ngờ leak.

**Solution:** Tối ưu query/transaction chậm nếu đó là nguyên nhân; sửa leak nếu phát hiện; chỉ
tăng pool size như biện pháp cuối cùng, có benchmark.

**Prevention:** Alert chủ động trên `hikaricp.connections.pending` > 0 kéo dài, thay vì chờ tới
khi connection-timeout xảy ra hàng loạt.

## Debug Checklist

- [ ] Thấy lỗi "Connection is not available"? → đọc ngay `active`/`idle`/`waiting` trong chính
      exception message, đây là bước đầu tiên, nhanh nhất.
- [ ] `active` áp sát `total` liên tục, không chỉ lúc traffic cao? → nghi ngờ connection leak hoặc
      query/transaction chậm bất thường, không phải pool quá nhỏ.
- [ ] Cần xác định vị trí code đang giữ connection quá lâu? → bật `leak-detection-threshold` tạm
      thời, đọc stack trace trong log cảnh báo.

## Summary

HikariCP quản lý một số lượng connection giới hạn (`maximum-pool-size`); khi request đồng thời
vượt quá số connection khả dụng, các request sau phải chờ tới `connection-timeout` rồi thất bại
với lỗi "Connection is not available" — đã tái hiện bằng thực nghiệm với pool size 2 và 3 request
đồng thời, thông báo lỗi thật cho thấy chính xác trạng thái pool (`total=2, active=2, idle=0`) tại
thời điểm timeout. `leak-detection-threshold` cảnh báo khi một connection bị giữ quá lâu, kèm stack
trace nơi nó được lấy ra, và tự xác nhận "unleaked" nếu connection cuối cùng vẫn được trả lại đúng
cách. Điều tra lỗi này ở production nên bắt đầu từ việc đọc exception message và metric
`hikaricp.connections.usage`, chỉ tăng pool size sau khi loại trừ nguyên nhân query chậm/leak.

## Interview Questions

**Mid**

- `maximum-pool-size` và `connection-timeout` của HikariCP có ý nghĩa gì?
- HikariCP starts throwing "Connection is not available" errors during peak traffic. What would
  you investigate?

**Senior**

- `leak-detection-threshold` hoạt động như thế nào? Nó có phân biệt được connection leak thật với
  connection chỉ đơn giản bị giữ lâu không?
- Vì sao tăng `maximum-pool-size` không phải lúc nào cũng là giải pháp đúng khi gặp lỗi "Connection
  is not available"?

## Exercises

- [ ] Chạy lại kịch bản ở Story với `maximum-pool-size=5` thay vì 2, xác nhận cả 3 request đều
      thành công (pool đủ lớn cho traffic đó).
- [ ] Đặt `leak-detection-threshold=500` (thấp hơn cả thời gian request bình thường), xác nhận cả
      những connection dùng bình thường cũng bị cảnh báo (false positive) — hiểu vì sao ngưỡng cần
      đặt hợp lý.
- [ ] So sánh `hikaricp.connections.usage` (Chapter 02) trước và trong lúc chạy kịch bản cạn kiệt
      pool ở Story.

## Cheat Sheet

| Tham số | Ý nghĩa | Hậu quả nếu sai |
| --- | --- | --- |
| `maximum-pool-size` | Số connection tối đa | Quá nhỏ → nghẽn; quá lớn → quá tải DB |
| `connection-timeout` | Thời gian chờ tối đa trước khi thất bại | Quá lớn → request "treo" lâu |
| `leak-detection-threshold` | Ngưỡng cảnh báo giữ connection quá lâu | Không đặt → khó phát hiện leak sớm |

| Trong exception message | Ý nghĩa |
| --- | --- |
| `total` | Tổng connection trong pool (= `maximum-pool-size`) |
| `active` | Đang bị giữ bởi request khác |
| `idle` | Đang rảnh, sẵn sàng cấp phát |
| `waiting` | Số thread khác đang xếp hàng chờ |

## References

- HikariCP GitHub Wiki — "About Pool Sizing".
- HikariCP GitHub Wiki — Configuration (`connectionTimeout`, `leakDetectionThreshold`).
- Spring Boot Documentation — DataSource Configuration.
