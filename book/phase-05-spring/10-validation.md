---
tags:
  - Spring
  - Validation
---

# Validation với Bean Validation (@Valid)

> Phase: Phase 5 — Spring
> Chapter slug: `validation`

## Metadata

```yaml
Chapter: Validation
Phase: Phase 5 — Spring
Difficulty: ★★
Importance: ★★★★
Interview Frequency: 65%
Prerequisites:
  - Chapter 09 — REST
Used Later:
  - Chapter 11 — Exception Handler
Estimated Reading: 12 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 09](09-rest.md) — `@RequestBody Product input` tự động parse JSON thành
> record Java. Nhưng "parse thành công" không có nghĩa là "dữ liệu hợp lệ" — client hoàn toàn
> có thể gửi `name` rỗng hoặc `price` âm, những giá trị **cú pháp đúng nhưng vô nghĩa về nghiệp
> vụ**.

Thử gửi dữ liệu rõ ràng vô lý (`name` rỗng, `price` âm) tới endpoint **không có** `@Valid`:

```
=== POST khong co @Valid, du lieu KHONG hop le van duoc chap nhan ===
{"id":2,"name":"","price":-999.0}
HTTP_STATUS:201
```

Server chấp nhận và lưu lại một "sản phẩm" tên rỗng, giá **âm 999** — hoàn toàn vô nghĩa về mặt
nghiệp vụ, nhưng không có cơ chế nào ngăn nó. Thêm `@Valid` vào đúng cùng endpoint, gửi lại đúng
dữ liệu đó:

```
=== POST /api/products (KHONG hop le: name rong, price am) ===
{"timestamp":"...","status":400,"error":"Bad Request","fieldErrors":{"name":"ten san pham khong duoc de trong","price":"gia phai lon hon 0"}}
HTTP_STATUS:400
```

Cùng dữ liệu, cùng endpoint — chỉ thêm một annotation `@Valid` đã thay đổi hoàn toàn kết quả.

## Interview Question (Central)

> `@Valid` hoạt động như thế nào trong Spring MVC? Điều gì xảy ra nếu quên thêm nó vào tham số
> `@RequestBody`?

## Objectives

- [ ] Dùng thành thạo các annotation validation phổ biến: `@NotBlank`, `@NotNull`, `@Positive`,
      `@Size`, `@Email`
- [ ] Tự tay chứng minh bằng thực nghiệm: thiếu `@Valid`, dữ liệu vô lý vẫn được chấp nhận; có
      `@Valid`, request bị từ chối với thông tin lỗi chi tiết theo từng field
- [ ] Hiểu `@Valid` chỉ là một "công tắc" kích hoạt Bean Validation, bản thân validation logic
      nằm trong annotation gắn trên field

## Prerequisites

- Chapter 09 — hiểu `@RequestBody`, luồng parse JSON thành object Java.

## Used Later

- **Chapter 11 (Exception Handler)** — xử lý `MethodArgumentNotValidException` (lỗi ném ra khi
  `@Valid` thất bại) để trả về response lỗi có định dạng nhất quán.

## Problem

`@RequestBody` (Chapter 09) chỉ đảm bảo dữ liệu **đúng cú pháp JSON** và **đúng kiểu dữ liệu**
Java — nó hoàn toàn không biết (và không nên biết) về các quy tắc nghiệp vụ như "tên không được
rỗng" hay "giá phải dương". Nếu không có cơ chế validate riêng, mọi endpoint phải tự viết logic
kiểm tra thủ công (`if (input.name().isBlank()) throw ...`), lặp lại ở khắp nơi, dễ bỏ sót.

## Concept

**Bean Validation** (chuẩn Jakarta, không phải riêng của Spring) cho phép khai báo quy tắc hợp
lệ **ngay trên field** của một class/record bằng annotation (`@NotBlank`, `@NotNull`,
`@Positive`, `@Size`, `@Email`, ...). `@Valid` (đặt trước tham số `@RequestBody` trong
controller) là "công tắc" báo cho Spring MVC: "hãy chạy Bean Validation trên object này trước
khi vào thân method" — nếu có vi phạm, Spring ném `MethodArgumentNotValidException` **trước
khi** code trong controller method được thực thi.

## Why?

Tách quy tắc validation ra khỏi logic controller (đặt trực tiếp trên field của DTO) cho phép
**tái sử dụng** cùng một quy tắc ở nhiều nơi dùng chung DTO đó, và giữ controller method tập
trung vào logic nghiệp vụ thực sự — không lẫn lộn với hàng loạt câu lệnh `if` kiểm tra dữ liệu.
`@Valid` như một bước gác cổng tự động trước khi vào thân method đảm bảo: nếu code bên trong
method chạy được, dữ liệu **chắc chắn** đã hợp lệ theo mọi quy tắc khai báo — loại bỏ hoàn toàn
khả năng quên kiểm tra một trường hợp nào đó.

## How?

```java
record Product(
    Long id,
    @NotBlank(message = "ten san pham khong duoc de trong") String name,
    @Positive(message = "gia phai lon hon 0") double price
) {}

