---
tags:
  - Java
  - TreeMap
  - Collections
---

# TreeMap

> Phase: Phase 2 — Collections
> Chapter slug: `treemap`

## Metadata

```yaml
Chapter: TreeMap
Phase: Phase 2 — Collections
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 55%
Prerequisites:
  - Chapter 08 — HashMap
  - Chapter 14 — Comparable vs Comparator (xem trước nếu chưa học)
Used Later:
  - MVCC, Index (Phase 7 — Database) — B-Tree/cây cân bằng là nền tảng của index database
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 09](09-linkedhashmap.md) — `LinkedHashMap` giữ thứ tự **thêm vào**, nhưng
> đó là thứ tự bạn quyết định lúc gọi `put()`, không phải một quy luật nội tại của dữ liệu.

Giả sử bạn cần một danh sách giá sản phẩm, luôn hiển thị **theo thứ tự giá tăng dần**, bất kể
bạn thêm chúng vào theo thứ tự nào. `LinkedHashMap` không giúp được gì ở đây — nó chỉ nhớ thứ
tự bạn thêm, không tự sắp xếp. Bạn cũng cần tìm nhanh "sản phẩm rẻ nhất có giá ít nhất 300.000"
— với `HashMap`/`LinkedHashMap`, không có cách nào làm việc này ngoài duyệt qua **toàn bộ** và
tự so sánh.

`TreeMap` giải quyết cả hai bài toán cùng lúc — đánh đổi tốc độ O(1) của `HashMap` lấy O(log n)
để đổi lấy một khả năng mà `HashMap` không bao giờ có: **luôn giữ thứ tự sắp xếp**, và truy vấn
được các câu hỏi kiểu "cái gần nhất", "cái nhỏ hơn/lớn hơn X" — nhờ cấu trúc cây cân bằng đứng
sau nó.

## Interview Question (Central)

> `TreeMap` giữ được thứ tự sắp xếp bằng cấu trúc dữ liệu nào? Đánh đổi gì so với `HashMap`
> (Chapter 08) để có được khả năng đó?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Biết `TreeMap` dùng **Red-Black Tree** (cây đỏ-đen — chính cấu trúc đã gặp khi `HashMap`
      treeify bucket ở Chapter 08), không phải bucket/hash
- [ ] Dùng thành thạo các method điều hướng (`NavigableMap`): `firstKey()`, `floorKey()`,
      `ceilingKey()`, `headMap()`, `tailMap()`
- [ ] Giải thích vì sao mọi thao tác của `TreeMap` là O(log n), không phải O(1)
- [ ] Biết `TreeMap` sắp xếp dựa trên `Comparable`/`Comparator` (Chapter 14), không phải
      `hashCode()`

## Prerequisites

- Chapter 08 — đã biết Red-Black Tree xuất hiện khi bucket `HashMap` bị treeify; `TreeMap`
  dùng chính cấu trúc đó làm nền tảng chính, không phải trường hợp đặc biệt.

## Used Later

- **MVCC, Index** (Phase 7 — Database) — cây cân bằng (B-Tree, họ hàng gần với Red-Black Tree)
  là nền tảng của hầu hết index trong các hệ quản trị cơ sở dữ liệu quan hệ — hiểu `TreeMap` ở
  mức Java giúp dễ tiếp cận khái niệm index ở tầng database hơn nhiều.

## Problem

`HashMap` (Chapter 08) không giữ thứ tự nào cả — thứ tự phụ thuộc hoàn toàn vào bucket, không
đoán trước được, và **không có cách nào** hỏi "phần tử nhỏ nhất", "phần tử lớn hơn X gần nhất"
mà không duyệt toàn bộ O(n). Cần một cấu trúc dữ liệu tra cứu vẫn nhanh (không phải O(n)) nhưng
**luôn** giữ thứ tự sắp xếp và hỗ trợ các câu hỏi "định vị theo thứ tự".

## Concept

`TreeMap` cài đặt bằng **Red-Black Tree** — một loại cây nhị phân tìm kiếm (Binary Search Tree)
**tự cân bằng**: mỗi node có key lớn hơn mọi key ở cây con bên trái, nhỏ hơn mọi key ở cây con
bên phải, và cây tự điều chỉnh cấu trúc sau mỗi lần thêm/xoá để **chiều cao luôn xấp xỉ
log(n)** — đảm bảo không bao giờ suy biến thành một danh sách liên kết dài (điều có thể xảy ra
với cây nhị phân tìm kiếm thường, không tự cân bằng).

## Why?

Nếu dùng cây nhị phân tìm kiếm **không** tự cân bằng: thêm dữ liệu theo thứ tự đã sắp xếp sẵn
(rất phổ biến trong thực tế — ví dụ ID tăng dần) khiến cây suy biến thành một chuỗi dài như
danh sách liên kết, độ phức tạp tra cứu tệ đi thành O(n), mất hoàn toàn lợi thế của cây. Red-
Black Tree giải quyết bằng cách **tự cân bằng lại** (đổi màu node, xoay cây) sau mỗi lần
thêm/xoá — đảm bảo chiều cao luôn O(log n) trong **mọi** trường hợp, kể cả worst case, không
phụ thuộc thứ tự dữ liệu đưa vào.

## How?

```java
TreeMap<Integer, String> prices = new TreeMap<>();
prices.put(500000, "Giay");
prices.put(100000, "Ao thun");
prices.put(250000, "Quan jean");
prices.put(1000000, "Balo");
```

Dù thêm theo thứ tự bất kỳ, `TreeMap` luôn tự sắp xếp lại theo key — duyệt qua nó luôn cho ra
đúng thứ tự tăng dần, không cần gọi `sort()` gì thêm.

## Visualization

```
Cây đỏ-đen (Red-Black Tree) sau khi thêm 500000, 100000, 250000, 1000000:

                    (250000)
                   /         \
             (100000)      (500000)
                                  \
                               (1000000)

