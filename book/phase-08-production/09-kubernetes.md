---
tags:
  - Kubernetes
  - Liveness Probe
  - Readiness Probe
---

# Kubernetes: Deployment, Service, và Health Probes

> Phase: Phase 8 — Production
> Chapter slug: `kubernetes`

## Metadata

```yaml
Chapter: Kubernetes
Phase: Phase 8 — Production
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 60%
Prerequisites:
  - Phase 8, Chapter 08 — Docker
Used Later:
  - Chapter 10 — CI/CD
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

Deploy 3 replica của image `demo-app:multistage` (Chapter 08) lên một cluster Kubernetes thật
(tạo bằng `kind`, chạy cục bộ), với readiness/liveness probe trỏ tới
`/actuator/health/readiness` và `/actuator/health/liveness` — **hai endpoint không hề được cấu
hình tường minh** trong `application.properties` của ứng dụng. Chạy đúng cùng image này bằng
`docker run` thông thường (không phải Kubernetes):

```bash
docker run -d -p 8082:8080 demo-app:multistage
curl http://localhost:8082/actuator/health/readiness
```

```
readiness: 404
liveness: 404
```

Hai endpoint **không tồn tại**. Nhưng deploy chính image đó vào cluster Kubernetes thật:

```bash
kubectl apply -f deployment.yaml
kubectl get pods
```

```
NAME                        READY   STATUS    RESTARTS   AGE
demo-app-689f6bf669-426nx   1/1     Running   0          77s
demo-app-689f6bf669-fxh54   1/1     Running   0          77s
demo-app-689f6bf669-jbc52   1/1     Running   0          77s
```

Cả 3 pod đạt trạng thái `1/1 Running` — readiness/liveness probe **hoạt động bình thường**, dù
không có bất kỳ dòng cấu hình nào được thêm vào ứng dụng giữa hai lần chạy. Kiểm tra biến môi
trường bên trong pod:

```bash
kubectl exec <pod> -- env | grep KUBERNETES_SERVICE_HOST
```

```
KUBERNETES_SERVICE_HOST=10.96.0.1
```

Chính biến môi trường này — **tự động được Kubernetes tiêm vào mọi pod** — là nguyên nhân: Spring
Boot Actuator tự phát hiện đang chạy trong môi trường Kubernetes và tự động bật nhóm health-check
`readiness`/`liveness`, điều **không xảy ra** khi cùng image chạy như một container Docker đơn
thuần.

## Interview Question (Central)

> Readiness probe và liveness probe trong Kubernetes khác nhau ở điểm nào? Vì sao một ứng dụng
> Spring Boot có thể hoạt động bình thường khi test cục bộ bằng Docker nhưng readiness probe của
> nó vẫn "vô hình" (404) cho tới khi thực sự chạy trong Kubernetes?

## Objectives

- [ ] Tự tay deploy một ứng dụng Spring Boot thật lên cluster Kubernetes thật (tạo bằng `kind`),
      xác nhận Deployment/Service/Pod hoạt động đúng bằng dữ liệu thật
- [ ] Chứng minh bằng thực nghiệm cơ chế Spring Boot tự phát hiện môi trường Kubernetes qua biến
      môi trường `KUBERNETES_SERVICE_HOST`, tự động bật health probe groups
- [ ] Đọc hiểu đúng `resources.requests`/`resources.limits` và mối liên hệ trực tiếp với cơ chế
      JVM container-aware đã học ở Chapter 08

## Prerequisites

- Phase 8, Chapter 08 — image Docker `demo-app:multistage` được dùng trực tiếp trong chapter này;
  cơ chế JVM đọc giới hạn bộ nhớ cgroup áp dụng y hệt khi giới hạn đó tới từ
  `resources.limits.memory` của Kubernetes thay vì `docker run --memory`.

## Used Later

- **Chapter 10 (CI/CD)** — pipeline CI/CD thường kết thúc bằng bước `kubectl apply` triển khai
  manifest đã viết ở đây.

## Problem

Chạy một container Docker đơn lẻ (Chapter 08) đủ cho môi trường phát triển, nhưng production cần
nhiều thứ hơn: chạy **nhiều bản sao** (replica) để chịu tải và chịu lỗi, tự động **khởi động lại**
container bị crash, tự động **ngừng gửi traffic** tới một instance chưa sẵn sàng hoặc đang gặp sự
cố, và một địa chỉ **ổn định** để các service khác gọi tới dù các pod phía sau liên tục thay đổi
(restart, scale, rolling update). Đây chính là những vấn đề Kubernetes giải quyết ở tầng
orchestration, phía trên lớp container hoá của Docker.

## Concept

- **Deployment** — khai báo "tôi muốn N bản sao (replica) của Pod này luôn chạy", Kubernetes tự
  động tạo, giám sát, và khởi động lại Pod khi cần để duy trì đúng số lượng.
- **Pod** — đơn vị triển khai nhỏ nhất, chứa một hoặc nhiều container cùng chia sẻ network/storage.
- **Service** — một địa chỉ mạng **ổn định** (không đổi dù Pod phía sau bị thay thế), tự động load
  balance traffic tới các Pod đang khớp label selector.
- **Readiness Probe** — kiểm tra định kỳ "Pod này đã sẵn sàng nhận traffic chưa?"; nếu thất bại,
  Kubernetes **ngừng gửi traffic** tới Pod đó qua Service (Pod vẫn chạy, chỉ bị loại khỏi danh sách
  nhận traffic).
- **Liveness Probe** — kiểm tra định kỳ "Pod này còn sống/hoạt động đúng không?"; nếu thất bại liên
  tục, Kubernetes **khởi động lại** container đó.

## Why?

Vì Pod có thể "còn sống" (tiến trình chưa crash) nhưng **chưa sẵn sàng** xử lý request (ví dụ đang
khởi tạo connection pool, đang warm-up cache) — hoặc ngược lại, "sẵn sàng lúc khởi động" nhưng sau
đó rơi vào trạng thái treo (deadlock, kẹt vòng lặp) mà vẫn chưa crash hẳn — hai khái niệm readiness
và liveness được tách riêng để Kubernetes phản ứng đúng với từng tình huống: readiness thất bại chỉ
cần **tạm ngừng nhận traffic** (không cần khởi động lại gì cả, tự phục hồi khi sẵn sàng trở lại),
trong khi liveness thất bại thật sự cần **khởi động lại** để thoát khỏi trạng thái treo.

## How?

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
spec:
  replicas: 3
  selector:
    matchLabels: { app: demo-app }
  template:
    metadata:
      labels: { app: demo-app }
    spec:
      containers:
        - name: demo-app
          image: demo-app:multistage
          ports: [{ containerPort: 8080 }]
          resources:
            requests: { memory: "512Mi", cpu: "250m" }
            limits: { memory: "512Mi", cpu: "1000m" }
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            initialDelaySeconds: 30
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: demo-app-svc
spec:
  selector: { app: demo-app }
  ports: [{ port: 80, targetPort: 8080 }]
```

