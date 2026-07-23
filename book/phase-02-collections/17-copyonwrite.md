---
tags:
  - Java
  - CopyOnWrite
  - Collections
  - Concurrency
---

# CopyOnWrite

> Phase: Phase 2 — Collections
> Chapter slug: `copyonwrite`

## Metadata

```yaml
Chapter: CopyOnWrite
Phase: Phase 2 — Collections
Difficulty: ★★★★
Importance: ★★★
Interview Frequency: 40%
Prerequisites:
  - Chapter 11 — ConcurrentHashMap
  - Chapter 01 — Collection Framework (fail-fast)
Used Later:
  - Thread, Memory Visibility (Phase 3) — cơ chế "visibility" đứng sau CopyOnWrite sẽ có nền tảng lý thuyết đầy đủ ở đó
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 01](01-collection-framework.md) — sửa một `ArrayList` trong lúc for-each
> đang duyệt nó (kể cả đơn luồng) ném `ConcurrentModificationException`.

Thử chính xác thao tác đó — xoá một phần tử ngay trong lúc for-each — nhưng lần này trên
`CopyOnWriteArrayList`:

```java
List<String> cow = new CopyOnWriteArrayList<>(List.of("A", "B", "C"));
for (String s : cow) {
    if (s.equals("B")) cow.remove(s); // sửa TRONG LÚC đang duyệt
}
System.out.println(cow); // [A, C] — KHÔNG có exception nào!
```

Không một exception nào được ném ra — điều mà `ArrayList` (Chapter 03) coi là vi phạm nghiêm
trọng lại hoàn toàn hợp lệ ở đây. `CopyOnWriteArrayList` không phải "sửa lỗi" fail-fast — nó
giải quyết vấn đề bằng một chiến lược hoàn toàn khác: **không bao giờ sửa mảng đang được
duyệt**. Mỗi lần ghi, nó tạo ra một **bản sao mới toàn bộ**. Chapter này giải thích chiến lược
đánh đổi táo bạo này, và khi nào nó thực sự đáng giá.

## Interview Question (Central)

> `CopyOnWriteArrayList` cho phép sửa collection trong lúc đang duyệt mà không ném
> `ConcurrentModificationException`. Nó đạt được điều đó bằng cách nào, và cái giá phải trả là
> gì?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích chiến lược **copy-on-write**: mỗi lần ghi tạo bản sao toàn bộ mảng mới
- [ ] Tự tay tái hiện được sự khác biệt: `ArrayList` ném exception, `CopyOnWriteArrayList` thì
      không, cho cùng một thao tác
- [ ] Biết cái giá cụ thể: mỗi lần ghi là O(n) (copy toàn bộ), phù hợp cho **đọc nhiều, ghi
      hiếm**
- [ ] Phân biệt "snapshot iterator" (không phản ánh thay đổi sau khi tạo) với iterator thường

## Prerequisites

- Chapter 11 — đã hiểu `ConcurrentHashMap` giải quyết concurrency bằng khoá phạm vi nhỏ;
  `CopyOnWriteArrayList` là một triết lý hoàn toàn khác.
- Chapter 01 — đã biết fail-fast của `ArrayList` và tại sao sửa trong lúc duyệt thường nguy
  hiểm.

## Used Later

- **Thread, Memory Visibility** (Phase 3) — vì sao thay đổi từ một thread "nhìn thấy được" bởi
  thread khác một cách an toàn (visibility) là một khái niệm sâu hơn, sẽ được học đầy đủ ở
  Phase 3; `CopyOnWriteArrayList` là một ứng dụng cụ thể của nguyên lý đó.

## Problem

`ArrayList` (Chapter 03) không cho sửa cấu trúc trong lúc duyệt — kể cả trong ngữ cảnh đơn
luồng, đây là biện pháp an toàn hợp lý (Chapter 01). Nhưng trong một số bài toán thực tế — ví
dụ danh sách **listener/subscriber** của một hệ thống sự kiện, được **đọc rất thường xuyên**
(mỗi sự kiện phải duyệt qua để thông báo) nhưng **hiếm khi thay đổi** (chỉ khi có
listener đăng ký/huỷ đăng ký) — việc phải đồng bộ hoá đọc bằng khoá (`synchronized`, hoặc
`ConcurrentHashMap`-style) cho một thao tác đọc rất thường xuyên tạo ra chi phí không cần thiết.

## Concept

`CopyOnWriteArrayList` áp dụng chiến lược **copy-on-write**: mọi thao tác **ghi** (`add()`,
`remove()`, `set()`) tạo ra một **bản sao hoàn toàn mới** của toàn bộ mảng nội bộ, thực hiện
thay đổi trên bản sao đó, rồi thay thế tham chiếu mảng cũ bằng mảng mới (một phép gán tham
chiếu đơn giản, an toàn). Thao tác **đọc** (`get()`, iterator, for-each) luôn làm việc trên
snapshot của mảng **tại thời điểm bắt đầu đọc**, hoàn toàn không cần khoá gì cả.

## Why?

Vì mọi thao tác ghi tạo mảng **mới**, không bao giờ sửa mảng **đang tồn tại** (mảng mà các
thao tác đọc/duyệt khác có thể đang cầm tham chiếu tới) — một iterator đã bắt đầu duyệt sẽ tiếp
tục thấy đúng mảng cũ (snapshot) từ đầu tới cuối, dù ai đó đã thêm/xoá phần tử ở "phiên bản
mới" song song. Không có khái niệm "sửa cấu trúc trong khi có ai đó đang duyệt" nữa — vì "cấu
trúc đang được duyệt" và "cấu trúc bị sửa" **là hai mảng vật lý khác nhau** ngay từ khoảnh khắc
ghi đầu tiên xảy ra. Đây là lý do fail-fast không còn cần thiết — không phải vì
`CopyOnWriteArrayList` "khắc phục" được vấn đề fail-fast, mà vì nó loại bỏ hoàn toàn tình huống
mà fail-fast được sinh ra để phát hiện.

## How?

```
list.add(e):
1. Khoá (chỉ để đảm bảo các THAO TÁC GHI không đụng nhau, KHÔNG ảnh hưởng thao tác đọc)
2. Copy TOÀN BỘ mảng hiện tại sang mảng MỚI, lớn hơn 1 phần tử  — O(n)
3. Thêm e vào cuối mảng mới
4. Gán tham chiếu nội bộ "array" TRỎ SANG mảng mới (một phép gán, an toàn)
5. Mở khoá

