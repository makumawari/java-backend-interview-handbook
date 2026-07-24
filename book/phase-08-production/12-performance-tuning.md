---
tags:
  - Performance Tuning
  - Lock Contention
  - Thread Dump
---

# Performance Tuning: Điều tra Độ trễ Production

> Phase: Phase 8 — Production
> Chapter slug: `performance-tuning`

## Metadata

```yaml
Chapter: Performance Tuning
Phase: Phase 8 — Production
Difficulty: ★★★★★
Importance: ★★★★★
Interview Frequency: 80%
Prerequisites:
  - Phase 3, Chapter 05 — synchronized
  - Phase 8, Chapter 01 — Logging
  - Phase 8, Chapter 02 — Metrics
  - Phase 8, Chapter 04 — HikariCP
  - Phase 8, Chapter 11 — JVM Tuning
Estimated Reading: 18 phút
Estimated Practice: 15 phút
```

## Story

Một endpoint, test đơn lẻ cục bộ, phản hồi nhanh:

```bash
curl http://localhost:8080/api/perf-demo/slow
```

```
slow-done in 305ms, counter=1
```

Gửi 25 request **đồng thời** (mô phỏng traffic thật ở production) tới đúng endpoint đó:

```bash
for i in $(seq 1 25); do curl -o /dev/null -w "%{time_total}s " http://localhost:8080/api/perf-demo/slow & done; wait
```

Kết quả thật:

```
0.36s 0.66s 0.96s 1.27s 1.57s 1.88s 2.18s 2.48s 2.79s 3.09s 3.40s 3.70s 4.00s
4.30s 4.59s 4.89s 5.20s 5.50s 5.80s 6.10s 6.41s 6.71s 7.01s 7.32s 7.63s
```

Request cuối cùng mất **7.63 giây** — gần đúng với triệu chứng kinh điển "300ms local, 8 giây
production". Cùng một endpoint, cùng một đoạn code, **không hề chậm hơn về mặt tính toán** — vấn đề
chỉ xuất hiện khi có nhiều request cùng lúc. Lấy thread dump thật ngay giữa lúc burst 15 request
đồng thời:

```bash
jstack <pid> > threaddump.txt
grep -c "BLOCKED (on object monitor)" threaddump.txt
```

```
10
```

**10 thread** đang ở trạng thái `BLOCKED`, tất cả cùng chờ khoá **một object monitor duy nhất**,
tại đúng một dòng code:

```
"http-nio-8080-exec-3" ... java.lang.Thread.State: BLOCKED (on object monitor)
    at com.handbook.demo.logging.PerfDemoController.slowEndpoint(PerfDemoController.java:21)
    - waiting to lock <0x000000070227ceb0> (a java.lang.Object)
```

Nguyên nhân: một khối `synchronized` bọc quanh **toàn bộ** logic xử lý (bao gồm cả
`Thread.sleep(300)` mô phỏng công việc), khiến 300ms "công việc" của mỗi request buộc phải chạy
**tuần tự**, không phải song song — độ trễ tích luỹ tuyến tính theo số request đồng thời.

## Interview Question (Central)

> An API is suddenly taking 8 seconds in production, but only 300ms locally. Where do you start
> debugging?

## Objectives

- [ ] Tự tay tái hiện chính xác kịch bản "nhanh khi test đơn lẻ, chậm dần dưới tải đồng thời" bằng
      một lock contention thật
- [ ] Dùng `jstack` để lấy thread dump thật, đọc đúng trạng thái `BLOCKED` và xác định chính xác
      dòng code gây tranh chấp
- [ ] Áp dụng đúng quy trình điều tra hệ thống, tận dụng mọi công cụ đã học trong Phase 8
      (logging, metrics, tracing, connection pool) theo đúng thứ tự ưu tiên

## Prerequisites

- Phase 3, Chapter 05 — `synchronized`, monitor, nền tảng để hiểu trạng thái `BLOCKED`.
- Phase 8, Chapter 01, 02, 04, 11 — toàn bộ công cụ quan sát/chẩn đoán đã xây dựng xuyên suốt
  phase này, được tổng hợp lại thành một quy trình điều tra có hệ thống ở chapter này.

