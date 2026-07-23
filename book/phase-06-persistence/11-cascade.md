---
tags:
  - JPA
  - Cascade
  - Hibernate
---

# Cascade

> Phase: Phase 6 — Persistence
> Chapter slug: `cascade`

## Metadata

```yaml
Chapter: Cascade
Phase: Phase 6 — Persistence
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 65%
Prerequisites:
  - Chapter 07 — Entity State
Used Later:
  - Thiết kế mô hình quan hệ dữ liệu phức tạp (Phase 8)
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 07](07-entity-state.md) — một entity `Transient` (vừa `new`) cần
> `persist()`/`save()` tường minh để trở thành Managed. Nhưng nếu `Customer` có một `Order` mới
> tạo, gắn vào `List<Order>` của nó — có cần gọi `orderRepo.save(order)` **riêng** không?

```java
Customer c = new Customer("Cascade Test");
c.addOrder(new Order("NEW")); // CHUA HE goi orderRepo.save() rieng
customerRepo.save(c); // CHI save Customer
```

Kết quả thật:

```
Hibernate: insert into customer (name,id) values (?,default)
Hibernate: insert into orders (customer_id,status,version,id) values (?,?,?,default)
c.getOrders().get(0).getId() = 3 (Order DA co ID, du chua goi save() rieng)
```

Chỉ gọi `customerRepo.save(c)` — nhưng cả `Order` cũng được `INSERT` tự động, có ID thật, dù
chưa từng gọi `orderRepo.save()`.

## Interview Question (Central)

> Cascade trong JPA là gì? `orphanRemoval` khác các loại `CascadeType` khác như thế nào?

## Objectives

- [ ] Dùng thành thạo `CascadeType.ALL` để tự động lan truyền thao tác persist/remove từ entity
      cha sang entity con
- [ ] Tự tay chứng minh bằng thực nghiệm: `save()` entity cha tự động `INSERT` cả entity con
      chưa từng được lưu riêng
- [ ] Tự tay chứng minh `orphanRemoval=true`: xoá entity con khỏi collection của cha (không gọi
      `delete()` riêng) tự động xoá nó khỏi database

## Prerequisites

- Chapter 07 — hiểu trạng thái Transient/Managed, nền tảng để hiểu cascade "lan truyền" các
  thao tác chuyển trạng thái đó.

## Used Later

- **Thiết kế mô hình quan hệ dữ liệu phức tạp** (Phase 8) — cascade là công cụ cốt lõi khi thiết
  kế quan hệ cha-con (aggregate) trong domain model.

## Problem

Với một quan hệ "cha sở hữu con" thực sự (ví dụ `Order` sở hữu `OrderItem` — `OrderItem` không
có ý nghĩa tồn tại độc lập nếu tách khỏi `Order`), buộc lập trình viên phải tự tay gọi
`save()`/`delete()` riêng biệt cho **từng** entity con mỗi khi thao tác trên entity cha là lặp
lại không cần thiết, và dễ **quên** — dẫn tới entity con "mồ côi" (tồn tại trong database, không
còn cha nào tham chiếu) hoặc lỗi do quên lưu entity con trước khi lưu cha.

## Concept

**Cascade** (`cascade = CascadeType.ALL`/`PERSIST`/`REMOVE`/...) khai báo trên quan hệ
(`@OneToMany`, `@ManyToOne`, ...) cho phép **lan truyền tự động** một thao tác từ entity cha sang
entity con — gọi `persist()`/`save()` trên cha tự động `persist()` mọi entity con trong quan hệ
đó (nếu có `PERSIST` hoặc `ALL` trong cascade), tương tự với `remove()`. **`orphanRemoval = true`**
là một cơ chế **riêng biệt**, mạnh hơn: tự động `DELETE` một entity con ngay khi nó bị **loại bỏ
khỏi collection** của cha (không cần gọi `remove()` tường minh trên chính entity con đó).

## Why?

Cascade phản ánh đúng ngữ nghĩa quan hệ "sở hữu" (ownership/composition trong thuật ngữ OOP) —
nếu entity con **không có ý nghĩa tồn tại độc lập** khỏi entity cha, việc thao tác trên cha nên
tự động áp dụng cho con, giảm boilerplate và giảm nguy cơ quên đồng bộ giữa hai bên.
`orphanRemoval` đi xa hơn `CascadeType.REMOVE`: `REMOVE` chỉ lan truyền khi **chính entity cha**
bị xoá, còn `orphanRemoval` phản ứng với việc con bị "ly khai" khỏi cha (bị xoá khỏi collection)
ngay cả khi bản thân cha vẫn tồn tại — đúng ngữ nghĩa "con không có cha thì không nên tồn tại".

## How?

```java
@Entity
class Customer {
    @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, orphanRemoval = true)
    List<Order> orders;
}

