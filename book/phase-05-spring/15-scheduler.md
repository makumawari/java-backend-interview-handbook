---
tags:
  - Spring
  - Scheduler
---

# @Scheduled

> Phase: Phase 5 — Spring
> Chapter slug: `scheduler`

## Metadata

```yaml
Chapter: Scheduler
Phase: Phase 5 — Spring
Difficulty: ★★★
Importance: ★★★
Interview Frequency: 55%
Prerequisites:
  - Chapter 14 — Async
Used Later:
  - Kiến trúc triển khai nhiều instance (Phase 8, microservice/container hoá)
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Một job `@Scheduled` chạy báo cáo hằng ngày hoạt động hoàn hảo suốt quá trình phát triển —
> chạy local, một tiến trình duy nhất, đúng một lần mỗi khi tới giờ. Sau khi triển khai lên
> production với **2 instance** (để đảm bảo tính sẵn sàng cao — nếu một instance chết, instance
> còn lại vẫn phục vụ được), báo cáo đột nhiên được gửi **hai lần** mỗi ngày.

Mô phỏng đúng tình huống này: 2 `ApplicationContext` độc lập (đại diện cho 2 instance của cùng
một ứng dụng), cùng bật `@Scheduled` cho cùng một job:

```
=== Mo phong 2 'instance' cua CUNG mot ung dung (2 ApplicationContext rieng biet) ===
  [Instance pool-2-thread-1] DailyReportJob chay lan thu 1
  [Instance pool-3-thread-1] DailyReportJob chay lan thu 2
  [Instance pool-2-thread-1] DailyReportJob chay lan thu 3
  [Instance pool-3-thread-1] DailyReportJob chay lan thu 4
  ...
Tong so lan job chay (2 instance CUNG chay job, khong ai biet ve nhau): 8
```

Cả hai instance đều chạy job đúng theo lịch của **chính nó**, hoàn toàn không biết tới sự tồn
tại của instance còn lại — tổng số lần thực thi **nhân đôi** so với dự kiến.

## Interview Question (Central)

> A scheduled job runs only once locally but executes multiple times after deploying multiple
> instances. Why?

## Objectives

- [ ] Hiểu `@Scheduled` chỉ hoạt động trong phạm vi **một JVM/instance đơn lẻ**, không có nhận
      thức gì về các instance khác
- [ ] Tự tay chứng minh bằng thực nghiệm: 2 instance độc lập cùng chạy một job theo lịch, tổng số
      lần thực thi nhân đôi
- [ ] Biết các giải pháp thực tế: distributed lock (ShedLock), hoặc chỉ định một instance duy
      nhất chịu trách nhiệm (leader election)

## Prerequisites

- Chapter 14 — hiểu `@Async`, nền tảng liên quan tới việc lập lịch chạy task nền không chặn
  thread chính.

## Used Later

- **Kiến trúc triển khai nhiều instance** (Phase 8) — vấn đề "duplicate execution" là một trong
  những cân nhắc kiến trúc quan trọng nhất khi scale một ứng dụng theo chiều ngang
  (horizontal scaling).

## Problem

`@Scheduled` được thiết kế để hoạt động **hoàn toàn độc lập trong phạm vi một JVM** — nó không
có bất kỳ cơ chế nào để biết "có bao nhiêu bản sao khác của cùng ứng dụng đang chạy" hay "đã có
ai chạy job này chưa". Khi một ứng dụng chỉ chạy **một instance duy nhất** (phổ biến trong phát
triển và các hệ thống nhỏ), điều này không gây vấn đề gì. Nhưng khi triển khai với **nhiều
instance** (chuẩn mực cho production nghiêm túc, đảm bảo tính sẵn sàng cao) — mỗi instance đều tự
tin "tôi là người duy nhất chạy job này", dẫn tới thực thi trùng lặp không mong muốn.

## Concept

**`@Scheduled`** đánh dấu một method sẽ được gọi tự động theo lịch (`fixedRate`, `fixedDelay`,
hoặc biểu thức `cron`) — do một `TaskScheduler` nội bộ của **chính JVM đó** quản lý, hoàn toàn
tách biệt khỏi bất kỳ JVM/instance nào khác của cùng ứng dụng. Với ứng dụng chạy nhiều instance,
cần một cơ chế **bổ sung, độc lập với Spring** để đảm bảo chỉ một instance thực sự thực thi job
tại một thời điểm — phổ biến nhất là **distributed lock** (khoá phân tán, ví dụ dùng database
hoặc Redis làm nơi lưu trạng thái khoá dùng chung giữa các instance).

## Why?

Bản chất `@Scheduled` chỉ là một `TaskScheduler` chạy trong bộ nhớ của một tiến trình JVM đơn lẻ
— nó không giao tiếp qua mạng, không biết gì về các tiến trình khác. Đây là thiết kế **có chủ
đích đơn giản** (không cố gắng giải quyết bài toán phân tán phức tạp ngay trong lõi framework) —
Spring để việc điều phối giữa nhiều instance cho các công cụ chuyên biệt hơn (thư viện distributed
lock, hoặc hạ tầng orchestration như Kubernetes CronJob chỉ chạy đúng một pod). Hiểu rõ ranh giới
này giúp tránh ngộ nhận rằng `@Scheduled` "tự động" xử lý đúng trong môi trường nhiều instance.

## How?

```java
@EnableScheduling // BAT BUOC, dat tren @Configuration class
class SchedulerConfig {}

