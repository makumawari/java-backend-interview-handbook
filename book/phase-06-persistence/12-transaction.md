---
tags:
  - JPA
  - Transaction
  - Hibernate
---

# Transaction (JPA Transaction Boundary) và Flush

> Phase: Phase 6 — Persistence
> Chapter slug: `transaction`

## Metadata

```yaml
Chapter: Transaction (JPA Transaction Boundary)
Phase: Phase 6 — Persistence
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 60%
Prerequisites:
  - Chapter 06 — Dirty Checking
  - Phase 5, Chapter 13 — Transaction (Spring Transaction Management)
Used Later:
  - Chapter 13 — Optimistic Lock
  - Chapter 14 — Pessimistic Lock
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 06](06-dirty-checking.md) — sửa một entity Managed, `UPDATE` chỉ xuất hiện
> lúc `commit()`. Nhưng nếu, **trong cùng transaction**, sau khi sửa entity đó, code chạy một
> câu **JPQL query** khác liên quan tới đúng dữ liệu vừa sửa — Hibernate có "biết" trả về đúng
> dữ liệu mới, hay vẫn trả về dữ liệu cũ (vì `UPDATE` "lẽ ra" chỉ chạy lúc commit)?

```java
Customer managed = em.find(Customer.class, id);
managed.setName("Ten Moi Flush"); // CHUA flush() tuong minh
// Ngay sau do, chay JPQL query theo TEN CU va TEN MOI...
```

Kết quả thật:

```
Da sua managed.setName(), CHUA flush() tuong minh, sap chay JPQL...
Hibernate: update customer set name=? where id=?
Hibernate: select count(c1_0.id) from customer c1_0 where c1_0.name='Ten Cu Flush'
Hibernate: select count(c1_0.id) from customer c1_0 where c1_0.name='Ten Moi Flush'
Dem theo TEN CU sau khi query: 0
Dem theo TEN MOI: 1
```

`UPDATE` chạy **trước** cả hai câu JPQL — dù chưa `commit()`, dù chưa gọi `flush()` tường minh.
Kết quả JPQL phản ánh **đúng** dữ liệu mới.

## Interview Question (Central)

> Flush trong JPA là gì? Nó khác commit như thế nào? Khi nào Hibernate tự động flush?

## Objectives

- [ ] Phân biệt chính xác **flush** (đồng bộ Persistence Context xuống database qua SQL) và
      **commit** (kết thúc transaction database, làm thay đổi vĩnh viễn/khả kiến với giao dịch
      khác)
- [ ] Tự tay chứng minh bằng thực nghiệm: Hibernate **tự động flush** trước khi chạy một JPQL
      query có thể bị ảnh hưởng bởi thay đổi chưa lưu
- [ ] Hiểu ranh giới transaction JPA nằm ở đâu so với ranh giới `@Transactional` của Spring
      (Phase 5, Chapter 13)

## Prerequisites

- Chapter 06 — hiểu Dirty Checking, cơ chế phát hiện thay đổi mà flush thực thi.
- Phase 5, Chapter 13 — hiểu `@Transactional` của Spring, lớp quản lý transaction bao bọc bên
  ngoài JPA transaction.

## Used Later

- **Chapter 13 (Optimistic Lock)**, **Chapter 14 (Pessimistic Lock)** — cả hai cơ chế khoá đều
  liên quan trực tiếp tới thời điểm flush/commit thực sự ghi dữ liệu xuống database.

## Problem

Nếu Hibernate chỉ đồng bộ Persistence Context xuống database **duy nhất một lần** lúc commit,
mọi truy vấn (query) chạy **trong** cùng transaction, **sau** khi đã sửa đổi entity nhưng
**trước** khi commit, sẽ trả về dữ liệu **cũ** — dẫn tới sự không nhất quán ngay trong chính
logic của một transaction duy nhất (code tưởng đã sửa dữ liệu, nhưng query ngay sau đó lại
"không thấy" thay đổi).

## Concept

**Flush** là hành động Hibernate đồng bộ hoá trạng thái Persistence Context (mọi thay đổi Dirty
Checking phát hiện được, Chapter 06) xuống database bằng cách gửi các câu lệnh SQL
(`INSERT`/`UPDATE`/`DELETE`) — **nhưng chưa `COMMIT` transaction database**. **Commit** là bước
riêng biệt, xảy ra **sau** flush, thực sự hoàn tất transaction (làm thay đổi vĩnh viễn, khả kiến
với các transaction/kết nối khác). Hibernate tự động flush trong một số tình huống nhất định —
phổ biến nhất là **ngay trước khi thực thi một query** có khả năng bị ảnh hưởng bởi thay đổi
chưa đồng bộ (default flush mode: `AUTO`).

## Why?

Tách flush khỏi commit giải quyết chính xác vấn đề đã nêu ở Problem: đồng bộ SQL xuống database
(flush) đủ để các **query tiếp theo trong cùng transaction** nhìn thấy dữ liệu nhất quán, nhưng
**không** cần hoàn tất transaction ngay (commit) — cho phép transaction tiếp tục thực hiện thêm
thao tác, và vẫn có khả năng rollback toàn bộ nếu có lỗi xảy ra sau đó. Flush mode `AUTO` (mặc
định) tự động kích hoạt trước mỗi query — Hibernate đủ thông minh để nhận biết "query này có thể
bị ảnh hưởng bởi thay đổi Dirty Checking đang chờ" và tự flush trước, giải quyết đúng vấn đề nhất
quán trong cùng transaction mà không cần lập trình viên tự gọi `flush()` tường minh ở mọi nơi.

## How?

```java
Customer managed = em.find(Customer.class, id);
managed.setName("Ten Moi"); // CHI o Persistence Context, CHUA co SQL nao chay

