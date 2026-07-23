---
tags:
  - JPA
  - FetchJoin
  - Hibernate
---

# Fetch Join

> Phase: Phase 6 — Persistence
> Chapter slug: `fetch-join`

## Metadata

```yaml
Chapter: Fetch Join
Phase: Phase 6 — Persistence
Difficulty: ★★★★
Importance: ★★★★★
Interview Frequency: 75%
Prerequisites:
  - Chapter 03 — Repository
  - Chapter 08 — Lazy Loading
Used Later:
  - Chapter 10 — N+1
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 08](08-lazy-loading.md) — lazy loading trì hoãn việc load quan hệ tới khi
> thực sự truy cập. Nhưng nếu bạn **biết trước** mình sẽ cần dữ liệu quan hệ đó cho **mọi** bản
> ghi trong một danh sách, trì hoãn từng cái một có thực sự tối ưu?

So sánh trực tiếp: lấy 5 `Customer`, mỗi Customer có 1 `Order`, rồi đọc `orders` của từng
Customer — một lần dùng lazy loading thông thường, một lần dùng **fetch join**:

```
=== VAN DE N+1 ===
Tong so query SQL thuc thi (N+1): 6

=== FIX bang Fetch Join ===
Hibernate: select distinct c1_0.id,c1_0.name,o1_0.customer_id,o1_0.id,o1_0.status,o1_0.version
           from customer c1_0 join orders o1_0 on c1_0.id=o1_0.customer_id
Tong so query SQL thuc thi (fetch join): 1
```

Cùng lấy đúng 5 Customer + 5 Order — nhưng **6 câu SQL** so với **1 câu SQL duy nhất**.

## Interview Question (Central)

> Fetch Join trong JPQL là gì? Nó khác `JOIN` thông thường như thế nào?

## Objectives

- [ ] Dùng thành thạo `JOIN FETCH` trong JPQL để load quan hệ ngay trong cùng một câu truy vấn
- [ ] Tự tay chứng minh bằng thực nghiệm: fetch join giảm từ 6 câu SQL xuống còn 1 câu duy nhất
      cho cùng một tập dữ liệu
- [ ] Phân biệt `JOIN FETCH` (thay đổi dữ liệu trả về, load luôn quan hệ) và `JOIN` thông thường
      (chỉ dùng để lọc điều kiện, không load quan hệ)

## Prerequisites

- Chapter 03 — hiểu `@Query`, cú pháp JPQL cơ bản.
- Chapter 08 — hiểu lazy loading, vấn đề cốt lõi mà fetch join giải quyết.

## Used Later

- **Chapter 10 (N+1)** — fetch join là giải pháp chính để khắc phục vấn đề N+1 query.

## Problem

Với lazy loading (Chapter 08), truy cập quan hệ của **từng** entity trong một danh sách (ví dụ
lặp qua 5 `Customer`, đọc `orders` của mỗi cái) sinh ra **một câu SQL riêng cho mỗi entity** —
với danh sách càng lớn, số câu SQL càng tăng tuyến tính, dù về bản chất toàn bộ dữ liệu cần
thiết hoàn toàn có thể lấy được trong **một** câu truy vấn duy nhất bằng `JOIN`.

## Concept

**Fetch Join** (`JOIN FETCH` trong JPQL) là một chỉ thị đặc biệt yêu cầu Hibernate **load luôn**
dữ liệu của quan hệ liên kết **trong cùng** câu truy vấn chính, thay vì để nó ở trạng thái lazy
proxy chờ load riêng sau này. Khác với `JOIN` thông thường (chỉ dùng để **lọc điều kiện** dựa
trên bảng liên kết, không thay đổi cấu trúc dữ liệu Entity trả về), `JOIN FETCH` **thay đổi**
kết quả: entity trả về đã có sẵn quan hệ được "lấp đầy" hoàn toàn (`Hibernate.isInitialized() ==
true` ngay lập tức), không cần thêm bất kỳ câu SQL nào khi truy cập sau đó.

## Why?

Fetch join tận dụng khả năng của SQL `JOIN` để lấy dữ liệu từ nhiều bảng liên kết **trong một
lần round-trip** tới database — thay vì N+1 lần round-trip riêng biệt (Chapter 10). Về bản chất,
đây là đánh đổi ngược lại với lazy loading: chấp nhận trả về **nhiều dữ liệu hơn cần thiết ngay
lúc đó** (nếu code cuối cùng không dùng tới quan hệ đó) để đổi lấy **giảm số lượng round-trip**
mạng tới database — với những trường hợp biết chắc sẽ cần dùng quan hệ đó cho toàn bộ (hoặc phần
lớn) bản ghi trong kết quả, đây gần như luôn là đánh đổi có lợi.

## How?

```java
// JOIN thong thuong: CHI loc dieu kien, KHONG load orders vao ket qua
@Query("select c from Customer c join c.orders o where o.status = 'NEW'")
List<Customer> findByOrderStatus(String status);

