---
tags:
  - Java
  - synchronized
  - Concurrency
---

# synchronized

> Phase: Phase 3 — Concurrency
> Chapter slug: `synchronized`

## Metadata

```yaml
Chapter: synchronized
Phase: Phase 3 — Concurrency
Difficulty: ★★★★
Importance: ★★★★★
Interview Frequency: 90%
Prerequisites:
  - Chapter 04 — volatile
  - Phase 2, Chapter 11 — ConcurrentHashMap (đã thấy hậu quả race condition)
Used Later:
  - Monitor (Chapter 06) — cơ chế đứng sau synchronized
  - ReentrantLock (Chapter 10) — phiên bản "tường minh" của cùng ý tưởng
Estimated Reading: 25 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 04](04-volatile.md) — `volatile int counter` vẫn mất 76% dữ liệu dưới 10
> thread cùng `counter++`, vì `volatile` không đảm bảo atomicity.

`synchronized` giải quyết đúng phần còn thiếu đó:

```java
static int counter = 0;
static final Object lock = new Object();
// mỗi thread: synchronized (lock) { counter++; }
```

Kết quả đo được: chính xác tuyệt đối `1000000/1000000`, không thiếu một đơn vị nào. Nhưng
`synchronized` cũng có cạm bẫy riêng, tinh vi không kém: khoá **cái gì** mới là điều quan
trọng, không phải chỉ cần viết từ khoá `synchronized` là đủ. Thử khoá trên **một object mới**
mỗi lần:

```java
synchronized (new Object()) { counter++; } // mỗi lần lock KHÁC NHAU!
```

Kết quả: `795578` — gần như không khá hơn không có `synchronized` gì cả! Chapter này giải
thích chính xác `synchronized` bảo vệ **cái gì**, để không mắc phải cạm bẫy tưởng như vô hại
này.

## Interview Question (Central)

> `synchronized` đảm bảo những gì? Điều gì xảy ra nếu hai thread `synchronized` trên hai object
> khác nhau?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Tự tay chứng minh `synchronized` giải quyết trọn vẹn cả visibility lẫn atomicity cho bài
      toán đếm mà `volatile` (Chapter 04) không giải quyết được
- [ ] Hiểu `synchronized` hoạt động dựa trên **một object cụ thể** làm lock — khoá trên object
      khác nhau không bảo vệ được gì cả (đã chứng minh bằng thực nghiệm)
- [ ] Dùng đúng `synchronized` ở cả hai dạng: method và block
- [ ] Biết `synchronized` là **reentrant** — một thread đã giữ lock có thể giành lại chính lock
      đó mà không bị tự deadlock

## Prerequisites

- Chapter 04 — đã hiểu `volatile` giải quyết visibility nhưng không giải quyết atomicity.
- Phase 2, Chapter 11 — đã thấy hậu quả cụ thể của race condition trên `HashMap`.

## Used Later

- **Monitor** (Chapter 06) — cơ chế lý thuyết đầy đủ đứng sau `synchronized`: mỗi object có
  một "monitor" gắn liền.
- **ReentrantLock** (Chapter 10) — phiên bản tường minh (explicit), linh hoạt hơn của cùng ý
  tưởng loại trừ lẫn nhau mà `synchronized` cung cấp.

## Problem

`volatile` (Chapter 04) chỉ bảo vệ được từng thao tác đọc/ghi **riêng lẻ** — không bảo vệ được
một **chuỗi** thao tác cần thực hiện "liền mạch, không bị chen ngang" (đọc-sửa-ghi của
`counter++`, hay phức tạp hơn: kiểm tra điều kiện rồi mới cập nhật). Cần một cơ chế đảm bảo
**chỉ một thread tại một thời điểm** được thực thi một đoạn code cụ thể — mọi thread khác phải
chờ tới lượt.

## Concept

`synchronized` cung cấp **mutual exclusion** (loại trừ lẫn nhau): tại một thời điểm, chỉ **một**
thread được phép thực thi bên trong một khối `synchronized` **trên cùng một object khoá** —
mọi thread khác cố vào cùng khối đó (khoá cùng object) sẽ bị chặn (`BLOCKED`, Chapter 02) cho
tới khi thread đang giữ khoá thoát ra. Ngoài mutual exclusion, `synchronized` còn đảm bảo
**visibility** (Chapter 03) — mọi thay đổi bộ nhớ thực hiện bên trong khối được đảm bảo visible
với thread tiếp theo giành được cùng khoá đó.

## Why?

Nếu chỉ có `volatile`: như đã thấy ở Chapter 04, các thao tác nhiều bước vẫn có thể bị chen
ngang giữa chừng. `synchronized` giải quyết bằng cách biến cả một **khối code** (không chỉ một
biến) thành một đơn vị "không thể chen ngang" — một thread khác **không thể** bắt đầu thực thi
cùng khối đó (trên cùng khoá) cho tới khi thread hiện tại hoàn toàn thoát ra, loại bỏ hoàn toàn
khả năng hai thread cùng đọc một giá trị "cũ" trước khi ai đó kịp ghi giá trị mới.

## How?

```java
// Dạng 1: synchronized METHOD — khoá NGẦM ĐỊNH là chính "this" (instance method)
//         hoặc chính đối tượng Class (static method)
class Counter {
    private int count = 0;
    synchronized void increment() { count++; } // khoá = "this"
}

