---
tags:
  - Java
  - Generics
  - Foundation
---

# Generics

> Phase: Phase 1 — Java Foundation
> Chapter slug: `generics`

## Metadata

```yaml
Chapter: Generics
Phase: Phase 1 — Java Foundation
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 65%
Prerequisites:
  - Chapter 05 — Execution Engine (bytecode, javap)
  - Chapter 08 — OOP Fundamentals
Used Later:
  - List, Map, mọi Collection (Phase 2) — Generics là nền tảng của toàn bộ Collection Framework
  - Optional, Stream (Phase 4)
  - Repository<T, ID> (Phase 6 — Spring Data JPA) — generic interface là trung tâm thiết kế
Estimated Reading: 25 phút
Estimated Practice: 25 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

Chạy đoạn code sau và đoán kết quả trước khi đọc tiếp:

```java
List<String> strings = new ArrayList<>();
List<Integer> ints = new ArrayList<>();
System.out.println(strings.getClass() == ints.getClass());
```

Trực giác nói: `List<String>` và `List<Integer>` là hai kiểu khác nhau — trình biên dịch chắc
chắn không cho gán `List<String>` vào biến `List<Integer>`. Vậy kết quả chắc là `false`?

Kết quả thật: **`true`**. Cả `strings` lẫn `ints` đều có `getClass()` trả về **chính xác cùng
một object** `java.util.ArrayList`. Generics — thứ bạn dùng hàng ngày để có được sự an toàn
kiểu dữ liệu lúc biên dịch — **hoàn toàn biến mất** lúc runtime. Đây không phải bug, mà là một
quyết định thiết kế có chủ đích, để lại dấu ấn trong gần như mọi ngóc ngách của Generics Java:
tại sao không tạo được mảng `new T[10]`, tại sao `instanceof List<String>` không hợp lệ, tại
sao có `<? extends T>` và `<? super T>` kỳ lạ. Chapter này giải thích tất cả, bắt đầu từ đúng
cơ chế đứng sau: **type erasure**.

## Interview Question (Central)

> Vì sao `List<String>` và `List<Integer>` có cùng một `.class` lúc runtime? Type erasure là
> gì, và nó gây ra những giới hạn cụ thể nào khi dùng Generics?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích được type erasure: thông tin kiểu generic chỉ tồn tại lúc compile, bị "xoá"
      ở bytecode
- [ ] Tự tay xác nhận erasure bằng `getClass()` và bằng đọc bytecode thật
- [ ] Viết được generic method/class của riêng bạn
- [ ] Dùng đúng wildcard `<? extends T>` và `<? super T>`, hiểu vì sao chúng tồn tại (nguyên
      tắc PECS)
- [ ] Biết những giới hạn cụ thể mà erasure gây ra: không `new T[]`, không `instanceof T`

## Prerequisites

- Chapter 05 — biết đọc `javap -c` ở mức cơ bản.
- Chapter 08 — hiểu Inheritance, để hiểu `<? extends T>`/`<? super T>` liên hệ tới quan hệ
  kế thừa.

## Used Later

- **List, Map, mọi Collection** (Phase 2) — toàn bộ Collection Framework được thiết kế dựa
  trên Generics; hiểu erasure giải thích được một số hành vi "kỳ lạ" của Collection (ví dụ vì
  sao không thể tạo `new ArrayList<String>[10]`).
- **Repository<T, ID>** (Phase 6 — Spring Data JPA) — Spring Data dùng generic interface làm
  trung tâm thiết kế (`JpaRepository<Product, Long>`) — hiểu Generics ở mức nguồn gốc giúp
  đọc hiểu code Spring dễ dàng hơn nhiều.

## Problem

Trước Java 5 (2004), `Collection` (Phase 2) không có Generics — một `List` có thể chứa **bất
kỳ kiểu gì** lẫn lộn (`Object`), buộc phải tự ép kiểu (cast) mỗi khi lấy phần tử ra, và lỗi ép
kiểu sai chỉ lộ ra lúc **runtime** (`ClassCastException`), không phải lúc biên dịch. Cần một
cơ chế để trình biên dịch **tự kiểm tra kiểu** ngay lúc viết code, bắt lỗi sớm hơn, nhưng đồng
thời phải **không phá vỡ** hàng triệu dòng code Java cũ (trước Generics) đã và đang chạy trong
production trên khắp thế giới.

## Concept

**Type erasure**: trình biên dịch dùng thông tin kiểu generic (`<String>`, `<Integer>`...) để
**kiểm tra** tại thời điểm biên dịch, rồi **xoá bỏ hoàn toàn** thông tin đó khi sinh bytecode
— thay `T` bằng `Object` (hoặc bằng giới hạn cụ thể nếu có `extends`), và tự động chèn thêm
lệnh ép kiểu (`checkcast`) ở đúng những chỗ cần thiết để đảm bảo an toàn.

```java
class Box<T> {
    T value;
    T get() { return value; }
}
```

Sau erasure, bytecode thực chất tương đương (về mặt khái niệm) với:

```java
class Box {
    Object value;
    Object get() { return value; }
}
```

## Why?

Nếu Java giữ nguyên thông tin kiểu generic tới tận runtime (như C# làm — gọi là *reified
generics*): mọi thư viện `.class` đã biên dịch trước Java 5 sẽ **không tương thích** với
Generics mới, buộc phải biên dịch lại toàn bộ hệ sinh thái Java đang chạy khắp nơi — một chi
phí không tưởng. Type erasure là lựa chọn có chủ đích để đạt **tương thích ngược hoàn toàn**
(binary compatibility): code cũ (không dùng Generics) và code mới (có dùng Generics) sinh ra
bytecode **tương thích lẫn nhau**, có thể trộn lẫn trong cùng một chương trình mà không cần
biên dịch lại bất cứ thứ gì đã có.

## How?

```
Compile-time (javac kiểm tra kiểu):
  List<String> list = new ArrayList<>();
  list.add("ok");        ✓ đúng kiểu
  list.add(123);          ✗ LỖI BIÊN DỊCH — javac chặn ngay tại đây

        │  ERASURE — xoá thông tin kiểu generic
        ▼

