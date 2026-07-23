---
tags:
  - Spring
  - Cache
---

# Spring Cache Abstraction (@Cacheable)

> Phase: Phase 5 — Spring
> Chapter slug: `cache`

## Metadata

```yaml
Chapter: Cache
Phase: Phase 5 — Spring
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 65%
Prerequisites:
  - Chapter 12 — AOP
Used Later:
  - Kiến trúc hệ thống hiệu năng cao (Phase 8)
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Cache tồn tại để giảm tải cho database — nhưng chính nó lại tạo ra một vấn đề mới: dữ liệu
> trong cache có thể **không còn khớp** với dữ liệu thật trong database, nếu database thay đổi
> mà cache không được thông báo.

Gọi một method `@Cacheable` lấy giá sản phẩm, cập nhật giá **thật** trong database (không qua
cache), rồi gọi lại:

```
=== Lan goi 1: chua co cache ===
  -> TRUY VAN DATABASE THAT cho 'laptop' (lan thu 1)
Ket qua: 1200.0

=== Lan goi 2: DA co cache, khong truy van DB ===
Ket qua: 1200.0

=== Cap nhat gia THAT trong database (khong qua cache) ===
  -> DA CAP NHAT database: laptop = 999.0 (cache CHUA biet gi ca)

=== Lan goi 3: cache VAN CON, tra ve gia CU (STALE) ===
Ket qua (ky vong VAN LA gia cu do cache): 1200.0
```

Database đã thực sự có giá mới (`999.0`), nhưng lần gọi thứ 3 vẫn trả về `1200.0` — giá **cũ**,
vì cache hoàn toàn không biết database đã thay đổi.

## Interview Question (Central)

> Cache improves response time, but users start seeing outdated data. How would you solve it?

## Objectives

- [ ] Dùng thành thạo `@Cacheable`, `@CacheEvict` để cache và làm mới dữ liệu
- [ ] Tự tay chứng minh bằng thực nghiệm vấn đề "stale data": cache không tự động cập nhật khi
      dữ liệu gốc thay đổi
- [ ] Biết các chiến lược giải quyết: evict tường minh, TTL (time-to-live), write-through

## Prerequisites

- Chapter 12 — hiểu AOP, vì `@Cacheable` cũng hiện thực hoá qua proxy giống mọi annotation khác
  đã học trong Phase 5.

## Used Later

- **Kiến trúc hệ thống hiệu năng cao** (Phase 8) — cache là công cụ tiêu chuẩn để giảm tải
  database ở quy mô lớn, đi kèm đánh đổi về tính nhất quán dữ liệu.

## Problem

Truy vấn database cho **cùng một dữ liệu, nhiều lần liên tiếp, khi dữ liệu đó ít thay đổi** là
lãng phí tài nguyên — mỗi lần truy vấn tốn thời gian I/O, tải lên database tăng không cần thiết.
Nhưng "nhớ lại" kết quả cũ (cache) đưa ra một đánh đổi cố hữu: dữ liệu cache có thể **lệch pha**
so với dữ liệu thật, nếu dữ liệu gốc thay đổi mà không có cơ chế thông báo cho cache.

## Concept

**Spring Cache Abstraction** (`@Cacheable`, `@CacheEvict`, `@CachePut`) cho phép "nhớ lại" kết
quả của một method — lần gọi đầu tiên với một tham số cụ thể thực thi method thật (truy vấn
database), các lần gọi sau **với cùng tham số** trả về ngay kết quả đã lưu, **không** chạy lại
thân method. `@CacheEvict` xoá một entry khỏi cache (buộc lần gọi tiếp theo phải truy vấn lại
dữ liệu thật).

## Why?

Cache đánh đổi **tính nhất quán tức thời** (dữ liệu luôn mới nhất) lấy **hiệu năng** (giảm tải
truy vấn) — đây là một đánh đổi có chủ đích, không phải khiếm khuyết. Việc `@Cacheable` không tự
động biết database đã thay đổi là hệ quả tất yếu: cache chỉ là một bản sao lưu trong bộ nhớ,
hoàn toàn tách biệt khỏi nguồn dữ liệu gốc — không có "phép màu" nào giúp nó tự đồng bộ, trừ khi
lập trình viên chủ động thiết kế cơ chế thông báo/làm mới.

## How?

```java
@Cacheable("prices")
public double getPrice(String product) {
    return database.get(product); // CHI chay khi CHUA co trong cache
}

