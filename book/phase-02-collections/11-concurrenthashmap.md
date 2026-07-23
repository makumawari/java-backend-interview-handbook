---
tags:
  - Java
  - ConcurrentHashMap
  - Collections
  - Concurrency
---

# ConcurrentHashMap

> Phase: Phase 2 — Collections
> Chapter slug: `concurrenthashmap`

## Metadata

```yaml
Chapter: ConcurrentHashMap
Phase: Phase 2 — Collections
Difficulty: ★★★★
Importance: ★★★★★
Interview Frequency: 80%
Prerequisites:
  - Chapter 08 — HashMap
  - Chapter 01 — Collection Framework (fail-fast là best-effort, không phải thread-safe)
Used Later:
  - Thread, Lock, CAS (Phase 3 — Concurrency) — cơ chế khoá tinh vi của ConcurrentHashMap sẽ được đào sâu lý thuyết ở đó
  - Spring Cache, mọi cache dùng chung giữa nhiều request (Phase 5, 8)
Estimated Reading: 25 phút
Estimated Practice: 25 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 08 — Engineering Insight](08-hashmap.md) đã hé lộ: "`HashMap` không
> thread-safe, có chủ đích."

Câu nói đó nghe trừu tượng — cho tới khi bạn tự tay đo được hậu quả cụ thể. Cho 8 thread cùng
tăng một bộ đếm chung trong một `HashMap` bình thường, mỗi thread tăng 50.000 lần — kỳ vọng kết
quả cuối cùng là 400.000. Thực tế:

```
HashMap thuong: ket qua = 74935 (mong doi 400000)
```

**Mất hơn 80% dữ liệu** — không một dòng exception nào được ném ra, không cảnh báo gì cả,
chương trình chạy "bình thường" từ đầu tới cuối. Đây là điều tệ nhất có thể xảy ra với một hệ
thống: sai lệch dữ liệu **âm thầm tuyệt đối**. Chạy đúng cùng bài toán đó với
`ConcurrentHashMap`:

```
ConcurrentHashMap: ket qua = 400000 (mong doi 400000)
```

Chính xác tuyệt đối. Chapter này giải thích `ConcurrentHashMap` đạt được điều đó bằng cách nào
— và quan trọng không kém, tại sao nó **không** đơn giản là "`HashMap` cộng thêm
`synchronized`".

## Interview Question (Central)

> `HashMap` khác `ConcurrentHashMap` ở điểm nào? Tại sao `ConcurrentHashMap` không đơn giản
> khoá (`synchronized`) toàn bộ map cho mọi thao tác?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Tự tay đo được bằng thực nghiệm hậu quả cụ thể của việc dùng `HashMap` trong môi trường
      đa luồng — mất dữ liệu âm thầm, không exception
- [ ] Xác nhận `ConcurrentHashMap` cho kết quả đúng tuyệt đối trong cùng điều kiện
- [ ] Giải thích ở mức khái niệm: `ConcurrentHashMap` khoá **từng bucket riêng lẻ**, không khoá
      toàn bộ map như `Hashtable`/`Collections.synchronizedMap()`
- [ ] Phân biệt "fail-fast" (best-effort, Chapter 01) với "thread-safe" (đảm bảo đúng đắn thật
      sự) — hai khái niệm rất dễ nhầm lẫn
- [ ] Biết `ConcurrentHashMap` không chấp nhận `null` key/value, và tại sao

## Prerequisites

- Chapter 08 — hiểu trọn vẹn cơ chế bucket/hash/resize/treeify của `HashMap`.
- Chapter 01 — đã biết fail-fast chỉ là best-effort, không phải cơ chế đảm bảo an toàn.

## Used Later

- **Thread, Lock, CAS** (Phase 3 — Concurrency) — chapter này giới thiệu **hiện tượng** và
  **hệ quả**; cơ chế lý thuyết đầy đủ (`synchronized`, CAS, memory visibility) đứng sau
  `ConcurrentHashMap` sẽ được đào sâu khi có đủ nền tảng ở Phase 3.
- **Spring Cache và mọi cache dùng chung** (Phase 5, 8) — bất kỳ cache nào được truy cập từ
  nhiều request/thread đồng thời (gần như luôn đúng với server xử lý nhiều request song song)
  đều cần `ConcurrentHashMap` hoặc cấu trúc tương đương, không phải `HashMap` thường.

## Problem

Một web server xử lý hàng nghìn request đồng thời, mỗi request chạy trên một thread riêng
(sẽ học kỹ ở Phase 3). Nếu nhiều request cùng đọc/ghi một `HashMap` dùng chung (ví dụ một cache
đếm số lượt truy cập theo endpoint) mà không có cơ chế bảo vệ, dữ liệu có thể bị hỏng theo cách
không thể đoán trước và không để lại dấu vết lỗi rõ ràng — đúng hiện tượng đã đo ở Story.

## Concept

`ConcurrentHashMap` là một implementation `Map` được thiết kế **để dùng an toàn bởi nhiều
thread cùng lúc**, mà vẫn giữ hiệu năng cao — thay vì khoá **toàn bộ** cấu trúc cho mọi thao
tác (cách làm của `Hashtable` cũ và `Collections.synchronizedMap()`), nó chỉ khoá ở phạm vi rất
nhỏ (từng bucket, hoặc dùng CAS không cần khoá cho nhiều trường hợp) — cho phép nhiều thread
thao tác trên các phần khác nhau của map **thực sự song song**, không phải chờ nhau tuần tự.

## Why?

Nếu khoá toàn bộ map cho mọi thao tác (dù chỉ đọc): với nhiều thread cùng truy cập, chúng sẽ
phải **xếp hàng chờ nhau** dù đang thao tác trên các key hoàn toàn khác nhau, không liên quan
gì tới nhau — lãng phí khả năng chạy song song thực sự của CPU đa nhân. Khoá theo phạm vi nhỏ
(fine-grained locking) cho phép hai thread thao tác trên hai bucket khác nhau chạy **thực sự
đồng thời**, chỉ thread nào chạm cùng bucket mới cần chờ nhau — tận dụng tối đa phần cứng đa
nhân hiện đại.

## How?

```
HashMap (Chapter 08) trong môi trường đa luồng:
  KHÔNG có cơ chế bảo vệ gì cả — 2 thread cùng resize/cùng sửa 1 bucket
  → cấu trúc liên kết nội bộ bị hỏng, dữ liệu mất mát hoặc vòng lặp vô hạn

