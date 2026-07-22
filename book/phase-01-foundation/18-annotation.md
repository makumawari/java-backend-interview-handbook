---
tags:
  - Java
  - Annotation
  - Foundation
---

# Annotation

> Phase: Phase 1 — Java Foundation
> Chapter slug: `annotation`

## Metadata

```yaml
Chapter: Annotation
Phase: Phase 1 — Java Foundation
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 55%
Prerequisites:
  - Chapter 08 — OOP Fundamentals
Used Later:
  - Reflection (Chapter 19) — annotation RUNTIME chỉ có ý nghĩa khi kết hợp với Reflection
  - Toàn bộ Spring (Phase 5) — @Autowired, @Component, @RestController... đều là annotation RUNTIME
  - JPA Entity (Phase 6) — @Entity, @Id, @Column là annotation định nghĩa mapping xuống database
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

Bạn đã gõ `@Override` hàng trăm lần (Chapter 08). Thử tự hỏi: dòng `@Override` đó có **ảnh
hưởng gì tới hành vi chương trình lúc chạy** không? Xoá nó đi, code có chạy khác đi không?

Câu trả lời: **không có gì thay đổi lúc runtime cả** — `@Override` chỉ là một tín hiệu cho
`javac` kiểm tra lúc biên dịch (nếu method không thực sự override gì, báo lỗi), rồi **biến mất
hoàn toàn** sau khi biên dịch xong, giống hệt số phận của thông tin kiểu Generics (đã học ở
Chapter 16).

Nhưng bạn cũng đã gõ (hoặc sẽ sớm gõ, khi học Spring ở Phase 5) `@Autowired`,
`@RestController`, `@Transactional` — những annotation **hoàn toàn thay đổi hành vi runtime**
của chương trình, dù về mặt cú pháp trông giống hệt `@Override`. Sự khác biệt này không phải
ngẫu nhiên — nó nằm ở một khai báo nhỏ, dễ bỏ qua: `@Retention`. Chapter này giải thích chính
xác cơ chế đứng sau, để khi học Spring, bạn không còn coi annotation là "phép màu".

## Interview Question (Central)

> Annotation chỉ là "chú thích trang trí" hay có thể ảnh hưởng thực sự tới hành vi lúc chạy?
> Điều gì quyết định một annotation "biến mất" sau biên dịch hay "sống sót" tới tận runtime?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Phân biệt 3 mức `RetentionPolicy`: `SOURCE`, `CLASS`, `RUNTIME` — và biết chính xác
      `@Override` thuộc loại nào
- [ ] Tự tay xác nhận bằng thực nghiệm: chỉ annotation `RUNTIME` mới đọc được qua Reflection
- [ ] Viết được một custom annotation hoàn chỉnh (`@Retention`, `@Target`, tham số)
- [ ] Hiểu rằng bản thân annotation **không tự làm gì cả** — nó chỉ là metadata, cần một đoạn
      code khác (thường dùng Reflection) chủ động đọc và hành động dựa trên nó

## Prerequisites

- Chapter 08 — đã dùng `@Override` nhiều lần, biết annotation về mặt cú pháp là gì.

## Used Later

- **Reflection** (Chapter 19) — chapter này giới thiệu annotation RUNTIME "tồn tại được tới
  đâu"; Chapter 19 sẽ dạy cách **chủ động đọc** chúng bằng Reflection API, hai chapter này
  luôn đi cùng nhau trong thực tế.
- **Spring** (Phase 5) — gần như toàn bộ "phép màu" của Spring (tự động tạo bean, tự động
  tiêm dependency, tự động map REST endpoint) đều dựa trên việc Spring dùng Reflection quét
  annotation RUNTIME lúc khởi động ứng dụng — không có gì huyền bí hơn cơ chế học ở chapter
  này.
- **JPA Entity** (Phase 6) — `@Entity`, `@Id`, `@Column` là annotation RUNTIME mà Hibernate
  đọc để biết cách map object Java xuống bảng database.

## Problem

Có những thông tin về code (metadata) không thuộc về **logic nghiệp vụ** nhưng vẫn cần gắn
liền với code đó — ví dụ: "method này ghi đè method cha, hãy kiểm tra giúp tôi"
(`@Override`), hoặc "class này nên được đăng ký làm REST controller" (`@RestController`,
Spring). Nếu không có cơ chế annotation, cách duy nhất để truyền metadata này là qua tên gọi
theo quy ước (convention) hoặc file cấu hình riêng (XML) — cả hai đều tách rời khỏi chính đoạn
code mà metadata đó mô tả, dễ lệch nhau theo thời gian khi code thay đổi mà quên cập nhật.

## Concept

Annotation là một dạng "chú thích có cấu trúc" gắn trực tiếp vào code (class, method, field,
tham số...), có thể mang theo dữ liệu (tham số). Vòng đời của một annotation — nó "sống sót"
qua được bao nhiêu giai đoạn — do chính annotation **định nghĩa nó** quyết định, thông qua
meta-annotation `@Retention`:

- **`RetentionPolicy.SOURCE`** — chỉ tồn tại trong mã nguồn, bị `javac` loại bỏ hoàn toàn,
  không xuất hiện trong `.class`. Ví dụ: `@Override`, `@SuppressWarnings`.
- **`RetentionPolicy.CLASS`** (mặc định nếu không khai báo `@Retention`) — có mặt trong
  `.class` (bytecode), nhưng JVM **không nạp vào bộ nhớ lúc runtime** — không đọc được qua
  Reflection.
- **`RetentionPolicy.RUNTIME`** — có mặt trong `.class` **và** được JVM giữ lại, đọc được qua
  Reflection lúc chương trình đang chạy. Đây là loại duy nhất Spring/Hibernate/mọi framework
  dựa vào annotation runtime có thể sử dụng.

## Why?

Nếu **mọi** annotation đều giữ tới runtime: mỗi annotation (kể cả những cái chỉ có ý nghĩa lúc
biên dịch như `@Override`) sẽ tốn thêm bộ nhớ Metaspace (Chapter 04) và một chút overhead khi
JVM nạp class — lãng phí cho những annotation không bao giờ được đọc lúc chạy. Việc chia 3 mức
`Retention` cho phép người viết annotation **chọn đúng chi phí cần thiết**: `@Override` không
cần tồn tại quá giai đoạn biên dịch, nên `SOURCE` là lựa chọn tối ưu; `@Autowired` (Spring)
cần Spring đọc được lúc khởi động ứng dụng, bắt buộc phải `RUNTIME`.

## How?

Viết một custom annotation hoàn chỉnh, dùng để đánh dấu method là một "REST endpoint" (mô
phỏng tối giản ý tưởng `@GetMapping` của Spring, sẽ gặp đầy đủ ở Phase 5):

```java
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME) // giữ tới runtime — bắt buộc để đọc được qua Reflection
@Target(ElementType.METHOD)          // chỉ được gắn lên method
@interface ApiEndpoint {
    String path(); // tham số bắt buộc
}
```

Annotation **tự nó không làm gì cả** — nó chỉ là dữ liệu mô tả. Muốn nó "có tác dụng", cần một
đoạn code khác chủ động đọc và hành động (Chapter 19 sẽ đi sâu, ở đây chỉ minh hoạ đủ để thấy
nguyên lý):

```java
Method m = SomeController.class.getMethod("listProducts");
ApiEndpoint anno = m.getAnnotation(ApiEndpoint.class);
System.out.println("Đường dẫn: " + anno.path()); // đọc được dữ liệu annotation lúc runtime
```

## Visualization

```
Mã nguồn (.java)
     │  gắn @SourceOnly, @ClassOnly, @RuntimeVisible lên cùng một method
     ▼