@Component
class DailyReportJob {
    @Scheduled(fixedRate = 200) // chay moi 200ms (thuc te thuong dung cron: "0 0 8 * * *")
    void runReport() { ... }
}
```

## Visualization

```
1 instance (KHONG van de):

  Instance duy nhat ──► @Scheduled chay dung 1 lan moi khi toi lich

Nhieu instance (VAN DE: thuc thi TRUNG LAP):

  Instance 1 (JVM rieng) ──► TaskScheduler rieng ──► chay job theo lich CUA NO
  Instance 2 (JVM rieng) ──► TaskScheduler rieng ──► chay job theo lich CUA NO
       (2 instance HOAN TOAN KHONG BIET ve nhau)
  => Tong so lan thuc thi = 2x du kien (hoac hon, tuy so instance)

Giai phap: Distributed Lock (VD: ShedLock)

  Instance 1 ──► xin khoa (database/Redis) ──► GIANH duoc ──► chay job
  Instance 2 ──► xin khoa (database/Redis) ──► KHONG gianh duoc ──► BO QUA lan nay
```

## Example

```java
@Component
static class DailyReportJob {
    @Scheduled(fixedRate = 200)
    void runReport() {
        System.out.println("DailyReportJob chay tren " + this);
    }
}

@Configuration
@EnableScheduling
static class AppConfig {
    @Bean DailyReportJob job() { return new DailyReportJob(); }
}
```

Mô phỏng 2 instance bằng 2 `ApplicationContext` độc lập trong cùng JVM test (mỗi context đại
diện cho một tiến trình ứng dụng độc lập):

```java
try (var instance1 = new AnnotationConfigApplicationContext(AppConfig.class);
     var instance2 = new AnnotationConfigApplicationContext(AppConfig.class)) {
    Thread.sleep(700);
}
```

Kết quả thật (Spring Framework 6.1.13):

```
[Instance pool-2-thread-1] DailyReportJob chay lan thu 1 (tren ...@609640d5)
[Instance pool-3-thread-1] DailyReportJob chay lan thu 2 (tren ...@9bd0fa6)
[Instance pool-2-thread-1] DailyReportJob chay lan thu 3 (tren ...@609640d5)
[Instance pool-3-thread-1] DailyReportJob chay lan thu 4 (tren ...@9bd0fa6)
...
Tong so lan job chay: 8
```

Hai object `DailyReportJob` khác nhau hoàn toàn (`@609640d5` và `@9bd0fa6` — hai instance riêng
biệt, mỗi cái thuộc một `ApplicationContext`/pool thread riêng: `pool-2-thread-1` và
`pool-3-thread-1`) đều tự chạy job theo đúng lịch `fixedRate = 200`. Trong ~700ms, tổng cộng **8
lần thực thi** — gấp đôi so với kỳ vọng nếu chỉ có một instance duy nhất (dự kiến khoảng 4 lần).

## Deep Dive

**Distributed lock (ví dụ thư viện ShedLock) giải quyết bài toán này như thế nào?** Nguyên lý
chung: trước khi thực thi job, mỗi instance phải "xin" một khoá được lưu ở một nơi **dùng chung**
giữa mọi instance (một bảng trong database, hoặc Redis) — thao tác "xin khoá" này bản thân nó
phải **nguyên tử** (atomic, tương tự khái niệm CAS đã học ở Phase 3, Chapter 12-13) để đảm bảo
chỉ đúng một instance giành được khoá tại một thời điểm, dù nhiều instance cùng thử "xin" gần như
đồng thời. Instance giành được khoá mới thực sự chạy job; các instance còn lại phát hiện khoá đã
bị giữ, **bỏ qua** lần thực thi đó. Khoá thường có thời gian hết hạn (lockAtMostFor) để tránh
trường hợp instance giữ khoá bị crash giữa chừng, khiến khoá "mắc kẹt" vĩnh viễn không ai giành
lại được.

## Engineering Insight

**Vì sao vấn đề "duplicate execution" không chỉ là vấn đề hiệu năng (chạy thừa), mà có thể là
vấn đề nghiêm trọng về tính đúng đắn nghiệp vụ?** Với một job **idempotent** (chạy nhiều lần cho
cùng kết quả, ví dụ tính toán lại một báo cáo tổng hợp từ dữ liệu hiện có), thực thi trùng lặp
chỉ lãng phí tài nguyên. Nhưng với một job **không idempotent** (ví dụ gửi email thông báo, trừ
tiền định kỳ, tạo bản ghi mới mỗi lần chạy), thực thi trùng lặp gây hậu quả nghiêm trọng: khách
hàng nhận **hai email** giống hệt nhau, bị trừ tiền **hai lần**, hoặc có **hai bản ghi trùng lặp**
trong database. Đây là lý do khi thiết kế bất kỳ `@Scheduled` job nào dự định chạy trong môi
trường nhiều instance, câu hỏi đầu tiên cần trả lời là: "job này có idempotent không, và nếu
không, tôi có cơ chế nào đảm bảo nó chỉ chạy đúng một lần chưa?" — trước khi nghĩ tới việc tối ưu
lịch trình hay hiệu năng.

## Historical Note

`@Scheduled` xuất hiện từ Spring 3.0 (2009), cùng đợt với `@Async` (Chapter 14) — cả hai đều
thuộc module `spring-context` xử lý task execution/scheduling. Vấn đề "duplicate execution" trở
nên phổ biến và được thảo luận nhiều hơn hẳn từ khi kiến trúc microservice và container hoá
(Kubernetes) trở thành chuẩn mực triển khai — trước đó, nhiều ứng dụng monolithic chỉ chạy một
instance duy nhất, khiến vấn đề này ít khi lộ diện trong thực tế vận hành.

## Myth vs Reality

- **Myth:** "`@Scheduled` tự động đảm bảo job chỉ chạy một lần trên toàn hệ thống, bất kể có bao
  nhiêu instance."
  **Reality:** Đã chứng minh bằng thực nghiệm — mỗi instance hoàn toàn độc lập, không có sự phối
  hợp nào giữa chúng; cần cơ chế bổ sung (distributed lock) để đảm bảo điều này.

- **Myth:** "Vấn đề duplicate execution chỉ là lãng phí tài nguyên nhỏ, không đáng lo ngại."
  **Reality:** Xem Engineering Insight — với job không idempotent, hậu quả có thể nghiêm trọng
  về mặt nghiệp vụ (gửi email trùng, trừ tiền hai lần).

## Common Mistakes

- **Triển khai `@Scheduled` job không idempotent lên môi trường nhiều instance mà không có
  distributed lock** — nguồn gốc phổ biến của lỗi "trùng lặp" khó tái hiện (chỉ xảy ra khi có từ
  2 instance chạy đồng thời, không xảy ra khi test local với 1 instance).
- **Nhầm lẫn `@Scheduled` với một cơ chế phân tán có sẵn** — nó chỉ hoạt động đúng trong phạm vi
  một JVM, không có nhận thức gì về instance khác.
- **Dùng distributed lock nhưng không đặt thời gian hết hạn hợp lý** — nếu instance giữ khoá
  crash giữa chừng mà khoá không tự hết hạn, không instance nào khác có thể chạy job đó nữa.

## Best Practices

- Với mọi `@Scheduled` job dự định chạy ở môi trường nhiều instance, đánh giá tính idempotent
  trước tiên.
- Dùng thư viện distributed lock chuyên dụng (ví dụ ShedLock, tích hợp sẵn với database/Redis)
  thay vì tự viết cơ chế khoá phân tán thủ công.
- Cân nhắc kiến trúc thay thế: giao trách nhiệm lập lịch cho hạ tầng orchestration (Kubernetes
  CronJob chỉ tạo đúng một pod cho mỗi lần chạy) thay vì để mỗi instance ứng dụng tự lập lịch.

## Debug Checklist

- [ ] Job chạy trùng lặp chỉ khi triển khai production (nhiều instance), không tái hiện được khi
      test local? → xác nhận nghi ngờ đây chính là vấn đề multiple-instance duplicate execution.
- [ ] Cần xác nhận có bao nhiêu instance đang chạy một job cụ thể? → kiểm tra log của từng
      instance riêng biệt (theo hostname/pod name) xem job có chạy đồng thời ở nhiều nơi không.

## Summary

`@Scheduled` chỉ hoạt động trong phạm vi một JVM/instance đơn lẻ, hoàn toàn không có nhận thức
về các instance khác của cùng ứng dụng. Đã chứng minh bằng thực nghiệm: 2 instance độc lập
(2 `ApplicationContext`) cùng chạy một job theo `fixedRate`, tổng số lần thực thi **nhân đôi** so
với dự kiến (8 lần thay vì ~4 lần trong cùng khoảng thời gian) — đúng nguyên nhân của câu hỏi
phỏng vấn kinh điển "job chạy một lần local, nhiều lần khi deploy nhiều instance". Giải pháp
chuẩn là distributed lock (ví dụ ShedLock) đảm bảo chỉ một instance giành được quyền chạy tại
một thời điểm. Với job không idempotent, thực thi trùng lặp có thể gây hậu quả nghiêm trọng về
nghiệp vụ (email trùng, trừ tiền hai lần), không chỉ là lãng phí tài nguyên.

## Interview Questions

- A scheduled job runs only once locally but executes multiple times after deploying multiple
  instances. Why?

**Senior**

- Đề xuất giải pháp để đảm bảo một `@Scheduled` job chỉ chạy đúng một lần khi ứng dụng triển
  khai với nhiều instance.
- Vì sao cần đặt thời gian hết hạn cho distributed lock? Điều gì xảy ra nếu không có?

## Exercises

- [ ] Chạy lại `MultiInstanceSchedulerTest` ở trên, xác nhận tổng số lần thực thi gấp đôi khi có
      2 "instance" so với chỉ 1.
- [ ] Tìm hiểu thư viện ShedLock, tích hợp vào ví dụ trên để đảm bảo chỉ một instance chạy job
      tại một thời điểm.
- [ ] Thiết kế một `@Scheduled` job idempotent (ví dụ: tính lại tổng doanh thu từ dữ liệu hiện
      có) so với một job không idempotent (gửi email nhắc nhở) — thảo luận rủi ro khác nhau giữa
      hai loại khi có duplicate execution.

## Cheat Sheet

| | 1 instance | Nhiều instance (không distributed lock) | Nhiều instance (có distributed lock) |
| --- | --- | --- | --- |
| Số lần job chạy mỗi lịch | 1 | N (bằng số instance) | 1 (chỉ instance giành khoá) |
| Job idempotent | An toàn | Lãng phí tài nguyên | An toàn, tối ưu |
| Job KHÔNG idempotent | An toàn | **Nguy hiểm** (email trùng, trừ tiền 2 lần) | An toàn |

## References

- Spring Framework Documentation — Task Execution and Scheduling: https://docs.spring.io/spring-framework/reference/integration/scheduling.html
- ShedLock — https://github.com/lukas-krecan/ShedLock