Customer c = new Customer("A");
c.addOrder(new Order("NEW")); // CHUA persist rieng
customerRepo.save(c); // CascadeType.ALL -> Order TU DONG duoc persist theo

managed.getOrders().remove(someOrder); // orphanRemoval=true -> Order do TU DONG bi DELETE khi commit
```

## Visualization

```
CascadeType.PERSIST/ALL:

  customerRepo.save(customer) ──► Hibernate THAY customer co orders CHUA persist
                                        │
                                   TU DONG persist() TUNG Order trong orders
                                        │
                                   INSERT customer, INSERT moi Order

orphanRemoval = true:

  managed.getOrders().remove(order)  <- CHI thao tac tren List Java trong bo nho
       │
       ▼  luc flush/commit, Hibernate PHAT HIEN Order nay khong con trong collection cua cha
       │
       ▼  TU DONG: DELETE FROM orders WHERE id = ?  (du KHONG goi orderRepo.delete())
```

## Example

**Cascade persist:**

```java
Customer c = new Customer("Cascade Test");
c.addOrder(new Order("NEW"));
customerRepo.save(c);
System.out.println("Order id: " + c.getOrders().get(0).getId());
```

Kết quả thật (Hibernate ORM 6.5.3.Final):

```
Hibernate: insert into customer (name,id) values (?,default)
Hibernate: insert into orders (customer_id,status,version,id) values (?,?,?,default)
c.getOrders().get(0).getId() = 3 (Order DA co ID, du chua goi save() rieng)
```

**orphanRemoval:**

```java
Customer managed = em.find(Customer.class, c.getId());
managed.getOrders().removeIf(o -> o.getId().equals(orderId1)); // CHI xoa khoi List
txManager.commit(tx); // orphanRemoval kich hoat o day