## Used Later

Đây là chapter cuối cùng của Phase 8 — kiến thức được dùng xuyên suốt mọi phase sau trong công
việc thực tế.

## Problem

"Chậm ở production nhưng nhanh khi test cục bộ" là một trong những triệu chứng khó chịu nhất trong
vận hành hệ thống thực tế — nó **không tái hiện được** bằng cách chạy lại đúng request đó một lần
duy nhất trên máy cá nhân, vì bản chất vấn đề không nằm ở logic tính toán của request, mà nằm ở
**tương tác giữa nhiều request chạy đồng thời** — một điều kiện chỉ xuất hiện dưới tải thật.

## Concept

Khi nhiều thread cùng cố giữ một **monitor lock** (`synchronized`), chỉ một thread được vào tại một
thời điểm — các thread còn lại chuyển sang trạng thái **`BLOCKED`**, xếp hàng chờ. Nếu công việc
bên trong khối khoá có **thời lượng cố định** (như `Thread.sleep(300)`), tổng thời gian một thread
phải chờ tỉ lệ **tuyến tính** với số thread đang xếp hàng trước nó — request thứ N phải chờ
khoảng `(N-1) × 300ms` trước khi được xử lý, cộng thêm 300ms xử lý chính nó.

## Why?

Vì mục đích của `synchronized` là bảo vệ **trạng thái chia sẻ** (shared mutable state) khỏi truy
cập đồng thời không an toàn, nó **cố ý** buộc các thread chạy tuần tự qua đoạn code được bảo vệ.
Vấn đề chỉ nảy sinh khi phạm vi khoá (critical section) **rộng hơn mức cần thiết** — bọc luôn cả
phần công việc tốn thời gian mà đáng lẽ không cần bảo vệ, khiến toàn bộ hệ thống mất khả năng xử lý
song song một cách không cần thiết, biến một endpoint vốn có thể chạy hoàn toàn độc lập giữa các
request thành một hàng đợi tuần tự trá hình.

## How?

```java
// TRUOC: toan bo cong viec nam trong synchronized -> tranh chap khong can thiet
@GetMapping("/api/perf-demo/slow")
public String slowEndpoint() throws InterruptedException {
    synchronized (LOCK) {
        Thread.sleep(300); // "cong viec" khong can bao ve, nhung van bi khoa
        slowCounter++;
    }
    return "...";
}

// SAU: chi phan thuc su can dong bo (tang counter) moi dung synchronized/Atomic
@GetMapping("/api/perf-demo/fast")
public String fastEndpoint() throws InterruptedException {
    Thread.sleep(300);              // chay SONG SONG binh thuong
    long value = fastCounter.incrementAndGet(); // AtomicLong, khong can lock rieng
    return "...";
}
```

```bash
jstack <pid> | grep -A3 "BLOCKED (on object monitor)"
```

## Visualization

```
25 request dong thoi, /slow (synchronized bao ca cong viec):

  Req 1: [====300ms====]
  Req 2:                [====300ms====]         <- phai CHO Req 1 xong
  Req 3:                               [====300ms====]  <- cho ca 2 request truoc
  ...
  Req 25:                                                        [====300ms====]
                                                                  ↑ ~7.6s sau khi bat dau

25 request dong thoi, /fast (khong khoa toan cuc):

  Req 1:  [====300ms====]
  Req 2:  [====300ms====]     <- chay SONG SONG that
  Req 3:  [====300ms====]
  ...
  Req 25: [====300ms====]
                          ↑ tat ca hoan tat trong ~1s (gioi han boi thread pool, khong phai lock)
```

## Example

**So sánh trực tiếp — cùng 25 request đồng thời, một bên có lock toàn cục, một bên không:**

| Endpoint | Request nhanh nhất | Request chậm nhất |
| --- | --- | --- |
| `/api/perf-demo/slow` (có `synchronized` bao công việc) | 0.36s | **7.63s** |
| `/api/perf-demo/fast` (không khoá, `AtomicLong`) | 0.39s | **1.04s** |

