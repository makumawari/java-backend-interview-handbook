---
tags:
  - Java
  - List
  - Collections
---

# List

> Phase: Phase 2 — Collections
> Chapter slug: `list`

## Metadata

```yaml
Chapter: List
Phase: Phase 2 — Collections
Difficulty: ★★
Importance: ★★★
Interview Frequency: 55%
Prerequisites:
  - Chapter 01 — Collection Framework
Used Later:
  - ArrayList (Chapter 03), LinkedList (Chapter 04) — hai implementation chính của List
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 01](01-collection-framework.md) — `List` là một trong ba nhánh chính của
> `Collection`.

Bạn cần lấy ra 3 sản phẩm ở giữa một danh sách 10 sản phẩm để hiển thị trang 2 của kết quả tìm
kiếm. Viết:

```java
List<Product> page2 = products.subList(4, 7);
```

Gọn gàng — không cần vòng lặp copy thủ công. Nhưng thử thêm một sản phẩm mới vào `products`
(danh sách gốc) ngay sau đó, rồi dùng `page2`:

```java
products.add(newProduct);
page2.get(0); // ⚠️ ConcurrentModificationException
```

`page2` không phải một danh sách độc lập — nó là một **view** (cửa sổ nhìn) vào chính
`products`. Đây là một trong những cạm bẫy phổ biến nhất khi mới dùng `List`, và là điểm khởi
đầu tốt để hiểu `List` không chỉ là "mảng có thể co giãn" — nó có những đảm bảo và cạm bẫy
riêng cần nắm rõ trước khi học tới `ArrayList`/`LinkedList` (Chapter 03-04).

## Interview Question (Central)

> `List` đảm bảo những gì mà `Collection` (interface cha) không đảm bảo? `subList()` trả về
> một bản sao hay một view của list gốc?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Biết `List` đảm bảo thêm so với `Collection`: có thứ tự chèn cố định, cho phép trùng lặp,
      truy cập theo chỉ số (index)
- [ ] Dùng đúng các method đặc trưng của `List`: `get(index)`, `set(index, e)`, `indexOf()`,
      `subList()`
- [ ] Tự tay tái hiện được cạm bẫy `subList()` là view, không phải bản sao
- [ ] Biết khi nào nên chọn `List` thay vì `Set` (Chapter 05) cho cùng một bài toán

## Prerequisites

- Chapter 01 — hiểu `List` là một nhánh của `Collection`, hiểu fail-fast.

## Used Later

- **ArrayList** (Chapter 03) và **LinkedList** (Chapter 04) — hai implementation chính hiện
  thực hoá đầy đủ interface `List` học ở đây, mỗi cái đánh đổi hiệu năng khác nhau cho cùng một
  bộ hợp đồng.

## Problem

`Collection` chỉ đảm bảo "một nhóm phần tử", không đảm bảo gì về **thứ tự** hay **trùng lặp**.
Nhiều bài toán cần chính xác hai điều `Collection` không hứa hẹn: giữ đúng thứ tự đã thêm vào
(danh sách sản phẩm trong giỏ hàng, theo đúng thứ tự người dùng thêm), và cho phép trùng lặp
(hai sản phẩm khác nhau nhưng cùng tên vẫn là hai mục riêng biệt trong giỏ hàng).

## Concept

`List` mở rộng `Collection`, thêm ba đảm bảo cụ thể:

1. **Có thứ tự (ordered)** — phần tử giữ đúng thứ tự được thêm vào (khác `Set`, Chapter 05).
2. **Cho phép trùng lặp** — hai phần tử `equals()` nhau vẫn có thể cùng tồn tại.
3. **Truy cập theo chỉ số (index-based)** — `get(int index)`, `set(int index, E e)`,
   `indexOf(Object o)` — những method này không tồn tại ở `Collection` gốc.

## Why?

Nếu không có đảm bảo thứ tự: không thể biểu diễn đúng "lịch sử" hay "trình tự" (danh sách bước
xử lý đơn hàng, lịch sử giao dịch) — thứ tự chính là một phần ý nghĩa dữ liệu, không chỉ là chi
tiết cài đặt. Nếu không cho trùng lặp: không thể biểu diễn giỏ hàng có 2 sản phẩm giống hệt
nhau (`Set`, Chapter 05, sẽ coi chúng là một).

## How?

```java
List<String> tags = new ArrayList<>();
tags.add("giam-gia");      // index 0
tags.add("moi");           // index 1
tags.add("giam-gia");      // index 2 — TRÙNG LẶP, vẫn được thêm

System.out.println(tags.get(1));        // "moi" — truy cập theo index
System.out.println(tags.indexOf("moi")); // 1 — tìm index đầu tiên khớp equals()
tags.set(0, "hot");                      // thay thế phần tử tại index 0
```

## Visualization

```
List (có thứ tự, cho trùng lặp)
index:  0          1        2
       "giam-gia" "moi"   "giam-gia"   ← trùng "giam-gia" vẫn hợp lệ,
                                          thứ tự thêm vào được giữ nguyên
