---
tags:
  - JPA
  - LazyLoading
  - Hibernate
---

# Lazy Loading

> Phase: Phase 6 — Persistence
> Chapter slug: `lazy-loading`

## Metadata

```yaml
Chapter: Lazy Loading
Phase: Phase 6 — Persistence
Difficulty: ★★★★
Importance: ★★★★★
Interview Frequency: 85%
Prerequisites:
  - Chapter 05 — Persistence Context
  - Chapter 07 — Entity State
Used Later:
  - Chapter 09 — Fetch Join
  - Chapter 10 — N+1
Estimated Reading: 18 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: [Chapter 07](07-entity-state.md) — entity Detached không còn được Persistence Context
> theo dõi. Nếu entity đó có một quan hệ `@OneToMany` **chưa từng được truy cập**, và bạn cố
> truy cập nó **sau khi** đã Detached — điều gì xảy ra?

```java
Customer detachedLater = em.find(Customer.class, id);
txManager.commit(tx2); // transaction DONG o day, entity tro thanh DETACHED

detachedLater.getOrders().size(); // truy cap SAU KHI transaction da dong
```

Kết quả thật:

```
LazyInitializationException: failed to lazily initialize a collection of role:
com.handbook.demo.jpa.Customer.orders: could not initialize proxy - no Session
```

So sánh với truy cập **trong** transaction:

```
Hibernate.isInitialized(orders) NGAY SAU find(): false
Hibernate: select o1_0.customer_id,o1_0.id,o1_0.status,o1_0.version from orders o1_0 where o1_0.customer_id=?
orders.size() TRONG transaction: 1 (lazy load THANH CONG)
```

Cùng một dòng code (`getOrders().size()`), nhưng kết quả hoàn toàn khác nhau tuỳ vào entity còn
Managed hay đã Detached.

## Interview Question (Central)

> Lazy Loading trong JPA là gì? `LazyInitializationException` xảy ra khi nào, và làm sao khắc
> phục?

## Objectives

- [ ] Hiểu cơ chế lazy loading: quan hệ `FetchType.LAZY` không load ngay, chỉ load khi thực sự
      truy cập
- [ ] Tự tay chứng minh bằng thực nghiệm: `Hibernate.isInitialized()` trả về `false` ngay sau
      `find()`, và truy cập collection lazy trong transaction hoạt động bình thường
- [ ] Tự tay tái hiện `LazyInitializationException` khi truy cập lazy collection sau khi entity
      đã Detached

## Prerequisites

- Chapter 05 — hiểu Persistence Context, nơi lazy proxy cần một Session còn "sống" để load dữ
  liệu thật.
- Chapter 07 — hiểu trạng thái Detached, điều kiện chính xác gây ra
  `LazyInitializationException`.

## Used Later

- **Chapter 09 (Fetch Join)** — một trong các giải pháp chính để tránh
  `LazyInitializationException`, chủ động load trước dữ liệu cần dùng.
- **Chapter 10 (N+1)** — lazy loading, nếu dùng không cẩn thận trong vòng lặp, chính là nguyên
  nhân trực tiếp gây ra vấn đề N+1.

## Problem

Load một entity kèm **toàn bộ** dữ liệu liên quan tới nó (mọi quan hệ `@OneToMany`/`@ManyToOne`)
ngay lập tức, dù phần lớn trường hợp code chỉ cần dữ liệu entity gốc, là lãng phí hiệu năng
nghiêm trọng — đặc biệt với các quan hệ có thể chứa hàng nghìn bản ghi liên quan. Nhưng nếu trì
hoãn việc load đó, cần một cơ chế đảm bảo dữ liệu thật sự có sẵn **đúng lúc** code cần dùng tới
nó — và báo lỗi rõ ràng nếu không thể làm được điều đó.

## Concept

**Lazy Loading** (`FetchType.LAZY`, mặc định cho `@OneToMany`/`@ManyToMany`) trì hoãn việc load
dữ liệu quan hệ cho tới khi code thực sự **truy cập** nó (gọi `getOrders()`, `.size()`, lặp qua
collection, ...) — tại thời điểm đó, Hibernate mới thực sự chạy một câu `SELECT` bổ sung để lấy
dữ liệu. Việc này chỉ khả thi khi entity vẫn **Managed** (Persistence Context, tức "Session" của
Hibernate, còn tồn tại) — nếu entity đã **Detached** (Chapter 07), Hibernate không còn cách nào
kết nối tới database để thực hiện lazy load đó, dẫn tới `LazyInitializationException`.

## Why?

Lazy loading là sự đánh đổi có chủ đích giữa hiệu năng (không load dữ liệu không cần thiết) và
sự phức tạp thêm vào (phải quản lý đúng thời điểm entity còn Managed khi truy cập quan hệ lazy).
Hibernate hiện thực hoá bằng cách trả về một **proxy** (đối tượng "vỏ rỗng", tương tự cơ chế CGLIB
proxy đã gặp nhiều lần ở Phase 5) thay vì dữ liệu thật — proxy này giữ một tham chiếu tới Session
tạo ra nó; khi bị truy cập lần đầu, nó dùng Session đó để chạy câu `SELECT` thật, "lấp đầy" chính
nó với dữ liệu thật. Nếu Session đó đã đóng (entity Detached), proxy không còn cách nào lấy dữ
liệu — nó **buộc phải** báo lỗi thay vì âm thầm trả về dữ liệu sai hoặc rỗng, tránh việc code
nhầm tưởng "không có order nào" trong khi thực ra chỉ là chưa load được.

## How?

```java
@Entity
class Customer {
    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY) // MAC DINH cho @OneToMany
    List<Order> orders;
}

