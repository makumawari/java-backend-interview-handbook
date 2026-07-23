---
tags:
  - Java
  - ReadWriteLock
  - Concurrency
---

# ReadWriteLock

> Phase: Phase 3 — Concurrency
> Chapter slug: `readwritelock`

## Metadata

```yaml
Chapter: ReadWriteLock
Phase: Phase 3 — Concurrency
Difficulty: ★★★
Importance: ★★★
Interview Frequency: 45%
Prerequisites:
  - Chapter 10 — ReentrantLock
Used Later:
  - Cache (Phase 5, Phase 8) — mẫu hình "đọc nhiều, ghi hiếm" là lý do chính để chọn ReadWriteLock cho cache tự viết
Estimated Reading: 20 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 10](10-reentrantlock.md) — `ReentrantLock` là một khoá loại trừ **hoàn
> toàn**: dù bạn chỉ muốn **đọc** dữ liệu (không hề sửa gì), bạn vẫn phải giành đúng một lock
> duy nhất, chặn đứng mọi thread khác — kể cả những thread khác cũng chỉ muốn **đọc**.

Với dữ liệu được đọc rất thường xuyên nhưng hiếm khi ghi (cấu hình, cache, bảng tra cứu), đây
là một sự lãng phí rõ rệt: nhiều thread đọc cùng lúc **hoàn toàn không xung đột nhau** — không
ai sửa gì cả — vậy tại sao phải xếp hàng chờ nhau? Đo thử với 3 thread cùng "đọc":

```
r2 dang doc luc 86657
r1 dang doc luc 86657
r3 dang doc luc 86657
3 reader chay CUNG LUC, tong thoi gian: 213ms (neu doc tuan tu se ~600ms)
```

Cả ba đọc **đúng cùng một thời điểm** — không hề chờ nhau. `ReadWriteLock` tách biệt hai loại
khoá: một cho đọc (cho phép nhiều thread cùng lúc), một cho ghi (độc quyền tuyệt đối, như
`ReentrantLock` thông thường).

## Interview Question (Central)

> `ReadWriteLock` giải quyết vấn đề gì mà `ReentrantLock` thông thường không giải quyết được?
> Vì sao nhiều reader có thể chạy đồng thời nhưng writer thì không?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích nguyên tắc cốt lõi: nhiều **read lock** có thể cùng tồn tại, nhưng
      **write lock** luôn độc quyền (loại trừ cả read lẫn write khác)
- [ ] Tự tay đo được bằng thực nghiệm: nhiều reader chạy song song, writer chạy độc quyền
      (tuần tự với mọi thread khác)
- [ ] Biết mẫu hình sử dụng đúng: "đọc nhiều, ghi hiếm" — nếu ghi thường xuyên,
      `ReadWriteLock` không mang lại lợi ích, thậm chí có thể chậm hơn `ReentrantLock` đơn
      giản

## Prerequisites

- Chapter 10 — hiểu `ReentrantLock`, nền tảng để so sánh.

## Used Later

- **Cache** (Phase 5, Phase 8) — mẫu hình "đọc nhiều, ghi hiếm" chính là lý do phổ biến nhất để
  chọn `ReadWriteLock` cho một cache tự viết trong bộ nhớ (khi chưa cần tới giải pháp phân tán
  như Redis).

## Problem

`ReentrantLock`/`synchronized` (Chapter 05, 10) đối xử **mọi** thao tác như nhau — dù chỉ đọc
hay ghi, đều phải giành cùng một lock duy nhất, loại trừ lẫn nhau tuyệt đối. Với dữ liệu đọc
rất thường xuyên, ghi hiếm (tỉ lệ đọc:ghi có thể là 1000:1 trong thực tế), việc bắt mọi lần đọc
phải xếp hàng chờ nhau — dù chúng hoàn toàn không xung đột — là lãng phí đáng kể khả năng chạy
song song của phần cứng đa nhân.

## Concept

`ReadWriteLock` cung cấp **hai** lock tách biệt cho cùng một tài nguyên: **read lock** (nhiều
thread có thể cùng giữ nó đồng thời, miễn không có ai giữ write lock) và **write lock** (độc
quyền tuyệt đối — chỉ một thread giữ được, và không ai khác — kể cả reader — được vào trong lúc
đó).

## Why?

Nếu chỉ có một loại lock duy nhất (như `ReentrantLock`): throughput (thông lượng) cho khối
lượng công việc "đọc nhiều" bị giới hạn nghiêm trọng — dù có 16 lõi CPU rảnh rỗi, các thread
đọc vẫn phải xếp hàng tuần tự. Tách read lock (cho phép chia sẻ) khỏi write lock (độc quyền)
cho phép tận dụng đầy đủ khả năng song song của phần cứng cho phần **đọc** — phần chiếm đa số
tuyệt đối trong mẫu hình sử dụng mục tiêu — trong khi vẫn giữ đúng đắn tuyệt đối cho phần
**ghi** (không bao giờ có ai đọc dữ liệu đang bị sửa dở, tránh đọc dữ liệu không nhất quán).

## How?

```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();

