---
tags:
  - Kafka
  - Consumer Group
  - Offset
---

# Kafka: Producer, Consumer, và Offset Reset

> Phase: Phase 8 — Production
> Chapter slug: `kafka`

## Metadata

```yaml
Chapter: Kafka
Phase: Phase 8 — Production
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 65%
Prerequisites:
  - Phase 3, Chapter 15 — Thread Pool
  - Phase 8, Chapter 05 — Redis
Used Later:
  - Chapter 07 — Messaging
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

Sản xuất 2 message vào một topic **trước khi** bất kỳ consumer group nào từng tồn tại:

```java
producer.send(new ProducerRecord<>(TOPIC, "key1", "message-1-truoc-khi-co-consumer")).get();
producer.send(new ProducerRecord<>(TOPIC, "key2", "message-2-truoc-khi-co-consumer")).get();
```

Sau đó tạo **hai** consumer group hoàn toàn mới, chỉ khác nhau ở `auto.offset.reset`:

```java
Properties earliestProps = consumerProps("group-earliest-demo", "earliest");
Properties latestProps   = consumerProps("group-latest-demo", "latest");
```

Kết quả thật khi cả hai cùng subscribe rồi `poll()`:

```
[earliest] So message doc duoc: 2
[earliest] nhan duoc: key=key1, offset=0
[earliest] nhan duoc: key=key2, offset=1

[latest]   So message doc duoc TRUOC khi co message moi: 0
```

Cùng một topic, cùng hai message đã tồn tại sẵn — nhưng group `earliest` đọc được **cả hai**,
còn group `latest` đọc được **con số 0**. Sau đó, sản xuất thêm một message thứ ba:

```
[latest] So message doc duoc SAU khi co message moi: 1
[latest] nhan duoc: key=key3, offset=2
```

Group `latest` giờ nhận được đúng **message mới**, không phải hai message cũ. Đây chính là hành
vi nền tảng gây ra vô số bug thực tế dạng "consumer của tôi không nhận được message dù producer đã
gửi thành công".

## Interview Question (Central)

> Một consumer group mới được deploy, kết nối vào một topic đã có sẵn dữ liệu — nhưng không nhận
> được bất kỳ message cũ nào. Producer log xác nhận gửi thành công. Nguyên nhân là gì?

## Objectives

- [ ] Tự tay chứng minh bằng thực nghiệm sự khác biệt giữa `auto.offset.reset=earliest` và
      `latest` với cùng một tập dữ liệu
- [ ] Dùng thành thạo `KafkaTemplate` để gửi message và `@KafkaListener` để nhận message trong
      Spring Boot
- [ ] Hiểu đúng khái niệm offset và consumer group — nền tảng để debug mọi vấn đề "message bị
      thiếu"/"message bị đọc lại" trong hệ thống Kafka thực tế

## Prerequisites

- Phase 3, Chapter 15 — mô hình thread pool, nền tảng để hiểu consumer poll loop chạy trên thread
  riêng, tách biệt với thread xử lý HTTP request.
- Phase 8, Chapter 05 — Redis đã giới thiệu khái niệm hạ tầng chạy bằng Docker container thật.

## Used Later

- **Chapter 07 (Messaging)** — Kafka là một trong các hệ message broker được so sánh với
  JMS/RabbitMQ.

## Problem

Khác với gọi HTTP đồng bộ (request-response), giao tiếp qua message broker là **bất đồng bộ** —
producer gửi message xong không biết (và không cần biết) khi nào, hay liệu có consumer nào đọc nó
hay không. Khi một consumer **mới** tham gia hệ thống (deploy lần đầu, hoặc đổi group ID), nó cần
một quy tắc rõ ràng: bắt đầu đọc từ đâu trong lịch sử message đã có sẵn trên topic?

## Concept

Kafka lưu message trên mỗi partition theo **offset** tăng dần (0, 1, 2, ...) — một con số định vị
tuyệt đối vị trí message trong log của partition đó. Một **consumer group** (định danh bằng
`group.id`) theo dõi vị trí nó đã đọc tới bằng cách **commit offset**. Khi một group **hoàn toàn
mới** (chưa từng commit offset nào) subscribe vào topic, `auto.offset.reset` quyết định điểm bắt
đầu:

- **`earliest`** — bắt đầu từ offset **cũ nhất còn tồn tại** trên topic (đọc lại toàn bộ lịch sử).
- **`latest`** (mặc định) — bắt đầu từ **cuối** topic, chỉ nhận message phát sinh **sau** thời
  điểm subscribe.

## Why?

Vì Kafka không "đẩy" (push) message tới consumer như một hàng đợi truyền thống mà để consumer chủ
động "kéo" (pull) và tự quản lý vị trí đọc, `auto.offset.reset` là cơ chế bắt buộc để trả lời câu
hỏi "consumer hoàn toàn mới nên bắt đầu từ đâu". Giá trị mặc định `latest` phù hợp với các hệ thống
streaming thời gian thực (chỉ quan tâm dữ liệu mới), trong khi `earliest` phù hợp khi cần xử lý lại
toàn bộ lịch sử (ví dụ rebuild một read model, hoặc consumer group phục vụ mục đích audit).

## How?

```java
@RestController
public class KafkaDemoController {
    private final KafkaTemplate<String, String> kafkaTemplate;