javac biên dịch
     │
     ├── @SourceOnly    ──✗── BỊ LOẠI, không vào .class
     │
     ├── @ClassOnly     ──→── CÓ trong .class, nhưng JVM không nạp lúc runtime
     │
     └── @RuntimeVisible ──→── CÓ trong .class VÀ JVM giữ lại lúc runtime
                                        │
                                        ▼
                          Reflection đọc được (Chapter 19)
```

## Example

Xác nhận trực tiếp: dù cả 3 annotation cùng được gắn trong mã nguồn, chỉ loại `RUNTIME` mới
đọc được qua Reflection lúc chạy:

```java
import java.lang.annotation.*;
import java.lang.reflect.*;

@Retention(RetentionPolicy.SOURCE)
@interface SourceOnly {}

@Retention(RetentionPolicy.CLASS)
@interface ClassOnly {}

@Retention(RetentionPolicy.RUNTIME)
@interface RuntimeVisible {}

public class RetentionDemo {
    @SourceOnly
    @ClassOnly
    @RuntimeVisible
    public void demo() {}

    public static void main(String[] args) throws Exception {
        Method m = RetentionDemo.class.getMethod("demo");
        Annotation[] annos = m.getAnnotations();
        System.out.println("So annotation nhin thay luc runtime: " + annos.length);
        for (Annotation a : annos) {
            System.out.println(" - " + a);
        }
    }
}
```

Kết quả thật (JDK 17):

```
So annotation nhin thay luc runtime: 1
 - @RuntimeVisible()
