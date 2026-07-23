---
tags:
  - Java
  - Collections
  - Utility
---

# Collections Utility

> Phase: Phase 2 — Collections
> Chapter slug: `collections-utility`

## Metadata

```yaml
Chapter: Collections Utility
Phase: Phase 2 — Collections
Difficulty: ★★
Importance: ★★★
Interview Frequency: 45%
Prerequisites:
  - Chapter 14 — Comparable vs Comparator
Used Later:
  - Immutable Collection (Chapter 16) — Collections.unmodifiableXxx() là tiền thân của List.of()
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

`Collections` (số nhiều, khác `Collection` số ít đã học ở Chapter 01) là một class tiện ích
— toàn bộ method đều `static`, không thể tạo instance. Nó không phải một cấu trúc dữ liệu,
mà là một "hộp công cụ" thao tác **trên** các cấu trúc dữ liệu đã có: sắp xếp, tìm kiếm, đảo
ngược, và — dễ gây hiểu lầm nhất — "bọc" một collection thường thành phiên bản
đồng bộ hoá hoặc bất biến.

Từ "bọc" (wrap) chính là chìa khoá của chapter này. `Collections.synchronizedList(list)`
không tạo ra một cấu trúc dữ liệu mới an toàn tuyệt đối — nó chỉ thêm một lớp khoá **quanh mỗi
lời gọi method riêng lẻ**. Điều này nghe an toàn, nhưng ẩn chứa một cạm bẫy tinh vi mà rất
nhiều lập trình viên khám phá ra theo cách khó chịu nhất: giữa production.

## Interview Question (Central)

> `Collections.synchronizedList()` có làm cho việc duyệt (`for-each`) một list trở nên an toàn
> tuyệt đối với nhiều thread không?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Dùng thành thạo các method tĩnh phổ biến: `sort()`, `reverse()`, `max()`/`min()`,
      `binarySearch()`, `unmodifiableXxx()`, `synchronizedXxx()`
- [ ] Hiểu `unmodifiableList()` chỉ là một **view chặn ghi**, không phải bản sao bất biến thật
      sự — vẫn thay đổi theo nếu list gốc bị sửa
- [ ] Tự tay tái hiện được cạm bẫy: `synchronizedList()` không bảo vệ được vòng lặp `for-each`
- [ ] Biết `binarySearch()` yêu cầu list đã được sắp xếp trước, nếu không kết quả không xác
      định

## Prerequisites

- Chapter 14 — hiểu `Comparable`/`Comparator`, dùng bởi `Collections.sort()`.

## Used Later

- **Immutable Collection** (Chapter 16) — `Collections.unmodifiableXxx()` (Java 1.2) là tiền
  thân của `List.of()`/`Map.of()` (Java 9+) — chapter kế tiếp so sánh trực tiếp hai thế hệ giải
  pháp cho cùng một nhu cầu.

## Problem

Rất nhiều thao tác phổ biến (sắp xếp, tìm max/min, tìm kiếm nhị phân, đảo ngược) áp dụng được
cho **mọi** implementation của `List`/`Collection`, không phụ thuộc cấu trúc bên dưới là gì.
Viết các thao tác này thành method của từng interface riêng biệt sẽ trùng lặp code; đặt chúng
vào một class tiện ích tĩnh dùng chung là giải pháp gọn hơn.

## Concept

`java.util.Collections` là class chứa các method `static` thao tác trên `Collection`/`List`/
`Map`, chia làm ba nhóm chính:

1. **Thuật toán**: `sort()`, `reverse()`, `shuffle()`, `max()`, `min()`, `binarySearch()`,
   `frequency()`.
2. **Wrapper bất biến**: `unmodifiableList()`, `unmodifiableSet()`, `unmodifiableMap()`.
3. **Wrapper đồng bộ hoá**: `synchronizedList()`, `synchronizedSet()`, `synchronizedMap()`.

## Why?

Nếu không có class tiện ích tập trung: mỗi implementation (`ArrayList`, `LinkedList`,
`HashSet`...) sẽ phải tự viết lại `sort()`/`binarySearch()` riêng, dù thuật toán hoàn toàn giống
nhau và chỉ cần thao tác qua interface chung (`List`, Chapter 02) là đủ. `Collections` tận
dụng đúng nguyên tắc lập trình hướng interface (Phase 1, Chapter 08) để viết thuật toán **một
lần**, áp dụng được cho **mọi** implementation tuân theo interface đó.

## How?

```java
List<Integer> nums = new ArrayList<>(List.of(5, 3, 8, 1, 9));
Collections.sort(nums);              // [1, 3, 5, 8, 9]
Collections.reverse(nums);           // [9, 8, 5, 3, 1]
Collections.max(nums);               // 9
Collections.binarySearch(sortedList, 8); // yêu cầu list ĐÃ sắp xếp trước