@CacheEvict("prices")
public void evictPrice(String product) {
    // XOA entry khoi cache, lan goi getPrice() tiep theo se truy van lai THAT
}
```

## Visualization

```
Lan goi 1 (chua co cache):

  getPrice("laptop") ──► KHONG co trong cache ──► chay THAN METHOD (truy van DB that)
                                                        │
                                                   luu vao cache, tra ve 1200.0

Lan goi 2 (da co cache):

  getPrice("laptop") ──► CO trong cache ──► tra ve NGAY 1200.0, KHONG chay than method

Database thay doi (KHONG qua cache):

  updatePriceInDatabase("laptop", 999.0) ──► CHI sua database, cache VAN LA 1200.0

Lan goi 3 (cache CU, STALE):

  getPrice("laptop") ──► CO trong cache (nhung DA CU) ──► tra ve 1200.0 (SAI, that su la 999.0)

Sau @CacheEvict:

  evictPrice("laptop") ──► XOA entry "laptop" khoi cache
  getPrice("laptop") ──► KHONG con trong cache ──► chay THAN METHOD lai, tra ve 999.0 (DUNG)
```

## Example

```java
@Service
public class PriceService {
    private final Map<String, Double> database = new ConcurrentHashMap<>();

    @Cacheable("prices")
    public double getPrice(String product) {
        System.out.println("  -> TRUY VAN DATABASE THAT cho '" + product + "'");
        return database.get(product);
    }

    public void updatePriceInDatabase(String product, double newPrice) {
        database.put(product, newPrice); // sua THAT, khong lien quan gi toi cache
    }

    @CacheEvict("prices")
    public void evictPrice(String product) { }
}
```

Kết quả thật (Spring Boot 3.3.4, `@EnableCaching`):

```
=== Lan goi 1: chua co cache ===
  -> TRUY VAN DATABASE THAT cho 'laptop'
Ket qua: 1200.0

=== Lan goi 2: DA co cache, khong truy van DB ===
Ket qua: 1200.0

=== Cap nhat gia THAT trong database ===
  -> DA CAP NHAT database: laptop = 999.0

=== Lan goi 3: cache VAN CON, tra ve gia CU (STALE) ===
Ket qua (ky vong VAN LA gia cu do cache): 1200.0

=== Xoa cache (@CacheEvict) ===
  -> DA XOA cache cho 'laptop'

=== Lan goi 4: cache da bi xoa, truy van DB lai, tra ve gia MOI ===
  -> TRUY VAN DATABASE THAT cho 'laptop'
