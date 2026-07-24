---
tags:
  - Redis
  - RedisTemplate
  - Serialization
---

# Redis: RedisTemplate và Serialization

> Phase: Phase 8 — Production
> Chapter slug: `redis`

## Metadata

```yaml
Chapter: Redis
Phase: Phase 8 — Production
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 60%
Prerequisites:
  - Phase 5, Chapter 16 — Cache
  - Phase 7, Chapter 10 — NoSQL
Used Later:
  - Chapter 06 — Kafka
  - Chapter 12 — Performance Tuning
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

Lưu một object xuống Redis bằng `RedisTemplate<Object, Object>` **tự động cấu hình mặc định**
của Spring Boot (không tuỳ chỉnh serializer):

```java
redisTemplate.opsForValue().set("order:default:ORD-1", order);
```

Đọc lại đúng qua chính `RedisTemplate` — hoàn toàn bình thường:

```
read back: OrderDto[id=ORD-1, item=Ban phim co, amount=1500000.0]
```

Nhưng thử liệt kê key trực tiếp bằng `redis-cli` — công cụ chuẩn của chính Redis:

```bash
docker exec redis-demo redis-cli --no-raw KEYS '*'
```

Kết quả thật:

```
1) "\xac\xed\x00\x05t\x00\x13order:default:ORD-1"
```

Đây **không phải** key `"order:default:ORD-1"` — nó là một chuỗi byte bắt đầu bằng
`\xac\xed\x00\x05` (chính là **magic header của Java serialization stream**: `0xACED` +
version `0x0005`), theo sau là `t\x00\x13` (tag kiểu String + độ dài 19), rồi mới tới nội dung
thật. Hệ quả: `redis-cli GET "order:default:ORD-1"` (với key literal, không có byte header) trả về
`(nil)` — Redis, một ứng dụng ngôn ngữ-độc-lập, **không thể đọc được key** do chính ứng dụng Java
này ghi, vì `RedisTemplate` mặc định serialize cả **key lẫn value** bằng
`JdkSerializationRedisSerializer`.

## Interview Question (Central)

> `RedisTemplate` và `StringRedisTemplate` khác nhau ở điểm nào? Vì sao dữ liệu Java lưu vào Redis
> qua `RedisTemplate` mặc định lại không đọc được bằng `redis-cli` hay ứng dụng viết bằng ngôn ngữ
> khác?

## Objectives

- [ ] Tự tay chứng minh bằng thực nghiệm: `RedisTemplate<Object, Object>` mặc định serialize key
      bằng JDK serialization, tạo ra key nhị phân không tương thích với công cụ/ngôn ngữ khác
- [ ] Cấu hình đúng `RedisTemplate` với `StringRedisSerializer` (key) và
      `GenericJackson2JsonRedisSerializer` (value) để dữ liệu đọc được bằng mọi client
- [ ] Đọc hiểu cơ chế TTL (Time To Live) của Redis qua dữ liệu thật, bao gồm trạng thái sau khi
      key hết hạn

## Prerequisites

- Phase 5, Chapter 16 — khái niệm cache, `@Cacheable`, vấn đề dữ liệu cũ (stale data).
- Phase 7, Chapter 10 — Redis là một trong các hệ NoSQL đã giới thiệu (key-value store).

## Used Later

- **Chapter 06 (Kafka)** — cùng nhóm hạ tầng "message/data broker" cần Docker container thật.
- **Chapter 12 (Performance Tuning)** — Redis thường là lớp cache đầu tiên được cân nhắc khi tối
  ưu độ trễ đọc dữ liệu.

## Problem

Redis lưu trữ dữ liệu dưới dạng **byte thô** — bản thân Redis không biết và không quan tâm dữ liệu
là String, JSON, hay object Java serialize. Client (Spring Data Redis) phải tự quyết định cách
**chuyển đổi** đối tượng Java thành byte trước khi gửi đi, và ngược lại. Nếu chọn sai cách chuyển
đổi, dữ liệu tuy vẫn hoạt động đúng **trong nội bộ ứng dụng Java đó**, nhưng trở nên vô dụng với
bất kỳ ai/công cụ nào khác cố đọc trực tiếp từ Redis.

## Concept

**`RedisTemplate<K, V>`** là API tổng quát của Spring Data Redis, cho phép tuỳ chỉnh **serializer**
riêng cho key, value, hash key, hash value. Mặc định (không cấu hình gì thêm), Spring Boot dùng
`JdkSerializationRedisSerializer` — tức `ObjectOutputStream` tiêu chuẩn của Java — cho cả key lẫn
value. **`StringRedisTemplate`** là một `RedisTemplate<String, String>` được cấu hình sẵn dùng
`StringRedisSerializer` (UTF-8 thuần) cho mọi thành phần — đây là lựa chọn an toàn khi dữ liệu vốn
đã là String.

## Why?

Vì Redis được dùng phổ biến như một hạ tầng **chia sẻ** giữa nhiều service (có thể viết bằng ngôn
ngữ khác nhau) hoặc được thao tác trực tiếp qua `redis-cli`/công cụ giám sát khi debug, dữ liệu cần
ở dạng **con người và hệ thống khác đọc được** (text/JSON), không phải định dạng binary đặc thù
của một JVM cụ thể. `JdkSerializationRedisSerializer` (mặc định) chỉ Java mới giải mã được, và tệ
hơn nữa — áp dụng cho **cả key** khiến ngay cả việc liệt kê/tìm key bằng `redis-cli KEYS`/`SCAN`
cũng trở nên vô nghĩa với người vận hành.

## How?

```java
@Configuration
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> jsonRedisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