List<Integer> readOnly = Collections.unmodifiableList(nums); // VIEW, không phải bản sao
```

## Visualization

```
Collections.unmodifiableList(list)
        │
        ▼
   một VIEW chặn ghi TRỰC TIẾP qua chính view đó
        │
        ├── list.add(x) qua VIEW    → UnsupportedOperationException (chặn)
        └── list gốc bị sửa từ NƠI KHÁC → VIEW VẪN THẤY thay đổi đó
                                          (không phải bản sao độc lập,
                                           giống hệt bài học subList() ở Chapter 02)
```

## Example

**`unmodifiableList()` chặn sửa, nhưng chỉ là view:**

```java
import java.util.*;

public class CollectionsUtilDemo {
    public static void main(String[] args) {
        List<Integer> nums = new ArrayList<>(List.of(5, 3, 8, 1, 9));
        Collections.sort(nums);
        System.out.println("sort: " + nums);
        Collections.reverse(nums);
        System.out.println("reverse: " + nums);
        System.out.println("max: " + Collections.max(nums) + ", min: " + Collections.min(nums));
        System.out.println("binarySearch(8) tren list da sort tang: "
            + Collections.binarySearch(new ArrayList<>(List.of(1,3,5,8,9)), 8));

        List<Integer> unmod = Collections.unmodifiableList(nums);
        try {
            unmod.add(100);
        } catch (UnsupportedOperationException e) {
            System.out.println("unmodifiableList chan sua: " + e.getClass().getSimpleName());
        }
    }
}
```

Kết quả thật (JDK 17):

```
sort: [1, 3, 5, 8, 9]
reverse: [9, 8, 5, 3, 1]
max: 9, min: 1
binarySearch(8) tren list da sort tang: 3
unmodifiableList chan sua: UnsupportedOperationException
```

**Cạm bẫy `synchronizedList()`** — mỗi method riêng lẻ được khoá, nhưng vòng lặp `for-each`
gồm nhiều lời gọi (`hasNext()`, `next()`) thì **không**:

```java
List<Integer> syncList = Collections.synchronizedList(new ArrayList<>(List.of(1, 2, 3, 4)));
try {
    for (Integer i : syncList) {
        if (i == 2) syncList.remove(i);
    }
    System.out.println("Khong nem exception - syncList = " + syncList);
} catch (ConcurrentModificationException e) {
    System.out.println("synchronizedList VAN fail-fast khi duyet: " + e.getClass().getSimpleName());
}
```

Kết quả thật:

```
synchronizedList VAN fail-fast khi duyet: ConcurrentModificationException
```

`synchronizedList()` **không** biến vòng lặp `for-each` thành một khối nguyên tử — nó chỉ khoá
**từng lời gọi method riêng lẻ** (`add()`, `remove()`, `get()`...). Javadoc chính thức của
`Collections.synchronizedList()` thực ra đã ghi rõ: khi duyệt bằng iterator, người dùng **phải
tự** bọc `synchronized (list) { ... }` quanh toàn bộ vòng lặp — điều rất ít người đọc kỹ tới.

## Deep Dive

**Vì sao `Collections.synchronizedList()` không tự động bảo vệ luôn cả vòng lặp?** Vì làm vậy
đòi hỏi giữ khoá trong suốt **toàn bộ** thời gian duyệt (có thể rất lâu nếu xử lý phức tạp bên
trong vòng lặp) — chặn đứng mọi thread khác muốn thao tác trên list trong suốt thời gian đó,
một chi phí quá lớn để tự động áp đặt cho mọi trường hợp. Thay vào đó, `synchronizedList()`
chỉ khoá ở **phạm vi từng lời gọi method**, để lại quyết định "có cần khoá cả một khối thao
tác dài hay không" cho người dùng tự làm tường minh (`synchronized (list) { for (...) {...} }`)
— nếu thực sự cần. Đây chính là góc nhìn để so sánh với [Chapter 11 —
ConcurrentHashMap](11-concurrenthashmap.md): `ConcurrentHashMap` được thiết kế lại hoàn toàn
để tránh chính vấn đề này (khoá phạm vi nhỏ, không cần khoá thô toàn bộ cho từng thao tác),
trong khi `synchronizedList()` chỉ là một lớp bọc đơn giản quanh cấu trúc dữ liệu **không**
thread-safe, không giải quyết triệt để bài toán concurrency.

## Engineering Insight

**`Collections` là bằng chứng cho một giai đoạn "quá độ" trong lịch sử API tiện ích của Java.**
Nhiều method của nó (`unmodifiableList`, `synchronizedList`) ra đời từ Java 1.2 (1998), theo
đúng phong cách lập trình thời kỳ đó: một class tiện ích tĩnh, bọc quanh object đã có. Java 8+
(Stream API, Phase 4) và Java 9+ (`List.of()`, sẽ gặp ở Chapter 16) dần chuyển hướng sang phong
cách hàm/khai báo hơn (functional/declarative), gọn hơn và ít cạm bẫy hơn (ví dụ `List.of()`
luôn là bản sao bất biến thật, không phải view như `unmodifiableList()`). `Collections` vẫn còn
hữu dụng và được dùng rộng rãi, nhưng nó là lớp API "thế hệ trước" — hiểu rõ giới hạn của nó
(như cạm bẫy `synchronizedList` ở Example) là cần thiết để dùng đúng, không phải để tránh hoàn
toàn.

## Historical Note

```
Java 1.2 (1998)
    ↓
