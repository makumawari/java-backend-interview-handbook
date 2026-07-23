---
tags:
  - Java
  - Set
  - Collections
---

# Set

> Phase: Phase 2 — Collections
> Chapter slug: `set`

## Metadata

```yaml
Chapter: Set
Phase: Phase 2 — Collections
Difficulty: ★★
Importance: ★★★
Interview Frequency: 50%
Prerequisites:
  - Chapter 01 — Collection Framework
  - Phase 1, Chapter 12-13 — equals()/hashCode()
Used Later:
  - HashSet (Chapter 06) — implementation chính, dựa trên HashMap
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Phase 1, Chapter 13](../phase-01-foundation/13-hashcode.md) — bạn đã tự tay chứng
> minh: thêm hai `Product` "giống hệt nhau" (theo `equals()`) vào `HashSet` chỉ giữ lại **một**
> nếu `equals()`/`hashCode()` được override đúng cặp.

Chapter đó tập trung vào hợp đồng `equals()`/`hashCode()`. Chapter này quay lại đúng hiện
tượng ấy, nhưng từ góc nhìn khác: **`Set` không phải một cấu trúc dữ liệu — nó là một lời hứa
về hành vi.** Lời hứa đó là: "không chứa hai phần tử mà `e1.equals(e2)` là `true`". Toàn bộ
việc "làm sao để giữ lời hứa đó" hoàn toàn phụ thuộc vào việc `equals()`/`hashCode()` của phần
tử được viết đúng hay sai — `Set` bản thân nó không có "phép màu" nào khác.

## Interview Question (Central)

> `Set` đảm bảo điều gì mà `Collection` không đảm bảo? Cơ chế đó phụ thuộc vào điều gì ở phía
> phần tử được thêm vào?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Biết `Set` đảm bảo duy nhất một điều: không chứa phần tử trùng lặp theo `equals()`
- [ ] Hiểu vì sao đảm bảo đó **hoàn toàn phụ thuộc** vào `equals()`/`hashCode()` của phần tử
- [ ] Phân biệt ba implementation chính: `HashSet` (không thứ tự), `LinkedHashSet` (thứ tự
      thêm vào), `TreeSet` (thứ tự sắp xếp)
- [ ] Biết chọn đúng implementation dựa trên nhu cầu về thứ tự

## Prerequisites

- Chapter 01 — hiểu `Set` là một nhánh của `Collection`.
- Phase 1, Chapter 12-13 — hiểu hợp đồng `equals()`/`hashCode()`.

## Used Later

- **HashSet** (Chapter 06) — implementation chính, chapter kế tiếp mở nắp ra xem nó thực chất
  dùng gì bên dưới.

## Problem

`List` (Chapter 02) cho phép trùng lặp — hợp lý cho danh sách giao dịch, danh sách bước xử lý.
Nhưng nhiều bài toán cần ngược lại: một tập hợp **ID sản phẩm đã xử lý** (không nên xử lý trùng
hai lần), một tập **email đã đăng ký** (không được trùng). Dùng `List` cho các bài toán này
buộc phải tự viết logic kiểm tra trùng lặp thủ công (`if (!list.contains(x)) list.add(x)`) mỗi
lần thêm — dễ quên, dễ sai.

## Concept

`Set` mở rộng `Collection`, thêm đúng **một** đảm bảo: không chứa hai phần tử `e1`, `e2` sao
cho `e1.equals(e2)` là `true`. Không có method mới nào so với `Collection` — `Set` không thêm
API, nó chỉ thêm **một ràng buộc hành vi** lên các method đã có sẵn (`add()` sẽ không thêm nếu
đã tồn tại phần tử "bằng" nó).

## Why?

Đảm bảo "không trùng lặp" chỉ có ý nghĩa nếu có một định nghĩa rõ ràng về "trùng lặp là gì" —
và Java đã có sẵn định nghĩa đó: hợp đồng `equals()` (Phase 1, Chapter 12). `Set` không tự phát
minh ra khái niệm "bằng nhau" riêng — nó **tái sử dụng** đúng hợp đồng đã có, nhất quán với
toàn bộ hệ sinh thái Collection Framework (Chapter 01). Đây là lý do một `Set<Product>` chỉ
"đúng" nếu `Product.equals()`/`hashCode()` được viết đúng — nếu không, `Set` sẽ "giữ lời hứa
sai" (cho phép trùng lặp mà bạn tưởng đã bị chặn, hoặc ngược lại).

## How?

Ba implementation chính, khác nhau về **thứ tự** khi duyệt (đảm bảo "không trùng lặp" giống
hệt nhau ở cả ba):

- **`HashSet`** — không đảm bảo thứ tự nào cả (dựa trên `HashMap`, Chapter 06/08). Nhanh nhất
  cho `add()`/`contains()` (trung bình O(1)).
- **`LinkedHashSet`** — giữ đúng **thứ tự thêm vào** (dựa trên `LinkedHashMap`, sẽ gặp lại ở
  Chapter 09). Chậm hơn `HashSet` một chút, đổi lại thứ tự dự đoán được.
- **`TreeSet`** — luôn giữ thứ tự **sắp xếp** (dựa trên `TreeMap`, Chapter 10, dùng
  Comparable/Comparator — Chapter 14). O(log n) thay vì O(1), đổi lại luôn có thứ tự.

## Visualization

```
Set (đảm bảo DUY NHẤT: không trùng lặp theo equals())
  │
  ├── HashSet         — không thứ tự, nhanh nhất       [Ch.06 — dựa trên HashMap]
  ├── LinkedHashSet    — thứ tự thêm vào                [dựa trên LinkedHashMap, Ch.09]
  └── TreeSet          — thứ tự sắp xếp                 [dựa trên TreeMap, Ch.10]
