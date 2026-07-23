---
tags:
  - Java
  - LinkedHashMap
  - Collections
  - LRU
---

# LinkedHashMap

> Phase: Phase 2 — Collections
> Chapter slug: `linkedhashmap`

## Metadata

```yaml
Chapter: LinkedHashMap
Phase: Phase 2 — Collections
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 55%
Prerequisites:
  - Chapter 08 — HashMap
Used Later:
  - Redis, Cache (Phase 5, Phase 8) — LRU cache là mẫu hình caching phổ biến nhất trong production
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 08](08-hashmap.md) — bucket của `HashMap` được sắp xếp theo chỉ số tính
> từ `hashCode()`, hoàn toàn **không liên quan** tới thứ tự bạn `put()` các phần tử.

Duyệt một `HashMap` chứa `{apple, banana, cherry}`, thứ tự in ra hoàn toàn không đoán trước
được — có thể là `banana, apple, cherry`, có thể khác, tuỳ vào bucket mỗi key rơi vào. Với rất
nhiều bài toán thực tế (cache "gần đây nhất dùng", lịch sử thao tác theo đúng trình tự), thứ tự
**là một phần ý nghĩa của dữ liệu**, không thể chấp nhận bị xáo trộn ngẫu nhiên.

`LinkedHashMap` giải quyết đúng vấn đề này — không phải bằng cách viết lại thuật toán hash từ
đầu, mà bằng một bổ sung rất gọn: **giữ thêm một danh sách liên kết đôi xuyên suốt mọi entry**,
độc lập với vị trí bucket. Kết quả: giữ nguyên toàn bộ tốc độ O(1) của `HashMap`, cộng thêm khả
năng duyệt theo đúng thứ tự mong muốn — và với một tham số nhỏ trong constructor, nó còn trở
thành nền tảng cho một trong những cấu trúc dữ liệu được hỏi nhiều nhất trong phỏng vấn: **LRU
Cache**.

## Interview Question (Central)

> `LinkedHashMap` giữ được thứ tự (thêm vào hoặc truy cập gần nhất) bằng cách nào, mà vẫn giữ
> nguyên tốc độ O(1) của `HashMap`? Làm sao dùng nó để cài đặt một LRU Cache chỉ trong vài dòng?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích cơ chế: `LinkedHashMap` kế thừa `HashMap`, thêm một danh sách liên kết đôi
      xuyên suốt các entry
- [ ] Phân biệt hai chế độ: **insertion-order** (mặc định) và **access-order** (tham số thứ 3
      trong constructor)
- [ ] Tự tay cài đặt một LRU Cache hoàn chỉnh, chạy được, chỉ bằng cách override một method
- [ ] Biết khi nào chọn `LinkedHashMap` thay vì `HashMap` (Chapter 08) hoặc `TreeMap`
      (Chapter 10)

## Prerequisites

- Chapter 08 — hiểu trọn vẹn cơ chế bucket/hash của `HashMap`, vì `LinkedHashMap` kế thừa toàn
  bộ cơ chế đó.

## Used Later

- **Redis, Cache** (Phase 5, Phase 8) — LRU (Least Recently Used) là chiến lược cache phổ biến
  nhất trong hệ thống thực tế; `LinkedHashMap` là cách cài đặt LRU cache **trong bộ nhớ JVM**
  đơn giản nhất, trước khi cần tới hệ thống cache phân tán như Redis.

## Problem

`HashMap` (Chapter 08) tối ưu tuyệt đối cho tốc độ tra cứu, đánh đổi hoàn toàn việc bỏ thứ tự.
Nhưng nhiều bài toán cần **cả hai**: tra cứu nhanh O(1) **và** giữ được thứ tự — ví dụ một cache
giới hạn kích thước, cần biết chính xác "phần tử nào lâu chưa được dùng nhất" để loại bỏ khi
đầy.

## Concept

`LinkedHashMap extends HashMap` — nó **không** viết lại cơ chế bucket/hash, mà tái sử dụng
100% cơ chế đã học ở Chapter 08. Điểm khác biệt duy nhất: mỗi `Entry` trong `LinkedHashMap`
(kế thừa từ `Node` của `HashMap`) có thêm hai con trỏ `before`/`after`, tạo thành một **danh
sách liên kết đôi xuyên suốt toàn bộ entry**, độc lập hoàn toàn với vị trí bucket của chúng.
Khi duyệt (`entrySet()`, `keySet()`...), `LinkedHashMap` đi theo danh sách liên kết này, không
đi theo thứ tự bucket như `HashMap`.

## Why?

Nếu muốn thứ tự mà không có `LinkedHashMap`: phải tự duy trì một cấu trúc phụ (ví dụ một
`List<K>` riêng ghi lại thứ tự) song song với `HashMap` — tốn thêm bộ nhớ, và dễ để hai cấu
trúc "lệch pha" nhau nếu quên đồng bộ khi thêm/xoá. `LinkedHashMap` tích hợp thứ tự **ngay bên
trong** cấu trúc entry, đảm bảo luôn nhất quán tự động.

## How?

```java
// Mặc định: insertion-order — giữ đúng thứ tự THÊM VÀO lần đầu
Map<String, Integer> insertion = new LinkedHashMap<>();