**Thread dump thật, chụp giữa lúc 15 request đồng thời đang chạy:**

```
"http-nio-8080-exec-2" ... java.lang.Thread.State: BLOCKED (on object monitor)
    at com.handbook.demo.logging.PerfDemoController.slowEndpoint(PerfDemoController.java:21)
    - waiting to lock <0x000000070227ceb0> (a java.lang.Object)

"http-nio-8080-exec-3" ... java.lang.Thread.State: BLOCKED (on object monitor)
    at com.handbook.demo.logging.PerfDemoController.slowEndpoint(PerfDemoController.java:21)
    - waiting to lock <0x000000070227ceb0> (a java.lang.Object)

"http-nio-8080-exec-7" ... java.lang.Thread.State: TIMED_WAITING (sleeping)
    at java.lang.Thread.sleep(java.base@17.0.12/Native Method)
    at com.handbook.demo.logging.PerfDemoController.slowEndpoint(PerfDemoController.java:21)
    - locked <0x000000070227ceb0> (a java.lang.Object)
```

10 thread `BLOCKED`, tất cả `waiting to lock` **đúng cùng một địa chỉ monitor**
(`<0x000000070227ceb0>`), tại **đúng một dòng code** (`PerfDemoController.java:21`) — bằng chứng
trực tiếp, không cần suy đoán, về nguyên nhân chính xác của độ trễ.

## Deep Dive

**Vì sao độ trễ tăng gần như tuyến tính hoàn hảo (0.3s một bậc, như thấy trong dữ liệu thật ở
Story) thay vì tăng đột biến hay hỗn loạn?** Vì tất cả 25 request có **cùng thời lượng công việc cố
định** (300ms) và cùng tranh chấp **một** monitor duy nhất, hàng đợi chờ khoá hoạt động giống hệt
một hàng đợi FIFO đơn giản: request thứ N chỉ có thể bắt đầu phần việc của nó sau khi (N-1) request
trước hoàn tất — tạo ra cấp số cộng chính xác với bước nhảy 300ms, đúng như log thật thể hiện
(0.36s, 0.66s, 0.96s...). Trong hệ thống thực tế phức tạp hơn (thời lượng công việc mỗi request
khác nhau, nhiều loại lock khác nhau tương tác), độ trễ vẫn tăng theo tải nhưng ít khi đều đặn như
vậy — sự đều đặn ở đây là dấu hiệu đặc trưng của **một** điểm tranh chấp lock duy nhất, đơn giản.

## Engineering Insight

**Quy trình điều tra thực tế đúng thứ tự khi gặp "nhanh local, chậm production" — tổng hợp mọi
công cụ đã học trong Phase 8 — là gì?**

1. **Xác nhận đây là vấn đề do tải (concurrency), không phải do dữ liệu khác nhau** — chạy lại
   chính request đó với dữ liệu production thật trên máy local; nếu vẫn nhanh, loại trừ nguyên nhân
   do input, tập trung vào hành vi dưới tải đồng thời.
2. **Kiểm tra `http.server.requests`** (Chapter 02) theo `uri` cụ thể — xác nhận độ trễ tăng có
   tương quan với thời điểm traffic cao hay không.
3. **Kiểm tra `hikaricp.connections.*`** (Chapter 04) — loại trừ khả năng nghẽn cổ chai ở tầng
   connection pool database trước khi nghi ngờ tới lock ở tầng ứng dụng.
4. **Lấy thread dump thật** (`jstack`, như Story) khi hệ thống đang chậm — đây là bằng chứng
   **trực tiếp nhất**, cho biết chính xác thread đang chờ gì, ở dòng code nào, không cần suy đoán.
   Nếu thấy nhiều thread `BLOCKED` cùng chờ một monitor, đã tìm ra nguyên nhân gốc.
5. **Kiểm tra log GC** (Chapter 11) nếu thread dump không cho thấy lock contention rõ ràng — độ
   trễ đôi khi tới từ GC pause dồn dập dưới tải cao, không phải lock.