```

Dù `demo()` được gắn **3** annotation trong mã nguồn, `getAnnotations()` chỉ thấy **1** — bằng
chứng thực nghiệm trực tiếp: `@Retention` quyết định annotation nào "sống sót" đủ lâu để
Reflection nhìn thấy.

## Deep Dive

**`@Target` giới hạn annotation được gắn ở đâu — vì sao cần thiết?** Nếu không có `@Target`,
một annotation thiết kế riêng cho method (như `@ApiEndpoint`) có thể vô tình bị gắn nhầm lên
class hoặc field — về mặt cú pháp vẫn biên dịch được, nhưng vô nghĩa về logic (framework đọc
annotation đó, mong đợi ngữ cảnh method, sẽ xử lý sai hoặc bỏ qua trong im lặng).
`@Target(ElementType.METHOD)` khiến `javac` **chặn ngay lúc biên dịch** nếu ai đó cố gắn
`@ApiEndpoint` lên một field — chuyển một lớp lỗi tiềm ẩn từ "phát hiện lúc chạy (nếu may
mắn)" sang "chặn lúc biên dịch", đúng triết lý chung của Java ưu tiên bắt lỗi sớm đã thấy
xuyên suốt nhiều chapter trước (Generics — Chapter 16, hợp đồng equals — Chapter 12).

## Engineering Insight

**Vì sao Spring "thần kỳ" tự động tạo bean, tự động tiêm dependency chỉ từ vài annotation —
liệu có phép màu nào không?** Không có phép màu nào cả — cơ chế hoàn toàn nằm gọn trong những
gì chapter này vừa học: annotation RUNTIME (`@Component`, `@Autowired`...) chỉ là **dữ liệu
mô tả**, hoàn toàn thụ động. "Phép màu" thực sự nằm ở một đoạn code Spring viết sẵn (chạy lúc
ứng dụng khởi động), dùng Reflection (Chapter 19) quét toàn bộ class trong classpath, tìm
những class/method/field có gắn annotation RUNTIME nó quan tâm, rồi **tự viết code thay bạn**
dựa trên dữ liệu đọc được (tạo instance, gọi setter/constructor để tiêm dependency...). Nói
cách khác: annotation là "hợp đồng khai báo ý định", còn Reflection (Chapter 19) là "cơ chế
thực thi hợp đồng đó" — hai thứ tách biệt nhưng luôn đi cùng nhau trong mọi framework dựa trên
annotation.

## Historical Note

```
Trước Java 5 (2004)
    ↓
Metadata gắn với code chủ yếu qua: comment đặc biệt (Javadoc tags),
hoặc file cấu hình XML riêng biệt (rất phổ biến trong các framework Java
thời kỳ đầu như EJB 2, Struts, Hibernate bản cũ — "XML hell")
    ↓
Java 5 (2004) — JSR 175: A Metadata Facility for the Java Programming
Language — annotation ra đời như một phần cú pháp ngôn ngữ chính thức
    ↓