Runtime (bytecode, sau erasure):
  List list = new ArrayList();      // không còn <String>
  list.add("ok");
  String s = (String) list.get(0);  // javac tự chèn checkcast ở ĐIỂM LẤY RA,
                                      // không phải điểm thêm vào
```

Đây là lý do `ClassCastException` (nếu có, ví dụ khi trộn code generic với code cũ không dùng
Generics) luôn xảy ra ở **điểm lấy phần tử ra và ép kiểu**, không phải điểm thêm vào — vì bản
thân việc "thêm vào" một `List` đã erasure không hề kiểm tra kiểu gì ở tầng bytecode.

## Visualization

```
                Trước erasure (chỉ tồn tại lúc compile)
                ┌──────────────────────────┐
List<String>    │  javac KIỂM TRA KIỂU Ở    │
                │  ĐÂY — nếu sai, chặn      │
                │  ngay, KHÔNG cho biên     │
                │  dịch ra .class           │
                └──────────────────────────┘
                            │
                            │  ERASURE
                            ▼
                ┌──────────────────────────┐
List (raw)      │  Bytecode: List           │
                │  KHÔNG còn dấu vết         │
                │  <String> ở đây nữa        │
                └──────────────────────────┘
```

## Example

Xác nhận erasure bằng hai cách độc lập — `getClass()` và đọc bytecode thật:

```java
import java.util.ArrayList;
import java.util.List;

public class ErasureDemo {
    public static void main(String[] args) {
        List<String> strings = new ArrayList<>();
        List<Integer> ints = new ArrayList<>();
        System.out.println(strings.getClass() == ints.getClass());
        System.out.println(strings.getClass());
    }

