---
tags:
  - Java
  - ReentrantLock
  - Concurrency
---

# ReentrantLock

> Phase: Phase 3 — Concurrency
> Chapter slug: `reentrantlock`

## Metadata

```yaml
Chapter: ReentrantLock
Phase: Phase 3 — Concurrency
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 65%
Prerequisites:
  - Chapter 09 — Lock (Java-level Locking)
  - Chapter 05 — synchronized (để so sánh tính reentrant)
Used Later:
  - ReadWriteLock (Chapter 11) — mở rộng ý tưởng ReentrantLock cho đọc/ghi tách biệt
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 05 — Deep Dive](05-synchronized.md) — `synchronized` là **reentrant**: một
> thread đã giữ lock có thể giành lại chính nó mà không tự deadlock.

`ReentrantLock` — đúng như tên gọi — mang tính chất đó vào thế giới `Lock` tường minh (Chapter
09). Nhưng nó đi xa hơn: `ReentrantLock` còn **đếm chính xác** số lần một thread đã giành lại
lock của chính nó:

```java
lock.lock();                    // holdCount = 1
reentrant(2);                   // gọi đệ quy, mỗi lần lock() lại
//   depth=2: lock() lại        // holdCount = 2
//   depth=1: lock() lại        // holdCount = 3
```

`getHoldCount()` trả về chính xác `1`, `2`, `3` qua từng lớp gọi đệ quy — một khả năng
`synchronized` hoàn toàn không có (nó cũng reentrant, nhưng không cho bạn **quan sát** được số
lớp). Chapter này đi sâu vào `ReentrantLock` — implementation phổ biến nhất của `Lock`
(Chapter 09) — và giải thích chính xác cơ chế tính "hold count" cùng với `tryLock()` có
timeout.

## Interview Question (Central)

> `ReentrantLock` là gì? "Reentrant" (tái vào) trong tên gọi của nó có ý nghĩa gì, và vì sao
> tính chất này quan trọng?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Tự tay xác nhận `ReentrantLock` cho phép cùng một thread giành lại chính lock nó đang
      giữ, tăng dần "hold count"
- [ ] Dùng đúng `tryLock(timeout, unit)` — thử giành lock trong một khoảng thời gian giới hạn
- [ ] Biết `ReentrantLock` phải nhả đúng số lần đã giành — `unlock()` không đủ số lần
      tương ứng sẽ khiến lock **vẫn còn bị giữ**
- [ ] Biết khái niệm fairness (`new ReentrantLock(true)`) và đánh đổi hiệu năng của nó

## Prerequisites

- Chapter 09 — hiểu interface `Lock` nói chung.
- Chapter 05 — đã biết `synchronized` cũng reentrant, để so sánh.

## Used Later

- **ReadWriteLock** (Chapter 11) — mở rộng cùng ý tưởng, tách biệt lock cho đọc và ghi.

## Problem

Một method `synchronized` gọi một method `synchronized` khác trên cùng object (rất phổ biến —
Chapter 05) không tự deadlock nhờ tính reentrant. `Lock` (Chapter 09) tường minh cần một
implementation cụ thể tái tạo đúng hành vi hữu ích này — nếu không, code hiện có dùng nhiều
lớp gọi lẫn nhau trên cùng một khoá sẽ tự deadlock ngay khi chuyển từ `synchronized` sang
`Lock` tường minh.

## Concept

`ReentrantLock` là implementation phổ biến nhất của `Lock` (Chapter 09) — như tên gọi, nó
**reentrant**: một thread đã giữ lock có thể gọi `lock()` thêm nhiều lần nữa mà không bị chặn
(tăng dần một bộ đếm nội bộ, "hold count"), và phải gọi `unlock()` đúng **số lần tương ứng** để
thực sự nhả lock hoàn toàn (khi hold count về 0).

## Why?

Nếu `ReentrantLock` không reentrant: bất kỳ thiết kế nào có method A (đã giữ lock) gọi method
B (cũng cần giữ chính lock đó) trên cùng thread sẽ tự deadlock ngay lập tức — thread tự chờ
chính lock mà nó đang giữ, không bao giờ thoát ra được. Việc "đếm số lần giành lại" (thay vì
chỉ ghi nhận "đã giữ hay chưa") đảm bảo lock chỉ thực sự được nhả (cho thread khác dùng) khi
**mọi lớp** đã gọi `unlock()` tương ứng — khớp chính xác với cách các lời gọi method lồng
nhau hoạt động.

## How?

```java
ReentrantLock lock = new ReentrantLock();