// Access-order: mỗi lần get()/put() một key đã tồn tại, nó được ĐẨY XUỐNG CUỐI
// (coi như "vừa mới dùng")
Map<String, Integer> access = new LinkedHashMap<>(16, 0.75f, true);
```

Tham số thứ ba (`accessOrder`) chính là chìa khoá cho LRU Cache: kết hợp access-order với việc
override `removeEldestEntry()` (một hook có sẵn, mặc định luôn trả về `false`) để tự động loại
bỏ phần tử **lâu chưa dùng nhất** khi vượt quá kích thước cho phép.

## Visualization

```
HashMap thường (Chapter 08): thứ tự duyệt = thứ tự BUCKET (không đoán trước được)
  table[0]: banana→date   table[1]: apple→cherry   table[4]: egg

LinkedHashMap: bucket giống hệt HashMap, NHƯNG có thêm danh sách liên kết
đôi RIÊNG, xuyên suốt các entry theo thứ tự thêm vào/truy cập:

  head ⇄ [apple] ⇄ [banana] ⇄ [cherry] ⇄ [date] ⇄ [egg] ⇄ tail
         (thứ tự duyệt ĐI THEO DÂY NÀY, không đi theo bucket)
```

## Example

Cài đặt một **LRU Cache hoàn chỉnh** chỉ bằng cách override một method — chạy được, đúng hành
vi kinh điển được hỏi trong hầu hết buổi phỏng vấn:

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LruDemo {
    public static void main(String[] args) {
        int capacity = 3;
        Map<Integer, String> cache = new LinkedHashMap<>(16, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<Integer, String> eldest) {
                return size() > capacity;
            }
        };

        cache.put(1, "A");
        cache.put(2, "B");
        cache.put(3, "C");
        System.out.println("Sau khi them 3: " + cache.keySet());

        cache.get(1); // truy cap 1 -> day 1 len "moi nhat"
        cache.put(4, "D"); // vuot capacity -> loai bo phan tu it dung nhat
        System.out.println("Sau khi truy cap 1 roi them 4: " + cache.keySet());
    }
}
```

Kết quả thật (JDK 17):

```
Sau khi them 3: [1, 2, 3]
Sau khi truy cap 1 roi them 4: [3, 1, 4]
```

Đúng hành vi LRU: sau khi có `[1, 2, 3]`, việc `get(1)` đẩy `1` thành "vừa dùng gần nhất". Khi
thêm `4` (vượt capacity 3), `LinkedHashMap` tự động loại bỏ phần tử **lâu chưa dùng nhất** —
`2` (chưa từng được truy cập lại sau khi thêm) — không phải `1` (vừa được `get()`). Kết quả
cuối `[3, 1, 4]` xác nhận đúng: `2` đã biến mất, `1` vẫn còn (dù được thêm trước `3`) vì vừa
được truy cập.

## Deep Dive

**`removeEldestEntry()` được gọi ở đâu, và vì sao mặc định luôn `false`?** Method này được gọi
tự động **ngay sau mỗi `put()`** thành công — tham số `eldest` luôn là entry ở đầu danh sách
liên kết (phần tử "cũ nhất" theo đúng chế độ order đang dùng). Mặc định trả về `false` vì
`LinkedHashMap` **không tự áp đặt giới hạn kích thước** — nó chỉ cung cấp sẵn "điểm móc"
(hook) để lớp con (như ví dụ `cache` ở Example, dùng anonymous class) tự quyết định **khi nào**
nên loại bỏ. Đây là một ví dụ kinh điển của **Template Method Pattern**: `LinkedHashMap` định
nghĩa khung xử lý chung (put → kiểm tra `removeEldestEntry` → tự xoá nếu `true`), còn quyết
định "khi nào xoá" được để hoàn toàn cho người dùng tuỳ biến.

## Engineering Insight