Collections ra đời cùng Collection Framework — sort/reverse/unmodifiableXxx/
synchronizedXxx là các công cụ chính cho thao tác tập thể trên collection
    ↓
Java 5 (2004) — Collections.checkedList()/checkedMap() bổ sung, hỗ trợ
kiểm tra kiểu runtime khi làm việc với code cũ chưa dùng Generics
    ↓
Java 8 (2014) — Stream API (Phase 4) dần thay thế nhiều use case của
Collections cho việc xử lý/biến đổi dữ liệu phức tạp hơn sort/reverse đơn thuần
    ↓
Java 9 (2017) — List.of()/Set.of()/Map.of() (Chapter 16) thay thế
unmodifiableXxx() cho nhu cầu tạo collection bất biến, với ngữ nghĩa rõ ràng
hơn (bản sao thật, không phải view)
```

## Myth vs Reality

- **Myth:** "`Collections.synchronizedList()` khiến mọi thao tác trên list, kể cả vòng lặp
  `for-each`, đều an toàn tuyệt đối với nhiều thread."
  **Reality:** Xem Example — chỉ từng lời gọi method riêng lẻ được khoá. Vòng lặp `for-each`
  (nhiều lời gọi `hasNext()`/`next()` liên tiếp) **không** được bảo vệ tự động, vẫn cần tự bọc
  `synchronized` thủ công quanh toàn bộ vòng lặp nếu cần an toàn tuyệt đối.

- **Myth:** "`Collections.unmodifiableList(list)` tạo ra một bản sao bất biến của `list`."
  **Reality:** Nó chỉ là một **view** chặn ghi trực tiếp — nếu `list` gốc bị sửa từ nơi khác,
  view vẫn phản ánh thay đổi đó, giống hệt bài học `subList()` ở Chapter 02.

## Common Mistakes

- **Tin tưởng tuyệt đối vào `synchronizedList()` mà không tự khoá thủ công khi duyệt** — đúng
  lỗi trọng tâm của chapter, dẫn tới `ConcurrentModificationException` khó tái hiện ổn định
  trong môi trường đa luồng thật.
- **Coi `unmodifiableList()` là cách "đóng băng" dữ liệu vĩnh viễn** — không đúng nếu vẫn còn
  tham chiếu tới list gốc có thể sửa được ở đâu đó.
- **Gọi `binarySearch()` trên một list chưa sắp xếp** — kết quả trả về hoàn toàn không xác
  định (không phải lỗi, chỉ là sai một cách âm thầm).

## Best Practices

- Nếu cần an toàn tuyệt đối cho toàn bộ một chuỗi thao tác (không chỉ một lời gọi đơn lẻ) trên
  `synchronizedList()`, tự bọc `synchronized (list) { ... }` quanh toàn bộ khối đó, đặc biệt
  khi duyệt bằng iterator/for-each.
- Cho nhu cầu dùng chung nhiều thread hiện đại, cân nhắc `CopyOnWriteArrayList` (Chapter 17)
  hoặc `ConcurrentHashMap`-tương-đương thay vì `synchronizedList()`.
- Cho nhu cầu tạo collection bất biến mới, ưu tiên `List.of()`/`Map.of()` (Chapter 16) thay vì
  `unmodifiableXxx()` — ngữ nghĩa rõ ràng hơn (bản sao thật).
- Luôn `sort()` trước khi gọi `binarySearch()`.

## Production Notes

**Vấn đề:** một service dùng `Collections.synchronizedList()` cho một danh sách dùng chung
giữa nhiều thread xử lý request, thỉnh thoảng ném `ConcurrentModificationException` dưới tải
cao, dù đã "làm đúng theo tài liệu" bọc `synchronizedList()`.

- **Triệu chứng:** lỗi chỉ xuất hiện dưới tải cao (nhiều thread cùng lúc), không tái hiện được
  ổn định trên máy dev với tải thấp.
- **Root cause:** đúng cạm bẫy đã học — code có đoạn duyệt bằng `for-each` (hoặc
  `Iterator`) trên `synchronizedList()` mà không tự khoá thêm quanh toàn bộ vòng lặp; một
  thread khác gọi `add()`/`remove()` (đã được khoá đúng ở mức method) đúng lúc thread kia đang
  duyệt giữa chừng, gây fail-fast kích hoạt.
- **Debug:** tìm mọi chỗ duyệt (`for-each`/`Iterator`) trên các collection được bọc bằng
  `Collections.synchronizedXxx()`.
- **Solution:** bọc tường minh `synchronized (list) { for (...) {...} }` quanh toàn bộ khối
  duyệt, hoặc đổi sang cấu trúc dữ liệu concurrent phù hợp hơn (`CopyOnWriteArrayList`, Chapter
  17, nếu đọc nhiều ghi ít).
- **Prevention:** đọc kỹ Javadoc của `Collections.synchronizedXxx()` — phần ghi chú về việc
  cần tự khoá khi duyệt thường bị bỏ qua; đưa quy tắc "luôn tự khoá khi duyệt synchronizedList"
  vào coding standard nội bộ.

## Debug Checklist

- [ ] `ConcurrentModificationException` chập chờn dù đã dùng `synchronizedList()`? → tìm chỗ
      duyệt bằng for-each/Iterator, kiểm tra có tự khoá `synchronized (list) {...}` quanh toàn
      bộ vòng lặp không.
- [ ] `binarySearch()` cho kết quả sai/không nhất quán? → xác nhận list đã được `sort()` trước
      đó chưa.
- [ ] Dữ liệu "bất biến" (qua `unmodifiableList()`) vẫn thay đổi? → kiểm tra list gốc có bị sửa
      từ một tham chiếu khác không.

## Source Code Walkthrough

`Collections.synchronizedList()` trả về một object `SynchronizedList` (implements `List`), mỗi
method được override để bọc `synchronized (mutex) { return list.method(...); }` — `mutex`
mặc định là chính object list được bọc. Việc iterator (`iterator()`) của `SynchronizedList`
**không** tự động khoá trong suốt vòng đời của nó là quyết định thiết kế có chủ đích (như đã
phân tích ở Deep Dive) — Javadoc yêu cầu người dùng tự làm điều đó bằng tay khi cần.

## Summary

`Collections` là class tiện ích tĩnh với ba nhóm method chính: thuật toán (`sort`, `reverse`,
`binarySearch`...), wrapper bất biến (`unmodifiableXxx()` — chỉ là **view** chặn ghi, không
phải bản sao), và wrapper đồng bộ hoá (`synchronizedXxx()` — chỉ khoá **từng lời gọi method
riêng lẻ**, không tự động bảo vệ cả một vòng lặp `for-each`, đã chứng minh bằng thực nghiệm).
Cho nhu cầu hiện đại, `List.of()` (Chapter 16) và `ConcurrentHashMap`-tương-đương
(`CopyOnWriteArrayList`, Chapter 17) thường là lựa chọn tốt hơn `unmodifiableXxx()`/
`synchronizedXxx()` — nhưng hiểu rõ giới hạn của thế hệ API cũ vẫn cần thiết vì chúng còn xuất
hiện rất phổ biến trong code hiện tại.

## Interview Questions

**Junior**

- Kể tên một vài method tĩnh phổ biến của class `Collections`.
- `Collections.unmodifiableList()` trả về bản sao hay view của list gốc?

**Mid**

- `Collections.synchronizedList()` có bảo vệ được vòng lặp `for-each` khỏi
  `ConcurrentModificationException` không? Giải thích.
- `binarySearch()` yêu cầu điều kiện gì trước khi gọi?

**Senior**

- Giải thích vì sao `Collections.synchronizedList()` không tự động khoá toàn bộ vòng lặp
  duyệt — đây là hạn chế thiết kế hay lựa chọn có chủ đích? Phân tích đánh đổi.
- So sánh `Collections.unmodifiableList()`/`synchronizedList()` (Java 1.2) với `List.of()`
  (Chapter 16, Java 9) và `CopyOnWriteArrayList` (Chapter 17) — vì sao thế hệ API mới ra đời dù
  API cũ vẫn hoạt động được?

## Exercises

- [ ] Chạy lại đúng ví dụ `CollectionsUtilDemo` ở trên.
- [ ] Tái hiện đúng cạm bẫy `synchronizedList()` — xác nhận `ConcurrentModificationException`
      vẫn xảy ra dù đã bọc `synchronizedList()`.
- [ ] Sửa lại ví dụ đó bằng cách tự bọc `synchronized (syncList) { ... }` quanh toàn bộ vòng
      lặp — xác nhận không còn lỗi (trong ngữ cảnh đơn luồng của ví dụ, việc bọc synchronized
      không thay đổi hành vi fail-fast của việc sửa trong lúc duyệt — thử nghiệm và tự rút ra
      kết luận đúng là `synchronized` bảo vệ khỏi *thread khác* xen vào, không bảo vệ khỏi việc
      chính bạn sửa list đang duyệt).

## Cheat Sheet

| Method | Việc gì | Lưu ý |
| --- | --- | --- |
| `Collections.sort(list)` | Sắp xếp (dùng `Comparable`/`Comparator`) | — |
| `Collections.binarySearch(list, key)` | Tìm kiếm nhị phân | List phải đã sắp xếp trước |
| `Collections.unmodifiableList(list)` | View chặn ghi | KHÔNG phải bản sao |
| `Collections.synchronizedList(list)` | Bọc khoá từng method | KHÔNG bảo vệ cả vòng lặp for-each |

## References

- Java SE API Documentation — `java.util.Collections`.
- Javadoc `Collections.synchronizedList()` — ghi chú về yêu cầu tự khoá khi duyệt.
