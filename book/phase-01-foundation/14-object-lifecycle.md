---
tags:
  - Java
  - JVM
  - Object
  - Lifecycle
  - Foundation
---

# Object Lifecycle

> Phase: Phase 1 — Java Foundation
> Chapter slug: `object-lifecycle`

## Metadata

```yaml
Chapter: Object Lifecycle
Phase: Phase 1 — Java Foundation
Difficulty: ★★★
Importance: ★★★
Interview Frequency: 45%
Prerequisites:
  - Chapter 03 — Class Loader
  - Chapter 07 — Garbage Collection
  - Chapter 08 — OOP Fundamentals (super, constructor chain)
Used Later:
  - Bean Lifecycle (Phase 5 — Spring) — Spring bean lifecycle là lớp bọc thêm quanh chính vòng đời này
  - Entity State (Phase 6 — Persistence) — vòng đời một JPA Entity phức tạp hơn dựa trên nền tảng ở đây
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

Chín chapter vừa qua, mỗi chapter mổ xẻ **một lát cắt** của việc một object tồn tại trong JVM:
[Chapter 03](03-class-loader.md) nạp thông tin class, [Chapter 04](04-runtime-data-areas.md)
cấp phát bộ nhớ, [Chapter 08](08-oop-fundamentals.md) chạy constructor qua `super`, [Chapter
07](07-garbage-collection.md) dọn dẹp khi không còn ai dùng tới.

Nhưng chưa chapter nào ghép chúng lại thành **một dòng thời gian liền mạch, đúng thứ tự**. Câu
hỏi tưởng đơn giản này thực ra dễ trả lời sai: "Static block chạy trước hay `static` field khởi
tạo trước? Instance block chạy trước hay constructor chạy trước? Nếu có kế thừa, `Parent` khởi
tạo xong hoàn toàn rồi mới tới `Child`, hay chúng xen kẽ nhau?" Chapter này là chapter tổng
hợp — không giới thiệu khái niệm JVM mới, mà **ghép lại** đúng thứ tự toàn bộ những gì bạn đã
học.

## Interview Question (Central)

> Một object trải qua đầy đủ những giai đoạn nào, theo đúng thứ tự nào, từ lúc class của nó
> còn chưa được nạp cho tới lúc bị Garbage Collector dọn dẹp?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Vẽ được dòng thời gian đầy đủ: nạp class → cấp phát bộ nhớ → khởi tạo static → khởi tạo
      instance → sử dụng → mất reachability → GC
- [ ] Biết chính xác thứ tự chạy giữa static block, instance block, và constructor — đặc
      biệt khi có kế thừa
- [ ] Tự tay xác nhận thứ tự đó bằng thực nghiệm
- [ ] Giải thích được vì sao `static` chỉ chạy **đúng một lần** cho cả class, dù tạo bao
      nhiêu object

## Prerequisites

- Chapter 03 — Class Loader, Loading/Linking/Initialization.
- Chapter 07 — Garbage Collection, reachability.
- Chapter 08 — `super()`, thứ tự gọi constructor cha trước.

## Used Later

- **Bean Lifecycle** (Phase 5 — Spring) — vòng đời một Spring Bean (khởi tạo, inject
  dependency, gọi `@PostConstruct`, sẵn sàng dùng, gọi `@PreDestroy`) là một lớp trừu tượng
  **bọc thêm** quanh chính vòng đời object thuần Java ở chapter này — Spring không thay thế
  nó, chỉ thêm các "điểm móc" (hook) vào giữa các bước đã có sẵn.
- **Entity State** (Phase 6 — Persistence) — một JPA Entity có thêm các trạng thái riêng
  (Transient, Managed, Detached, Removed) chồng lên trên vòng đời object cơ bản — cần hiểu
  chapter này trước để không nhầm lẫn hai khái niệm vòng đời khác nhau.

## Problem

Nếu không nắm chắc đúng thứ tự các giai đoạn, rất dễ viết code dựa trên giả định sai — ví dụ
tưởng rằng một field sẽ có giá trị nhất định khi constructor lớp con chạy, trong khi thực tế
theo đúng thứ tự, giá trị đó chưa được gán ở thời điểm đó (đúng tình huống rủi ro đã cảnh báo ở
[Chapter 11 — Engineering Insight](11-immutable.md) về việc để `this` rò rỉ quá sớm).

## Concept

Vòng đời đầy đủ của một object, gộp lại từ các chapter trước:

```
1. Loading    — Class Loader (Ch.03) tìm & đọc bytecode, tạo object Class trong Metaspace
2. Linking    — Verify, Prepare (cấp phát static field, gán default), Resolve (Ch.03)
3. Initialization — chạy static block + gán giá trị THẬT cho static field
                     (CHỈ CHẠY MỘT LẦN cho cả class, lần đầu class được dùng chủ động)
        │
        │  new Product(...)
        ▼