// Dạng 2: synchronized BLOCK — khoá TƯỜNG MINH, linh hoạt hơn
class Counter2 {
    private int count = 0;
    private final Object lock = new Object();
    void increment() {
        synchronized (lock) { count++; } // khoá = "lock", CỐ ĐỊNH, dùng chung
    }
}
```

Điểm mấu chốt: mọi thread phải `synchronized` trên **cùng một object** thì mới thực sự loại
trừ lẫn nhau. Khoá trên các object khác nhau (như `new Object()` mỗi lần) hoàn toàn không bảo
vệ gì cả — mỗi thread "giữ" một khoá riêng, không ai chờ ai.

## Visualization

```
Đúng — mọi thread synchronized trên CÙNG một "lock":

  Thread A ──synchronized(lock)──▶ [thực thi] ──▶ nhả lock
  Thread B ──synchronized(lock)──▶ [CHỜ — BLOCKED] ──▶ giành được lock sau A
                                                          → THỰC SỰ LOẠI TRỪ LẪN NHAU

Sai — mỗi thread synchronized trên MỘT OBJECT KHÁC NHAU (new Object() mỗi lần):

  Thread A ──synchronized(lockA)──▶ [thực thi]   ← lockA và lockB
  Thread B ──synchronized(lockB)──▶ [thực thi]     KHÁC NHAU, không ai
                                                     chờ ai — CHẠY SONG SONG,
                                                     KHÔNG LOẠI TRỪ GÌ CẢ
```

## Example

**`synchronized` sửa trọn vẹn bài toán đếm mà `volatile` không sửa được:**

```java
import java.util.concurrent.*;