    static <T> T firstElement(List<T> list) {
        return list.get(0);
    }
}
```

Kết quả thật (JDK 17):

```
true
class java.util.ArrayList
```

Và bytecode thật của method generic `firstElement`:

```
static <T> T firstElement(java.util.List<T>);
  Code:
     0: aload_0
     1: iconst_0
     2: invokeinterface #29,  2  // InterfaceMethod java/util/List.get:(I)Ljava/lang/Object;
     7: areturn
```

Chú ý dòng `invokeinterface`: nó gọi `List.get(I)` và khai báo kiểu trả về là
**`Ljava/lang/Object;`** — không phải `T`, vì bytecode không có khái niệm `T`. Điểm ép kiểu về
đúng `T` (thành `Object` cụ thể tuỳ ngữ cảnh gọi) xảy ra ở **phía người gọi**, không phải bên
trong `firstElement`.

## Deep Dive

**Vì sao không thể viết `new T[10]` bên trong một class generic?** Vì tạo mảng
(`anewarray` trong bytecode) cần biết **chính xác kiểu phần tử ngay lúc runtime** để JVM gắn
đúng metadata kiểu cho mảng đó (mảng trong Java, khác `List`, có kiểm tra kiểu lúc runtime khi
gán phần tử — gọi là *covariant array with runtime checks*, một chi tiết riêng của mảng Java).
Nhưng sau erasure, `T` **không còn tồn tại** ở bytecode — JVM không có cách nào biết "tạo mảng
kiểu gì" tại điểm đó. Đây là giới hạn trực tiếp, cụ thể, dễ kiểm chứng nhất của type erasure —
không phải một quy tắc tuỳ tiện, mà là hệ quả tất yếu của thiết kế đã học ở phần Concept.

Tương tự, `instanceof List<String>` không hợp lệ (chỉ `instanceof List` hợp lệ) — vì lúc
runtime, JVM chỉ còn biết đó là một `List`, không còn biết `<String>` hay `<Integer>` để so
sánh.

## Engineering Insight

**Wildcard `<? extends T>` và `<? super T>` giải quyết vấn đề gì mà `<T>` thường không giải
quyết được?** Xét bài toán: một method nhận `List<Product>` để **đọc** dữ liệu, nhưng bạn
muốn truyền vào cả `List<DiscountedProduct>` (lớp con — Chapter 08). Với khai báo
`List<Product>` thông thường, điều này **không được phép** (dù `DiscountedProduct extends
Product`) — vì Generics không tự động "hiệp biến" (covariant) như mảng: nếu cho phép, code bên
trong method có thể `list.add(new Product(...))` vào một `List` mà thực chất chỉ nên chứa
`DiscountedProduct`, phá vỡ an toàn kiểu ngay lúc runtime.

`<? extends Product>` giải quyết đúng bài toán "chỉ đọc từ list của Product hoặc lớp con nào
đó" mà không quan tâm chính xác là lớp con nào — đổi lại, bên trong method **không được phép
`add()`** gì vào list đó nữa (trình biên dịch không biết chính xác kiểu cụ thể, không thể đảm
bảo an toàn nếu cho thêm vào). Nguyên tắc ghi nhớ kinh điển là **PECS** — *Producer Extends,
Consumer Super*: dùng `<? extends T>` khi list chỉ **sản xuất** (đưa) dữ liệu ra cho bạn đọc;
dùng `<? super T>` khi list chỉ **tiêu thụ** (nhận) dữ liệu bạn đưa vào.

## Historical Note

```
Trước Java 5 (2004)
    ↓
Collection không có Generics — List chứa Object, phải tự ép kiểu thủ công,
lỗi ép kiểu sai chỉ lộ ra lúc RUNTIME (ClassCastException)
    ↓
Java 5 (2004) — Generics ra đời, dùng TYPE ERASURE để giữ tương thích ngược
hoàn toàn với toàn bộ code/thư viện đã biên dịch trước đó
    ↓