// Hibernate TU DONG flush() TRUOC khi chay query nay (flush mode AUTO)
Long count = em.createQuery("select count(c) from Customer c where c.name = 'Ten Moi'", Long.class)
    .getSingleResult(); // se dem DUNG, vi UPDATE da flush truoc do

txManager.commit(tx); // COMMIT that su, hoan tat transaction database
```

## Visualization

```
Timeline trong MOT transaction:

  find() ──► setName() (chi sua trong bo nho, Persistence Context)
                  │
                  ▼  chuan bi chay JPQL query lien quan
             AUTO FLUSH: Hibernate tu dong chay UPDATE THAT
                  │
                  ▼
             JPQL query chay, THAY duoc du lieu MOI (vi UPDATE da flush)
                  │
                  ▼
             commit() ──► transaction database THAT SU hoan tat

Flush KHONG PHAI la commit:
  - Flush: gui SQL xuong DB, nhung transaction VAN CON MO, VAN CO THE rollback
  - Commit: hoan tat transaction, thay doi tro nen vinh vien/kha kien voi giao dich khac
```

## Example

```java
Customer managed = em.find(Customer.class, id);
managed.setName("Ten Moi Flush");
System.out.println("Da sua managed.setName(), CHUA flush() tuong minh, sap chay JPQL...");

Long countOld = em.createQuery("select count(c) from Customer c where c.name = 'Ten Cu Flush'", Long.class)
    .getSingleResult();
Long countNew = em.createQuery("select count(c) from Customer c where c.name = 'Ten Moi Flush'", Long.class)
    .getSingleResult();