@PostMapping
ResponseEntity<Product> create(@Valid @RequestBody Product input) { // "cong tac" kich hoat validation
    // Neu code chay toi day, input DA CHAC CHAN hop le
    ...
}
```

## Visualization

```
KHONG co @Valid:

  JSON request ──► parse thanh Product ──► THANG vao than method
                                             (KHONG kiem tra gi ca, du lieu vo ly van lot qua)

CO @Valid:

  JSON request ──► parse thanh Product ──► Bean Validation kiem tra TUNG field
                                                  │
                                    Hop le ──► vao than method (DAM BAO du lieu dung)
                                    KHONG hop le ──► nem MethodArgumentNotValidException
                                                      (than method KHONG BAO GIO chay toi)
```

## Example

```java
record Product(
    Long id,
    @NotBlank(message = "ten san pham khong duoc de trong") String name,
    @Positive(message = "gia phai lon hon 0") double price
) {}

@PostMapping
ResponseEntity<Product> create(@Valid @RequestBody Product input) {
    long id = idSeq.getAndIncrement();
    Product created = new Product(id, input.name(), input.price());
    store.put(id, created);
    return ResponseEntity.status(HttpStatus.CREATED).body(created);
}
```

**Không có `@Valid`** — gửi dữ liệu vô lý:

```
$ curl -X POST http://localhost:8080/api/products -d '{"name":"","price":-999}'
{"id":2,"name":"","price":-999.0}
HTTP_STATUS:201
```

Server chấp nhận và lưu — hoàn toàn không có kiểm tra nào.

**Có `@Valid`** — gửi đúng dữ liệu đó:

```
$ curl -X POST http://localhost:8080/api/products -d '{"name":"","price":-999}'
{"timestamp":"2026-07-23T08:00:52.85Z","status":400,"error":"Bad Request",
 "fieldErrors":{"name":"ten san pham khong duoc de trong","price":"gia phai lon hon 0"}}
