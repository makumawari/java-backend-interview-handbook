---
tags:
  - Java
  - Reflection
  - Foundation
---

# Reflection

> Phase: Phase 1 — Java Foundation
> Chapter slug: `reflection`

## Metadata

```yaml
Chapter: Reflection
Phase: Phase 1 — Java Foundation
Difficulty: ★★★★
Importance: ★★★
Interview Frequency: 45%
Prerequisites:
  - Chapter 09 — Object (getClass())
  - Chapter 18 — Annotation
Used Later:
  - Toàn bộ Spring (Phase 5) — DI/AOP/MVC dựa trực tiếp trên Reflection để đọc annotation và gọi method động
  - JPA/Hibernate (Phase 6) — Hibernate dùng Reflection để đọc/ghi field của Entity mà không qua getter/setter
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 11](11-immutable.md) — bạn đã thiết kế `ImmutableProduct` với field
> `private final`, đủ 5 quy tắc để đảm bảo bất biến tuyệt đối.

Thử đoạn code sau, nhắm thẳng vào chính field `private final` mà bạn tin là bất khả xâm phạm:

```java
Product p = new Product("Áo thun", 199000);
System.out.println("Trước khi hack: " + p);

Field priceField = Product.class.getDeclaredField("price");
priceField.setAccessible(true); // "xin phép" JVM bỏ qua kiểm tra truy cập
priceField.set(p, 1);

System.out.println("Sau khi hack qua Reflection: " + p);
```

Kết quả thật: **`price` bị sửa thành `1`**, dù nó là `private` (Encapsulation, Chapter 08)
**và** `final` (Immutable, Chapter 11) — hai lớp bảo vệ tưởng như tuyệt đối ở tầng ngôn ngữ.
Đây không phải lỗ hổng bảo mật ngoài ý muốn — nó là một API chính thức, có chủ đích, cực kỳ
mạnh mẽ của Java: **Reflection**. Chính cơ chế "phá vỡ mọi quy tắc" này lại là nền tảng cho
hầu hết framework bạn sẽ dùng suốt sự nghiệp: Spring, Hibernate, Jackson, JUnit — tất cả đều
dựa vào Reflection để làm những việc mà code thông thường không làm được.

## Interview Question (Central)

> Reflection cho phép chương trình làm được những gì mà lập trình thông thường (gọi trực tiếp
> qua tên) không cho phép? Vì sao các framework như Spring lại phụ thuộc nặng vào nó?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Hiểu Reflection là gì: khả năng chương trình **tự kiểm tra và thao tác chính nó** lúc
      runtime, thông qua object `Class`
- [ ] Dùng được các API cốt lõi: `getDeclaredFields()`, `getDeclaredMethods()`,
      `Method.invoke()`, `Field.set()`
- [ ] Tự tay tái hiện việc Reflection bỏ qua `private`/`final` như ở Story
- [ ] Kết hợp Annotation (Chapter 18) + Reflection để tự viết một cơ chế "quét và xử lý" đơn
      giản — đúng nguyên lý lõi mà Spring dùng

## Prerequisites

- Chapter 09 — biết `getClass()` là gì, điểm khởi đầu của Reflection.
- Chapter 18 — biết annotation RUNTIME "tồn tại được" tới runtime; chapter này dạy cách
  **chủ động đọc** chúng.

## Used Later

- **Spring** (Phase 5) — toàn bộ Dependency Injection (quét class có `@Component`, tạo
  instance, tiêm dependency vào field/constructor có `@Autowired`) và AOP (tạo proxy động bọc
  quanh method có `@Transactional`) đều dựa trực tiếp trên Reflection.
- **JPA/Hibernate** (Phase 6) — Hibernate đọc/ghi giá trị field của Entity **trực tiếp qua
  Reflection**, bỏ qua getter/setter — đây là lý do một số hành vi JPA "kỳ lạ" (ví dụ:
  validation trong setter không được gọi khi Hibernate load dữ liệu từ database) chỉ có thể
  giải thích được khi đã hiểu Reflection.

## Problem

Thông thường, code Java gọi method/truy cập field theo tên **cố định lúc biên dịch** —
`product.getName()` chỉ hoạt động nếu `javac` biết chắc `Product` có method `getName()`. Nhưng
một số bài toán cần thao tác với class **mà bạn không biết trước lúc viết code** — ví dụ: một
framework như Spring cần tạo instance và gọi method của **bất kỳ class nào** người dùng
framework tự viết sau này, mà bản thân code Spring được biên dịch **trước khi** class đó tồn
tại. Không có cách nào viết `new TênClassCủaBạn()` trực tiếp trong source code Spring vì
Spring không hề biết trước class đó.

## Concept

**Reflection** là khả năng một chương trình Java **tự kiểm tra và thao tác cấu trúc của chính
nó** (class, method, field, constructor, annotation) lúc runtime — thay vì viết cố định tên
class/method lúc biên dịch, chương trình có thể làm việc với chúng dưới dạng **dữ liệu** (đọc
từ chuỗi tên class, hoặc duyệt qua object `Class` đã có sẵn).

Điểm khởi đầu luôn là object `Class` (đã gặp ở [Chapter 09](09-object.md) qua `getClass()`) —
từ đó có thể lấy ra: `Field[]` (danh sách field), `Method[]` (danh sách method),
`Constructor[]` (danh sách constructor), và annotation RUNTIME gắn trên chúng (Chapter 18).

## Why?

Nếu không có Reflection: mọi framework phải yêu cầu người dùng **tự tay** viết code nối kết
(ví dụ tự gọi `new TênClass()` và tự truyền dependency vào constructor cho từng class) —
chính xác là những gì lập trình Java trông như trước khi Spring phổ biến (thời kỳ EJB, code
"boilerplate" rất nhiều). Reflection cho phép framework **tổng quát hoá**: viết code xử lý
"mọi class có annotation X" một lần duy nhất, áp dụng được cho bất kỳ class nào người dùng
framework tự viết sau này, không cần framework "biết trước" về nó.

## How?

Kết hợp Annotation (Chapter 18) + Reflection — mô phỏng tối giản nguyên lý DI của Spring: đọc
annotation trên method, rồi **chủ động gọi** method đó dựa trên dữ liệu đọc được:

```java
import java.lang.annotation.*;
import java.lang.reflect.*;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface ApiEndpoint {
    String path();
}

