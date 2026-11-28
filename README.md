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

모든 목차는 최신순(날짜 내림차순)으로 정렬되어 있습니다.

### 🔥 최근 학습 (최신 7편)

| 날짜 | 분야 | 주제 |
| --- | --- | --- |
| 11.28 | AI·에이전트 | [AI 에이전트 오케스트레이션 — ReAct vs Plan-and-Execute, LangGraph 상태 그래프·Checkpointer·interrupt, Supervisor/Handoff 멀티 에이전트와 컨텍스트 격리·비용 설계](daily-study/2026-11-28-agent-orchestration-react-langgraph-supervisor.md) |
| 11.27 | CS | [가상 메모리와 페이지 교체 알고리즘 — OPT·FIFO·LRU·Clock 비교와 Belady의 이상현상·스래싱·워킹셋, Redis·InnoDB LRU 실무](daily-study/2026-11-27-virtual-memory-and-page-replacement-algorithms.md) |
| 11.26 | 인프라·DevOps | [Kubernetes 무중단 배포와 프로브 — Deployment 롤링 업데이트·Service/Ingress 트래픽 경로부터 Readiness/Liveness/Startup Probe·preStop·Graceful Shutdown까지](daily-study/2026-11-26-kubernetes-zero-downtime-rollout-and-probes.md) |
| 11.25 | 백엔드·DB | [분산 트랜잭션 — 2PC·Saga·Transactional Outbox: 이중 쓰기 문제부터 보상 트랜잭션·발행 원자성·멱등 소비까지](daily-study/2026-11-25-distributed-transactions-saga-and-outbox-pattern.md) |
| 11.24 | CS | [TLS 1.3 핸드셰이크와 QUIC·HTTP/3 — 1-RTT 키 스케줄·0-RTT 재전송 위험부터 Nginx·CloudFront 구성까지](daily-study/2026-11-24-tls13-handshake-and-quic-http3.md) |
| 11.23 | AI·에이전트 | [프롬프트 인젝션과 LLM 가드레일 — Lethal Trifecta·Agents Rule of Two·CaMeL과 백엔드 방어 설계](daily-study/2026-11-23-prompt-injection-and-llm-guardrails.md) |
| 11.22 | CS | [CPU 스케줄링 — FCFS·SJF·SRTF·RR·MLFQ와 평균 대기시간 계산, Linux CFS·EEVDF](daily-study/2026-11-22-cpu-scheduling-algorithms-and-linux-cfs.md) |

### 📅 월별 아카이브

<details>
<summary><b>2026-11 (25편)</b></summary>