list.iterator()/for-each:
   Lấy tham chiếu tới array HIỆN TẠI ngay lúc bắt đầu — KHÔNG khoá gì cả
   Duyệt trên CHÍNH mảng đó cho tới hết, DÙ ai đó ghi thêm dữ liệu song song
   (ghi tạo mảng MỚI khác, không đụng tới mảng đang được duyệt)
```

## Visualization

```
Thời điểm T0: array trỏ tới [A, B, C]
      │
      │  Thread 1 bắt đầu for-each — LẤY THAM CHIẾU tới [A, B, C] lúc này
      │  (dù sau đó "array" đổi, Thread 1 vẫn duyệt đúng bản snapshot này)
      ▼
Thời điểm T1: Thread 2 gọi remove("B")
      │
      │  Copy [A, B, C] → mảng MỚI [A, C]
      │  array (biến tham chiếu) đổi sang trỏ [A, C]
      ▼
Thread 1 TIẾP TỤC duyệt [A, B, C] (bản snapshot cũ, KHÔNG đổi giữa chừng)
   → Thread 1 vẫn thấy "B", không hề biết nó vừa bị xoá — KHÔNG có exception
```

## Example

Tái hiện đúng sự khác biệt đã nêu ở Story — sửa trong lúc duyệt trên cả `ArrayList` lẫn
`CopyOnWriteArrayList`:

```java
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;

