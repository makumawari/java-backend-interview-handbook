---
tags:
  - Java
  - Map
  - Collections
---

# Map

> Phase: Phase 2 — Collections
> Chapter slug: `map`

## Metadata

```yaml
Chapter: Map
Phase: Phase 2 — Collections
Difficulty: ★★
Importance: ★★★★
Interview Frequency: 65%
Prerequisites:
  - Chapter 01 — Collection Framework (vì sao Map không kế thừa Collection)
  - Phase 1, Chapter 12-13 — equals()/hashCode()
Used Later:
  - HashMap (Chapter 08) — implementation trung tâm của toàn bộ handbook
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 01](01-collection-framework.md) — `Map` đứng tách riêng, không kế thừa
> `Collection`, vì `put(key, value)` cần hai tham số, không khớp chữ ký `add(E element)`.

Trước khi có `computeIfAbsent()` (Java 8), đếm số đơn hàng theo từng user phải viết:

```java
List<String> orders = ordersByUser.get(userId);
if (orders == null) {
    orders = new ArrayList<>();
    ordersByUser.put(userId, orders);
}
orders.add(newOrder);
```

Bốn dòng cho một thao tác rất phổ biến: "lấy giá trị, nếu chưa có thì tạo mới, rồi thao tác
tiếp". Chapter này không chỉ giới thiệu `Map` ở mức API cơ bản (`get`/`put`/`containsKey`) — nó
còn giới thiệu bộ method Java 8 (`computeIfAbsent`, `merge`, `getOrDefault`) đã biến đoạn code 4
dòng trên thành đúng một dòng.

## Interview Question (Central)

> `Map` khác `Collection` ở điểm cốt lõi nào? `key` trong `Map` đóng vai trò gì tương đương với
> phần tử trong `Set` (Chapter 05)?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Hiểu `Map` biểu diễn ánh xạ key → value, `key` phải duy nhất (tương tự đảm bảo của `Set`)
- [ ] Dùng thành thạo bộ method Java 8: `getOrDefault()`, `computeIfAbsent()`,
      `computeIfPresent()`, `merge()`
- [ ] Biết `Map.Entry` là gì, dùng để duyệt cả key lẫn value cùng lúc
- [ ] Biết các implementation `Map` khác nhau về việc có chấp nhận `null` key/value hay không

## Prerequisites

- Chapter 01 — đã hiểu vì sao `Map` không kế thừa `Collection`.
- Phase 1, Chapter 12-13 — hợp đồng `equals()`/`hashCode()`, áp dụng cho `key` của `Map` giống
  hệt cho phần tử của `Set` (Chapter 05).

## Used Later

- **HashMap** (Chapter 08) — implementation trung tâm, được dùng làm ví dụ xuyên suốt gần như
  toàn bộ phần còn lại của handbook (cache, đếm, nhóm dữ liệu...).

## Problem

Nhiều bài toán cần **tra cứu nhanh theo một khoá** — tìm sản phẩm theo ID, tìm user theo email,
đếm số lần xuất hiện của mỗi từ. Dùng `List` (Chapter 02) cho việc này đòi hỏi duyệt tuyến tính
O(n) mỗi lần tra cứu (`list.stream().filter(...)`) — không hiệu quả cho dữ liệu lớn hoặc tra
cứu thường xuyên.

## Concept

`Map<K, V>` biểu diễn một ánh xạ từ **key** (kiểu `K`) sang **value** (kiểu `V`). Giống hệt
`Set` (Chapter 05): **key phải duy nhất** — không có hai entry với key mà `k1.equals(k2)` là
`true`. Thực tế, `Map` cung cấp một method `keySet()` trả về đúng một `Set<K>` — bằng chứng
trực tiếp cho việc "tập hợp các key" tuân theo đúng đảm bảo của `Set`.

## Why?

Nếu không tách `Map` thành một khái niệm riêng: mọi bài toán "tra cứu theo khoá" sẽ phải tự mô
phỏng bằng cách kết hợp hai `List` song song (một cho key, một cho value) hoặc một `List<Pair>`
— mất đi khả năng tra cứu nhanh O(1) (Chapter 08) mà một cấu trúc chuyên biệt cho ánh xạ
key-value mới tận dụng được.

## How?

```java
Map<String, List<String>> ordersByUser = new HashMap<>();