    @GetMapping("/api/kafka-demo/send")
    public String send(@RequestParam String message) {
        kafkaTemplate.send("demo-app-topic", message);
        return "sent: " + message;
    }

    @KafkaListener(topics = "demo-app-topic", groupId = "demo-app-group")
    public void listen(ConsumerRecord<String, String> record) {
        log.info("Consumer nhan duoc: partition={}, offset={}, value={}",
            record.partition(), record.offset(), record.value());
    }
}
```

```properties
spring.kafka.consumer.group-id=demo-app-group
spring.kafka.consumer.auto-offset-reset=earliest
```

## Visualization

```
Topic (1 partition), da co san 2 message (offset 0, 1):

  offset:  0                1                2 (chua co)
           [message-1]      [message-2]

  Group "earliest" (MOI, chua tung commit offset):
    subscribe -> bat dau doc tu offset 0 -> nhan CA 2 message cu

  Group "latest" (MOI, chua tung commit offset):
    subscribe -> bat dau doc tu offset 2 (cuoi topic) -> KHONG nhan 2 message cu
    ... san xuat message-3 (offset 2) ...
    poll() -> nhan duoc message-3 (offset 2)
```

## Example

**Roundtrip Spring Boot thật — `KafkaTemplate` gửi, `@KafkaListener` nhận:**

```bash
curl "http://localhost:8080/api/kafka-demo/send?message=hello-tu-controller"
```

Log thật:

```
c.h.demo.logging.KafkaDemoController - Da gui message len topic demo-app-topic: hello-tu-controller
c.h.demo.logging.KafkaDemoController - Consumer nhan duoc message: partition=0, offset=0, value=hello-tu-controller
```

**Bằng chứng offset reset — dữ liệu thật (JUnit test dùng `KafkaProducer`/`KafkaConsumer` thô,
không qua Spring, để kiểm soát chính xác thời điểm mỗi group được tạo):**

```
Da san xuat 2 message TRUOC khi bat ky consumer group nao subscribe

[earliest] So message doc duoc: 2
[earliest] nhan duoc: key=key1, value=message-1-truoc-khi-co-consumer, offset=0
[earliest] nhan duoc: key=key2, value=message-2-truoc-khi-co-consumer, offset=1