// JOIN FETCH: load LUON orders vao ket qua, KHONG con lazy proxy rong
@Query("select distinct c from Customer c join fetch c.orders")
List<Customer> findAllWithOrders();
```

## Visualization

```
JOIN thuong (chi loc):

  SELECT c FROM Customer c JOIN c.orders o WHERE o.status = 'NEW'
       │
       ▼  ket qua: List<Customer>, nhung c.orders VAN la LAZY PROXY RONG
          (JOIN chi dung de LOC, khong anh huong du lieu tra ve)

JOIN FETCH (load luon):

  SELECT c FROM Customer c JOIN FETCH c.orders
       │
       ▼  ket qua: List<Customer>, va c.orders DA DUOC LAP DAY SAN
          (KHONG con la proxy rong, truy cap KHONG can SELECT them)
```

## Example

```java
// Khong fetch join - N+1
List<Customer> customers = customerRepo.findAll(); // 1 query
long total1 = 0;
for (Customer c : customers) {
    total1 += c.getOrders().size(); // MOI Customer them 1 query rieng
}

// Co fetch join
@Query("select distinct c from Customer c join fetch c.orders")
List<Customer> findAllWithOrders();

List<Customer> withOrders = customerRepo.findAllWithOrders();
long total2 = 0;
for (Customer c : withOrders) {
    total2 += c.getOrders().size(); // KHONG THEM query nao, da fetch join san
}
```

Kết quả thật (Hibernate ORM 6.5.3.Final, 5 Customer, mỗi Customer 1 Order):

```
=== Khong fetch join ===
Hibernate: select c1_0.id,c1_0.name from customer c1_0
Hibernate: select o1_0.customer_id,... from orders o1_0 where o1_0.customer_id=?
Hibernate: select o1_0.customer_id,... from orders o1_0 where o1_0.customer_id=?
Hibernate: select o1_0.customer_id,... from orders o1_0 where o1_0.customer_id=?
Hibernate: select o1_0.customer_id,... from orders o1_0 where o1_0.customer_id=?
Hibernate: select o1_0.customer_id,... from orders o1_0 where o1_0.customer_id=?
Tong so query SQL thuc thi (N+1): 6
Tong so order dem duoc: 5

=== Co fetch join ===
Hibernate: select distinct c1_0.id,c1_0.name,o1_0.customer_id,o1_0.id,o1_0.status,o1_0.version
           from customer c1_0 join orders o1_0 on c1_0.id=o1_0.customer_id
