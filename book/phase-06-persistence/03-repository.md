---
tags:
  - JPA
  - SpringDataJPA
  - Repository
---

# Spring Data JPA Repository

> Phase: Phase 6 — Persistence
> Chapter slug: `repository`

## Metadata

```yaml
Chapter: Repository
Phase: Phase 6 — Persistence
Difficulty: ★★★
Importance: ★★★★★
Interview Frequency: 85%
Prerequisites:
  - Chapter 02 — Entity
  - Phase 1, Chapter 19 — Reflection
Used Later:
  - Chapter 09 — Fetch Join
  - Chapter 10 — N+1
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: Phase 1, Chapter 19 (Reflection) — annotation tự nó không thực thi gì cả. Vậy làm
> sao chỉ khai báo một **interface** — không viết bất kỳ implementation nào — lại có được đầy
> đủ các phương thức `save()`, `findById()`, và thậm chí một phương thức **chưa từng viết logic**
> như `findByName(String name)` vẫn hoạt động đúng?

```java
public interface CustomerRepository extends JpaRepository<Customer, Long> {
    Optional<Customer> findByName(String name); // KHONG CO THAN METHOD!
}
```

Gọi thử và xem SQL thật được sinh ra:

```
=== Derived query method: findByName (Spring TU SINH JPQL tu TEN method) ===
Hibernate: select c1_0.id,c1_0.name from customer c1_0 where c1_0.name=?
Ket qua: Nguyen Van A
```

Chỉ từ **tên phương thức** (`findByName`), Spring Data JPA tự suy luận ra chính xác câu lệnh
`WHERE c1_0.name = ?` — không một dòng SQL/JPQL nào được viết tay.

## Interview Question (Central)

> What is the use of the @Query annotation in Spring Data JPA?

## Objectives

- [ ] Dùng thành thạo derived query method (Spring tự sinh query từ tên phương thức)
- [ ] Tự tay chứng minh bằng thực nghiệm SQL thật sinh ra từ `findByName`,
      `findByNameContaining`, và `@Query` tuỳ chỉnh
- [ ] Hiểu vì sao `@Query` cần thiết khi derived query method không đủ biểu đạt (join fetch, câu
      lệnh phức tạp)

## Prerequisites

- Chapter 02 — hiểu Entity, đối tượng mà Repository thao tác lên.
- Phase 1, Chapter 19 — hiểu Reflection, cơ chế nền tảng Spring dùng để tạo implementation cho
  interface Repository lúc runtime.

## Used Later

- **Chapter 09 (Fetch Join)**, **Chapter 10 (N+1)** — cả hai đều dùng `@Query` để kiểm soát chính
  xác câu lệnh JPQL, giải quyết vấn đề hiệu năng khi truy vấn dữ liệu liên kết.

## Problem

Viết tay code CRUD (Create, Read, Update, Delete) cơ bản cho mỗi Entity — `save`, `findById`,
`deleteById`, `findAll` — là logic **lặp lại gần như giống hệt nhau** giữa các Entity khác nhau,
chỉ khác kiểu dữ liệu. Ngoài ra, viết các câu truy vấn đơn giản (tìm theo một field, tìm theo
khoảng giá trị) bằng JPQL/SQL thủ công cho mỗi trường hợp cũng là một dạng boilerplate không cần
thiết khi bản thân điều kiện tìm kiếm có thể suy luận trực tiếp từ tên phương thức.

## Concept

**Spring Data JPA Repository** cho phép khai báo một **interface** kế thừa `JpaRepository<Entity,
IdType>` — Spring **tự động tạo implementation** lúc runtime (dùng JDK dynamic proxy, tương tự
cơ chế đã học ở Phase 5, Chapter 12), cung cấp sẵn các phương thức CRUD cơ bản. **Derived query
method** (tên phương thức theo quy ước `findBy...`, `countBy...`, `deleteBy...`) được Spring
**phân tích tên phương thức** để tự sinh JPQL tương ứng — không cần viết thân method.
**`@Query`** cho phép khai báo tường minh JPQL/SQL khi câu truy vấn phức tạp hơn khả năng suy
luận từ tên phương thức (join, group by, subquery).

## Why?

Cơ chế "chỉ khai báo interface, tự động sinh implementation" tận dụng đúng khả năng Reflection
và dynamic proxy đã học trước đó: Spring quét interface, dùng Reflection đọc **tên** và **chữ ký**
của từng phương thức, phân tích cú pháp tên đó (`findBy` + `Name` → `WHERE name = ?`,
`findBy` + `NameContaining` → `WHERE name LIKE ?`) để xây dựng JPQL tương ứng lúc khởi động —
loại bỏ hoàn toàn boilerplate CRUD và các câu truy vấn đơn giản. Với truy vấn phức tạp vượt quá
khả năng biểu đạt của tên phương thức (join nhiều bảng, tổng hợp dữ liệu), `@Query` cho phép
"thoát" khỏi cơ chế suy luận tự động, viết JPQL/SQL tường minh khi thực sự cần kiểm soát chi
tiết.

## How?

```java
public interface CustomerRepository extends JpaRepository<Customer, Long> {
    // Derived query method - Spring TU SINH JPQL tu ten
    Optional<Customer> findByName(String name);
    List<Customer> findByNameContaining(String keyword);

