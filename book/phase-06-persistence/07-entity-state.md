---
tags:
  - JPA
  - EntityState
  - Hibernate
---

# Entity State (Transient/Managed/Detached/Removed)

> Phase: Phase 6 — Persistence
> Chapter slug: `entity-state`

## Metadata

```yaml
Chapter: Entity State
Phase: Phase 6 — Persistence
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 65%
Prerequisites:
  - Chapter 05 — Persistence Context
  - Chapter 06 — Dirty Checking
Used Later:
  - Chapter 08 — Lazy Loading
  - Chapter 11 — Cascade
Estimated Reading: 18 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: [Chapter 06](06-dirty-checking.md) — Dirty Checking chỉ hoạt động với entity
> **Managed**. Nhưng một entity Java, xuyên suốt vòng đời của nó, thực ra đi qua **4 trạng
> thái** khác nhau — mỗi trạng thái có hành vi hoàn toàn khác khi bạn sửa field của nó.

Theo dõi cùng một object `Customer` qua 4 giai đoạn, kiểm tra `em.contains()` (có đang được
Persistence Context theo dõi không) ở mỗi bước:

```
=== 1. TRANSIENT ===
em.contains(transientC): false

=== 2. MANAGED (sau persist()) ===
em.contains(transientC) SAU persist(): true

=== 3. DETACHED (sau khi transaction ket thuc) ===
em.contains(transientC) SAU KHI COMMIT: false
Doc lai tu DB: Transient (ky vong VAN LA 'Transient', vi sua tren DETACHED KHONG tu luu)
```

Sửa field trên entity DETACHED **hoàn toàn không có tác dụng** xuống database — đọc lại từ
transaction mới vẫn thấy giá trị cũ, dù object Java trong bộ nhớ đã bị thay đổi.

## Interview Question (Central)

> JPA Entity có những trạng thái nào trong vòng đời của nó? Sự khác biệt giữa Managed và
> Detached là gì, và vì sao điều này quan trọng?

## Objectives

- [ ] Phân biệt chính xác 4 trạng thái: Transient, Managed, Detached, Removed
- [ ] Tự tay chứng minh bằng thực nghiệm hành vi khác biệt của từng trạng thái qua
      `em.contains()`
- [ ] Hiểu `merge()` tạo ra một **instance Managed mới**, không "làm sống lại" chính object
      Detached ban đầu

## Prerequisites

- Chapter 05 — hiểu Persistence Context, nơi định nghĩa trạng thái Managed.
- Chapter 06 — hiểu Dirty Checking chỉ áp dụng cho Managed entity.

## Used Later

- **Chapter 08 (Lazy Loading)** — truy cập một quan hệ lazy trên entity Detached là nguồn gốc
  phổ biến nhất của `LazyInitializationException`.
- **Chapter 11 (Cascade)** — cascade quyết định trạng thái của entity liên kết thay đổi theo
  entity cha như thế nào.

## Problem

Không phải mọi object `Customer` trong bộ nhớ đều "giống nhau" về mặt quan hệ với database — một
object vừa `new` ra chưa hề liên quan gì tới Hibernate, một object vừa `find()` được đang được
theo dõi tích cực, một object giữ lại sau khi transaction/request đã kết thúc lại "trôi nổi" tự
do, không còn ai theo dõi. Nếu không phân biệt rõ các trạng thái này, dễ mắc lỗi "sửa dữ liệu
nhưng không hiểu tại sao nó không được lưu" (hoặc ngược lại, không hiểu vì sao nó lại được lưu).

## Concept

Một entity JPA trải qua 4 trạng thái: **Transient** (vừa `new`, chưa từng liên quan tới
Hibernate, không có trong Persistence Context, thường chưa có ID) → **Managed** (sau
`persist()`/`find()`, đang nằm trong Persistence Context, được Dirty Checking theo dõi, Chapter
06) → **Detached** (Persistence Context đã đóng — transaction kết thúc — entity vẫn là object
Java hợp lệ nhưng không còn được theo dõi) → **Removed** (sau `remove()`, đã đánh dấu xoá, sẽ
biến mất khỏi database khi flush).

## Why?

Bốn trạng thái phản ánh chính xác **mối quan hệ hiện tại** giữa một object Java và Persistence
Context (Chapter 05) — không phải một thuộc tính cố định của bản thân entity. Việc phân biệt rõ
ràng cho phép trả lời chính xác câu hỏi "sửa entity này có tự động lưu xuống database không" chỉ
bằng cách xác định nó đang ở trạng thái nào: chỉ Managed mới được Dirty Checking (Chapter 06) tự
động theo dõi; Transient và Detached đều cần một hành động tường minh (`persist()` hoặc
`merge()`) để "gia nhập lại" vào Persistence Context.

## How?

```java
Customer c = new Customer("A");        // TRANSIENT
em.contains(c); // false

