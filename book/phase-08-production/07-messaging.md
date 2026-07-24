---
tags:
  - JMS
  - Messaging
  - Queue vs Topic
---

# Messaging (JMS, RabbitMQ, Spring Integration)

> Phase: Phase 8 — Production
> Chapter slug: `messaging`

## Metadata

```yaml
Chapter: Messaging (JMS, RabbitMQ, Spring Integration)
Phase: Phase 8 — Production
Difficulty: ★★★
Importance: ★★★
Interview Frequency: 40%
Prerequisites:
  - Phase 8, Chapter 06 — Kafka
Used Later:
  - Chapter 08 — Docker
Estimated Reading: 12 phút
Estimated Practice: 10 phút
```

## Story

Cấu hình một `JmsTemplate` tuỳ chỉnh để gửi vào Topic:

```java
@Bean
public JmsTemplate topicJmsTemplate(ConnectionFactory connectionFactory) {
    JmsTemplate template = new JmsTemplate(connectionFactory);
    template.setPubSubDomain(true);
    return template;
}
```

`JmsDemoController` inject **hai** `JmsTemplate` — một để gửi Queue (`jmsTemplate`, dùng bean mặc
định của Spring Boot), một để gửi Topic (`topicJmsTemplate`). Gửi thử tới Queue:

```bash
curl "http://localhost:8080/api/jms-demo/send-queue?message=queue-msg-1"
```

Kết quả thật:

```json
{"timestamp":"...","status":500,"error":"Internal Server Error","path":"/api/jms-demo/send-queue"}
```

```
jakarta.jms.InvalidDestinationException: Destination demo.queue exists, but does not support MULTICAST routing
```

Kiểm tra `/actuator/beans` — chỉ có **một** bean kiểu `JmsTemplate` tồn tại: `topicJmsTemplate`.
Bean `jmsTemplate` mặc định của Spring Boot **chưa từng được tạo**. Nguyên nhân: cấu hình
`JmsTemplate` tự viết (kiểu `JmsTemplate`, bất kể đặt tên gì) đã thoả mãn điều kiện
`@ConditionalOnMissingBean(JmsTemplate.class)` mà Spring Boot dùng để quyết định có tự tạo bean
`jmsTemplate` mặc định hay không — Spring Boot thấy **đã có** một bean kiểu `JmsTemplate` (dù tên
khác) nên bỏ qua việc tạo bean mặc định. Hệ quả: tham số constructor tên `jmsTemplate` (không khớp
tên bean nào) rơi về match theo kiểu, và vì chỉ còn đúng một bean `JmsTemplate` tồn tại
(`topicJmsTemplate`), **cả hai tham số constructor cùng nhận về đúng một bean đó** — request gửi
"tới Queue" thực chất đang dùng template có `pubSubDomain=true`, gây lỗi.

## Interview Question (Central)

> Sự khác biệt cốt lõi giữa mô hình Queue (point-to-point) và Topic (publish-subscribe) trong JMS
> là gì? Vì sao tự định nghĩa một bean `JmsTemplate` tuỳ chỉnh trong Spring Boot đôi khi làm hỏng
> hành vi gửi message ở nơi khác trong cùng ứng dụng?

## Objectives

- [ ] Tự tay chứng minh bằng thực nghiệm sự khác biệt giữa Queue (point-to-point, nhiều consumer
      cạnh tranh) và Topic (publish-subscribe, mọi subscriber đều nhận)
- [ ] Hiểu đúng cơ chế `@ConditionalOnMissingBean` của Spring Boot và cách nó có thể vô tình vô
      hiệu hoá bean autoconfigure mặc định khi tự định nghĩa bean cùng kiểu
- [ ] So sánh mô hình phân phối message của JMS Queue/Topic với mô hình consumer group của Kafka
      (Chapter 06)

## Prerequisites

- Phase 8, Chapter 06 — mô hình consumer group của Kafka, dùng làm nền so sánh trực tiếp.

## Used Later

- **Chapter 08 (Docker)** — đóng gói broker message (Artemis/RabbitMQ) thành container trong một
  hệ thống triển khai thực tế.

## Problem

