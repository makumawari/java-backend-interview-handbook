---
tags:
  - Spring
  - REST
  - SpringMVC
---

# Xây dựng REST API với Spring MVC

> Phase: Phase 5 — Spring
> Chapter slug: `rest`

## Metadata

```yaml
Chapter: REST
Phase: Phase 5 — Spring
Difficulty: ★★★
Importance: ★★★★★
Interview Frequency: 90%
Prerequisites:
  - Chapter 08 — Spring Boot
Used Later:
  - Chapter 10 — Validation
  - Chapter 11 — Exception Handler
Estimated Reading: 18 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: [Chapter 08](08-spring-boot.md) — embedded Tomcat đã sẵn sàng phục vụ HTTP. Nhưng
> làm sao một method Java bình thường (`List<Product> findAll()`) lại "trở thành" một endpoint
> HTTP, và giá trị trả về (một `List<Product>`) tự động biến thành JSON mà không cần viết một
> dòng code serialize thủ công nào?

```java
@RestController
@RequestMapping("/api/products")
class ProductController {
    @GetMapping
    List<Product> findAll() { return List.copyOf(store.values()); }
}
```

Gọi thử:

```
$ curl http://localhost:8080/api/products
[{"id":1,"name":"Laptop","price":1200.0}]
```

Một `List<Product>` (kiểu dữ liệu Java thuần) tự động trở thành JSON hợp lệ trong response HTTP
— không có `ObjectMapper.writeValueAsString()` nào được gọi tường minh trong code.

## Interview Question (Central)

> `@RestController` khác `@Controller` như thế nào? Spring MVC chuyển đổi giá trị trả về của
> một method Java thành JSON response bằng cơ chế nào?

## Objectives

- [ ] Dùng thành thạo `@RestController`, `@GetMapping`/`@PostMapping`, `@PathVariable`,
      `@RequestBody`
- [ ] Tự tay chứng minh bằng thực nghiệm luồng request/response đầy đủ: GET danh sách, GET theo
      id, POST tạo mới
- [ ] Hiểu cơ chế `HttpMessageConverter` đứng sau việc tự động chuyển đổi Java object ↔ JSON

## Prerequisites

- Chapter 08 — hiểu Spring Boot đã có embedded server sẵn sàng nhận HTTP request.

## Used Later

- **Chapter 10 (Validation)** — validate dữ liệu đầu vào của request body.
- **Chapter 11 (Exception Handler)** — xử lý lỗi tập trung, trả về response lỗi nhất quán.

## Problem

Xây dựng một REST API theo cách thủ công (không có framework) đòi hỏi tự viết: parse URL để lấy
path variable, parse JSON request body thành object Java, serialize object Java trả về thành
JSON, set đúng HTTP status code và header — tất cả đều là logic **lặp lại giống hệt nhau** giữa
các endpoint khác nhau, chỉ khác phần logic nghiệp vụ thực sự.

## Concept

**Spring MVC** (module xử lý web của Spring Framework, được Spring Boot tự động cấu hình qua
Auto Configuration, Chapter 07) cho phép khai báo một REST endpoint chỉ bằng annotation:
`@RestController` (đánh dấu class xử lý HTTP, mọi method trả về sẽ tự động serialize thành
response body thay vì tên view), `@GetMapping`/`@PostMapping`/... (ánh xạ method Java với một
route HTTP cụ thể), `@PathVariable` (lấy giá trị từ URL), `@RequestBody` (parse JSON request
body thành object Java).

## Why?

Tách "khai báo route" (annotation) khỏi "logic xử lý" (thân method) cho phép Spring MVC tự động
hoá toàn bộ phần lặp lại (parse URL, parse/serialize JSON, set HTTP status) thông qua một cơ chế
tổng quát — `HttpMessageConverter` — chỉ cần khai báo **kiểu dữ liệu** Java mong muốn
(`Product`, `List<Product>`), framework tự tìm converter phù hợp (mặc định là Jackson cho JSON)
để chuyển đổi hai chiều. Lập trình viên chỉ cần tập trung vào logic nghiệp vụ thực sự, không lặp
lại code hạ tầng.

## How?

```java
@RestController // = @Controller + @ResponseBody tren MOI method
@RequestMapping("/api/products")
class ProductController {
    @GetMapping
    List<Product> findAll() { return ...; }

    @GetMapping("/{id}")
    Product findById(@PathVariable Long id) { return ...; } // id lay tu URL

