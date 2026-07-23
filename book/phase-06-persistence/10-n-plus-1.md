---
tags:
  - JPA
  - NPlusOne
  - Hibernate
---

# Vấn đề N+1 Query

> Phase: Phase 6 — Persistence
> Chapter slug: `n-plus-1`

## Metadata

```yaml
Chapter: N+1
Phase: Phase 6 — Persistence
Difficulty: ★★★★★
Importance: ★★★★★
Interview Frequency: 90%
Prerequisites:
  - Chapter 08 — Lazy Loading
  - Chapter 09 — Fetch Join
Used Later:
  - Tối ưu hiệu năng tầng dữ liệu trong production (Phase 8)
Estimated Reading: 18 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: [Chapter 09](09-fetch-join.md) — đã thấy 5 Customer + 5 Order tốn 6 câu SQL thay vì
> 1. Nhưng con số "6" chỉ là khởi đầu — vấn đề này **tệ đi tuyến tính** theo số lượng bản ghi.

Tăng số Customer lên 20 (mỗi Customer vẫn chỉ có 1 Order), đo lại số câu SQL:

```
20 Customer -> tong so query: 21 (ky vong 1 + 20 = 21)
```

Đúng **21** câu — `1` (lấy danh sách Customer) `+ 20` (một câu riêng cho mỗi Customer để lấy
Order của nó) `= N + 1`, với `N = 20`. Với một bảng thực tế có hàng chục nghìn bản ghi, con số
này hoàn toàn có thể "giết chết" hiệu năng một endpoint API tưởng chừng đơn giản.

## Interview Question (Central)

> Vấn đề N+1 query là gì? Vì sao nó xảy ra, và làm sao phát hiện/khắc phục nó trong thực tế?

## Objectives

- [ ] Giải thích chính xác cơ chế gây ra N+1: kết hợp lazy loading (Chapter 08) với vòng lặp
      truy cập quan hệ
- [ ] Tự tay chứng minh bằng thực nghiệm số câu SQL tăng tuyến tính theo N (5 Customer → 6 câu,
      20 Customer → 21 câu)
- [ ] Biết cách phát hiện N+1 trong thực tế (logging, công cụ giám sát) và áp dụng đúng giải
      pháp (fetch join, Chapter 09)

## Prerequisites

- Chapter 08 — hiểu lazy loading, cơ chế nền tảng khiến N+1 xảy ra.
- Chapter 09 — hiểu fetch join, giải pháp chính cho vấn đề này.

## Used Later

- **Tối ưu hiệu năng tầng dữ liệu** (Phase 8) — N+1 là một trong những vấn đề hiệu năng phổ biến
  và tốn kém nhất trong các ứng dụng dùng ORM ở quy mô production.

## Problem

Kết hợp hai tính năng hữu ích riêng lẻ (lazy loading — Chapter 08 — và code lặp qua danh sách
entity truy cập quan hệ của từng cái) tạo ra một vấn đề hiệu năng nghiêm trọng mà **không hề có
dấu hiệu cảnh báo rõ ràng** ở tầng code Java: mọi dòng code đều hợp lệ, không có exception, kết
quả logic hoàn toàn đúng — chỉ có **số lượng round-trip tới database** âm thầm tăng vọt.

## Concept

**N+1 Query** là hiện tượng: lấy `N` bản ghi cha bằng **một** câu truy vấn, sau đó với **mỗi**
bản ghi cha, chạy **thêm một** câu truy vấn riêng để lấy dữ liệu quan hệ của nó (do lazy loading
kích hoạt khi vòng lặp truy cập quan hệ) — tổng cộng `1 + N` câu truy vấn, thay vì lẽ ra chỉ cần
**một** câu truy vấn duy nhất (dùng `JOIN`/fetch join, Chapter 09) để lấy toàn bộ dữ liệu cần
thiết.

## Why?

N+1 là hệ quả **tất yếu** khi lazy loading (thiết kế để tối ưu cho trường hợp không cần dữ liệu
quan hệ) gặp đúng tình huống ngược lại — code **thực sự cần** dữ liệu quan hệ cho **mọi** bản ghi
trong một danh sách. Lazy loading tối ưu cho "chỉ thỉnh thoảng cần dữ liệu quan hệ, cho một entity
đơn lẻ"; khi dùng trong vòng lặp qua toàn bộ danh sách, giả định đó không còn đúng — mỗi lần lặp
kích hoạt đúng một round-trip riêng biệt, trong khi bản chất bài toán ("lấy N Customer và Order
của tất cả") hoàn toàn có thể giải quyết bằng một câu SQL `JOIN` duy nhất.

## How?

```java
// GAY N+1
List<Customer> customers = customerRepo.findAll();      // 1 query
for (Customer c : customers) {
    c.getOrders().size(); // MOI vong lap: 1 query rieng (lazy loading kich hoat)
}
// Tong: 1 + N query