Order stillExists = em.find(Order.class, orderId1);
System.out.println("Sau khi xoa khoi List + commit: " + stillExists);
```

Kết quả thật:

```
Da xoa Order khoi List trong bo nho, CHUA goi orderRepo.delete() nao ca
Hibernate: delete from orders where id=? and version=?
em.find(Order.class, orderId1) SAU KHI xoa khoi List + commit: null
```

Chỉ gọi `List.removeIf(...)` (thao tác Java thuần trên collection trong bộ nhớ) — không hề gọi
`orderRepo.delete()`/`em.remove()` cho `Order` — nhưng lúc `commit()`, Hibernate tự phát hiện
`Order` đó đã "ly khai" khỏi `orders` của `Customer` và tự sinh `DELETE`. Xác nhận bằng
`em.find()` cho `Order` đó trả về `null`.

## Deep Dive

**Vì sao câu `DELETE` trong ví dụ orphanRemoval có điều kiện `and version=?` — điều này liên
quan gì tới các chapter sau?** Đây là dấu hiệu trực tiếp của **Optimistic Locking**
(`@Version`, Chapter 13) — `Order` trong domain model của chapter này có field `@Version`, và
Hibernate tự động thêm điều kiện kiểm tra version vào mọi câu `UPDATE`/`DELETE` để đảm bảo không
xoá/sửa nhầm một bản ghi đã bị thay đổi bởi giao dịch khác kể từ lần đọc gần nhất — một cơ chế
hoàn toàn độc lập với cascade, nhưng tự động áp dụng cho **mọi** thao tác ghi trên entity có
`@Version`, kể cả thao tác do cascade/orphanRemoval kích hoạt.

## Engineering Insight

**Vì sao `CascadeType.ALL` trên quan hệ `@ManyToOne`/`@ManyToMany` thường là một quyết định
thiết kế nguy hiểm, trong khi hoàn toàn hợp lý trên `@OneToMany` (như ví dụ `Customer` →
`Order`)?** Cascade phản ánh đúng ngữ nghĩa "sở hữu" — một `Customer` **sở hữu** các `Order` của
nó (Order không có ý nghĩa tồn tại độc lập khỏi Customer), nên cascade trên hướng
`Customer → Order` là hợp lý. Nhưng nếu đặt `CascadeType.ALL`/`REMOVE` trên phía
`@ManyToOne`/`@ManyToMany` (ví dụ nhiều `Order` cùng tham chiếu một `Product` chung), xoá **một**
`Order` sẽ vô tình cascade xoá luôn `Product` — dù `Product` đó vẫn đang được **các** `Order`
khác tham chiếu! Đây là lỗi thiết kế cascade phổ biến và nguy hiểm nhất: áp dụng cascade cho quan
hệ "chia sẻ" (nhiều entity cùng tham chiếu một entity khác) thay vì chỉ áp dụng đúng cho quan hệ
"sở hữu độc quyền" (composition, đúng 1-nhiều mà con chỉ thuộc về đúng một cha).

## Historical Note

Cascade trong JPA lấy cảm hứng trực tiếp từ khái niệm "cascade" trong ràng buộc khoá ngoại của
SQL chuẩn (`ON DELETE CASCADE`) — nhưng cascade của JPA hoạt động **ở tầng ứng dụng** (Hibernate
tự quyết định khi nào lan truyền, dựa trên trạng thái entity trong Persistence Context), khác
với `ON DELETE CASCADE` của database (hoạt động hoàn toàn ở tầng database, độc lập với bất kỳ
ứng dụng nào). Hai cơ chế này có thể **xung đột hoặc trùng lặp** nếu dùng cùng lúc — một quyết
định kiến trúc cần cân nhắc kỹ khi thiết kế schema.

## Myth vs Reality

- **Myth:** "Phải gọi `save()`/`persist()` riêng cho mọi entity, kể cả entity con trong một quan
  hệ cascade."
  **Reality:** Đã chứng minh bằng thực nghiệm — `CascadeType.ALL`/`PERSIST` tự động lan truyền,
  không cần gọi riêng.

- **Myth:** "`CascadeType.REMOVE` và `orphanRemoval` là hai cách viết khác nhau cho cùng một
  hành vi."
  **Reality:** Khác nhau — `REMOVE` chỉ kích hoạt khi chính entity cha bị xoá; `orphanRemoval`
  kích hoạt ngay khi con bị loại khỏi collection, dù cha vẫn tồn tại (đã chứng minh bằng thực
  nghiệm).

## Common Mistakes

- **Đặt `CascadeType.ALL` trên quan hệ `@ManyToOne`/`@ManyToMany` (quan hệ chia sẻ)** — nguy cơ
  xoá nhầm entity vẫn đang được tham chiếu bởi entity khác, xem Engineering Insight.
- **Quên `orphanRemoval=true` khi thực sự cần "xoá con khi ly khai khỏi cha"** — chỉ dùng
  `CascadeType.REMOVE` không đủ, xoá khỏi collection mà không xoá bản thân entity cha sẽ không
  kích hoạt xoá con.
- **Dùng cascade cho quan hệ không thực sự là "sở hữu"** — vi phạm ngữ nghĩa composition, dẫn
  tới hành vi bất ngờ khi thao tác trên entity cha ảnh hưởng entity con không nên bị ảnh hưởng.

## Best Practices

- Chỉ dùng `CascadeType.ALL`/`orphanRemoval` cho quan hệ thực sự là "sở hữu độc quyền"
  (composition) — con không có ý nghĩa tồn tại độc lập khỏi cha.
- Tránh cascade trên quan hệ chia sẻ (`@ManyToOne`/`@ManyToMany` trỏ tới entity dùng chung).
- Cân nhắc kỹ việc kết hợp cascade JPA với `ON DELETE CASCADE` của database — tránh xung đột
  hoặc trùng lặp logic giữa hai tầng.

## Debug Checklist

- [ ] Entity bị xoá ngoài ý muốn khi xoá một entity khác? → kiểm tra cascade có đang đặt trên
      quan hệ chia sẻ (nhiều entity cùng tham chiếu) không.
- [ ] Xoá entity khỏi collection của cha nhưng nó vẫn còn trong database? → kiểm tra đã bật
      `orphanRemoval=true` chưa — `CascadeType.REMOVE` một mình không đủ cho trường hợp này.
- [ ] Câu `DELETE`/`UPDATE` do cascade sinh ra có điều kiện `version=?` bất ngờ? → bình thường
      nếu entity có `@Version` (Optimistic Locking, Chapter 13).

## Summary

Cascade (`CascadeType.ALL`/`PERSIST`/`REMOVE`) lan truyền tự động thao tác từ entity cha sang
entity con, phản ánh ngữ nghĩa "sở hữu" (composition). Đã chứng minh bằng thực nghiệm:
`customerRepo.save(customer)` tự động `INSERT` cả `Order` con chưa từng `save()` riêng.
`orphanRemoval=true` là cơ chế riêng biệt, mạnh hơn `CascadeType.REMOVE`: xoá entity con khỏi
collection của cha (không gọi `delete()` riêng) tự động `DELETE` nó khi commit — đã chứng minh
trực tiếp qua `em.find()` trả về `null` sau khi chỉ thao tác trên `List` Java. Cascade nguy hiểm
khi áp dụng sai cho quan hệ chia sẻ (`@ManyToOne`/`@ManyToMany`) thay vì chỉ dùng cho quan hệ sở
hữu độc quyền.

## Interview Questions

**Mid**

- Cascade trong JPA là gì?
- `orphanRemoval` khác các loại `CascadeType` khác như thế nào?

**Senior**

- Vì sao đặt `CascadeType.ALL` trên quan hệ `@ManyToOne`/`@ManyToMany` thường là quyết định
  thiết kế nguy hiểm?
- So sánh cascade của JPA (tầng ứng dụng) với `ON DELETE CASCADE` của SQL (tầng database). Rủi
  ro gì nếu dùng cả hai cùng lúc?

## Exercises

- [ ] Chạy lại `CascadeTest` ở trên, xác nhận cả cascade persist và orphanRemoval hoạt động
      đúng như mô tả.
- [ ] Thử đặt `CascadeType.ALL` trên một quan hệ `@ManyToMany` giả lập (ví dụ `Order` và
      `Product` dùng chung), xoá một `Order`, quan sát `Product` có bị xoá nhầm không.
- [ ] Viết một entity `Order` có `@Version`, xoá một `Order` qua orphanRemoval trong khi một
      transaction khác đang sửa cùng `Order` đó, quan sát `OptimisticLockException` (Chapter 13).

## Cheat Sheet

| | `CascadeType.PERSIST` | `CascadeType.REMOVE` | `orphanRemoval=true` |
| --- | --- | --- | --- |
| Kích hoạt khi | `persist()`/`save()` cha | `remove()` cha | Con bị loại khỏi collection của cha |
| Cha có bị xoá không? | N/A | Có (đây là điều kiện) | Không cần |
| Con tự động persist/xoá theo | Có | Có (khi cha bị xoá) | Có (dù cha vẫn tồn tại) |

## References

- Jakarta Persistence Specification — Cascading Operations.
- Hibernate ORM Documentation — Cascade types, `orphanRemoval`.