```java
jsonRedisTemplate.opsForValue().set("order:json:ORD-2", order);
```

## Visualization

```
RedisTemplate<Object,Object> MAC DINH (JdkSerializationRedisSerializer ca key va value):

  Java: redisTemplate.opsForValue().set("order:default:ORD-1", order)
                                            │
                                            ▼
  Redis luu key that su: \xac\xed\x00\x05 t \x00\x13 order:default:ORD-1
                          └──── header JDK serialization ────┘  └── noi dung ──┘
  redis-cli GET "order:default:ORD-1"  ->  (nil)   <- KHONG KHOP!

RedisTemplate TUY CHINH (StringRedisSerializer key + GenericJackson2JsonRedisSerializer value):

  Redis luu key that su: order:json:ORD-2                              <- KHOP CHINH XAC
  Redis luu value that su: {"@class":"...OrderDto","id":"ORD-2",...}   <- JSON doc duoc
```

## Example

**Key bị mã hoá JDK serialization — dữ liệu thật:**

```bash
docker exec redis-demo redis-cli --no-raw KEYS '*'
```
```
1) "\xac\xed\x00\x05t\x00\x13order:default:ORD-1"
2) "order:string:ORD-1"
```

Key thứ hai (`order:string:ORD-1`, lưu qua `StringRedisTemplate`) hoàn toàn sạch, đọc được trực
tiếp — vì `StringRedisTemplate` dùng `StringRedisSerializer` cho key.

**Sau khi cấu hình `StringRedisSerializer` (key) + `GenericJackson2JsonRedisSerializer` (value):**

```bash
docker exec redis-demo redis-cli GET order:json:ORD-2
```
```json
{"@class":"com.handbook.demo.logging.RedisDemoController$OrderDto","id":"ORD-2","item":"Man hinh","amount":4200000.0}
```

Key sạch (`order:json:ORD-2`, tìm được bằng `KEYS`/`SCAN` bình thường), value là JSON đọc được bởi
bất kỳ ngôn ngữ/công cụ nào — `GenericJackson2JsonRedisSerializer` tự thêm field `@class` để giữ
thông tin kiểu, phục vụ deserialize đa hình (polymorphic) khi đọc lại bằng chính Spring Data Redis.

**TTL (Time To Live) — dữ liệu thật, set với TTL=3 giây:**

```bash
curl http://localhost:8080/api/redis-demo/ttl-demo   # set voi Duration.ofSeconds(3)
docker exec redis-demo redis-cli TTL session:demo     # ngay sau khi set
sleep 2 && docker exec redis-demo redis-cli TTL session:demo
sleep 2 && docker exec redis-demo redis-cli TTL session:demo
docker exec redis-demo redis-cli GET session:demo
```

```
TTL ngay sau khi set: 3
TTL sau 2 giay:       1
TTL sau 4 giay:       -2    <- key da het han VA bi xoa
GET session:demo:     (nil)
```

`-2` là giá trị đặc biệt của lệnh `TTL` trong Redis, nghĩa là **key không tồn tại** (đã hết hạn và
bị dọn dẹp) — khác với `-1` (key tồn tại nhưng không có TTL nào được đặt).

## Deep Dive