HTTP_STATUS:400
```

Cùng payload, chỉ khác một annotation `@Valid` trong chữ ký method — request bị từ chối với
`400 Bad Request`, kèm thông tin lỗi **cho từng field cụ thể** (lấy từ `message` khai báo trong
annotation validation tương ứng). Đối chiếu trực tiếp hai kết quả xác nhận `@Valid` chính là yếu
tố quyết định duy nhất tạo ra khác biệt này.

## Deep Dive

**`MethodArgumentNotValidException` được ném ra ở đâu trong luồng xử lý request, và vì sao thân
method `create()` "không bao giờ chạy tới" khi validation thất bại?** `@Valid` được xử lý bởi
`ArgumentResolver` của Spring MVC — thành phần chịu trách nhiệm "chuẩn bị" từng tham số của
controller method **trước khi** gọi chính method đó. Khi resolver này thấy `@Valid`, nó chạy
Bean Validation (thông qua implementation mặc định — Hibernate Validator, đã thấy log
`"HV000001: Hibernate Validator 8.0.1.Final"` khi ứng dụng khởi động) trên object vừa parse
được; nếu có vi phạm, nó **ném exception ngay tại bước chuẩn bị tham số này** — trước khi
`DispatcherServlet` thực sự gọi `create(input)`. Đây là lý do có thể khẳng định chắc chắn: nếu
logic bên trong `create()` chạy, `input` chắc chắn đã hợp lệ — không cần thêm bất kỳ `if` kiểm
tra nào trong thân method.

## Engineering Insight

**Validation ở tầng API (`@Valid` + Bean Validation) có thay thế được validation ở tầng
database (constraint, ví dụ `NOT NULL`/`CHECK`) không?** Không nên. Đây là hai lớp phòng thủ độc
lập, bảo vệ trước các nguồn dữ liệu khác nhau: `@Valid` chỉ bảo vệ đường đi **qua chính REST API
này** — nếu có một đường ghi dữ liệu khác vào cùng bảng (batch job, migration script, một service
khác trong hệ thống ghi trực tiếp vào database dùng chung), `@Valid` hoàn toàn không có tác dụng
gì. Nguyên tắc "defense in depth" (phòng thủ nhiều lớp) khuyến nghị **cả hai** lớp validation
cùng tồn tại: `@Valid` cho trải nghiệm người dùng tốt (phản hồi lỗi rõ ràng, nhanh, đúng field
ngay tại tầng API), và constraint tầng database như lớp bảo vệ cuối cùng không thể bị "lách qua"
dù dữ liệu tới từ đường nào.

## Historical Note

Bean Validation là một đặc tả chuẩn (JSR 303, sau đó JSR 380 cho phiên bản 2.0) thuộc Java EE/
Jakarta EE — không phải sáng tạo riêng của Spring. Hibernate Validator (dù tên có "Hibernate")
là implementation tham chiếu phổ biến nhất, độc lập hoàn toàn với Hibernate ORM (Phase 6) dù
cùng do nhóm phát triển Hibernate duy trì — một điểm dễ gây nhầm lẫn tên gọi phổ biến.

## Myth vs Reality

- **Myth:** "`@RequestBody` tự động validate dữ liệu, không cần thêm gì khác."
  **Reality:** Đã chứng minh bằng thực nghiệm — thiếu `@Valid`, dữ liệu vô lý (tên rỗng, giá âm)
  vẫn được chấp nhận hoàn toàn. `@Valid` là bước bắt buộc phải thêm tường minh.

- **Myth:** "Validate dữ liệu ở tầng API (`@Valid`) là đủ, không cần validate lại ở tầng
  database."
  **Reality:** Xem Engineering Insight — đây là hai lớp phòng thủ độc lập, bảo vệ các đường đi
  dữ liệu khác nhau.

## Common Mistakes

- **Quên `@Valid` trên tham số `@RequestBody`** — annotation validation trên field (`@NotBlank`,
  ...) hoàn toàn vô tác dụng nếu thiếu `@Valid`, đã chứng minh bằng thực nghiệm.
- **Không cung cấp `message` tuỳ chỉnh cho annotation validation** — thông điệp lỗi mặc định của
  Bean Validation thường là tiếng Anh chung chung, không thân thiện với người dùng cuối.
- **Chỉ validate ở tầng API mà bỏ qua constraint tầng database** — mất lớp phòng thủ cuối cùng
  cho các đường ghi dữ liệu khác ngoài API.

## Best Practices

- Luôn thêm `@Valid` cho mọi tham số `@RequestBody` cần đảm bảo tính hợp lệ dữ liệu.
- Khai báo `message` tường minh, rõ ràng bằng ngôn ngữ người dùng cuối hiểu được cho mỗi
  annotation validation.
- Kết hợp validation tầng API với constraint tầng database (Phase 6) theo nguyên tắc phòng thủ
  nhiều lớp.

## Debug Checklist

- [ ] Dữ liệu vô lý vẫn được chấp nhận qua API dù đã khai báo `@NotBlank`/`@Positive` trên
      field? → kiểm tra tham số `@RequestBody` có thiếu `@Valid` không.
- [ ] Response lỗi validation không có thông tin rõ ràng cho từng field? → cần
      `@ExceptionHandler(MethodArgumentNotValidException.class)` (Chapter 11) để định dạng lại
      response, tránh trả về stack trace mặc định.

## Summary

Bean Validation (chuẩn Jakarta, không riêng Spring) cho phép khai báo quy tắc hợp lệ trực tiếp
trên field của DTO (`@NotBlank`, `@Positive`, ...). `@Valid` là công tắc kích hoạt validation đó
tại tầng controller — đã chứng minh bằng thực nghiệm đối chiếu trực tiếp: thiếu `@Valid`, dữ
liệu vô lý (tên rỗng, giá âm 999) được chấp nhận với `201 Created`; có `@Valid`, cùng dữ liệu bị
từ chối với `400 Bad Request` kèm thông tin lỗi chi tiết từng field. Validation thất bại ném
`MethodArgumentNotValidException` **trước khi** thân controller method được gọi, đảm bảo logic
nghiệp vụ luôn làm việc với dữ liệu đã hợp lệ. Nên kết hợp với constraint tầng database cho
phòng thủ nhiều lớp.

## Interview Questions

**Mid**

- `@Valid` hoạt động như thế nào trong Spring MVC? Điều gì xảy ra nếu quên thêm nó vào tham số
  `@RequestBody`?
- Validate dữ liệu ở tầng API có thay thế được validate ở tầng database không? Vì sao?

## Exercises

- [ ] Chạy lại ví dụ trên (có và không có `@Valid`), xác nhận sự khác biệt đúng như mô tả.
- [ ] Thêm annotation `@Size(min = 3, max = 50)` cho field `name`, thử gửi tên quá ngắn/quá dài,
      quan sát thông điệp lỗi.
- [ ] Tạo một class validation tuỳ chỉnh (`@Constraint`) kiểm tra một quy tắc nghiệp vụ phức tạp
      hơn (ví dụ giá phải là bội số của 1000).

## Cheat Sheet

| Annotation | Điều kiện hợp lệ |
| --- | --- |
| `@NotNull` | Không được `null` |
| `@NotBlank` | Không `null`, không rỗng, không chỉ toàn khoảng trắng (chỉ cho `String`) |
| `@NotEmpty` | Không `null`, không rỗng (cho `String`/`Collection`/`Map`/mảng) |
| `@Positive` / `@PositiveOrZero` | Số dương / số dương hoặc 0 |
| `@Size(min=, max=)` | Độ dài/kích thước trong khoảng |
| `@Email` | Đúng định dạng email |
| `@Valid` | Kích hoạt validation trên object (đặt ở tham số method) |

## References

- Jakarta Bean Validation Specification — https://beanvalidation.org/
- Spring Framework Documentation — Validation: https://docs.spring.io/spring-framework/reference/core/validation/beanvalidation.html