Hệ sinh thái Java dần chuyển từ "cấu hình bằng XML" sang "cấu hình bằng
annotation ngay trong code" — Spring, Hibernate, JAX-RS đều đi theo xu
hướng này, dù XML vẫn còn hỗ trợ song song ở nhiều nơi vì lý do tương
thích ngược
```

## Myth vs Reality

- **Myth:** "Annotation tự nó thay đổi hành vi chương trình, giống như một dạng lệnh đặc
  biệt."
  **Reality:** Annotation hoàn toàn **thụ động** — nó chỉ là dữ liệu. Hành vi thực tế (nếu
  có) luôn đến từ một đoạn code khác (thường dùng Reflection) chủ động đọc annotation đó rồi
  quyết định làm gì — xem Engineering Insight.

- **Myth:** "Mọi annotation đều đọc được bằng Reflection, vì chúng đều xuất hiện trong code."
  **Reality:** Chỉ annotation có `@Retention(RetentionPolicy.RUNTIME)` mới đọc được — bằng
  chứng trực tiếp ở Example (`@SourceOnly`/`@ClassOnly` biến mất hoàn toàn khỏi tầm nhìn của
  Reflection).

## Common Mistakes

- **Viết custom annotation quên khai báo `@Retention(RetentionPolicy.RUNTIME)`** — mặc định
  (không khai báo `@Retention` gì cả) là `CLASS`, khiến annotation "có vẻ đúng" lúc biên dịch
  nhưng Reflection **không bao giờ đọc được** lúc chạy — lỗi rất khó nhận ra vì không có bất
  kỳ cảnh báo/exception nào, code chỉ âm thầm "không hoạt động như mong đợi".
- **Quên `@Target`** — annotation bị gắn nhầm chỗ, framework đọc annotation không tìm thấy ở
  vị trí mong đợi, gây lỗi khó truy vết.
- **Tưởng annotation có "logic" bên trong** — annotation chỉ có thể khai báo tham số (kiểu dữ
  liệu, giá trị mặc định), **không thể chứa method có thân/logic thực thi** như một class
  thông thường.

## Best Practices

- Luôn khai báo tường minh `@Retention` cho custom annotation, đừng dựa vào giá trị mặc định
  (`CLASS`) — dễ gây nhầm lẫn cho người đọc code sau này.
- Luôn khai báo `@Target` phù hợp để trình biên dịch tự chặn việc gắn sai chỗ.
- Khi thiết kế annotation cho một framework/thư viện nội bộ, luôn viết kèm đoạn code đọc
  annotation đó (dùng Reflection) ngay từ đầu — annotation không đi kèm code xử lý là một
  annotation vô dụng.

## Production Notes

**Vấn đề:** một tính năng dựa trên custom annotation (ví dụ tự viết validation, tự viết
audit-logging bằng annotation nội bộ) hoạt động đúng trên máy dev khi test riêng lẻ, nhưng
"không có tác dụng gì" khi chạy trong ứng dụng thật.

- **Triệu chứng:** không có exception, không có log lỗi — annotation "trông như được gắn
  đúng" trong code nhưng hành vi mong đợi (validate, log...) không bao giờ xảy ra.
- **Root cause:** gần như luôn là annotation quên khai báo `RetentionPolicy.RUNTIME` (mặc
  định `CLASS`) — đúng Common Mistakes phổ biến nhất của chapter này.
- **Debug:** viết một đoạn code Reflection tối thiểu (giống `RetentionDemo` ở Example) để tự
  kiểm tra annotation có thực sự đọc được lúc runtime không, trước khi nghi ngờ bất kỳ logic
  nào khác.
- **Solution:** thêm `@Retention(RetentionPolicy.RUNTIME)` vào định nghĩa annotation.
- **Prevention:** viết unit test riêng cho **chính annotation** (không chỉ test logic nghiệp
  vụ dùng nó) — xác nhận annotation đọc được qua Reflection như một phần của bộ test, không
  chỉ ngầm giả định.

## Debug Checklist

- [ ] Custom annotation "không có tác dụng" dù code trông đúng? → kiểm tra ngay
      `@Retention(RetentionPolicy.RUNTIME)` có được khai báo tường minh không.
- [ ] Annotation bị framework "bỏ qua" ở một vị trí cụ thể? → kiểm tra `@Target` có cho phép
      gắn ở đúng vị trí đó không.
- [ ] Muốn xác nhận nhanh một annotation có đọc được qua Reflection không? → viết đoạn code
      tối thiểu như `RetentionDemo` ở Example, chạy thử trực tiếp.

## Source Code Walkthrough

Annotation `RUNTIME` được lưu trong `.class` dưới dạng một cấu trúc dữ liệu riêng gọi là
`RuntimeVisibleAnnotations` (một phần của class file format, JVM Specification §4.7.16) —
đây chính là "kho dữ liệu" mà `Method.getAnnotations()` (Reflection, Chapter 19) đọc ra lúc
runtime. Annotation `CLASS` cũng được lưu trong `.class` nhưng ở một mục khác
(`RuntimeInvisibleAnnotations`) mà JVM không nạp vào cấu trúc `Method`/`Class` khả dụng cho
Reflection lúc chạy — đây là cơ chế cụ thể (không chỉ là quy ước) giải thích chính xác tại sao
`RetentionPolicy` lại tạo ra khác biệt rõ ràng đến vậy ở tầng bytecode.

## Summary

Annotation là metadata gắn vào code, bản thân **hoàn toàn thụ động**, không tự làm gì cả —
hành vi thực tế (nếu có) luôn tới từ code khác (thường dùng Reflection, Chapter 19) chủ động
đọc và xử lý nó. `@Retention` quyết định annotation "sống sót" tới đâu: `SOURCE` (biến mất
sau biên dịch, như `@Override`), `CLASS` (có trong bytecode nhưng JVM không nạp lúc runtime,
mặc định nếu không khai báo), `RUNTIME` (đọc được qua Reflection — bắt buộc cho mọi annotation
mà framework như Spring/Hibernate cần xử lý lúc chạy). Quên khai báo `RUNTIME` là lỗi phổ biến
nhất, âm thầm khiến annotation "vô dụng" mà không có bất kỳ cảnh báo nào.

## Interview Questions

**Junior**

- Annotation là gì? `@Override` có ảnh hưởng gì tới chương trình lúc chạy không?
- `@Retention` dùng để làm gì?

**Mid**

- Phân biệt `RetentionPolicy.SOURCE`, `CLASS`, `RUNTIME`. Loại nào Spring cần dùng cho
  `@Autowired`, vì sao?
- Viết một custom annotation có tham số, khai báo đúng `@Retention` và `@Target`.

**Senior**

- Giải thích cơ chế Spring dùng annotation + Reflection để "tự động" tạo bean/tiêm
  dependency — không có phép màu nào, hãy mô tả từng bước cụ thể.
- Annotation được lưu ở đâu trong file `.class`? Giải thích vì sao `RuntimeVisibleAnnotations`
  và `RuntimeInvisibleAnnotations` là hai cấu trúc tách biệt, không phải một cờ đơn giản.

## Exercises

- [ ] Chạy lại đúng ví dụ `RetentionDemo` ở trên, xác nhận chỉ 1/3 annotation đọc được.
- [ ] Viết custom annotation `@ApiEndpoint` như ở phần How?, gắn lên một vài method của một
      class tự chọn, viết đoạn code Reflection quét toàn bộ method của class đó, in ra
      `path()` của những method có gắn annotation.
- [ ] Cố tình quên `@Retention` (dùng mặc định) cho một annotation, viết code Reflection cố
      đọc nó — xác nhận nhận được `null`/không thấy annotation nào, tự tái hiện đúng lỗi phổ
      biến đã nêu ở Common Mistakes/Production Notes.

## Cheat Sheet

| RetentionPolicy | Có trong `.class`? | Đọc được qua Reflection? | Ví dụ |
| --- | --- | --- | --- |
| `SOURCE` | Không | Không | `@Override`, `@SuppressWarnings` |
| `CLASS` (mặc định) | Có | Không | Công cụ phân tích bytecode (ít gặp trong code nghiệp vụ) |
| `RUNTIME` | Có | Có | `@Autowired`, `@Entity`, mọi annotation Spring/Hibernate cần |

| Meta-annotation | Việc gì |
| --- | --- |
| `@Retention` | Quyết định vòng đời annotation |
| `@Target` | Giới hạn vị trí được phép gắn (method, field, class...) |

## References

- Java Language Specification (JLS) — Chapter 9.6: Annotation Types.
- Java Virtual Machine Specification (Oracle) — §4.7.16: The RuntimeVisibleAnnotations
  Attribute.
- JSR 175: A Metadata Facility for the Java Programming Language.