Hashtable / Collections.synchronizedMap() (cách cũ):
  synchronized TOÀN BỘ map cho MỌI thao tác — an toàn nhưng mất khả năng
  chạy song song, mọi thread phải xếp hàng dù thao tác trên key khác nhau

ConcurrentHashMap (Java 8+):
  Khoá ở PHẠM VI TỪNG BUCKET (dùng CAS + synchronized cục bộ trên node đầu
  bucket) — hai thread thao tác 2 bucket khác nhau chạy THỰC SỰ song song
```

## Visualization

```
Thread A: put("apple", ...)      Thread B: put("egg", ...)
     │                                │
     ▼                                ▼
bucket[1] (apple, cherry)        bucket[4] (egg)
     │                                │
   CHỈ khoá bucket[1]              CHỈ khoá bucket[4]
   (Thread B KHÔNG bị chặn,        (Thread A KHÔNG bị chặn,
    vì đang thao tác bucket khác)   vì đang thao tác bucket khác)
     │                                │
     ▼                                ▼
  CHẠY SONG SONG THỰC SỰ, không phải xếp hàng chờ nhau
```

## Example

Đo thật hậu quả cụ thể — 8 thread, mỗi thread tăng một bộ đếm chung 50.000 lần
(`merge(key, 1, Integer::sum)`, đã học ở [Chapter 07](07-map.md)):

```java
import java.util.Map;
import java.util.HashMap;
import java.util.concurrent.*;