    // @Query - JPQL tuong minh, khi ten phuong thuc khong du bieu dat
    @Query("select c from Customer c join fetch c.orders where c.id = :id")
    Optional<Customer> findWithOrdersById(Long id);
}
```

## Visualization

```
interface CustomerRepository extends JpaRepository<Customer, Long> {
    Optional<Customer> findByName(String name);
}
       │
       ▼  Spring quet bang REFLECTION, phan tich TEN phuong thuc

  "findBy" + "Name"
       │
       ▼  sinh JPQL tuong ung

  select c from Customer c where c.name = :name
       │
       ▼  tao PROXY implement interface nay luc runtime

  CustomerRepository customerRepo = (proxy tu dong tao, KHONG AI viet class nay thu cong)
```

## Example

```java
public interface CustomerRepository extends JpaRepository<Customer, Long> {
    Optional<Customer> findByName(String name);
    List<Customer> findByNameContaining(String keyword);

    @Query("select c from Customer c join fetch c.orders where c.id = :id")
    Optional<Customer> findWithOrdersById(Long id);
}
```

Kết quả thật (Hibernate ORM 6.5.3.Final, `spring.jpa.show-sql=true`):

```
=== Derived query method: findByName ===
Hibernate: select c1_0.id,c1_0.name from customer c1_0 where c1_0.name=?
Ket qua: Nguyen Van A

=== Derived query method: findByNameContaining ===
Hibernate: select c1_0.id,c1_0.name from customer c1_0 where c1_0.name like ? escape '\'
Ket qua: [Nguyen Van A, Nguyen Van C]

=== @Query tuy chinh (JPQL) voi join fetch ===
Hibernate: select c1_0.id,c1_0.name,o1_0.customer_id,o1_0.id,o1_0.status,o1_0.version
           from customer c1_0 join orders o1_0 on c1_0.id=o1_0.customer_id where c1_0.id=?
Ket qua: Nguyen Van A
```

`findByName` sinh đúng `WHERE c1_0.name=?`. `findByNameContaining` sinh đúng `LIKE ? escape '\'`
(tự thêm ký tự escape để xử lý an toàn các ký tự đặc biệt trong `LIKE`). `@Query` tuỳ chỉnh sinh
chính xác câu `JOIN` đã viết trong JPQL — không thể đạt được chỉ bằng đặt tên phương thức, vì
`join fetch` (nạp sẵn dữ liệu liên kết, Chapter 09) là một chỉ thị không có tương đương tự nhiên
trong quy ước đặt tên `findBy...`.