| 날짜 | 분야 | 주제 |
| --- | --- | --- |
| 11.28 | AI·에이전트 | [AI 에이전트 오케스트레이션 — ReAct vs Plan-and-Execute, LangGraph 상태 그래프·Checkpointer·interrupt, Supervisor/Handoff 멀티 에이전트와 컨텍스트 격리·비용 설계](daily-study/2026-11-28-agent-orchestration-react-langgraph-supervisor.md) |
| 11.27 | CS | [가상 메모리와 페이지 교체 알고리즘 — OPT·FIFO·LRU·Clock 비교와 Belady의 이상현상·스래싱·워킹셋, Redis·InnoDB LRU 실무](daily-study/2026-11-27-virtual-memory-and-page-replacement-algorithms.md) |
| 11.26 | 인프라·DevOps | [Kubernetes 무중단 배포와 프로브 — Deployment 롤링 업데이트·Service/Ingress 트래픽 경로부터 Readiness/Liveness/Startup Probe·preStop·Graceful Shutdown까지](daily-study/2026-11-26-kubernetes-zero-downtime-rollout-and-probes.md) |
| 11.25 | 백엔드·DB | [분산 트랜잭션 — 2PC·Saga·Transactional Outbox: 이중 쓰기 문제부터 보상 트랜잭션·발행 원자성·멱등 소비까지](daily-study/2026-11-25-distributed-transactions-saga-and-outbox-pattern.md) |
| 11.24 | CS | [TLS 1.3 핸드셰이크와 QUIC·HTTP/3 — 1-RTT 키 스케줄·0-RTT 재전송 위험부터 Nginx·CloudFront 구성까지](daily-study/2026-11-24-tls13-handshake-and-quic-http3.md) |
| 11.23 | AI·에이전트 | [프롬프트 인젝션과 LLM 가드레일 — Lethal Trifecta·Agents Rule of Two·CaMeL과 백엔드 방어 설계](daily-study/2026-11-23-prompt-injection-and-llm-guardrails.md) |
| 11.22 | CS | [CPU 스케줄링 — FCFS·SJF·SRTF·RR·MLFQ와 평균 대기시간 계산, Linux CFS·EEVDF](daily-study/2026-11-22-cpu-scheduling-algorithms-and-linux-cfs.md) |
| 11.21 | 백엔드·DB | [Spring Boot 테스트 전략 — 슬라이스·Testcontainers·@Transactional 함정](daily-study/2026-11-21-spring-boot-test-strategy-and-testcontainers.md) |
| 11.20 | 인프라·DevOps | [Docker 컨테이너 격리 원리와 멀티스테이지 빌드](daily-study/2026-11-20-docker-container-isolation-and-multistage-build.md) |
| 11.19 | 백엔드·DB | [REST API 설계와 멱등성 — Idempotency-Key와 재시도 방어](daily-study/2026-11-19-rest-api-idempotency-and-retry-safety.md) |
| 11.18 | 백엔드·DB | [JVM 메모리 구조와 GC — G1·ZGC, 컨테이너 메모리 산정](daily-study/2026-11-18-jvm-memory-layout-and-gc-g1-zgc.md) |
| 11.17 | AI·에이전트 | [MCP(Model Context Protocol) 구조와 Spring AI 2.0 도구 서버](daily-study/2026-11-17-mcp-model-context-protocol-and-spring-ai-server.md) |
| 11.16 | 인프라·DevOps | [Kafka 메시징과 Consumer Group — 파티셔닝·오프셋·멱등 소비·DLQ](daily-study/2026-11-16-kafka-messaging-and-consumer-groups.md) |
| 11.15 | CS | [Blocking/Non-Blocking vs Sync/Async I/O 모델 (NIO·Netty·WebFlux)](daily-study/2026-11-15-blocking-nonblocking-sync-async-io-models.md) |
| 11.14 | AI·에이전트 | [RAG 아키텍처와 Hybrid Search·Reranking](daily-study/2026-11-14-rag-architecture-and-hybrid-search.md) |
| 11.13 | 백엔드·DB | [SQL 윈도우 함수와 그룹별 집계·순위 최적화](daily-study/2026-11-13-sql-window-functions-and-group-ranking-optimization.md) |
| 11.12 | 백엔드·DB | [Spring AOP 동작 원리와 프록시 (CGLIB vs JDK Dynamic Proxy)](daily-study/2026-11-12-spring-aop-and-proxy-mechanisms.md) |
| 11.11 | 백엔드·DB | [JWT 인증 보안과 Refresh Token Rotation·Redis 무효화](daily-study/2026-11-11-jwt-authentication-and-refresh-token-rotation.md) |
| 11.10 | 백엔드·DB | [Redis 자료구조 활용(ZSET 랭킹)과 캐싱 전략 — Cache Stampede 방어](daily-study/2026-11-10-redis-data-structures-and-caching-strategies.md) |
| 11.09 | 백엔드·DB | [MySQL InnoDB MVCC와 트랜잭션 격리 수준](daily-study/2026-11-09-mysql-innodb-mvcc-and-transaction-isolation-levels.md) |
| 11.08 | 백엔드·DB | [Spring @Transactional 동작 원리와 전파 속성·자가 호출 한계](daily-study/2026-11-08-spring-transactional-aop-propagation-and-self-invocation.md) |
| 11.07 | 백엔드·DB | [MySQL InnoDB B+Tree 인덱스와 커버링 인덱스](daily-study/2026-11-07-mysql-innodb-b-plus-tree-and-covering-index.md) |
| 11.06 | 백엔드·DB | [분산 환경 동시성 제어: 낙관적 락 vs 비관적 락 vs Redisson 분산락](daily-study/2026-11-06-concurrency-control-locks-and-redisson.md) |
| 11.05 | 백엔드·DB | [JPA N+1 문제와 컬렉션 페이징 최적화 (Fetch Join·@EntityGraph·batch size)](daily-study/2026-11-05-jpa-n-plus-one-and-collection-paging.md) |
| 11.04 | 백엔드·DB | [Java 21 가상 스레드와 Carrier Thread 피닝](daily-study/2026-11-04-java21-virtual-threads-and-pinning.md) |

</details>

### 🗂️ 분야별 목차

<details>
<summary><b>백엔드·DB (14편)</b></summary>

