---
tags:
  - Spring
  - Security
---

# Spring Security cơ bản

> Phase: Phase 5 — Spring
> Chapter slug: `security`

## Metadata

```yaml
Chapter: Security
Phase: Phase 5 — Spring
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 70%
Prerequisites:
  - Chapter 12 — AOP
  - Chapter 09 — REST
Used Later:
  - Kiến trúc bảo mật API (Phase 8)
Estimated Reading: 18 phút
Estimated Practice: 15 phút
```

## Story

> Chỉ cần thêm `spring-boot-starter-security` vào `pom.xml` — không viết một dòng cấu hình nào —
> khởi động lại ứng dụng, và **mọi** endpoint trước đó mở tự do bỗng nhiên yêu cầu đăng nhập.

```
Using generated security password: 6d690f55-0319-4095-a579-0fb2e4f048a3

This generated password is for development use only. Your security configuration must be
updated before running your application in production.
```

Gọi thử endpoint không kèm thông tin đăng nhập:

```
$ curl -i http://localhost:8080/api/secure/whoami
HTTP/1.1 401
WWW-Authenticate: Basic realm="Realm"
```

Gọi lại với đúng username `user` và password vừa in trong log:

```
$ curl -i -u "user:6d690f55-0319-4095-a579-0fb2e4f048a3" http://localhost:8080/api/secure/whoami
Xin chao, user! Quyen: []
```

Không có bất kỳ `@Bean` hay class cấu hình nào được viết — toàn bộ hành vi này tới từ đúng cơ
chế Auto Configuration đã học ở Chapter 07.

## Interview Question (Central)

> Chỉ cần thêm dependency `spring-boot-starter-security`, ứng dụng đã có hành vi bảo mật gì mặc
> định? Làm sao để tuỳ chỉnh nó?

## Objectives

- [ ] Hiểu hành vi mặc định khi thêm Spring Security: mọi endpoint yêu cầu xác thực, một user
      tạm với password sinh ngẫu nhiên mỗi lần khởi động
- [ ] Tự tay chứng minh bằng thực nghiệm: request không có credential nhận `401`; đúng credential
      nhận `200`; sai credential vẫn `401`
- [ ] Dùng `SecurityFilterChain` để tuỳ chỉnh endpoint nào cần xác thực, endpoint nào mở tự do

## Prerequisites

- Chapter 12 — hiểu AOP, vì Spring Security dùng Servlet Filter (không hoàn toàn giống AOP proxy
  nhưng cùng triết lý "chặn trước khi vào logic nghiệp vụ").
- Chapter 09 — hiểu REST controller cơ bản, nơi Security áp dụng bảo vệ.

## Used Later

- **Kiến trúc bảo mật API** (Phase 8) — nền tảng cho JWT, OAuth2, và các cơ chế xác thực nâng
  cao hơn trong hệ thống thực tế.

## Problem

Một API không được bảo vệ đồng nghĩa **bất kỳ ai** trên internet đều có thể gọi nó — với các
API chỉ đọc dữ liệu công khai, điều này có thể chấp nhận được, nhưng với API thao tác dữ liệu
nhạy cảm (thông tin cá nhân, giao dịch tài chính), việc quên bảo vệ dù chỉ một endpoint có thể
gây hậu quả nghiêm trọng.

## Concept

**Spring Security** là module bảo mật của hệ sinh thái Spring — khi có mặt trên classpath
(`spring-boot-starter-security`), Auto Configuration (Chapter 07) tự động áp dụng một cấu hình
bảo mật **mặc định an toàn** (secure by default): mọi endpoint yêu cầu xác thực (authentication),
một user tạm thời (`user`) với password **sinh ngẫu nhiên** mỗi lần khởi động (in ra log, chỉ
dùng được tới khi ứng dụng restart). `SecurityFilterChain` là nơi lập trình viên **ghi đè** cấu
hình mặc định này, khai báo tường minh endpoint nào cần xác thực, endpoint nào mở tự do
(`permitAll()`), và cơ chế xác thực nào được dùng (HTTP Basic, form login, JWT, ...).