```bash
kind create cluster --name demo-cluster
kind load docker-image demo-app:multistage --name demo-cluster
kubectl apply -f deployment.yaml -f service.yaml
```

## Visualization

```
Docker container don thuan (docker run):
  Khong co KUBERNETES_SERVICE_HOST --> Spring Boot KHONG bat probe groups
  /actuator/health/readiness --> 404
  /actuator/health/liveness  --> 404

Pod trong Kubernetes that:
  KUBERNETES_SERVICE_HOST=10.96.0.1 (tu dong duoc tiem vao MOI pod)
                    │
                    ▼
  Spring Boot tu phat hien "dang chay trong Kubernetes"
                    │
                    ▼
  Tu dong bat probe groups
  /actuator/health/readiness --> 200 {"status":"UP"}
  /actuator/health/liveness  --> 200 {"status":"UP"}
```

## Example

**Deploy thật, 3 replica, tất cả đạt trạng thái Ready:**

```bash
kubectl get pods
```
```
NAME                        READY   STATUS    RESTARTS   AGE
demo-app-689f6bf669-426nx   1/1     Running   0          77s
demo-app-689f6bf669-fxh54   1/1     Running   0          77s
demo-app-689f6bf669-jbc52   1/1     Running   0          77s
```

**Sự kiện thật lúc khởi động — readiness probe thất bại NGẮN trước khi ứng dụng kịp lắng nghe
port, đây là hành vi bình thường, không phải lỗi:**