| 날짜 | 주제 |
| --- | --- |
| 26.11.25 | [분산 트랜잭션 — 2PC·Saga·Transactional Outbox: 이중 쓰기 문제부터 보상 트랜잭션·발행 원자성·멱등 소비까지](daily-study/2026-11-25-distributed-transactions-saga-and-outbox-pattern.md) |
| 26.11.21 | [Spring Boot 테스트 전략 — 슬라이스·Testcontainers·@Transactional 함정](daily-study/2026-11-21-spring-boot-test-strategy-and-testcontainers.md) |
| 26.11.19 | [REST API 설계와 멱등성 — Idempotency-Key와 재시도 방어](daily-study/2026-11-19-rest-api-idempotency-and-retry-safety.md) |
| 26.11.18 | [JVM 메모리 구조와 GC — G1·ZGC, 컨테이너 메모리 산정](daily-study/2026-11-18-jvm-memory-layout-and-gc-g1-zgc.md) |
| 26.11.13 | [SQL 윈도우 함수와 그룹별 집계·순위 최적화](daily-study/2026-11-13-sql-window-functions-and-group-ranking-optimization.md) |
| 26.11.12 | [Spring AOP 동작 원리와 프록시 (CGLIB vs JDK Dynamic Proxy)](daily-study/2026-11-12-spring-aop-and-proxy-mechanisms.md) |
| 26.11.11 | [JWT 인증 보안과 Refresh Token Rotation·Redis 무효화](daily-study/2026-11-11-jwt-authentication-and-refresh-token-rotation.md) |
| 26.11.10 | [Redis 자료구조 활용(ZSET 랭킹)과 캐싱 전략 — Cache Stampede 방어](daily-study/2026-11-10-redis-data-structures-and-caching-strategies.md) |
| 26.11.09 | [MySQL InnoDB MVCC와 트랜잭션 격리 수준](daily-study/2026-11-09-mysql-innodb-mvcc-and-transaction-isolation-levels.md) |
| 26.11.08 | [Spring @Transactional 동작 원리와 전파 속성·자가 호출 한계](daily-study/2026-11-08-spring-transactional-aop-propagation-and-self-invocation.md) |
| 26.11.07 | [MySQL InnoDB B+Tree 인덱스와 커버링 인덱스](daily-study/2026-11-07-mysql-innodb-b-plus-tree-and-covering-index.md) |
| 26.11.06 | [분산 환경 동시성 제어: 낙관적 락 vs 비관적 락 vs Redisson 분산락](daily-study/2026-11-06-concurrency-control-locks-and-redisson.md) |
| 26.11.05 | [JPA N+1 문제와 컬렉션 페이징 최적화 (Fetch Join·@EntityGraph·batch size)](daily-study/2026-11-05-jpa-n-plus-one-and-collection-paging.md) |
| 26.11.04 | [Java 21 가상 스레드와 Carrier Thread 피닝](daily-study/2026-11-04-java21-virtual-threads-and-pinning.md) |

</details>

<details>
<summary><b>CS (4편)</b></summary>

| 날짜 | 주제 |
| --- | --- |
| 26.11.27 | [가상 메모리와 페이지 교체 알고리즘 — OPT·FIFO·LRU·Clock 비교와 Belady의 이상현상·스래싱·워킹셋, Redis·InnoDB LRU 실무](daily-study/2026-11-27-virtual-memory-and-page-replacement-algorithms.md) |
| 26.11.24 | [TLS 1.3 핸드셰이크와 QUIC·HTTP/3 — 1-RTT 키 스케줄·0-RTT 재전송 위험부터 Nginx·CloudFront 구성까지](daily-study/2026-11-24-tls13-handshake-and-quic-http3.md) |
| 26.11.22 | [CPU 스케줄링 — FCFS·SJF·SRTF·RR·MLFQ와 평균 대기시간 계산, Linux CFS·EEVDF](daily-study/2026-11-22-cpu-scheduling-algorithms-and-linux-cfs.md) |
| 26.11.15 | [Blocking/Non-Blocking vs Sync/Async I/O 모델 (NIO·Netty·WebFlux)](daily-study/2026-11-15-blocking-nonblocking-sync-async-io-models.md) |

</details>

<details>
<summary><b>인프라·DevOps (3편)</b></summary>

| 날짜 | 주제 |
| --- | --- |
| 26.11.26 | [Kubernetes 무중단 배포와 프로브 — Deployment 롤링 업데이트·Service/Ingress 트래픽 경로부터 Readiness/Liveness/Startup Probe·preStop·Graceful Shutdown까지](daily-study/2026-11-26-kubernetes-zero-downtime-rollout-and-probes.md) |
| 26.11.20 | [Docker 컨테이너 격리 원리와 멀티스테이지 빌드](daily-study/2026-11-20-docker-container-isolation-and-multistage-build.md) |
| 26.11.16 | [Kafka 메시징과 Consumer Group — 파티셔닝·오프셋·멱등 소비·DLQ](daily-study/2026-11-16-kafka-messaging-and-consumer-groups.md) |

</details>

<details>
<summary><b>AI·에이전트 (4편)</b></summary>

| 날짜 | 주제 |
| --- | --- |
| 26.11.28 | [AI 에이전트 오케스트레이션 — ReAct vs Plan-and-Execute, LangGraph 상태 그래프·Checkpointer·interrupt, Supervisor/Handoff 멀티 에이전트와 컨텍스트 격리·비용 설계](daily-study/2026-11-28-agent-orchestration-react-langgraph-supervisor.md) |
| 26.11.23 | [프롬프트 인젝션과 LLM 가드레일 — Lethal Trifecta·Agents Rule of Two·CaMeL과 백엔드 방어 설계](daily-study/2026-11-23-prompt-injection-and-llm-guardrails.md) |
| 26.11.17 | [MCP(Model Context Protocol) 구조와 Spring AI 2.0 도구 서버](daily-study/2026-11-17-mcp-model-context-protocol-and-spring-ai-server.md) |
| 26.11.14 | [RAG 아키텍처와 Hybrid Search·Reranking](daily-study/2026-11-14-rag-architecture-and-hybrid-search.md) |

</details>
