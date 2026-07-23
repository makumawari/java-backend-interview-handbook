---
tags:
  - Java
  - notify
  - Concurrency
---

# notify()

> Phase: Phase 3 — Concurrency
> Chapter slug: `notify`

## Metadata

```yaml
Chapter: notify()
Phase: Phase 3 — Concurrency
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 55%
Prerequisites:
  - Chapter 07 — wait()
Used Later:
  - Lock, ReentrantLock (Chapter 09-10) — Condition là phiên bản tường minh, khắc phục hạn chế "một wait set duy nhất" ở đây
Estimated Reading: 20 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 07](07-wait.md) — bạn đã xây dựng producer-consumer với **một** producer,
> **một** consumer, dùng `notifyAll()`.

Thử mở rộng lên **hai** producer và **hai** consumer, và đổi `notifyAll()` thành `notify()`
(nghe có vẻ hợp lý — "chỉ cần đánh thức một thread là đủ" mà). Đo thời gian hoàn thành:

```
Dung notify(): mat 411ms
```

Rồi đổi lại `notifyAll()`:

```
Dung notifyAll(): mat 0ms
```

Cùng logic, cùng dữ liệu — chỉ đổi `notify()` thành `notifyAll()` mà chênh lệch **hàng trăm
lần**. Nguyên nhân: `notify()` đánh thức **một thread ngẫu nhiên** trong wait set — với nhiều
loại thread đang chờ **những điều kiện khác nhau** (producer chờ "còn chỗ trống", consumer chờ
"có dữ liệu"), `notify()` có thể đánh thức nhầm loại, khiến thread đó kiểm tra lại điều kiện
(đúng theo bài học `while` ở Chapter 07), thấy vẫn sai, và quay lại chờ tiếp — lãng phí toàn bộ
"cơ hội" đánh thức đó.

## Interview Question (Central)

> `notify()` và `notifyAll()` khác nhau như thế nào? Khi nào dùng `notify()` là an toàn, khi
> nào bắt buộc phải dùng `notifyAll()`?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích `notify()` đánh thức **một** thread ngẫu nhiên trong wait set;
      `notifyAll()` đánh thức **toàn bộ**
- [ ] Tự tay đo được hậu quả hiệu năng cụ thể khi dùng sai `notify()` với nhiều loại điều kiện
      chờ khác nhau
- [ ] Biết chính xác điều kiện an toàn duy nhất để dùng `notify()`: mọi thread trong wait set
      đều chờ **cùng một điều kiện**, hoàn toàn tương đương nhau
- [ ] Áp dụng đúng nguyên tắc "mặc định dùng `notifyAll()`, chỉ dùng `notify()` khi đã chứng
      minh được an toàn"

## Prerequisites

- Chapter 07 — đã hiểu `wait()`, wait set, và nguyên tắc `while` thay vì `if`.

## Used Later

- **Lock, ReentrantLock** (Chapter 09-10) — `Condition` (gắn với `ReentrantLock`) giải quyết
  triệt để vấn đề "một wait set chung cho mọi loại điều kiện" bằng cách cho phép tạo **nhiều**
  wait set riêng biệt trên cùng một lock — producer và consumer có thể chờ trên hai
  `Condition` khác nhau, khiến `notify()`-tương-đương luôn đánh thức đúng loại thread.

## Problem

`wait()` (Chapter 07) đưa thread vào wait set của Monitor — nhưng một Monitor chỉ có **một**
wait set duy nhất, dùng chung cho **mọi lý do chờ khác nhau**. Khi có nhiều loại thread chờ
những điều kiện khác nhau trên cùng một lock (producer chờ "còn chỗ", consumer chờ "có dữ
liệu"), cần một cách đánh thức **đúng loại** thread cần thiết, không lãng phí việc đánh thức
nhầm loại.

## Concept

- **`notify()`** — đánh thức **đúng một** thread, chọn **ngẫu nhiên** (không xác định trước)
  trong số các thread đang ở wait set của Monitor đó.
- **`notifyAll()`** — đánh thức **toàn bộ** thread đang ở wait set — mỗi thread được đánh thức
  sẽ tự tranh giành lại lock (entry set), rồi tự kiểm tra lại điều kiện của chính nó (đúng
  nguyên tắc `while`, Chapter 07) — thread nào điều kiện chưa đúng sẽ tự `wait()` lại.

## Why?

Nếu chỉ có `notify()`: với **một loại** điều kiện chờ duy nhất (mọi thread chờ giống hệt
nhau), đánh thức một thread ngẫu nhiên là đủ và hiệu quả — không lãng phí đánh thức những thread
không cần thiết. Nhưng với **nhiều loại** điều kiện khác nhau trên cùng lock (Example), `notify()`
có nguy cơ đánh thức nhầm loại, gây "mất" tín hiệu hữu ích — `notifyAll()` giải quyết bằng cách
đánh thức **tất cả**, để chính từng thread tự quyết định (qua kiểm tra `while`) mình có nên
tiếp tục chạy hay quay lại chờ, đảm bảo không "bỏ sót" thread nào thực sự cần được đánh thức.

## How?

```java
// AN TOÀN với notify(): CHỈ khi mọi thread chờ đều TƯƠNG ĐƯƠNG nhau
synchronized (lock) {
    while (!hasWork) lock.wait();
    // ... mọi thread ở đây đều làm CÙNG một việc khi được đánh thức
}
// nơi khác: lock.notify(); // OK — thread nào được đánh thức cũng "đúng"