## Deep Dive

**Spring Data JPA thực sự "sinh" implementation cho interface như thế nào lúc runtime?** Không
có bất kỳ file `.java`/`.class` nào được sinh ra trước khi biên dịch — Spring dùng **JDK dynamic
proxy** (Phase 5, Chapter 12 đã nhắc AOP dùng CGLIB cho class cụ thể; ở đây vì `CustomerRepository`
là **interface**, JDK dynamic proxy thuần tuý là đủ, không cần CGLIB) tạo một proxy implement
`CustomerRepository` lúc khởi động ứng dụng. Mọi lời gọi phương thức trên proxy đó được chuyển
tới một `InvocationHandler` trung tâm (cụ thể là `SimpleJpaRepository` cho các phương thức CRUD
kế thừa, hoặc một `QueryExecutor` được xây dựng sẵn cho mỗi derived query method dựa trên phân
tích tên lúc khởi động) — không có code Java "thật" nào implement `findByName` theo nghĩa
truyền thống, toàn bộ hành vi được tổng hợp động từ metadata (tên phương thức + kiểu Entity).

## Engineering Insight

**Khi nào derived query method trở nên "quá tải" và nên chuyển sang `@Query`?** Derived query
method rất mạnh cho điều kiện đơn giản (`findByNameAndStatus`, `findByPriceGreaterThan`), nhưng
tên phương thức trở nên **cực kỳ dài và khó đọc** khi điều kiện phức tạp hơn (nhiều điều kiện
kết hợp `AND`/`OR`, sắp xếp, phân trang tuỳ chỉnh) — đây chính là lúc code trở nên khó bảo trì
hơn chính JPQL tường minh mà nó đang cố tránh viết. Ngoài ra, derived query method **không thể**
biểu đạt các chỉ thị JPQL đặc biệt như `JOIN FETCH` (Chapter 09, giải quyết N+1 — Chapter 10),
`GROUP BY`, hay subquery — với những trường hợp này, `@Query` không chỉ là lựa chọn thay thế, mà
là **cách duy nhất** để đạt được hành vi mong muốn. Kinh nghiệm thực tế: dùng derived query
method cho điều kiện đơn giản, rõ ràng; chuyển sang `@Query` ngay khi tên phương thức bắt đầu
vượt quá 3-4 điều kiện hoặc cần join/fetch tường minh.

## Historical Note

Spring Data JPA ra đời năm 2011, xây dựng trên nền tảng Spring Data Commons (dự án chung cho
nhiều loại datastore — JPA, MongoDB, Redis, ... đều dùng chung quy ước derived query method).
Cơ chế phân tích tên phương thức (`findBy`, `countBy`, `existsBy`, `deleteBy` kết hợp
`And`/`Or`/`Containing`/`GreaterThan`/`OrderBy`) đã trở thành một trong những "dấu ấn" API dễ
nhận biết nhất của hệ sinh thái Spring, được nhiều framework khác học hỏi theo.

## Myth vs Reality

- **Myth:** "Derived query method chỉ là 'ma thuật', không có gì đảm bảo nó sinh đúng SQL mong
  muốn."
  **Reality:** Đã chứng minh bằng thực nghiệm — SQL sinh ra hoàn toàn có thể quan sát trực tiếp
  qua `show-sql`, và tuân theo quy tắc phân tích tên phương thức nhất quán, có thể dự đoán
  được.

- **Myth:** "`@Query` chỉ là một cách viết khác của derived query method, không có khác biệt về
  khả năng."
  **Reality:** Xem Engineering Insight — `@Query` là cách **duy nhất** để dùng `JOIN FETCH`
  hoặc các chỉ thị JPQL phức tạp mà derived query method không biểu đạt được.

## Common Mistakes

- **Viết tên phương thức quá dài, quá nhiều điều kiện kết hợp** — khó đọc hơn cả JPQL tường
  minh; nên chuyển sang `@Query` khi vượt quá 3-4 điều kiện.
