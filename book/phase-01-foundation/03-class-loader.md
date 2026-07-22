---
tags:
  - Java
  - JVM
  - ClassLoader
  - Foundation
---

# Class Loader

> Phase: Phase 1 — Java Foundation
> Chapter slug: `class-loader`

## Metadata

```yaml
Chapter: Class Loader
Phase: Phase 1 — Java Foundation
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 55%
Prerequisites:
  - Chapter 01 — Một chương trình Java chạy như thế nào?
  - Chapter 02 — JDK, JRE, JVM
Used Later:
  - Runtime Data Areas (Chapter 04) — Metaspace là nơi Class Loader lưu class đã nạp
  - Reflection (Chapter 17)
  - Spring (Phase 5) — Spring Boot dùng custom ClassLoader để nạp lại code khi DevTools reload
Estimated Reading: 20 phút
Estimated Practice: 25 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 01](01-mot-chuong-trinh-java-chay-nhu-the-nao.md) — bạn đã chạy
> `java -Xlog:class+load=info Main` và thấy `Main` và `Product` xuất hiện trong log, cùng với
> hàng trăm class khác bạn chưa từng viết.

Hàng trăm class đó đến từ đâu? Ai quyết định thứ tự nạp chúng? Và đây mới là câu hỏi thú vị
nhất: **nếu bạn tự viết một class tên là `java.lang.String` trong project của mình, chuyện gì
sẽ xảy ra khi chương trình chạy?**

Trực giác có thể nói: "class của tôi sẽ ghi đè lên `String` thật, làm hỏng toàn bộ chương
trình — kể cả code Java tôi không hề đụng tới." Nếu điều đó đúng, Java sẽ là một ngôn ngữ cực
kỳ nguy hiểm: bất kỳ thư viện bên thứ ba nào cũng có thể âm thầm thay thế `String`, `Object`,
hay bất kỳ class lõi nào để chèn mã độc.

Điều đó **không** xảy ra. Và cơ chế ngăn nó xảy ra chính là chủ đề chương này: Class Loader
không phải một, mà là một **hệ thống phân cấp có chủ đích**.

## Interview Question (Central)

> Class Loader hoạt động như thế nào? Tại sao JVM dùng tới 3 loại Class Loader phân cấp thay
> vì chỉ một?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Kể tên 3 loại Class Loader chuẩn (Bootstrap, Platform, Application) và biết loại nào
      nạp loại class nào
- [ ] Giải thích được cơ chế **Parent Delegation Model** và vì sao nó tồn tại (bảo mật)
- [ ] Phân biệt được 3 giai đoạn Loading → Linking (Verify/Prepare/Resolve) → Initialization
- [ ] Tự tay quan sát được `getClassLoader()` trả về gì cho class hệ thống và class tự viết
- [ ] Đọc hiểu (ở mức khái niệm) thuật toán `loadClass()` thật trong `java.lang.ClassLoader`

## Prerequisites

- Chapter 01 — hiểu khái niệm nạp class (class loading) ở mức tổng quan.
- Chapter 02 — hiểu JVM là gì, để phân biệt "Class Loader" là một thành phần *bên trong* JVM.

## Used Later

- **Runtime Data Areas** (Chapter 04) — thông tin class do Class Loader nạp được lưu ở vùng
  nhớ gọi là Metaspace; chapter đó sẽ giải thích Metaspace nằm ở đâu, khác Heap ra sao.
- **Reflection** (Chapter 17) — API Reflection thao tác trực tiếp lên các đối tượng `Class`
  mà Class Loader tạo ra.
- **Spring** (Phase 5) — Spring Boot DevTools và các application server (Tomcat) dùng
  Class Loader tuỳ chỉnh để hỗ trợ hot-reload, cô lập class giữa các ứng dụng web khác nhau
  chạy chung một JVM.

## Problem

JVM cần nạp class từ nhiều nguồn khác nhau: class lõi của chính Java (`java.lang.String`,
`java.util.List`...), class thuộc các module JDK nội bộ, và class do bạn/thư viện bên thứ ba
viết. Nếu dùng chung một cơ chế nạp không phân biệt nguồn gốc, hai vấn đề nghiêm trọng xảy ra:

1. **Bảo mật:** code của bạn (hoặc một thư viện độc hại) có thể tự định nghĩa một class tên
   `java.lang.String`, "giả mạo" class lõi để đánh cắp dữ liệu đi qua nó.
2. **Xung đột:** hai thư viện bên thứ ba khác nhau có thể chứa class trùng tên — JVM cần biết
   dùng bản nào, từ đâu.

## Concept

JVM giải quyết bằng cách chia việc nạp class cho **nhiều Class Loader xếp theo tầng**, mỗi
tầng chỉ tin tưởng những nguồn nhất định, và bắt buộc tầng con phải **hỏi tầng cha trước** khi
tự nạp — gọi là **Parent Delegation Model**.

Ba Class Loader chuẩn (từ trong ra ngoài, từ tin cậy nhất đến ít tin cậy nhất):

- **Bootstrap Class Loader** — nạp class lõi tuyệt đối (`java.lang.*`, `java.util.*`...,
  nằm trong module `java.base`). Viết bằng native code (không phải Java), không có "class
  loader cha" — đây là gốc của toàn bộ hệ thống. Đây là lý do
  `String.class.getClassLoader()` trả về `null` (Chapter 01 đã thấy hiện tượng "class được
  nạp trước main()" — Bootstrap chính là loader đứng sau phần lớn class đó).
- **Platform Class Loader** (tên cũ trước Java 9: *Extension Class Loader*) — nạp các module
  JDK không thuộc lõi tuyệt đối (ví dụ các API bổ trợ đi kèm JDK).
- **Application Class Loader** (còn gọi *System Class Loader*) — nạp class từ classpath của
  chính ứng dụng bạn viết. Đây là loader nạp `Main`, `Product` ở Chapter 01.

## Why?

Nếu không có Parent Delegation:

- Một `class Hacker extends ClassLoader` có thể tự định nghĩa `java.lang.String` phiên bản
  giả, nạp nó *trước* khi JVM kịp nạp `String` thật từ Bootstrap — và mọi đoạn code dùng
  `String` trong toàn bộ chương trình (kể cả code bạn không viết) sẽ chạy trên phiên bản giả
  đó.
- Với Parent Delegation, `Hacker` classloader dù có tự định nghĩa `java.lang.String`, khi có
  ai đó yêu cầu nạp `java.lang.String`, yêu cầu luôn bị đẩy lên Bootstrap trước — Bootstrap
  đã có sẵn `String` thật, nạp nó, và **không bao giờ hỏi xuống** class loader con nữa. Bản
  giả của `Hacker` không bao giờ có cơ hội được dùng cho namespace `java.lang.*`.

## How?

Việc nạp một class gồm 3 giai đoạn tách biệt:

1. **Loading** — tìm file `.class` (từ classpath, jar, mạng...), đọc bytes, tạo đối tượng
   `Class` tương ứng trong bộ nhớ.
2. **Linking**, gồm 3 bước con:
   - *Verify* — kiểm tra bytecode hợp lệ, không vi phạm quy tắc an toàn của JVM (không nhảy
     lung tung, không truy cập bộ nhớ trái phép).
   - *Prepare* — cấp phát bộ nhớ cho các static field, gán giá trị mặc định (0, `null`...).
   - *Resolve* — phân giải các tham chiếu symbolic (tên class/method dạng chuỗi trong
     bytecode) thành tham chiếu trực tiếp trong bộ nhớ.
3. **Initialization** — chạy static initializer và gán giá trị thật cho static field (đoạn
   code trong khối `static { ... }` hoặc `static int x = 5;`).

Còn thứ tự **ai hỏi ai** khi cần nạp một class (Parent Delegation):

```
Application Class Loader cần nạp class X
        │
        │  (1) hỏi cha trước, KHÔNG tự nạp ngay
        ▼