em.persist(c);                          // MANAGED
em.contains(c); // true

txManager.commit(tx);                   // transaction ket thuc -> DETACHED
em.contains(c); // false
c.setName("B"); // sua tren DETACHED, KHONG tu dong luu

Customer merged = em.merge(c);          // dua tro lai MANAGED (instance MOI!)
// merged != c

em.remove(managed);                     // REMOVED
```

## Visualization

```
   new Customer()
        │
        ▼
   TRANSIENT ─────persist()────► MANAGED ◄────merge()─────┐
                                    │  │                    │
                          Dirty Checking                    │
                          (Chapter 06)                       │
                                    │                        │
                        transaction ket thuc                 │
                                    │                        │
                                    ▼                        │
                               DETACHED ──────────────────────┘
                                    (sua field: KHONG tu luu)

   MANAGED ───remove()───► REMOVED ───flush/commit───► (bien mat khoi DB)
```

## Example

```java
System.out.println("=== 1. TRANSIENT ===");
Customer transientC = new Customer("Transient");
System.out.println("em.contains: " + em.contains(transientC)); // false

System.out.println("=== 2. MANAGED ===");
em.persist(transientC);
System.out.println("em.contains SAU persist(): " + em.contains(transientC)); // true
System.out.println("id: " + transientC.getId());

