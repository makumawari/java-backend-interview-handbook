---
tags:
  - Java
  - WeakHashMap
  - Collections
  - GC
---

# WeakHashMap

> Phase: Phase 2 — Collections
> Chapter slug: `weakhashmap`

## Metadata

```yaml
Chapter: WeakHashMap
Phase: Phase 2 — Collections
Difficulty: ★★★★
Importance: ★★
Interview Frequency: 30%
Prerequisites:
  - Chapter 08 — HashMap
  - Phase 1, Chapter 07 — Garbage Collection (reachability, GC Roots)
Used Later:
  - JVM Tuning (Phase 8) — quản lý bộ nhớ cache là bài toán production thường xuyên gặp
Estimated Reading: 20 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Phase 1, Chapter 07](../phase-01-foundation/07-garbage-collection.md) — GC chỉ thu
> hồi object **không còn reachable** từ bất kỳ GC Root nào.

Đây chính là nguồn gốc của một loại memory leak rất phổ biến: một `HashMap` dùng làm cache,
với key là chính các object nghiệp vụ (không phải String/ID đơn giản). Dù logic nghiệp vụ đã
"dùng xong" một object từ lâu, chừng nào nó còn là **key trong `HashMap`**, `HashMap` vẫn giữ
một tham chiếu **mạnh** (strong reference) tới nó — khiến GC coi nó vẫn "còn sống", không bao
giờ thu hồi được, dù không còn ai khác cần tới nó nữa.

`WeakHashMap` giải quyết đúng vấn đề này bằng một ý tưởng táo bạo: giữ key bằng một loại tham
chiếu **yếu** (weak reference) — loại tham chiếu mà GC **được phép bỏ qua** khi quyết định một
object có còn sống hay không.

## Interview Question (Central)

> `WeakHashMap` khác `HashMap` ở điểm nào? Tại sao một entry có thể tự "biến mất" khỏi
> `WeakHashMap` mà không cần gọi `remove()`?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Phân biệt tham chiếu mạnh (strong reference, mặc định) với tham chiếu yếu (weak
      reference, `WeakReference`)
- [ ] Tự tay quan sát bằng thực nghiệm: một entry trong `WeakHashMap` biến mất sau khi GC chạy,
      nếu key không còn tham chiếu mạnh nào khác
- [ ] Biết đúng use case của `WeakHashMap`: cache không muốn tự mình gây memory leak
- [ ] Hiểu vì sao chỉ **key** (không phải value) dùng weak reference trong `WeakHashMap`

## Prerequisites

- Chapter 08 — hiểu cơ chế bucket/hash của `HashMap`, `WeakHashMap` chỉ thay đổi cách giữ key.
- Phase 1, Chapter 07 — hiểu reachability và GC Roots.

## Used Later

- **JVM Tuning** (Phase 8) — quản lý cache đúng cách để tránh memory leak là một bài toán
  production thường xuyên; `WeakHashMap` là một trong các công cụ (dù không phải luôn là lựa
  chọn tốt nhất — xem Best Practices) cho bài toán này.

## Problem

Một cache dùng object nghiệp vụ (không phải String/ID nguyên thuỷ) làm key sẽ giữ tham chiếu
mạnh tới object đó **mãi mãi**, trừ khi có ai chủ động `remove()` — nhưng thường không có tín
hiệu rõ ràng nào báo "key này không còn cần thiết nữa" để gọi `remove()` đúng lúc. Kết quả: bộ
nhớ tăng dần không giới hạn theo thời gian, một dạng memory leak "kiểu Java" đã được cảnh báo ở
Phase 1 Chapter 07.

## Concept

`WeakHashMap` là một implementation `Map` giữ **key** bằng `WeakReference` thay vì tham chiếu
mạnh thông thường. Khi **không còn tham chiếu mạnh nào khác** trỏ tới một key (nghĩa là object
đó chỉ còn được "biết tới" qua chính `WeakHashMap`), GC được phép thu hồi nó **dù nó vẫn đang
là key trong map** — entry tương ứng sẽ tự động biến mất khỏi `WeakHashMap` sau đó, không cần
gọi `remove()` tường minh.

## Why?

Nếu key trong `WeakHashMap` vẫn giữ tham chiếu mạnh (như `HashMap` thường): dùng nó làm cache
sẽ y hệt vấn đề đã nêu ở Problem — object không bao giờ được GC dù logic nghiệp vụ đã "quên"
nó từ lâu. Bằng cách dùng tham chiếu yếu, `WeakHashMap` để cho **GC quyết định** khi nào một
entry không còn ý nghĩa (dựa trên reachability thực tế của chính key đó ở phần còn lại của
chương trình), thay vì buộc lập trình viên phải tự quản lý vòng đời cache một cách thủ công và
dễ sai sót.

## How?

```java
WeakHashMap<Object, String> cache = new WeakHashMap<>();
Object key = new Object();
cache.put(key, "du lieu cache");
// ... cache.size() == 1