Từ đó tới nay — thiết kế erasure không đổi, dù gây một số giới hạn (Deep Dive)
được đánh đổi lấy khả năng tương thích ngược tuyệt đối — một quyết định vẫn
còn gây tranh luận trong cộng đồng Java về việc có nên đổi sang reified
generics hay không, nhưng chưa có thay đổi chính thức nào
```

## Myth vs Reality

- **Myth:** "`List<String>` và `List<Integer>` là hai kiểu khác nhau lúc runtime, giống như
  hai class khác nhau."
  **Reality:** Xem Example — cả hai đều là cùng một `java.util.ArrayList` lúc runtime, khác
  biệt **chỉ tồn tại lúc biên dịch**, hoàn toàn biến mất sau erasure.

- **Myth:** "Generics làm chương trình chạy chậm hơn vì phải kiểm tra kiểu lúc runtime."
  **Reality:** Ngược lại — vì erasure loại bỏ thông tin kiểu khỏi bytecode, **không có** kiểm
  tra kiểu generic nào xảy ra lúc runtime (chỉ có `checkcast` ở đúng những điểm cần thiết, y
  hệt như code Java cũ tự ép kiểu thủ công trước Java 5) — Generics không thêm chi phí runtime
  đáng kể nào so với cách viết code thủ công tương đương trước đó.

## Common Mistakes

- **Nhầm lẫn lỗi biên dịch (compile-time) với an toàn tuyệt đối lúc runtime.** Generics chỉ
  bảo vệ ở tầng biên dịch; nếu trộn lẫn code generic với code "raw type" cũ (không dùng
  Generics) một cách cẩu thả, `ClassCastException` vẫn có thể xảy ra lúc runtime — Generics
  không phải "phép màu" loại bỏ hoàn toàn khả năng lỗi kiểu.
- **Cố tạo mảng generic** (`new T[10]` hoặc `new List<String>[10]`) rồi bối rối vì lỗi biên
  dịch — xem lý do chính xác ở Deep Dive.
- **Dùng `<? extends T>` rồi cố `add()` vào nó** — vi phạm đúng nguyên tắc PECS ở Engineering
  Insight, trình biên dịch sẽ chặn ngay.

## Best Practices

- Luôn khai báo kiểu generic cụ thể (`List<Product>`), tránh dùng "raw type" (`List` không có
  `<>`) trong code mới — raw type chỉ nên gặp khi tương tác với code rất cũ, trước Java 5.
- Áp dụng đúng PECS khi thiết kế method nhận tham số kiểu Collection: `<? extends T>` cho
  tham số chỉ đọc, `<? super T>` cho tham số chỉ ghi vào.
- Không cần lo lắng về "chi phí hiệu năng của Generics" — như Myth vs Reality đã làm rõ, gần
  như không có chi phí runtime đáng kể.

## Production Notes

Type erasure hiếm khi trực tiếp gây sự cố production nghiêm trọng như các chapter JVM trước
đó (Chapter 04-07) — ảnh hưởng thực tế chủ yếu là **cảnh báo lúc biên dịch** (`unchecked cast`
warning) khi trộn code generic với code cũ/thư viện không dùng Generics đúng cách. Ảnh hưởng
gián tiếp đáng chú ý nhất: serialization/deserialization JSON (sẽ gặp ở Phase 5 — Spring REST)
với kiểu generic phức tạp (`List<Product>`) đôi khi gặp vấn đề chính xác vì thông tin kiểu đã
bị erasure — các thư viện như Jackson phải dùng kỹ thuật đặc biệt (`TypeReference`) để "vòng
qua" giới hạn erasure này khi deserialize.

## Debug Checklist

- [ ] Cảnh báo biên dịch "unchecked cast" hoặc "raw type"? → thường là dấu hiệu đang trộn code
      generic với code cũ không dùng `<>` — nên rà soát và thêm kiểu cụ thể.
- [ ] `ClassCastException` xảy ra dù code "trông có vẻ" dùng Generics đúng? → tìm điểm nào
      trong luồng dữ liệu có dùng raw type hoặc ép kiểu thủ công, đó thường là nơi an toàn
      kiểu bị phá vỡ.
- [ ] Deserialize JSON vào `List<CustomType>` nhưng nhận được lỗi/kiểu sai? → nghi ngờ mất
      thông tin kiểu generic do erasure, tìm giải pháp kiểu `TypeReference` của thư viện JSON
      đang dùng (Phase 5).

## Source Code Walkthrough

Đã thực hiện trực tiếp ở phần Example: bytecode của một generic method (`firstElement`) không
chứa bất kỳ dấu vết nào của tham số kiểu `T` — chỉ có `Ljava/lang/Object;`. Đây là bằng chứng
bytecode cụ thể, tự kiểm chứng được bằng `javap -c` trên bất kỳ generic method nào bạn tự viết,
không cần đọc source OpenJDK.

## Summary

**Type erasure**: `javac` dùng thông tin kiểu generic để kiểm tra an toàn lúc biên dịch, rồi
xoá bỏ hoàn toàn khi sinh bytecode (thay `T` bằng `Object`, tự chèn `checkcast` ở điểm lấy dữ
liệu ra) — mục đích là giữ **tương thích ngược hoàn toàn** với code Java trước Java 5. Hệ quả
trực tiếp: `List<String>` và `List<Integer>` có cùng `.class` lúc runtime; không thể
`new T[]`; không thể `instanceof List<String>`. Wildcard `<? extends T>`/`<? super T>` giải
quyết bài toán kế thừa trong Generics (không hiệp biến tự động như mảng), theo nguyên tắc
PECS: Producer Extends, Consumer Super.

## Interview Questions

**Junior**

- Type erasure là gì?
- `List<String>` và `List<Integer>` có phải hai kiểu khác nhau lúc runtime không?

**Mid**

- Vì sao không thể viết `new T[10]` bên trong một class generic?
- `<? extends T>` và `<? super T>` khác nhau như thế nào? Cho ví dụ khi nào dùng cái nào.

**Senior**

- Giải thích nguyên tắc PECS (Producer Extends, Consumer Super) bằng một ví dụ cụ thể bạn tự
  nghĩ ra, không phải ví dụ trong chapter.
- So sánh type erasure (Java) với reified generics (C#) — đánh đổi chính giữa hai cách tiếp
  cận là gì? Nếu Java thiết kế lại từ đầu hôm nay, bạn nghĩ nên chọn cách nào?

## Exercises

- [ ] Chạy lại đúng ví dụ `ErasureDemo` ở trên, xác nhận `getClass()` giống hệt và đọc
      bytecode bằng `javap -c` để tự thấy `Ljava/lang/Object;`.
- [ ] Viết một class generic `Box<T>` với method `T get()`/`void set(T value)`, dùng
      `javap -c` xem bytecode, xác nhận field bên trong thực chất là kiểu `Object`.
- [ ] Viết một method `static void copy(List<? extends Number> src, List<? super Number> dst)`
      — thử cố tình `add()` vào `src` để tự thấy lỗi biên dịch PECS ngăn chặn.

## Cheat Sheet

| Giới hạn do erasure | Vì sao |
| --- | --- |
| Không `new T[10]` | JVM cần biết kiểu chính xác lúc tạo mảng, `T` đã bị xoá |
| Không `instanceof List<String>` | Lúc runtime chỉ còn biết là `List`, không còn `<String>` |
| Không tạo `static` field kiểu `T` | Field static dùng chung cho MỌI tham số hoá kiểu của class |

| Wildcard | Dùng khi | Được phép |
| --- | --- | --- |
| `<? extends T>` | Chỉ đọc (Producer) | `get()`, không `add()` |
| `<? super T>` | Chỉ ghi (Consumer) | `add()`, `get()` trả về `Object` |
| `<T>` | Cả đọc lẫn ghi, kiểu cố định | `get()` và `add()` |

## References

- Java Language Specification (JLS) — Chapter 4.6: Type Erasure.
- Oracle: "The Java Tutorials — Generics" (phần Type Erasure, Wildcards).
- Effective Java (Joshua Bloch) — Item 28: "Prefer lists to arrays", Item 31: "Use bounded
  wildcards to increase API flexibility" (nguồn gốc thuật ngữ PECS).