4. Cấp phát bộ nhớ — object được tạo trên Heap (Ch.04)
5. Khởi tạo instance — instance block + gán giá trị field, THEO ĐÚNG THỨ TỰ
                        từ lớp cha xuống lớp con (Ch.08, cơ chế super())
6. Constructor chạy xong — object sẵn sàng dùng
        │
        │  ... object được dùng, tham chiếu được truyền đi khắp nơi ...
        │
        │  không còn tham chiếu nào từ GC Roots trỏ tới object nữa
        ▼
7. Unreachable — object đủ điều kiện bị GC thu hồi (Ch.07), nhưng CHƯA CHẮC bị dọn ngay
8. GC thu hồi — bộ nhớ được giải phóng thật sự (thời điểm không xác định trước)
```

## Why?

Nếu không tách static (bước 1-3, chạy một lần cho cả class) khỏi instance (bước 4-6, chạy mỗi
lần `new`): mọi field sẽ phải khởi tạo lại từ đầu cho mỗi object, kể cả những giá trị dùng
chung không đổi (như hằng số, cấu hình) — lãng phí và mất khả năng chia sẻ trạng thái giữa các
instance của cùng một class. Việc tách bạch rõ ràng hai pha này là nền tảng cho rất nhiều cơ
chế Java dựa vào (singleton qua `static` field, hằng số `static final`, đếm số instance đã
tạo...).

## How?

Với kế thừa, thứ tự **static** luôn hoàn tất trước khi bất kỳ **instance** nào bắt đầu — và
trong mỗi pha, luôn đi từ lớp cha xuống lớp con, không bao giờ ngược lại (đúng nguyên lý
`super()` phải là dòng đầu tiên, Chapter 08):

```
Static (một lần, khi class LẦN ĐẦU được dùng chủ động):
  Parent static block/field  →  Child static block/field

Instance (mỗi lần "new", CHỈ SAU KHI static đã xong hoàn toàn):
  Parent instance block/field  →  Parent constructor  →
  Child instance block/field   →  Child constructor
```

## Visualization

```
new Child()
     │
     ▼
Class Child đã được nạp/khởi tạo (static) chưa?
     │
     ├── Chưa → nạp Parent trước (Ch.03: delegation không liên quan,
     │           đây là quan hệ kế thừa, nhưng nguyên tắc "cha trước" giống nhau)
     │           → chạy static Parent → chạy static Child
     │           (CHỈ XẢY RA MỘT LẦN DUY NHẤT, dù sau này new Child() bao nhiêu lần)
     │
     └── Rồi (đã init từ trước) → bỏ qua toàn bộ static, đi thẳng xuống dưới
     ▼
Cấp phát object trên Heap (Ch.04)
     ▼
instance block + field của Parent  →  constructor Parent chạy
     ▼
instance block + field của Child   →  constructor Child chạy
     ▼
new Child() trả về object đã sẵn sàng
```

## Example

Xác nhận toàn bộ thứ tự trên bằng thực nghiệm — mỗi dòng in ra đánh số theo đúng dự đoán lý
thuyết:

```java
public class LifecycleDemo {
    public static void main(String[] args) {
        System.out.println("main() bat dau");
        new Child();
    }
}

class Parent {
    static { System.out.println("1. Parent static block"); }
    { System.out.println("3. Parent instance block"); }
    Parent() { System.out.println("4. Parent constructor"); }
}

