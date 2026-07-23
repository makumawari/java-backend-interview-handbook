---
tags:
  - Java
  - Collections
  - Foundation
---

# Collection Framework

> Phase: Phase 2 — Collections
> Chapter slug: `collection-framework`

## Metadata

```yaml
Chapter: Collection Framework
Phase: Phase 2 — Collections
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 60%
Prerequisites:
  - Phase 1, Chapter 08 — OOP Fundamentals (Polymorphism, interface)
  - Phase 1, Chapter 12-13 — equals()/hashCode()
Used Later:
  - Toàn bộ Phase 2 — mọi chapter sau đều là một implementation cụ thể của interface học ở đây
Estimated Reading: 20 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

Trước Java 2 (1998), mỗi cấu trúc dữ liệu trong Java (`Vector`, `Hashtable`, mảng...) có API
**hoàn toàn riêng biệt**, không liên quan tới nhau. Muốn duyệt một `Vector` phải dùng
`Enumeration`; muốn duyệt một mảng phải dùng vòng lặp chỉ số. Một method nhận "một danh sách
sản phẩm" phải chọn cứng: nhận `Vector`, hay nhận mảng, hay nhận kiểu khác — không có cách nào
viết một method dùng chung cho tất cả.

Java 2 giới thiệu **Collection Framework** — không phải thêm một cấu trúc dữ liệu mới, mà là
một **bộ khung interface thống nhất** mà mọi cấu trúc dữ liệu (cũ lẫn mới) đều tuân theo. Đây
là chapter mở đầu Phase 2 — không đi sâu bất kỳ implementation cụ thể nào (đó là việc của 18
chapter còn lại), mà vẽ **bản đồ tổng thể**: những interface nào tồn tại, quan hệ giữa chúng ra
sao, và một chi tiết cực dễ gây nhầm lẫn: `Map` **không phải** một `Collection`.

## Interview Question (Central)

> Collection Framework được tổ chức thành những interface chính nào? Vì sao `Map` không kế
> thừa `Collection`, dù rõ ràng nó cũng là một "tập hợp dữ liệu"?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Vẽ được sơ đồ phân cấp interface chính: `Iterable` → `Collection` →
      `List`/`Set`/`Queue`, và `Map` đứng tách riêng
- [ ] Giải thích được vì sao `Map` không kế thừa `Collection`
- [ ] Hiểu cơ chế **fail-fast** của iterator, và biết nó là "best-effort", không phải đảm bảo
      tuyệt đối
- [ ] Biết chính xác lộ trình học Phase 2: `List` (Ch.02-04) → `Set` (Ch.05-06) → `Map`
      (Ch.07-11) → `Queue`/`Deque` (Ch.12-13) → công cụ hỗ trợ (Ch.14-19)

## Prerequisites

- Phase 1, Chapter 08 — hiểu interface, Polymorphism (một tham chiếu `List`, nhiều
  implementation `ArrayList`/`LinkedList` thật đứng sau).
- Phase 1, Chapter 12-13 — hiểu hợp đồng `equals()`/`hashCode()` — nền tảng bắt buộc cho
  `Set`/`Map` (Chapter 05 trở đi).

## Used Later

Toàn bộ 18 chapter còn lại của Phase 2 là các implementation cụ thể của interface học ở
chapter này — `List` (Chapter 02-04), `Set` (Chapter 05-06), `Map` (Chapter 07-11),
`Queue`/`Deque` (Chapter 12-13).

## Problem

Một method xử lý "danh sách sản phẩm" cần hoạt động được với **bất kỳ** cấu trúc dữ liệu nào
biểu diễn một danh sách — `ArrayList`, `LinkedList`, hay bất kỳ implementation nào chưa tồn
tại lúc method đó được viết. Nếu không có một interface chung, method phải khai báo cứng một
kiểu cụ thể, mất hoàn toàn khả năng thay đổi cấu trúc dữ liệu bên dưới sau này mà không sửa
lại chữ ký method.

## Concept

Collection Framework chia thành hai nhánh **không giao nhau**:

```
Iterable
   │
   ▼