firstKey()          = 100000    (node trái nhất)
lastKey()            = 1000000   (node phải nhất)
floorKey(300000)     = 250000    (lớn nhất mà <= 300000)
ceilingKey(300000)   = 500000    (nhỏ nhất mà >= 300000)
```

## Example

Xác nhận toàn bộ bộ method `NavigableMap` — điểm khác biệt lớn nhất so với `HashMap`:

```java
import java.util.TreeMap;

public class TreeMapDemo {
    public static void main(String[] args) {
        TreeMap<Integer, String> prices = new TreeMap<>();
        prices.put(100000, "Ao thun");
        prices.put(500000, "Giay");
        prices.put(250000, "Quan jean");
        prices.put(1000000, "Balo");

        System.out.println("Sap xep tu dong: " + prices);
        System.out.println("firstKey: " + prices.firstKey());
        System.out.println("lastKey: " + prices.lastKey());
        System.out.println("floorKey(300000): " + prices.floorKey(300000));
        System.out.println("ceilingKey(300000): " + prices.ceilingKey(300000));
        System.out.println("headMap(500000): " + prices.headMap(500000));
        System.out.println("tailMap(500000): " + prices.tailMap(500000));
    }
}
```

Kết quả thật (JDK 17):

```
Sap xep tu dong: {100000=Ao thun, 250000=Quan jean, 500000=Giay, 1000000=Balo}
firstKey: 100000
lastKey: 1000000
floorKey(300000): 250000
ceilingKey(300000): 500000
headMap(500000): {100000=Ao thun, 250000=Quan jean}
tailMap(500000): {500000=Giay, 1000000=Balo}
```

Dù được thêm vào theo thứ tự `500000, 100000, 250000, 1000000`, kết quả duyệt luôn đúng thứ tự
tăng dần — và các method điều hướng (`floorKey`, `ceilingKey`, `headMap`, `tailMap`) trả lời
chính xác các câu hỏi "lân cận theo thứ tự" mà `HashMap` không thể làm được mà không duyệt toàn
bộ.

## Deep Dive

**`floorKey()`/`ceilingKey()` hoạt động thế nào trên cây, tại sao vẫn O(log n)?** Bắt đầu từ
gốc cây, tại mỗi node, so sánh key đang tìm với key của node hiện tại: nếu nhỏ hơn, đi sang
trái (lưu lại node hiện tại như một ứng viên "ceiling" tiềm năng); nếu lớn hơn, đi sang phải
(lưu lại như ứng viên "floor"); cứ thế cho tới khi tới node lá — độ sâu tối đa của đường đi này
chính là chiều cao cây, luôn O(log n) nhờ tính chất tự cân bằng. Đây là lý do `TreeMap` trả lời
được các câu hỏi "lân cận theo thứ tự" nhanh gần như tra cứu chính xác, không cần duyệt tuyến
tính.

## Engineering Insight

**`TreeMap` và bucket đã treeify của `HashMap` (Chapter 08) dùng chung một cấu trúc — đây có
phải trùng hợp?** Không — đó là bằng chứng cho một nguyên lý sâu hơn: Red-Black Tree là cấu
trúc tối ưu cho **chính xác bài toán** "tra cứu + duyệt có thứ tự trong O(log n), đảm bảo worst
case, tự cân bằng" — bất kể ngữ cảnh sử dụng là gì. `HashMap` mượn nó để **chặn** worst case
O(n) khi có quá nhiều collision (Chapter 08); `TreeMap` dùng nó làm **cấu trúc chính**, không
phải giải pháp dự phòng, vì mục tiêu chính của `TreeMap` ngay từ đầu đã là "luôn có thứ tự" —
một mục tiêu mà `HashMap` không bao giờ theo đuổi. Cùng một công cụ toán học, hai vai trò khác
nhau tuỳ vào bài toán cần giải.

## Historical Note

```
Java 2 (1998)
    ↓