Tong so query SQL thuc thi (fetch join): 1
Tong so order dem duoc: 5
```

Cả hai đều đếm đúng **5 order** — kết quả logic giống hệt nhau. Nhưng số câu SQL thực thi khác
biệt hoàn toàn: **6 câu** (1 cho danh sách Customer + 5 câu riêng cho từng Customer's orders) so
với **1 câu duy nhất** khi dùng `JOIN FETCH` — chỉ thay đổi cách viết JPQL, không đổi logic
nghiệp vụ.

## Deep Dive

**Vì sao câu `JOIN FETCH` cần `distinct` trong ví dụ trên?** Khi `JOIN` một `Customer` với nhiều
`Order` (quan hệ một-nhiều), SQL trả về **một dòng cho mỗi cặp** Customer-Order — nếu một
Customer có 3 Order, kết quả SQL thô có **3 dòng** đều mang cùng dữ liệu Customer (chỉ khác phần
Order). Hibernate tự động "gộp" các dòng trùng lặp đó thành đúng một object `Customer` Java (nhờ
Persistence Context, Chapter 05, đảm bảo cùng ID → cùng object), nhưng nếu không có `distinct`
trong JPQL, **danh sách kết quả trả về** (`List<Customer>`) vẫn có thể chứa **nhiều lần cùng một
Customer** (dù là cùng object reference) — `distinct` trong JPQL loại bỏ các bản sao đó ở tầng
kết quả cuối cùng, đảm bảo mỗi Customer chỉ xuất hiện đúng một lần trong danh sách trả về.

## Engineering Insight

**Fetch join có nhược điểm gì, và khi nào KHÔNG nên dùng nó?** Nhược điểm chính: (1) fetch join
nhiều quan hệ `@OneToMany` cùng lúc trong một câu truy vấn (ví dụ Customer join fetch Orders join
fetch OrderItems) tạo ra **tích Descartes** (Cartesian product) — số dòng SQL thô tăng theo cấp
số nhân, có thể trả về lượng dữ liệu khổng lồ, tệ hơn cả N+1 ban đầu; JPA thực tế **giới hạn**
chỉ cho phép fetch join **một** collection tại một thời điểm trong cùng câu truy vấn (fetch join
quan hệ thứ hai sẽ gây lỗi `MultipleBagFetchException` với Hibernate). (2) Fetch join không phù
hợp khi chỉ cần dữ liệu quan hệ cho **một phần nhỏ** kết quả (ví dụ chỉ 1 trong 1000 Customer
thực sự cần xem Orders) — tải toàn bộ dữ liệu quan hệ cho mọi bản ghi là lãng phí ngược lại. Quy
tắc thực tế: dùng fetch join khi **biết chắc** sẽ cần dữ liệu quan hệ đó cho phần lớn/toàn bộ kết
quả; với trường hợp chỉ cần một phần nhỏ, cân nhắc lazy loading thông thường hoặc truy vấn riêng
biệt theo nhu cầu thực tế.

## Historical Note

`JOIN FETCH` là cú pháp thuộc chuẩn JPQL (từ đặc tả JPA 2006), không phải mở rộng riêng của
Hibernate — mọi implementation JPA tuân thủ đặc tả đều hỗ trợ cú pháp này theo cùng ngữ nghĩa.
`@EntityGraph` (Spring Data JPA, một cách khai báo khác để đạt hiệu quả tương tự mà không cần
viết JPQL tường minh) xuất hiện muộn hơn, như một lựa chọn thay thế gọn hơn cho các trường hợp
đơn giản.

## Myth vs Reality

- **Myth:** "`JOIN` và `JOIN FETCH` trong JPQL có cùng tác dụng, chỉ khác cú pháp."
  **Reality:** Đã giải thích ở Concept — `JOIN` chỉ lọc điều kiện, không thay đổi dữ liệu Entity
  trả về; `JOIN FETCH` thực sự load và "lấp đầy" quan hệ vào kết quả.

- **Myth:** "Luôn nên dùng fetch join cho mọi truy vấn có quan hệ, để tránh N+1 hoàn toàn."
  **Reality:** Xem Engineering Insight — fetch join nhiều collection cùng lúc gây tích Descartes,
  và không phù hợp khi chỉ cần dữ liệu quan hệ cho một phần nhỏ kết quả.

## Common Mistakes

- **Quên `distinct` khi fetch join quan hệ một-nhiều** — kết quả có thể chứa nhiều bản sao cùng
  một entity cha (dù là cùng reference).
- **Cố fetch join nhiều collection cùng lúc trong một câu JPQL** — gây tích Descartes hoặc lỗi
  `MultipleBagFetchException`.
- **Dùng fetch join cho mọi truy vấn "cho chắc", không cân nhắc liệu có thực sự cần dữ liệu quan
  hệ đó không** — tải dữ liệu thừa không cần thiết.

## Best Practices

- Dùng fetch join khi biết chắc sẽ cần dữ liệu quan hệ cho phần lớn/toàn bộ kết quả truy vấn.
- Luôn thêm `distinct` khi fetch join quan hệ một-nhiều để tránh kết quả trùng lặp.
- Chỉ fetch join **một** collection tại một thời điểm trong cùng câu truy vấn; với nhu cầu load
  nhiều quan hệ lồng nhau, cân nhắc tách thành nhiều truy vấn riêng hoặc dùng `@EntityGraph`.

## Debug Checklist

- [ ] Kết quả truy vấn có nhiều bản sao trùng lặp của cùng một entity? → kiểm tra đã thêm
      `distinct` khi fetch join quan hệ một-nhiều chưa.
- [ ] Lỗi `MultipleBagFetchException`? → đang cố fetch join nhiều hơn một collection trong cùng
      câu truy vấn — tách thành nhiều truy vấn riêng.
- [ ] Nghi ngờ fetch join trả về lượng dữ liệu khổng lồ bất thường? → kiểm tra có đang fetch
      join một quan hệ có số lượng bản ghi liên quan rất lớn, gây tích Descartes.

## Summary

Fetch Join (`JOIN FETCH` trong JPQL) load luôn dữ liệu quan hệ trong cùng câu truy vấn chính,
khác với `JOIN` thông thường (chỉ lọc điều kiện, không thay đổi dữ liệu trả về). Đã chứng minh
bằng thực nghiệm: cùng lấy 5 Customer + 5 Order, cách thông thường (lazy loading từng cái) tốn 6
câu SQL, fetch join chỉ tốn đúng 1 câu — giảm số round-trip database từ N+1 xuống còn 1
(Chapter 10). Cần `distinct` khi fetch join quan hệ một-nhiều để tránh kết quả trùng lặp, và chỉ
nên fetch join một collection tại một thời điểm để tránh tích Descartes.

## Interview Questions

**Mid**

- Fetch Join trong JPQL là gì? Nó khác `JOIN` thông thường như thế nào?
- Vì sao cần `distinct` khi dùng `JOIN FETCH` cho quan hệ một-nhiều?

**Senior**

- Fetch join có nhược điểm gì? Khi nào không nên dùng nó?
- Vì sao JPA/Hibernate giới hạn chỉ fetch join được một collection tại một thời điểm trong cùng
  câu truy vấn?

## Exercises

- [ ] Chạy lại `NPlusOneTest` ở trên, xác nhận số câu SQL giảm từ 6 xuống 1 khi dùng fetch join.
- [ ] Thử fetch join hai collection cùng lúc (ví dụ `Customer` join fetch `orders` join fetch
      `orders.items`), quan sát lỗi hoặc kích thước kết quả tăng bất thường.
- [ ] Tìm hiểu `@EntityGraph` của Spring Data JPA, viết lại `findAllWithOrders()` dùng
      `@EntityGraph` thay vì JPQL tường minh, so sánh SQL sinh ra.

## Cheat Sheet

| | `JOIN` | `JOIN FETCH` |
| --- | --- | --- |
| Mục đích | Lọc điều kiện | Load luôn dữ liệu quan hệ |
| Ảnh hưởng tới entity trả về | Không | Có — quan hệ được "lấp đầy" |
| Cần `distinct` (quan hệ 1-N)? | Tuỳ trường hợp | Nên có |
| Fetch nhiều collection cùng lúc | Không áp dụng | Không được (tích Descartes/lỗi) |

## References

- Jakarta Persistence Specification — JPQL FETCH JOIN.
- Hibernate ORM Documentation — Fetching strategies, `@EntityGraph`.