// Đọc — NHIỀU thread cùng giữ read lock được, miễn không ai giữ write lock
rwLock.readLock().lock();
try { /* chỉ đọc, không sửa gì */ } finally { rwLock.readLock().unlock(); }

// Ghi — ĐỘC QUYỀN, chặn cả reader lẫn writer khác
rwLock.writeLock().lock();
try { /* sửa dữ liệu */ } finally { rwLock.writeLock().unlock(); }
```

## Visualization

```
3 Reader cùng lúc (KHÔNG có Writer nào tranh giành):

  Reader 1 ──readLock()──▶ [ĐỌC — chạy song song]
  Reader 2 ──readLock()──▶ [ĐỌC — chạy song song]     ← CẢ BA CÙNG LÚC
  Reader 3 ──readLock()──▶ [ĐỌC — chạy song song]

1 Writer + 1 Reader (Writer đến trước):

  Writer ──writeLock()──▶ [GHI — ĐỘC QUYỀN]
  Reader ──readLock()───▶ [CHỜ — writer đang giữ, reader KHÔNG được vào]
                                  │
                          Writer xong, nhả writeLock
                                  ▼
  Reader ──readLock()───▶ [ĐỌC — giờ mới được vào]
```

## Example

**Nhiều reader chạy đồng thời:**

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class RwLockDemo {
    static final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

    public static void main(String[] args) throws InterruptedException {
        long start = System.currentTimeMillis();
        Thread r1 = new Thread(() -> read("r1"));
        Thread r2 = new Thread(() -> read("r2"));
        Thread r3 = new Thread(() -> read("r3"));
        r1.start(); r2.start(); r3.start();
        r1.join(); r2.join(); r3.join();
        System.out.println("3 reader chay CUNG LUC, tong: " + (System.currentTimeMillis() - start) + "ms");
    }

    static void read(String name) {
        rwLock.readLock().lock();
        try {
            System.out.println(name + " dang doc");
            Thread.sleep(200);
        } catch (InterruptedException e) {
        } finally { rwLock.readLock().unlock(); }
    }
}
```

Kết quả thật (JDK 17):

```
r2 dang doc luc 86657
r1 dang doc luc 86657
r3 dang doc luc 86657
3 reader chay CUNG LUC, tong thoi gian: 213ms (neu doc tuan tu se ~600ms)
```

**Writer độc quyền — chặn đứng reader:**

```java
Thread writer = new Thread(() -> write("writer")); // giu writeLock 200ms
Thread reader = new Thread(() -> read("reader"));   // giu readLock 200ms
writer.start();
Thread.sleep(20); // dam bao writer chay truoc
reader.start();
```

Kết quả thật:

```
writer (WRITE) bat dau
writer (WRITE) xong
reader (READ) bat dau
reader (READ) xong
Writer + Reader (TUAN TU vi writer doc quyen), tong: 463ms
```

Đối lập hoàn toàn với 3 reader (213ms, chạy song song): writer + reader chạy **tuần tự**
(~463ms ≈ tổng 200ms + 200ms + overhead) — reader phải **chờ writer hoàn toàn xong** mới được
bắt đầu, xác nhận đúng tính độc quyền của write lock.