Customer c = em.find(Customer.class, id); // Customer duoc load NGAY, orders thi CHUA
Hibernate.isInitialized(c.getOrders()); // false - orders VAN la proxy rong

c.getOrders().size(); // TRUY CAP LAN DAU -> Hibernate chay SELECT that de load orders
```

## Visualization

```
Truy cap TRONG transaction (entity con MANAGED):

  em.find(Customer.class, id) ──► Customer load NGAY, orders la PROXY rong (chua init)
  c.getOrders().size() ──► Session VAN CON SONG ──► SELECT that ──► orders duoc "lap day"
                                                                       ──► size() tra ve dung

Truy cap SAU KHI transaction dong (entity da DETACHED):

  em.find(Customer.class, id) ──► Customer load NGAY, orders la PROXY rong
  txManager.commit() ──► Session DONG, Persistence Context bien mat
  c.getOrders().size() ──► Session KHONG CON ──► LazyInitializationException!
```

## Example

```java
Customer c = new Customer("Lazy Test");
c.addOrder(new Order("NEW"));
customerRepo.save(c);
Long id = c.getId();
txManager.commit(setup);

System.out.println("=== TRONG transaction ===");
TransactionStatus tx = txManager.getTransaction(new DefaultTransactionDefinition());
Customer inside = em.find(Customer.class, id);
System.out.println("isInitialized NGAY SAU find(): " + Hibernate.isInitialized(inside.getOrders()));
int size = inside.getOrders().size();
System.out.println("orders.size() TRONG transaction: " + size);
txManager.commit(tx);

System.out.println("=== SAU KHI transaction dong ===");
TransactionStatus tx2 = txManager.getTransaction(new DefaultTransactionDefinition());
Customer detachedLater = em.find(Customer.class, id);
txManager.commit(tx2);
try {
    detachedLater.getOrders().size();
} catch (LazyInitializationException e) {
    System.out.println("LazyInitializationException: " + e.getMessage());
}
```

Kết quả thật (Hibernate ORM 6.5.3.Final):

```
=== TRONG transaction ===
Hibernate: select c1_0.id,c1_0.name from customer c1_0 where c1_0.id=?
isInitialized NGAY SAU find(): false
Hibernate: select o1_0.customer_id,o1_0.id,o1_0.status,o1_0.version from orders o1_0 where o1_0.customer_id=?
orders.size() TRONG transaction: 1 (lazy load THANH CONG)