Platform Class Loader
        │
        │  (2) hỏi cha trước
        ▼
Bootstrap Class Loader
        │
        │  (3) Bootstrap có X không?
        ├── Có → nạp, trả kết quả xuống. DỪNG Ở ĐÂY.
        └── Không → báo lại cho Platform
                       │
                       │  (4) Platform có X không (trong phạm vi nó phụ trách)?
                       ├── Có → nạp, trả kết quả xuống. DỪNG Ở ĐÂY.
                       └── Không → báo lại cho Application
                                      │
                                      │  (5) Application tự tìm X trên classpath
                                      ├── Có → nạp
                                      └── Không → ClassNotFoundException
```

## Visualization

```
Bootstrap Class Loader          (nạp java.lang.*, java.util.* — lõi tuyệt đối)
        ▲  delegation (hỏi lên)
        │
Platform Class Loader           (nạp các module JDK khác)
        ▲  delegation (hỏi lên)
        │
Application Class Loader        (nạp class của BẠN — Main, Product, thư viện bên thứ ba)
```

Mũi tên hướng LÊN vì đó là hướng **yêu cầu được hỏi tới** (delegation) — còn việc **thực sự
nạp** thì luôn cố gắng xảy ra ở tầng cao nhất (gần Bootstrap nhất) có khả năng, rồi kết quả
mới "chảy" ngược xuống.

## Example

```java
public class ClassLoaderDemo {
    public static void main(String[] args) {
        System.out.println("String's loader: " + String.class.getClassLoader());
        System.out.println("ClassLoaderDemo's loader: "
            + ClassLoaderDemo.class.getClassLoader());
        System.out.println("ClassLoaderDemo loader's parent: "
            + ClassLoaderDemo.class.getClassLoader().getParent());
    }
}
```

Chạy `javac ClassLoaderDemo.java && java ClassLoaderDemo`, kết quả thật (JDK 17):

```
String's loader: null
ClassLoaderDemo's loader: jdk.internal.loader.ClassLoaders$AppClassLoader@22d8cfe0
ClassLoaderDemo loader's parent: jdk.internal.loader.ClassLoaders$PlatformClassLoader@331bc1aa
```

Ba điều xác nhận đúng lý thuyết ở trên:

- `String` được nạp bởi Bootstrap — nhưng Bootstrap không phải một object Java thật (nó là
  native code), nên `getClassLoader()` trả về `null` thay vì tên một class loader.
- Class tự viết (`ClassLoaderDemo`) được nạp bởi `AppClassLoader` — đúng là Application Class
  Loader.
- Cha của `AppClassLoader` là `PlatformClassLoader` — đúng thứ tự phân cấp.

## Deep Dive

**Vì sao gọi là "Parent Delegation", không phải "Parent chỉ đạo"?** Vì Application Class
Loader không hề bị Bootstrap ra lệnh — nó **chủ động** chọn hỏi cha trước. Đây là quy ước
được cài trong hành vi mặc định của `java.lang.ClassLoader.loadClass()` (xem Source Code
Walkthrough), không phải luật cứng của JVM. Điều này có nghĩa: bạn **có thể** viết một
ClassLoader tuỳ chỉnh phá vỡ quy tắc delegation (một số application server làm vậy để cô lập
class giữa các ứng dụng web) — nhưng đó là ngoại lệ có chủ đích, không phải hành vi mặc định.

## Engineering Insight

**Vì sao Platform Class Loader tồn tại — tại sao không gộp thẳng Bootstrap và Application?**

Trước Java 9, lớp giữa này tên là *Extension Class Loader*, phụ trách nạp các thư viện "mở
rộng" đặt trong thư mục `jre/lib/ext`. Sau Project Jigsaw (Java 9, module hoá JDK — xem
Chapter 02 Engineering Insight), vai trò đổi thành nạp các **module JDK chuẩn nhưng không
thuộc `java.base`** (module lõi tuyệt đối do Bootstrap giữ). Việc tách một tầng trung gian
cho phép JDK tự tổ chức nội bộ thành nhiều module rời rạc mà không phải chọn giữa hai thái
cực: hoặc coi mọi thứ đều "lõi tuyệt đối" (Bootstrap phải ôm hết, mất linh hoạt), hoặc coi mọi
thứ đều "do ứng dụng cung cấp" (mất ranh giới tin cậy). Tầng Platform là vùng đệm cho những
thứ "thuộc JDK nhưng không phải lõi tuyệt đối".

## Historical Note

```
Trước Java 9
    ↓