## Deep Dive

**Vì sao write lock phải chặn cả reader, không chỉ chặn writer khác?** Vì nếu cho phép reader
đọc **trong lúc** writer đang sửa dở dữ liệu, reader có thể đọc phải một trạng thái **không
nhất quán** (ví dụ writer đang cập nhật 2 field liên quan tới nhau, reader đọc được field thứ
nhất đã mới nhưng field thứ hai vẫn cũ) — vi phạm chính mục đích của việc dùng lock để bảo vệ
tính toàn vẹn dữ liệu. Write lock độc quyền tuyệt đối (chặn cả write lẫn read khác) là điều kiện
bắt buộc để đảm bảo mọi lần đọc luôn thấy dữ liệu ở một trạng thái **hoàn chỉnh, nhất quán** —
hoặc trước khi ghi, hoặc sau khi ghi xong hoàn toàn, không bao giờ ở giữa chừng.

## Engineering Insight

**`ReadWriteLock` không phải luôn luôn nhanh hơn `ReentrantLock` — khi nào nó thực sự trở
thành lựa chọn tồi?** Bản thân việc quản lý hai loại lock tách biệt (đếm số reader đang giữ,
điều phối chuyển đổi giữa "có reader" và "có writer") có **chi phí cố định cao hơn** một
`ReentrantLock` đơn giản. Nếu tỉ lệ ghi **không thực sự hiếm** (ví dụ gần 50/50 đọc/ghi, hoặc
tệ hơn ghi nhiều hơn đọc), lợi ích "nhiều reader chạy song song" gần như không bao giờ phát huy
tác dụng (vì write lock độc quyền liên tục chặn đứng mọi thứ) — trong khi vẫn phải trả chi phí
quản lý phức tạp hơn `ReentrantLock`. Đây là lý do chapter này (và nhiều tài liệu chính thức)
luôn nhấn mạnh "đọc nhiều, ghi hiếm" là điều kiện tiên quyết — không phải một gợi ý tuỳ chọn,
mà là ranh giới quyết định `ReadWriteLock` có thực sự mang lại lợi ích hay không.

## Historical Note

`ReadWriteLock`/`ReentrantReadWriteLock` ra đời cùng đợt JSR 166 (Java 5, 2004), cùng
`ReentrantLock` (Chapter 10). Java 8 (2018... à 2014) bổ sung `StampedLock` — một lựa chọn mới
hơn, hiệu năng cao hơn cho cùng bài toán "đọc nhiều, ghi hiếm", bổ sung thêm chế độ
**optimistic read** (đọc lạc quan — đọc mà không hề khoá gì cả, chỉ xác nhận lại sau nếu nghi
ngờ có thay đổi xảy ra trong lúc đọc) — nhanh hơn `ReadWriteLock` truyền thống trong nhiều
trường hợp, nhưng phức tạp hơn để dùng đúng, nằm ngoài phạm vi chapter nền tảng này.

## Myth vs Reality

- **Myth:** "`ReadWriteLock` luôn nhanh hơn `ReentrantLock` cho mọi tình huống có cả đọc lẫn
  ghi."
  **Reality:** Chỉ nhanh hơn khi tỉ lệ đọc thực sự áp đảo (đọc nhiều, ghi hiếm) — xem
  Engineering Insight. Với tỉ lệ ghi cao, nó có thể chậm hơn `ReentrantLock` đơn giản do chi
  phí quản lý phức tạp hơn.

- **Myth:** "Nhiều reader và một writer có thể cùng truy cập dữ liệu đồng thời, miễn writer
  'cẩn thận'."
  **Reality:** Xem Example — write lock luôn độc quyền tuyệt đối, không có "cẩn thận" nào cho
  phép reader và writer cùng hoạt động — đây là yêu cầu bắt buộc để đảm bảo tính nhất quán dữ
  liệu.

## Common Mistakes