public class HashMapRace {
    public static void main(String[] args) throws InterruptedException {
        Map<String, Long> map = new HashMap<>();
        map.put("counter", 0L);
        int threads = 8;
        int incrementsPerThread = 50_000;
        ExecutorService pool = Executors.newFixedThreadPool(threads);
        for (int t = 0; t < threads; t++) {
            pool.submit(() -> {
                for (int i = 0; i < incrementsPerThread; i++) {
                    try {
                        map.merge("counter", 1L, Long::sum);
                    } catch (Exception e) { /* HashMap khong thread-safe */ }
                }
            });
        }
        pool.shutdown();
        pool.awaitTermination(30, TimeUnit.SECONDS);
        long expected = (long) threads * incrementsPerThread;
        System.out.println("HashMap thuong: ket qua = " + map.get("counter") + " (mong doi " + expected + ")");
    }
}
```

Kết quả thật (JDK 17, kết quả có thể khác nhau giữa các lần chạy — đây chính là bản chất của
race condition):

```
HashMap thuong: ket qua = 74935 (mong doi 400000)
```

Đổi `Map<String, Long> map = new HashMap<>();` thành `new ConcurrentHashMap<>();` (giữ nguyên
mọi thứ khác), kết quả thật:

```
ConcurrentHashMap: ket qua = 400000 (mong doi 400000)
```

Chính xác tuyệt đối, mỗi lần chạy lại. Đây là bằng chứng thực nghiệm trực tiếp, không phải lý
thuyết suông — chỉ đổi **một** dòng khai báo kiểu đã biến một hệ thống mất 80% dữ liệu thành
một hệ thống chính xác 100%.

## Deep Dive

**Vì sao chạy `HashMapRace` nhiều lần có thể cho ra các con số khác nhau (không phải luôn
đúng 74935)?** Vì đây là một **race condition** đúng nghĩa — kết quả phụ thuộc vào thời điểm
chính xác các thread "đụng độ" nhau khi cùng đọc-sửa-ghi cùng một bucket (đọc giá trị cũ, cộng
1, ghi giá trị mới — ba bước **không nguyên tử** trên `HashMap` thường). Nếu hai thread cùng đọc
giá trị cũ **trước khi** bất kỳ ai ghi giá trị mới, một trong hai lần cộng sẽ bị "mất" (lost
update) — số lần điều này xảy ra phụ thuộc vào lịch trình (scheduling) thực tế của hệ điều
hành, khác nhau mỗi lần chạy. Đây chính là lý do race condition được coi là loại bug khó chịu
nhất: **không tái hiện ổn định**, có thể "may mắn" chạy đúng hàng trăm lần trên máy dev rồi mới
lộ ra trong production dưới tải thật.

## Engineering Insight

**Vì sao câu hỏi phỏng vấn "vì sao ConcurrentHashMap không dùng synchronized toàn bộ" lại quan
trọng đến vậy?** Vì nó kiểm tra đúng hiểu biết về đánh đổi **throughput vs. correctness** —
`Hashtable`/`Collections.synchronizedMap()` (khoá toàn bộ) **cũng đúng đắn** (correct), chỉ là
chậm hơn nhiều dưới tải cao nhiều thread. `ConcurrentHashMap` không "đúng đắn hơn"
`Hashtable` — cả hai đều cho kết quả chính xác 400000 ở Example — nó chỉ **nhanh hơn đáng kể**
nhờ khoá phạm vi nhỏ, đặc biệt khi số lượng thread và bucket đủ lớn để tận dụng được tính song
song thực sự. Việc chọn `ConcurrentHashMap` thay vì `Hashtable` trong code hiện đại không phải
vì "an toàn hơn" — cả hai đều an toàn — mà vì hiệu năng tốt hơn nhiều trong ngữ cảnh nhiều
thread, nhiều lõi CPU của hệ thống production hiện đại.

## Historical Note

```
Java 1.0 (1996)
    ↓