// KHÔNG AN TOÀN với notify(): có NHIỀU LOẠI điều kiện khác nhau
synchronized (lock) {
    while (buffer.size() == CAPACITY) lock.wait(); // producer chờ "còn chỗ"
}
synchronized (lock) {
    while (buffer.isEmpty()) lock.wait();           // consumer chờ "có dữ liệu"
}
// → PHẢI dùng notifyAll(), vì notify() có thể đánh thức NHẦM LOẠI
```

## Visualization

```
Wait set có CẢ producer (chờ "còn chỗ") LẪN consumer (chờ "có dữ liệu"):

  notify() → đánh thức NGẪU NHIÊN 1 thread
       │
       ├── May mắn: đánh thức ĐÚNG loại cần thiết → xử lý ngay, hiệu quả
       │
       └── Không may: đánh thức NHẦM loại → thread đó kiểm tra while,
                        thấy điều kiện CHƯA đúng → wait() LẠI NGAY
                        → TÍN HIỆU BỊ LÃNG PHÍ, thread ĐÚNG cần được
                          đánh thức vẫn đang chờ (phải chờ notify() TIẾP THEO,
                          hoặc hết timeout — đúng nguyên nhân của 411ms ở Story)

  notifyAll() → đánh thức TẤT CẢ
       │
       └── Mỗi thread tự kiểm tra while của CHÍNH MÌNH:
            đúng loại → tiếp tục chạy
            sai loại → tự wait() lại NGAY (không tốn nhiều chi phí,
                        chỉ là kiểm tra điều kiện rồi quay lại)
           → KHÔNG BAO GIỜ bỏ sót thread cần được đánh thức