```
Warning  Unhealthy  21s  kubelet  Readiness probe failed: Get "http://10.244.0.6:8080/actuator/health/readiness":
                                  dial tcp 10.244.0.6:8080: connect: connection refused
```

Vài giây sau, Spring Boot khởi động xong, probe bắt đầu thành công, Pod chuyển sang `1/1 Running`
— đúng như thiết kế: `initialDelaySeconds`/`periodSeconds` tồn tại chính để cho phép vài lần thất
bại ban đầu trong lúc ứng dụng khởi động.

**`resources.requests`/`limits` áp dụng thật, xác nhận qua `kubectl describe`:**

```
Limits:
  cpu:     1
  memory:  512Mi
Requests:
  cpu:        250m
  memory:     512Mi
```

**Truy cập qua Service (load balancing thật qua cả 3 Pod):**

```bash
kubectl port-forward svc/demo-app-svc 8090:80
curl http://localhost:8090/api/products
curl http://localhost:8090/actuator/health/readiness
```
```
via Service: 200
{"status":"UP"}
```

## Deep Dive

**Cơ chế chính xác nào khiến Spring Boot tự bật `/actuator/health/readiness` và
`/actuator/health/liveness` chỉ khi chạy trong Kubernetes?** Spring Boot Actuator có logic phát
hiện nền tảng đám mây (`CloudPlatform` detection) — với Kubernetes, nó kiểm tra sự tồn tại của cặp
biến môi trường `KUBERNETES_SERVICE_HOST`/`KUBERNETES_SERVICE_PORT`. Đây là cặp biến môi trường mà
**chính Kubernetes** tự động tiêm vào **mọi** container chạy trong cluster (phục vụ service
discovery kiểu cũ, trước khi DNS-based discovery phổ biến) — không phải thứ ứng dụng hay
Dockerfile tự thêm vào. Khi phát hiện các biến này tồn tại, Spring Boot tự động bật
`management.endpoint.health.probes.enabled=true` một cách ngầm định, expose thêm hai nhóm health
indicator `readiness` và `liveness`. Chạy cùng image bằng `docker run` thông thường không có các
biến môi trường này (vì không có Kubernetes nào tiêm vào), nên hành vi tự động này **không kích
hoạt** — giải thích chính xác sự khác biệt 404 vs 200 đã quan sát ở Story.

## Engineering Insight

**Vì sao readiness probe ban đầu thất bại (`connection refused`) không làm Pod bị đánh dấu lỗi
hay bị khởi động lại, trong khi liveness probe thất bại liên tục lại khiến container bị restart?**
Đây chính là điểm khác biệt cốt lõi giữa hai loại probe: readiness probe **chỉ điều khiển việc có
đưa Pod vào danh sách nhận traffic của Service hay không** — thất bại không kích hoạt bất kỳ hành
động "sửa chữa" nào từ phía Kubernetes, nó chỉ đơn giản chờ và thử lại theo `periodSeconds`. Đây là
lý do `initialDelaySeconds` của readiness probe (10s trong ví dụ) thường đặt **ngắn hơn** liveness
probe (30s) — sẵn sàng nhận traffic sớm ngay khi có thể, nhưng cho phép nhiều thời gian hơn trước
khi kết luận "container này thực sự có vấn đề cần khởi động lại". Nếu đảo ngược hai giá trị này
(liveness kiểm tra sớm hơn readiness), một ứng dụng khởi động chậm (như Spring Boot với nhiều
auto-configuration) có nguy cơ bị Kubernetes **khởi động lại liên tục** ngay trong lúc đang khởi
động bình thường — một lỗi cấu hình thực tế khá phổ biến gọi là "crash loop do liveness probe quá
sớm".

