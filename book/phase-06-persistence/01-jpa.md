---
tags:
  - JPA
  - Hibernate
  - Persistence
---

# JPA (Java Persistence API)

> Phase: Phase 6 — Persistence
> Chapter slug: `jpa`

## Metadata

```yaml
Chapter: JPA
Phase: Phase 6 — Persistence
Difficulty: ★★★
Importance: ★★★★★
Interview Frequency: 85%
Prerequisites:
  - Phase 1, Chapter 08 — OOP Fundamentals (interface vs implementation)
  - Phase 5, Chapter 08 — Spring Boot
Used Later:
  - Chapter 02 — Entity
  - Chapter 03 — Repository
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: Phase 1, Chapter 08 — một `interface` chỉ định nghĩa **hợp đồng** (contract), không
> có logic thật. Code gọi tới `jakarta.persistence.EntityManager` (một interface) — nhưng thực
> thi thật sự nằm ở đâu?

In ra kiểu thật sự của instance đang chạy đằng sau `EntityManager`:

```java
System.out.println("Kieu khai bao (interface): jakarta.persistence.EntityManager");
System.out.println("Instance THAT SU luc runtime: " + em.getClass().getName());
System.out.println("em.getDelegate(): " + em.getDelegate().getClass().getName());
```

Kết quả thật:

```
Kieu khai bao (interface): jakarta.persistence.EntityManager
Instance THAT SU luc runtime: jdk.proxy2.$Proxy135
em.getDelegate(): org.hibernate.internal.SessionImpl
```

Code chỉ biết tới interface `EntityManager` — nhưng đằng sau lớp vỏ proxy (Spring bọc thêm để
quản lý transaction-scoped entity manager, tương tự cơ chế đã học ở Phase 5, Chapter 12),
implementation thật sự là `org.hibernate.internal.SessionImpl` — **Hibernate**.

## Interview Question (Central)

> JPA là gì? JPA khác Hibernate như thế nào?

## Objectives

- [ ] Phân biệt chính xác JPA (đặc tả/specification) và Hibernate (một implementation cụ thể)
- [ ] Tự tay chứng minh bằng thực nghiệm: `EntityManager` (interface JPA) thực chất được backing
      bởi `SessionImpl` (class Hibernate) lúc runtime
- [ ] Hiểu vai trò của Hibernate ORM trong việc tự động sinh DDL và SQL từ entity Java

## Prerequisites

- Phase 1, Chapter 08 — hiểu khái niệm interface tách biệt khỏi implementation.
- Phase 5, Chapter 08 — hiểu Spring Boot, nền tảng auto-configuration cho JPA.

## Used Later

- **Chapter 02 (Entity)** — cách định nghĩa một class Java để JPA/Hibernate ánh xạ vào bảng
  database.
- **Chapter 03 (Repository)** — Spring Data JPA xây dựng trên nền tảng JPA đã học ở chapter này.

## Problem

Trước JPA (2006), mỗi ORM framework trên Java (Hibernate, TopLink, iBATIS) có API riêng biệt,
không tương thích lẫn nhau — code viết cho Hibernate không thể chuyển sang TopLink mà không viết
lại phần lớn logic truy cập dữ liệu. Điều này khiến việc **đổi implementation ORM** (ví dụ vì lý
do license, hiệu năng, hoặc hỗ trợ dài hạn) trở thành một dự án tái cấu trúc lớn, tốn kém.

## Concept

**JPA (Java Persistence API)** là một **đặc tả** (specification, thuộc Jakarta EE) định nghĩa
tập hợp interface và annotation chuẩn (`EntityManager`, `@Entity`, `@Id`, `@OneToMany`, ...) để
ánh xạ object Java sang bảng database (Object-Relational Mapping — ORM) — **không tự nó chứa
logic thực thi nào**. **Hibernate** là implementation phổ biến nhất của đặc tả này (các
implementation khác gồm EclipseLink, OpenJPA) — chứa toàn bộ logic thật: sinh SQL, quản lý cache,
theo dõi thay đổi entity.

## Why?

Tách đặc tả (JPA) khỏi implementation (Hibernate) áp dụng đúng nguyên lý lập trình hướng
interface (Phase 1, Chapter 08) ở quy mô toàn ngành công nghiệp: code nghiệp vụ chỉ phụ thuộc
vào interface chuẩn (`EntityManager`, `@Entity`), không phụ thuộc trực tiếp vào bất kỳ chi tiết
implementation cụ thể nào của Hibernate — về lý thuyết, có thể đổi sang implementation JPA khác
(EclipseLink) mà không cần sửa code nghiệp vụ, chỉ cần đổi dependency và cấu hình. Trong thực tế,
Hibernate áp đảo thị phần tới mức nhiều lập trình viên dùng lẫn lộn hai thuật ngữ, nhưng ranh
giới kỹ thuật vẫn tồn tại rõ ràng, như đã chứng minh bằng thực nghiệm.

## How?

```java
@Entity // annotation CUA JPA (jakarta.persistence)
public class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
}