Hashtable — khoá TOÀN BỘ map (mọi method đều synchronized), tồn tại song
song với HashMap ngay từ đầu nhưng cách tiếp cận khoá thô, hiệu năng kém
dưới tải nhiều thread
    ↓
Java 5 (2004)
    ↓
ConcurrentHashMap ra đời (JSR 166) — dùng kỹ thuật "lock striping":
chia map thành nhiều SEGMENT độc lập, mỗi Segment có khoá RIÊNG — giảm
tranh chấp khoá đáng kể so với Hashtable, nhưng vẫn còn khoá ở mức thô
(cả một Segment, không phải từng bucket)
    ↓
Java 8 (2014)
    ↓
Thiết kế lại hoàn toàn — bỏ Segment, khoá trực tiếp ở MỨC BUCKET (node
đầu mỗi bucket), kết hợp CAS (Compare-And-Swap, sẽ học kỹ ở Phase 3) cho
nhiều thao tác không cần khoá hoàn toàn — mức độ tranh chấp khoá giảm
xuống tối thiểu, gần hiệu năng của HashMap thường trong nhiều trường hợp
```

## Myth vs Reality

- **Myth:** "`ConcurrentHashMap` chỉ đơn giản là `HashMap` cộng thêm `synchronized` trên mỗi
  method."
  **Reality:** Đó chính xác là cách `Hashtable` hoạt động (khoá toàn bộ), không phải
  `ConcurrentHashMap` — xem Historical Note. `ConcurrentHashMap` khoá ở phạm vi rất nhỏ (từng
  bucket), phức tạp hơn nhiều so với "thêm `synchronized`" đơn thuần.

- **Myth:** "Vì `HashMap` có fail-fast (`ConcurrentModificationException`), nó cũng tương đối
  an toàn trong môi trường đa luồng."
  **Reality:** Fail-fast (Chapter 01) chỉ là cơ chế **phát hiện lỗi khi duyệt** (best-effort,
  không đảm bảo tuyệt đối) — hoàn toàn không liên quan tới an toàn dữ liệu khi nhiều thread
  **ghi** đồng thời, như Example đã chứng minh (74935 thay vì 400000, không hề có
  `ConcurrentModificationException` nào được ném ra).

## Common Mistakes

- **Dùng `HashMap` cho dữ liệu dùng chung giữa nhiều thread** (biến `static`, cache dùng
  chung, field của một bean singleton trong Spring — Phase 5) — đúng lỗi đã minh hoạ ở Story.
- **Nghĩ rằng bọc `Collections.synchronizedMap(hashMap)` là đủ hiện đại** — về mặt đúng đắn thì
  đúng (giống `Hashtable`), nhưng bỏ lỡ hoàn toàn lợi ích hiệu năng của khoá phạm vi nhỏ mà
  `ConcurrentHashMap` cung cấp.
- **Cố `put(null, ...)` vào `ConcurrentHashMap`** — ném `NullPointerException` ngay lập tức
  (đã nhắc ở [Chapter 07](07-map.md)) — trong ngữ cảnh đa luồng, `null` gây mơ hồ nghiêm trọng
  hơn nhiều: không thể phân biệt an toàn giữa "key chưa tồn tại" và "một thread khác đang trong
  quá trình xoá nó" chỉ bằng cách kiểm tra `get() == null`.

## Best Practices

- Mặc định dùng `ConcurrentHashMap` cho bất kỳ `Map` nào được truy cập từ nhiều thread —
  không dùng `HashMap` "tạm thời rồi sửa sau", vì hậu quả (mất dữ liệu âm thầm) có thể không
  bị phát hiện cho tới khi đã gây thiệt hại thực tế.
- Ưu tiên các thao tác nguyên tử có sẵn (`merge()`, `computeIfAbsent()`, đã học ở Chapter 07)
  thay vì tự viết "get rồi put" thủ công, kể cả khi dùng `ConcurrentHashMap` — thao tác thủ
  công vẫn có thể có khoảng hở race condition nếu không dùng đúng method nguyên tử.
- Không dùng `null` làm giá trị có ý nghĩa trong bất kỳ `Map` nào dự định dùng ở môi trường đa
  luồng.

## Production Notes

**Vấn đề:** một bộ đếm thống kê (ví dụ số lượt xem sản phẩm) dùng `HashMap` lưu trong bộ nhớ,
số liệu báo cáo cuối ngày luôn thấp hơn đáng kể so với số lượt request thực tế đã ghi nhận qua
log truy cập.

- **Triệu chứng:** chênh lệch không cố định, không có lỗi/exception nào trong log, khó phát
  hiện nếu không chủ động đối chiếu số liệu.
- **Root cause:** `HashMap` được dùng làm bộ đếm dùng chung giữa nhiều thread xử lý request
  đồng thời (Phase 3 sẽ giải thích rõ mô hình này) — đúng race condition đã đo ở Example, chỉ
  là xảy ra âm thầm trong production thay vì trong một bài test có chủ đích.
- **Debug:** đối chiếu số liệu đếm được với một nguồn đáng tin cậy khác (log truy cập, số dòng
  trong database) — chênh lệch đáng kể mà không có lỗi rõ ràng là dấu hiệu kinh điển của race
  condition.
- **Solution:** đổi `HashMap` sang `ConcurrentHashMap`, dùng `merge()`/`computeIfAbsent()` cho
  các thao tác đếm/cập nhật thay vì "get rồi put" thủ công.
- **Prevention:** mọi cấu trúc dữ liệu dùng chung giữa nhiều thread (đặc biệt field `static`
  hoặc field của bean singleton trong Spring) phải được review kỹ về tính thread-safe ngay từ
  thiết kế, không đợi tới khi phát hiện sai lệch số liệu mới sửa.

## Debug Checklist

- [ ] Số liệu đếm/thống kê không khớp, sai lệch không ổn định, không có exception? → nghi ngờ
      race condition trên một `Map`/cấu trúc dữ liệu dùng chung giữa nhiều thread — kiểm tra có
      đang dùng `HashMap` thường không.
- [ ] `NullPointerException` khi `put()`/`get()` vào một `Map` sau khi đổi sang
      `ConcurrentHashMap`? → kiểm tra có đang dùng `null` key/value không (Chapter 07).
- [ ] Cần xác nhận một đoạn code có an toàn với nhiều thread không? → viết lại đúng bài test
      như `HashMapRace`/`ConcurrentDemo` ở Example, chạy thử với cấu trúc dữ liệu thực tế đang
      dùng.

## Source Code Walkthrough

`ConcurrentHashMap.put()` (Java 8+, ở mức khái niệm, chi tiết CAS đầy đủ thuộc Phase 3): nếu
bucket đích đang rỗng, dùng CAS để gán node mới **không cần khoá** (nếu thất bại — nghĩa là
thread khác vừa chen vào — thử lại); nếu bucket đã có phần tử, khoá `synchronized` **chỉ trên
node đầu bucket đó** trong lúc chèn/duyệt danh sách liên kết (hoặc cây, nếu đã treeify) — phạm
vi khoá cực nhỏ, không ảnh hưởng tới các bucket khác đang được thread khác xử lý đồng thời.
`resize()` cũng được thiết kế để nhiều thread có thể **cùng tham gia** quá trình chuyển dữ liệu
sang bảng mới (transfer), thay vì chỉ một thread gánh toàn bộ công việc — một chi tiết engineering
tinh vi vượt ngoài phạm vi chapter nền tảng này.

## Summary

`ConcurrentHashMap` là implementation `Map` an toàn cho nhiều thread, đạt hiệu năng cao bằng
cách khoá ở phạm vi **từng bucket** (kết hợp CAS không cần khoá cho nhiều trường hợp), khác hẳn
`Hashtable`/`Collections.synchronizedMap()` (khoá toàn bộ map). Dùng `HashMap` thường trong môi
trường đa luồng gây **mất dữ liệu âm thầm** — đã đo thực nghiệm mất hơn 80% số lần cập nhật mà
không hề có exception nào báo hiệu — trong khi `ConcurrentHashMap` cho kết quả chính xác tuyệt
đối trong cùng điều kiện. Fail-fast (Chapter 01) và thread-safe là hai khái niệm hoàn toàn khác
nhau — không được nhầm lẫn. `ConcurrentHashMap` không chấp nhận `null` key/value, vì `null` gây
mơ hồ nguy hiểm hơn nhiều trong ngữ cảnh đa luồng.

## Interview Questions

**Junior**

- HashMap khác ConcurrentHashMap ở điểm nào?
- Dùng `HashMap` trong môi trường nhiều thread có an toàn không?

**Mid**

- Fail-fast của `HashMap` (Chapter 01) có liên quan gì tới an toàn đa luồng không? Giải thích
  rõ sự khác biệt.
- Vì sao `ConcurrentHashMap` không chấp nhận `null` key/value?

**Senior**

- Tại sao `ConcurrentHashMap` không dùng `synchronized` toàn bộ (như `Hashtable`)? Giải thích
  cơ chế khoá phạm vi nhỏ mang lại lợi ích gì cụ thể.
- Thiết kế `ConcurrentHashMap` thay đổi thế nào từ Java 5 (Segment) sang Java 8 (khoá theo
  bucket)? Vì sao thay đổi này cần thiết?

## Exercises

- [ ] Chạy lại đúng hai ví dụ `HashMapRace` và bản `ConcurrentHashMap` tương ứng, xác nhận
      chênh lệch kết quả tương tự trên máy bạn (số liệu chính xác có thể khác, nhưng `HashMap`
      luôn cho kết quả sai, `ConcurrentHashMap` luôn cho kết quả đúng).
- [ ] Thử thay `map.merge(...)` bằng thao tác "get rồi put" thủ công
      (`map.put("counter", map.get("counter") + 1)`) trên `ConcurrentHashMap` — quan sát liệu
      kết quả có còn chính xác tuyệt đối không, tự giải thích vì sao (gợi ý: `merge()` nguyên
      tử, "get rồi put" thủ công thì không, kể cả trên `ConcurrentHashMap`).
- [ ] Đo thời gian chạy của `HashMapRace` (dù kết quả sai) so với bản `ConcurrentHashMap` —
      xác nhận `ConcurrentHashMap` không chỉ đúng hơn mà còn không chậm hơn đáng kể so với
      `HashMap` không an toàn.

## Cheat Sheet

| | HashMap | Hashtable | ConcurrentHashMap |
| --- | --- | --- | --- |
| Thread-safe | Không | Có (khoá toàn bộ) | Có (khoá phạm vi nhỏ) |
| Hiệu năng đa luồng | N/A (không an toàn) | Kém (tranh chấp khoá cao) | Tốt |
| `null` key/value | Cho phép | Không cho phép | Không cho phép |
| Fail-fast iterator | Có (best-effort) | Có | Weakly consistent (không ném CME) |

**Nguyên tắc:** `Map` dùng chung giữa nhiều thread → luôn `ConcurrentHashMap`, không bao giờ
`HashMap` thường.

## References

- Java SE API Documentation — `java.util.concurrent.ConcurrentHashMap`.
- JSR 166: Concurrency Utilities.
- OpenJDK source — `java.base/java/util/concurrent/ConcurrentHashMap.java`.
- Java Concurrency in Practice (Brian Goetz) — chương về concurrent collections.