System.out.println("=== 3. DETACHED ===");
txManager.commit(tx);
System.out.println("em.contains SAU commit: " + em.contains(transientC)); // false
transientC.setName("Sua khi DA Detached");
```

Kết quả thật (Hibernate ORM 6.5.3.Final):

```
=== 1. TRANSIENT ===
em.contains(transientC): false
=== 2. MANAGED ===
Hibernate: insert into customer (name,id) values (?,default)
em.contains(transientC) SAU persist(): true
id da duoc gan: 1
=== 3. DETACHED ===
em.contains(transientC) SAU KHI COMMIT: false
Doc lai tu DB: Transient (ky vong VAN LA 'Transient', vi sua tren DETACHED KHONG tu luu)
```

`em.contains()` thay đổi chính xác theo dự đoán ở từng bước. Sửa `transientC.setName(...)` sau
khi đã Detached **không hề** ảnh hưởng database — đọc lại từ transaction mới vẫn thấy giá trị cũ
`"Transient"`.

**`merge()` — đưa Detached quay lại Managed:**

```java
Customer mergedResult = em.merge(transientC);
System.out.println("transientC == mergedResult: " + (transientC == mergedResult));
txManager.commit(txMerge);
```

Kết quả thật:

```
transientC == mergedResult: false (merge() KHONG sua doi tuong goc, tra ve INSTANCE KHAC)
Hibernate: update customer set name=? where id=?
...
Sau merge() + commit, doc lai: Sua khi DA Detached
```

`merge()` trả về một **instance hoàn toàn khác** (`transientC == mergedResult` là `false`) —
`merge()` **không** biến `transientC` (vẫn Detached) thành Managed; nó tạo/tìm một entity Managed
mới, sao chép dữ liệu từ `transientC` vào đó, rồi trả về entity Managed mới này. Chỉ sau khi
`merge()` + `commit()`, thay đổi mới thực sự xuống database.

**Removed:**

```java
Customer toRemove = em.find(Customer.class, id);
em.remove(toRemove);
txManager.commit(txRemove);
Customer afterRemove = em.find(Customer.class, id);
System.out.println("Sau remove()+commit: " + afterRemove); // null
```

Kết quả thật:

```
Hibernate: delete from customer where id=?
...
Sau khi remove()+commit, em.find() tra ve: null
```

## Deep Dive

**Vì sao `merge()` không thể đơn giản "biến" object Detached thành Managed tại chỗ, mà phải tạo
một instance mới?** Persistence Context (Chapter 05) đảm bảo tính chất "identity map" — với một
ID cụ thể, chỉ có **đúng một** object Managed đại diện cho nó tại một thời điểm. Nếu Persistence
Context **đã có sẵn** một entity Managed khác cho cùng ID đó (ví dụ đã được `find()` trước đó
trong cùng transaction), việc "biến" object Detached thành Managed trực tiếp sẽ **vi phạm** tính
duy nhất này — sẽ có hai object Managed cho cùng một ID, hoàn toàn mâu thuẫn với chính định nghĩa
identity map. `merge()` giải quyết bằng cách: tìm (hoặc tạo) đúng entity Managed duy nhất cho ID
đó trong Persistence Context hiện tại, **sao chép** dữ liệu từ object Detached vào entity Managed
đó, rồi trả về entity Managed đó — object Detached ban đầu (`transientC`) vẫn giữ nguyên trạng
thái Detached, không hề bị thay đổi bởi lời gọi `merge()`.

## Engineering Insight

**Đây là cạm bẫy phổ biến nhất khi làm việc với entity Detached trong kiến trúc nhiều tầng (ví
dụ tầng Controller nhận entity từ tầng Service đã đóng transaction) — vì sao?** Một pattern sai
phổ biến: tầng Service load entity, trả về cho Controller (lúc này entity đã Detached vì
transaction của Service đã kết thúc), Controller sửa đổi entity đó rồi **gọi lại** một method
Service khác, kỳ vọng thay đổi tự động lưu (vì "tôi đã sửa entity rồi mà"). Nhưng vì entity đã
Detached, Dirty Checking (Chapter 06) hoàn toàn không áp dụng — thay đổi "biến mất" một cách âm
thầm, không có exception nào cảnh báo. Giải pháp đúng: hoặc giữ toàn bộ logic sửa đổi bên trong
ranh giới transaction (Service method `@Transactional`, Phase 5 Chapter 13), hoặc gọi `merge()`
tường minh khi thực sự cần "đưa" một entity Detached (ví dụ đến từ tầng API, sau khi deserialize
từ JSON) trở lại Managed.

## Historical Note

Bốn trạng thái entity (Transient/Managed/Detached/Removed) được chuẩn hoá chính thức trong đặc
tả JPA (2006), nhưng khái niệm này vốn đã tồn tại trong Hibernate từ trước đó (dùng tên gọi hơi
khác: transient/persistent/detached) — một trong nhiều trường hợp Hibernate định hình khái niệm
nền tảng, sau này được đặc tả JPA chuẩn hoá lại (đã nhắc ở Chapter 01).

## Myth vs Reality

- **Myth:** "Sửa một entity Java bất kỳ luôn được tự động lưu xuống database, miễn nó từng được
  load từ database."
  **Reality:** Đã chứng minh bằng thực nghiệm — chỉ đúng khi entity đang Managed. Entity
  Detached (dù từng được load) hoàn toàn không tự động lưu thay đổi.

- **Myth:** "`merge()` sửa trực tiếp object Detached, biến nó thành Managed."
  **Reality:** Đã chứng minh bằng thực nghiệm — `merge()` trả về một **instance khác**, object
  Detached ban đầu không hề thay đổi trạng thái.

## Common Mistakes

- **Giữ entity Detached qua nhiều tầng, sửa đổi rồi kỳ vọng tự động lưu** — cạm bẫy phổ biến
  nhất, xem Engineering Insight.
- **Dùng object trả về từ `transientC` sau khi gọi `merge()` thay vì dùng giá trị trả về của
  chính `merge()`** — chỉ instance được `merge()` trả về mới thực sự Managed, object gốc vẫn
  Detached.
- **Không hiểu vì sao entity vừa `remove()` vẫn còn dùng được trong code cho tới khi flush** —
  `remove()` chỉ **đánh dấu** entity, `DELETE` thật sự chạy lúc flush/commit, entity Java vẫn
  tồn tại bình thường trong bộ nhớ tới thời điểm đó.

## Best Practices

- Giữ mọi logic sửa đổi entity bên trong ranh giới transaction (`@Transactional`, Phase 5
  Chapter 13) — tránh giữ entity Detached qua nhiều lớp rồi sửa đổi.
- Khi thực sự cần cập nhật một entity đến từ bên ngoài (ví dụ payload API), dùng `merge()` tường
  minh và luôn dùng giá trị **trả về** từ `merge()`, không dùng object gốc truyền vào.
- Kiểm tra `em.contains()` khi debug để xác nhận chính xác trạng thái hiện tại của một entity.

## Debug Checklist

- [ ] Sửa entity nhưng thay đổi không lưu xuống database, không có exception nào? → kiểm tra
      entity có đang Detached không (`em.contains()` trả về `false`).
- [ ] `merge()` không có tác dụng như mong đợi? → kiểm tra có đang dùng object trả về từ
      `merge()`, hay vẫn dùng object gốc truyền vào.
- [ ] Entity vừa `remove()` vẫn "sống" trong code sau đó? → bình thường, `DELETE` thật sự chỉ
      chạy lúc flush/commit.

## Summary

Entity JPA trải qua 4 trạng thái: Transient (mới `new`, không liên quan Hibernate) → Managed
(sau `persist()`/`find()`, trong Persistence Context, được Dirty Checking theo dõi) → Detached
(transaction kết thúc, không còn theo dõi) → Removed (sau `remove()`, sẽ xoá khi flush). Đã
chứng minh bằng thực nghiệm qua `em.contains()` ở từng bước: sửa entity Detached hoàn toàn không
tự động lưu (đọc lại vẫn thấy giá trị cũ); `merge()` không sửa trực tiếp object Detached mà trả
về một **instance Managed khác** — phải dùng đúng giá trị trả về từ `merge()` để thay đổi thực
sự có hiệu lực. Giữ entity Detached qua nhiều tầng rồi sửa đổi, kỳ vọng tự động lưu, là cạm bẫy
phổ biến nhất liên quan tới vòng đời entity.

## Interview Questions

**Mid**

- JPA Entity có những trạng thái nào trong vòng đời của nó?
- Sự khác biệt giữa Managed và Detached là gì?

**Senior**

- Vì sao `merge()` không thể đơn giản "biến" object Detached thành Managed tại chỗ?
- Giải thích một cạm bẫy thực tế liên quan tới entity Detached trong kiến trúc nhiều tầng, và
  cách khắc phục.

## Exercises

- [ ] Chạy lại `EntityStateTest` ở trên, xác nhận `em.contains()` đúng ở từng bước và hành vi
      `merge()` đúng như mô tả.
- [ ] Viết một ví dụ minh hoạ đúng cạm bẫy đã nêu ở Engineering Insight (Service trả entity
      Detached, Controller sửa rồi gọi Service khác), quan sát thay đổi "biến mất".
- [ ] Sửa lại ví dụ trên để dùng `merge()` đúng cách, xác nhận thay đổi được lưu.

## Cheat Sheet

| Trạng thái | `em.contains()` | Dirty Checking hoạt động? | Cách chuyển sang Managed |
| --- | --- | --- | --- |
| Transient | `false` | Không | `persist()` |
| Managed | `true` | Có | (đã Managed) |
| Detached | `false` | Không | `merge()` (trả về instance MỚI) |
| Removed | `true` (tới khi flush) | Không (đang chờ xoá) | N/A |

## References

- Jakarta Persistence Specification — Entity Instance's Life Cycle.
- Hibernate ORM Documentation — Entity states.