// SUA bang Fetch Join (Chapter 09)
@Query("select distinct c from Customer c join fetch c.orders")
List<Customer> findAllWithOrders();

List<Customer> withOrders = customerRepo.findAllWithOrders(); // 1 query DUY NHAT
for (Customer c : withOrders) {
    c.getOrders().size(); // KHONG query them, da fetch san
}
```

## Visualization

```
N+1 (5 Customer):                        Fetch Join (5 Customer):

  SELECT * FROM customer  (1 query)        SELECT c.*, o.* FROM customer c
  ── lap qua tung Customer ──                 JOIN orders o ON c.id=o.customer_id
  SELECT * FROM orders WHERE customer_id=1    (1 query DUY NHAT, lay het moi thu)
  SELECT * FROM orders WHERE customer_id=2
  SELECT * FROM orders WHERE customer_id=3
  SELECT * FROM orders WHERE customer_id=4
  SELECT * FROM orders WHERE customer_id=5
  = 1 + 5 = 6 query                        = 1 query

N tang -> so query TANG TUYEN TINH:      N tang -> VAN LA 1 query duy nhat
  N=5   -> 6 query
  N=20  -> 21 query
  N=1000 -> 1001 query (!)
```

## Example

```java
List<Customer> customers = customerRepo.findAll();
long total = 0;
for (Customer c : customers) {
    total += c.getOrders().size();
}
```

Kết quả thật (Hibernate ORM 6.5.3.Final, đo bằng `Statistics.getPrepareStatementCount()`):

```
So Customer: 5
Tong so query SQL thuc thi (N+1): 6