TreeMap ra đời, dùng Red-Black Tree ngay từ đầu — SỚM HƠN 16 năm so với
lúc HashMap mượn cùng cấu trúc này cho treeify (Java 8, 2014)
```

Red-Black Tree không phải phát minh riêng của Java — nó là một cấu trúc dữ liệu kinh điển trong
khoa học máy tính (mô tả lần đầu bởi Rudolf Bayer năm 1972, dưới tên khác, được đặt tên "đỏ-đen"
bởi Leonidas J. Guibas và Robert Sedgewick năm 1978). `TreeMap` là một trong những chỗ Java áp
dụng sớm nhất và trực tiếp nhất cấu trúc dữ liệu kinh điển này vào thư viện chuẩn.

## Myth vs Reality

- **Myth:** "`TreeMap` luôn là lựa chọn 'an toàn hơn' `HashMap` vì có thứ tự, không mất gì cả."
  **Reality:** Mọi thao tác `TreeMap` là O(log n), chậm hơn O(1) trung bình của `HashMap`
  (Chapter 08) — có thứ tự là một tính năng có giá, không phải "miễn phí".

- **Myth:** "`TreeMap` sắp xếp dựa trên `hashCode()`, giống `HashMap`."
  **Reality:** `TreeMap` hoàn toàn không dùng `hashCode()` — nó sắp xếp dựa trên
  `Comparable.compareTo()` (nếu key tự implement) hoặc `Comparator` truyền vào constructor
  (Chapter 14).

## Common Mistakes

- **Dùng key không implement `Comparable` và không truyền `Comparator`** — ném
  `ClassCastException` ngay khi `put()` phần tử thứ hai (không so sánh được).
- **Chọn `TreeMap` mặc định "cho chắc" thay vì `HashMap`** dù không thực sự cần thứ tự — trả
  giá O(log n) không cần thiết cho một lợi ích không dùng tới.
- **Quên `TreeMap` không chấp nhận key `null`** (đã nhắc ở [Chapter 07](07-map.md)) — vì cần so
  sánh để định vị trí trong cây, không so sánh được với `null`.

## Best Practices

- Chỉ chọn `TreeMap` khi thực sự cần thứ tự sắp xếp hoặc các truy vấn "lân cận"
  (`floorKey`/`ceilingKey`...) — nếu chỉ cần tra cứu nhanh, dùng `HashMap`.
- Đảm bảo key implement `Comparable` nhất quán với `equals()` (một quy tắc quan trọng ở
  Chapter 14: "consistent with equals") trước khi dùng làm key `TreeMap`.

## Production Notes

Chapter này thuần cấu trúc dữ liệu — ứng dụng thực tế phổ biến nhất của `TreeMap` trong
production là các bài toán liên quan tới **khoảng giá trị** (range query): tìm đơn hàng trong
khoảng thời gian, tìm mức giá gần nhất trong một bảng giá bậc thang (tiered pricing). Vấn đề
production thường gặp không phải lỗi của `TreeMap` mà là **chọn nhầm** nó khi không cần thiết,
gây chậm không đáng có cho những bài toán chỉ cần tra cứu đơn thuần — xem lại Best Practices.

## Debug Checklist

- [ ] `ClassCastException` khi `put()` vào `TreeMap`? → key chưa implement `Comparable` và
      không có `Comparator` truyền vào constructor.
- [ ] `NullPointerException` khi `put(null, ...)` vào `TreeMap`? → `TreeMap` không chấp nhận
      key `null`, khác `HashMap`.
- [ ] `TreeMap` chậm hơn mong đợi cho tra cứu đơn thuần? → cân nhắc có thực sự cần thứ tự
      không, nếu không, đổi sang `HashMap`.

## Source Code Walkthrough

`TreeMap` cài đặt interface `NavigableMap` (mở rộng `SortedMap`) — các method như
`floorEntry()`, `ceilingEntry()`, `pollFirstEntry()`, `pollLastEntry()` đều thao tác trực tiếp
trên cấu trúc cây (`Entry` với con trỏ `left`/`right`/`parent`, giống cấu trúc `TreeNode` đã
gặp khi `HashMap` treeify ở Chapter 08). Việc thêm/xoá node kích hoạt các thao tác cân bằng lại
cây (`fixAfterInsertion()`/`fixAfterDeletion()`) — đổi màu node và xoay cây (rotate) theo đúng
thuật toán Red-Black Tree kinh điển, đảm bảo chiều cao luôn O(log n) sau mỗi thao tác.

## Summary

`TreeMap` dùng **Red-Black Tree** — cùng cấu trúc mà `HashMap` (Chapter 08) mượn khi treeify
bucket quá tải — để luôn giữ thứ tự sắp xếp (dựa trên `Comparable`/`Comparator`, Chapter 14),
đổi lại mọi thao tác là O(log n) thay vì O(1) trung bình của `HashMap`. Bộ method
`NavigableMap` (`firstKey`, `floorKey`, `ceilingKey`, `headMap`, `tailMap`) trả lời được các
câu hỏi "lân cận theo thứ tự" mà `HashMap` không bao giờ làm được hiệu quả. Chỉ chọn `TreeMap`
khi thực sự cần thứ tự — nếu không, cái giá O(log n) là không cần thiết.

## Interview Questions

**Junior**

- `TreeMap` khác `HashMap` ở điểm cơ bản nào?
- `TreeMap` sắp xếp dựa trên tiêu chí gì?

**Mid**

- Giải thích `floorKey()`/`ceilingKey()` làm gì, cho ví dụ cụ thể.
- Vì sao mọi thao tác của `TreeMap` là O(log n) thay vì O(1)?

**Senior**

- Giải thích vì sao Red-Black Tree cần "tự cân bằng" — điều gì xảy ra nếu dùng cây nhị phân
  tìm kiếm thường (không tự cân bằng) và dữ liệu đưa vào đã có thứ tự sẵn?
- So sánh vai trò của Red-Black Tree trong `TreeMap` với vai trò của nó khi `HashMap` treeify
  một bucket (Chapter 08) — cùng cấu trúc, khác mục đích thế nào?

## Exercises

- [ ] Chạy lại đúng ví dụ `TreeMapDemo` ở trên, xác nhận kết quả giống hệt.
- [ ] Thử `put(null, "x")` vào một `TreeMap`, xác nhận ném `NullPointerException`; so sánh với
      `HashMap` (Chapter 07) chấp nhận `null` key bình thường.
- [ ] Viết một `TreeMap<Product, Integer>` với `Product` implement `Comparable` theo giá — xác
      nhận duyệt map luôn cho ra đúng thứ tự giá tăng dần bất kể thứ tự `put()`.

## Cheat Sheet

| Method | Ý nghĩa |
| --- | --- |
| `firstKey()`/`lastKey()` | Key nhỏ nhất/lớn nhất |
| `floorKey(k)` | Key lớn nhất mà ≤ k |
| `ceilingKey(k)` | Key nhỏ nhất mà ≥ k |
| `headMap(k)` | Mọi entry có key < k |
| `tailMap(k)` | Mọi entry có key ≥ k |

| So sánh | HashMap | TreeMap |
| --- | --- | --- |
| Cấu trúc | Bucket array + hash | Red-Black Tree |
| Thứ tự | Không | Luôn sắp xếp |
| Độ phức tạp | O(1) trung bình | O(log n) |
| Key `null` | Cho phép | Không cho phép |

## References

- Java SE API Documentation — `java.util.TreeMap`, `java.util.NavigableMap`.
- OpenJDK source — `java.base/java/util/TreeMap.java`.
