# Cross-Reference Index

> Theo đề xuất bổ sung ở cuối `spec/part-5-knowledge-graph.md`, vai trò được làm rõ ở §21:
> mục lục tra cứu nhanh cấp sách "Muốn hiểu X thì đọc chapter nào". Không thay thế
> Interview Mapping / Production Mapping chi tiết hơn trong từng chapter.

| Muốn hiểu | Đọc các chapter |
| --- | --- |
| Vì sao `@Transactional` không rollback? | [Transaction (Spring)](phase-05-spring/13-transaction.md), [AOP](phase-05-spring/12-aop.md), [Exception](phase-01-foundation/15-exception.md) |
| Vì sao HashMap mất dữ liệu / trả sai kết quả? | [HashMap](phase-02-collections/08-hashmap.md), [equals()](phase-01-foundation/12-equals.md), [hashCode()](phase-01-foundation/13-hashcode.md), [ConcurrentHashMap](phase-02-collections/11-concurrenthashmap.md) |
| API chậm ở production | [Index](phase-07-database/03-index.md), [Execution Plan](phase-07-database/04-execution-plan.md), [N+1](phase-06-persistence/10-n-plus-1.md), [Cache](phase-05-spring/16-cache.md), [HikariCP](phase-08-production/04-hikaricp.md), [Performance Tuning](phase-08-production/12-performance-tuning.md) |
| Duplicate record dù đã check tồn tại | [Isolation](phase-07-database/06-isolation.md), [Lock (Database)](phase-07-database/07-lock.md), [Optimistic Lock](phase-06-persistence/13-optimistic-lock.md), [Pessimistic Lock](phase-06-persistence/14-pessimistic-lock.md) |
| Connection Pool Exhausted | [HikariCP](phase-08-production/04-hikaricp.md), [Transaction (JPA)](phase-06-persistence/12-transaction.md) |
| Cache trả dữ liệu cũ (stale) | [Cache](phase-05-spring/16-cache.md), [Caching (Architecture)](phase-09-architecture/08-caching.md), [Redis](phase-08-production/05-redis.md) |
| Scheduled job chạy nhiều lần khi scale nhiều instance | [Scheduler](phase-05-spring/15-scheduler.md), [Microservices](phase-09-architecture/10-microservices.md) |
| OutOfMemoryError | [Garbage Collection](phase-01-foundation/07-garbage-collection.md), [Runtime Data Areas](phase-01-foundation/04-runtime-data-areas.md), [JVM Tuning](phase-08-production/11-jvm-tuning.md) |
| Deadlock | [Isolation](phase-07-database/06-isolation.md), [Lock (Database)](phase-07-database/07-lock.md), [Lock (Java)](phase-03-concurrency/09-lock.md) |
| Một API chậm bất ngờ kéo chậm cả hệ thống | [Resilience Patterns](phase-09-architecture/07-resilience-patterns.md) |
| Kết nối nhiều database trong 1 Spring Boot app | [Multi-Datasource Configuration](phase-06-persistence/04-multi-datasource.md) |

---

> **TODO:** bổ sung dần khi viết nội dung từng chapter — mỗi hàng nên tham chiếu tối thiểu
> 3 chapter theo tinh thần "Interview Mapping" ở `spec/part-5-knowledge-graph.md` §6.