Collection                              Map  (ĐỨNG RIÊNG, không kế thừa Collection)
   │
   ├── List   (có thứ tự, cho phép trùng, truy cập theo chỉ số)
   ├── Set    (không trùng, dựa trên equals()/hashCode() — Ch.12-13 Phase 1)
   └── Queue  (xử lý theo thứ tự FIFO/ưu tiên, gồm cả Deque — hai đầu)
```

`Collection` là interface gốc cho "một nhóm nhiều phần tử cùng kiểu". `Map` biểu diễn một khái
niệm khác về bản chất: **ánh xạ** từ key sang value — phần tử của nó là **cặp** key-value, không
phải một phần tử đơn lẻ như `List`/`Set`/`Queue`.

## Why?

Nếu `Map` kế thừa `Collection`: `Collection.add(E element)` nhận **một** tham số, nhưng thêm
một entry vào `Map` cần **hai** giá trị (key và value) — không khớp chữ ký method. Ép `Map` vào
khuôn `Collection` sẽ buộc phải "giả vờ" mỗi cặp key-value là một phần tử kiểu `Map.Entry`,
làm mất đi API tự nhiên nhất của `Map` (`get(key)`, `put(key, value)`) — đây chính là lý do
thiết kế Collection Framework tách `Map` ra làm một nhánh riêng ngay từ đầu, dù về mặt khái
niệm "tập hợp dữ liệu" thì `Map` và `Collection` có vẻ giống nhau.

## How?

`Iterable` là interface tối thiểu để một cấu trúc dữ liệu dùng được với vòng lặp `for-each` —
chỉ yêu cầu method `iterator()`. `Collection` mở rộng `Iterable`, thêm các thao tác chung:
`add()`, `remove()`, `size()`, `contains()`... Mọi iterator sinh ra từ `Collection` chuẩn của
JDK (`ArrayList`, `HashMap`...) đều có cơ chế **fail-fast**: nếu cấu trúc dữ liệu bị sửa đổi
(thêm/xoá phần tử) **trong lúc** đang duyệt bằng iterator (mà không qua chính iterator đó), lần
gọi `next()` tiếp theo sẽ ném `ConcurrentModificationException` — một cơ chế phát hiện lỗi sớm,
không phải cơ chế đồng bộ hoá thread-safe thực sự (sẽ phân biệt rõ ở Chapter 11
ConcurrentHashMap).

## Visualization

```
                    Iterable
                       │
              ┌────────┴────────┐
              ▼                 ▼
         Collection           (Map — KHÔNG kế thừa Iterable/Collection)
              │                 │
      ┌───────┼───────┐         ├── HashMap        [Ch.08]
      ▼       ▼       ▼         ├── LinkedHashMap   [Ch.09]
    List     Set    Queue       ├── TreeMap          [Ch.10]
      │       │       │         └── ConcurrentHashMap [Ch.11]
 ArrayList  HashSet  PriorityQueue
 [Ch.03]   [Ch.06]    [Ch.12]
LinkedList            Deque
 [Ch.04]              [Ch.13]
```

## Example

Xác nhận trực tiếp: `Map` không phải `Collection`, và cơ chế fail-fast hoạt động thật (chạy
được trên máy bạn):

```java
import java.util.*;

public class FrameworkDemo {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>();
        Set<Integer> set = new HashSet<>();
        Map<Integer, String> map = new HashMap<>();

        System.out.println("List la Collection? " + (list instanceof Collection));
        System.out.println("Set la Collection?  " + (set instanceof Collection));
        System.out.println("Map la Collection?  " + (map instanceof Collection));
    }
}
```

Kết quả thật (JDK 17):

```
List la Collection? true
Set la Collection?  true
Map la Collection?  false
```

**Fail-fast không phải đảm bảo tuyệt đối** — bằng chứng thực nghiệm gây bất ngờ:

```java
List<String> threeItems = new ArrayList<>(List.of("A", "B", "C"));
for (String s : threeItems) {
    if (s.equals("B")) threeItems.remove(s);
}
System.out.println("List 3 phan tu, xoa phan tu giua: " + threeItems + " (KHONG nem exception!)");