6. **Sửa bằng cách thu hẹp phạm vi khoá** — chỉ bảo vệ đúng phần thao tác trên trạng thái chia sẻ
   (như `fastEndpoint` ở How?), không bọc cả phần công việc độc lập.

Thứ tự này ưu tiên các nguồn dữ liệu **rẻ và nhanh** (metrics) trước, chỉ dùng công cụ **tốn kém
hơn nhưng chính xác tuyệt đối** (thread dump) khi các bước trước đã thu hẹp phạm vi nghi vấn.

## Historical Note

`jstack` (và cơ chế thread dump nói chung, có thể lấy qua tín hiệu `SIGQUIT`/`kill -3` từ những
phiên bản JVM rất sớm) là một trong những công cụ chẩn đoán lâu đời nhất của nền tảng Java — tồn
tại từ trước cả khi có JVisualVM, JFR (Java Flight Recorder, JDK 11+ miễn phí), hay các APM
thương mại hiện đại (Datadog, New Relic). Dù công cụ giám sát đã phát triển rất nhiều, thread dump
vẫn là kỹ thuật **nhanh nhất và trực tiếp nhất** để trả lời câu hỏi "ngay bây giờ, các thread đang
làm gì" — không cần cài đặt agent, không cần cấu hình trước, chỉ cần quyền truy cập vào tiến trình.

## Myth vs Reality

- **Myth:** "Nếu code chạy nhanh khi test một request duy nhất, code đó không có vấn đề về hiệu
  năng."
  **Reality:** Đã chứng minh bằng thực nghiệm — đúng cùng đoạn code, cùng 305ms khi chạy đơn lẻ,
  nhưng tạo ra độ trễ 7.6 giây dưới 25 request đồng thời. Test đơn lẻ hoàn toàn không phát hiện
  được lock contention.
- **Myth:** "Độ trễ cao ở production luôn do database chậm hoặc network chậm."
  **Reality:** Đã chứng minh bằng thực nghiệm — nguyên nhân ở đây hoàn toàn nằm trong tầng ứng
  dụng (một khối `synchronized` quá rộng), không liên quan gì tới database hay network.

## Common Mistakes

- **Bọc `synchronized`/lock quanh cả những thao tác không cần đồng bộ** (I/O, `Thread.sleep`, gọi
  service khác) — biến một đoạn code có thể song song thành hàng đợi tuần tự, như đã tái hiện.
- **Chỉ test hiệu năng bằng một request đơn lẻ** trước khi triển khai — không phát hiện được các
  vấn đề chỉ xuất hiện dưới tải đồng thời như lock contention.
- **Bỏ qua thread dump, chỉ dựa vào suy đoán** khi gặp độ trễ bất thường — thread dump cho bằng
  chứng trực tiếp, chính xác hơn nhiều so với phỏng đoán dựa trên đọc code tĩnh.

## Best Practices

- Giữ phạm vi `synchronized`/lock **càng hẹp càng tốt** — chỉ bảo vệ đúng thao tác đọc/ghi trạng
  thái chia sẻ, không bao gồm I/O hay tính toán độc lập.
- Ưu tiên cấu trúc dữ liệu concurrent có sẵn (`AtomicLong`, `ConcurrentHashMap` — Phase 2/3) thay
  vì tự viết `synchronized` thủ công khi có thể.
- Luôn test hiệu năng dưới **tải đồng thời thật** (dù chỉ mô phỏng bằng vài chục request song song
  như ở Story) trước khi coi một endpoint là "đủ nhanh", không chỉ test một request đơn lẻ.
- Thiết lập sẵn quy trình lấy thread dump nhanh (`jstack <pid>`) như một phản xạ đầu tiên khi gặp
  độ trễ bất thường ở production, trước khi thử các giả thuyết phức tạp hơn.

## Debug Checklist

- [ ] Endpoint nhanh khi test đơn lẻ nhưng chậm ở production? → nghi ngờ hàng đầu là vấn đề chỉ
      xuất hiện dưới tải đồng thời (lock contention, connection pool cạn kiệt — Chapter 04, GC
      pause dồn dập — Chapter 11), không phải logic tính toán.