```

## Example

Đo thật hậu quả hiệu năng — 2 producer, 2 consumer, buffer dung lượng 1 (dễ xảy ra tranh chấp):

```java
// dùng notify()
lock.notify(); // (thay cho notifyAll() ở cả producer và consumer)
```

Kết quả thật (JDK 17, mỗi `wait()` có timeout 200ms để tránh treo vĩnh viễn khi đo):

```
Dung notify(): mat 411ms
```

Đổi thành `notifyAll()`, giữ nguyên mọi thứ khác:

```java
lock.notifyAll();
```

Kết quả thật:

```
Dung notifyAll(): mat 0ms
```

Chênh lệch không phải "một chút chậm hơn" — nó là bằng chứng trực tiếp cho việc `notify()` đã
nhiều lần đánh thức nhầm loại thread, buộc chương trình phải chờ tới khi **hết timeout** (200ms
mỗi lần) thay vì được đánh thức đúng lúc.

## Deep Dive

**Tại sao không dùng `notify()` "một cách thông minh hơn" — ví dụ chỉ định rõ đánh thức đúng
loại thread nào?** Vì API `notify()`/`notifyAll()` của Monitor (Object, Chapter 06) **không có
khái niệm** phân loại thread trong wait set — chúng chỉ biết "có bao nhiêu thread đang chờ",
không biết "thread nào đang chờ vì lý do gì". Đây chính là hạn chế cố hữu của mô hình Monitor
nguyên thuỷ (một wait set duy nhất cho mọi lý do chờ) — và chính xác là lý do
`java.util.concurrent.locks.Condition` (Chapter 10) ra đời: nó cho phép tạo **nhiều wait set
riêng biệt** (`lock.newCondition()`) trên cùng một `Lock`, để producer chờ trên
`notFull.await()` và consumer chờ trên `notEmpty.await()` — khi đó, `notFull.signal()` **chỉ**
đánh thức đúng producer đang chờ, không bao giờ đánh thức nhầm consumer, giải quyết triệt để
vấn đề chapter này minh hoạ.

## Engineering Insight

**Nguyên tắc thực dụng: "mặc định luôn `notifyAll()`, chỉ dùng `notify()` khi đã chứng minh
được an toàn" — vì sao lời khuyên lại nghiêng hẳn về một phía như vậy?** Vì cái giá của việc
dùng sai `notify()` (một tín hiệu bị lãng phí, dẫn tới delay khó dự đoán, như đã đo ở Example —
có thể còn tệ hơn: **treo vĩnh viễn** nếu không có timeout dự phòng và không có `notify()`/
`notifyAll()` nào khác tới "giải cứu") là **nghiêm trọng hơn nhiều** so với cái giá của việc
dùng thừa `notifyAll()` khi thực ra `notify()` đã đủ (chỉ là một chút overhead: các thread bị
đánh thức "nhầm" tự kiểm tra `while` rồi `wait()` lại ngay — rẻ, không gây hại). Đây là một
ví dụ điển hình của nguyên tắc kỹ thuật "chọn sai lệch về phía an toàn khi cái giá của hai loại
sai lầm chênh lệch rất lớn" — không phải vì `notify()` "tệ", mà vì điều kiện để nó an toàn
(mọi thread chờ hoàn toàn tương đương) khó đảm bảo và dễ vỡ khi code phát triển thêm theo thời
gian (ai đó thêm một loại điều kiện chờ mới mà quên kiểm tra lại toàn bộ giả định cũ).

## Historical Note

`notify()`/`notifyAll()` tồn tại từ Java 1.0 (1996), cùng `wait()`. Vấn đề "một wait set dùng
chung cho mọi điều kiện" mà chapter này minh hoạ là một trong những động lực chính thúc đẩy
JSR 166 (Java 5, 2004) tạo ra `Condition` — một trong nhiều cải tiến `java.util.concurrent`
mang lại nhằm khắc phục các hạn chế cố hữu, đã được nhận diện rõ qua nhiều năm sử dụng thực tế,
của mô hình Monitor nguyên thuỷ.

## Myth vs Reality

- **Myth:** "`notify()` luôn hiệu quả hơn `notifyAll()` vì chỉ đánh thức một thread thay vì
  toàn bộ, nên nên ưu tiên dùng nó khi có thể."
  **Reality:** Xem Example — nếu dùng sai điều kiện an toàn, `notify()` có thể **chậm hơn
  nhiều** `notifyAll()` do đánh thức nhầm loại thread liên tục, không phải luôn "hiệu quả hơn".

- **Myth:** "`notify()` đánh thức thread đã chờ lâu nhất (kiểu FIFO)."
  **Reality:** JLS **không** đảm bảo thứ tự nào cho `notify()` — nó chọn một thread bất kỳ
  trong wait set, hoàn toàn không có gì đảm bảo là thread chờ lâu nhất.

## Common Mistakes

- **Dùng `notify()` khi có nhiều loại điều kiện chờ khác nhau trên cùng một lock** — đúng lỗi
  trọng tâm của chapter, gây delay khó dự đoán hoặc tệ hơn.
- **Giả định `notify()` có tính công bằng (fairness) hoặc thứ tự cụ thể** — không có đảm bảo
  nào như vậy.
- **Không đặt timeout dự phòng** (`wait(ms)` thay vì `wait()` vô thời hạn) khi có rủi ro
  `notify()` bị dùng sai — mất đi "lưới an toàn" cuối cùng nếu tín hiệu bị lãng phí hoàn toàn.

## Best Practices

- Mặc định dùng `notifyAll()`. Chỉ cân nhắc `notify()` khi đã **chứng minh chắc chắn** mọi
  thread trong wait set đều chờ điều kiện hoàn toàn tương đương, và hiệu năng của
  `notifyAll()` thực sự là vấn đề đã đo lường được (không phải phỏng đoán).
- Với hệ thống thực tế có nhiều loại điều kiện chờ, ưu tiên `Condition` (Chapter 10) thay vì cố
  gắng làm cho `notify()` hoạt động đúng trên một wait set chung.
- Luôn cân nhắc thêm timeout dự phòng cho `wait()` trong hệ thống quan trọng, để tránh treo
  vĩnh viễn nếu có lỗi logic đánh thức.

## Production Notes

**Vấn đề:** một hệ thống xử lý hàng đợi nội bộ dùng `wait()`/`notify()` tự viết, có độ trễ xử
lý tăng dần và không ổn định khi số lượng loại công việc (nhiều loại điều kiện chờ khác nhau
trên cùng một lock) tăng lên theo thời gian phát triển sản phẩm.

- **Triệu chứng:** độ trễ tăng dần, không tương ứng với tăng tải thực tế — hệ thống "chậm đi"
  ngay cả khi khối lượng công việc không đổi, chỉ vì có thêm loại công việc mới.
- **Root cause:** code ban đầu dùng `notify()` khi chỉ có một loại điều kiện chờ (an toàn lúc
  đó) — theo thời gian, thêm loại công việc mới (thêm một loại điều kiện chờ khác) mà không ai
  quay lại kiểm tra giả định "mọi thread chờ đều tương đương" đã không còn đúng.
- **Debug:** rà soát lại mọi nơi gọi `wait()` trên cùng một lock, liệt kê xem có bao nhiêu loại
  điều kiện chờ khác nhau — nếu nhiều hơn một loại, `notify()` không còn an toàn.
- **Solution:** đổi sang `notifyAll()`, hoặc tốt hơn, tái cấu trúc sang `Condition` riêng biệt
  cho từng loại điều kiện (Chapter 10).
- **Prevention:** coi việc dùng `notify()` (thay vì `notifyAll()`) là một quyết định thiết kế
  cần **ghi chú rõ ràng lý do** trong code, và bắt buộc review lại mỗi khi có thay đổi liên
  quan tới logic chờ/đánh thức trên cùng lock đó.

## Debug Checklist

- [ ] Độ trễ xử lý tăng dần theo thời gian phát triển sản phẩm, không tương ứng tải thực tế? →
      kiểm tra có đang dùng `notify()` trong khi số loại điều kiện chờ đã tăng lên không.
- [ ] Hệ thống dùng `wait()`/`notify()` "treo" ngẫu nhiên? → nghi ngờ `notify()` đánh thức nhầm
      loại thread gây mất tín hiệu — kiểm tra có timeout dự phòng không.
- [ ] Cần đánh giá `notify()` có an toàn không? → liệt kê mọi điều kiện `while` khác nhau đang
      chờ trên cùng lock — nếu nhiều hơn một loại, không an toàn.

## Source Code Walkthrough

`Object.notify()`/`notifyAll()` (native method, HotSpot) thao tác trực tiếp lên wait set của
`ObjectMonitor` (Chapter 06): `notify()` di chuyển **một** thread (lựa chọn phụ thuộc hiện thực
JVM cụ thể, không có đảm bảo nào ở tầng đặc tả JLS) từ wait set sang entry set (để nó tranh
giành lại lock); `notifyAll()` di chuyển **toàn bộ** thread trong wait set sang entry set cùng
lúc — tất cả sau đó tự tranh giành lock theo cơ chế thông thường của Monitor, không có đảm bảo
thứ tự nào giữa chúng.

## Summary

`notify()` đánh thức **một** thread ngẫu nhiên trong wait set của Monitor; `notifyAll()` đánh
thức **toàn bộ**. Với nhiều loại điều kiện chờ khác nhau trên cùng một lock (ví dụ producer chờ
"còn chỗ", consumer chờ "có dữ liệu"), `notify()` có nguy cơ đánh thức nhầm loại, gây lãng phí
tín hiệu — đã đo thực nghiệm chênh lệch từ 0ms (`notifyAll()`) lên 411ms (`notify()`) cho cùng
một bài toán chỉ với 2 producer + 2 consumer. Nguyên tắc an toàn: mặc định `notifyAll()`, chỉ
dùng `notify()` khi chắc chắn mọi thread chờ hoàn toàn tương đương. `Condition` (gắn với
`ReentrantLock`, Chapter 10) giải quyết triệt để vấn đề này bằng cách cho phép nhiều wait set
riêng biệt trên cùng một lock.

## Interview Questions

**Junior**

- `notify()` và `notifyAll()` khác nhau như thế nào?
- `notify()` có đảm bảo đánh thức thread chờ lâu nhất không?

**Mid**

- Khi nào dùng `notify()` là an toàn? Cho ví dụ khi nó KHÔNG an toàn.
- Giải thích vì sao dùng `notify()` sai có thể gây delay nghiêm trọng, dựa trên ví dụ cụ thể.

**Senior**

- Giải thích hạn chế cố hữu "một wait set duy nhất" của mô hình Monitor, và `Condition`
  (Chapter 10) giải quyết nó như thế nào.
- Một hệ thống dùng `notify()` an toàn lúc đầu, nhưng dần trở nên không an toàn khi sản phẩm
  phát triển thêm tính năng. Bạn sẽ thiết kế quy trình review/kiểm soát nào để tránh loại lỗi
  này tái diễn?

## Exercises

- [ ] Chạy lại đúng hai ví dụ `NotifyVsNotifyAll` và `NotifyAllDemo` ở trên, xác nhận chênh
      lệch thời gian tương tự trên máy bạn.
- [ ] Viết một bài toán chỉ có **một loại** điều kiện chờ (ví dụ nhiều worker cùng chờ "có
      task mới", không phân biệt loại) — xác nhận `notify()` hoạt động đúng và hiệu quả trong
      trường hợp này.
- [ ] Thử tăng số lượng producer/consumer trong ví dụ `NotifyVsNotifyAll` lên nhiều hơn (ví dụ
      5 producer, 5 consumer) — quan sát chênh lệch thời gian giữa `notify()` và `notifyAll()`
      có tăng thêm không.

## Cheat Sheet

| | `notify()` | `notifyAll()` |
| --- | --- | --- |
| Đánh thức | Một thread ngẫu nhiên | Toàn bộ thread trong wait set |
| An toàn khi | Mọi thread chờ tương đương nhau | Luôn an toàn (mặc định nên dùng) |
| Rủi ro nếu dùng sai | Mất tín hiệu, delay/treo | Không có (chỉ tốn chút overhead) |

**Nguyên tắc:** mặc định `notifyAll()`; `notify()` chỉ khi đã chứng minh mọi thread chờ tương
đương và đã đo được lợi ích hiệu năng thực sự.

## References

- Java SE API Documentation — `java.lang.Object#notify()`, `#notifyAll()`.
- Java Language Specification (JLS) — Chapter 17.2: Wait Sets and Notification.
- Java Concurrency in Practice (Brian Goetz) — Chapter 14.2: Using Condition Queues.
