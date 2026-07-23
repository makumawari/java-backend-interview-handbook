---
tags:
  - Java
  - Monitor
  - Concurrency
---

# Monitor

> Phase: Phase 3 — Concurrency
> Chapter slug: `monitor`

## Metadata

```yaml
Chapter: Monitor
Phase: Phase 3 — Concurrency
Difficulty: ★★★★
Importance: ★★★
Interview Frequency: 40%
Prerequisites:
  - Chapter 05 — synchronized
Used Later:
  - wait()/notify() (Chapter 07-08) — hai method này CHỈ gọi được từ bên trong monitor của đúng object
Estimated Reading: 20 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 05](05-synchronized.md) — `synchronized (lock) { ... }` cần một **object**
> để khoá lên.

Câu hỏi ít ai đặt ra: **object nào cũng dùng làm khoá được sao?** Câu trả lời: đúng vậy —
**mọi** object trong Java, không ngoại lệ, đều có thể làm khoá cho `synchronized`, kể cả một
chuỗi `String` hay một số `Integer`. Đây không phải vì `synchronized` "linh hoạt" một cách tuỳ
tiện — nó phản ánh một sự thật sâu hơn: **mọi object trong Java đều có sẵn một "Monitor" gắn
liền với nó**, ngay từ lúc được tạo ra, dù bạn không bao giờ dùng tới. Chapter này giải thích
chính xác Monitor là gì, và tại sao thực tế "object nào cũng khoá được" lại ẩn chứa một cạm bẫy
nguy hiểm khi vô tình dùng sai loại object.

## Interview Question (Central)

> Monitor trong Java là gì? Vì sao mọi object đều có thể dùng làm khoá cho `synchronized`?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Hiểu Monitor là một cấu trúc gắn liền với **mọi** object Java (kế thừa từ `Object`,
      Phase 1 Chapter 09), không phải tính năng riêng của một số class đặc biệt
- [ ] Biết Monitor gồm: một lock (entry set) và một wait set — nền tảng cho cả
      `synchronized` (Chapter 05) lẫn `wait()`/`notify()` (Chapter 07-08)
- [ ] Tự tay xác nhận một cạm bẫy thực tế: khoá trên object `Integer`/`String` có thể vô tình
      "chia sẻ" lock với code hoàn toàn không liên quan
- [ ] Biết lý do lịch sử/kỹ thuật vì sao Java chọn gắn Monitor vào mọi object thay vì tạo một
      kiểu "Lock" riêng biệt ngay từ đầu

## Prerequisites

- Chapter 05 — đã dùng `synchronized`, đã biết khoá trên "một object cụ thể".

## Used Later

- **wait()/notify()** (Chapter 07-08) — hai method này (kế thừa từ `Object`, không phải của
  riêng `Thread`) chỉ gọi được hợp lệ từ **bên trong** khối `synchronized` trên đúng object đó
  — vì chúng thao tác trực tiếp lên Monitor của object.

## Problem

`synchronized` (Chapter 05) cần một khái niệm cụ thể để "gắn" lock, hàng đợi chờ, và (sẽ học ở
Chapter 07-08) cơ chế chờ-đánh thức vào **đúng một object**. Cần trả lời: cấu trúc này tồn tại
ở đâu, và tại sao bất kỳ object nào cũng dùng được cho việc này.

## Concept

**Monitor** (còn gọi *intrinsic lock*) là một cấu trúc đồng bộ hoá gắn liền với **mọi** object
trong Java — không phải một class hay interface riêng biệt bạn phải implement, mà là một phần
"ẩn" của chính cơ chế mà JVM quản lý mọi object. Mỗi Monitor gồm hai thành phần chính:

- **Entry set** (hay "lock"): tập hợp các thread đang chờ **giành quyền vào** khối
  `synchronized` (trạng thái `BLOCKED`, Chapter 02).
- **Wait set**: tập hợp các thread đã **giành được** lock, nhưng chủ động gọi `wait()`
  (Chapter 07) để tạm nhường lại, chờ được `notify()` (Chapter 08).

## Why?

Nếu Monitor chỉ gắn với một số class đặc biệt (ví dụ chỉ class nào implement một interface
"Lockable"): mọi object thông thường sẽ không dùng được trực tiếp với `synchronized`, buộc
phải tạo thêm một object khoá riêng cho mọi trường hợp cần đồng bộ hoá. Việc gắn Monitor vào
**mọi** object (kế thừa từ `Object`, Phase 1 Chapter 09) — dù tuyệt đại đa số object không bao
giờ dùng tới nó — cho phép **bất kỳ** object nào cũng có thể lập tức trở thành một khoá hợp lệ,
đơn giản hoá cú pháp `synchronized` tới mức chỉ cần một object có sẵn, không cần chuẩn bị gì
thêm.

## How?

```java
Object lock = new Object(); // MỌI object, kể cả Object() trần trụi, đều có Monitor sẵn