public class SyncRace {
    static int counterNoSync = 0;
    static int counterSync = 0;
    static final Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        int threads = 10, increments = 100_000;
        ExecutorService pool = Executors.newFixedThreadPool(threads);
        for (int t = 0; t < threads; t++) {
            pool.submit(() -> {
                for (int i = 0; i < increments; i++) {
                    counterNoSync++;
                    synchronized (lock) { counterSync++; }
                }
            });
        }
        pool.shutdown();
        pool.awaitTermination(30, TimeUnit.SECONDS);
        System.out.println("Mong doi: " + (threads * increments));
        System.out.println("counterNoSync: " + counterNoSync);
        System.out.println("counterSync: " + counterSync);
    }
}
```

Kết quả thật (JDK 17):

```
Mong doi: 1000000
counterNoSync (khong dong bo): 998474
counterSync (co synchronized): 1000000
```

**Cạm bẫy khoá sai object** — cùng ý định `synchronized` nhưng khoá một object mới mỗi lần:

```java
synchronized (new Object()) { counter++; } // MỖI LẦN GỌI TẠO LOCK MỚI
```

Kết quả thật:

```
synchronized(new Object()) moi lan: 795578 (mong doi 1000000)
```

Gần như không tốt hơn hoàn toàn không có `synchronized` — vì mỗi thread thực chất khoá một
object **khác nhau**, không hề chờ đợi lẫn nhau, `synchronized` ở đây chỉ là vỏ bọc cú pháp
không có tác dụng bảo vệ thực sự.

## Deep Dive

**`synchronized` là *reentrant* — nghĩa là gì, và vì sao quan trọng?** Một thread **đã đang
giữ** một lock có thể **giành lại chính lock đó** (ví dụ khi gọi một method `synchronized`
khác từ bên trong một method `synchronized` đã đang chạy) mà **không bị tự chặn**. Nếu
`synchronized` không reentrant: một method `synchronized` gọi một method `synchronized` khác
trên cùng object (rất phổ biến — ví dụ method A gọi method B, cả hai đều `synchronized`) sẽ
khiến chính thread đó tự `BLOCKED` chờ một lock mà **chính nó** đang giữ — tự deadlock ngay lập
tức. Tính reentrant loại bỏ hoàn toàn lớp lỗi này, cho phép thiết kế class với nhiều method
`synchronized` gọi lẫn nhau một cách tự nhiên.

## Engineering Insight

**Vì sao `synchronized` method static và instance dùng hai lock khác nhau — đây có phải nguồn
gốc của lỗi tinh vi?** `synchronized` trên method **instance** khoá `this` (chính instance đang
gọi); `synchronized` trên method **static** khoá **object `Class`** của chính class đó (ví dụ
`Counter.class`) — hai lock hoàn toàn khác nhau, dù cùng nằm trong một class. Một lỗi thực tế
phổ biến: một field `static` được bảo vệ bởi một method **instance** `synchronized` — vì lock
là `this` (khác nhau giữa các instance), nhiều instance khác nhau **không hề loại trừ lẫn
nhau** khi cùng sửa field `static` dùng chung — chính xác cùng bản chất lỗi đã minh hoạ ở
Example (khoá "khác object" tưởng như bảo vệ nhưng thực chất không): field dùng chung
(`static`) cần một khoá dùng chung (`static` lock hoặc một object `static final` dùng chung
tường minh), không phải khoá theo từng instance.

## Historical Note

`synchronized` tồn tại từ Java 1.0 (1996) — là cơ chế đồng bộ hoá **nguyên thuỷ** đầu tiên của
Java, tích hợp sẵn ở cấp ngôn ngữ (từ khoá, không phải thư viện). Trước Java 5 (2004),
`synchronized` là công cụ **duy nhất** cho mutual exclusion trong Java — `ReentrantLock`
(Chapter 10) và các công cụ `java.util.concurrent` khác chỉ xuất hiện từ Java 5, mang lại tính
linh hoạt (tryLock, timeout, fairness) mà `synchronized` nguyên thuỷ không có. Java 6 (2006)
cải thiện đáng kể hiệu năng nội bộ của `synchronized` (biased locking, lock coarsening — các kỹ
thuật tối ưu JIT áp dụng riêng cho monitor lock), thu hẹp đáng kể khoảng cách hiệu năng từng
tồn tại giữa `synchronized` và các lock tường minh mới hơn.

## Myth vs Reality

- **Myth:** "Chỉ cần thấy từ khoá `synchronized` xuất hiện, đoạn code đó chắc chắn thread-safe."
  **Reality:** Xem Example — `synchronized (new Object())` về mặt cú pháp là `synchronized`
  hoàn chỉnh, nhưng hoàn toàn không bảo vệ được gì vì mỗi lần khoá một object khác nhau.

- **Myth:** "`synchronized` instance method và `synchronized` static method dùng chung một
  lock trong cùng class."
  **Reality:** Chúng dùng hai lock khác nhau (`this` vs `Class` object) — xem Engineering
  Insight về hậu quả cụ thể nếu nhầm lẫn điều này.

## Common Mistakes

- **Khoá trên một object mới mỗi lần** (`synchronized (new Object())`, hoặc khoá trên một
  local variable được tạo mới mỗi lần gọi method) — đúng lỗi trọng tâm của chapter.
- **Trộn lẫn `synchronized` instance method (khoá `this`) với `synchronized static`
  (khoá `Class`) cho cùng dữ liệu dùng chung** — hai lock khác nhau, không loại trừ lẫn nhau.
- **Khoá trên một object có thể bị thay đổi/gán lại** (ví dụ một field không `final`) — nếu
  field đó bị gán một object mới giữa chừng, các thread có thể kết thúc khoá trên các object
  khác nhau mà không nhận ra.

## Best Practices

- Luôn khoá trên một object **cố định, `final`, dùng chung** giữa mọi thread cần loại trừ lẫn
  nhau — không bao giờ tạo object khoá mới bên trong đoạn code được bảo vệ.
- Với field `static` dùng chung, đảm bảo cơ chế bảo vệ (lock) cũng là `static`/dùng chung, không
  phải lock theo từng instance.
- Ưu tiên `synchronized` block (khoá tường minh trên một object riêng) hơn `synchronized`
  method khi cần kiểm soát chính xác phạm vi khoá, tránh khoá nhầm `this` cho toàn bộ object
  khi chỉ cần bảo vệ một phần dữ liệu.

## Production Notes

**Vấn đề:** một class Singleton (một instance duy nhất dùng chung toàn ứng dụng) có method
`synchronized` cập nhật một bộ đếm nội bộ; số liệu đếm cuối cùng vẫn sai lệch dưới tải cao, dù
rõ ràng đã dùng `synchronized`.

- **Triệu chứng:** số liệu sai lệch tương tự hoàn toàn không có bảo vệ nào, dù code "trông có
  vẻ" đã đồng bộ hoá đúng.
- **Root cause:** kiểm tra kỹ phát hiện bộ đếm là field `static`, nhưng method cập nhật nó lại
  là `synchronized` **instance** (khoá `this`) — nếu vì lý do nào đó có nhiều instance của
  class tồn tại (không thực sự là Singleton hoàn hảo, hoặc bị tái khởi tạo ở đâu đó), mỗi
  instance khoá `this` của chính nó, không loại trừ lẫn nhau — đúng lỗi đã phân tích ở
  Engineering Insight.
- **Debug:** xác nhận chính xác object nào đang được dùng làm lock (`this` hay `Class`), và
  dữ liệu đang bảo vệ có thực sự dùng chung ở cùng phạm vi với lock đó không.
- **Solution:** đổi lock cho khớp phạm vi dữ liệu — dữ liệu `static` cần lock `static`
  (`synchronized static` hoặc khoá trên `ClassName.class`/một object `static final` riêng).
- **Prevention:** khi review code `synchronized`, luôn tự hỏi "lock này có cùng phạm vi
  (instance hay static/dùng chung) với dữ liệu nó bảo vệ không?".

## Debug Checklist

- [ ] Dữ liệu vẫn sai lệch dù đã có `synchronized`? → kiểm tra mọi nơi truy cập dữ liệu đó có
      đang khoá **đúng cùng một object** không — nghi ngờ hàng đầu: khoá object mới mỗi lần,
      hoặc lẫn lộn lock instance/static.
- [ ] Deadlock khi một method `synchronized` gọi method `synchronized` khác? → kiểm tra chúng
      có đang khoá trên object khác nhau không — nếu cùng object, tính reentrant đảm bảo không
      tự deadlock.

## Source Code Walkthrough

Ở tầng bytecode, `synchronized` block sinh ra hai lệnh: `monitorenter` (trước khối) và
`monitorexit` (sau khối, và cả trong exception handler ẩn — đảm bảo lock luôn được nhả dù có
exception xảy ra, một dạng try-finally ngầm định). `synchronized` method không dùng
`monitorenter`/`monitorexit` trực tiếp trong bytecode của chính method — thay vào đó, method đó
được đánh dấu cờ `ACC_SYNCHRONIZED` trong class file, và JVM tự động giành/nhả lock khi
vào/ra method, tương đương về mặt ngữ nghĩa nhưng khác cách hiện thực ở tầng bytecode.

## Summary

`synchronized` đảm bảo **mutual exclusion** (chỉ một thread thực thi khối code tại một thời
điểm, trên cùng một object khoá) **và** visibility — giải quyết trọn vẹn bài toán đếm mà
`volatile` (Chapter 04) không giải quyết được, đã xác nhận bằng thực nghiệm (1000000/1000000
chính xác). Nhưng bảo vệ chỉ có hiệu lực nếu mọi thread khoá **cùng một object** — khoá trên
các object khác nhau (kể cả với cú pháp `synchronized` đầy đủ) hoàn toàn không bảo vệ gì, đã
chứng minh bằng thực nghiệm (795578/1000000, gần như vô dụng). `synchronized` là **reentrant**
— một thread đã giữ lock có thể giành lại chính nó mà không tự deadlock, cho phép các method
`synchronized` gọi lẫn nhau tự nhiên.

## Interview Questions

**Junior**

- `synchronized` dùng để làm gì?
- `synchronized` method và `synchronized` block khác nhau như thế nào?

**Mid**

- Điều gì xảy ra nếu hai thread `synchronized` trên hai object khác nhau? Cho ví dụ.
- `synchronized` là "reentrant" nghĩa là gì? Vì sao điều này quan trọng?

**Senior**

- Giải thích vì sao `synchronized` instance method và `synchronized` static method dùng hai
  lock khác nhau — cho một ví dụ cụ thể lỗi phát sinh nếu nhầm lẫn điều này.
- So sánh `synchronized` với `volatile` (Chapter 04) — chúng đảm bảo những gì khác nhau, và
  khi nào chỉ cần dùng `volatile` là đủ, khi nào bắt buộc cần `synchronized`?

## Exercises

- [ ] Chạy lại đúng ví dụ `SyncRace` ở trên, xác nhận `counterSync` chính xác 1000000.
- [ ] Chạy lại đúng ví dụ khoá sai object (`synchronized (new Object())`), xác nhận kết quả
      sai lệch gần như không có bảo vệ.
- [ ] Viết một class có field `static` và một method `synchronized` **instance** cập nhật nó
      — tạo nhiều instance, chạy đồng thời, tái hiện đúng lỗi đã phân tích ở Production Notes.

## Cheat Sheet

| Đảm bảo | `synchronized` |
| --- | --- |
| Visibility | Có |
| Atomicity (mutual exclusion) | Có, nếu khoá đúng cùng một object |
| Reentrant | Có |

| Loại | Lock ngầm định |
| --- | --- |
| `synchronized` instance method | `this` |
| `synchronized` static method | `ClassName.class` |
| `synchronized (obj) { }` | `obj` (tường minh) |

**Nguyên tắc:** mọi thread cần loại trừ lẫn nhau phải khoá trên **cùng một object cố định**.

## References

- Java Language Specification (JLS) — Chapter 17.1: Synchronization.
- Java Concurrency in Practice (Brian Goetz) — Chapter 2: Thread Safety.