**Vì sao access-order lại đủ để cài LRU mà không cần cấu trúc dữ liệu phức tạp hơn (như kết
hợp HashMap + Doubly Linked List tự viết tay, cách kinh điển hay được dạy để giải bài toán LRU
Cache trên LeetCode)?** Vì đó **chính xác** là những gì `LinkedHashMap` **đã làm sẵn bên
trong**: một `HashMap` (tra cứu O(1)) kết hợp một danh sách liên kết đôi (theo dõi thứ tự,
di chuyển node O(1)). Bài toán LRU Cache kinh điển yêu cầu tự tay viết lại đúng cấu trúc này từ
đầu để hiểu **nguyên lý** — nhưng trong công việc thực tế, `LinkedHashMap` với access-order +
`removeEldestEntry()` đã cung cấp sẵn implementation production-ready cho đúng bài toán đó,
không cần viết lại. Hiểu rõ điều này giúp trả lời tốt hơn câu hỏi phỏng vấn kinh điển "cài đặt
LRU Cache": vừa biết cách tự viết từ đầu (thể hiện hiểu nguyên lý), vừa biết công cụ có sẵn
trong JDK (thể hiện tư duy thực dụng).

## Historical Note

`LinkedHashMap` xuất hiện từ Java 1.4 (2002) — khá muộn so với `HashMap` (Java 2, 1998), cho
thấy nhu cầu "giữ thứ tự nhưng vẫn nhanh" được nhận ra và giải quyết riêng biệt sau một thời
gian sử dụng `HashMap` trong thực tế. `removeEldestEntry()` — nền tảng cho LRU Cache — có mặt
ngay từ phiên bản đầu tiên của `LinkedHashMap`, không phải bổ sung sau này.

## Myth vs Reality

- **Myth:** "`LinkedHashMap` chậm hơn đáng kể so với `HashMap` vì phải duy trì thêm danh sách
  liên kết."
  **Reality:** Chi phí thêm chỉ là cập nhật vài con trỏ mỗi lần `put()`/`get()` (access-order)
  — O(1), không ảnh hưởng tới độ phức tạp tổng thể. Trên thực tế, chênh lệch hiệu năng so với
  `HashMap` thường không đáng kể cho phần lớn ứng dụng.

- **Myth:** "Muốn cài LRU Cache, bắt buộc phải tự viết `HashMap` + Doubly Linked List từ đầu."
  **Reality:** Xem Example — `LinkedHashMap` với access-order và override
  `removeEldestEntry()` cho một implementation hoàn chỉnh, chạy đúng, chỉ trong vài dòng.

## Common Mistakes

- **Dùng `LinkedHashMap` mặc định (insertion-order) khi thực sự cần access-order cho LRU** —
  quên tham số thứ ba `true` trong constructor, khiến cache không loại bỏ đúng phần tử "lâu
  chưa dùng nhất".
- **Quên rằng `removeEldestEntry()` mặc định luôn `false`** — tạo một `LinkedHashMap` mong đợi
  tự giới hạn kích thước nhưng quên override method này, khiến map lớn vô hạn.

## Best Practices

- Khi cần LRU Cache đơn giản trong bộ nhớ JVM (không cần phân tán), ưu tiên `LinkedHashMap` với
  access-order thay vì tự viết cấu trúc dữ liệu từ đầu.
- Dùng insertion-order (mặc định) khi chỉ cần giữ đúng thứ tự thêm vào để hiển thị, không cần
  logic loại bỏ.
- Với cache cần chia sẻ giữa nhiều instance ứng dụng (không chỉ một JVM), `LinkedHashMap` không
  đủ — cần giải pháp phân tán như Redis (Phase 8).

## Production Notes

**Vấn đề:** một cache nội bộ dùng để giảm tải truy vấn database (lưu kết quả tra cứu gần đây)
tăng dần kích thước không giới hạn, cuối cùng gây `OutOfMemoryError`.

- **Triệu chứng:** bộ nhớ tăng đều theo thời gian, tỉ lệ thuận với số lượng key **khác nhau**
  đã từng được tra cứu — không bao giờ giảm.
- **Root cause:** cache được cài bằng `HashMap` thường (hoặc `LinkedHashMap` không override
  `removeEldestEntry()`) — không có cơ chế loại bỏ entry cũ, mọi key mới đều được giữ mãi mãi.
- **Debug:** heap dump (đã học ở [Phase 1, Chapter 04](../phase-01-foundation/04-runtime-data-areas.md))
  cho thấy chính cache đang giữ phần lớn bộ nhớ, số lượng entry tăng không giới hạn.