synchronized (lock) {
    // Giành được lock của Monitor gắn với "lock" → vào ENTRY, chạy code này
    // Thread khác cố synchronized(lock) → vào ENTRY SET, chờ (BLOCKED)

    lock.wait(); // TỪ BỎ lock tạm thời, chuyển sang WAIT SET (Chapter 07)
    // ... một thread khác notify() → quay lại ENTRY, tranh giành lock lại
}
```

## Visualization

```
                    Monitor của object "lock"
        ┌───────────────────────────────────────────┐
        │                                             │
Thread A ──synchronized(lock)──▶ ĐANG GIỮ LOCK, chạy code
        │                                             │
Thread B ──synchronized(lock)──▶  [ENTRY SET — BLOCKED, chờ giành lock]
Thread C ──synchronized(lock)──▶  [ENTRY SET — BLOCKED, chờ giành lock]
        │                                             │
Thread A gọi lock.wait() ────────▶ [WAIT SET — WAITING, đã nhả lock]
        │                                             │
        │   (B hoặc C giành được lock, chạy code)     │
        │   ... lock.notify() ─────────────────────────┼──▶ A quay lại ENTRY SET
        └───────────────────────────────────────────┘
```

## Example

Xác nhận trực tiếp: mọi object, kể cả kiểu bọc số nguyên (`Integer`) hay `String`, đều dùng
`synchronized` được — nhưng ẩn chứa một cạm bẫy nghiêm trọng liên quan tới **cache** của chúng:

```java
public class IntegerLockTrap {
    public static void main(String[] args) {
        Integer a = 100; // TRONG khoảng cache của Integer (-128..127, Integer.valueOf())
        Integer b = 100;
        System.out.println("a == b (Integer 100, co cache): " + (a == b));

        Integer c = 200; // NGOÀI khoảng cache
        Integer d = 200;
        System.out.println("c == d (Integer 200, khong cache): " + (c == d));
    }
}
```

Kết quả thật (JDK 17):

```
a == b (Integer 100, co cache): true
c == d (Integer 200, khong cache): false
```

Hậu quả thực tế: nếu bạn (vô tình) `synchronized (someIntegerVariable)` với giá trị nằm trong
khoảng cache (`-128..127`), bạn có thể đang **khoá chung Monitor** với bất kỳ đoạn code nào
khác trong toàn bộ chương trình (kể cả trong một thư viện bên thứ ba) cũng tình cờ dùng
`Integer.valueOf(100)` làm khoá — hai đoạn code hoàn toàn không liên quan tới nhau **vô tình
loại trừ lẫn nhau**, gây độ trễ khó hiểu hoặc thậm chí deadlock. Đây chính là lý do quy tắc
"không bao giờ khoá trên `Integer`/`String`/kiểu bọc nguyên thuỷ" tồn tại — không phải quy tắc
tuỳ tiện, mà xuất phát trực tiếp từ chính cơ chế cache nội bộ của các class này.

## Deep Dive

**`wait()`/`notify()` (Chapter 07-08) là method của `Object` (Phase 1, Chapter 09), không phải
của `Thread` — vì sao thiết kế lại như vậy?** Vì chúng thao tác trực tiếp lên **Monitor của
chính object đó** (chuyển thread giữa entry set và wait set) — đặt chúng ở `Object` (nơi Monitor
thực sự "sống") là lựa chọn nhất quán về mặt logic, dù ban đầu có thể gây bối rối (tại sao
"chờ" và "đánh thức" lại là method của `Object`, không phải `Thread`?). Đặt trên `Thread` sẽ
sai về ngữ nghĩa: bạn không "chờ một Thread" — bạn "chờ trên Monitor của một object cụ thể",
cho tới khi có ai đó `notify()` **đúng Monitor đó**.

## Engineering Insight

**Vì sao JVM (HotSpot) tối ưu để hầu hết `synchronized` không tốn chi phí đáng kể, dù về mặt
khái niệm mọi object đều "mang theo" một Monitor?** Vì HotSpot dùng chiến lược
**biased locking** (tối ưu, thay đổi qua các phiên bản — xem Historical Note) và
**lock coarsening**: phần lớn object trong thực tế **không bao giờ** thực sự bị tranh chấp bởi
nhiều thread — HotSpot phát hiện điều này và tránh chi phí đồng bộ hoá đầy đủ (cấp một cấu trúc
Monitor "nặng" ở tầng hệ điều hành) cho tới khi thực sự có tranh chấp xảy ra. Đây là lý do
`synchronized` hiện đại có chi phí gần như không đáng kể trong trường hợp không tranh chấp
(uncontended) — một minh chứng khác cho triết lý "JIT tối ưu dựa trên hành vi runtime thật" đã
học ở [Phase 1, Chapter 06](../phase-01-foundation/06-jit-compiler.md), áp dụng cụ thể cho cơ
chế khoá.

## Historical Note

```
Java 1.0-1.5
    ↓