[latest] So message doc duoc TRUOC khi co message moi: 0
[latest] So message doc duoc SAU khi co message moi: 1
[latest] nhan duoc: key=key3, value=message-3-SAU-khi-co-consumer, offset=2
```

Group `group-earliest-demo` và `group-latest-demo` cùng subscribe vào **cùng một topic**, cùng
thời điểm tồn tại 2 message có sẵn — nhưng chỉ group `earliest` đọc được chúng. Group `latest`
chỉ bắt đầu nhận dữ liệu **kể từ message được sản xuất sau khi nó đã subscribe** (`offset=2`).

## Deep Dive

**Vì sao `auto.offset.reset` chỉ có tác dụng với group hoàn toàn mới, không ảnh hưởng gì tới một
group đã từng chạy trước đó?** Cấu hình này **chỉ được Kafka tham khảo khi không tìm thấy offset
đã commit** cho cặp (group, partition) đó — tức là lần đầu tiên một `group.id` cụ thể subscribe
vào một partition cụ thể. Nếu group đã từng commit offset (dù chỉ một lần, dù consumer instance cũ
đã dừng từ lâu), lần subscribe tiếp theo của **cùng group.id đó** sẽ tiếp tục từ offset đã commit,
bất kể `auto.offset.reset` được đặt là gì. Đây là lý do một lỗi debug thường gặp: đổi
`auto.offset.reset` từ `latest` sang `earliest` trong config rồi restart consumer, nhưng **không
thấy gì thay đổi** — vì group đó đã có offset commit từ trước, cấu hình mới không được áp dụng.
Cách khắc phục thực tế: đổi sang một `group.id` mới, hoặc dùng công cụ
`kafka-consumer-groups.sh --reset-offsets` để chủ động đặt lại vị trí offset của group hiện tại.

## Engineering Insight

**Vì sao thiết kế Kafka lại yêu cầu consumer tự quản lý offset (pull-based), thay vì broker tự
đẩy message và tự theo dõi trạng thái từng consumer (push-based) như nhiều message queue truyền
thống?** Với mô hình push-based, broker phải tự quyết định tốc độ gửi cho từng consumer (dễ làm
consumer chậm bị quá tải) và phải lưu trạng thái "đã gửi cho ai" cho **từng consumer riêng lẻ**.
Với mô hình pull-based của Kafka, broker chỉ cần lưu **một con trỏ offset duy nhất cho mỗi
partition** trong log tuần tự — cực kỳ đơn giản và có thể lưu trữ vô cùng lâu dài mà không tốn
thêm chi phí theo số lượng consumer. Hệ quả trực tiếp: nhiều consumer group **độc lập** có thể đọc
lại **cùng một dữ liệu lịch sử** theo tốc độ riêng của mình (như demo ở Story, group `earliest`
đọc lại toàn bộ dữ liệu cũ mà không ảnh hưởng gì tới các consumer khác) — điều gần như không thể
làm hiệu quả với hầu hết message queue kiểu push-based truyền thống.

## Myth vs Reality

- **Myth:** "Đổi `auto.offset.reset` và restart consumer là đủ để thay đổi vị trí bắt đầu đọc."
  **Reality:** Đã phân tích ở Deep Dive — cấu hình này **chỉ** áp dụng khi group chưa từng commit
  offset; với group đã tồn tại, cần dùng công cụ reset offset chuyên dụng hoặc đổi `group.id`.
- **Myth:** "Kafka là một hàng đợi (queue) — mỗi message chỉ được xử lý một lần, bởi một consumer
  duy nhất."
  **Reality:** Đã chứng minh ở Engineering Insight — nhiều consumer group độc lập có thể cùng đọc
  toàn bộ lịch sử của cùng một topic, mỗi group theo tốc độ và vị trí offset riêng của mình.

## Common Mistakes

- **Deploy consumer group mới với `auto.offset.reset=latest` (giá trị mặc định) khi thực chất cần
  xử lý dữ liệu lịch sử** — bỏ lỡ toàn bộ message đã tồn tại trước đó, như đã tái hiện ở Story.
- **Đổi `auto.offset.reset` để "sửa" hành vi của một consumer group đang chạy** — không có tác
  dụng vì group đã có offset commit; cần reset offset tường minh hoặc đổi group ID.
- **Không đặt `group.id` khi chỉ muốn test nhanh, dẫn tới Kafka tự sinh group ID ngẫu nhiên mỗi
  lần chạy** — mỗi lần chạy lại được coi là group hoàn toàn mới, dễ nhầm lẫn khi debug vì hành vi
  không nhất quán giữa các lần chạy.

## Best Practices

- Chọn `auto.offset.reset` dựa trên use case: `earliest` cho các luồng xử lý cần đầy đủ lịch sử
  (audit, rebuild read model), `latest` cho các luồng chỉ quan tâm sự kiện thời gian thực.
- Đặt `group.id` tường minh, ổn định qua các lần deploy — tránh để framework tự sinh ngẫu nhiên.
- Khi cần "phát lại" (replay) dữ liệu cho một group đã tồn tại, dùng công cụ reset offset chính
  thức (`kafka-consumer-groups.sh --reset-offsets`) thay vì chỉ đổi config rồi restart.
- Dùng `record.offset()`/`record.partition()` trong log (như `KafkaDemoController`) để luôn có đủ
  thông tin định vị chính xác một message khi cần debug.

## Debug Checklist

- [ ] Consumer group mới không nhận được message cũ dù producer xác nhận gửi thành công? → kiểm
      tra `auto.offset.reset`, mặc định là `latest`.
- [ ] Đổi `auto.offset.reset` nhưng consumer vẫn hành xử như cũ sau restart? → group đã có offset
      commit từ trước — cần reset offset tường minh, không chỉ đổi config.
- [ ] Cần xác định chính xác một message nằm ở vị trí nào để điều tra? → dùng cặp
      (partition, offset) log được ở mỗi message nhận, không chỉ dựa vào nội dung message.

## Summary

Kafka lưu message theo `offset` tuần tự trên mỗi partition; một consumer group hoàn toàn mới quyết
định điểm bắt đầu đọc dựa trên `auto.offset.reset` — `earliest` đọc lại toàn bộ lịch sử,
`latest` (mặc định) chỉ nhận message phát sinh sau thời điểm subscribe. Đã chứng minh bằng thực
nghiệm: với cùng 2 message đã tồn tại sẵn trên topic, group `earliest` đọc được cả hai, group
`latest` đọc được 0 message cho tới khi có message mới phát sinh sau đó. Cấu hình này **chỉ** có
tác dụng với group chưa từng commit offset — một group đã tồn tại sẽ luôn tiếp tục từ offset đã
commit bất kể `auto.offset.reset` được đặt lại là gì, một cạm bẫy debug rất phổ biến trong thực tế.

## Interview Questions

**Mid**

- Sự khác biệt giữa `auto.offset.reset=earliest` và `latest`?
- Offset trong Kafka có ý nghĩa gì? Nó được lưu ở cấp độ nào (topic, partition, hay message)?

**Senior**

- Một consumer group mới không nhận được message cũ dù producer xác nhận gửi thành công. Nguyên
  nhân là gì?
- Vì sao đổi `auto.offset.reset` rồi restart consumer đôi khi không có tác dụng gì? Cách reset
  offset đúng cách cho một group đã tồn tại?

## Exercises

- [ ] Chạy lại kịch bản ở Story, sau đó chạy thêm một lần nữa **cùng group ID** `group-latest-demo`
      — xác nhận nó tiếp tục từ offset đã commit trước đó, không quay lại đọc từ đầu dù topic vẫn
      còn dữ liệu cũ.
- [ ] Dùng `kafka-consumer-groups.sh --describe --group demo-app-group` để xem offset hiện tại và
      lag (số message chưa được đọc) của consumer group trong `KafkaDemoController`.
- [ ] Gửi 5 message liên tiếp, xác nhận `@KafkaListener` nhận đúng thứ tự offset tăng dần.

## Cheat Sheet

| `auto.offset.reset` | Group mới bắt đầu đọc từ đâu? |
| --- | --- |
| `earliest` | Offset cũ nhất còn tồn tại (toàn bộ lịch sử) |
| `latest` (mặc định) | Cuối topic (chỉ message mới, sau thời điểm subscribe) |

| Tình huống | `auto.offset.reset` có tác dụng không? |
| --- | --- |
| Group hoàn toàn mới, chưa từng commit offset | Có |
| Group đã tồn tại, đã từng commit offset | Không — cần reset offset tường minh |

## References

- Apache Kafka Documentation — Consumer Configs (`auto.offset.reset`, `group.id`).
- Apache Kafka Documentation — Consumer Groups và Offset Management.
- Spring for Apache Kafka Documentation — `KafkaTemplate`, `@KafkaListener`.