## Historical Note

Khái niệm liveness/readiness probe tách biệt xuất hiện từ phiên bản Kubernetes rất sớm (trước
1.0), phản ánh trực tiếp kinh nghiệm vận hành hệ thống phân tán quy mô lớn của Google (tiền thân
Borg) — nơi phân biệt rõ "tiến trình còn chạy" và "tiến trình sẵn sàng phục vụ" là yêu cầu vận hành
cơ bản, không phải tính năng "thêm sau". Spring Boot Actuator bổ sung hỗ trợ health groups
`readiness`/`liveness` tường minh (thay vì chỉ dùng health indicator tổng hợp chung) từ Spring Boot
2.3 (2020), đúng thời điểm container hoá và Kubernetes trở thành tiêu chuẩn triển khai phổ biến
trong hệ sinh thái Spring.

## Myth vs Reality

- **Myth:** "Chỉ cần thêm `readinessProbe`/`livenessProbe` trỏ đúng path là đủ, không cần quan tâm
  ứng dụng có thực sự expose các endpoint đó hay không trong mọi môi trường."
  **Reality:** Đã chứng minh bằng thực nghiệm — cùng một image, cùng path, nhưng endpoint chỉ tồn
  tại khi chạy thật trong Kubernetes (nhờ tự phát hiện qua biến môi trường), trả về 404 khi chạy
  Docker thông thường. Test "local bằng Docker" không đại diện đầy đủ cho hành vi thật trong
  cluster.
- **Myth:** "`kubectl apply --dry-run=client` luôn chạy được hoàn toàn offline, không cần cluster
  nào cả."
  **Reality:** Đã xác nhận trong quá trình chuẩn bị chapter này — kubectl phiên bản hiện đại (1.33)
  vẫn cần **liên lạc được với một API server** để thực hiện discovery (nhận diện loại resource),
  dù không thực sự tạo resource nào; nếu không có cluster nào khả dụng, `--dry-run=client` cũng báo
  lỗi kết nối, không phải hoàn toàn "client-only" như tên gọi gợi ý.

## Common Mistakes

- **Test readiness/liveness endpoint chỉ bằng `docker run` cục bộ rồi kết luận chúng hoạt động
  đúng** — như đã thấy, hành vi có thể khác hẳn khi thực sự chạy trong Kubernetes.
- **Đặt `initialDelaySeconds` của liveness probe quá ngắn** cho một ứng dụng khởi động chậm — gây
  crash loop do Kubernetes khởi động lại container ngay khi nó còn đang khởi tạo bình thường.
- **Không đặt `resources.limits`** — Pod có thể dùng tài nguyên không giới hạn, ảnh hưởng tới các
  Pod khác cùng node; đồng thời JVM (Chapter 08) sẽ không có cơ sở cgroup rõ ràng để tự tính
  `MaxHeapSize` hợp lý.

## Best Practices

- Luôn đặt `initialDelaySeconds` của liveness probe **lớn hơn** readiness probe, và đủ lớn so với
  thời gian khởi động thực tế đo được của ứng dụng (không đoán mò).
- Kiểm tra hành vi probe **trong môi trường gần giống production nhất có thể** (cluster Kubernetes
  thật, dù chỉ là `kind` cục bộ), không chỉ dựa vào `docker run` đơn thuần.
- Luôn đặt cả `resources.requests` và `resources.limits` — không chỉ để Kubernetes lập lịch đúng,
  mà còn để JVM (Chapter 08) tính `MaxHeapSize` dựa trên giới hạn thực tế thay vì bộ nhớ toàn node.
- Dùng Service (không phải gọi trực tiếp IP của Pod) để giao tiếp giữa các thành phần — địa chỉ ổn
  định dù Pod phía sau liên tục thay đổi qua các lần deploy/scale.

## Debug Checklist