    @PostMapping
    ResponseEntity<Product> create(@RequestBody Product input) { // input tu JSON body
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
}
```

## Visualization

```
HTTP Request                          Spring MVC                        Java method

GET /api/products/1        ──►   DispatcherServlet dinh tuyen  ──►   findById(1L)
                                  (khop @GetMapping("/{id}"))              │
                                  {id} -> chuyen doi String->Long          │
                                                                            ▼
HTTP Response  <──   HttpMessageConverter (Jackson)  <──   return Product("Laptop", 1200.0)
{"id":1,"name":"Laptop","price":1200.0}   serialize Java object -> JSON
```

## Example

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    private final Map<Long, Product> store = new ConcurrentHashMap<>();
    private final AtomicLong idSeq = new AtomicLong(1);

    public ProductController() {
        store.put(1L, new Product(1L, "Laptop", 1200.0));
    }

    @GetMapping
    public List<Product> findAll() { return List.copyOf(store.values()); }

    @GetMapping("/{id}")
    public Product findById(@PathVariable Long id) {
        Product p = store.get(id);
        if (p == null) throw new ProductNotFoundException(id);
        return p;
    }

    @PostMapping
    public ResponseEntity<Product> create(@Valid @RequestBody Product input) {
        long id = idSeq.getAndIncrement();
        Product created = new Product(id, input.name(), input.price());
        store.put(id, created);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
}
```

Kết quả thật (Spring Boot 3.3.4, `curl` gọi trực tiếp ứng dụng đang chạy trên `localhost:8080`):

```
=== GET /api/products ===
[{"id": 1, "name": "Laptop", "price": 1200.0}]

=== GET /api/products/1 ===
{"id": 1, "name": "Laptop", "price": 1200.0}

=== POST /api/products (hop le) ===
{"id":2,"name":"Mouse","price":25.0}
HTTP_STATUS:201
```

`@PathVariable Long id` tự động chuyển `"1"` (chuỗi trong URL) thành `1L` (kiểu `Long`).
`@RequestBody Product input` tự động parse `{"name":"Mouse","price":25}` (JSON) thành một record
`Product` Java. Response trả về status `201 Created` (theo `ResponseEntity.status(...)` khai báo
tường minh trong code) kèm body JSON của sản phẩm vừa tạo, với `id` mới do server tự sinh.

## Deep Dive

**`HttpMessageConverter` hoạt động thế nào để "biết" chuyển `Product` (Java) ↔ JSON?** Khi Spring
MVC cần serialize giá trị trả về (hoặc deserialize `@RequestBody`), nó xem xét header
`Content-Type`/`Accept` của request và duyệt qua danh sách `HttpMessageConverter` đã đăng ký
(được Auto Configuration, Chapter 07, tự động thêm `MappingJackson2HttpMessageConverter` nếu
Jackson có trên classpath — mặc định đúng vậy với `spring-boot-starter-web`), tìm converter đầu
tiên **hỗ trợ** kiểu dữ liệu Java đó và loại `Content-Type` đó. Với JSON, `Jackson`
(`ObjectMapper`) làm việc thực sự: dùng Reflection (Phase 1, Chapter 19) để đọc/ghi field của
record/class Java. Đây là lý do đổi `Content-Type` của request (ví dụ sang `application/xml`)
có thể khiến Spring MVC chọn một converter khác hoàn toàn (nếu có `MappingJackson2XmlHttpMessageConverter` trên classpath) mà không cần sửa bất kỳ dòng code nào trong controller — logic
nghiệp vụ hoàn toàn tách biệt khỏi định dạng truyền tải dữ liệu.

## Engineering Insight

**Vì sao dùng `record` (Phase 4, Chapter 07) làm DTO cho request/response API là lựa chọn tự
nhiên, và nó tương tác với `HttpMessageConverter` như thế nào?** Jackson (từ phiên bản 2.12+, và
đặc biệt hỗ trợ tốt hơn ở các bản mới) nhận diện được các "accessor" đặc biệt của record (`name()`
thay vì `getName()`, đã học ở Phase 4 Chapter 07) để serialize, và dùng **compact constructor**
(hoặc canonical constructor) để deserialize — nghĩa là mọi validation đặt trong compact
constructor của record (ví dụ kiểm tra giá trị âm) sẽ **tự động** áp dụng ngay cả khi dữ liệu tới
từ một JSON request body, không cần thêm bước validate thủ công riêng cho tầng API. Đây là một
minh chứng cụ thể cho việc kiến thức Phase 4 (record) trực tiếp nâng cao chất lượng thiết kế API
ở Phase 5, không phải hai mảng kiến thức tách rời.

## Historical Note

Spring MVC ra đời từ Spring Framework 2.5 (2007) dựa trên mô hình Servlet truyền thống của Java
EE. `@RestController` (gộp `@Controller` + `@ResponseBody`) chỉ xuất hiện từ Spring 4.0 (2013) —
trước đó, mọi method trong một `@Controller` phải tự đánh dấu `@ResponseBody` riêng lẻ nếu muốn
trả JSON thay vì tên view (mô hình MVC truyền thống, trả về HTML template) — sự xuất hiện của
`@RestController` phản ánh REST API dần trở thành mô hình chủ đạo, lấn át MVC trả HTML truyền
thống trong phần lớn ứng dụng backend hiện đại.

## Myth vs Reality

- **Myth:** "`@RestController` là một loại annotation hoàn toàn khác biệt với `@Controller`."
  **Reality:** `@RestController` chỉ đơn giản là `@Controller` + `@ResponseBody` gộp lại — nếu
  bạn tự thêm `@ResponseBody` cho từng method của một `@Controller` thông thường, hành vi hoàn
  toàn tương đương.

- **Myth:** "Spring MVC chỉ hỗ trợ JSON, muốn trả XML hay định dạng khác phải tự viết logic
  chuyển đổi."
  **Reality:** Xem Deep Dive — cơ chế `HttpMessageConverter` là tổng quát, hỗ trợ nhiều định
  dạng nếu converter tương ứng có trên classpath, không cần sửa code controller.

## Common Mistakes

- **Trả về entity JPA trực tiếp làm response API thay vì DTO/record riêng** — vô tình lộ cấu
  trúc database ra ngoài, khó kiểm soát field nào thực sự nên public qua API.
- **Không set đúng HTTP status code, luôn mặc định trả `200 OK` kể cả khi tạo mới thành công**
  (nên là `201 Created`) hoặc khi có lỗi (nên trả `4xx`/`5xx` phù hợp).
- **Dùng `@PathVariable` với kiểu không khớp và không xử lý lỗi chuyển đổi** — dữ liệu URL không
  hợp lệ (ví dụ chữ thay vì số) gây lỗi chuyển đổi kiểu cần được xử lý tường minh (Chapter 11).

## Best Practices

- Dùng DTO/record riêng cho request/response, không trả trực tiếp entity JPA (Phase 6) ra API.
- Trả đúng HTTP status code theo ngữ nghĩa REST chuẩn (`200` đọc thành công, `201` tạo mới,
  `204` xoá thành công không có nội dung trả về, `4xx` lỗi phía client, `5xx` lỗi phía server).
- Tận dụng record (Phase 4, Chapter 07) cho DTO để có validation tích hợp qua compact
  constructor, tương tác tự nhiên với Jackson.

## Debug Checklist

- [ ] Response trả về không đúng định dạng mong đợi (JSON/XML)? → kiểm tra header
      `Content-Type`/`Accept` của request và các `HttpMessageConverter` có trên classpath.
- [ ] `@RequestBody` không parse đúng dữ liệu? → kiểm tra JSON gửi lên có khớp cấu trúc record/
      class Java (tên field, kiểu dữ liệu) không.
- [ ] `@PathVariable` gây lỗi khi giá trị URL không hợp lệ? → cần `@ExceptionHandler` xử lý
      `MethodArgumentTypeMismatchException` (Chapter 11).

## Summary

Spring MVC cho phép khai báo REST endpoint chỉ bằng annotation
(`@RestController`/`@GetMapping`/`@PathVariable`/`@RequestBody`), tự động hoá việc parse URL và
chuyển đổi Java object ↔ JSON qua cơ chế `HttpMessageConverter` (mặc định dùng Jackson). Đã
chứng minh bằng thực nghiệm luồng đầy đủ: GET danh sách trả về JSON tự động, GET theo id chuyển
đổi path variable thành `Long`, POST parse JSON request body thành record Java và trả về đúng
status `201 Created`. `@RestController` chỉ là `@Controller` + `@ResponseBody` gộp lại. Dùng
record (Phase 4) cho DTO tận dụng được validation tích hợp qua compact constructor khi Jackson
deserialize.

## Interview Questions

**Junior**

- `@RestController` khác `@Controller` như thế nào?
- `@PathVariable` và `@RequestBody` dùng để làm gì?

**Mid**

- Spring MVC chuyển đổi giá trị trả về của một method Java thành JSON response bằng cơ chế nào?

**Senior**

- Giải thích cách `HttpMessageConverter` chọn converter phù hợp dựa trên header
  `Content-Type`/`Accept`. Điều gì xảy ra nếu không có converter nào phù hợp?

## Exercises

- [ ] Chạy lại `ProductController` ở trên, dùng `curl` để tự gọi GET/POST và xác nhận kết quả
      đúng như mô tả.
- [ ] Thêm một endpoint `DELETE /api/products/{id}`, trả về status `204 No Content` khi thành
      công.
- [ ] Thử gửi request với header `Content-Type: application/json` nhưng body không phải JSON
      hợp lệ (ví dụ thiếu dấu ngoặc), quan sát lỗi Spring MVC trả về.

## Cheat Sheet

| Annotation | Vai trò |
| --- | --- |
| `@RestController` | `@Controller` + `@ResponseBody`, mọi method trả JSON/data trực tiếp |
| `@RequestMapping` | Ánh xạ tiền tố URL chung cho cả class |
| `@GetMapping`/`@PostMapping`/... | Ánh xạ method với route + HTTP method cụ thể |
| `@PathVariable` | Lấy giá trị từ URL (`/products/{id}`) |
| `@RequestBody` | Parse JSON request body thành object Java |
| `@RequestParam` | Lấy query parameter (`?key=value`) |
| `ResponseEntity<T>` | Kiểm soát tường minh status code + header + body |

## References

- Spring Framework Documentation — Web on Servlet Stack (Spring MVC): https://docs.spring.io/spring-framework/reference/web/webmvc.html
- Jackson Documentation — Java JSON binding.