Extension Class Loader — nạp thư viện trong jre/lib/ext
    ↓
Java 9 (Project Jigsaw, module hoá JDK)
    ↓
Đổi tên thành Platform Class Loader — nạp các module JDK không thuộc java.base
```

> Metaspace (nơi lưu thông tin class do các Class Loader này nạp) cũng đổi lớn ở Java 8 —
> xem chi tiết ở [Chapter 04 — Runtime Data Areas](04-runtime-data-areas.md), không nhắc lại
> ở đây để tránh trùng lặp.

## Myth vs Reality

- **Myth:** "Tất cả class trong chương trình được nạp ngay khi JVM khởi động."
  **Reality:** JVM nạp **lazy** — một class chỉ được nạp khi lần đầu tiên nó được *dùng chủ
  động* (tạo instance, gọi static method, truy cập static field...). Đây là lý do log
  `-Xlog:class+load` ở Chapter 01 tiếp tục xuất hiện thêm dòng mới trong lúc chương trình
  đang chạy, không chỉ ngay lúc khởi động.

- **Myth:** "Class Loader chỉ đơn giản là đọc file `.class` từ đĩa."
  **Reality:** Class Loader còn thực hiện Verify (kiểm tra bytecode hợp lệ, chống bytecode
  bị chỉnh sửa/độc hại) — một bước bảo mật quan trọng, không chỉ đọc file.

## Common Mistakes

- **Nhầm `ClassNotFoundException` với `NoClassDefFoundError`.**
  > Xem lại: [Chapter 01 — Production Notes](01-mot-chuong-trinh-java-chay-nhu-the-nao.md)
  đã phân biệt hai lỗi này ở mức triệu chứng; ở đây bổ sung góc nhìn Class Loader:
  `ClassNotFoundException` xảy ra khi **chính Class Loader** chủ động tìm nhưng không thấy
  class (ví dụ gọi `Class.forName("...")` sai tên). `NoClassDefFoundError` xảy ra khi class
  *đã từng được resolve thành công lúc compile*, nhưng lúc runtime, Class Loader không tìm
  lại được nó nữa (thường do classpath lúc chạy khác lúc build).
- **Tưởng rằng hai class cùng tên, cùng nội dung, do hai Class Loader khác nhau nạp là "cùng
  một class".** JVM coi định danh đầy đủ của một class là **(tên class, Class Loader nạp
  nó)** — cùng tên nhưng khác Class Loader là hai class khác nhau, gây lỗi khó hiểu dạng
  `ClassCastException` dù tên class in ra giống hệt nhau.

## Best Practices

- Khi gặp lỗi liên quan tới class bị trùng/lệch version giữa các thư viện (rất phổ biến với
  Maven/Gradle dependency conflict), dùng `-Xlog:class+load=info` (Chapter 01) kèm việc kiểm
  tra Class Loader nạp class đó để xác định chính xác nó đến từ jar nào.
- Không tự viết Class Loader tuỳ chỉnh trừ khi thật sự cần cô lập (plugin system, hot reload,
  multi-tenant) — đa số ứng dụng backend không cần đụng tới tầng này.

## Production Notes

**Vấn đề:** ứng dụng chạy được cục bộ, nhưng khi deploy lên môi trường có nhiều thư viện
dùng chung (ví dụ application server chạy nhiều app, hoặc "uber jar" gộp nhiều dependency),
gặp `ClassCastException: X cannot be cast to X` — **dòng chữ y hệt hai bên `cannot be cast
to`**.

- **Triệu chứng:** exception có vẻ vô lý — class được cast trông giống hệt class đích.
- **Root cause:** đúng như Common Mistakes đã nói — có **hai bản** của cùng một class được
  nạp bởi **hai Class Loader khác nhau** (ví dụ: một bản trong jar ứng dụng, một bản trong
  jar của framework/app server). JVM coi chúng là hai kiểu dữ liệu khác nhau dù tên giống hệt.
- **Debug:** in ra `object.getClass().getClassLoader()` và so sánh với
  `TargetType.class.getClassLoader()` — nếu khác nhau, đây chính là nguyên nhân.
- **Solution:** loại bỏ bản trùng lặp của thư viện (kiểm tra cây dependency:
  `mvn dependency:tree` / `gradle dependencies`), đảm bảo chỉ một bản được nạp bởi cùng một
  Class Loader.
- **Prevention:** quét dependency conflict như một bước CI thường xuyên, đặc biệt sau khi
  thêm thư viện mới.

## Debug Checklist

- [ ] Lỗi là `ClassNotFoundException` hay `NoClassDefFoundError`? → xem phân biệt ở Common
      Mistakes.
- [ ] Lỗi là `ClassCastException` với tên class hai bên giống hệt nhau? → nghi ngờ class bị
      nạp bởi hai Class Loader khác nhau (xem Production Notes).
- [ ] Muốn biết một class cụ thể được nạp từ đâu? → `SomeClass.class.getClassLoader()` rồi
      `.getResource(...)`, hoặc chạy với `-Xlog:class+load=info`.

## Source Code Walkthrough

Thuật toán delegation mặc định nằm trong `java.lang.ClassLoader.loadClass()`, về bản chất
(diễn giải theo hành vi thật, không chép nguyên văn source):

```
loadClass(tên class):
    đã nạp rồi? → trả về luôn, không nạp lại
    chưa nạp:
        có class loader cha? → gọi parent.loadClass(tên class)
        không có cha (đây là Bootstrap) → tự tìm trong lõi JDK
        nếu cha trả về lỗi "không tìm thấy":
            tự gọi findClass(tên class) — bước này mới THỰC SỰ đọc bytecode