=== SAU KHI transaction dong ===
Hibernate: select c1_0.id,c1_0.name from customer c1_0 where c1_0.id=?
LazyInitializationException: failed to lazily initialize a collection of role:
com.handbook.demo.jpa.Customer.orders: could not initialize proxy - no Session
```

Ngay sau `find()`, `orders` **chưa** initialize (`false`). Truy cập `.size()` trong transaction
kích hoạt một câu `SELECT` bổ sung thành công. Với cùng thao tác nhưng **sau khi** transaction đã
đóng, chính xác cùng một dòng code (`getOrders().size()`) ném `LazyInitializationException` —
thông điệp lỗi nói rõ nguyên nhân: `"could not initialize proxy - no Session"`.

## Deep Dive

**Vì sao thông điệp lỗi cụ thể nói "no Session" — "Session" ở đây chính xác là gì?** "Session"
là thuật ngữ gốc của Hibernate cho khái niệm mà JPA gọi là `EntityManager` (đã thấy ở Chapter 01,
`em.getDelegate()` trả về `SessionImpl`) — nó đại diện cho **kết nối tới database và Persistence
Context** đang hoạt động. Khi transaction kết thúc, Session đó bị đóng (hoặc trả về connection
pool) — proxy lazy (giữ tham chiếu tới đúng Session đó) không còn cách nào "hỏi" Session để lấy
dữ liệu, dẫn tới thông điệp lỗi chính xác này. Đây là lý do lazy loading về bản chất là một
**ràng buộc thời gian** (temporal coupling) — code truy cập quan hệ lazy phải chạy trong đúng
"cửa sổ thời gian" khi Session/transaction vẫn còn sống, một ràng buộc dễ vi phạm khi kiến trúc
ứng dụng tách rời tầng load dữ liệu và tầng sử dụng dữ liệu (ví dụ Controller dùng dữ liệu
Service đã load nhưng transaction đã đóng).

## Engineering Insight

**`spring.jpa.open-in-view` (mặc định `true` trong Spring Boot — đã thấy cảnh báo này trong log
khởi động xuyên suốt Phase 6) "giải quyết" vấn đề `LazyInitializationException` như thế nào,
và vì sao nó gây tranh cãi trong cộng đồng?** OSIV (Open Session In View) mở rộng vòng đời
Session/EntityManager ra **toàn bộ** thời gian xử lý một HTTP request (không chỉ trong phạm vi
transaction) — nghĩa là lazy loading vẫn hoạt động được ngay cả ở tầng view/Controller, sau khi
transaction của Service đã commit, miễn còn trong cùng request. Điều này "che giấu" vấn đề
`LazyInitializationException` trong nhiều trường hợp, nhưng đánh đổi lấy hai rủi ro: (1) kết nối
database bị giữ lâu hơn cần thiết (suốt cả request, không chỉ lúc cần ghi/đọc dữ liệu), tốn tài
nguyên connection pool dưới tải cao; (2) lazy loading có thể xảy ra **âm thầm** ở tầng view (ví
dụ khi serialize entity thành JSON, Jackson vô tình truy cập một quan hệ lazy), gây ra N+1 query
(Chapter 10) khó phát hiện vì không có exception nào cảnh báo. Nhiều chuyên gia Spring khuyến
nghị **tắt** `open-in-view` (`spring.jpa.open-in-view=false`) trong production, chấp nhận
`LazyInitializationException` xuất hiện rõ ràng hơn để buộc thiết kế đúng (dùng Fetch Join,
Chapter 09, thay vì dựa vào OSIV).

## Historical Note

`FetchType.LAZY` mặc định cho `@OneToMany`/`@ManyToMany` (nhưng `FetchType.EAGER` mặc định cho
`@ManyToOne`/`@OneToOne`) phản ánh giả định hợp lý của JPA: quan hệ "một-nhiều" thường chứa số
lượng lớn bản ghi (không nên load ngay), trong khi quan hệ "nhiều-một"/"một-một" thường chỉ một
bản ghi liên quan (chi phí load ngay thấp hơn nhiều). OSIV (Open Session In View) là một pattern
tồn tại từ trước Spring Boot rất lâu (các framework Java EE khác cũng có khái niệm tương tự) —
tranh cãi về việc nên bật hay tắt nó mặc định đã kéo dài nhiều năm trong cộng đồng Hibernate/
Spring, phản ánh đúng bản chất đánh đổi không có câu trả lời tuyệt đối.

## Myth vs Reality

- **Myth:** "`FetchType.EAGER` luôn là lựa chọn an toàn hơn `LAZY`, tránh được
  `LazyInitializationException`."
  **Reality:** `EAGER` chỉ "che giấu" vấn đề bằng cách luôn load mọi thứ ngay — có thể gây tải
  dữ liệu khổng lồ không cần thiết, và là nguyên nhân trực tiếp của N+1 query (Chapter 10) khi
  không kiểm soát cẩn thận.

- **Myth:** "`spring.jpa.open-in-view=true` (mặc định) là giải pháp triệt để cho
  `LazyInitializationException`."
  **Reality:** Xem Engineering Insight — nó chỉ mở rộng "cửa sổ thời gian" cho phép lazy
  loading, đánh đổi lấy rủi ro giữ connection lâu và N+1 âm thầm ở tầng view.

## Common Mistakes

- **Trả entity có quan hệ lazy trực tiếp làm response API, dựa vào OSIV để "may mắn" không lỗi**
  — dễ gây N+1 âm thầm khi serialize (Chapter 10), và phụ thuộc vào một cấu hình có thể bị tắt
  sau này.
- **Đổi `FetchType.LAZY` thành `EAGER` để "sửa" `LazyInitializationException` mà không cân nhắc
  hậu quả hiệu năng** — chỉ nên coi đây là giải pháp tạm thời, không phải sửa đúng gốc rễ.
- **Không biết `Hibernate.isInitialized()` để kiểm tra trạng thái một quan hệ lazy khi debug** —
  bỏ lỡ công cụ chẩn đoán hữu ích.

## Debug Checklist

- [ ] `LazyInitializationException: could not initialize proxy - no Session`? → kiểm tra entity
      có đang Detached không (transaction đã kết thúc trước khi truy cập quan hệ lazy).
- [ ] Không chắc một quan hệ đã load hay chưa? → dùng `Hibernate.isInitialized(collection)`.
- [ ] Nghi ngờ N+1 query âm thầm do OSIV? → kiểm tra `spring.jpa.open-in-view`, cân nhắc tắt và
      dùng Fetch Join (Chapter 09) tường minh thay thế.

## Summary

Lazy Loading (`FetchType.LAZY`, mặc định cho `@OneToMany`/`@ManyToMany`) trì hoãn việc load dữ
liệu quan hệ tới khi thực sự truy cập, dùng proxy giữ tham chiếu Session để load "đúng lúc". Đã
chứng minh bằng thực nghiệm: `Hibernate.isInitialized()` trả về `false` ngay sau `find()`; truy
cập trong transaction kích hoạt SELECT bổ sung thành công; truy cập **sau** khi transaction đã
đóng (entity Detached) ném `LazyInitializationException: could not initialize proxy - no
Session`. `spring.jpa.open-in-view` (mặc định bật) mở rộng vòng đời Session ra toàn request,
"che giấu" vấn đề nhưng đánh đổi lấy rủi ro giữ connection lâu và N+1 âm thầm — nhiều chuyên gia
khuyến nghị tắt trong production, dùng Fetch Join (Chapter 09) tường minh thay thế.

## Interview Questions

- Lazy Loading trong JPA là gì? `LazyInitializationException` xảy ra khi nào, và làm sao khắc
  phục?

**Senior**

- `spring.jpa.open-in-view` giải quyết vấn đề gì? Vì sao nó gây tranh cãi, và khi nào nên tắt
  nó?
- So sánh đánh đổi giữa `FetchType.LAZY` và `FetchType.EAGER` cho một quan hệ `@OneToMany`.

## Exercises

- [ ] Chạy lại `LazyLoadingTest` ở trên, xác nhận kết quả đúng như mô tả cho cả hai kịch bản.
- [ ] Tắt `spring.jpa.open-in-view`, thử serialize một entity có quan hệ lazy thành JSON qua REST
      API (Phase 5, Chapter 09), quan sát `LazyInitializationException` xuất hiện rõ ràng thay
      vì bị OSIV che giấu.
- [ ] Dùng `Hibernate.isInitialized()` để viết một helper method kiểm tra an toàn trước khi truy
      cập một quan hệ lazy, tránh exception không mong muốn.

## Cheat Sheet

| | Trong transaction (Managed) | Sau khi transaction đóng (Detached) |
| --- | --- | --- |
| Truy cập quan hệ `LAZY` chưa load | SELECT bổ sung, thành công | `LazyInitializationException` |
| `Hibernate.isInitialized()` | `false` trước truy cập, `true` sau | Không đổi (vẫn `false`) |

**Mặc định `FetchType`:** `@OneToMany`/`@ManyToMany` → `LAZY`; `@ManyToOne`/`@OneToOne` →
`EAGER`.

## References

- Jakarta Persistence Specification — Fetching Strategies.
- Hibernate ORM Documentation — Lazy loading, Open Session in View.