- **Không biết derived query method không hỗ trợ `JOIN FETCH`** — dẫn tới cố gắng đặt tên
  phương thức phức tạp một cách vô ích thay vì dùng `@Query` ngay từ đầu.
- **Quên `@Param` khi dùng tham số đặt tên (`:id`) trong `@Query`** — gây lỗi runtime vì
  Hibernate không biết ánh xạ tham số method nào vào placeholder nào (thường cần bật
  `-parameters` khi biên dịch, hoặc dùng `@Param` tường minh).

## Best Practices

- Dùng derived query method cho điều kiện đơn giản, rõ ràng (1-3 điều kiện).
- Chuyển sang `@Query` khi cần `JOIN FETCH`, `GROUP BY`, subquery, hoặc khi tên phương thức trở
  nên khó đọc.
- Luôn kiểm tra SQL thật sinh ra qua `show-sql` khi không chắc derived query method có suy luận
  đúng điều kiện mong muốn không.

## Debug Checklist

- [ ] Không chắc derived query method có sinh đúng SQL mong muốn không? → bật
      `spring.jpa.show-sql=true`, quan sát SQL thật sinh ra.
- [ ] Cần join dữ liệu liên kết trong một query nhưng derived query method không hỗ trợ? → dùng
      `@Query` với `JOIN FETCH` (Chapter 09).
- [ ] Lỗi runtime liên quan tới tham số trong `@Query`? → kiểm tra `@Param` đã khai báo đúng
      tên khớp với `:paramName` trong JPQL chưa.

## Summary

Spring Data JPA Repository chỉ cần khai báo interface kế thừa `JpaRepository` — Spring dùng JDK
dynamic proxy (Reflection) tạo implementation lúc runtime. Derived query method suy luận JPQL
trực tiếp từ tên phương thức (`findByName` → `WHERE name = ?`) — đã chứng minh bằng thực nghiệm,
quan sát trực tiếp SQL thật sinh ra qua `show-sql`. `@Query` cho phép khai báo JPQL tường minh
khi điều kiện phức tạp hơn khả năng biểu đạt của tên phương thức — đặc biệt là `JOIN FETCH`
(Chapter 09), một chỉ thị derived query method hoàn toàn không hỗ trợ được.

## Interview Questions

- What is the use of the @Query annotation in Spring Data JPA?

**Mid**

- Derived query method hoạt động như thế nào? Spring "hiểu" tên phương thức bằng cách nào?
- Khi nào nên dùng `@Query` thay vì derived query method?

## Exercises

- [ ] Chạy lại `RepositoryDemoTest` ở trên, xác nhận SQL sinh ra đúng như mô tả cho cả 3
      trường hợp.
- [ ] Viết một derived query method `findByNameAndStatus` kết hợp điều kiện trên hai Entity liên
      kết (`Customer`/`Order`), quan sát JPQL sinh ra.
- [ ] Viết một `@Query` dùng native SQL (`nativeQuery = true`) thay vì JPQL, so sánh sự khác
      biệt về khả năng portable giữa các loại database.

## Cheat Sheet

| Tiền tố derived query | Ý nghĩa |
| --- | --- |
| `findBy` | `SELECT ... WHERE ...` |
| `countBy` | `SELECT COUNT(...) WHERE ...` |
| `existsBy` | Kiểm tra tồn tại |
| `deleteBy` | `DELETE ... WHERE ...` |
| `...Containing` | `LIKE %...%` |
| `...GreaterThan`/`...LessThan` | So sánh lớn hơn/nhỏ hơn |
| `...OrderBy...Asc/Desc` | Sắp xếp kết quả |

**Khi nào dùng `@Query`:** cần `JOIN FETCH`, `GROUP BY`, subquery, hoặc tên phương thức quá dài/
khó đọc.

## References

- Spring Data JPA Documentation — Query Methods: https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html
- Spring Data JPA Documentation — @Query.