System.out.println("Dem theo TEN CU: " + countOld);
System.out.println("Dem theo TEN MOI: " + countNew);
```

Kết quả thật (Hibernate ORM 6.5.3.Final):

```
Da sua managed.setName(), CHUA flush() tuong minh, sap chay JPQL...
Hibernate: update customer set name=? where id=?
Hibernate: select count(c1_0.id) from customer c1_0 where c1_0.name='Ten Cu Flush'
Hibernate: select count(c1_0.id) from customer c1_0 where c1_0.name='Ten Moi Flush'
Dem theo TEN CU sau khi query: 0
Dem theo TEN MOI: 1
```

Câu `UPDATE` xuất hiện **ngay trước** cả hai câu JPQL — Hibernate đã tự động flush thay đổi
xuống database **trước khi** chạy query, dù chưa hề `commit()`. Kết quả JPQL hoàn toàn nhất
quán với dữ liệu mới (`0` bản ghi tên cũ, `1` bản ghi tên mới) — xác nhận trực tiếp auto-flush
đã hoạt động đúng, không phải một sự trùng hợp.

## Deep Dive

**Vì sao Hibernate không đơn giản flush TRƯỚC MỌI query, mà chỉ flush khi "cần thiết"?** Flush
mặc định (`FlushModeType.AUTO`) có logic thông minh: nó chỉ tự động flush khi Hibernate **suy
luận** được rằng query sắp chạy **có khả năng** bị ảnh hưởng bởi thay đổi đang chờ trong
Persistence Context (ví dụ query JPQL trên cùng Entity vừa bị sửa). Với native SQL query
(`createNativeQuery`) hoặc query không liên quan tới entity đã thay đổi, Hibernate **không nhất
thiết** flush trước — vì bản chất không biết chắc native query đó có phụ thuộc dữ liệu chưa flush
hay không (Hibernate không phân tích được native SQL để suy luận phụ thuộc). Đây là lý do khi
làm việc với native SQL xen giữa các thao tác JPA, cần **tự flush tường minh** (`em.flush()`)
nếu biết chắc native query đó cần thấy dữ liệu vừa thay đổi — không nên dựa hoàn toàn vào
auto-flush cho mọi trường hợp.

## Engineering Insight

**Ranh giới "transaction" trong `@Transactional` (Phase 5, Chapter 13) và ranh giới flush/commit
của JPA quan hệ với nhau như thế nào?** `@Transactional` của Spring (dùng AOP proxy, Phase 5
Chapter 12) là lớp **quản lý** bên ngoài — nó quyết định **khi nào** transaction database bắt
đầu (`begin`) và **khi nào** thực sự `commit`/`rollback` (dựa trên method có return bình thường
hay ném exception, và loại exception đó — Phase 5 Chapter 13). Bên **trong** ranh giới đó, JPA/
Hibernate tự quản lý các lần **flush** (có thể xảy ra nhiều lần, tại nhiều thời điểm khác nhau
trong cùng một transaction Spring) — flush **không** kết thúc transaction, chỉ đồng bộ SQL.
`commit` **thật sự** (kết thúc transaction database) chỉ xảy ra **một lần**, đúng tại thời điểm
method `@Transactional` (Phase 5, Chapter 13) hoàn tất — đây là lý do hiểu rõ JPA flush cần thiết
để hiểu chính xác **khi nào dữ liệu thực sự "khả kiến"** với các phần khác của hệ thống, một chi
tiết quan trọng khi debug các vấn đề liên quan tới đồng thời (concurrency, Chapter 13-14).

## Historical Note

Khái niệm flush (tách biệt khỏi commit) đã tồn tại từ những phiên bản Hibernate rất sớm — phản
ánh trực tiếp mô hình "Session" như một đơn vị công việc (Unit of Work pattern, một mẫu hình
kinh điển khác của Martin Fowler, cùng với Identity Map đã nhắc ở Chapter 05) — Session tích luỹ
mọi thay đổi, chỉ thực sự "nói chuyện" với database khi cần (flush), và transaction database
thực sự chỉ đóng vai trò "khung bao" (envelope) quyết định khi nào toàn bộ các thay đổi đó được
xác nhận vĩnh viễn hay huỷ bỏ.

## Myth vs Reality

- **Myth:** "Mọi thay đổi trên entity Managed chỉ thực sự chạy SQL đúng lúc `commit()`, không
  sớm hơn."
  **Reality:** Đã chứng minh bằng thực nghiệm — Hibernate tự động flush (chạy SQL thật) **trước**
  khi thực thi một query có khả năng bị ảnh hưởng, hoàn toàn có thể xảy ra nhiều lần trước khi
  commit.

- **Myth:** "Flush và commit là hai tên gọi khác nhau cho cùng một hành động."
  **Reality:** Flush chỉ đồng bộ SQL, transaction **vẫn có thể rollback** sau đó; commit mới
  thực sự hoàn tất, làm thay đổi vĩnh viễn.

## Common Mistakes

- **Giả định native SQL query luôn thấy được thay đổi JPA chưa flush** — không đảm bảo, cần
  `em.flush()` tường minh trước native query nếu cần dữ liệu mới nhất.
- **Nhầm lẫn flush với commit khi debug**, dẫn tới hiểu sai thời điểm dữ liệu thực sự "an toàn"
  (không thể rollback nữa).
- **Gọi `em.flush()` quá thường xuyên "cho chắc"** — mất đi lợi ích của Dirty Checking (Chapter
  06) gộp nhiều thay đổi thành ít câu SQL hơn khi thực sự cần.

## Debug Checklist

- [ ] Query trong cùng transaction không thấy thay đổi vừa sửa trên entity? → kiểm tra query đó
      có phải native SQL không (Hibernate không tự flush trước native query một cách đáng tin
      cậy) — cân nhắc `em.flush()` tường minh.
- [ ] Không chắc một thao tác đã thực sự ghi xuống database hay chỉ mới flush (chưa commit)? →
      nhớ rằng flush không đồng nghĩa transaction đã hoàn tất — vẫn có thể rollback sau flush.
- [ ] Cần debug chính xác thời điểm flush xảy ra? → bật `show-sql`, quan sát vị trí câu lệnh SQL
      xuất hiện tương đối so với code Java.

## Summary

Flush (đồng bộ Persistence Context xuống database qua SQL thật) và commit (hoàn tất transaction,
làm thay đổi vĩnh viễn) là hai khái niệm tách biệt. Đã chứng minh bằng thực nghiệm: sửa một
entity Managed (dirty checking), chạy JPQL query liên quan **ngay sau đó** (chưa `commit()`,
chưa `flush()` tường minh) — Hibernate **tự động flush** trước khi chạy query, kết quả JPQL phản
ánh đúng dữ liệu mới. Flush mode `AUTO` (mặc định) chỉ tự flush khi suy luận được query có thể
bị ảnh hưởng — với native SQL, cần `em.flush()` tường minh nếu cần đảm bảo thấy dữ liệu mới
nhất. Ranh giới `@Transactional` (Phase 5, Chapter 13) quyết định khi nào commit thực sự xảy ra;
bên trong đó, JPA có thể flush nhiều lần độc lập.

## Interview Questions

**Mid**

- Flush trong JPA là gì? Nó khác commit như thế nào?
- Khi nào Hibernate tự động flush?

**Senior**

- Giải thích quan hệ giữa ranh giới `@Transactional` của Spring và các lần flush của JPA bên
  trong một transaction.
- Vì sao native SQL query không đáng tin cậy để tự động thấy được thay đổi JPA chưa flush?

## Exercises

- [ ] Chạy lại `FlushTimingTest` ở trên, xác nhận `UPDATE` xuất hiện trước cả hai câu JPQL, và
      kết quả đếm đúng theo dữ liệu mới.
- [ ] Thử thay JPQL bằng native SQL (`em.createNativeQuery`) cho cùng kịch bản, quan sát liệu
      auto-flush có còn xảy ra hay không.
- [ ] Đặt `FlushModeType.COMMIT` tường minh (`em.setFlushMode(...)`, tắt auto-flush trước
      query), chạy lại ví dụ, quan sát JPQL trả về dữ liệu cũ dù đã sửa entity.

## Cheat Sheet

| | Flush | Commit |
| --- | --- | --- |
| Hành động | Gửi SQL (`INSERT`/`UPDATE`/`DELETE`) xuống database | Hoàn tất transaction database |
| Có thể rollback sau đó? | Có | Không (đã hoàn tất) |
| Tần suất trong 1 transaction | Nhiều lần (auto hoặc tường minh) | Đúng 1 lần |
| Trigger tự động (`AUTO` mode) | Trước query JPQL liên quan | Khi method `@Transactional` hoàn tất |

## References

- Jakarta Persistence Specification — Flush.
- Martin Fowler — "Patterns of Enterprise Application Architecture", Unit of Work pattern (2002).
- Hibernate ORM Documentation — Flushing.