@PersistenceContext
EntityManager em; // interface CUA JPA, KHONG phai class Hibernate cu the
```

## Visualization

```
Code nghiep vu
     │  chi biet toi
     ▼
jakarta.persistence.EntityManager  <- INTERFACE (dac ta JPA, khong co logic)
     │  luc runtime, duoc "lap day" boi
     ▼
org.hibernate.internal.SessionImpl <- IMPLEMENTATION THAT (Hibernate)
     │
     ▼
SQL that gui xuong database (INSERT/SELECT/UPDATE/DELETE)
```

## Example

```java
@PersistenceContext EntityManager em;

Customer c = new Customer("Nguyen Van A");
customerRepo.save(c);
System.out.println("Sau khi save(): c.getId() = " + c.getId());

Customer plain = new Customer("Chi la Java object");
System.out.println("plain.getId() = " + plain.getId());

System.out.println("Instance THAT SU luc runtime: " + em.getClass().getName());
System.out.println("em.getDelegate(): " + em.getDelegate().getClass().getName());
```

Kết quả thật (Spring Boot 3.3.4, Hibernate ORM 6.5.3.Final, H2):

```
Hibernate: create table customer (id bigint generated by default as identity, name varchar(255), primary key (id))
Hibernate: insert into customer (name,id) values (?,default)
Sau khi save(): c.getId() = 1 (JPA da tu GAN ID)
plain.getId() = null (van la null, JPA CHUA he biet ve no)
Instance THAT SU luc runtime: jdk.proxy2.$Proxy135
em.getDelegate(): org.hibernate.internal.SessionImpl
```

Bảng `customer` được **tự động tạo** từ class `Customer` (annotation `@Entity`) — không có một
dòng SQL DDL nào được viết tay. `save()` sinh ra câu lệnh `INSERT` thật, và `id` được tự động
gán sau khi lưu — nhưng một object `Customer` tạo bằng `new` thuần tuý (`plain`), chưa qua
`save()`, vẫn có `id = null`: JPA/Hibernate hoàn toàn "không biết" về nó cho tới khi nó được
truyền qua `EntityManager`. `em.getDelegate()` xác nhận trực tiếp: đằng sau interface
`EntityManager`, implementation thật là `SessionImpl` của Hibernate.

## Deep Dive

**Vì sao `em.getClass().getName()` không trực tiếp trả về `SessionImpl`, mà là một proxy JDK
(`jdk.proxy2.$Proxy135`)?** Spring quản lý `EntityManager` theo mô hình **transaction-scoped
proxy** — vì một `EntityManager`/`Session` thật của Hibernate gắn liền với một transaction cụ
thể (không nên tái sử dụng xuyên suốt vòng đời bean singleton, Phase 5 Chapter 04), Spring tiêm
vào field `@PersistenceContext` một **proxy** (dùng kỹ thuật tương tự AOP proxy, Phase 5 Chapter
12) — proxy này tự động "chuyển tiếp" (delegate) tới đúng `EntityManager`/`Session` thật gắn với
transaction hiện tại mỗi khi một phương thức được gọi. Gọi `em.getDelegate()` mới thực sự "vượt
qua" lớp proxy này để lấy về implementation Hibernate gốc.

## Engineering Insight

**Trong thực tế, có bao nhiêu dự án thực sự "đổi từ Hibernate sang implementation JPA khác"?**
Cực kỳ hiếm. Dù về mặt lý thuyết JPA cho phép điều này, trong thực tế các dự án thường **dựa
vào các tính năng đặc thù của Hibernate** (nằm ngoài đặc tả JPA chuẩn — ví dụ
`@DynamicUpdate`, `@BatchSize`, các chiến lược cache nâng cao) để tối ưu hiệu năng, khiến việc
đổi implementation trở nên không thực tế dù interface tầng trên vẫn tương thích. Giá trị thực sự
của việc tách JPA/Hibernate không nằm ở khả năng "đổi implementation dễ dàng" (hiếm khi xảy ra),
mà nằm ở việc **chuẩn hoá kiến thức và API** trên toàn ngành — một lập trình viên hiểu JPA có thể
làm việc với bất kỳ dự án Java ORM nào (Hibernate, EclipseLink) mà không cần học lại từ đầu.

## Historical Note

JPA ra đời năm 2006 (JSR 220, EJB 3.0) — một phần trong nỗ lực đơn giản hoá Java EE sau khi EJB
2.x (nổi tiếng cồng kềnh) bị chỉ trích rộng rãi. Hibernate (Gavin King, 2001) ra đời **trước**
JPA 5 năm — thực tế Hibernate có ảnh hưởng ngược lại, định hình nhiều khái niệm cốt lõi của
chính đặc tả JPA sau này (persistence context, entity lifecycle — Chapter 07). Từ Jakarta EE 9
(2020), gói namespace đổi từ `javax.persistence` sang `jakarta.persistence` (thấy trực tiếp
trong import của Example) do vấn đề bản quyền tên miền `javax` khi Java EE chuyển giao cho
Eclipse Foundation.

## Myth vs Reality

- **Myth:** "JPA và Hibernate là hai tên gọi khác nhau cho cùng một thứ."
  **Reality:** Đã chứng minh bằng thực nghiệm — JPA là interface/đặc tả, Hibernate là
  implementation cụ thể chạy thật đằng sau nó.

- **Myth:** "Dùng JPA nghĩa là code hoàn toàn độc lập khỏi Hibernate, có thể đổi implementation
  bất cứ lúc nào không tốn chi phí."
  **Reality:** Xem Engineering Insight — trong thực tế, việc dùng các tính năng đặc thù của
  Hibernate khiến điều này khó khả thi hơn nhiều so với lý thuyết.

## Common Mistakes

- **Dùng lẫn lộn thuật ngữ "JPA" và "Hibernate" khi trả lời phỏng vấn mà không phân biệt được
  ranh giới kỹ thuật** — thể hiện thiếu hiểu biết nền tảng về kiến trúc ORM.
- **Import nhầm `javax.persistence` thay vì `jakarta.persistence`** khi làm việc với Spring Boot
  3+ (đã chuyển hẳn sang namespace mới, Historical Note).
- **Không biết `show-sql`/log SQL để debug** — bỏ lỡ công cụ chẩn đoán cơ bản nhất khi làm việc
  với JPA/Hibernate (dùng xuyên suốt các chapter tiếp theo của Phase 6).

## Best Practices

- Luôn phân biệt rõ trong tư duy: JPA là hợp đồng, Hibernate là người thực hiện hợp đồng đó.
- Bật `spring.jpa.show-sql=true` (chỉ trong môi trường phát triển) để quan sát trực tiếp SQL
  thật Hibernate sinh ra — công cụ học tập và debug quan trọng nhất của Phase 6.
- Với dự án Spring Boot 3+, luôn dùng `jakarta.persistence.*`, không phải `javax.persistence.*`.

## Debug Checklist

- [ ] Không chắc một hành vi là do đặc tả JPA hay do đặc thù Hibernate? → tra cứu tài liệu JPA
      spec chính thức trước, nếu không tìm thấy, khả năng cao đó là tính năng riêng của
      Hibernate.
- [ ] Cần xem SQL thật được sinh ra cho một thao tác cụ thể? → bật `spring.jpa.show-sql=true`.
- [ ] Lỗi import `javax.persistence` không tìm thấy trên Spring Boot 3+? → đổi sang
      `jakarta.persistence`.

## Summary

JPA là đặc tả (Jakarta EE) định nghĩa interface/annotation chuẩn cho ORM trên Java, không chứa
logic thực thi. Hibernate là implementation phổ biến nhất, chứa toàn bộ logic thật (sinh SQL,
quản lý cache, theo dõi thay đổi). Đã chứng minh bằng thực nghiệm: `EntityManager` (interface
JPA) thực chất được backing bởi `org.hibernate.internal.SessionImpl` lúc runtime (qua
`em.getDelegate()`), và Hibernate tự động sinh DDL (`CREATE TABLE`) cùng SQL (`INSERT`/`SELECT`)
từ các annotation `@Entity` trên class Java thuần. Trong thực tế, việc đổi implementation JPA
hiếm khi xảy ra do phụ thuộc vào các tính năng đặc thù của Hibernate.

## Interview Questions

**Junior**

- JPA là gì? JPA khác Hibernate như thế nào?

**Mid**

- Vì sao lại tách JPA (đặc tả) khỏi Hibernate (implementation)? Lợi ích thực tế của việc này là
  gì?

## Exercises

- [ ] Chạy lại `JpaBasicsTest` ở trên, xác nhận `em.getDelegate()` trả về đúng
      `org.hibernate.internal.SessionImpl`.
- [ ] Bật `spring.jpa.show-sql=true`, quan sát SQL DDL được sinh ra khi khởi động ứng dụng cho
      một entity mới bạn tự định nghĩa.
- [ ] Tìm hiểu EclipseLink (implementation JPA khác), so sánh cấu hình dependency với Hibernate.

## Cheat Sheet

| | JPA | Hibernate |
| --- | --- | --- |
| Loại | Đặc tả (specification), Jakarta EE | Implementation cụ thể |
| Chứa logic thật? | Không | Có |
| Ví dụ | `EntityManager`, `@Entity`, `@Id` | `SessionImpl`, `@DynamicUpdate`, `@BatchSize` |
| Namespace (Spring Boot 3+) | `jakarta.persistence.*` | `org.hibernate.*` |

## References

- Jakarta Persistence Specification — https://jakarta.ee/specifications/persistence/
- Hibernate ORM Documentation — https://hibernate.org/orm/documentation/
