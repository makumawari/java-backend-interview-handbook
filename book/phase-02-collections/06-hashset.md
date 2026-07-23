---
tags:
  - Java
  - HashSet
  - Collections
---

# HashSet

> Phase: Phase 2 — Collections
> Chapter slug: `hashset`

## Metadata

```yaml
Chapter: HashSet
Phase: Phase 2 — Collections
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 60%
Prerequisites:
  - Chapter 05 — Set
Used Later:
  - HashMap (Chapter 08) — HashSet chỉ là lớp bọc mỏng quanh HashMap, học kỹ HashMap trước
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 05 — Source Code Walkthrough](05-set.md) đã hé lộ: `HashSet` "không tự cài
> đặt cấu trúc dữ liệu riêng — chỉ là lớp bọc mỏng quanh `HashMap`".

Đây là một khẳng định mạnh — kiểm chứng được trực tiếp, không chỉ là lời đồn. Dùng đúng kỹ
thuật Reflection đã quen thuộc từ Phase 1 (Chapter 19) và vừa dùng lại ở Chapter 03
(`ArrayList`), ta có thể "mở nắp" một `HashSet` ra và nhìn thấy chính xác nó cất giữ dữ liệu ở
đâu.

## Interview Question (Central)

> `HashSet` được hiện thực dựa trên cấu trúc dữ liệu nào bên dưới? Nếu nó chỉ là một lớp bọc,
> value thật sự lưu trong đó là gì?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Xác nhận bằng Reflection: field nội bộ của `HashSet` chính xác là một `HashMap`
- [ ] Hiểu vì sao value của `HashMap` bên trong luôn là một hằng số dùng chung, vô nghĩa
- [ ] Suy luận đúng: mọi đặc tính hiệu năng của `HashSet` (O(1) trung bình, phụ thuộc
      `hashCode()`) đều **thừa hưởng nguyên vẹn** từ `HashMap` (Chapter 08)
- [ ] Biết constructor `new HashSet<>(collection)` bên trong làm gì để ước lượng capacity ban
      đầu hợp lý

## Prerequisites

- Chapter 05 — hiểu `Set` là một đảm bảo hành vi, không phải một cấu trúc dữ liệu cụ thể.

## Used Later

- **HashMap** (Chapter 08) — vì `HashSet` chỉ là lớp bọc, toàn bộ cơ chế sâu (bucket, resize,
  treeify) thực sự thuộc về `HashMap` — chapter đó mới là nơi đào sâu, chapter này chỉ xác
  nhận mối quan hệ.

## Problem

JDK cần cung cấp một implementation `Set` nhanh (O(1) trung bình). Nhưng `HashMap` (sẽ học kỹ
ở Chapter 08) đã giải quyết đúng bài toán "tra cứu nhanh dựa trên `hashCode()`/`equals()`" —
viết một cấu trúc dữ liệu hoàn toàn mới, riêng biệt, chỉ để bỏ qua phần "value" của một `Map`
sẽ là trùng lặp code không cần thiết.

## Concept

`HashSet` không tự quản lý bucket/hash bên trong nó — nó giữ một field `private transient
HashMap<E,Object> map`, và mọi phần tử của `Set` thực chất là một **key** trong `HashMap` đó.
Value tương ứng luôn là cùng một object hằng số dùng chung (JDK gọi nó `PRESENT`) — bản thân
giá trị của `PRESENT` không quan trọng, nó chỉ đóng vai trò "có mặt" (khác `null` để phân biệt
với key không tồn tại).

## Why?

Nếu viết `HashSet` như một cấu trúc bucket/hash hoàn toàn riêng: phải duy trì **hai** hiện
thực song song của cùng một thuật toán (chèn, tra cứu, resize, treeify — Chapter 08) — gấp đôi
công sức bảo trì, gấp đôi khả năng có bug, và mọi cải tiến hiệu năng cho `HashMap` sẽ không tự
động áp dụng cho `HashSet` trừ khi sửa cả hai nơi. Bọc `HashMap` và "bỏ qua" value là cách tận
dụng tối đa code đã có, đảm bảo `HashSet` luôn thừa hưởng **mọi** cải tiến của `HashMap` miễn
phí.

## How?

```java
public class HashSet<E> ... {
    private transient HashMap<E,Object> map;
    private static final Object PRESENT = new Object();

    public HashSet() {
        map = new HashMap<>();
    }

    public boolean add(E e) {
        return map.put(e, PRESENT) == null; // "thêm vào Set" = "put key vào Map"
    }

    public boolean contains(Object o) {
        return map.containsKey(o); // "Set có chứa" = "Map có key"
    }
}
```

## Visualization

```
HashSet<String> set = new HashSet<>();
set.add("A");
       │
       ▼