public class CowDemo {
    public static void main(String[] args) {
        List<String> normal = new ArrayList<>(List.of("A", "B", "C", "D"));
        try {
            for (String s : normal) {
                if (s.equals("B")) normal.remove(s);
            }
        } catch (ConcurrentModificationException e) {
            System.out.println("ArrayList: " + e.getClass().getSimpleName() + " khi sua trong luc duyet");
        }

        List<String> cow = new CopyOnWriteArrayList<>(List.of("A", "B", "C"));
        for (String s : cow) {
            if (s.equals("B")) cow.remove(s); // sua trong luc dang duyet - AN TOAN
        }
        System.out.println("CopyOnWriteArrayList sau khi sua trong luc duyet: " + cow);
    }
}
```

Kết quả thật (JDK 17):

```
ArrayList: ConcurrentModificationException khi sua trong luc duyet
CopyOnWriteArrayList sau khi sua trong luc duyet: [A, C]
```

`ArrayList` ném lỗi (theo đúng cơ chế fail-fast, Chapter 01). `CopyOnWriteArrayList` chạy hoàn
toàn êm đẹp, và kết quả cuối cùng `[A, C]` xác nhận `remove("B")` đã thực sự thành công — chỉ
là không ảnh hưởng tới chính vòng lặp đang chạy.

## Deep Dive

**"Snapshot iterator" nghĩa là gì, và nó có thể gây bất ngờ ra sao?** Vì iterator của
`CopyOnWriteArrayList` cầm tham chiếu tới mảng **tại thời điểm bắt đầu duyệt**, nó **sẽ không
bao giờ thấy** những thay đổi xảy ra sau đó — kể cả những thay đổi hợp lệ, có chủ đích. Ví dụ:
nếu code ở Example thêm một dòng `cow.add("E")` ngay trong vòng lặp, phần tử `"E"` đó **sẽ
không xuất hiện** trong chính vòng lặp đang chạy (dù `cow.size()` sau đó phản ánh đúng 4 phần
tử) — vòng lặp for-each hiện tại vẫn tiếp tục dựa trên snapshot 3 phần tử ban đầu. Đây không
phải bug — là hệ quả tất yếu, có chủ đích của thiết kế copy-on-write, nhưng có thể gây bối rối
nghiêm trọng nếu không hiểu rõ nguyên lý này.

## Engineering Insight

**Vì sao `CopyOnWriteArrayList` chỉ phù hợp cho "đọc nhiều, ghi hiếm" — điều gì xảy ra nếu ghi
thường xuyên?** Mỗi lần ghi là O(n) — copy **toàn bộ** mảng, bất kể chỉ thêm/xoá một phần tử.
Với danh sách lớn và ghi thường xuyên, chi phí này cộng dồn rất nhanh, tệ hơn nhiều so với
`ArrayList.add()` thông thường (Chapter 03, O(1) amortized) hay thậm chí
`Collections.synchronizedList()` (Chapter 15, chỉ khoá, không copy). Đây là lý do
`CopyOnWriteArrayList` **không phải** một "phiên bản an toàn hơn của `ArrayList` cho mọi
trường hợp" — nó là công cụ chuyên biệt cho đúng một mẫu hình sử dụng: danh sách được **đọc/
duyệt liên tục** (như danh sách listener sự kiện — mỗi sự kiện phải duyệt qua toàn bộ) nhưng
**hiếm khi thay đổi cấu trúc** (đăng ký/huỷ đăng ký listener không xảy ra thường xuyên như bản
thân sự kiện). Chọn sai ngữ cảnh (ghi thường xuyên) biến "giải pháp an toàn" thành "vấn đề hiệu
năng nghiêm trọng".

## Historical Note

`CopyOnWriteArrayList`/`CopyOnWriteArraySet` ra đời cùng đợt với `ConcurrentHashMap` — Java 5
(2004), JSR 166, cùng một nỗ lực bổ sung các cấu trúc dữ liệu concurrent chuyên biệt vào JDK,
thay vì chỉ có lựa chọn duy nhất là khoá toàn bộ (`Hashtable`/`Vector`/`Collections.
synchronizedXxx()`, đã học ở Chapter 11 và 15).

## Myth vs Reality

- **Myth:** "`CopyOnWriteArrayList` là phiên bản 'nâng cấp' của `ArrayList`, luôn nên dùng cho
  mọi trường hợp cần thread-safe."
  **Reality:** Chỉ phù hợp cho đọc nhiều/ghi hiếm — với ghi thường xuyên, chi phí O(n) mỗi lần
  ghi khiến nó chậm hơn nhiều so với các lựa chọn khác (`synchronizedList`, hoặc thiết kế lại
  bài toán).

- **Myth:** "Iterator của `CopyOnWriteArrayList` luôn phản ánh trạng thái mới nhất của danh
  sách."
  **Reality:** Ngược lại — nó là **snapshot** tại thời điểm bắt đầu duyệt, không bao giờ thấy
  thay đổi xảy ra sau đó, dù thay đổi đó hoàn toàn hợp lệ.

## Common Mistakes

- **Dùng `CopyOnWriteArrayList` cho danh sách bị ghi thường xuyên** (ví dụ hàng đợi công việc
  liên tục thêm/xoá) — chọn sai công cụ, gây hiệu năng tệ hơn nhiều so với các lựa chọn khác.
- **Kỳ vọng vòng lặp đang chạy thấy được các thay đổi ghi vào giữa chừng** — hiểu sai bản chất
  snapshot iterator.

## Best Practices

- Chỉ dùng `CopyOnWriteArrayList`/`CopyOnWriteArraySet` cho danh sách đọc/duyệt rất thường
  xuyên nhưng hiếm khi ghi — ví dụ điển hình: danh sách listener/observer trong một hệ thống
  sự kiện.
- Nếu cần cả đọc lẫn ghi thường xuyên trong môi trường đa luồng, cân nhắc
  `ConcurrentHashMap`-tương-đương (Chapter 11) hoặc các cấu trúc dữ liệu concurrent chuyên biệt
  khác thay vì copy-on-write.

## Production Notes

**Vấn đề:** một hệ thống publish-subscribe nội bộ (dùng `CopyOnWriteArrayList<Listener>` để
lưu danh sách subscriber) hoạt động tốt lúc đầu, nhưng CPU tăng cao bất thường khi số lượng
subscriber và tần suất đăng ký/huỷ đăng ký tăng lên đáng kể.

- **Triệu chứng:** CPU cao, độ trễ tăng khi có nhiều thao tác đăng ký/huỷ đăng ký subscriber
  diễn ra dồn dập (ví dụ trong một đợt scale-out/scale-in nhiều instance).
- **Root cause:** giả định ban đầu ("đăng ký/huỷ đăng ký hiếm khi xảy ra") không còn đúng ở quy
  mô mới — mỗi lần đăng ký/huỷ đăng ký giờ tốn O(n) copy toàn bộ danh sách subscriber (đã lớn),
  và tần suất này giờ đã cao, biến "chiến lược tối ưu cho ghi hiếm" thành gánh nặng.
- **Debug:** profiling cho thấy phần lớn thời gian CPU nằm trong việc copy mảng nội bộ của
  `CopyOnWriteArrayList`.
- **Solution:** đánh giá lại giả định "ghi hiếm" có còn đúng ở quy mô hiện tại không; nếu
  không, cân nhắc cấu trúc dữ liệu khác phù hợp hơn với mẫu hình đọc/ghi mới.
- **Prevention:** khi chọn `CopyOnWriteArrayList`, ghi rõ giả định "đọc nhiều/ghi hiếm" như một
  ràng buộc thiết kế tường minh (comment hoặc tài liệu), để dễ phát hiện khi giả định đó không
  còn đúng theo thời gian quy mô hệ thống thay đổi.

## Debug Checklist

- [ ] CPU cao bất thường liên quan tới một `CopyOnWriteArrayList`? → kiểm tra tần suất ghi
      (`add`/`remove`) thực tế — có còn "hiếm" như giả định ban đầu không.
- [ ] Vòng lặp không thấy phần tử vừa được thêm/xoá bởi thao tác khác? → xác nhận đây là hành
      vi snapshot iterator đúng thiết kế, không phải bug.

## Source Code Walkthrough

`CopyOnWriteArrayList` giữ một field `volatile Object[] array` (từ khoá `volatile` — sẽ học kỹ
ở Phase 3, đảm bảo mọi thread luôn thấy giá trị mới nhất của chính tham chiếu `array` sau khi
gán). Mọi thao tác ghi dùng một `ReentrantLock` nội bộ (Phase 3) để đảm bảo hai thao tác ghi
đồng thời không đụng nhau, nhưng **không hề khoá thao tác đọc** — `iterator()` chỉ đơn giản đọc
giá trị hiện tại của `array` (nhờ `volatile`, luôn là giá trị mới nhất tại thời điểm đọc) rồi
duyệt trên đúng mảng đó tới hết, không quan tâm `array` có đổi tiếp theo hay không.

## Summary

`CopyOnWriteArrayList` dùng chiến lược copy-on-write: mọi thao tác ghi tạo một bản sao **toàn
bộ mảng mới** (O(n)), thay vì sửa mảng hiện tại — nhờ vậy, sửa collection trong lúc đang duyệt
(kể cả từ thread khác) không bao giờ ném `ConcurrentModificationException`, vì mảng đang được
duyệt và mảng bị sửa luôn là hai object khác nhau. Đổi lại, mỗi lần ghi tốn O(n), khiến nó chỉ
phù hợp cho mẫu hình "đọc/duyệt rất thường xuyên, ghi hiếm" (ví dụ danh sách listener sự kiện)
— dùng sai ngữ cảnh (ghi thường xuyên) biến lợi thế an toàn thành gánh nặng hiệu năng nghiêm
trọng. Iterator của nó là một **snapshot**, không bao giờ phản ánh thay đổi xảy ra sau khi bắt
đầu duyệt.

## Interview Questions

**Junior**

- `CopyOnWriteArrayList` là gì? Nó có ném `ConcurrentModificationException` khi sửa trong lúc
  duyệt không?
- `CopyOnWriteArrayList` phù hợp cho use case nào?

**Mid**

- Giải thích chiến lược copy-on-write. Vì sao nó tránh được `ConcurrentModificationException`?
- "Snapshot iterator" nghĩa là gì? Cho ví dụ nó có thể gây bất ngờ ra sao.

**Senior**

- Phân tích chi phí O(n) mỗi lần ghi của `CopyOnWriteArrayList` — trong tình huống nào chi phí
  này chấp nhận được, tình huống nào thì không?
- So sánh triết lý `CopyOnWriteArrayList` (copy toàn bộ, không khoá đọc) với
  `ConcurrentHashMap` (Chapter 11, khoá phạm vi nhỏ) — vì sao JDK cần cả hai chiến lược khác
  nhau thay vì chỉ một giải pháp chung?

## Exercises

- [ ] Chạy lại đúng ví dụ `CowDemo` ở trên, xác nhận `ArrayList` ném lỗi còn
      `CopyOnWriteArrayList` thì không.
- [ ] Thêm một phần tử mới (`cow.add("E")`) ngay trong vòng lặp for-each đang chạy trên
      `CopyOnWriteArrayList` — xác nhận phần tử đó KHÔNG xuất hiện trong chính vòng lặp đang
      chạy, dù `cow.size()` sau đó đã tăng.
- [ ] Đo thời gian thêm 10.000 phần tử liên tục vào `CopyOnWriteArrayList` so với `ArrayList`
      — xác nhận chênh lệch rất lớn, minh hoạ trực tiếp chi phí O(n) mỗi lần ghi.

## Cheat Sheet

| | ArrayList | CopyOnWriteArrayList |
| --- | --- | --- |
| Sửa khi đang duyệt | `ConcurrentModificationException` | Không lỗi (snapshot) |
| Chi phí ghi | O(1) amortized | O(n) (copy toàn bộ) |
| Thread-safe | Không | Có |
| Phù hợp cho | Mục đích chung | Đọc nhiều, ghi hiếm (listener list) |

## References

- Java SE API Documentation — `java.util.concurrent.CopyOnWriteArrayList`.
- JSR 166: Concurrency Utilities.
- Java Concurrency in Practice (Brian Goetz) — chương về concurrent collections.