Ket qua (gia MOI): 999.0
```

Log `"TRUY VAN DATABASE THAT"` chỉ xuất hiện ở lần gọi 1 và lần gọi 4 (sau khi evict) — lần gọi 2
và 3 hoàn toàn không chạm tới thân method thật, chỉ đọc từ cache. Đây chính là bằng chứng trực
tiếp cho vấn đề "stale data": lần gọi 3 trả về `1200.0` dù dữ liệu thật trong database đã là
`999.0` từ trước đó — cache "không biết" điều này cho tới khi bị `@CacheEvict` xoá tường minh.

## Deep Dive

**Có những chiến lược nào để giải quyết vấn đề stale data, và đánh đổi của từng chiến lược là
gì?** (1) **Evict tường minh** (như ví dụ) — mọi nơi cập nhật dữ liệu gốc phải **nhớ** gọi
`@CacheEvict` tương ứng; đơn giản nhưng dễ **bỏ sót** (nếu có nhiều đường cập nhật dữ liệu, quên
evict ở một đường là đủ để gây bug). (2) **TTL (Time-To-Live)** — cache tự động hết hạn sau một
khoảng thời gian, chấp nhận dữ liệu có thể "cũ" trong khoảng thời gian đó, đổi lại đơn giản, không
cần nhớ evict thủ công ở mọi nơi — phù hợp khi ứng dụng chấp nhận được độ trễ nhất quán trong vài
giây/phút. (3) **Write-through** — mọi lần ghi dữ liệu đi qua cùng một lớp trừu tượng vừa cập nhật
database vừa cập nhật cache đồng thời (dùng `@CachePut` thay vì để dữ liệu tự "mồ côi" trong
cache) — nhất quán hơn (1) nhưng đòi hỏi kỷ luật thiết kế nghiêm ngặt hơn: **mọi** đường ghi dữ
liệu phải đi qua đúng lớp trừu tượng đó, không có ngoại lệ.

## Engineering Insight

**Vì sao câu hỏi phỏng vấn "cache cải thiện tốc độ nhưng dữ liệu bị cũ" không có một đáp án
"đúng" duy nhất?** Đây là bài toán về **đánh đổi có chủ đích** (trade-off), câu trả lời đúng phụ
thuộc vào ngữ cảnh nghiệp vụ cụ thể: với dữ liệu giá sản phẩm thay đổi vài lần một ngày, TTL vài
phút hoàn toàn chấp nhận được. Với dữ liệu số dư tài khoản ngân hàng, stale data dù chỉ vài giây
cũng không chấp nhận được — cần evict tường minh hoặc write-through nghiêm ngặt, thậm chí cân
nhắc không cache dữ liệu loại này. Một câu trả lời phỏng vấn tốt không chỉ nêu "dùng TTL" hay
"dùng evict" một cách máy móc, mà thể hiện được khả năng **đánh giá mức độ chấp nhận được của
stale data** theo đúng ngữ cảnh nghiệp vụ trước khi chọn chiến lược.

## Historical Note

Spring Cache Abstraction ra đời ở Spring 3.1 (2011) — thiết kế có chủ đích là một **abstraction**
(lớp trừu tượng) tách biệt hoàn toàn khỏi công nghệ cache cụ thể bên dưới (mặc định dùng
`ConcurrentHashMap` đơn giản trong bộ nhớ nếu không cấu hình gì thêm, nhưng có thể cắm
Redis/Ehcache/Caffeine qua cùng annotation `@Cacheable` mà không cần sửa code nghiệp vụ) — cùng
triết lý "tách interface khỏi implementation cụ thể" đã thấy xuyên suốt Spring (`DataSource`,
`PlatformTransactionManager`, ...).

## Myth vs Reality

- **Myth:** "`@Cacheable` tự động phát hiện khi dữ liệu gốc thay đổi và tự làm mới cache."
  **Reality:** Đã chứng minh bằng thực nghiệm — cache hoàn toàn "mù" trước thay đổi của dữ liệu
  gốc nếu không cập nhật qua đúng đường (`updatePriceInDatabase` trực tiếp, không qua
  `@CachePut`/`@CacheEvict`).

- **Myth:** "Chỉ cần thêm `@Cacheable` là đủ để tăng tốc, không cần suy nghĩ gì thêm."
  **Reality:** Xem Engineering Insight — cần đánh giá mức độ chấp nhận được của stale data theo
  ngữ cảnh nghiệp vụ trước khi quyết định chiến lược evict/TTL/write-through phù hợp.

## Common Mistakes

- **Cập nhật dữ liệu gốc mà quên `@CacheEvict`/`@CachePut` tương ứng** — nguồn gốc phổ biến nhất
  của bug stale data, đã chứng minh trực tiếp bằng thực nghiệm.
- **Cache dữ liệu nhạy cảm về tính nhất quán (số dư, tồn kho) mà không đánh giá kỹ rủi ro stale
  data** — có thể gây hậu quả nghiêm trọng hơn lợi ích hiệu năng mang lại.
- **Không đặt TTL cho cache dùng lâu dài** — nếu quên evict ở một đường cập nhật nào đó, dữ liệu
  có thể "cũ" vĩnh viễn cho tới khi ứng dụng restart.

## Best Practices

- Luôn đảm bảo mọi đường cập nhật dữ liệu gốc đều có `@CacheEvict`/`@CachePut` tương ứng, hoặc
  dùng TTL như lớp bảo vệ bổ sung.
- Đánh giá mức độ chấp nhận được của stale data theo ngữ cảnh nghiệp vụ trước khi cache — không
  phải mọi dữ liệu đều nên cache.
- Với hệ thống nhiều instance (Chapter 15), cân nhắc cache phân tán (Redis) thay vì cache trong
  bộ nhớ cục bộ (mỗi instance có bản cache riêng, dễ lệch nhau hơn nữa).

## Debug Checklist

- [ ] Người dùng báo cáo thấy dữ liệu cũ dù đã cập nhật? → kiểm tra đường cập nhật dữ liệu đó có
      gọi đúng `@CacheEvict`/`@CachePut` không.
- [ ] Cache không hoạt động dù đã đánh dấu `@Cacheable`? → kiểm tra `@EnableCaching` đã được
      khai báo chưa; kiểm tra self-invocation (Chapter 12) nếu gọi từ bên trong cùng class.
- [ ] Dữ liệu cache lệch nhau giữa các instance? → cân nhắc chuyển sang cache phân tán (Redis)
      thay vì cache cục bộ trong bộ nhớ từng instance.

## Summary

Spring Cache Abstraction (`@Cacheable`/`@CacheEvict`) giảm tải database bằng cách "nhớ lại" kết
quả method, đổi lấy rủi ro dữ liệu cache lệch pha với dữ liệu gốc (stale data). Đã chứng minh
bằng thực nghiệm: sau khi cache đã lưu kết quả, cập nhật dữ liệu gốc trực tiếp (không qua
`@CacheEvict`) khiến lần gọi tiếp theo vẫn trả về giá trị **cũ** — cache hoàn toàn không biết
database đã thay đổi cho tới khi bị evict tường minh. Ba chiến lược giải quyết: evict tường minh
(đơn giản, dễ bỏ sót), TTL (đơn giản hơn, chấp nhận độ trễ nhất quán), write-through (nhất quán
nhất, đòi hỏi kỷ luật thiết kế nghiêm ngặt). Không có đáp án đúng tuyệt đối — cần đánh giá theo
ngữ cảnh nghiệp vụ cụ thể.

## Interview Questions

- Cache improves response time, but users start seeing outdated data. How would you solve it?

**Mid**

- `@Cacheable` và `@CacheEvict` khác nhau như thế nào?

**Senior**

- Nêu 3 chiến lược giải quyết vấn đề stale data trong cache, phân tích đánh đổi của từng chiến
  lược.
- Với dữ liệu số dư tài khoản ngân hàng, bạn có nên cache không? Nếu có, chiến lược nào phù hợp?

## Exercises

- [ ] Chạy lại `CacheStaleDataTest` ở trên, xác nhận vấn đề stale data đúng như mô tả.
- [ ] Sửa `updatePriceInDatabase` để tự động gọi `@CachePut` cập nhật cache đồng thời (write-through), xác nhận lần gọi tiếp theo trả về đúng giá mới ngay lập tức mà không cần evict riêng.
- [ ] Cấu hình TTL cho cache (dùng Caffeine hoặc cấu hình `spring.cache.cache-names` +
      `spec`), quan sát cache tự hết hạn sau thời gian đã đặt.

## Cheat Sheet

| Annotation | Vai trò |
| --- | --- |
| `@Cacheable` | Lưu kết quả, bỏ qua thân method nếu đã có trong cache |
| `@CacheEvict` | Xoá một/nhiều entry khỏi cache |
| `@CachePut` | Luôn chạy thân method, đồng thời cập nhật cache (write-through) |
| `@Caching` | Kết hợp nhiều thao tác cache trên cùng một method |

| Chiến lược | Đơn giản | Nhất quán | Rủi ro |
| --- | --- | --- | --- |
| Evict tường minh | Trung bình | Cao (nếu không quên) | Dễ bỏ sót một đường cập nhật |
| TTL | Cao | Trung bình (có độ trễ) | Dữ liệu cũ trong khoảng TTL |
| Write-through | Thấp | Cao | Đòi hỏi kỷ luật thiết kế nghiêm ngặt |

## References

- Spring Framework Documentation — Cache Abstraction: https://docs.spring.io/spring-framework/reference/integration/cache.html