- [ ] Readiness/liveness probe trả về 404 dù ứng dụng chạy bình thường? → kiểm tra có đang test
      ngoài môi trường Kubernetes thật hay không (thiếu biến `KUBERNETES_SERVICE_HOST`); trong
      Kubernetes thật, kiểm tra biến này có được tiêm đúng vào Pod không.
- [ ] Pod liên tục bị restart (crash loop) ngay sau khi deploy? → kiểm tra `initialDelaySeconds`
      của liveness probe có đủ lớn so với thời gian khởi động thực tế của ứng dụng không.
- [ ] Pod chạy (`Running`) nhưng không nhận được traffic qua Service? → kiểm tra trạng thái
      readiness (`READY` cột trong `kubectl get pods`), không chỉ trạng thái `STATUS`.

## Summary

Kubernetes Deployment quản lý số lượng replica mong muốn, Service cung cấp địa chỉ ổn định kèm
load balancing, readiness/liveness probe kiểm soát việc Pod có nhận traffic hay có cần khởi động
lại hay không — đã triển khai thật lên cluster tạo bằng `kind`, xác nhận 3 Pod đạt `1/1 Running`,
Service phân phối traffic đúng. Phát hiện quan trọng nhất từ thực nghiệm: Spring Boot Actuator chỉ
tự động expose `/actuator/health/readiness` và `/actuator/health/liveness` khi phát hiện biến môi
trường `KUBERNETES_SERVICE_HOST` (do chính Kubernetes tự động tiêm vào mọi Pod) — cùng image chạy
bằng `docker run` thông thường trả về 404 ở cả hai endpoint này, một khác biệt hành vi dễ gây bất
ngờ nếu chỉ test cục bộ bằng Docker mà không thử trong môi trường Kubernetes thật.

## Interview Questions

**Mid**

- Sự khác biệt giữa Readiness Probe và Liveness Probe trong Kubernetes?
- Vì sao dùng Service thay vì gọi trực tiếp IP của Pod?

**Senior**

- Một ứng dụng Spring Boot hoạt động bình thường khi test cục bộ bằng Docker, nhưng readiness
  probe trả về 404 khi deploy lên Kubernetes bị đảo ngược lại — hoặc ngược lại, hoạt động đúng
  trong Kubernetes nhưng không hoạt động khi test Docker cục bộ. Giải thích nguyên nhân.
- `initialDelaySeconds` của liveness probe đặt quá ngắn gây ra hậu quả gì? Cách chẩn đoán một crash
  loop do nguyên nhân này?

## Exercises

- [ ] Đặt `livenessProbe.initialDelaySeconds` rất ngắn (ví dụ 2 giây) cho một ứng dụng Spring Boot
      khởi động chậm, quan sát crash loop thật bằng `kubectl get pods -w`.
- [ ] Xoá `resources.limits.memory`, deploy lại, so sánh `MaxHeapSize` JVM đọc được (Chapter 08)
      trước và sau khi có giới hạn tường minh.
- [ ] Scale Deployment lên 5 replica (`kubectl scale deployment demo-app --replicas=5`), xác nhận
      Service tự động phân phối traffic tới toàn bộ 5 Pod mới.

## Cheat Sheet

| | Readiness Probe | Liveness Probe |
| --- | --- | --- |
| Thất bại thì sao? | Loại khỏi danh sách nhận traffic của Service | Khởi động lại container |
| `initialDelaySeconds` khuyến nghị | Ngắn hơn (sẵn sàng nhận traffic sớm) | Dài hơn (tránh restart trong lúc khởi động bình thường) |

| Môi trường chạy | `/actuator/health/readiness` |
| --- | --- |
| `docker run` thông thường | 404 (không tồn tại) |
| Pod trong Kubernetes thật | 200, tự động bật nhờ `KUBERNETES_SERVICE_HOST` |

## References

- Kubernetes Documentation — Liveness, Readiness, and Startup Probes.
- Kubernetes Documentation — Deployments, Services.
- Spring Boot Documentation — Kubernetes Probes (`management.endpoint.health.probes`).
- `kind` (Kubernetes IN Docker) Documentation.