- [ ] Nghi ngờ lock contention? → lấy thread dump thật (`jstack`) ngay lúc hệ thống đang chậm, tìm
      nhiều thread `BLOCKED` cùng chờ một địa chỉ monitor.
- [ ] Đã xác định đúng dòng code gây tranh chấp? → kiểm tra phạm vi khối `synchronized` có bao gồm
      thao tác không cần đồng bộ hay không (I/O, sleep, gọi service ngoài).

## Summary

"Nhanh local, chậm production" (300ms → 8 giây) là triệu chứng kinh điển của một vấn đề **chỉ xuất
hiện dưới tải đồng thời**, không nằm trong logic tính toán của bản thân request — đã tái hiện chính
xác bằng thực nghiệm: một khối `synchronized` bọc cả công việc không cần đồng bộ khiến 25 request
đồng thời phải xếp hàng tuần tự, độ trễ request cuối leo tới 7.63 giây, trong khi cùng công việc
không có lock toàn cục chỉ mất 1.04 giây dưới cùng tải. Thread dump thật (`jstack`) là công cụ chẩn
đoán trực tiếp và chính xác nhất cho lớp vấn đề này — 10 thread `BLOCKED` cùng chờ một địa chỉ
monitor, tại đúng một dòng code, xác định nguyên nhân ngay lập tức. Quy trình điều tra hệ thống nên
đi từ công cụ rẻ (metrics — Chapter 02, connection pool — Chapter 04) tới công cụ chính xác nhưng
tốn kém hơn (thread dump), tổng hợp toàn bộ công cụ quan sát đã xây dựng xuyên suốt Phase 8.

## Interview Questions

**Mid**

- Trạng thái `BLOCKED` của một thread trong thread dump nghĩa là gì?
- Vì sao một endpoint nhanh khi test đơn lẻ có thể chậm hẳn dưới tải đồng thời?

**Senior**

- An API is suddenly taking 8 seconds in production, but only 300ms locally. Where do you start
  debugging?
- Đọc một thread dump thật có nhiều thread `BLOCKED`, làm sao xác định chính xác nguyên nhân gốc
  và đề xuất hướng sửa?

## Exercises

- [ ] Sửa `slowEndpoint` theo mẫu `fastEndpoint` (thu hẹp phạm vi lock), lặp lại bài test 25
      request đồng thời, xác nhận độ trễ tối đa giảm từ ~7.6s xuống gần 1s.
- [ ] Lấy thread dump khi hệ thống **không** có tải, xác nhận không còn thread nào ở trạng thái
      `BLOCKED` chờ cùng monitor.
- [ ] Thử với số lượng request đồng thời khác nhau (10, 50, 100), vẽ biểu đồ độ trễ tối đa theo số
      request, xác nhận mối quan hệ tuyến tính đã phân tích ở Deep Dive.

## Cheat Sheet

| Công cụ | Cho biết gì | Chi phí |
| --- | --- | --- |
| Metrics (`http.server.requests`, Chapter 02) | Độ trễ có tăng theo tải không | Rất rẻ, luôn bật |
| `hikaricp.connections.*` (Chapter 04) | Có nghẽn ở connection pool không | Rất rẻ, luôn bật |
| Log GC (Chapter 11) | Có GC pause dồn dập không | Rẻ, luôn bật |
| Thread dump (`jstack`) | Chính xác thread nào đang chờ gì, dòng code nào | Tức thời, cần truy cập tiến trình |

| Trạng thái thread | Ý nghĩa |
| --- | --- |
| `BLOCKED (on object monitor)` | Đang chờ vào một khối `synchronized` |
| `TIMED_WAITING (sleeping)` | Đang `Thread.sleep()` hoặc chờ có thời hạn |
| `RUNNABLE` | Đang thực thi hoặc sẵn sàng thực thi |

## References

- Oracle — `jstack` Documentation.
- OpenJDK — Thread State Definitions (`java.lang.Thread.State`).
- Phase 3, Chapter 05 (`synchronized`) và Chapter 06 (Monitor) — nền tảng lý thuyết.