**Vì sao đúng 4 byte đầu `\xac\xed\x00\x05` xuất hiện ở mọi key/value dùng
`JdkSerializationRedisSerializer`, không phải ngẫu nhiên?** Đây chính là **`STREAM_MAGIC`**
(`0xACED`) và **`STREAM_VERSION`** (`0x0005`) — hai giá trị hằng số cố định mở đầu **mọi**
`ObjectOutputStream` trong Java, được định nghĩa trong chính `java.io.ObjectStreamConstants`. Bất
kỳ dữ liệu nào serialize bằng cơ chế Java serialization tiêu chuẩn đều mang "chữ ký" này ở đầu —
đây là cách nhanh nhất để nhận diện một blob dữ liệu có phải Java-serialized hay không, kể cả khi
không có source code, một kỹ năng debug hữu ích khi gặp dữ liệu binary lạ trong bất kỳ hệ thống
lưu trữ nào (không riêng Redis).

## Engineering Insight

**`GenericJackson2JsonRedisSerializer` tự thêm field `@class` vào JSON — đây có phải hành vi mong
muốn trong mọi trường hợp?** Field `@class` (thấy trong Example) cho phép Spring Data Redis
**deserialize đúng kiểu cụ thể** khi đọc lại dữ liệu (quan trọng nếu `RedisTemplate<String, Object>`
lưu nhiều kiểu object khác nhau dưới cùng một cấu trúc key). Nhưng đây cũng là **con dao hai lưỡi**:
(1) nó gắn chặt dữ liệu lưu trong Redis với **tên class Java đầy đủ** (`com.handbook.demo....`) —
đổi tên hoặc di chuyển class sẽ làm hỏng khả năng đọc lại dữ liệu cũ; (2) nó làm dữ liệu **không
còn thuần JSON chuẩn** theo góc nhìn của một service khác (Node.js, Python) đọc cùng key — field
`@class` trở thành nhiễu không cần thiết. Với dữ liệu cần chia sẻ liên ngôn ngữ, nên dùng
`Jackson2JsonRedisSerializer` (chỉ định rõ class cụ thể, không kèm `@class`) thay vì
`GenericJackson2JsonRedisSerializer`.

## Historical Note

`JdkSerializationRedisSerializer` là serializer **mặc định** của Spring Data Redis từ những phiên
bản đầu tiên (kế thừa thói quen serialization "an toàn, không cần cấu hình gì" phổ biến trong hệ
sinh thái Java giai đoạn 2010s, ví dụ session replication, RMI). Khi Redis ngày càng được dùng như
một hạ tầng chia sẻ đa ngôn ngữ/đa service, cộng đồng dần chuyển sang khuyến nghị JSON serializer
làm best practice mặc định trong mọi hướng dẫn cấu hình Spring Data Redis hiện đại — dù bản thân
Spring Boot **chưa** đổi default vì lý do tương thích ngược.

## Myth vs Reality

- **Myth:** "Đọc lại dữ liệu đúng qua chính `RedisTemplate` nghĩa là cấu hình serialize đã đúng."
  **Reality:** Đã chứng minh bằng thực nghiệm — dữ liệu đọc lại đúng **trong nội bộ cùng một JVM**
  dùng cùng serializer, nhưng key/value hoàn toàn không tương thích với `redis-cli` hay bất kỳ
  client/ngôn ngữ nào khác.
- **Myth:** "`StringRedisTemplate` và `RedisTemplate<String, String>` tự tạo là giống hệt nhau."
  **Reality:** Về kiểu generic thì giống, nhưng `StringRedisTemplate` được Spring Boot **cấu hình
  sẵn** `StringRedisSerializer` cho mọi thành phần; tự tạo `RedisTemplate<String, String>` bằng
  `new RedisTemplate<>()` mà không set serializer vẫn rơi vào mặc định JDK serialization.

## Common Mistakes

- **Dùng `RedisTemplate<Object, Object>` mặc định mà không cấu hình serializer** — dữ liệu hoạt
  động "đúng" trong test nội bộ nhưng không đọc được bằng công cụ giám sát Redis hay service khác.
- **Chọn `GenericJackson2JsonRedisSerializer` cho dữ liệu cần chia sẻ với hệ thống không phải
  Java** — field `@class` gắn chặt với tên class Java, gây khó hiểu/không tương thích phía nhận.
- **Không kiểm tra dữ liệu thật trong Redis bằng `redis-cli`** khi debug — chỉ test qua chính
  `RedisTemplate` của ứng dụng có thể che giấu vấn đề serialization vì cùng cấu hình đọc/ghi khớp
  nhau, như đã thấy ở Story.

## Best Practices

- Luôn cấu hình tường minh `keySerializer`/`valueSerializer` cho `RedisTemplate`, ưu tiên
  `StringRedisSerializer` cho key và JSON serializer cho value.
- Dùng `redis-cli` (hoặc RedisInsight) để xác minh trực tiếp dữ liệu thật trong Redis khi debug,
  không chỉ tin vào việc đọc lại qua chính ứng dụng.