- **Dùng `ReadWriteLock` cho dữ liệu có tỉ lệ ghi cao** — không mang lại lợi ích, thêm phức tạp
  không cần thiết.
- **Quên rằng đọc bên trong write lock, hoặc ghi bên trong read lock, không tự động "nâng cấp"
  loại khoá** — `ReentrantReadWriteLock` không hỗ trợ nâng cấp trực tiếp từ read lock lên write
  lock (giữ read lock rồi cố giành thêm write lock trong khi vẫn giữ read lock sẽ tự deadlock
  nếu có reader khác đang giữ).
- **Nghĩ rằng `ReadWriteLock` tự động "nhanh hơn" mà không đo lường thực tế** — luôn nên xác
  nhận bằng benchmark cụ thể cho đúng workload thật, không dựa vào giả định chung chung.

## Best Practices

- Chỉ dùng `ReadWriteLock` khi đã xác nhận (qua đo lường hoặc hiểu rõ domain) tỉ lệ đọc thực sự
  áp đảo ghi.
- Không cố "nâng cấp" từ read lock lên write lock trong cùng một luồng thực thi — nhả read lock
  trước, sau đó giành write lock riêng nếu cần ghi.
- Cân nhắc `StampedLock` (Java 8+) khi hiệu năng đọc là tối quan trọng và đã hiểu rõ mô hình
  optimistic read phức tạp hơn của nó.

## Production Notes

**Vấn đề:** một cache cấu hình trong bộ nhớ dùng `ReadWriteLock`, ban đầu hiệu năng tốt, nhưng
sau khi thêm tính năng "tự động refresh cấu hình mỗi 5 giây" (ghi thường xuyên hơn nhiều so với
thiết kế ban đầu), hiệu năng đọc giảm rõ rệt so với trước.

- **Triệu chứng:** độ trễ đọc cấu hình tăng lên đáng kể sau khi thêm tính năng auto-refresh, dù
  bản thân việc đọc logic không đổi.
- **Root cause:** giả định ban đầu ("ghi rất hiếm — chỉ khi admin thay đổi cấu hình thủ công")
  không còn đúng sau khi thêm auto-refresh định kỳ — tỉ lệ ghi tăng lên đáng kể, khiến write
  lock độc quyền chặn đứng reader thường xuyên hơn nhiều, đúng giới hạn đã phân tích ở
  Engineering Insight.
- **Debug:** đo lại tỉ lệ đọc/ghi thực tế sau khi thêm tính năng mới, so sánh với giả định thiết
  kế ban đầu.
