# Production Playbook

> Theo đề xuất bổ sung ở cuối `spec/part-5-knowledge-graph.md`, vai trò làm rõ ở §21: tập
> hợp các tình huống production thực tế, mỗi tình huống dẫn người đọc tới các chapter liên
> quan — dùng như một "sổ tay xử lý sự cố" độc lập với thứ tự đọc tuần tự của sách.
>
> Mỗi mục nên theo cấu trúc ở `spec/part-3-book-wide-standard.md` §13:
> Bối cảnh → Triệu chứng → Nguyên nhân gốc → Điều tra → Khắc phục → Phòng tránh.

## API chậm

TODO — xem hàng "API chậm ở production" trong [cross-reference-index.md](cross-reference-index.md)

## Memory Leak

TODO — xem [Garbage Collection](phase-01-foundation/07-garbage-collection.md)

## OutOfMemoryError

TODO — xem [Garbage Collection](phase-01-foundation/07-garbage-collection.md), [Runtime Data Areas](phase-01-foundation/04-runtime-data-areas.md)

## Deadlock

TODO — xem [Isolation](phase-07-database/06-isolation.md), [Lock (Database)](phase-07-database/07-lock.md)

## Duplicate Record

TODO — xem [Optimistic Lock](phase-06-persistence/13-optimistic-lock.md), [Pessimistic Lock](phase-06-persistence/14-pessimistic-lock.md)

## Connection Pool Exhausted

TODO — xem [HikariCP](phase-08-production/04-hikaricp.md)

## Cache Stale

TODO — xem [Cache](phase-05-spring/16-cache.md)

## N+1 Query

TODO — xem [N+1](phase-06-persistence/10-n-plus-1.md)

## Transaction không rollback

TODO — xem [Transaction (Spring)](phase-05-spring/13-transaction.md)

## Scheduled job chạy trùng lặp khi scale-out

TODO — xem [Scheduler](phase-05-spring/15-scheduler.md)

## Slow external API kéo chậm cả hệ thống

TODO — xem [Resilience Patterns](phase-09-architecture/07-resilience-patterns.md)