public class AnnoDemo {
    @ApiEndpoint(path = "/products")
    public void listProducts() {
        System.out.println("Dang xu ly request toi /products");
    }

    public static void main(String[] args) throws Exception {
        Method m = AnnoDemo.class.getMethod("listProducts");
        ApiEndpoint anno = m.getAnnotation(ApiEndpoint.class);
        System.out.println("Method: " + m.getName());
        System.out.println("Annotation path: " + anno.path());

        AnnoDemo instance = new AnnoDemo();
        m.invoke(instance); // GỌI METHOD MÀ KHÔNG VIẾT "instance.listProducts()" TRỰC TIẾP
    }
}
```

`m.invoke(instance)` gọi `listProducts()` **không cần code biết tên method này lúc biên
dịch** — đây chính xác là cách một framework như Spring, sau khi quét thấy annotation
`@GetMapping("/products")`, tự động "biết" cần gọi method nào khi có request tới đúng đường
dẫn, dù Spring hoàn toàn không biết trước tên `listProducts` khi chính Spring được biên dịch.

## Visualization

```
Class<AnnoDemo>
     │
     ├── getMethod("listProducts") ──▶ Method (object đại diện cho method,
     │                                   KHÔNG PHẢI lời gọi method)
     │                                        │
     │                                        ├── getAnnotation(ApiEndpoint.class)
     │                                        │        → đọc dữ liệu (Chapter 18)
     │                                        │
     │                                        └── invoke(instance)
     │                                                 → THỰC SỰ GỌI method,
     │                                                   tương đương
     │                                                   instance.listProducts()
     │                                                   nhưng KHÔNG viết tên
     │                                                   method trong source code
     ▼
Field[] getDeclaredFields()  ──▶  từng Field.setAccessible(true) + Field.set(...)
                                    → đọc/ghi TRỰC TIẾP, bỏ qua private/final
```

## Example

Tái hiện đúng "cuộc tấn công" ở Story, đồng thời liệt kê toàn bộ field bằng Reflection:

```java
import java.lang.reflect.*;

public class ReflectionDemo {
    public static void main(String[] args) throws Exception {
        Product p = new Product("Ao thun", 199000);
        System.out.println("Truoc khi hack: " + p);

        Field priceField = Product.class.getDeclaredField("price");
        priceField.setAccessible(true);
        priceField.set(p, 1);

        System.out.println("Sau khi hack qua Reflection: " + p);

        System.out.println("--- Liet ke toan bo field cua Product ---");
        for (Field f : Product.class.getDeclaredFields()) {
            System.out.println(Modifier.toString(f.getModifiers()) + " "
                + f.getType().getSimpleName() + " " + f.getName());
        }
    }
}