key = null; // bỏ tham chiếu MẠNH duy nhất còn lại tới key
System.gc(); // yêu cầu GC chạy
// ... sau khi GC thực sự thu hồi được key: cache.size() == 0
```

## Visualization

```
HashMap thường:
  GC Roots ──(không trực tiếp)──   key object  ◀── HashMap GIỮ THAM CHIẾU MẠNH
                                        ▲             → GC KHÔNG BAO GIỜ thu hồi được
                                        │               dù không còn ai khác cần nó
                                   (chỉ còn map biết tới nó)

WeakHashMap:
                                    key object  ◀╌╌ WeakHashMap giữ THAM CHIẾU YẾU (đường đứt)
                                        │             → GC ĐƯỢC PHÉP thu hồi, coi như
                                    (không ai khác                                  KHÔNG có tham chiếu
                                     giữ tham chiếu mạnh)                            nào từ WeakHashMap
                                        │
                                        ▼
                                   GC thu hồi → entry TỰ ĐỘNG biến mất khỏi WeakHashMap
```

## Example

Xác nhận trực tiếp: một entry `WeakHashMap` tự biến mất sau khi bỏ tham chiếu mạnh duy nhất và
yêu cầu GC chạy:

```java
import java.util.WeakHashMap;

public class WeakHashMapDemo {
    public static void main(String[] args) throws InterruptedException {
        WeakHashMap<Object, String> cache = new WeakHashMap<>();
        Object key = new Object();
        cache.put(key, "du lieu cache");
        System.out.println("Truoc GC, size = " + cache.size());

        key = null; // bo tham chieu MANH duy nhat toi key
        System.gc();
        Thread.sleep(200);

        System.out.println("Sau GC, size = " + cache.size());
    }
}
```

Kết quả thật (JDK 17):

```
Truoc GC, size = 1
Sau GC, size = 0
```

Không có dòng `cache.remove(key)` nào được gọi — entry biến mất hoàn toàn nhờ GC nhận ra key
không còn reachable từ đâu khác ngoài chính `WeakHashMap` (mà `WeakHashMap` chỉ giữ tham chiếu
yếu, không tính).

## Deep Dive

**Vì sao chỉ `System.gc()` thôi chưa chắc đã đủ trong thực tế — tại sao ví dụ dùng thêm
`Thread.sleep(200)`?** Vì `System.gc()` (đã cảnh báo ở
[Phase 1, Chapter 07](../phase-01-foundation/07-garbage-collection.md)) chỉ là một **gợi ý**
cho JVM, không đảm bảo GC chạy ngay lập tức hay chạy đầy đủ. Việc entry "biến mất" khỏi
`WeakHashMap` không xảy ra ngay tại thời điểm object bị GC thu hồi — nó cần một bước riêng:
`WeakReference` đã bị GC "dọn" được đưa vào một hàng đợi nội bộ
(`ReferenceQueue`), và `WeakHashMap` chỉ **dọn dẹp entry tương ứng** khi có thao tác tiếp theo
trên map (như gọi `size()`) kích hoạt việc xử lý hàng đợi đó — đây là lý do trong ví dụ thực tế,
đôi khi cần một khoảng chờ nhỏ hoặc một thao tác kích hoạt để thấy rõ kết quả, dù về bản chất GC
đã xảy ra.

## Engineering Insight

**Vì sao chỉ `key` dùng weak reference, không phải `value`?** Vì nếu `value` cũng chỉ được giữ
yếu: ngay cả khi `key` vẫn còn reachable mạnh mẽ từ nơi khác (nghĩa là entry đó **nên** vẫn còn
hợp lệ), `value` vẫn có thể bị GC thu hồi bất cứ lúc nào nếu không ai khác giữ tham chiếu mạnh
tới nó — biến `WeakHashMap` thành một cấu trúc không đáng tin cậy ngay cả cho các entry đang
"thực sự cần dùng". Chỉ làm yếu **key** giữ đúng ngữ nghĩa: "entry này chỉ nên tồn tại chừng
nào key của nó còn được ai đó (ngoài chính map) quan tâm tới" — value thì vẫn được giữ mạnh
**bởi chính entry**, đảm bảo một khi entry còn tồn tại, `value` của nó luôn truy cập được đầy
đủ, không bị GC thu hồi giữa chừng một cách bất ngờ.

## Historical Note

`WeakHashMap` xuất hiện từ Java 1.2 (1998), cùng đợt `java.lang.ref` package (`WeakReference`,
`SoftReference`, `PhantomReference`) — một trong những API level thấp lâu đời nhất của Java
liên quan trực tiếp tới việc "nói chuyện" với GC. Qua nhiều năm, `WeakHashMap` vẫn giữ nguyên
thiết kế ban đầu — không có thay đổi lớn nào, phản ánh đây là một công cụ chuyên dụng, hẹp,
không cần cập nhật thường xuyên như các cấu trúc dữ liệu phổ biến hơn (`HashMap`, `ArrayList`).

## Myth vs Reality

- **Myth:** "`WeakHashMap` là giải pháp chung cho mọi bài toán cache, tự động dọn dẹp thay bạn
  hoàn toàn."
  **Reality:** Nó chỉ giải quyết đúng một vấn đề hẹp: key không còn reachable từ nơi khác. Với
  cache cần chiến lược phức tạp hơn (giới hạn kích thước, TTL theo thời gian, LRU) —
  `LinkedHashMap` (Chapter 09) hoặc thư viện cache chuyên dụng (Caffeine, Guava Cache) phù hợp
  hơn nhiều.

- **Myth:** "Gọi `System.gc()` đảm bảo ngay lập tức entry biến mất khỏi `WeakHashMap`."
  **Reality:** Xem Deep Dive — `System.gc()` chỉ là gợi ý, và việc dọn entry còn phụ thuộc vào
  cơ chế `ReferenceQueue` nội bộ được kích hoạt.

## Common Mistakes

- **Dùng `WeakHashMap` cho cache mà key vẫn còn được tham chiếu mạnh ở nơi khác trong toàn bộ
  vòng đời ứng dụng** (ví dụ key là một hằng số `static`) — entry sẽ **không bao giờ** bị dọn,
  vì key luôn reachable, khiến `WeakHashMap` không mang lại lợi ích gì so với `HashMap` thường.
- **Kỳ vọng `WeakHashMap` dọn dẹp ngay lập tức, tức thời** — thời điểm dọn phụ thuộc hoàn toàn
  vào khi GC thực sự chạy, không xác định trước.
- **Dùng value (không phải key) làm đối tượng cần "tự động dọn"** — `WeakHashMap` chỉ áp dụng
  weak reference cho key, không phải value (xem Engineering Insight).

## Best Practices

- Chỉ dùng `WeakHashMap` khi thực sự hiểu rõ vòng đời của key — nó phải có khả năng trở nên
  "không còn ai cần" ở đâu đó trong chương trình, không phải một hằng số sống suốt vòng đời
  ứng dụng.
- Cho nhu cầu cache phức tạp hơn (giới hạn kích thước, TTL, LRU), ưu tiên `LinkedHashMap`
  (Chapter 09) với `removeEldestEntry()`, hoặc thư viện cache chuyên dụng thay vì
  `WeakHashMap`.
- Không dựa vào thời điểm chính xác một entry bị dọn — coi đó là hành vi "cuối cùng sẽ xảy ra",
  không phải "ngay lập tức".

## Production Notes

Chapter này thuần về một công cụ chuyên biệt, hẹp — ít khi tự nó là nguồn gốc trực tiếp của sự
cố production (ngược lại, nó thường được dùng **để tránh** sự cố memory leak). Vấn đề production
thường gặp nhất liên quan tới `WeakHashMap` thực chất là **hiểu lầm về nó**: dùng nó cho một
bài toán cache cần kiểm soát chặt chẽ hơn (giới hạn kích thước, TTL) trong khi nó chỉ giải
quyết đúng một khía cạnh hẹp (dọn theo reachability của key) — dẫn tới hành vi cache không dự
đoán được (khi nào entry biến mất phụ thuộc hoàn toàn vào lịch trình GC, không phải logic
nghiệp vụ), gây khó khăn khi debug các vấn đề liên quan tới cache-miss bất thường.

## Debug Checklist

- [ ] Entry trong `WeakHashMap` không bao giờ tự dọn dù mong đợi nó biến mất? → kiểm tra key có
      đang bị giữ tham chiếu mạnh ở nơi khác không (ví dụ biến `static`, field của object sống
      lâu).
- [ ] Cần cache có hành vi dọn dẹp **dự đoán được** (không phụ thuộc lịch trình GC)? →
      `WeakHashMap` không phù hợp, cân nhắc `LinkedHashMap` (Chapter 09) hoặc thư viện cache
      chuyên dụng.

## Source Code Walkthrough

Mỗi entry trong `WeakHashMap` thực chất là một `WeakReference` bọc quanh key, đăng ký với một
`ReferenceQueue` dùng chung của map. Khi GC xác định key không còn reachable mạnh, nó tự động
đưa `WeakReference` đó vào `ReferenceQueue`. Các method của `WeakHashMap` (`size()`, `get()`,
`put()`...) đều gọi một method nội bộ `expungeStaleEntries()` **trước khi** thực hiện thao tác
chính — method này rút hết các tham chiếu đã "chết" ra khỏi `ReferenceQueue` và xoá đúng entry
tương ứng khỏi bucket — đây chính xác là cơ chế đã quan sát được ở Example (`size()` sau GC trả
về 0, vì `expungeStaleEntries()` đã chạy ngay trước khi trả kết quả).

## Summary

`WeakHashMap` giữ **key** bằng `WeakReference` thay vì tham chiếu mạnh — khi key không còn
reachable từ bất kỳ đâu khác ngoài chính map, GC được phép thu hồi nó, và entry tương ứng tự
động biến mất (đã xác nhận bằng thực nghiệm). Chỉ **key** dùng weak reference, không phải
**value** — đảm bảo value luôn toàn vẹn chừng nào entry còn tồn tại. Đây là công cụ chuyên biệt
cho đúng một bài toán: tránh memory leak khi key có thể "hết cần thiết" mà không có tín hiệu rõ
ràng để gọi `remove()` — không phải giải pháp cache tổng quát, và không đảm bảo thời điểm dọn
dẹp chính xác (phụ thuộc lịch trình GC).

## Interview Questions

**Junior**

- `WeakHashMap` khác `HashMap` ở điểm nào?
- Một entry trong `WeakHashMap` có thể tự biến mất khi nào?

**Mid**

- Giải thích vì sao chỉ key (không phải value) trong `WeakHashMap` dùng weak reference.
- `WeakHashMap` có phù hợp cho mọi bài toán cache không? Khi nào nên dùng công cụ khác?

**Senior**

- Giải thích cơ chế `ReferenceQueue` đứng sau việc `WeakHashMap` tự dọn entry — vì sao entry
  không biến mất "ngay lập tức" tại thời điểm GC thu hồi key?
- So sánh `WeakHashMap` với `LinkedHashMap` (Chapter 09) + `removeEldestEntry()` cho bài toán
  cache — khi nào chọn cái nào?

## Exercises

- [ ] Chạy lại đúng ví dụ `WeakHashMapDemo` ở trên, xác nhận kết quả giống hệt.
- [ ] Sửa ví dụ để **giữ lại** một tham chiếu mạnh khác tới key (ví dụ gán vào một biến thứ
      hai trước khi gán biến gốc `= null`) — chạy lại, xác nhận entry **không** biến mất, giải
      thích vì sao.
- [ ] Tìm hiểu `SoftReference` (một loại tham chiếu yếu hơn `WeakReference` một bậc, chỉ bị GC
      thu hồi khi bộ nhớ thực sự thiếu) — so sánh với `WeakReference` dùng trong
      `WeakHashMap`.

## Cheat Sheet

| | HashMap | WeakHashMap |
| --- | --- | --- |
| Tham chiếu tới key | Mạnh (strong) | Yếu (weak) |
| Key có thể tự bị GC dù còn trong map? | Không | Có (nếu không còn tham chiếu mạnh khác) |
| Value có ảnh hưởng bởi weak reference? | — | Không (value vẫn giữ mạnh bởi entry) |
| Phù hợp cho | Mục đích chung | Cache tránh memory leak, key có vòng đời "hết cần thiết" |

## References

- Java SE API Documentation — `java.util.WeakHashMap`, `java.lang.ref.WeakReference`.
- Oracle: "The Java Tutorials" — phần Reference Objects.