- **Solution:** đổi sang `LinkedHashMap` với access-order, override `removeEldestEntry()` với
  một giới hạn kích thước hợp lý — hoặc nếu cần kiểm soát theo thời gian (TTL) chứ không chỉ số
  lượng, cân nhắc thư viện cache chuyên dụng (Caffeine, Guava Cache) hoặc Redis.
- **Prevention:** mọi cache nội bộ trong ứng dụng **phải** có giới hạn kích thước hoặc thời
  gian sống rõ ràng ngay từ khi thiết kế — không bao giờ dùng `Map` thường làm cache "vô tình".

## Debug Checklist

- [ ] Bộ nhớ tăng dần không giới hạn, nghi ngờ liên quan tới cache? → kiểm tra cache có giới
      hạn kích thước không (`removeEldestEntry()` có được override đúng không).
- [ ] LRU Cache không loại bỏ đúng phần tử mong đợi? → kiểm tra constructor có truyền
      `accessOrder = true` không — nhầm insertion-order với access-order là lỗi phổ biến nhất.

## Source Code Walkthrough

`LinkedHashMap` override `newNode()`/`afterNodeAccess()`/`afterNodeInsertion()` — các "điểm
móc" (hook method) mà `HashMap` (Chapter 08) đã cố tình để trống/rỗng ngay từ thiết kế ban đầu,
chờ sẵn cho `LinkedHashMap` cắm vào logic riêng. `afterNodeAccess()` (chỉ có tác dụng khi
`accessOrder = true`) di chuyển node vừa truy cập ra cuối danh sách liên kết.
`afterNodeInsertion()` gọi `removeEldestEntry()` ngay sau khi thêm — đúng cơ chế đã mô tả ở
Deep Dive. Đây là một minh chứng thiết kế đẹp: `HashMap` được viết với các "khe cắm" sẵn có chủ
đích, để `LinkedHashMap` mở rộng mà không cần override lại toàn bộ `put()`.

## Summary

`LinkedHashMap` kế thừa toàn bộ cơ chế bucket/hash của `HashMap` (Chapter 08), chỉ thêm một
danh sách liên kết đôi xuyên suốt các entry để giữ thứ tự — insertion-order (mặc định) hoặc
access-order (tham số thứ ba trong constructor). Kết hợp access-order với việc override
`removeEldestEntry()` (một hook có sẵn) cho một implementation LRU Cache hoàn chỉnh, chạy đúng,
chỉ trong vài dòng — đã xác nhận bằng thực nghiệm. `HashMap` được thiết kế với các hook
(`afterNodeAccess`, `afterNodeInsertion`) chờ sẵn để `LinkedHashMap` cắm logic riêng vào mà
không cần viết lại `put()`.

## Interview Questions

**Junior**

- `LinkedHashMap` khác `HashMap` ở điểm nào?
- Có mấy chế độ order trong `LinkedHashMap`? Mặc định là chế độ nào?

**Mid**

- Cài đặt một LRU Cache đơn giản bằng `LinkedHashMap`.
- `removeEldestEntry()` được gọi khi nào? Vì sao mặc định luôn trả về `false`?

**Senior**

- So sánh việc dùng `LinkedHashMap` để cài LRU Cache với việc tự viết `HashMap` + Doubly Linked
  List từ đầu — cả hai có giống nhau về bản chất không?
- Một cache nội bộ trong ứng dụng gây `OutOfMemoryError` sau một thời gian chạy. Trình bày quy
  trình điều tra và giải pháp.

## Exercises

- [ ] Chạy lại đúng ví dụ `LruDemo` ở trên, xác nhận kết quả giống hệt.
- [ ] Đổi `accessOrder` từ `true` thành `false` (mặc định), chạy lại ví dụ, quan sát và giải
      thích sự khác biệt trong kết quả.
- [ ] Viết một `LinkedHashMap` insertion-order (không LRU), thêm và xoá vài phần tử, xác nhận
      thứ tự duyệt luôn đúng thứ tự thêm vào lần đầu, kể cả sau khi `get()` nhiều lần.

## Cheat Sheet

| Chế độ | Constructor | Thứ tự duyệt |
| --- | --- | --- |
| Insertion-order (mặc định) | `new LinkedHashMap<>()` | Thứ tự thêm vào lần đầu |
| Access-order | `new LinkedHashMap<>(cap, 0.75f, true)` | Thứ tự truy cập gần nhất → cũ nhất |

**Công thức LRU Cache:**
```java
new LinkedHashMap<K, V>(16, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return size() > capacity;
    }
}
```

## References

- Java SE API Documentation — `java.util.LinkedHashMap`.
- OpenJDK source — `java.base/java/util/LinkedHashMap.java`.