class Product {
    private final String name;
    private final int price;
    Product(String name, int price) { this.name = name; this.price = price; }
    public String toString() { return name + " - " + price; }
}
```

Kết quả thật (JDK 17):

```
Truoc khi hack: Ao thun - 199000
Sau khi hack qua Reflection: Ao thun - 1
--- Liet ke toan bo field cua Product ---
private final String name
private final int price
```

`price` bị sửa thành `1` thành công — xác nhận `setAccessible(true)` thực sự bỏ qua cả
`private` (Encapsulation) lẫn `final` (Immutable). Phần liệt kê field xác nhận
`getDeclaredFields()` thấy được **cả field `private`** — khác với API thông thường
(`getFields()`, không có "Declared") chỉ thấy field `public`.

## Deep Dive

**`getDeclaredXxx()` khác `getXxx()` (không "Declared") như thế nào?** Đây là một cặp API dễ
nhầm, quan trọng để dùng đúng: `getFields()`/`getMethods()` (không "Declared") chỉ trả về
thành viên **`public`**, nhưng bao gồm cả những thành viên **kế thừa** từ lớp cha.
`getDeclaredFields()`/`getDeclaredMethods()` trả về **mọi** thành viên (bất kể
`public`/`private`/`protected`) nhưng **chỉ khai báo ngay trong chính class đó**, không bao
gồm thành viên kế thừa. Muốn có "mọi thành viên, kể cả private, kể cả kế thừa từ lớp cha" —
tình huống framework thường cần — phải tự viết vòng lặp gọi `getDeclaredXxx()` trên
`getSuperclass()` lặp đi lặp lại lên tới tận `Object` (Chapter 09).

## Engineering Insight

**Vì sao `setAccessible(true)` được phép tồn tại — đây không phải lỗ hổng bảo mật của Java
sao?** Đây là đánh đổi có chủ đích, không phải sơ suất: cơ chế `private`/`final` (Chapter 08,
11) được thiết kế để bảo vệ **lập trình viên khỏi chính mình** (tránh lỗi vô tình phá vỡ bất
biến/đóng gói trong code nghiệp vụ thông thường) — không phải để bảo vệ chống lại chính
JVM/framework đang chạy trong cùng một tiến trình với đầy đủ quyền hạn. Framework như
Hibernate **cần** ghi trực tiếp vào field `private` của Entity (không qua constructor/setter,
vì lúc load dữ liệu từ database, gọi setter có thể vô tình kích hoạt logic nghiệp vụ không
mong muốn) — nếu không có `setAccessible()`, những framework nền tảng của cả hệ sinh thái
Java sẽ không thể tồn tại theo đúng cách chúng đang hoạt động hiện nay. Đánh đổi: sức mạnh này
đòi hỏi **kỷ luật tự giác** — dùng Reflection để bỏ qua encapsulation trong code nghiệp vụ
thông thường (không phải viết framework) gần như luôn là dấu hiệu thiết kế có vấn đề ở nơi
khác.

## Historical Note

```
Java 1.1 (1997)
    ↓
java.lang.reflect ra đời — Reflection API cơ bản
    ↓
Java 9 (2017) — Module System (Chapter 22 sắp tới)
    ↓