Có hai nhu cầu phân phối message hoàn toàn khác nhau trong thực tế: (1) một tác vụ (ví dụ "gửi
email xác nhận đơn hàng") chỉ cần được xử lý **đúng một lần**, bởi **một** worker bất kỳ trong số
nhiều worker đang chạy song song; (2) một sự kiện (ví dụ "đơn hàng đã được tạo") cần được **mọi**
thành phần quan tâm trong hệ thống biết tới (cả service gửi email, cả service cập nhật kho, cả
service ghi log audit). Dùng sai mô hình cho một trong hai nhu cầu này dẫn tới hậu quả nghiêm
trọng: hoặc một tác vụ bị xử lý nhiều lần không cần thiết, hoặc một sự kiện chỉ tới được một trong
nhiều bên cần biết.

## Concept

JMS (Jakarta Messaging) định nghĩa hai mô hình phân phối tách biệt:

- **Queue (point-to-point)** — mỗi message chỉ được **một** consumer nhận, dù có bao nhiêu
  consumer cùng lắng nghe trên queue đó (các consumer **cạnh tranh** với nhau).
- **Topic (publish-subscribe)** — mỗi message được gửi tới **tất cả** subscriber đang lắng nghe
  tại thời điểm gửi, mỗi subscriber nhận được **bản sao riêng** của message.

## Why?

Vì hai nhu cầu ở Problem về bản chất khác nhau — "chia việc" (mỗi việc xử lý một lần) đối lập
hoàn toàn với "thông báo sự kiện" (mọi bên quan tâm đều cần biết) — JMS tách hẳn thành hai mô hình
API riêng biệt (`Queue`/`Topic`) thay vì gộp chung, để tránh nhầm lẫn ngay từ lúc thiết kế: một
consumer khai báo lắng nghe Queue biết chắc nó đang **chia sẻ tải** với các consumer khác cùng
queue; một subscriber khai báo lắng nghe Topic biết chắc nó sẽ nhận **mọi** message, không phải
chỉ một phần.

## How?

```java
@Configuration
@EnableJms
public class JmsConfig {
    @Bean
    public JmsTemplate queueJmsTemplate(ConnectionFactory cf) {
        JmsTemplate t = new JmsTemplate(cf);
        t.setPubSubDomain(false); // Queue
        return t;
    }

    @Bean
    public JmsTemplate topicJmsTemplate(ConnectionFactory cf) {
        JmsTemplate t = new JmsTemplate(cf);
        t.setPubSubDomain(true); // Topic
        return t;
    }

    @Bean
    public DefaultJmsListenerContainerFactory topicListenerFactory(ConnectionFactory cf) {
        DefaultJmsListenerContainerFactory f = new DefaultJmsListenerContainerFactory();
        f.setConnectionFactory(cf);
        f.setPubSubDomain(true);
        return f;
    }
}
```

```java
@JmsListener(destination = "demo.queue", containerFactory = "jmsListenerContainerFactory")
public void queueConsumerA(String message) { ... }

@JmsListener(destination = "demo.topic", containerFactory = "topicListenerFactory")
public void topicConsumerA(String message) { ... }
```

## Visualization

```
QUEUE "demo.queue" (point-to-point) - 4 message gui vao:

  Producer --> [msg1, msg2, msg3, msg4] --> Queue
                                              │
                        ┌─────────────────────┴─────────────────────┐
                        ▼                                           ▼
                  Consumer A (canh tranh)                    Consumer B (canh tranh)
                  nhan: msg2, msg4 (2/4)                      nhan: msg1, msg3 (2/4)
                  TONG: moi message CHI 1 consumer nhan

TOPIC "demo.topic" (publish-subscribe) - 3 message gui vao:

  Producer --> [msg1, msg2, msg3] --> Topic
                                        │
                    ┌───────────────────┴───────────────────┐
                    ▼                                       ▼
              Subscriber A                            Subscriber B
              nhan: msg1, msg2, msg3 (3/3)             nhan: msg1, msg2, msg3 (3/3)
              TONG: MOI subscriber nhan DU ca 3 message
```

## Example

**Queue — dữ liệu thật, gửi 4 message, hai consumer cùng lắng nghe `demo.queue`:**

```bash
for i in 1 2 3 4; do curl "http://localhost:8080/api/jms-demo/send-queue?message=queue-msg-$i"; done
curl http://localhost:8080/api/jms-demo/counts
```
```
queueA=2, queueB=2, topicA=0, topicB=0
```

Log thật:

```
[QUEUE-Consumer-B] nhan duoc (lan thu 1): queue-msg-1
[QUEUE-Consumer-A] nhan duoc (lan thu 1): queue-msg-2
[QUEUE-Consumer-B] nhan duoc (lan thu 2): queue-msg-3
[QUEUE-Consumer-A] nhan duoc (lan thu 2): queue-msg-4
```

4 message, chia đều 2/2 cho hai consumer — **không** consumer nào nhận cả 4.

**Topic — dữ liệu thật, gửi 3 message, hai subscriber cùng lắng nghe `demo.topic`:**

```bash
for i in 1 2 3; do curl "http://localhost:8080/api/jms-demo/send-topic?message=topic-msg-$i"; done
curl http://localhost:8080/api/jms-demo/counts
```
```
queueA=2, queueB=2, topicA=3, topicB=3
```

Log thật:

```
[TOPIC-Consumer-A] nhan duoc (lan thu 1): topic-msg-1
[TOPIC-Consumer-B] nhan duoc (lan thu 1): topic-msg-1
[TOPIC-Consumer-A] nhan duoc (lan thu 2): topic-msg-2
[TOPIC-Consumer-B] nhan duoc (lan thu 2): topic-msg-2
[TOPIC-Consumer-A] nhan duoc (lan thu 3): topic-msg-3
[TOPIC-Consumer-B] nhan duoc (lan thu 3): topic-msg-3
```

3 message, **cả hai** subscriber đều nhận đủ cả 3 — hoàn toàn trái ngược với Queue.

**Bug thật khi chỉ có một `JmsTemplate` tự định nghĩa — `/actuator/beans` trước khi sửa:**

```
topicJmsTemplate -> org.springframework.jms.core.JmsTemplate
(KHONG co bean "jmsTemplate" mac dinh nao khac)
```

Sau khi thêm tường minh bean `queueJmsTemplate` (thay vì trông chờ Spring Boot tự tạo
`jmsTemplate`):

```
topicJmsTemplate -> org.springframework.jms.core.JmsTemplate
queueJmsTemplate -> org.springframework.jms.core.JmsTemplate
```

Cả hai bean cùng tồn tại tường minh — request gửi Queue hoạt động đúng trở lại.

## Deep Dive

**Vì sao chỉ định nghĩa MỘT bean `JmsTemplate` tuỳ chỉnh lại xoá mất bean `jmsTemplate` mặc định
của toàn bộ ứng dụng, thay vì chỉ đơn giản "thêm một bean nữa"?** `JmsAutoConfiguration` của Spring
Boot đăng ký bean `jmsTemplate` mặc định với điều kiện `@ConditionalOnMissingBean(JmsTemplate.class)`
— điều kiện này kiểm tra theo **kiểu** (`JmsTemplate.class`), không phải theo tên bean. Khi ứng
dụng tự định nghĩa bất kỳ bean nào kiểu `JmsTemplate` (như `topicJmsTemplate` ở Story) trước khi
autoconfiguration chạy, Spring Boot thấy điều kiện "đã tồn tại ít nhất một bean kiểu này" được thoả
mãn và **bỏ qua hoàn toàn** việc tạo bean mặc định — bất kể tên bean tự định nghĩa là gì. Đây là
hành vi **có chủ đích** của `@ConditionalOnMissingBean` (tôn trọng cấu hình tuỳ chỉnh của người
dùng thay vì áp đặt), nhưng dễ gây bất ngờ khi lập trình viên nghĩ rằng mình chỉ đang "thêm một
template phụ cho Topic" mà không nhận ra nó đã âm thầm loại bỏ template mặc định dùng cho Queue ở
những nơi khác trong ứng dụng.

## Engineering Insight

**Bài học tổng quát nào rút ra được từ bug này, áp dụng cho mọi loại `@ConditionalOnMissingBean`
trong Spring Boot, không riêng gì JMS?** Bất cứ khi nào định nghĩa một bean tuỳ chỉnh có **cùng
kiểu** với một bean mà Spring Boot autoconfigure sẵn (`RestTemplate`, `ObjectMapper`,
`RedisTemplate` — đã thấy ở Chapter 05, `JmsTemplate`...), cần tự hỏi: "bean mặc định này có đang
được nơi khác trong ứng dụng dựa vào không?" Nếu có, giải pháp an toàn là định nghĩa **tường minh
tất cả** các bean cùng kiểu cần dùng (như đã sửa ở Story: `queueJmsTemplate` + `topicJmsTemplate`),
thay vì để một cấu hình tuỳ chỉnh "vô tình" trở thành bean duy nhất tồn tại. Đây là cùng một lớp
vấn đề với bug `RestTemplate`/tracing đã gặp ở Chapter 03 — Spring Boot's autoconfiguration mang
lại sự tiện lợi "không cấu hình gì cũng chạy", nhưng ẩn dưới đó luôn là các điều kiện
`@ConditionalOnMissingBean` có thể bị vô hiệu hoá một cách không chủ ý.

## Historical Note

JMS (Java Message Service) ra đời từ năm 2001 (JSR 914), là API tiêu chuẩn hoá đầu tiên cho
messaging trong hệ sinh thái Java, được các broker thương mại lớn thời kỳ đó (IBM MQSeries,
TIBCO) triển khai. Mô hình Queue/Topic của JMS ảnh hưởng trực tiếp tới thiết kế của rất nhiều hệ
thống messaging sau này. RabbitMQ (dựa trên chuẩn AMQP, không phải JMS) tổng quát hoá mô hình này
hơn nữa bằng khái niệm "exchange" định tuyến linh hoạt tới nhiều queue. Kafka (Chapter 06) là một
thiết kế khác hẳn — không phân biệt Queue/Topic ở tầng API, mà đạt được **cả hai hành vi** cùng lúc
thông qua khái niệm consumer group (xem Myth vs Reality).

## Myth vs Reality

- **Myth:** "Kafka không có khái niệm Queue hay Topic riêng biệt như JMS, nên nó chỉ hỗ trợ một
  trong hai mô hình phân phối."
  **Reality:** Kafka thực chất đạt được **cả hai hành vi cùng lúc**, chỉ bằng một cơ chế duy nhất
  — consumer group (Chapter 06): nhiều consumer **cùng group** cạnh tranh đọc các partition (giống
  hệt hành vi Queue vừa chứng minh ở đây), trong khi nhiều consumer ở **các group khác nhau** đều
  nhận được toàn bộ message độc lập với nhau (giống hệt hành vi Topic).
- **Myth:** "Thêm một `@Bean` tuỳ chỉnh chỉ đơn thuần bổ sung thêm lựa chọn, không ảnh hưởng gì
  tới các bean autoconfigure khác."
  **Reality:** Đã chứng minh bằng thực nghiệm — một bean tuỳ chỉnh cùng kiểu có thể vô hiệu hoá
  hoàn toàn bean autoconfigure mặc định do cơ chế `@ConditionalOnMissingBean`.

## Common Mistakes

- **Định nghĩa một bean tuỳ chỉnh cùng kiểu với bean autoconfigure mặc định mà không kiểm tra xem
  bean mặc định có đang được dùng ở nơi khác hay không** — như đã tái hiện ở Story.
- **Dùng Queue cho một sự kiện cần nhiều bên cùng biết** — chỉ một consumer nhận được, các bên còn
  lại "mất" sự kiện hoàn toàn mà không có lỗi rõ ràng nào báo hiệu.
- **Dùng Topic cho một tác vụ cần xử lý đúng một lần** — mọi subscriber đều nhận và xử lý, dẫn tới
  tác vụ bị lặp lại (ví dụ gửi email trùng nhiều lần).

## Best Practices

- Khi cần nhiều `JmsTemplate`/bean cùng kiểu với các cấu hình khác nhau (Queue vs Topic), định
  nghĩa **tường minh tất cả**, không dựa vào bean autoconfigure mặc định cho một số nơi và bean tự
  định nghĩa cho nơi khác.
- Chọn Queue khi mục tiêu là **chia tải công việc** giữa nhiều worker; chọn Topic khi mục tiêu là
  **thông báo sự kiện** cho nhiều bên độc lập.
- Sau khi thêm bất kỳ `@Bean` tuỳ chỉnh nào trùng kiểu với thành phần autoconfigure, kiểm tra
  `/actuator/beans` để xác nhận không có bean mặc định nào bị âm thầm biến mất.

## Debug Checklist

- [ ] Message gửi tới một destination nhưng consumer nhận sai loại hành vi (ví dụ tưởng gửi Queue
      nhưng hành xử như Topic, hoặc ngược lại)? → kiểm tra xem có đúng một bean `JmsTemplate` được
      autowire vào đúng chỗ hay không, dùng `/actuator/beans` để xác nhận số lượng và cấu hình
      từng bean.
- [ ] Một chức năng dùng `JmsTemplate`/`RestTemplate`/bean autoconfigure khác đột nhiên lỗi sau khi
      thêm một `@Configuration` mới? → nghi ngờ hàng đầu là `@ConditionalOnMissingBean` bị vô hiệu
      hoá bởi bean tuỳ chỉnh cùng kiểu.
- [ ] Một sự kiện chỉ tới được một trong nhiều service cần nhận? → kiểm tra destination đó có phải
      Queue (point-to-point) trong khi đáng lẽ cần Topic (publish-subscribe).

## Summary

JMS phân biệt rõ hai mô hình phân phối: Queue (point-to-point, nhiều consumer cạnh tranh, mỗi
message chỉ tới một consumer) và Topic (publish-subscribe, mỗi message tới toàn bộ subscriber) —
đã chứng minh bằng thực nghiệm: 4 message gửi vào Queue chia đều 2/2 cho hai consumer, trong khi 3
message gửi vào Topic được cả hai subscriber nhận đủ. Kafka (Chapter 06) đạt được cả hai hành vi
này cùng lúc chỉ bằng khái niệm consumer group, không cần API tách biệt như JMS. Một bug thật đã
gặp trong quá trình cấu hình: định nghĩa một bean `JmsTemplate` tuỳ chỉnh (cho Topic) đã vô tình vô
hiệu hoá bean `jmsTemplate` mặc định của Spring Boot (do `@ConditionalOnMissingBean` theo kiểu),
khiến cả hai nơi trong ứng dụng cùng dùng nhầm một template — bài học tổng quát cho mọi loại bean
autoconfigure trong Spring Boot.

## Interview Questions

**Mid**

- Sự khác biệt giữa Queue (point-to-point) và Topic (publish-subscribe) trong JMS?
- Kafka có khái niệm Queue/Topic tách biệt như JMS không? Nó đạt được các hành vi tương đương bằng
  cách nào?

**Senior**

- Vì sao tự định nghĩa một bean `JmsTemplate` tuỳ chỉnh có thể vô tình vô hiệu hoá bean
  `jmsTemplate` mặc định của Spring Boot ở những nơi khác trong ứng dụng?
- So sánh mô hình consumer group của Kafka với mô hình Queue/Topic của JMS — trường hợp nào một
  consumer group của Kafka hành xử giống Queue, trường hợp nào giống Topic?

## Exercises

- [ ] Gọi `/actuator/beans` trước và sau khi thêm bean `queueJmsTemplate` tường minh, xác nhận số
      lượng bean kiểu `JmsTemplate` thay đổi từ 1 lên 2.
- [ ] Thêm consumer thứ ba vào `demo.queue`, gửi 6 message, quan sát cách 3 consumer chia tải
      (không nhất thiết chia đều tuyệt đối, do cơ chế cạnh tranh phụ thuộc thời điểm poll).
- [ ] Với `demo.topic`, tắt một trong hai subscriber (dừng container) trước khi gửi message, xác
      nhận subscriber đã tắt **không** nhận lại được message đó sau khi bật lại (khác với consumer
      group `earliest` của Kafka ở Chapter 06).

## Cheat Sheet

| | Queue (point-to-point) | Topic (publish-subscribe) |
| --- | --- | --- |
| Nhiều consumer cùng lắng nghe | Cạnh tranh, mỗi message tới 1 consumer | Mỗi subscriber nhận mọi message |
| Dùng khi nào | Chia tải công việc | Thông báo sự kiện cho nhiều bên |
| Tương đương trong Kafka | Nhiều consumer CÙNG group | Nhiều consumer KHÁC group |

| `@ConditionalOnMissingBean(Type.class)` | Hành vi |
| --- | --- |
| Chưa có bean nào kiểu đó | Spring Boot tự tạo bean mặc định |
| Đã có ít nhất 1 bean kiểu đó (bất kể tên) | Spring Boot bỏ qua, KHÔNG tạo bean mặc định |

## References

- Jakarta Messaging (JMS) Specification — Point-to-Point và Publish-Subscribe Domains.
- Spring Framework Documentation — `JmsTemplate`, `@JmsListener`.
- Spring Boot Documentation — `@ConditionalOnMissingBean`.
