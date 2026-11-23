# daily-study

매일 하나씩 개발 지식을 정리하는 저장소입니다. Java·Spring 백엔드를 축으로 CS(운영체제·네트워크), 데이터베이스·SQL, 인프라·DevOps, AI·LLM까지 하루 한 편씩 학습 내용을 남깁니다.

## 구조

```
daily-study/
├── README.md
└── daily-study/
    ├── 2026-11-04-java21-virtual-threads-and-pinning.md
    └── ...
```

학습 노트는 `daily-study/` 폴더 아래 `YYYY-MM-DD-주제.md` 형식으로 쌓입니다. 각 노트는 맨 위에 한 줄 핵심 요약을 두고, 본문에서 원리 → 트레이드오프 → 실무 적용 순으로 정리합니다.

## 목차

| 날짜 | 주제 |
| --- | --- |
| 2026-11-04 | [Java 21 가상 스레드와 Carrier Thread 피닝](daily-study/2026-11-04-java21-virtual-threads-and-pinning.md) |
| 2026-11-05 | [JPA N+1 문제와 컬렉션 페이징 최적화 (Fetch Join·@EntityGraph·batch size)](daily-study/2026-11-05-jpa-n-plus-one-and-collection-paging.md) |
| 2026-11-06 | [분산 환경 동시성 제어: 낙관적 락 vs 비관적 락 vs Redisson 분산락](daily-study/2026-11-06-concurrency-control-locks-and-redisson.md) |
| 2026-11-07 | [MySQL InnoDB B+Tree 인덱스와 커버링 인덱스](daily-study/2026-11-07-mysql-innodb-b-plus-tree-and-covering-index.md) |
| 2026-11-08 | [Spring @Transactional 동작 원리와 전파 속성·자가 호출 한계](daily-study/2026-11-08-spring-transactional-aop-propagation-and-self-invocation.md) |
| 2026-11-09 | [MySQL InnoDB MVCC와 트랜잭션 격리 수준](daily-study/2026-11-09-mysql-innodb-mvcc-and-transaction-isolation-levels.md) |
| 2026-11-10 | [Redis 자료구조 활용(ZSET 랭킹)과 캐싱 전략 — Cache Stampede 방어](daily-study/2026-11-10-redis-data-structures-and-caching-strategies.md) |
| 2026-11-11 | [JWT 인증 보안과 Refresh Token Rotation·Redis 무효화](daily-study/2026-11-11-jwt-authentication-and-refresh-token-rotation.md) |
| 2026-11-12 | [Spring AOP 동작 원리와 프록시 (CGLIB vs JDK Dynamic Proxy)](daily-study/2026-11-12-spring-aop-and-proxy-mechanisms.md) |
| 2026-11-13 | [SQL 윈도우 함수와 그룹별 집계·순위 최적화](daily-study/2026-11-13-sql-window-functions-and-group-ranking-optimization.md) |
| 2026-11-14 | [RAG 아키텍처와 Hybrid Search·Reranking](daily-study/2026-11-14-rag-architecture-and-hybrid-search.md) |
| 2026-11-15 | [Blocking/Non-Blocking vs Sync/Async I/O 모델 (NIO·Netty·WebFlux)](daily-study/2026-11-15-blocking-nonblocking-sync-async-io-models.md) |
| 2026-11-16 | [Kafka 메시징과 Consumer Group — 파티셔닝·오프셋·멱등 소비·DLQ](daily-study/2026-11-16-kafka-messaging-and-consumer-groups.md) |
| 2026-11-17 | [MCP(Model Context Protocol) 구조와 Spring AI 2.0 도구 서버](daily-study/2026-11-17-mcp-model-context-protocol-and-spring-ai-server.md) |
| 2026-11-18 | [JVM 메모리 구조와 GC — G1·ZGC, 컨테이너 메모리 산정](daily-study/2026-11-18-jvm-memory-layout-and-gc-g1-zgc.md) |
| 2026-11-19 | [REST API 설계와 멱등성 — Idempotency-Key와 재시도 방어](daily-study/2026-11-19-rest-api-idempotency-and-retry-safety.md) |
| 2026-11-20 | [Docker 컨테이너 격리 원리와 멀티스테이지 빌드](daily-study/2026-11-20-docker-container-isolation-and-multistage-build.md) |
| 2026-11-21 | [Spring Boot 테스트 전략 — 슬라이스·Testcontainers·@Transactional 함정](daily-study/2026-11-21-spring-boot-test-strategy-and-testcontainers.md) |
| 2026-11-22 | [CPU 스케줄링 — FCFS·SJF·SRTF·RR·MLFQ와 평균 대기시간 계산, Linux CFS·EEVDF](daily-study/2026-11-22-cpu-scheduling-algorithms-and-linux-cfs.md) |
| 2026-11-23 | [프롬프트 인젝션과 LLM 가드레일 — Lethal Trifecta·Agents Rule of Two·CaMeL과 백엔드 방어 설계](daily-study/2026-11-23-prompt-injection-and-llm-guardrails.md) |