lock.lock(); // hold count = 1
try {
    doSomethingThatAlsoLocks(); // bên trong lại lock.lock()/unlock() trên CÙNG lock, CÙNG thread
} finally {
    lock.unlock(); // hold count = 0, lock THỰC SỰ được nhả
}

// tryLock với timeout — thử giành trong khoảng thời gian giới hạn
if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
    try { /* ... */ } finally { lock.unlock(); }
} else {
    // không giành được sau 500ms, làm việc khác
}
```

## Visualization

```
Thread T gọi lock.lock() lần 1  → holdCount = 1, T SỞ HỮU lock
Thread T gọi lock.lock() lần 2  → holdCount = 2 (T đã sở hữu, KHÔNG bị BLOCKED)
Thread T gọi lock.lock() lần 3  → holdCount = 3

Thread T gọi lock.unlock() lần 1 → holdCount = 2 (lock VẪN đang bị giữ bởi T)
Thread T gọi lock.unlock() lần 2 → holdCount = 1 (vẫn giữ)
Thread T gọi lock.unlock() lần 3 → holdCount = 0, LOCK THỰC SỰ ĐƯỢC NHẢ
                                     → thread khác giờ mới giành được
```

## Example

Xác nhận trực tiếp hold count qua nhiều lớp gọi đệ quy, và `tryLock()` với timeout:

```java
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.TimeUnit;

public class ReentrantLockDemo {
    static final ReentrantLock lock = new ReentrantLock();

    public static void main(String[] args) throws InterruptedException {
        lock.lock();
        try {
            System.out.println("Main giu lock, holdCount = " + lock.getHoldCount());
            reentrant(2);
        } finally {
            lock.unlock();
        }
        System.out.println("Sau khi main unlock, isLocked = " + lock.isLocked());

        Thread other = new Thread(() -> {
            try {
                boolean acquired = lock.tryLock(500, TimeUnit.MILLISECONDS);
                System.out.println("Thread khac tryLock() thanh cong: " + acquired);
                if (acquired) lock.unlock();
            } catch (InterruptedException e) {}
        });

        lock.lock();
        try {
            other.start();
            other.join();
        } finally {
            lock.unlock();
        }
    }

    static void reentrant(int depth) {
        if (depth == 0) return;
        lock.lock();
        try {
            System.out.println("  reentrant depth=" + depth + ", holdCount=" + lock.getHoldCount());
            reentrant(depth - 1);
        } finally {
            lock.unlock();
        }
    }
}
```

Kết quả thật (JDK 17):

```
Main giu lock, holdCount = 1
  reentrant depth=2, holdCount=2
  reentrant depth=1, holdCount=3