So Customer: 20
Tong so query SQL thuc thi: 21
```

Số câu SQL tăng **chính xác** theo công thức `N + 1`: 5 Customer → 6 câu, 20 Customer → 21 câu.
Không phải ước lượng, mà là quan hệ tuyến tính chính xác — xác nhận trực tiếp cơ chế "một câu
truy vấn riêng cho mỗi vòng lặp".

## Deep Dive

**Vì sao N+1 đặc biệt nguy hiểm trong bối cảnh REST API (Phase 5, Chapter 09)?** Serialize một
`List<Customer>` (có quan hệ lazy) thành JSON response thường **tự động** truy cập mọi field,
bao gồm cả quan hệ — nếu `spring.jpa.open-in-view=true` (mặc định, Chapter 08) đang bật, việc
serialize response **âm thầm kích hoạt N+1** ở tầng Jackson, hoàn toàn nằm ngoài tầm nhìn của
code nghiệp vụ (không ai chủ động viết vòng lặp gây N+1 — nó xảy ra "ẩn" bên trong quá trình
serialize). Đây là lý do N+1 thường **không bị phát hiện khi test với dữ liệu nhỏ** (vài bản
ghi, chênh lệch hiệu năng không đáng chú ý) nhưng gây sự cố nghiêm trọng khi dữ liệu production
tăng lên hàng chục nghìn bản ghi — một endpoint từng phản hồi trong 50ms có thể mất hàng giây
khi N đủ lớn, dù không một dòng code nào thay đổi.

## Engineering Insight

**Công cụ nào giúp phát hiện N+1 trong thực tế trước khi nó gây sự cố production?** Ba cách phổ
biến: (1) **Hibernate Statistics** (như đã dùng trong Example, `getPrepareStatementCount()`) —
có thể tích hợp vào test tích hợp, tự động fail nếu số câu truy vấn vượt ngưỡng kỳ vọng cho một
thao tác cụ thể (một kỹ thuật kiểm thử hiệu năng gọi là "query count assertion"). (2) Thư viện
chuyên dụng như **datasource-proxy** hoặc **p6spy** — ghi log/đếm mọi câu SQL thực thi trong môi
trường development, giúp phát hiện trực quan các pattern N+1 khi duyệt qua log. (3) **APM
(Application Performance Monitoring)** như New Relic, Datadog — theo dõi số lượng database call
trên mỗi request ở production thực tế, cảnh báo khi một endpoint có số lượng query bất thường
cao. Cách tiếp cận tốt nhất kết hợp cả ba: query count assertion trong test tự động (bắt lỗi sớm
nhất, trong CI), datasource-proxy khi debug cục bộ, APM để giám sát liên tục ở production.

## Historical Note

N+1 query không phải vấn đề riêng của Hibernate/JPA — đây là một cạm bẫy phổ biến của **mọi**
ORM trên mọi ngôn ngữ (Ruby on Rails ActiveRecord, Django ORM, Entity Framework của .NET đều có
thuật ngữ và vấn đề tương tự) — bất cứ khi nào một ORM cung cấp lazy loading tiện lợi, nguy cơ
N+1 xuất hiện đi kèm như một đánh đổi tất yếu. Đây là lý do nhiều ORM hiện đại (bao gồm Hibernate
những phiên bản gần đây) đầu tư đáng kể vào công cụ cảnh báo N+1 tự động trong log hoặc chế độ
phát triển.

## Myth vs Reality

- **Myth:** "N+1 chỉ là vấn đề lý thuyết, hiếm khi thực sự ảnh hưởng hiệu năng đáng kể trong
  thực tế."
  **Reality:** Đã chứng minh bằng thực nghiệm quan hệ tuyến tính chính xác — với dữ liệu
  production quy mô lớn (hàng chục nghìn bản ghi), N+1 là một trong những nguyên nhân phổ biến
  nhất gây sự cố hiệu năng nghiêm trọng trong thực tế.

- **Myth:** "N+1 chỉ xảy ra khi lập trình viên chủ động viết vòng lặp truy cập quan hệ."
  **Reality:** Xem Deep Dive — nó có thể xảy ra âm thầm trong quá trình serialize JSON response,
  hoàn toàn không có vòng lặp tường minh nào trong code nghiệp vụ.

## Common Mistakes

- **Không kiểm tra số lượng câu SQL thực thi khi viết test cho các endpoint trả về danh sách có
  quan hệ** — bỏ lỡ cơ hội phát hiện N+1 sớm nhất, ngay trong giai đoạn phát triển.
- **Trả entity có quan hệ lazy trực tiếp làm response API mà không kiểm soát fetch strategy** —
  nguy cơ N+1 âm thầm khi serialize, đặc biệt khi `open-in-view` đang bật.
- **Chỉ test với dữ liệu nhỏ (vài bản ghi) trong môi trường development**, không đại diện đúng
  cho quy mô dữ liệu production, khiến N+1 "vô hình" cho tới khi triển khai thực tế.

## Production Notes

**Vấn đề:** endpoint `GET /api/customers` trả về danh sách khách hàng kèm đơn hàng, phản hồi
nhanh (~50ms) khi test với 20 khách hàng, nhưng chậm nghiêm trọng (~8 giây) trong production với
15,000 khách hàng.

- **Triệu chứng:** thời gian phản hồi tăng không tuyến tính theo cảm nhận người dùng khi dữ liệu
  tăng, dù logic không đổi giữa môi trường test và production.
- **Root cause:** N+1 query — endpoint lặp qua danh sách khách hàng, truy cập `orders` cho từng
  người (trực tiếp hoặc gián tiếp qua serialize JSON) — với 15,000 khách hàng, tổng cộng 15,001
  câu SQL, mỗi câu tốn thời gian round-trip mạng riêng biệt.
- **Debug:** bật `datasource-proxy` trong môi trường staging, xác nhận đúng số lượng câu SQL
  tăng tuyến tính theo số khách hàng.
- **Solution:** thay `findAll()` bằng phương thức dùng Fetch Join (Chapter 09) hoặc
  `@EntityGraph`, giảm về đúng 1 câu SQL cho toàn bộ endpoint.
- **Prevention:** thêm test tích hợp assert số lượng câu SQL tối đa cho phép với mọi endpoint
  trả về danh sách có quan hệ, chạy trong CI với dữ liệu đủ lớn để phát hiện N+1 trước khi lên
  production.

## Debug Checklist

- [ ] Endpoint trả về danh sách chậm bất thường khi dữ liệu tăng, dù nhanh khi test dữ liệu nhỏ?
      → nghi ngờ hàng đầu là N+1 — bật `show-sql`/`datasource-proxy` để xác nhận.
- [ ] Không chắc N+1 có đang xảy ra trong quá trình serialize JSON hay không? → kiểm tra
      `spring.jpa.open-in-view` và cấu trúc DTO trả về (có trực tiếp trả entity với quan hệ lazy
      không).
- [ ] Cần một cách tự động phát hiện N+1 trước khi deploy? → thêm test tích hợp assert
      `Statistics.getPrepareStatementCount()` cho các endpoint quan trọng.

## Summary

N+1 query xảy ra khi lấy N bản ghi cha (1 query), rồi truy cập quan hệ của từng bản ghi trong
vòng lặp (thêm N query riêng do lazy loading, Chapter 08), tổng cộng N+1 câu SQL thay vì 1. Đã
chứng minh bằng thực nghiệm quan hệ tuyến tính chính xác: 5 Customer → 6 câu, 20 Customer → 21
câu. Vấn đề đặc biệt nguy hiểm vì có thể xảy ra âm thầm khi serialize JSON (nếu `open-in-view`
bật), không cần vòng lặp tường minh trong code nghiệp vụ, và thường "vô hình" khi test với dữ
liệu nhỏ. Giải pháp chính là Fetch Join (Chapter 09), giảm về đúng 1 câu SQL. Nên tích hợp query
count assertion vào test tự động để phát hiện N+1 sớm, trước khi ảnh hưởng production.

## Interview Questions

- Vấn đề N+1 query là gì? Vì sao nó xảy ra, và làm sao phát hiện/khắc phục nó trong thực tế?

**Senior**

- Vì sao N+1 có thể xảy ra ngay cả khi không có vòng lặp tường minh nào trong code nghiệp vụ?
- Đề xuất một chiến lược đầy đủ để phát hiện N+1 trước khi nó ảnh hưởng tới production.

## Exercises

- [ ] Chạy lại `NPlusOneTest` và `NPlusOneScaleTest` ở trên, xác nhận quan hệ tuyến tính chính
      xác giữa N và số câu SQL.
- [ ] Viết một test tích hợp assert `Statistics.getPrepareStatementCount()` không vượt quá 1 cho
      endpoint `findAllWithOrders()`, xác nhận test fail nếu ai đó vô tình đổi lại thành
      `findAll()` + lazy loading.
- [ ] Bật `spring.jpa.open-in-view=false`, thử serialize một entity có quan hệ lazy chưa fetch
      qua REST API, quan sát lỗi rõ ràng thay vì N+1 âm thầm.

## Cheat Sheet

| N (số bản ghi cha) | Số query (N+1, không fetch join) | Số query (có fetch join) |
| --- | --- | --- |
| 5 | 6 | 1 |
| 20 | 21 | 1 |
| 1000 | 1001 | 1 |

**Công thức:** `1 (lấy danh sách cha) + N (một query riêng cho mỗi bản ghi cha)`.

## References

- Hibernate ORM Documentation — N+1 select problem.
- Vlad Mihalcea — "High-Performance Java Persistence" (nguồn tham khảo chuyên sâu về N+1 và tối
  ưu Hibernate).