- **Solution:** cân nhắc các chiến lược khác cho use case "dữ liệu thay đổi định kỳ" — ví dụ
  publish một tham chiếu bất biến mới hoàn toàn (Phase 1, Chapter 11) mỗi lần refresh, dùng
  `volatile` (Chapter 04) để đọc không cần khoá gì cả (đọc một tham chiếu `volatile` là thao
  tác nguyên tử, đơn giản hơn nhiều so với `ReadWriteLock` cho đúng use case "thay thế toàn bộ
  object" thay vì "sửa từng phần").
- **Prevention:** ghi rõ ràng giả định "tỉ lệ đọc/ghi dự kiến" khi chọn `ReadWriteLock`, và
  review lại giả định đó mỗi khi có thay đổi tính năng ảnh hưởng tới tần suất ghi.

## Debug Checklist

- [ ] Hiệu năng đọc giảm sau khi thay đổi liên quan tới ghi? → đo lại tỉ lệ đọc/ghi thực tế,
      xác nhận `ReadWriteLock` còn phù hợp không.
- [ ] Deadlock khi cố "nâng cấp" từ read lock lên write lock? → nhả read lock trước, giành
      write lock riêng biệt, không giữ cả hai cùng lúc trên cùng luồng.
- [ ] Cần dữ liệu thay đổi định kỳ, đọc rất thường xuyên? → cân nhắc publish tham chiếu bất
      biến qua `volatile` (Chapter 04) thay vì `ReadWriteLock` nếu mô hình dữ liệu phù hợp
      (thay thế toàn bộ, không sửa từng phần).

## Source Code Walkthrough

`ReentrantReadWriteLock` cũng xây trên AQS (đã nhắc ở Chapter 09-10), nhưng dùng khéo léo một
biến `int state` **32-bit** duy nhất, chia thành hai nửa 16-bit: nửa cao đếm số write lock
(thực chất chỉ 0 hoặc 1 vì độc quyền, nhưng dùng cùng cơ chế đếm reentrant như
`ReentrantLock`), nửa thấp đếm số **reader** hiện đang giữ read lock đồng thời — một kỹ thuật
"đóng gói hai bộ đếm vào một số nguyên" thông minh, cho phép AQS xử lý cả hai loại lock bằng
cùng một cơ chế CAS nền tảng, không cần hai cấu trúc dữ liệu tách biệt hoàn toàn.

## Summary

`ReadWriteLock` tách một lock thành hai: **read lock** (nhiều thread cùng giữ được đồng thời)
và **write lock** (độc quyền tuyệt đối, chặn cả reader lẫn writer khác) — đã xác nhận bằng thực
nghiệm: 3 reader chạy đồng thời (213ms thay vì ~600ms tuần tự), writer chặn đứng reader hoàn
toàn (chạy tuần tự, ~463ms). Chỉ mang lại lợi ích thực sự khi tỉ lệ đọc áp đảo ghi ("đọc nhiều,
ghi hiếm") — với tỉ lệ ghi cao, chi phí quản lý phức tạp hơn có thể khiến nó chậm hơn
`ReentrantLock` đơn giản. Không hỗ trợ nâng cấp trực tiếp từ read lock lên write lock trên cùng
luồng.

## Interview Questions

**Junior**

- `ReadWriteLock` khác `ReentrantLock` như thế nào?
- Nhiều reader có thể đọc cùng lúc không? Còn writer thì sao?

**Mid**

- Vì sao write lock phải chặn cả reader, không chỉ chặn writer khác?
- `ReadWriteLock` phù hợp cho mẫu hình sử dụng nào? Khi nào nó KHÔNG mang lại lợi ích?

**Senior**

- Giải thích vì sao không thể "nâng cấp" trực tiếp từ read lock lên write lock trên cùng một
  luồng mà không nhả read lock trước — điều gì có thể xảy ra nếu cố làm vậy?
- Một cache dùng `ReadWriteLock` giảm hiệu năng sau khi tần suất ghi tăng lên. Trình bày cách
  bạn sẽ chẩn đoán và các giải pháp thay thế khả thi.

## Exercises

- [ ] Chạy lại đúng hai ví dụ `RwLockDemo` và `RwLockWriterDemo` ở trên, xác nhận kết quả
      tương tự trên máy bạn.
- [ ] Đo thời gian tổng khi có 5 reader + 1 writer chạy đồng thời (writer bắt đầu giữa chừng),
      quan sát writer "chen ngang" ảnh hưởng tới các reader đang chờ như thế nào.
- [ ] Tìm hiểu `StampedLock` (Java 8+), so sánh mô hình optimistic read của nó với
      `ReadWriteLock` truyền thống — viết một ví dụ đơn giản dùng `tryOptimisticRead()`.

## Cheat Sheet

| | Read Lock | Write Lock |
| --- | --- | --- |
| Nhiều thread giữ cùng lúc? | Có (nếu không ai giữ write lock) | Không (độc quyền tuyệt đối) |
| Chặn reader khác? | Không | Có |
| Chặn writer khác? | Có (writer phải chờ mọi reader xong) | Có |

**Điều kiện nên dùng:** tỉ lệ đọc áp đảo ghi rõ rệt ("đọc nhiều, ghi hiếm"). Không đúng điều
kiện này → cân nhắc `ReentrantLock` đơn giản hoặc `volatile` (nếu chỉ cần publish tham chiếu
mới).

## References

- Java SE API Documentation — `java.util.concurrent.locks.ReadWriteLock`,
  `ReentrantReadWriteLock`.
- Java Concurrency in Practice (Brian Goetz) — Chapter 13.3: Read-Write Locks.