setAccessible() bắt đầu bị GIỚI HẠN đối với code trong module khác không
"exports"/"opens" tường minh module đó — lần đầu tiên trong lịch sử Java,
sức mạnh "phá vỡ mọi quy tắc" của Reflection bị ràng buộc bởi ranh giới
module, không còn tuyệt đối như trước
```

Đây là một trong những lý do chính khiến việc nâng cấp lên Java 9+ từng gây khó khăn cho nhiều
framework cũ dựa nặng vào Reflection để truy cập sâu vào nội bộ JDK — chi tiết đầy đủ về module
và `opens` sẽ học ở [Chapter 22](22-module-system.md).

## Myth vs Reality

- **Myth:** "`private` và `final` là những ràng buộc tuyệt đối, không thể phá vỡ trong Java."
  **Reality:** Xem Example — Reflection với `setAccessible(true)` phá vỡ được cả hai, một
  cách hoàn toàn hợp pháp về mặt ngôn ngữ (không phải lỗi/hack ngoài ý muốn).

- **Myth:** "Reflection chỉ là một tính năng nâng cao, ít khi thực sự cần dùng trong công
  việc hàng ngày."
  **Reality:** Bạn hầu như chắc chắn **đang dùng** Reflection gián tiếp mỗi ngày nếu làm việc
  với Spring/Hibernate/JUnit — chỉ là framework đã "giấu" nó phía sau annotation, bạn không tự
  tay gọi `Method.invoke()` nhưng framework đang làm việc đó thay bạn liên tục.

## Common Mistakes

- **Lạm dụng Reflection trong code nghiệp vụ thông thường** để "lách" qua `private`/`final`
  thay vì sửa thiết kế cho đúng (thêm getter, đổi API công khai...) — dấu hiệu code smell
  nghiêm trọng, không phải giải pháp.
- **Không xử lý lỗi khi dùng Reflection.** `getMethod()`/`invoke()` có thể ném nhiều loại
  exception checked (`NoSuchMethodException`, `IllegalAccessException`,
  `InvocationTargetException` — Chapter 15) — code Reflection viết cẩu thả, không xử lý đủ,
  dễ gây crash khó chẩn đoán.
- **Không hiểu chi phí hiệu năng.** Gọi method qua Reflection (`invoke()`) chậm hơn đáng kể
  so với gọi trực tiếp (mất chi phí kiểm tra quyền truy cập, boxing/unboxing tham số...) — dù
  JVM có tối ưu (một số trường hợp JIT — Chapter 06 — có thể tối ưu tốt code Reflection dùng
  lặp lại nhiều lần), không nên dùng Reflection cho code chạy trong vòng lặp nóng
  (hot path) nếu có cách khác.

## Best Practices

- Chỉ dùng Reflection khi thực sự cần tính tổng quát (viết framework/thư viện, xử lý class
  không biết trước) — không dùng để "lách" thiết kế trong code nghiệp vụ thông thường.
- Luôn xử lý đầy đủ các exception mà API Reflection có thể ném ra.
- Cache lại object `Method`/`Field`/`Constructor` đã tra cứu nếu dùng lặp lại nhiều lần (tra
  cứu qua tên, `getMethod("...")`, có chi phí — không nên tra cứu lại mỗi lần gọi).

## Production Notes

**Vấn đề:** ứng dụng Spring chạy ổn định qua nhiều phiên bản Java, đến khi nâng cấp lên Java
9+ thì gặp `InaccessibleObjectException` khi cố `setAccessible(true)` vào một class thuộc
module JDK nội bộ.

- **Triệu chứng:** exception chỉ xuất hiện sau khi đổi version JDK, không liên quan tới thay
  đổi code nghiệp vụ nào.
- **Root cause:** Module System (Chapter 22) từ Java 9 giới hạn Reflection xuyên module nếu
  module đích không tường minh "mở" (`opens`) gói đó cho module khác truy cập sâu —
  `setAccessible()` không còn là "vượt qua mọi rào cản" tuyệt đối như trước Java 9.
- **Debug:** đọc kỹ thông báo lỗi — nó thường chỉ rõ chính xác module nào cần thêm
  `opens ... to ...` trong `module-info.java`, hoặc cần cờ dòng lệnh `--add-opens` khi chạy
  JVM.
- **Solution:** thêm khai báo `opens`/`--add-opens` tương ứng, hoặc nâng cấp framework/thư
  viện lên phiên bản đã hỗ trợ đúng module system.
- **Prevention:** khi nâng cấp version JDK lớn (đặc biệt qua mốc Java 9), luôn kiểm tra kỹ
  các thư viện dựa nặng vào Reflection sâu vào nội bộ JDK, đọc release note liên quan tới
  module system.

## Debug Checklist

- [ ] `IllegalAccessException` khi dùng Reflection? → quên gọi `setAccessible(true)` trước
      khi truy cập thành viên `private`/`protected`.
- [ ] `InaccessibleObjectException` (chỉ từ Java 9+)? → liên quan tới Module System (xem
      Production Notes, chi tiết đầy đủ ở Chapter 22).
- [ ] `NoSuchMethodException`/`NoSuchFieldException`? → sai tên chính xác (phân biệt hoa/
      thường) hoặc nhầm `getMethod()` (chỉ public) với `getDeclaredMethod()` (mọi mức truy
      cập) — xem Deep Dive.
- [ ] Code dùng Reflection chạy chậm bất thường trong vòng lặp lớn? → kiểm tra có đang tra cứu
      lại `Method`/`Field` mỗi lần lặp thay vì cache lại từ đầu không.

## Source Code Walkthrough

`Method.invoke()` ở tầng JVM không "diễn giải" lời gọi method theo cách thông dịch chậm chạp
như có thể tưởng tượng — với những method được gọi qua Reflection **nhiều lần**, HotSpot có cơ
chế tối ưu chuyển từ gọi qua tầng native (chậm, dùng cho vài lần gọi đầu) sang **tự sinh
bytecode gọi trực tiếp** method đích (nhanh hơn nhiều, áp dụng sau một ngưỡng số lần gọi) —
đúng tinh thần "tối ưu dựa trên hành vi runtime thật" đã học ở
[Chapter 06 — JIT Compiler](06-jit-compiler.md), chỉ là áp dụng cụ thể cho chính cơ chế
Reflection thay vì code bytecode thông thường.

## Summary

Reflection cho phép chương trình tự kiểm tra và thao tác cấu trúc của chính nó lúc runtime —
đọc field/method/constructor dưới dạng dữ liệu, gọi method mà không cần biết tên lúc biên
dịch (`Method.invoke()`), và với `setAccessible(true)`, **bỏ qua hoàn toàn** cả `private`
(Encapsulation, Chapter 08) lẫn `final` (Immutable, Chapter 11) — một sức mạnh có chủ đích,
không phải lỗ hổng. Đây chính là nền tảng để Spring/Hibernate/JUnit hoạt động: kết hợp
Annotation (Chapter 18, khai báo ý định) với Reflection (thực thi ý định đó) mà không cần biết
trước class người dùng sẽ viết. Từ Java 9, Module System giới hạn phần nào sức mạnh tuyệt đối
này khi truy cập xuyên module.

## Interview Questions

**Junior**

- Reflection là gì? Cho một ví dụ dùng nó để làm gì.
- `getClass()` liên quan gì tới Reflection?

**Mid**

- `getFields()` và `getDeclaredFields()` khác nhau như thế nào?
- Reflection có thể sửa được field `private final` không? Chứng minh bằng code.

**Senior**

- Giải thích cơ chế Spring dùng Reflection kết hợp Annotation để tự động tạo bean và tiêm
  dependency — mô tả từng bước cụ thể, không chỉ nói chung chung "Spring dùng Reflection".
- Từ Java 9, Module System ảnh hưởng thế nào tới sức mạnh của `setAccessible()`? Vì sao đây
  lại là một thay đổi lớn với các framework/thư viện cũ?

## Exercises

- [ ] Chạy lại đúng ví dụ `ReflectionDemo` ở trên, xác nhận `price` bị sửa thành công dù
      `private final`.
- [ ] Chạy lại đúng ví dụ `AnnoDemo` ở phần How?, thêm một method thứ hai có
      `@ApiEndpoint(path = "/orders")`, viết vòng lặp quét **toàn bộ** method của class, tự
      động in ra path và gọi (`invoke`) từng method có gắn `@ApiEndpoint` — mô phỏng đúng
      nguyên lý một router HTTP tối giản.
- [ ] Viết một method tiện ích `printAllFields(Object obj)` dùng Reflection, in ra tên và giá
      trị của **mọi** field (kể cả `private`) của bất kỳ object nào truyền vào — một công cụ
      debug nhỏ nhưng thực dụng, tương tự những gì IDE debugger làm phía sau.

## Cheat Sheet

| API | Việc gì |
| --- | --- |
| `obj.getClass()` | Lấy object `Class` — điểm khởi đầu của Reflection |
| `Class.getDeclaredFields()` | Mọi field khai báo trong class (mọi mức truy cập) |
| `Class.getFields()` | Chỉ field `public` (kể cả kế thừa) |
| `Field.setAccessible(true)` | Bỏ qua kiểm tra `private`/`final` |
| `Method.invoke(instance, args...)` | Gọi method không cần biết tên lúc biên dịch |
| `Class.getAnnotation(X.class)` | Đọc annotation RUNTIME (Chapter 18) |

## References

- Java SE API Documentation — gói `java.lang.reflect`.
- Oracle: "The Java Tutorials — Trail: The Reflection API".
- JEP 261: Module System (phần ảnh hưởng tới `setAccessible()`, liên quan Chapter 22).