## Why?

Thiết kế "an toàn theo mặc định" (secure by default) — thay vì "mở theo mặc định, phải tự thêm
bảo mật" — phản ánh nguyên tắc bảo mật cơ bản: **lỗi trong việc "quên bật bảo mật" nguy hiểm hơn
nhiều** so với lỗi "quên tắt bảo mật cho một endpoint công khai" (dễ nhận ra ngay khi test —
endpoint trả về 401 dù đáng lẽ phải công khai — trong khi endpoint bị lộ vì quên bảo mật có thể
không bị phát hiện cho tới khi bị khai thác trong thực tế). Password sinh ngẫu nhiên mỗi lần khởi
động (thay vì một password mặc định cố định như `admin`/`password`) ngăn chặn trực tiếp việc vô
tình để lại một cấu hình bảo mật "giả" (trông như có bảo mật nhưng dùng credential ai cũng biết)
trong production nếu lập trình viên quên cấu hình tường minh.

## How?

```java
@Configuration
class SecurityConfig {
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/products/**").permitAll() // MO TU DO
                .anyRequest().authenticated())                   // con lai PHAI xac thuc
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

## Visualization

```
KHONG co SecurityConfig tuy chinh (mac dinh cua Auto Configuration):

  MOI request ──► yeu cau xac thuc (HTTP Basic)
                       │
              Dung credential (user / password sinh ngau nhien trong log) ──► cho qua
              Sai/thieu credential ──► 401 Unauthorized