```

Điểm mấu chốt: bước "tự tìm" (`findClass`) chỉ được gọi **sau khi** cha đã từ chối — đúng
tinh thần Parent Delegation ở phần How?. Đây cũng là method bạn override khi viết Class
Loader tuỳ chỉnh (không override `loadClass()` trực tiếp, mà override `findClass()`, để không
vô tình phá vỡ cơ chế delegation).

## Summary

JVM dùng 3 Class Loader phân cấp — Bootstrap (lõi tuyệt đối), Platform (module JDK khác),
Application (class của bạn) — thay vì một Class Loader duy nhất, để tạo ra ranh giới tin cậy:
nhờ **Parent Delegation Model**, yêu cầu nạp một class luôn bị đẩy lên tầng cao nhất có thể
trước, khiến code ứng dụng không bao giờ "ghi đè" được các class lõi. Việc nạp một class gồm
3 giai đoạn (Loading → Linking → Initialization) và diễn ra **lazy** — chỉ khi class được
dùng lần đầu, không phải ngay lúc JVM khởi động.

## Interview Questions

**Junior**

- Kể tên 3 loại Class Loader chuẩn trong JVM. Loại nào nạp `java.lang.String`?
- `ClassNotFoundException` và `NoClassDefFoundError` khác nhau ở điểm nào?

**Mid**

- Parent Delegation Model là gì? Nó giải quyết vấn đề bảo mật gì?
- Một class được nạp "lazy" nghĩa là gì? Cho ví dụ chứng minh bằng `-Xlog:class+load`.

**Senior**

- Vì sao hai object có class cùng tên, cùng nội dung bytecode, do hai Class Loader khác nhau
  nạp, lại không thể cast qua lại (`ClassCastException`)? JVM định danh một class dựa trên
  những gì?
- Nếu bạn cần viết một hệ thống plugin cho phép nạp/gỡ plugin lúc runtime mà không restart
  ứng dụng, Class Loader đóng vai trò gì trong thiết kế đó?

## Exercises

- [ ] Chạy lại đúng ví dụ `ClassLoaderDemo` ở trên trên máy bạn, xác nhận kết quả giống hệt.
- [ ] Viết một chương trình gọi `Class.forName("com.khong.ton.tai.Gi.Ca")` (tên chắc chắn
      không tồn tại), bắt `ClassNotFoundException`, in ra thông điệp lỗi — đọc kỹ nó nói gì.
- [ ] Tìm trong project Java thật (nếu có) một trường hợp dùng `Thread.currentThread()
      .getContextClassLoader()` — thử giải thích vì sao code đó không dùng
      `this.getClass().getClassLoader()` thay thế.

## Cheat Sheet

| Class Loader | Nạp gì | `getClassLoader()` trả về |
| --- | --- | --- |
| Bootstrap | `java.lang.*`, `java.util.*` (module `java.base`) | `null` |
| Platform | Module JDK khác (tên cũ: Extension) | `PlatformClassLoader` |
| Application | Class của bạn, thư viện bên thứ ba | `AppClassLoader` |

| Giai đoạn | Việc gì |
| --- | --- |
| Loading | Tìm + đọc bytecode, tạo object `Class` |
| Linking → Verify | Kiểm tra bytecode hợp lệ |
| Linking → Prepare | Cấp phát bộ nhớ static field, gán default |
| Linking → Resolve | Phân giải tham chiếu symbolic → tham chiếu thật |
| Initialization | Chạy static initializer, gán giá trị thật |

## References

- Java Virtual Machine Specification (Oracle) — Chapter 5: Loading, Linking, and
  Initializing.
- `java.lang.ClassLoader` — Java SE API Documentation (Oracle).
- JEP 220: Modular Run-Time Images.