BÊN TRONG: map.put("A", PRESENT)
       │
       ▼
HashMap thật (Chapter 08): bucket, hashCode("A"), spread, resize, treeify...
       │
       ▼
"A" được lưu làm KEY, PRESENT (hằng số dùng chung, vô nghĩa) làm VALUE
```

## Example

Xác nhận trực tiếp bằng Reflection — field `map` bên trong một `HashSet` thật sự là một
`HashMap`:

```java
import java.lang.reflect.Field;
import java.util.HashSet;

public class HashSetInternal {
    public static void main(String[] args) throws Exception {
        HashSet<String> set = new HashSet<>();
        set.add("a");

        Field mapField = HashSet.class.getDeclaredField("map");
        mapField.setAccessible(true);
        Object internalMap = mapField.get(set);

        System.out.println("HashSet ben trong thuc chat la: " + internalMap.getClass().getName());
    }
}
```

Kết quả thật (JDK 17, chạy với `--add-opens java.base/java.util=ALL-UNNAMED`):

```
HashSet ben trong thuc chat la: java.util.HashMap
```

Không còn nghi ngờ gì nữa — đây không phải suy luận từ tài liệu, mà là bằng chứng trực tiếp từ
chính JVM đang chạy.

## Deep Dive

**Vì sao value không đơn giản dùng `null` thay vì tạo hẳn một object `PRESENT`?** Vì `HashMap`
(Chapter 08) coi `null` là một giá trị **hợp lệ** cho value — `map.get(key)` trả về `null` có
thể có nghĩa là "value thực sự là `null`" **hoặc** "key không tồn tại", hai trường hợp không
phân biệt được chỉ bằng `get()` (phải gọi thêm `containsKey()` để chắc chắn — một cạm bẫy kinh
điển của `HashMap` sẽ gặp lại ở Chapter 08). Nếu `HashSet` dùng `null` làm value dùng chung,
nội bộ nó sẽ phải tự xử lý đúng sự mơ hồ này. Dùng một object `PRESENT` cụ thể, khác `null`,
loại bỏ hoàn toàn sự mơ hồ đó — value luôn "có mặt" theo đúng nghĩa đen khi key tồn tại.

## Engineering Insight

**`new HashSet<>(Collection c)` bên trong tính capacity ban đầu như thế nào — đây có phải chỉ
`new HashMap<>()` trần trụi không?** Không — constructor này **ước lượng trước** capacity dựa
trên kích thước của `c` truyền vào, tính theo công thức tương tự
`Math.max((int) (c.size() / 0.75f) + 1, 16)`, rồi tạo `HashMap` với capacity đó ngay từ đầu.
Đây là một tối ưu nhỏ nhưng thực dụng: tạo `HashSet` từ một collection đã biết trước kích thước
(như ở [Chapter 05 — Example](05-set.md), `new HashSet<>(input)`) tránh được nhiều lần resize
không cần thiết trong lúc chèn hàng loạt — đúng tinh thần tối ưu "biết trước, cấp đủ ngay từ
đầu" đã học ở [Chapter 03 — ArrayList](03-arraylist.md).

## Historical Note

`HashSet` tồn tại từ Java 2 (1998), cùng đợt với `HashMap`. Kiến trúc "bọc `Map`" này không
thay đổi qua các phiên bản — kể cả khi `HashMap` được đại tu lớn ở Java 8 (thêm treeify, sẽ học
ở Chapter 08), `HashSet` không cần sửa gì cả vì nó chỉ gọi qua `HashMap`, tự động thừa hưởng cải
tiến mà không cần đổi một dòng code nào trong chính `HashSet`.

## Myth vs Reality

- **Myth:** "`HashSet` có thuật toán hash/bucket riêng, độc lập với `HashMap`."
  **Reality:** Xem Example — `HashSet` không có bucket riêng, nó **là** một `HashMap` được bọc
  lại, chỉ ẩn đi phần value.

- **Myth:** "Vì `HashSet` là một cấu trúc riêng, cải tiến `HashMap` không ảnh hưởng tới nó."
  **Reality:** Ngược lại hoàn toàn — mọi thay đổi/tối ưu ở `HashMap` (Chapter 08) tự động áp
  dụng cho `HashSet`, vì `HashSet` chỉ đơn thuần gọi qua nó.

## Common Mistakes

- **Học `HashSet` và `HashMap` như hai chủ đề tách biệt, học thuộc lòng đặc tính hiệu năng
  riêng cho từng cái** — lãng phí công sức. Hiểu kỹ `HashMap` (Chapter 08) là hiểu luôn
  `HashSet`, không cần học lại từ đầu.
- **Không tận dụng constructor `new HashSet<>(knownSize)`** khi biết trước số lượng phần tử sẽ
  thêm — bỏ lỡ tối ưu tránh resize giống hệt bài học ở `ArrayList` (Chapter 03).

## Best Practices

- Xem hiệu năng và cơ chế của `HashSet` **chính là** hiệu năng và cơ chế của `HashMap`
  (Chapter 08) — học một lần, áp dụng cho cả hai.
- Khi tạo `HashSet` từ một collection đã biết kích thước, luôn dùng constructor nhận
  `Collection` thay vì tạo rỗng rồi `addAll()` — tận dụng ước lượng capacity tự động.

## Production Notes

Vì `HashSet` chỉ là lớp bọc, **mọi vấn đề production liên quan tới `HashSet`** (hash collision,
resize không hợp lý, tra cứu chậm do `hashCode()` sai) thực chất là vấn đề của `HashMap` bên
dưới — xem đầy đủ Production Notes ở [Chapter 08 — HashMap](08-hashmap.md), không nhắc lại ở
đây để tránh trùng lặp.

## Debug Checklist

- [ ] Nghi ngờ `HashSet` hoạt động chậm/sai? → chuyển hướng điều tra sang đúng cơ chế
      `HashMap` ở [Chapter 08](08-hashmap.md) — nguyên nhân gốc luôn nằm ở đó.
- [ ] Muốn xác nhận trực tiếp cấu trúc bên trong một `HashSet`? → dùng kỹ thuật Reflection y
      hệt Example, hữu ích khi debug sâu hoặc học tập.

## Source Code Walkthrough

Đã thực hiện trực tiếp ở Example: field `map` (kiểu `HashMap<E, Object>`) chính là toàn bộ
"cơ thể" của `HashSet`. Các method chính (`add`, `remove`, `contains`, `size`, `iterator`) đều
chỉ là một dòng uỷ quyền (delegate) sang method tương ứng của `map` — `HashSet.java` trong
OpenJDK chỉ dài khoảng 300 dòng, phần lớn là Javadoc, vì logic thật nằm hoàn toàn ở
`HashMap.java`.

## Summary

`HashSet` không phải một cấu trúc dữ liệu độc lập — nó là lớp bọc mỏng quanh một
`HashMap<E, Object>`, dùng phần tử `Set` làm key và một hằng số `PRESENT` dùng chung làm value
(đã xác nhận trực tiếp bằng Reflection). Mọi đặc tính hiệu năng, mọi cơ chế sâu (bucket, hash
spreading, resize, treeify) đều thừa hưởng nguyên vẹn từ `HashMap` — học kỹ Chapter 08 chính là
học kỹ cả hai. Constructor nhận `Collection` tự ước lượng capacity ban đầu hợp lý, tránh resize
thừa.

## Interview Questions

**Junior**

- `HashSet` được hiện thực dựa trên cấu trúc dữ liệu nào?
- Value bên trong `HashMap` mà `HashSet` bọc quanh là gì?

**Mid**

- Vì sao `HashSet` không dùng `null` làm value dùng chung mà phải tạo một object `PRESENT`
  riêng?
- Giải thích lợi ích của việc `HashSet` được thiết kế như một lớp bọc thay vì một cấu trúc dữ
  liệu độc lập.

**Senior**

- Nếu bạn phải tối ưu hiệu năng cho một `HashSet` chứa hàng triệu phần tử, bạn sẽ tập trung
  điều tra vào đâu, dựa trên hiểu biết rằng nó chỉ là lớp bọc quanh `HashMap`?

## Exercises

- [ ] Chạy lại đúng ví dụ `HashSetInternal` ở trên, xác nhận field `map` đúng là `HashMap`.
- [ ] Dùng kỹ thuật tương tự, kiểm tra field nội bộ của `TreeSet` (Chapter 05 đã nhắc tới) —
      xác nhận nó bọc quanh `TreeMap`.
- [ ] Đọc source `HashSet.add()` (qua IDE hoặc trực tuyến), đối chiếu với đoạn code ở phần
      How? — xác nhận đúng là một dòng gọi `map.put(e, PRESENT)`.

## Cheat Sheet

| HashSet | Thực chất là |
| --- | --- |
| `set.add(e)` | `map.put(e, PRESENT)` |
| `set.contains(e)` | `map.containsKey(e)` |
| `set.remove(e)` | `map.remove(e)` |
| `set.size()` | `map.size()` |
| Cơ chế bucket/resize/treeify | 100% thuộc về `HashMap` — xem Chapter 08 |

## References

- OpenJDK source — `java.base/java/util/HashSet.java`.
- Java SE API Documentation — `java.util.HashSet`.