CO SecurityConfig tuy chinh:

  Request toi /api/products/** ──► permitAll() ──► KHONG can xac thuc, cho qua NGAY
  Request toi endpoint KHAC     ──► authenticated() ──► van yeu cau xac thuc nhu mac dinh
```

## Example

```java
@RestController
public class SecureController {
    @GetMapping("/api/secure/whoami")
    public String whoami(Authentication auth) {
        return "Xin chao, " + auth.getName() + "! Quyen: " + auth.getAuthorities();
    }
}
```

Log khi khởi động (Spring Boot 3.3.4, chỉ thêm `spring-boot-starter-security`, chưa cấu hình gì):

```
Using generated security password: 6d690f55-0319-4095-a579-0fb2e4f048a3

This generated password is for development use only. Your security configuration must be
updated before running your application in production.
```

Gọi thử với `curl`:

```
$ curl -i http://localhost:8080/api/secure/whoami
HTTP/1.1 401
WWW-Authenticate: Basic realm="Realm"

$ curl -i -u "user:6d690f55-0319-4095-a579-0fb2e4f048a3" http://localhost:8080/api/secure/whoami
Xin chao, user! Quyen: []

$ curl -i -u "user:sai-mat-khau" http://localhost:8080/api/secure/whoami
HTTP/1.1 401
WWW-Authenticate: Basic realm="Realm"
```

Không có credential: `401` kèm header `WWW-Authenticate: Basic realm="Realm"` — chính header này
báo cho client biết "cần xác thực kiểu HTTP Basic". Đúng username/password: `200`, trả về
`"Xin chao, user!"` — `Authentication` được tiêm thẳng vào tham số method, lấy được từ chính
context bảo mật mà Spring Security đã xác thực trước đó. Sai password: vẫn `401`, không phân
biệt "sai username" hay "sai password" (một quyết định bảo mật có chủ đích — không tiết lộ
username nào tồn tại).

## Deep Dive

**Spring Security hoạt động ở tầng nào trong luồng xử lý request — trước hay sau
`DispatcherServlet`/Spring MVC (Chapter 09)?** Spring Security dựa trên chuỗi **Servlet Filter**
(`FilterChainProxy`) — một cơ chế tồn tại **ở tầng Servlet**, **trước khi** request tới được
`DispatcherServlet` (nơi Spring MVC xử lý routing, Chapter 09). Đây là lý do một request bị chặn
bởi Spring Security (ví dụ thiếu credential) **không bao giờ chạm tới** logic controller — kể cả
`@ExceptionHandler` (Chapter 11) cũng không can thiệp được vào các lỗi xảy ra ở tầng Filter này
(đây chính là nguyên nhân của hiện tượng đã gặp ở Chapter 11: một request bị exception xảy ra
trong Servlet Filter, khi forward nội bộ tới `/error`, vẫn phải đi qua lại chính Security Filter
Chain — nếu `/error` không được `permitAll()`, response lỗi cũng bị chặn bởi Security).

## Engineering Insight

**Vì sao đôi khi một exception không liên quan gì tới bảo mật lại trả về `403 Forbidden` thay vì
mã lỗi mong đợi?** Đây là một cạm bẫy thực tế phổ biến: khi một exception không được xử lý xảy ra
trong luồng xử lý request, Spring Boot forward nội bộ request đó tới `/error` để tạo response lỗi
mặc định (Chapter 11) — nhưng vì `/error` là một **request HTTP mới** (dù chỉ nội bộ), nó
**cũng phải đi qua Security Filter Chain** như bất kỳ request nào khác. Nếu cấu hình
`SecurityFilterChain` chỉ `permitAll()` cho các endpoint nghiệp vụ mà quên `/error`, response lỗi
nội bộ đó bị chính Security chặn lại, trả về `403` — che giấu hoàn toàn nguyên nhân lỗi gốc, gây
nhầm lẫn nghiêm trọng khi debug (nhìn thấy `403` dễ khiến người debug nghĩ sai hướng, tưởng đây là
vấn đề phân quyền, trong khi nguyên nhân thật là một exception hoàn toàn khác ở tầng logic nghiệp
vụ). Luôn nhớ thêm `/error` vào danh sách `permitAll()` khi tuỳ chỉnh `SecurityFilterChain`.

## Historical Note

Spring Security (ban đầu tên "Acegi Security") ra đời độc lập với Spring Framework, được tích
hợp chính thức vào hệ sinh thái Spring từ năm 2004. Cơ chế "generated security password" (mật
khẩu sinh ngẫu nhiên mỗi lần khởi động khi chưa cấu hình `UserDetailsService` tường minh) là một
cải tiến của Spring Boot (không phải Spring Security thuần) — thiết kế này khuyến khích lập trình
viên **luôn** phải chủ động cấu hình bảo mật thật trước khi triển khai production, thay vì vô
tình để lại một credential mặc định dễ đoán.

## Myth vs Reality

- **Myth:** "Không thêm `spring-boot-starter-security` là ứng dụng không có bảo mật gì cả, cần
  tự viết mọi thứ từ đầu."
  **Reality:** Ngược lại — chỉ cần thêm dependency, Auto Configuration (Chapter 07) đã tự động
  bảo vệ **mọi** endpoint theo mặc định, đã chứng minh bằng thực nghiệm.

- **Myth:** "`403 Forbidden` luôn có nghĩa là vấn đề phân quyền."
  **Reality:** Xem Engineering Insight — đôi khi nó là hệ quả của một exception hoàn toàn không
  liên quan, bị Security chặn khi forward nội bộ tới `/error`.

## Common Mistakes

- **Quên thêm `/error` vào `permitAll()` khi tuỳ chỉnh `SecurityFilterChain`** — gây `403` gây
  nhầm lẫn cho mọi exception không được xử lý, thay vì response lỗi đúng như Chapter 11 mô tả.
- **Để mật khẩu sinh ngẫu nhiên (chỉ dành cho development) chạy trong môi trường production** —
  vì mật khẩu đổi mỗi lần restart, ứng dụng thực tế không thể vận hành được với cấu hình này,
  nhưng nếu vô tình để `UserDetailsService` tạm thời khác dễ đoán, đó là lỗ hổng nghiêm trọng.
- **Không phân biệt Authentication (bạn là ai) và Authorization (bạn được phép làm gì)** — Spring
  Security xử lý cả hai nhưng là hai khái niệm tách biệt, nhầm lẫn dễ dẫn tới cấu hình sai.

## Best Practices

- Luôn viết `SecurityFilterChain` tường minh cho mọi ứng dụng thực tế, không dựa vào cấu hình
  mặc định (chỉ phù hợp cho phát triển/demo nhanh).
- Thêm `/error` vào danh sách `permitAll()` để tránh response lỗi bị Security che giấu.
- Với API thực tế, cân nhắc cơ chế xác thực phù hợp hơn HTTP Basic cho production (JWT, OAuth2)
  — HTTP Basic gửi credential dưới dạng base64 (không mã hoá) trong mỗi request, chỉ an toàn khi
  kết hợp HTTPS.

## Debug Checklist

- [ ] Nhận `403 Forbidden` không rõ nguyên nhân cho một request không liên quan tới phân quyền?
      → kiểm tra `/error` có nằm trong `permitAll()` không — đây có thể là exception khác bị
      Security che giấu.
- [ ] Cần biết password tạm thời để test local? → tìm dòng log "Using generated security
      password" khi khởi động ứng dụng.
- [ ] Endpoint công khai vẫn yêu cầu xác thực dù đã `permitAll()`? → kiểm tra thứ tự các
      `requestMatchers` — quy tắc cụ thể hơn nên khai báo trước quy tắc tổng quát hơn.

## Summary

Spring Security, khi có trên classpath, tự động bảo vệ **mọi** endpoint theo mặc định (secure by
default) qua Auto Configuration — đã chứng minh bằng thực nghiệm: không credential nhận `401`
kèm `WWW-Authenticate: Basic`; đúng username/password (sinh ngẫu nhiên, in trong log khởi động)
nhận `200`; sai password vẫn `401`, không phân biệt lý do. `SecurityFilterChain` cho phép tuỳ
chỉnh endpoint nào công khai (`permitAll()`), endpoint nào cần xác thực. Security hoạt động ở
tầng Servlet Filter, **trước** Spring MVC (Chapter 09) — dẫn tới cạm bẫy thực tế: exception
không xử lý được forward tới `/error` vẫn phải qua Security Filter Chain, nếu quên `permitAll()`
cho `/error` sẽ trả về `403` gây nhầm lẫn hoàn toàn với nguyên nhân lỗi gốc.

## Interview Questions

**Mid**

- Chỉ cần thêm dependency `spring-boot-starter-security`, ứng dụng đã có hành vi bảo mật gì mặc
  định?
- Authentication và Authorization khác nhau như thế nào?

**Senior**

- Spring Security hoạt động ở tầng nào so với Spring MVC/DispatcherServlet? Điều này ảnh hưởng
  gì tới `@ExceptionHandler` (Chapter 11)?
- Vì sao một exception không liên quan tới bảo mật đôi khi lại trả về `403 Forbidden`?

## Exercises

- [ ] Chạy thử ứng dụng chỉ với `spring-boot-starter-security` (không cấu hình gì thêm), xác
      nhận hành vi mặc định đúng như mô tả.
- [ ] Viết `SecurityFilterChain` tuỳ chỉnh, mở tự do một endpoint cụ thể, giữ nguyên các endpoint
      khác yêu cầu xác thực, xác nhận bằng `curl`.
- [ ] Cố tình gây ra một exception không liên quan tới bảo mật trong một `SecurityFilterChain`
      chưa `permitAll()` cho `/error`, quan sát response `403` gây nhầm lẫn.

## Cheat Sheet

| | Mặc định (chưa cấu hình) | Sau khi tuỳ chỉnh `SecurityFilterChain` |
| --- | --- | --- |
| Endpoint được bảo vệ | Tất cả | Tuỳ khai báo (`permitAll()`/`authenticated()`) |
| Username | `user` | Tuỳ cấu hình `UserDetailsService` |
| Password | Sinh ngẫu nhiên, in log mỗi lần khởi động | Tuỳ cấu hình |
| Cơ chế xác thực | HTTP Basic | Tuỳ chọn (Basic, form login, JWT, OAuth2, ...) |

## References

- Spring Security Documentation — https://docs.spring.io/spring-security/reference/
- Spring Boot Documentation — Security: https://docs.spring.io/spring-boot/reference/web/spring-security.html