- Luôn đặt TTL rõ ràng cho dữ liệu cache (như `Duration.ofSeconds(...)`) — dữ liệu không có TTL
  tồn tại vĩnh viễn, dễ gây phình bộ nhớ Redis theo thời gian.
- Với dữ liệu chia sẻ đa ngôn ngữ/đa service, tránh `GenericJackson2JsonRedisSerializer` (có
  `@class`), cân nhắc `Jackson2JsonRedisSerializer` chỉ định class cụ thể hoặc tự định nghĩa schema
  JSON.

## Debug Checklist

- [ ] Dữ liệu trong Redis không đọc được bằng `redis-cli`/công cụ khác dù ứng dụng Java hoạt động
      bình thường? → nghi ngờ hàng đầu là `RedisTemplate` đang dùng
      `JdkSerializationRedisSerializer` mặc định — kiểm tra 4 byte đầu `\xac\xed\x00\x05`.
- [ ] Key tìm bằng `KEYS`/`SCAN` không khớp với key ứng dụng dùng? → cùng nguyên nhân, key cũng bị
      serialize nhị phân, không phải chuỗi thuần.
- [ ] Dữ liệu cache "sống" quá lâu, không tự xoá? → kiểm tra có set TTL khi ghi hay không; `TTL
      key` trả về `-1` nghĩa là key không có thời gian hết hạn.

## Summary

`RedisTemplate` mặc định (không cấu hình) dùng `JdkSerializationRedisSerializer` cho **cả key lẫn
value** — đã chứng minh bằng thực nghiệm: key lưu thực tế trong Redis mang theo magic header Java
serialization (`\xac\xed\x00\x05`), khiến `redis-cli GET` với key literal trả về `(nil)`, dù đọc
lại đúng qua chính `RedisTemplate` của ứng dụng. Cấu hình tường minh `StringRedisSerializer` (key)
và `GenericJackson2JsonRedisSerializer` (value) tạo ra dữ liệu sạch, đọc được bởi mọi công cụ/ngôn
ngữ — đã xác nhận qua key `order:json:ORD-2` và value JSON đọc được trực tiếp. TTL trong Redis
giảm dần theo thời gian thực (`TTL` trả về số giây còn lại), và trả về `-2` khi key đã hết hạn và
bị xoá, `-1` khi key tồn tại nhưng không có TTL.

## Interview Questions

**Mid**

- `RedisTemplate` và `StringRedisTemplate` khác nhau ở điểm nào?
- TTL của một key trong Redis hoạt động như thế nào? Ý nghĩa của giá trị `-1` và `-2` khi gọi lệnh
  `TTL`?

**Senior**

- Vì sao dữ liệu lưu bằng `RedisTemplate` mặc định không đọc được bằng `redis-cli`? Cách khắc
  phục?
- `GenericJackson2JsonRedisSerializer` và `Jackson2JsonRedisSerializer` khác nhau ở điểm nào? Khi
  nào nên tránh dùng loại có field `@class`?

## Exercises

- [ ] Chạy `redis-cli --no-raw KEYS '*'` trước và sau khi đổi `RedisTemplate` sang dùng
      `StringRedisSerializer`, so sánh trực tiếp định dạng key.
- [ ] Set một key với TTL rất ngắn (1 giây), viết vòng lặp gọi `TTL` mỗi 200ms để quan sát giá trị
      giảm dần tới `-2`.
- [ ] Serialize cùng một object bằng `Jackson2JsonRedisSerializer` (chỉ định class cụ thể) thay vì
      `GenericJackson2JsonRedisSerializer`, xác nhận JSON kết quả không còn field `@class`.

## Cheat Sheet

| Serializer | Key/Value dạng gì trong Redis? | Đọc được bằng `redis-cli`? |
| --- | --- | --- |
| `JdkSerializationRedisSerializer` (mặc định của `RedisTemplate`) | Binary, có magic header `\xac\xed` | Không |
| `StringRedisSerializer` | Text thuần | Có |
| `GenericJackson2JsonRedisSerializer` | JSON kèm `@class` | Có (nhưng có field thừa nếu đọc từ ngôn ngữ khác) |

| Giá trị lệnh `TTL key` | Ý nghĩa |
| --- | --- |
| `> 0` | Số giây còn lại trước khi hết hạn |
| `-1` | Key tồn tại, không có TTL |
| `-2` | Key không tồn tại (đã hết hạn/chưa từng có) |

## References

- Spring Data Redis Documentation — Serializers.
- Redis Documentation — `EXPIRE`, `TTL`.
- `java.io.ObjectStreamConstants` — `STREAM_MAGIC`, `STREAM_VERSION`.