class Child extends Parent {
    static { System.out.println("2. Child static block"); }
    { System.out.println("5. Child instance block"); }
    Child() { System.out.println("6. Child constructor"); }
}
```

Kết quả thật (JDK 17), khớp chính xác 100% với số thứ tự đã dự đoán:

```
main() bat dau
1. Parent static block
2. Child static block
3. Parent instance block
4. Parent constructor
5. Child instance block
6. Child constructor
```

Static luôn hoàn tất trước (bước 1-2), rồi mới tới instance (bước 3-6) — và trong mỗi nhóm,
`Parent` luôn đi trước `Child`, đúng lý thuyết ở phần How?.

## Deep Dive

**"Lần đầu class được dùng chủ động" — chính xác là khi nào?** JLS liệt kê cụ thể các tình
huống kích hoạt Initialization (bước 3): tạo instance đầu tiên (`new`), gọi static method lần
đầu, truy cập/gán static field lần đầu (trừ `static final` là hằng số biên dịch — hằng số này
được "inline" thẳng vào nơi dùng, không cần khởi tạo class), hoặc class đó là lớp con của một
class đang được khởi tạo (đúng ví dụ `Child` kéo theo khởi tạo `Parent`). Ngược lại, chỉ khai
báo kiểu (`Product p;` chưa gán `new`) hoặc load class qua `Class.forName(name, false, loader)`
(tham số thứ hai `false` = không kích hoạt Initialization) **không** kích hoạt bước 3 — vẫn
dừng ở Loading/Linking. Sự phân biệt tinh tế này giải thích tại sao đôi khi thấy static block
"không chạy" dù class rõ ràng "được nhắc tới" ở đâu đó trong code.

## Engineering Insight

**Vì sao static luôn phải hoàn tất trước instance, không được phép xen kẽ?** Vì static field
đại diện cho trạng thái **dùng chung** của cả class — nếu instance đầu tiên bắt đầu khởi tạo
trong khi static còn dang dở, nó có nguy cơ đọc phải static field ở trạng thái **chưa hoàn
chỉnh** (JVM đã cấp phát default nhưng chưa chạy xong static block gán giá trị thật — đúng
bước Prepare vs Initialization đã phân biệt ở [Chapter 03](03-class-loader.md)). Việc JVM đảm
bảo tuần tự nghiêm ngặt (static xong hoàn toàn trước khi bất kỳ instance nào bắt đầu) loại bỏ
hoàn toàn lớp lỗi này — bạn không bao giờ phải tự hỏi "static field này đã sẵn sàng chưa" khi
đang ở trong constructor, câu trả lời luôn là "chắc chắn rồi".

## Historical Note

Trình tự static-trước-instance, cha-trước-con này không đổi kể từ Java 1.0 — đây là một trong
những quy tắc nền tảng và ổn định nhất của ngôn ngữ, gần như không có tranh cãi hay đề xuất
thay đổi nào qua các JEP. Điều thực sự thay đổi qua thời gian là các framework xây **thêm**
các "pha" mới lên trên nền tảng cố định này — ví dụ vòng đời Spring Bean (Phase 5) thêm các
bước như "gọi `@PostConstruct` sau khi mọi dependency đã inject xong" — bản thân object Java
gốc vẫn tuân theo đúng trình tự đã học ở chapter này.

## Myth vs Reality

- **Myth:** "Constructor luôn là nơi đầu tiên code của bạn chạy khi tạo object."
  **Reality:** Instance block (khối `{ ... }` không có từ khoá, không phải `static { ... }`)
  chạy **trước** constructor, dù thường ít được dùng hơn constructor trong thực tế — xem thứ
  tự 3-4 và 5-6 ở Example.

- **Myth:** "Static block chạy khi chương trình khởi động (như `main()`)."
  **Reality:** Static block chạy khi class đó **lần đầu được dùng chủ động** — có thể là ngay
  lúc khởi động (nếu class chứa `main()`), nhưng cũng có thể trễ hơn nhiều nếu class đó không
  được dùng tới cho đến giữa chương trình (đúng tinh thần "lazy loading" đã học ở
  [Chapter 03](03-class-loader.md)).

## Common Mistakes

- **Giả định static field đã có giá trị "thật" ngay khi class được khai báo**, quên rằng phải
  đợi tới lần dùng chủ động đầu tiên (xem Deep Dive về `static final` hằng số là ngoại lệ).
- **Gọi method có thể bị override trong constructor lớp cha.** Nếu `Parent` constructor gọi
  một method mà `Child` override, method đó sẽ chạy **trước khi** field của `Child` được khởi
  tạo (vì theo đúng thứ tự ở Example, constructor `Parent` luôn chạy trước instance
  block/constructor của `Child`) — dễ dẫn tới đọc field `Child` khi nó vẫn còn giá trị mặc
  định (`null`/`0`), một biến thể tinh vi của vấn đề "để `this` rò rỉ quá sớm" đã cảnh báo ở
  [Chapter 11](11-immutable.md).

## Best Practices

- Tránh gọi method có thể bị override từ trong constructor — nếu bắt buộc cần logic đó, cân
  nhắc dùng static factory method thay vì đặt trong constructor trực tiếp.
- Dùng static block chỉ cho khởi tạo thật sự cần chạy một lần cho cả class (cấu hình dùng
  chung, đăng ký driver...); tránh nhồi logic phức tạp/có thể ném exception vào static block
  — lỗi ở static block gây `ExceptionInInitializerError`, khó debug hơn lỗi thông thường vì
  xảy ra ở bước rất sớm trong vòng đời class.

## Production Notes

**Vấn đề:** ứng dụng crash ngay khi khởi động với `java.lang.ExceptionInInitializerError`,
stack trace không trỏ rõ ràng tới đoạn code nghiệp vụ nào.

- **Triệu chứng:** lỗi xảy ra cực sớm, thường trước cả khi log framework kịp khởi tạo đầy đủ
  để ghi log chi tiết.
- **Root cause:** một static block hoặc khởi tạo `static field` ném exception (ví dụ đọc file
  cấu hình không tồn tại ngay trong `static { Properties p = ...; }`) — vì static
  Initialization (bước 3) là bắt buộc, nếu nó ném lỗi, JVM bọc lại thành
  `ExceptionInInitializerError` và **toàn bộ class đó coi như hỏng vĩnh viễn** cho cả vòng
  đời chương trình (kể cả nếu sau đó "sửa" được điều kiện gây lỗi, class đã hỏng không tự
  phục hồi).
- **Debug:** đọc kỹ **nguyên nhân gốc** (`getCause()`) của `ExceptionInInitializerError`, không
  chỉ nhìn thông báo lỗi ngoài cùng — nó luôn bọc quanh exception thật sự xảy ra bên trong
  static block.
- **Solution:** sửa nguyên nhân gốc (file cấu hình thiếu, kết nối thất bại...); tránh đặt logic
  dễ lỗi/phụ thuộc I/O trực tiếp trong static block, chuyển sang khởi tạo lazy (khởi tạo khi
  cần dùng lần đầu, có xử lý lỗi tường minh) nếu có thể.
- **Prevention:** hạn chế tối đa logic phức tạp trong static block; nếu bắt buộc, bọc try-catch
  và log rõ ràng nguyên nhân thay vì để exception tự lan ra ngoài không kiểm soát.

## Debug Checklist

- [ ] `ExceptionInInitializerError`? → tìm `getCause()`, kiểm tra static block/static field
      initializer của class được nhắc tới trong stack trace.
- [ ] Field của lớp con có giá trị `null`/`0` bất thường ngay trong logic của lớp cha? → nghi
      ngờ lớp cha đang gọi một method bị lớp con override quá sớm (trước khi field lớp con
      khởi tạo) — xem Common Mistakes.
- [ ] Static block "không chạy" dù class có vẻ đã được nhắc tới trong code? → kiểm tra có phải
      chỉ khai báo kiểu, hay đang dùng `static final` hằng số biên dịch (xem Deep Dive).

## Source Code Walkthrough

Trình tự Initialization (bước 3) được đặc tả chính xác trong JLS §12.4 (Initialization of
Classes and Interfaces) — mô tả bằng ngôn ngữ đặc tả hình thức, không phải bytecode có thể
`javap -c` trực tiếp. Điều có thể quan sát qua bytecode: static block và static field
initializer được `javac` gộp chung vào một method đặc biệt tên `<clinit>` (class
initializer); instance block và field initializer (không static) được gộp vào đầu **mỗi**
constructor, ngay sau lời gọi `super(...)` — đây là lý do chính xác vì sao instance block luôn
chạy trước phần thân constructor bạn tự viết, dù về mặt cú pháp instance block "nằm" ở đâu đó
giữa class.

## Summary

Vòng đời một object gồm hai pha tách biệt nghiêm ngặt: **static** (Loading → Linking →
Initialization, chạy đúng một lần khi class lần đầu được dùng chủ động, cha trước con) và
**instance** (cấp phát bộ nhớ → instance block/field → constructor, chạy mỗi lần `new`, cũng
cha trước con) — static luôn hoàn tất hoàn toàn trước khi bất kỳ instance nào bắt đầu. Object
kết thúc vòng đời khi không còn reachable từ GC Roots, chờ Garbage Collector thu hồi tại một
thời điểm không xác định trước (Chapter 07). Đây là bức tranh tổng hợp của toàn bộ Chapter
03-08, là nền tảng mà vòng đời Spring Bean (Phase 5) và JPA Entity (Phase 6) sẽ xây thêm lên
trên.

## Interview Questions

**Junior**

- Kể tên các giai đoạn chính trong vòng đời một object, từ lúc nạp class tới lúc bị GC.
- Static block chạy khi nào? Chạy một lần hay mỗi lần tạo object?

**Mid**

- Với hai class có quan hệ kế thừa, thứ tự chạy static block/instance block/constructor giữa
  lớp cha và lớp con là gì? Chứng minh bằng ví dụ.
- `ExceptionInInitializerError` xảy ra khi nào? Vì sao nó nguy hiểm hơn một exception thông
  thường?

**Senior**

- Giải thích vì sao gọi một method có thể bị override từ trong constructor lớp cha là một
  anti-pattern — liên hệ trực tiếp tới đúng thứ tự vòng đời đã học.
- So sánh vòng đời object Java thuần với vòng đời Spring Bean (nếu bạn đã biết Spring) — những
  "điểm móc" nào Spring thêm vào, và chúng nằm ở đâu trong dòng thời gian đã học ở đây?

## Exercises

- [ ] Chạy lại đúng ví dụ `LifecycleDemo` ở trên, xác nhận thứ tự in ra giống hệt.
- [ ] Thêm `new Child()` lần thứ hai ngay sau lần đầu trong `main()` — dự đoán trước static
      block có in ra lần nữa không, sau đó chạy thật để xác nhận.
- [ ] Viết một `Parent` có constructor gọi một method bị `Child` override, method đó in ra giá
      trị một field của `Child` — quan sát và giải thích tại sao giá trị in ra là `null`/`0`
      thay vì giá trị bạn gán trong field initializer của `Child`.

## Cheat Sheet

```
Thứ tự đầy đủ (kế thừa Parent → Child):

1. Parent: static block + static field initializer     ┐
2. Child:  static block + static field initializer      ├─ CHỈ 1 LẦN/class
                                                          ┘
3. Parent: instance block + field initializer           ┐
4. Parent: phần thân constructor                         │
5. Child:  instance block + field initializer            ├─ MỖI LẦN "new"
6. Child:  phần thân constructor                         ┘
```

| Tình huống | Có kích hoạt Initialization (static) không? |
| --- | --- |
| `new Product()` | Có |
| Gọi static method | Có |
| Đọc/gán static field thường | Có |
| Đọc `static final` hằng số biên dịch | Không (inline sẵn) |
| Chỉ khai báo kiểu (`Product p;`) | Không |

## References

- Java Language Specification (JLS) — Chapter 12.4: Initialization of Classes and Interfaces.
- Java Language Specification (JLS) — Chapter 8.6, 8.7: Instance/Static Initializers.