Sau khi main unlock, isLocked = false
Thread khac tryLock() thanh cong: false
```

`holdCount` tăng dần đúng 1, 2, 3 qua từng lớp đệ quy — bằng chứng trực tiếp cho tính
reentrant. `tryLock(500ms)` từ thread khác trả về `false` — đúng dự đoán, vì `main` đang giữ
lock suốt thời gian đó, không nhả trong 500ms.

## Deep Dive

**Vì sao `ReentrantLock(boolean fair)` có tuỳ chọn "fairness" (công bằng) — mặc định là
`false`, tại sao không mặc định `true`?** Với `fair = true`, `ReentrantLock` đảm bảo các thread
giành được lock theo đúng thứ tự **đã yêu cầu** (giống hàng đợi FIFO — thread chờ lâu nhất
được ưu tiên). Với `fair = false` (mặc định), JVM cho phép "chen ngang" (barging) — một thread
vừa mới tới có thể giành được lock trước cả những thread đã chờ lâu, nếu tình cờ lock vừa được
nhả đúng lúc nó tới. Nghe có vẻ "không công bằng", nhưng `fair = false` **nhanh hơn đáng kể**
trong thực tế — duy trì một hàng đợi FIFO nghiêm ngặt tốn thêm chi phí đồng bộ hoá, và trong
đa số trường hợp thực tế, thứ tự giành lock chính xác không quan trọng bằng thông lượng
(throughput) tổng thể. Đây là lý do JDK chọn `false` làm mặc định — ưu tiên hiệu năng cho
trường hợp phổ biến, để `fairness` là một lựa chọn tường minh cho những bài toán thực sự cần nó
(ví dụ tránh hiện tượng một thread "đói" — starvation — không bao giờ giành được lock vì luôn
bị các thread khác chen ngang).

## Engineering Insight

**`ReentrantLock` được cài đặt dựa trên nền tảng nào để đạt hiệu năng tốt mà vẫn linh hoạt hơn
`synchronized`?** Nó xây trên `AbstractQueuedSynchronizer` (AQS, đã nhắc ở
[Chapter 09](09-lock.md)) — một framework nội bộ dùng một biến `int state` (đại diện cho hold
count) được cập nhật bằng **CAS** (Compare-And-Swap, sẽ học kỹ ở Chapter 13) thay vì dựa vào cơ
chế Monitor cấp thấp của JVM (Chapter 06) như `synchronized`. Khi không có tranh chấp
(uncontended — trường hợp phổ biến nhất), `lock()`/`unlock()` chỉ là một thao tác CAS đơn, cực
nhanh, không cần "chặn" thread ở tầng hệ điều hành — chỉ khi thực sự có tranh chấp, AQS mới xếp
thread vào một hàng đợi nội bộ (dùng cấu trúc dữ liệu dạng danh sách liên kết, không phải
Monitor's wait set) và thực sự "chặn" nó. Đây là kiến trúc chung cho **mọi** lock trong
`java.util.concurrent.locks` (`ReentrantLock`, `ReentrantReadWriteLock` — Chapter 11), một
trong những thiết kế có ảnh hưởng lớn nhất của Doug Lea trong lịch sử Java concurrency.

## Historical Note

`ReentrantLock` ra đời cùng đợt `Lock` interface — Java 5 (2004), JSR 166. Trước khi có nó,
tái tạo hành vi "reentrant, có thể interrupt, có tryLock" hoàn toàn bằng tay là công việc rất
phức tạp, dễ sai sót — `ReentrantLock` đóng gói sẵn toàn bộ độ phức tạp đó thành một class sẵn
sàng dùng ngay, dựa trên nền tảng AQS đã được kiểm chứng kỹ lưỡng.

## Myth vs Reality

- **Myth:** "`ReentrantLock` luôn nhanh hơn `synchronized` vì nó 'hiện đại hơn'."
  **Reality:** Từ Java 6 trở đi (Phase 3, Chapter 06 — biased locking, lock coarsening),
  hiệu năng `synchronized` không tranh chấp gần tương đương `ReentrantLock` trong nhiều
  trường hợp — sự khác biệt chính không nằm ở hiệu năng thuần tuý, mà ở các khả năng bổ sung
  (tryLock, interrupt, fairness).

- **Myth:** "Chỉ cần gọi `unlock()` một lần là lock chắc chắn được nhả, bất kể gọi `lock()`
  bao nhiêu lần."
  **Reality:** Xem Example — phải gọi `unlock()` đúng **số lần** đã `lock()` (hold count phải
  về 0), nếu không lock vẫn bị giữ.

## Common Mistakes

- **Gọi `unlock()` không đủ số lần tương ứng với `lock()`** (đặc biệt khi có nhiều đường thoát/
  exception giữa các lớp gọi lồng nhau) — lock vẫn bị giữ dù tưởng đã nhả xong, gây "treo" cho
  thread khác.
- **Nhầm lẫn `isLocked()` (có ai đang giữ lock không) với `isHeldByCurrentThread()`** (thread
  hiện tại có đang giữ lock không) — hai câu hỏi khác nhau, dễ dùng nhầm khi debug.
- **Bật `fair = true` mà không đo lường hiệu năng thực tế** — chi phí công bằng có thể đáng kể
  hơn dự tính, nên chỉ bật khi thực sự cần tránh starvation, đã xác nhận là vấn đề thật.

## Best Practices

- Đảm bảo số lần `lock()` và `unlock()` luôn khớp nhau tuyệt đối — mỗi `lock()` phải có đúng
  một `unlock()` tương ứng trong `finally` (Chapter 09).
- Chỉ bật `fair = true` khi đã xác nhận cụ thể vấn đề starvation là có thật trong hệ thống,
  không bật "phòng ngừa" theo cảm tính.
- Dùng `tryLock(timeout, unit)` cho các bài toán cần giới hạn thời gian chờ, tránh treo vô thời
  hạn khi tài nguyên bị chiếm dụng bất thường lâu.

## Production Notes

**Vấn đề:** một service dùng `ReentrantLock` bảo vệ logic nghiệp vụ có nhiều lớp gọi lồng
nhau (method A gọi B gọi C, cả ba đều giành cùng lock), sau một lần refactor thêm exception
path mới, lock bắt đầu bị giữ vĩnh viễn trong một số tình huống hiếm.

- **Triệu chứng:** "treo" không thường xuyên, chỉ xảy ra khi đi qua đúng nhánh code mới thêm.
- **Root cause:** nhánh code mới có một đường thoát sớm (early return, hoặc một exception mới)
  nằm **giữa** `lock()` và khối try tương ứng, hoặc một lớp gọi lồng nhau thiếu `finally` —
  khiến số lần `unlock()` thực tế ít hơn số lần `lock()` đã xảy ra, hold count không bao giờ về
  0.
- **Debug:** rà soát lại toàn bộ các lớp gọi lồng nhau liên quan tới lock đó, đếm chính xác số
  `lock()`/`unlock()` trên từng đường thực thi có thể xảy ra, đặc biệt các đường mới thêm.
- **Solution:** đảm bảo mọi `lock()` đều có đúng một `unlock()` tương ứng trong `finally`, trên
  **mọi** đường thoát có thể (bao gồm exception mới).
- **Prevention:** khi refactor code có dùng `Lock` lồng nhau nhiều lớp, luôn kiểm tra lại đối
  xứng `lock()`/`unlock()` như một bước bắt buộc trong checklist review.

## Debug Checklist

- [ ] Lock bị giữ dù tưởng đã nhả xong? → dùng `getHoldCount()` để kiểm tra chính xác còn bao
      nhiêu lớp `lock()` chưa được `unlock()` tương ứng.
- [ ] Cần biết ai đang giữ lock hiện tại? → `isLocked()` (có ai giữ không) khác
      `isHeldByCurrentThread()` (thread hiện tại có giữ không) — dùng đúng câu hỏi cần trả lời.
- [ ] Nghi ngờ hiện tượng starvation (một thread không bao giờ giành được lock)? → cân nhắc bật
      `fair = true`, đo lại hiệu năng trước/sau để xác nhận đánh đổi chấp nhận được.

## Source Code Walkthrough

`ReentrantLock.lock()` (không tranh chấp) thực chất là một lệnh CAS đơn trên biến `state` của
AQS (Chapter 09 đã nhắc) — từ `0` (chưa ai giữ) sang `1` (thread hiện tại giữ, hold count = 1).
Gọi `lock()` thêm lần nữa từ **cùng thread** không cần CAS lại — AQS kiểm tra thread hiện tại
đã là chủ sở hữu (`exclusiveOwnerThread`), chỉ tăng `state` lên (không cần thao tác nguyên tử
phức tạp, vì chỉ có đúng thread sở hữu mới có thể thay đổi giá trị này một cách hợp lệ tại thời
điểm đó). `unlock()` giảm `state`, chỉ khi về `0` mới thực sự "mở khoá" và đánh thức thread
tiếp theo (nếu có) trong hàng đợi AQS.

## Summary

`ReentrantLock` (implementation chính của `Lock`, Chapter 09) mang tính chất **reentrant** vào
thế giới lock tường minh — một thread đã giữ lock có thể giành lại chính nó nhiều lần, tăng dần
"hold count" (đã xác nhận trực tiếp: 1, 2, 3 qua các lớp gọi đệ quy), và phải `unlock()` đúng
số lần tương ứng để thực sự nhả lock. `tryLock(timeout, unit)` cho phép chờ giành lock trong
một khoảng thời gian giới hạn, không treo vô thời hạn. Xây trên nền tảng AQS (dùng CAS, không
phải Monitor), `ReentrantLock` đạt hiệu năng tốt trong trường hợp không tranh chấp. Tuỳ chọn
`fair` đánh đổi thứ tự FIFO nghiêm ngặt lấy chi phí hiệu năng cao hơn — mặc định `false` ưu
tiên throughput.

## Interview Questions

**Junior**

- `ReentrantLock` là gì? "Reentrant" nghĩa là gì?
- Gọi `lock()` hai lần liên tiếp trên cùng thread có bị treo không?

**Mid**

- Giải thích "hold count" — nó thay đổi thế nào qua các lớp gọi lồng nhau?
- `tryLock(timeout, unit)` khác `lock()` thông thường như thế nào? Cho use case cụ thể.

**Senior**

- Giải thích đánh đổi giữa `fair = true` và `fair = false` — vì sao JDK chọn `false` làm mặc
  định?
- Mô tả kiến trúc AQS đứng sau `ReentrantLock` ở mức khái niệm — vì sao nó nhanh hơn dựa vào
  CAS thay vì Monitor trong trường hợp không tranh chấp?

## Exercises

- [ ] Chạy lại đúng ví dụ `ReentrantLockDemo` ở trên, xác nhận hold count đúng 1, 2, 3.
- [ ] Cố tình gọi `lock()` hai lần nhưng chỉ `unlock()` một lần — xác nhận lock vẫn bị giữ
      (`isLocked()` vẫn `true`), một thread khác chờ nó sẽ `BLOCKED`/không giành được.
- [ ] Tạo hai `ReentrantLock` — một `fair = true`, một `fair = false` — với nhiều thread cùng
      tranh giành liên tục, đo và so sánh thông lượng (throughput) tổng thể giữa hai loại.

## Cheat Sheet

| Method | Ý nghĩa |
| --- | --- |
| `lock()` | Giành lock, chờ vô thời hạn nếu cần |
| `tryLock()` | Thử giành ngay, trả về `boolean`, không chờ |
| `tryLock(timeout, unit)` | Thử giành trong khoảng thời gian giới hạn |
| `lockInterruptibly()` | Chờ giành lock, có thể bị `interrupt()` |
| `getHoldCount()` | Số lần thread hiện tại đã giành (chưa unlock hết) |
| `isHeldByCurrentThread()` | Thread hiện tại có đang giữ lock không |

## References

- Java SE API Documentation — `java.util.concurrent.locks.ReentrantLock`.
- Java Concurrency in Practice (Brian Goetz) — Chapter 13.1: Lock and ReentrantLock.
- Doug Lea, "The java.util.concurrent Synchronizer Framework" (tài liệu gốc về AQS).