List<String> fourItems = new ArrayList<>(List.of("A", "B", "C", "D"));
try {
    for (String s : fourItems) {
        if (s.equals("B")) fourItems.remove(s);
    }
} catch (ConcurrentModificationException e) {
    System.out.println("List 4 phan tu, xoa phan tu giua: NEM " + e.getClass().getSimpleName());
}
```

Kết quả thật:

```
List 3 phan tu, xoa phan tu giua: [A, C] (KHONG nem exception!)
List 4 phan tu, xoa phan tu giua: NEM ConcurrentModificationException
```

Cùng một thao tác "xoá phần tử đang xét trong lúc for-each", nhưng list 3 phần tử **không** ném
exception còn list 4 phần tử **có** — bằng chứng cụ thể cho việc fail-fast chỉ là "best-effort"
(cố gắng phát hiện, không đảm bảo phát hiện được mọi trường hợp), không phải một cơ chế được
kiểm tra chặt chẽ sau mỗi thao tác.

## Deep Dive

**Vì sao fail-fast lại "bỏ sót" đúng trường hợp 3 phần tử ở Example?** Cơ chế fail-fast của
`ArrayList` dựa trên một bộ đếm nội bộ `modCount` (tăng mỗi khi cấu trúc bị sửa) — nhưng việc
kiểm tra `modCount` chỉ xảy ra bên trong `next()`, và vòng lặp `for-each` luôn gọi `hasNext()`
**trước** `next()`. Với list 3 phần tử `[A, B, C]`: sau khi xoá `B` ở vị trí index 1, kích
thước list giảm còn 2, con trỏ iterator (đang ở index 2 sau khi vừa đọc `B`) lúc này **bằng
đúng** kích thước mới — `hasNext()` trả về `false` **trước khi** `next()` (nơi kiểm tra
`modCount`) kịp chạy lần nữa, vòng lặp kết thúc êm đẹp, không ai phát hiện ra đã có sửa đổi.
Với 4 phần tử, phép tính không "khớp" trùng hợp như vậy, `next()` vẫn được gọi thêm một lần
nữa và bắt được `modCount` đã đổi. Đây không phải một tính năng, mà là hệ quả của cách hiện
thực — chính là lý do JDK luôn cảnh báo fail-fast "should be used only to detect bugs", không
nên dựa vào nó như một cơ chế đảm bảo.

## Engineering Insight

**Vì sao Collection Framework chọn thiết kế dựa trên interface (`List`, `Set`, `Map`) thay vì
class cụ thể ngay từ đầu?** Đây là ứng dụng trực tiếp nguyên tắc "lập trình hướng interface,
không hướng implementation" (Polymorphism, Phase 1 Chapter 08): khai báo biến kiểu
`List<Product> products = new ArrayList<>();` cho phép đổi sang `new LinkedList<>()` mà
**không cần sửa bất kỳ dòng code nào khác** dùng biến `products` — miễn là chỉ gọi qua các
method của interface `List`. Thiết kế này là lý do bạn có thể học lần lượt từng implementation
cụ thể ở các chapter sau (ArrayList, LinkedList, HashMap...) mà không phải học lại API — chúng
đều tuân theo cùng bộ interface đã học ở chapter này.

## Historical Note

```
Trước Java 2 (1998)
    ↓
Vector, Hashtable, Enumeration — mỗi cấu trúc dữ liệu một API riêng,
không có interface chung, không tương tác được với nhau một cách thống nhất
    ↓
Java 2 / J2SE 1.2 (1998)
    ↓
Collection Framework ra đời — Collection, List, Set, Map thống nhất.
Vector/Hashtable KHÔNG bị xoá (tương thích ngược) nhưng được "tái cấu trúc"
để implement List/Map mới, trở thành phiên bản đồng bộ hoá (synchronized)
cũ kỹ, dần bị thay thế bởi ArrayList/HashMap trong code hiện đại
    ↓