// Trước Java 8 — 4 dòng
List<String> orders = ordersByUser.get("user1");
if (orders == null) {
    orders = new ArrayList<>();
    ordersByUser.put("user1", orders);
}
orders.add("Ao thun");

// Java 8+ — 1 dòng, cùng ý nghĩa
ordersByUser.computeIfAbsent("user1", k -> new ArrayList<>()).add("Ao thun");
```

`computeIfAbsent(key, fn)`: nếu `key` chưa tồn tại, tính giá trị mới bằng `fn` rồi lưu vào map
**và trả về** giá trị đó; nếu đã tồn tại, trả về giá trị hiện có, không gọi `fn`.

## Visualization

```
Map<K, V>
  key1 ──▶ value1
  key2 ──▶ value2
  key3 ──▶ value3

  keySet()   → Set<K>            (đảm bảo duy nhất, giống hệt Chapter 05)
  values()   → Collection<V>      (CHO PHÉP trùng lặp — nhiều key có thể trỏ cùng 1 value)
  entrySet() → Set<Map.Entry<K,V>> (duyệt cả key lẫn value cùng lúc)
```

## Example

Bộ method Java 8 giải quyết các bài toán "đếm" và "nhóm" cực gọn:

```java
import java.util.*;

public class MapDemo {
    public static void main(String[] args) {
        Map<String, List<String>> ordersByUser = new HashMap<>();
        ordersByUser.computeIfAbsent("user1", k -> new ArrayList<>()).add("Ao thun");
        ordersByUser.computeIfAbsent("user1", k -> new ArrayList<>()).add("Quan jean");
        ordersByUser.computeIfAbsent("user2", k -> new ArrayList<>()).add("Giay");
        System.out.println(ordersByUser);

        Map<String, Integer> wordCount = new HashMap<>();
        String[] words = {"cam", "tao", "cam", "buoi", "cam", "tao"};
        for (String w : words) {
            wordCount.merge(w, 1, Integer::sum); // dem so lan xuat hien
        }
        System.out.println(wordCount);

        System.out.println("getOrDefault(khongTonTai): " + wordCount.getOrDefault("xoai", 0));
    }
}
```

Kết quả thật (JDK 17):

```
{user1=[Ao thun, Quan jean], user2=[Giay]}
{buoi=1, tao=2, cam=3}
getOrDefault(khongTonTai): 0
```

`merge(key, 1, Integer::sum)`: nếu `key` chưa có, đặt value = `1`; nếu đã có, tính
`Integer.sum(giá trị cũ, 1)` — đúng ý nghĩa "đếm số lần xuất hiện" chỉ trong một dòng, không
cần `if/else` kiểm tra tồn tại thủ công.

## Deep Dive

**Không phải mọi implementation `Map` đều chấp nhận `null` key/value như nhau** — một chi tiết
dễ gây lỗi khi đổi implementation mà không kiểm tra kỹ:

```java
Map<String, String> hashMap = new HashMap<>();
hashMap.put(null, "gia tri cho null key"); // OK