```

## Example

Tái hiện đúng cạm bẫy `subList()` ở Story:

```java
import java.util.*;

public class SubListDemo {
    public static void main(String[] args) {
        List<Integer> full = new ArrayList<>(List.of(1, 2, 3, 4, 5));
        List<Integer> sub = full.subList(1, 4); // view, KHONG phai list doc lap
        System.out.println("sub truoc: " + sub);

        sub.set(0, 99); // sua QUA sub
        System.out.println("full sau khi sua qua sub: " + full);

        try {
            full.add(100); // sua CAU TRUC full trong khi sub dang "song"
            sub.get(0);    // dung sub sau do
        } catch (ConcurrentModificationException e) {
            System.out.println("Dung sub sau khi sua full: " + e.getClass().getSimpleName());
        }
    }
}
```

Kết quả thật (JDK 17):

```
sub truoc: [2, 3, 4]
full sau khi sua qua sub: [1, 99, 3, 4, 5]
Dung sub sau khi sua full: ConcurrentModificationException
```

Hai điều xác nhận `subList()` là **view**, không phải bản sao: (1) sửa qua `sub` làm thay đổi
`full` ngay lập tức; (2) sửa **cấu trúc** `full` (thêm phần tử) trong khi `sub` vẫn còn tồn tại
khiến lần dùng `sub` tiếp theo ném `ConcurrentModificationException` — đúng cơ chế fail-fast đã
học ở [Chapter 01](01-collection-framework.md), vì `sub` bên trong cũng theo dõi `modCount` của
chính `full`.

## Deep Dive

**Vì sao `subList()` được thiết kế là view thay vì tự động copy?** Vì copy luôn tốn thêm bộ nhớ
và thời gian O(k) (k = kích thước đoạn con) — nếu chỉ cần **đọc** hoặc sửa tạm thời một đoạn
con rồi bỏ đi ngay, view rẻ hơn nhiều (O(1), không copy gì cả). Đánh đổi: người dùng phải tự ý
thức `sub` và `full` liên kết chặt với nhau. Muốn một bản sao thực sự độc lập, phải tự tạo
tường minh: `new ArrayList<>(full.subList(1, 4))`.

## Engineering Insight

**Vì sao `List` có cả `indexOf()` (tuyến tính, dựa trên `equals()`) lẫn `get(index)` (có thể
O(1) hoặc O(n) tuỳ implementation) mà không quy định tốc độ cụ thể trong interface?** Vì
`List` là một **hợp đồng hành vi**, không phải hợp đồng hiệu năng — `ArrayList` (Chapter 03) và
`LinkedList` (Chapter 04) đều thực hiện đúng `get(index)` trả về đúng phần tử tại vị trí đó,
nhưng một cái là O(1), một cái là O(n). Interface `List` cố tình không ép buộc độ phức tạp
thời gian, để mỗi implementation tự do đánh đổi theo đúng use case của nó — đây chính là lý do
Phase 2 dành nguyên hai chapter riêng (03, 04) để so sánh hai lựa chọn đánh đổi khác nhau cho
cùng một interface.

## Historical Note

`List` tồn tại từ Java 2 (1998, cùng đợt Collection Framework — Chapter 01). Thay đổi đáng chú
ý nhất qua thời gian không phải interface `List` (gần như không đổi) mà là các phương thức
mặc định (default method) được thêm vào Java 8: `replaceAll()`, `sort(Comparator)`,
`removeIf()` (kế thừa từ `Collection`) — cho phép các thao tác phổ biến mà trước đó phải tự viết
vòng lặp thủ công.

## Myth vs Reality

- **Myth:** "`subList()` trả về một danh sách mới, độc lập với danh sách gốc."
  **Reality:** Nó là một **view** — sửa qua view ảnh hưởng list gốc, và sửa cấu trúc list gốc
  có thể làm view "hỏng" (ném `ConcurrentModificationException` ở lần dùng tiếp theo).

- **Myth:** "Mọi implementation của `List` đều có `get(index)` nhanh như nhau."
  **Reality:** Interface chỉ đảm bảo **hành vi đúng**, không đảm bảo **tốc độ** — sẽ thấy rõ
  chênh lệch cụ thể ở Chapter 04 khi so sánh `ArrayList` với `LinkedList`.

## Common Mistakes

- **Coi `subList()` là bản sao** rồi ngạc nhiên khi sửa nó làm hỏng dữ liệu gốc.
- **Giữ một `subList()` sống lâu** trong khi vẫn tiếp tục sửa cấu trúc list gốc — dẫn tới
  `ConcurrentModificationException` khó dự đoán ở một chỗ dùng `subList()` xa nơi thực sự gây
  lỗi.

## Best Practices

- Nếu cần một đoạn con **độc lập** với list gốc, bọc tường minh:
  `new ArrayList<>(list.subList(from, to))`.
- Không giữ tham chiếu `subList()` lâu dài nếu list gốc còn tiếp tục bị sửa cấu trúc.

## Production Notes

**Vấn đề:** một tính năng phân trang (pagination) dùng `subList()` để cắt ra từng trang dữ
liệu, hoạt động đúng lúc test, nhưng gây `ConcurrentModificationException` ngẫu nhiên trong
production khi có luồng xử lý khác đồng thời thêm/xoá phần tử vào danh sách gốc.

- **Triệu chứng:** lỗi chỉ xảy ra khi có thao tác ghi xảy ra đúng lúc trang dữ liệu đang được
  dùng, khó tái hiện ổn định.
- **Root cause:** danh sách gốc là một `List` dùng chung (không phải bản sao riêng cho mỗi
  request), và `subList()` trả về view sống — bất kỳ sửa đổi cấu trúc nào trên danh sách gốc từ
  luồng khác đều làm mọi `subList()` đang tồn tại "hỏng theo".
- **Debug:** xác nhận danh sách gốc có bị chia sẻ giữa nhiều luồng/request không; nếu có, đây
  gần như chắc chắn là nguyên nhân.
- **Solution:** copy tường minh đoạn con cần dùng thay vì giữ nguyên view; hoặc nếu cần chia sẻ
  giữa nhiều luồng, cân nhắc cấu trúc thread-safe (sẽ học ở Chapter 11, 17).
- **Prevention:** không dùng `subList()` của một list còn tiếp tục bị sửa cấu trúc bởi luồng
  khác; nếu chỉ cần đọc một lần, copy ngay tại chỗ.

## Debug Checklist

- [ ] `ConcurrentModificationException` liên quan tới `subList()`? → kiểm tra list gốc có bị
      sửa cấu trúc (thêm/xoá) sau khi tạo `subList()` không.
- [ ] Sửa một "bản sao" nhưng dữ liệu gốc cũng đổi theo? → xác nhận có đang dùng `subList()`
      (view) thay vì bản sao thật.

## Source Code Walkthrough

`AbstractList.subList()` trả về một object `SubList` nội bộ — không chứa dữ liệu riêng, mà giữ
tham chiếu tới list gốc cùng offset (`fromIndex`) và độ dài. Mọi thao tác đọc/ghi trên
`SubList` (`get()`, `set()`...) được **uỷ quyền** lại cho list gốc, cộng thêm offset — đây là
lý do sửa qua `sub` luôn phản ánh ngay trên `full` ở Example: chúng luôn thao tác trên cùng một
vùng dữ liệu vật lý, không có bản sao nào tồn tại.

## Summary

`List` mở rộng `Collection`, thêm ba đảm bảo: có thứ tự, cho phép trùng lặp, truy cập theo
chỉ số. Interface chỉ ràng buộc **hành vi**, không ràng buộc **hiệu năng** — hai implementation
chính (`ArrayList`, `LinkedList`, Chapter 03-04) đánh đổi tốc độ khác nhau cho cùng bộ method.
`subList()` là một **view**, không phải bản sao — sửa qua nó ảnh hưởng list gốc, và sửa cấu
trúc list gốc có thể làm view đó ném `ConcurrentModificationException` ở lần dùng kế tiếp,
đúng cơ chế fail-fast đã học ở Chapter 01.

## Interview Questions

**Junior**

- `List` khác `Collection` (interface cha) ở những điểm nào?
- `subList()` trả về bản sao hay view?

**Mid**

- Tại sao interface `List` không quy định `get(index)` phải nhanh (O(1))?
- Cho một ví dụ cụ thể `subList()` gây lỗi nếu dùng sai cách.

**Senior**

- Giải thích tại sao thiết kế `subList()` làm view (không tự copy) là một lựa chọn hợp lý, dù
  dễ gây lỗi cho người dùng không cẩn thận — đánh đổi giữa hiệu năng và an toàn ở đây là gì?

## Exercises

- [ ] Chạy lại đúng ví dụ `SubListDemo` ở trên, xác nhận kết quả giống hệt.
- [ ] Viết đoạn code tạo một bản sao **độc lập** thật sự từ `subList()`, xác nhận sửa list gốc
      sau đó không ảnh hưởng tới bản sao.
- [ ] Viết một method nhận `List<Product>`, dùng cả `get(index)` và `indexOf()` — thử truyền
      vào cả `ArrayList` và `LinkedList` (Chapter 04, xem trước), xác nhận cùng chạy đúng dù
      hiệu năng có thể khác nhau.

## Cheat Sheet

| Method | Ý nghĩa |
| --- | --- |
| `get(int index)` | Lấy phần tử tại vị trí |
| `set(int index, E e)` | Thay thế phần tử tại vị trí |
| `indexOf(Object o)` | Tìm index đầu tiên khớp `equals()` |
| `subList(from, to)` | View của một đoạn con — KHÔNG phải bản sao |

| Đảm bảo | List | Collection (gốc) |
| --- | --- | --- |
| Giữ thứ tự thêm vào | Có | Không đảm bảo |
| Cho phép trùng lặp | Có | Tuỳ implementation |
| Truy cập theo index | Có | Không |

## References

- Java SE API Documentation — `java.util.List`.
- Oracle: "The Java Tutorials — The List Interface".