Mọi synchronized đều dùng "heavyweight lock" — cấp phát cấu trúc đồng bộ
tại tầng hệ điều hành ngay cả khi không có tranh chấp, chi phí đáng kể
    ↓
Java 6 (2006)
    ↓
Biased locking + lock coarsening + adaptive spinning ra đời — HotSpot tối
ưu mạnh cho trường hợp không tranh chấp (phổ biến nhất trong thực tế),
thu hẹp khoảng cách hiệu năng với các lock tường minh mới (ReentrantLock)
    ↓
Java 15+ (2020 trở đi)
    ↓
Biased locking dần bị loại bỏ khỏi mặc định (JEP 374) — các kỹ thuật tối
ưu khác của JVM hiện đại đã đủ tốt, biased locking gây thêm phức tạp bảo
trì không còn tương xứng lợi ích trong hầu hết trường hợp
```

## Myth vs Reality

- **Myth:** "Chỉ những class đặc biệt (implement một interface nào đó) mới dùng được với
  `synchronized`."
  **Reality:** **Mọi** object Java đều có Monitor sẵn — kể cả `new Object()` trần trụi, không
  cần implement gì cả.

- **Myth:** "Khoá trên `Integer`/`String` chỉ là 'không nên làm' vì lý do phong cách code,
  không có hậu quả kỹ thuật cụ thể."
  **Reality:** Xem Example — do cơ chế cache (Integer -128..127, String Pool — Phase 1 Chapter
  10), khoá trên các kiểu này có thể gây **chia sẻ lock ngoài ý muốn** với code hoàn toàn
  không liên quan, một hậu quả kỹ thuật rất cụ thể, không chỉ là vấn đề phong cách.

## Common Mistakes

- **Khoá trên `Integer`/`Long`/`String`/kiểu bọc nguyên thuỷ** — đúng cạm bẫy trọng tâm của
  chapter, do cơ chế cache nội bộ của các class này.
- **Khoá trên `this` một cách bừa bãi trong class public** — bất kỳ code bên ngoài nào cũng có
  thể `synchronized (yourObjectInstance)` từ bên ngoài, vô tình (hoặc cố ý) can thiệp vào cơ
  chế khoá nội bộ của class bạn — nên dùng một object khoá `private` riêng thay vì `this`.
- **Nhầm lẫn Monitor với `ReentrantLock`/`Lock`** (Chapter 09-10) — Monitor là cơ chế **ngầm
  định**, gắn với `synchronized`; `Lock` là API **tường minh**, độc lập, linh hoạt hơn.

## Best Practices

- Không bao giờ khoá trên kiểu bọc nguyên thuỷ (`Integer`, `Long`, `Boolean`) hay `String`.
- Trong class `public`, ưu tiên khoá trên một object `private final` riêng, không khoá trên
  `this` — tránh code bên ngoài vô tình can thiệp vào cơ chế đồng bộ nội bộ.
- Hiểu Monitor ở mức khái niệm là đủ cho phần lớn công việc hàng ngày — không cần quan tâm chi
  tiết tối ưu biased locking/lock coarsening trừ khi đang tối ưu hiệu năng sâu ở Phase 8.

## Production Notes

**Vấn đề:** hai tính năng hoàn toàn không liên quan trong cùng một ứng dụng (ví dụ module xử lý
đơn hàng và module xử lý log) thỉnh thoảng gây độ trễ bất thường lẫn nhau, dù code review không
tìm thấy bất kỳ điểm gọi lẫn nhau nào.

- **Triệu chứng:** độ trễ tăng đột biến ở module A đúng lúc module B đang xử lý tải cao, dù hai
  module này về mặt logic hoàn toàn độc lập, không có lời gọi trực tiếp nào giữa chúng.
- **Root cause:** cả hai module tình cờ đều `synchronized` trên một `Integer`/`String` cùng giá
  trị (ví dụ cùng dùng một mã trạng thái nhỏ như `Integer` `1` hoặc `2` làm khoá) — do cache
  nội bộ, chúng đang vô tình khoá chung **một** Monitor, gây tranh chấp giả tạo giữa hai tính
  năng không liên quan.
- **Debug:** lấy thread dump (Chapter 02) lúc xảy ra độ trễ, tìm các thread `BLOCKED` — kiểm
  tra chính xác object nào đang được khoá; nếu là `Integer`/`String`, nghi ngờ ngay cơ chế
  cache.
- **Solution:** đổi sang khoá trên object `private final` riêng biệt cho từng module.
- **Prevention:** cấm tuyệt đối khoá trên kiểu bọc nguyên thuỷ/`String` trong coding standard;
  static analysis (nhiều công cụ có sẵn rule cảnh báo mẫu hình này).

## Debug Checklist

- [ ] Hai tính năng không liên quan gây độ trễ lẫn nhau khó hiểu? → lấy thread dump, kiểm tra
      object đang được khoá — nghi ngờ Integer/String cache nếu thấy khoá trên các kiểu này.
- [ ] `synchronized` trên `Integer`/`String` trong code review? → luôn gắn cờ, yêu cầu đổi
      sang object khoá riêng.

## Source Code Walkthrough

Ở mức HotSpot, Monitor được hiện thực (khi thực sự cần, sau khi biased/thin locking không còn
đủ do có tranh chấp) bằng cấu trúc C++ nội bộ tên `ObjectMonitor`, gắn với header của object
(mỗi object Java có một vùng header nhỏ chứa metadata, bao gồm thông tin khoá — chi tiết vượt
ngoài phạm vi chapter nền tảng này). Việc "mọi object đều có Monitor" ở mức khái niệm Java
không đồng nghĩa mọi object đều tốn chi phí bộ nhớ cho một `ObjectMonitor` đầy đủ ngay từ đầu —
cấu trúc nặng này chỉ thực sự được cấp phát khi có tranh chấp thật sự xảy ra (đúng tinh thần
tối ưu đã bàn ở Engineering Insight).

## Summary

Monitor (intrinsic lock) là cấu trúc đồng bộ hoá gắn liền với **mọi** object Java, gồm một
entry set (thread chờ giành lock, `BLOCKED`) và một wait set (thread đã nhường lock qua
`wait()`, sẽ học ở Chapter 07-08) — nền tảng cho cả `synchronized` (Chapter 05) lẫn
`wait()`/`notify()`. Vì mọi object đều có Monitor sẵn, bất kỳ object nào cũng khoá được — nhưng
điều này ẩn chứa cạm bẫy nghiêm trọng với các kiểu có **cache nội bộ** (`Integer` trong khoảng
`-128..127`, `String` trong String Pool — Phase 1, Chapter 10): khoá trên chúng có thể vô tình
chia sẻ Monitor với code hoàn toàn không liên quan, đã xác nhận cơ chế cache bằng thực nghiệm.
HotSpot tối ưu mạnh (biased locking lịch sử, lock coarsening) để `synchronized` không tranh
chấp gần như miễn phí về hiệu năng.

## Interview Questions

**Junior**

- Monitor trong Java là gì?
- Object nào dùng được làm khoá cho `synchronized`?

**Mid**

- Vì sao không nên khoá trên `Integer`/`String`? Giải thích nguyên nhân kỹ thuật cụ thể.
- `wait()`/`notify()` là method của class nào? Vì sao lại thiết kế như vậy?

**Senior**

- Giải thích Entry Set và Wait Set của một Monitor — mối quan hệ giữa chúng khi một thread gọi
  `wait()` rồi được `notify()`.
- Một hệ thống có hai module không liên quan gây độ trễ lẫn nhau khó hiểu. Trình bày quy trình
  điều tra, liên hệ trực tiếp tới cơ chế cache của Integer/String và Monitor.

## Exercises

- [ ] Chạy lại đúng ví dụ `IntegerLockTrap` ở trên, xác nhận hành vi cache đúng khoảng
      `-128..127`.
- [ ] Viết hai đoạn code độc lập, cả hai đều `synchronized (Integer.valueOf(50))` — chạy đồng
      thời, quan sát chúng có vô tình loại trừ lẫn nhau dù logic hoàn toàn không liên quan hay
      không.
- [ ] Đọc tài liệu JEP 374 (Deprecate and Disable Biased Locking) — tóm tắt vì sao Java quyết
      định tắt mặc định một tối ưu từng rất quan trọng.

## Cheat Sheet

| Thành phần Monitor | Ý nghĩa |
| --- | --- |
| Entry Set | Thread chờ giành lock (`BLOCKED`) |
| Wait Set | Thread đã gọi `wait()`, chờ `notify()` (`WAITING`) |

**Không bao giờ khoá trên:** `Integer`, `Long`, `Short`, `Byte`, `Boolean` (kiểu bọc, có cache),
`String` (có String Pool).
**Nên khoá trên:** một object `private final` riêng, tạo đúng một lần.

## References

- Java Language Specification (JLS) — Chapter 17.1: Synchronization.
- `java.lang.Object` — Java SE API Documentation (phần `wait`/`notify`/`notifyAll`).
- JEP 374: Deprecate and Disable Biased Locking.