Map<String, String> concurrent = new ConcurrentHashMap<>();
concurrent.put(null, "x"); // NullPointerException
```

Kết quả thật:

```
HashMap chap nhan null key: gia tri cho null key
ConcurrentHashMap KHONG chap nhan null key: NullPointerException
```

`HashMap` cho phép **một** key `null` (và nhiều value `null`). `ConcurrentHashMap` (Chapter 11)
**cấm tuyệt đối** cả key lẫn value `null` — lý do sẽ giải thích kỹ ở Chapter 11 (liên quan tới
sự mơ hồ giữa "value là null" và "key không tồn tại" trong ngữ cảnh đa luồng). `TreeMap`
(Chapter 10) cũng cấm key `null` (vì cần so sánh để sắp xếp, giống `TreeSet` ở Chapter 05).

## Engineering Insight

**Vì sao `computeIfAbsent()` lại quan trọng hơn một tiện ích cú pháp đơn thuần?** Vì trước Java
8, mẫu hình "get-check-null-put" (như đoạn code 4 dòng ở phần How?) không chỉ dài dòng — nó còn
tiềm ẩn lỗi tinh vi trong môi trường đa luồng: giữa lúc `get()` trả về `null` và lúc `put()`
được gọi, một thread khác có thể đã `put()` trước, khiến thread hiện tại **ghi đè mất** dữ liệu
vừa được thread kia thêm vào (một dạng lost-update, tương tự vấn đề sẽ minh hoạ rõ hơn ở
Chapter 11). `computeIfAbsent()` trên `ConcurrentHashMap` (Chapter 11) thực hiện toàn bộ thao
tác "kiểm tra + tạo mới nếu chưa có" một cách **nguyên tử** (atomic) — không chỉ gọn hơn về cú
pháp, mà còn an toàn hơn về bản chất trong ngữ cảnh đa luồng.

## Historical Note

`Map` tồn tại từ Java 2 (1998). Java 8 (2014) là cột mốc lớn nhất trong lịch sử interface này:
bổ sung một loạt default method (`getOrDefault`, `computeIfAbsent`, `computeIfPresent`,
`compute`, `merge`, `forEach`, `replaceAll`) — tất cả đều có thể viết lại bằng
`get`/`put`/`containsKey` cũ, nhưng giúp code ngắn hơn nhiều và (với `ConcurrentHashMap`) an
toàn hơn về nguyên tử tính.

## Myth vs Reality

- **Myth:** "Mọi `Map` đều cho phép key/value là `null`, giống `HashMap`."
  **Reality:** Xem Deep Dive — `ConcurrentHashMap` và `TreeMap` đều từ chối `null` key (và
  `ConcurrentHashMap` từ chối cả `null` value), khác hẳn `HashMap`.

- **Myth:** "`values()` của một `Map` cũng đảm bảo không trùng lặp, giống `keySet()`."
  **Reality:** Chỉ **key** phải duy nhất. `values()` trả về một `Collection` bình thường, cho
  phép trùng lặp — nhiều key khác nhau hoàn toàn có thể trỏ tới cùng một value.

## Common Mistakes

- **Dùng mẫu hình "get rồi kiểm tra null" thủ công** thay vì `getOrDefault()`/
  `computeIfAbsent()` — dài dòng và (với cấu trúc đa luồng) kém an toàn hơn.
- **Đổi từ `HashMap` sang `ConcurrentHashMap`/`TreeMap` mà không kiểm tra code có đang dùng
  `null` key/value không** — dẫn tới `NullPointerException` bất ngờ ở production sau khi
  "chỉ đổi một dòng khai báo kiểu".
- **Duyệt `Map` bằng `keySet()` rồi gọi `map.get(key)` trong vòng lặp** thay vì duyệt trực tiếp
  qua `entrySet()` — tốn thêm một lần tra cứu không cần thiết cho mỗi phần tử.

## Best Practices

- Ưu tiên `entrySet()` khi cần cả key lẫn value trong vòng lặp, không `keySet()` rồi `get()`
  lại.
- Dùng `computeIfAbsent()`/`merge()`/`getOrDefault()` thay cho mẫu hình get-check-null thủ
  công.
- Khi đổi implementation `Map`, luôn kiểm tra lại chính sách `null` key/value của implementation
  mới.

## Production Notes

**Vấn đề:** sau khi đổi từ `HashMap` sang `ConcurrentHashMap` để tăng tính an toàn đa luồng
(chuẩn bị cho Chapter 11), ứng dụng bắt đầu ném `NullPointerException` ở một nơi hoàn toàn
không liên quan tới thay đổi vừa thực hiện.

- **Triệu chứng:** exception xuất hiện ngay sau khi đổi kiểu khai báo, dù logic nghiệp vụ không
  đổi.
- **Root cause:** code cũ có chỗ dựa vào việc `HashMap` chấp nhận `null` (ví dụ dùng `null` làm
  giá trị "chưa xác định" hợp lệ) — `ConcurrentHashMap` từ chối thẳng thừng ngay khi `put()`.
- **Debug:** đọc kỹ stack trace, xác định chính xác `put()`/`get()` nào đang truyền `null`.
- **Solution:** thay `null` bằng một giá trị sentinel tường minh (ví dụ `Optional.empty()`
  hoặc một hằng số riêng biểu thị "chưa có"), hoặc dùng `containsKey()` để phân biệt "chưa có
  entry" thay vì dựa vào `null`.
- **Prevention:** khi thiết kế `Map` mới, tránh dùng `null` làm giá trị có ý nghĩa nghiệp vụ
  ngay từ đầu — giúp việc đổi implementation sau này an toàn hơn.

## Debug Checklist

- [ ] `NullPointerException` ngay sau khi đổi implementation `Map`? → kiểm tra implementation
      mới có cấm `null` key/value không (xem Deep Dive).
- [ ] `map.get(key)` trả về `null` — không rõ là "value thật sự null" hay "key không tồn
      tại"? → dùng `containsKey()` để phân biệt rõ, hoặc tránh dùng `null` làm value có ý
      nghĩa ngay từ đầu.

## Source Code Walkthrough

Cơ chế cụ thể của `get()`/`put()` (bucket, hash spreading, resize) thuộc về từng implementation
cụ thể — [Chapter 08 — HashMap](08-hashmap.md) sẽ đào sâu đầy đủ. Ở mức interface, đáng chú ý:
`computeIfAbsent()` trong `Map` (interface) có một implementation **mặc định** (default
method) viết bằng đúng mẫu hình get-check-null cũ — nhưng `HashMap`/`ConcurrentHashMap` đều
**override** lại bằng phiên bản tối ưu hơn (với `ConcurrentHashMap` là bản đảm bảo nguyên tử),
không dùng bản mặc định của interface.

## Summary

`Map<K, V>` biểu diễn ánh xạ key → value, với key phải duy nhất (đảm bảo giống `Set`, thể hiện
qua `keySet()` trả về đúng `Set<K>`). Bộ method Java 8 (`getOrDefault`, `computeIfAbsent`,
`merge`...) thay thế mẫu hình get-check-null-put thủ công bằng một dòng gọn gàng và (với cấu
trúc đa luồng) an toàn hơn. Không phải mọi implementation đều chấp nhận `null` key/value giống
nhau — `HashMap` cho phép, `ConcurrentHashMap`/`TreeMap` thì không, một chi tiết dễ gây lỗi khi
đổi implementation mà không kiểm tra kỹ.

## Interview Questions

**Junior**

- `Map` lưu trữ dữ liệu dưới dạng gì? Key có được phép trùng lặp không?
- `computeIfAbsent()` làm gì?

**Mid**

- Phân biệt `keySet()`, `values()`, `entrySet()`. Cái nào đảm bảo không trùng lặp?
- Vì sao `HashMap` cho phép `null` key nhưng `ConcurrentHashMap` thì không?

**Senior**

- Giải thích vì sao mẫu hình get-check-null-put thủ công có thể gây lost-update trong môi
  trường đa luồng, và `computeIfAbsent()` trên `ConcurrentHashMap` giải quyết vấn đề đó như
  thế nào (xem trước Chapter 11).
- Thiết kế một API dùng `Map` mà tránh hoàn toàn sự mơ hồ giữa "value null" và "key không tồn
  tại" — bạn sẽ làm gì?

## Exercises

- [ ] Chạy lại đúng ví dụ `MapDemo` ở trên.
- [ ] Viết lại đoạn code 4 dòng "get-check-null-put" ở phần How? bằng `computeIfAbsent()`, xác
      nhận kết quả giống hệt.
- [ ] Tái hiện đúng ví dụ ở Deep Dive: xác nhận `HashMap` chấp nhận `null` key nhưng
      `ConcurrentHashMap` ném `NullPointerException`.

## Cheat Sheet

| Method | Ý nghĩa |
| --- | --- |
| `getOrDefault(key, default)` | Lấy value, hoặc giá trị mặc định nếu key không tồn tại |
| `computeIfAbsent(key, fn)` | Nếu chưa có key, tính và lưu giá trị mới |
| `computeIfPresent(key, fn)` | Nếu đã có key, tính lại giá trị mới |
| `merge(key, value, fn)` | Kết hợp giá trị mới với giá trị cũ (nếu có) bằng `fn` |
| `entrySet()` | Duyệt cả key lẫn value cùng lúc |

| Implementation | Cho phép `null` key? | Cho phép `null` value? |
| --- | --- | --- |
| `HashMap` | Có (1 key) | Có |
| `LinkedHashMap` | Có | Có |
| `TreeMap` | Không | Có |
| `ConcurrentHashMap` | Không | Không |

## References

- Java SE API Documentation — `java.util.Map`.
- Oracle: "The Java Tutorials — The Map Interface".