```

## Example

Xác nhận cả ba implementation cùng giữ đúng đảm bảo "không trùng lặp", chỉ khác thứ tự:

```java
import java.util.*;

public class SetDemo {
    public static void main(String[] args) {
        List<String> input = List.of("cam", "tao", "cam", "chuoi", "tao", "buoi");

        Set<String> hash = new HashSet<>(input);
        Set<String> linked = new LinkedHashSet<>(input);
        Set<String> tree = new TreeSet<>(input);

        System.out.println("HashSet (khong thu tu ro rang): " + hash);
        System.out.println("LinkedHashSet (thu tu them vao): " + linked);
        System.out.println("TreeSet (thu tu sap xep): " + tree);
    }
}
```

Kết quả thật (JDK 17):

```
HashSet (khong thu tu ro rang): [buoi, chuoi, cam, tao]
LinkedHashSet (thu tu them vao): [cam, tao, chuoi, buoi]
TreeSet (thu tu sap xep): [buoi, cam, chuoi, tao]
```

Cả ba đều loại đúng 2 phần tử trùng (`cam`, `tao` xuất hiện 2 lần trong input, chỉ còn 1 trong
mỗi Set) — đảm bảo "không trùng lặp" giống hệt nhau. Nhưng thứ tự duyệt khác hẳn nhau: `HashSet`
không theo quy luật rõ ràng (phụ thuộc hash — Chapter 06), `LinkedHashSet` giữ đúng thứ tự
`cam, tao, chuoi, buoi` xuất hiện lần đầu trong input, `TreeSet` sắp theo alphabet.

## Deep Dive

**Vì sao `Set` không có method `get(index)` như `List`?** Vì bản chất toán học của một "tập
hợp" (set, theo lý thuyết tập hợp) không có khái niệm "phần tử thứ mấy" — chỉ có "phần tử này
có thuộc tập hợp hay không" (`contains()`). Ngay cả `LinkedHashSet`/`TreeSet` có thứ tự xác
định khi **duyệt**, chúng vẫn không cung cấp truy cập theo chỉ số — vì thứ tự đó là hệ quả phụ
của cách hiện thực (danh sách liên kết cho thứ tự thêm vào, cây cho thứ tự sắp xếp), không phải
bản chất cốt lõi của khái niệm `Set`. Muốn phần tử thứ k của một `TreeSet`, phải tự duyệt qua
iterator — không có API tính trực tiếp như `ArrayList.get(index)`.

## Engineering Insight

**`Set` là một ví dụ điển hình của nguyên tắc "hành vi qua hợp đồng, không qua kiểm tra kiểu
dữ liệu cụ thể".** `Set.add(e)` không quan tâm `e` thuộc class nào — nó chỉ gọi `e.equals()`
và `e.hashCode()` (với `HashSet`) hoặc `compareTo()` (với `TreeSet`, Chapter 14) — hoàn toàn uỷ
quyền việc "trùng lặp là gì" cho chính đối tượng được thêm vào định nghĩa. Đây là ứng dụng trực
tiếp của Polymorphism (Phase 1, Chapter 08): `Set` không cần biết trước kiểu dữ liệu cụ thể để
hoạt động đúng, miễn là kiểu đó tuân thủ đúng hợp đồng `equals()`/`hashCode()`/`Comparable`.

## Historical Note

`Set` tồn tại từ Java 2 (1998, Collection Framework). `LinkedHashSet` được thêm ở Java 1.4
(2002, cùng đợt `LinkedHashMap`). Java 8 (2014) bổ sung `Set.of(...)` (bất biến, xem Chapter
16) — tương tự `List.of()` đã gặp ở Phase 1 Chapter 11, tạo Set bất biến trong một dòng thay
vì phải bọc `Collections.unmodifiableSet(new HashSet<>(...))`.

## Myth vs Reality

- **Myth:** "`Set` là một cấu trúc dữ liệu cụ thể, giống như 'một loại danh sách không trùng
  lặp'."
  **Reality:** `Set` là một **interface** biểu diễn một **đảm bảo hành vi** — bản thân nó
  không quy định cách hiện thực. Ba implementation chính (`HashSet`, `LinkedHashSet`,
  `TreeSet`) đạt được đảm bảo đó bằng ba cách hoàn toàn khác nhau, với đánh đổi hiệu năng khác
  nhau.

- **Myth:** "`Set` luôn không có thứ tự."
  **Reality:** Chỉ `HashSet` không có thứ tự rõ ràng. `LinkedHashSet` và `TreeSet` đều có thứ
  tự xác định, dự đoán được — xem Example.

## Common Mistakes

- **Dùng `HashSet` khi thực sự cần giữ thứ tự thêm vào** — nên dùng `LinkedHashSet`.
- **Thêm object vào `Set` mà chưa override `equals()`/`hashCode()` đúng cách** — dẫn thẳng tới
  lỗi đã học ở [Phase 1, Chapter 13](../phase-01-foundation/13-hashcode.md): tưởng đã loại
  trùng nhưng thực chất chưa (dùng nguyên hành vi mặc định của `Object`, chỉ so sánh tham
  chiếu).
- **Kỳ vọng `TreeSet` chấp nhận phần tử `null`** — khác `HashSet`/`LinkedHashSet` (cho phép
  một `null`), `TreeSet` ném `NullPointerException` khi thêm `null` vì cần so sánh
  (`compareTo()`) để xác định vị trí, không so sánh được với `null`.

## Best Practices

- Mặc định chọn `HashSet` (nhanh nhất) trừ khi thực sự cần thứ tự.
- Nếu cần thứ tự thêm vào (ví dụ: hiển thị lại đúng thứ tự người dùng chọn), dùng
  `LinkedHashSet`.
- Nếu cần luôn duyệt theo thứ tự sắp xếp, dùng `TreeSet`.
- Luôn đảm bảo class của phần tử override `equals()`/`hashCode()` đúng cặp trước khi dùng làm
  phần tử của bất kỳ `Set` nào.

## Production Notes

Chapter này thuần khái niệm interface — các vấn đề production cụ thể (hash collision, resize,
tra cứu sai do `equals()`/`hashCode()` sai) đã được bàn kỹ ở
[Phase 1, Chapter 12-13](../phase-01-foundation/12-equals.md) và sẽ tiếp tục ở
[Chapter 06 — HashSet](06-hashset.md) với ngữ cảnh cụ thể hơn (bucket, resize) — không nhắc
lại ở đây để tránh trùng lặp.

## Debug Checklist

- [ ] `Set` chứa phần tử "trùng lặp" mà bạn nghĩ đã bị chặn? → kiểm tra `equals()`/`hashCode()`
      của class phần tử — xem lại [Phase 1, Chapter 13](../phase-01-foundation/13-hashcode.md).
- [ ] Cần thứ tự cụ thể khi duyệt `Set`? → xác nhận đang dùng đúng implementation
      (`LinkedHashSet`/`TreeSet`), không phải `HashSet`.
- [ ] `NullPointerException` khi thêm vào `TreeSet`? → `TreeSet` không chấp nhận `null` (cần so
      sánh để định vị trí).

## Source Code Walkthrough

Cả `HashSet`, `LinkedHashSet` đều **không tự cài đặt cấu trúc dữ liệu riêng** — chúng chỉ là
lớp bọc mỏng quanh `HashMap`/`LinkedHashMap` tương ứng (sẽ xác nhận bằng Reflection thật ở
[Chapter 06](06-hashset.md)), dùng phần tử `Set` làm **key**, với value là một object hằng số
dùng chung, vô nghĩa (`PRESENT`). `TreeSet` tương tự, bọc quanh `TreeMap` (Chapter 10). Đây là
một ví dụ tái sử dụng code rất triệt để trong chính JDK: thay vì viết 3 cấu trúc dữ liệu Set từ
đầu, JDK tái dùng 3 cấu trúc Map đã có sẵn, chỉ "bỏ qua" phần value.

## Summary

`Set` mở rộng `Collection`, thêm đúng một đảm bảo: không chứa phần tử trùng lặp theo
`equals()` — một ràng buộc hành vi hoàn toàn dựa trên hợp đồng `equals()`/`hashCode()` đã học ở
Phase 1, không phải cơ chế riêng của `Set`. Ba implementation chính đánh đổi khác nhau về thứ
tự: `HashSet` (không thứ tự, nhanh nhất), `LinkedHashSet` (thứ tự thêm vào), `TreeSet` (thứ tự
sắp xếp, dựa trên `Comparable`/`Comparator` — Chapter 14). Cả ba, dưới lớp vỏ, đều là lớp bọc
mỏng quanh các implementation `Map` tương ứng.

## Interview Questions

**Junior**

- `Set` đảm bảo điều gì mà `Collection` không đảm bảo?
- Kể tên 3 implementation chính của `Set`, khác nhau ở điểm gì?

**Mid**

- Đảm bảo "không trùng lặp" của `Set` phụ thuộc vào điều gì ở phía phần tử được thêm vào?
- Vì sao `TreeSet` ném `NullPointerException` khi thêm `null` trong khi `HashSet` thì không?

**Senior**

- Giải thích vì sao `HashSet`/`LinkedHashSet`/`TreeSet` đều được hiện thực bằng cách bọc quanh
  `Map` tương ứng thay vì viết cấu trúc dữ liệu riêng — đây là ứng dụng của nguyên tắc thiết kế
  nào?

## Exercises

- [ ] Chạy lại đúng ví dụ `SetDemo` ở trên, xác nhận cả ba `Set` loại đúng phần tử trùng lặp
      nhưng cho thứ tự duyệt khác nhau.
- [ ] Viết một class `Product` chưa override `equals()`/`hashCode()`, thêm hai instance "giống
      hệt nhau" vào `HashSet` — xác nhận cả hai đều tồn tại (size = 2), liên hệ lại
      [Phase 1, Chapter 13](../phase-01-foundation/13-hashcode.md).
- [ ] Thử thêm `null` vào cả `HashSet` và `TreeSet` — xác nhận `HashSet` chấp nhận,
      `TreeSet` ném `NullPointerException`.

## Cheat Sheet

| Implementation | Thứ tự | Tốc độ `add()`/`contains()` | Dựa trên |
| --- | --- | --- | --- |
| `HashSet` | Không đảm bảo | O(1) trung bình | `HashMap` (Chapter 06) |
| `LinkedHashSet` | Thứ tự thêm vào | O(1) trung bình (chậm hơn HashSet chút) | `LinkedHashMap` (Chapter 09) |
| `TreeSet` | Thứ tự sắp xếp | O(log n) | `TreeMap` (Chapter 10) |

## References

- Java SE API Documentation — `java.util.Set`.
- Oracle: "The Java Tutorials — The Set Interface".