Java 5 (2004) — Generics (Phase 1 Chapter 16) áp dụng vào Collection Framework,
loại bỏ nhu cầu ép kiểu thủ công khi lấy phần tử ra
```

## Myth vs Reality

- **Myth:** "`Map` cũng là một loại `Collection`, chỉ là lưu theo cặp key-value."
  **Reality:** Xem Example — `map instanceof Collection` là `false`. `Map` là một nhánh hoàn
  toàn tách biệt trong Collection Framework, không kế thừa `Collection`/`Iterable`.

- **Myth:** "`ConcurrentModificationException` luôn được ném ra nếu sửa collection trong lúc
  duyệt."
  **Reality:** Đây là cơ chế **best-effort**, không phải đảm bảo tuyệt đối — xem bằng chứng cụ
  thể ở Example/Deep Dive.

## Common Mistakes

- **Dựa vào fail-fast như một cơ chế đảm bảo an toàn**, thay vì hiểu đúng bản chất "chỉ để
  phát hiện bug, không phải cơ chế đồng bộ hoá" — nhầm lẫn này dẫn thẳng tới
  [Chapter 11 — ConcurrentHashMap](11-concurrenthashmap.md), nơi phân biệt rạch ròi
  "phát hiện lỗi" (fail-fast) với "thực sự an toàn khi nhiều thread cùng sửa" (thread-safe).
- **Sửa collection trực tiếp trong lúc for-each** (như ở Example) thay vì dùng
  `Iterator.remove()` (an toàn, không vi phạm fail-fast) hoặc `removeIf()` (Java 8+, gọn và an
  toàn hơn).

## Best Practices

- Luôn khai báo biến bằng kiểu interface (`List`, `Set`, `Map`), không khai báo bằng kiểu class
  cụ thể (`ArrayList`, `HashMap`) — trừ khi thực sự cần method riêng của class đó.
- Muốn sửa collection trong lúc duyệt, dùng `Iterator.remove()` hoặc `Collection.removeIf()`,
  không sửa trực tiếp qua chính collection trong vòng lặp `for-each`.

## Production Notes

**Vấn đề:** một đoạn code xử lý danh sách chạy ổn định qua rất nhiều lần test, nhưng thỉnh
thoảng ném `ConcurrentModificationException` trong production mà không tái hiện được ổn định
trên máy dev.

- **Triệu chứng:** lỗi "chập chờn", phụ thuộc chính xác số lượng phần tử trong danh sách tại
  thời điểm chạy — đúng hiện tượng đã thấy ở Example (3 phần tử vs 4 phần tử cho kết quả khác
  nhau).
- **Root cause:** code sửa collection trực tiếp trong lúc for-each — hoạt động "may mắn" không
  lỗi với một số kích thước dữ liệu cụ thể trong môi trường test, nhưng dữ liệu production có
  kích thước khác khiến lỗi lộ ra.
- **Debug:** tìm mọi chỗ gọi `list.add()`/`list.remove()` trực tiếp bên trong một vòng lặp
  `for-each` trên chính list đó.
- **Solution:** đổi sang `Iterator.remove()` hoặc `removeIf()`.
- **Prevention:** static analysis (nhiều IDE đã cảnh báo sẵn mẫu hình này); code review chú ý
  đặc biệt tới việc sửa collection trong vòng lặp duyệt qua chính nó.

## Debug Checklist

- [ ] `ConcurrentModificationException` chập chờn, không tái hiện ổn định? → tìm chỗ sửa
      collection trực tiếp trong for-each, xem lại Example để hiểu vì sao nó "chập chờn" theo
      đúng kích thước dữ liệu.
- [ ] Cần biết một biến kiểu gì có phải `Collection` không? → `instanceof Collection`, nhớ
      `Map` luôn `false`.

## Source Code Walkthrough

`ArrayList.Itr` (lớp iterator nội bộ của `ArrayList`) giữ một field `expectedModCount`, khởi
tạo bằng `modCount` của list lúc iterator được tạo. Mỗi lần `next()` được gọi, nó so sánh
`modCount` hiện tại của list với `expectedModCount` — khác nhau thì ném
`ConcurrentModificationException` ngay trước khi trả về phần tử. Đúng như Deep Dive đã phân
tích, `hasNext()` **không** thực hiện kiểm tra này — chỉ so sánh `cursor != size`, đây chính là
kẽ hở gây ra hiện tượng "3 phần tử không nem exception, 4 phần tử có" ở Example.

## Summary

Collection Framework tổ chức thành hai nhánh tách biệt: `Iterable → Collection → List/Set/Queue`
(một nhóm phần tử đơn) và `Map` (ánh xạ key-value, đứng riêng, không kế thừa `Collection`) —
vì `Map` cần chữ ký method khác hẳn (`put(key, value)` hai tham số). Mọi iterator chuẩn của JDK
có cơ chế **fail-fast**, nhưng đây chỉ là best-effort dựa trên so sánh `modCount` trong
`next()` — không kiểm tra trong `hasNext()`, dẫn tới việc có thể "bỏ sót" phát hiện sửa đổi
trong một số trường hợp cụ thể (đã chứng minh bằng thực nghiệm). Phase 2 từ đây sẽ đi vào từng
implementation cụ thể của các interface đã vẽ bản đồ ở chapter này.

## Interview Questions

**Junior**

- Kể tên các interface chính trong Collection Framework.
- `Map` có phải là một `Collection` không?

**Mid**

- Fail-fast là gì? Nó có đảm bảo phát hiện được mọi trường hợp sửa đổi trong lúc duyệt không?
- Vì sao nên khai báo biến bằng kiểu interface (`List`) thay vì kiểu class cụ thể
  (`ArrayList`)?

**Senior**

- Giải thích chính xác vì sao xoá phần tử ở giữa một `ArrayList` 3 phần tử trong lúc for-each
  không ném `ConcurrentModificationException`, trong khi làm y hệt với 4 phần tử thì có — dựa
  trên cơ chế `modCount`/`hasNext()`/`next()`.
- Nếu được yêu cầu thiết kế lại Collection Framework từ đầu, bạn có giữ nguyên quyết định tách
  `Map` khỏi `Collection` không? Phân tích ưu/nhược điểm của quyết định đó.

## Exercises

- [ ] Chạy lại đúng ví dụ `FrameworkDemo` ở trên, xác nhận `map instanceof Collection` là
      `false`.
- [ ] Tái hiện đúng hiện tượng fail-fast "chập chờn" ở Example với list 3 và 4 phần tử, tự
      giải thích lại bằng lời của chính bạn dựa trên cơ chế `modCount`.
- [ ] Viết lại đoạn code xoá phần tử trong lúc duyệt bằng `Iterator.remove()` và bằng
      `removeIf()` — xác nhận cả hai cách đều không ném exception, bất kể kích thước list.

## Cheat Sheet

```
Iterable
   └── Collection
         ├── List    (có thứ tự, cho trùng, truy cập theo index)
         ├── Set     (không trùng, dựa trên equals()/hashCode())
         └── Queue   (FIFO/ưu tiên, gồm Deque hai đầu)

Map (ĐỨNG RIÊNG — không kế thừa Collection/Iterable)
```

| Muốn sửa trong lúc duyệt | Dùng |
| --- | --- |
| Xoá theo điều kiện | `Iterator.remove()` hoặc `Collection.removeIf()` |
| KHÔNG BAO GIỜ | Sửa trực tiếp collection bên trong for-each |

## References

- Java SE API Documentation — `java.util.Collection`, `java.util.Map`.
- Oracle: "The Java Tutorials — Trail: Collections".
- Effective Java (Joshua Bloch) — Item 11 vùng Collection, và các item về interface trong
  Chapter 4.
