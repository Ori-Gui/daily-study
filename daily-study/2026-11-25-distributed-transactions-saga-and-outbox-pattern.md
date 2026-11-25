# [2026-11-25] 분산 트랜잭션 — 2PC·Saga·Transactional Outbox: 이중 쓰기 문제부터 보상 트랜잭션·발행 원자성·멱등 소비까지

> **한 줄 핵심 요약**: 서비스가 둘 이상으로 쪼개지면 `@Transactional` 한 장으로 묶을 수 있는 범위가 **곧 하나의 DB 커넥션 범위**로 줄어들고, 남는 문제는 두 가지입니다 — ① 서로 다른 저장소에 쓰는 **이중 쓰기(Dual Write)** 와 ② 한 서비스의 업무가 실패했을 때 **이미 끝난 다른 서비스의 일을 되돌리는 일**입니다. 전자는 **Transactional Outbox**(업무 행 + 이벤트 행을 같은 로컬 트랜잭션에 커밋)로, 후자는 **Saga**(롤백 대신 보상)로 풉니다. 2PC/XA는 원자성을 주지만 **코디네이터 장애 시 락을 쥔 채 멈추는 블로킹** 때문에 요청 경로에서 쓰지 않고, Saga는 격리성(Isolation)을 포기하는 대신 **의미적 락(Semantic Lock)** 으로 그 구멍을 메웁니다. 어느 쪽을 골라도 전달 보장은 **at-least-once**이므로, **컨슈머의 멱등성(유니크 제약)** 이 최종 방어선입니다.

---

## 1. 개념 및 동작 원리

### 1.1 로컬 트랜잭션이 끝나는 지점

모놀리식에서는 주문·재고·결제가 같은 DB, 같은 커넥션 안에 있어서 `@Transactional` 하나로 원자성이 보장됩니다(11-08 노트). 서비스가 분리되면 다음 세 가지가 동시에 깨집니다.

| 성질 | 로컬 트랜잭션 | 서비스 분리 후 |
| --- | --- | --- |
| 원자성(Atomicity) | 커밋/롤백이 하나 | 각 서비스가 따로 커밋 → 부분 성공 |
| 격리성(Isolation) | DB 격리 수준이 보장(11-09 노트) | 중간 상태가 다른 서비스에 그대로 노출 |
| 일관성(Consistency) | 제약·FK로 강제 | 최종 일관성(Eventual Consistency)으로 완화 |
| 내구성(Durability) | WAL/redo | 동일하지만, **메시지 발행은 durable하지 않을 수 있음** |

- 중요한 건 **"분산 트랜잭션이 필요하다"는 말이 곧 "2PC를 쓰겠다"는 말이 아니라는 점**입니다. 실무의 대부분은 *2PC를 피하는 설계*(Saga + Outbox)로 해결하고, 2PC는 같은 벤더의 DB 두 개를 묶는 좁은 경우(예: 한 조직의 MySQL 2대)에만 제한적으로 씁니다.
- 마이크로서비스에서 **DB를 공유하지 않는다**는 원칙을 지키면 2PC는 애초에 선택지에서 빠집니다. 다른 서비스의 DB에 직접 붙는 순간 그건 분산 모놀리식입니다.

### 1.2 이중 쓰기(Dual Write) 문제 — 실무에서 가장 자주 터지는 지점

```text
[잘못된 코드] DB 저장과 Kafka 발행을 각각 수행
──────────────────────────────────────────────
BEGIN
  INSERT INTO orders (...)          -- 성공
COMMIT                              -- 여기서 프로세스가 죽으면?
kafkaTemplate.send("order-created") -- 실행되지 않음 → 재고 서비스는 영원히 모름
```

```text
[반대 순서도 똑같이 깨진다]
kafkaTemplate.send("order-created") -- 발행 성공
BEGIN
  INSERT INTO orders (...)          -- 실패/롤백
COMMIT                              -- 존재하지 않는 주문에 대한 이벤트가 이미 나감
```

- **두 저장소를 하나의 원자적 연산으로 묶을 방법이 없습니다.** 하나는 커밋됐고 다른 하나는 안 된 상태가 반드시 생기며, 어느 쪽이든 업무가 깨집니다.
- `try { save(); send(); } catch { ... }` 도, `send()` 실패 시 DB를 되돌리는 코드도 답이 아닙니다. **DB는 이미 커밋됐고 되돌릴 수 없습니다.**
- `@TransactionalEventListener(phase = AFTER_COMMIT)`는 실패 창을 *줄일* 뿐 없애지 못합니다. 커밋과 리스너 실행 사이에 죽으면 이벤트가 **조용히 유실**됩니다(2.9절).
- 해법의 방향은 하나입니다. **"발행할 사실"을 업무 데이터와 같은 트랜잭션에 기록하고, 실제 발행은 그 기록을 읽어서 별도로 한다.** 이것이 Transactional Outbox입니다.

### 1.3 2PC/XA — 원자성은 얻고 가용성을 잃는다

2PC(Two-Phase Commit)는 **조정자(Coordinator, TM)** 가 참여자(RM)들에게 두 단계로 지시합니다.

```text
Phase 1: PREPARE   → 각 RM: "커밋 가능한가?" → 락 잡고 redo/undo 기록 후 투표
Phase 2: COMMIT    → 전원 YES면 커밋 / 한 명이라도 NO면 전원 롤백
```

| 장점 | 대가 |
| --- | --- |
| 강한 원자성(전원 커밋 또는 전원 롤백) | **블로킹**: PREPARE 후 코디네이터가 죽으면 참여자는 락을 쥔 채 `in-doubt`로 대기 |
| 애플리케이션 코드가 단순(그냥 커밋) | 코디네이터가 **단일 장애점(SPOF)** — 복구 지연이 곧 서비스 정지 |
| 표준(XA)이 있어 벤더 간 연동 가능 | 2-RTT + 디스크 fsync 2회 → **지연 시간 2배 이상** |
| | 참여자 수가 늘수록 실패 확률·락 보유 시간이 곱으로 증가 |

- **MySQL XA**는 지원되지만 실무에서는 거의 쓰지 않습니다. `XA RECOVER`로 남은 prepared 트랜잭션을 사람이 정리해야 하고, binlog와 XA의 상호작용 때문에 특정 버전에서 복제 지연·유실 이슈가 있었습니다.
- **"2PC는 언제 쓰나"** — 같은 DB 벤더의 인스턴스 2대를 묶어야 하고, 지연에 민감하지 않은 배치(정산 마감), 그리고 실패 시 사람이 개입할 수 있는 경로. API 요청 경로에서는 쓰지 않습니다.
- 더 근본적인 한계는 **2PC가 메시지 브로커·외부 API를 참여자로 넣기 어렵다**는 점입니다. PG 결제 승인은 롤백이 안 되므로 XA로 묶어도 소용이 없습니다. 이 지점에서 Saga가 필요해집니다.

### 1.4 Saga — 롤백 대신 보상(Compensation)

Saga는 1987년 Garcia-Molina·Salem이 제안한 개념으로, **긴 트랜잭션을 여러 로컬 트랜잭션(T1…Tn)으로 쪼개고, 중간 실패 시 이미 완료된 단계를 역순으로 "보상 트랜잭션"(C(n-1)…C1)으로 되돌리는** 방식입니다.

```text
정상: T1(주문 생성) → T2(재고 차감) → T3(결제 승인) → T4(배송 접수)
실패: T1 → T2 → T3(실패) → C2(재고 복원) → C1(주문 취소)
```

**보상은 롤백이 아닙니다.** 이 구분이 면접의 핵심입니다.

| | 롤백(Rollback) | 보상(Compensation) |
| --- | --- | --- |
| 성질 | 없던 일로 되돌림(undo log) | **새로운 업무 행위**로 상쇄 |
| 결과 | 상태가 이전과 동일 | 이전과 **다른** 상태(취소 이력·환불 기록이 남음) |
| 예시 | `DELETE` | `status='CANCELED'` + 환불 트랜잭션 |
| 실패 가능 | 거의 없음 | 네트워크·외부 시스템 때문에 **실패할 수 있음** → 재시도·수동 개입 |
| 격리 | 보장 | **보장 안 됨**(중간 상태가 노출됨) |

- 보상은 멱등해야 하고, 여러 번 호출돼도 한 번만 효과가 있어야 합니다(11-19 노트의 멱등성).
- **비가역(non-compensable) 단계**가 존재합니다. 메일 발송, SMS, 외부 PG 승인, 실물 출고는 "취소"는 가능해도 "없던 일"로 만들 수 없습니다. 이런 단계는 **사가의 마지막에 배치**하거나, 앞 단계에서 예약(reserve)만 하고 확정은 뒤로 미룹니다.
- Saga는 **ACID 중 A(원자성)를 "최종적 원자성"으로, I(격리성)를 포기**합니다. 이 손실을 어떻게 메울지가 설계의 본체입니다(1.7절).

### 1.5 오케스트레이션 vs 코레오그래피

| | 오케스트레이션(Orchestration) | 코레오그래피(Choreography) |
| --- | --- | --- |
| 제어 | 중앙 오케스트레이터가 순서를 지시 | 각 서비스가 이벤트를 구독해 다음 단계 수행 |
| 흐름 파악 | 한 파일에서 읽힘 | 전체 흐름이 서비스 5곳에 흩어짐(추적 필요) |
| 결합도 | 오케스트레이터가 모든 서비스에 의존 | 서비스 간 직접 의존 없음(브로커만 의존) |
| 장애 지점 | 오케스트레이터가 SPOF·병목 | 단일 지점 없음, 대신 **순환·무한 루프** 위험 |
| 적합 | 단계 3~7개, 보상 조건이 복잡, 추적이 중요 | 단계 2~3개, 느슨한 결합이 최우선 |
| 대표 도구 | Axon Framework Saga, Temporal, Camunda, 자체 상태 머신 | Kafka/SNS 이벤트, Spring `@EventListener` |

- 실무 기본값은 **오케스트레이션**입니다. 보상 순서·타임아웃·재시도를 한 곳에서 다뤄야 하는데, 코레오그래피로는 "지금 어디까지 갔는지"를 아무도 모르게 되기 쉽습니다.
- 코레오그래피는 **이벤트가 순환 참조**를 만들면 끝이 없습니다(A 실패 → B 보상 이벤트 → A가 다시 처리 → …). 각 이벤트에 사가 ID와 단계를 실어 중복 처리를 차단해야 합니다.
- 어느 쪽이든 **사가 상태를 DB에 저장**해야 합니다. 메모리에만 있으면 오케스트레이터 재시작 시 진행 중인 사가가 증발합니다.

### 1.6 사가를 구성하는 세 종류의 트랜잭션

Chris Richardson의 분류로, 이 표를 그리면 사가 설계가 절반은 끝납니다.

| 종류 | 의미 | 예시 | 실패 시 |
| --- | --- | --- | --- |
| **보상 가능(Compensable)** | 되돌릴 수 있는 단계 | 주문 생성, 재고 예약 | 보상 트랜잭션 실행 |
| **피벗(Pivot)** | **돌아올 수 없는 지점** | 결제 승인 | 여기서부터는 앞을 되돌리지 않고 **끝까지 간다** |
| **재시도 가능(Retryable)** | 실패해도 계속 재시도하면 성공 | 배송 접수, 알림 발송 | 재시도(보상하지 않음) |

- **피벗 앞에서는 실패하면 보상, 피벗 뒤에서는 실패해도 재시도**로 끝까지 밀어붙입니다. 결제가 승인된 뒤 배송 접수 실패로 주문을 취소하면 고객 돈만 빠져나갑니다.
- 따라서 **피벗은 가능한 한 사가의 뒤쪽**에 두고, 앞 단계는 "예약"처럼 되돌리기 쉬운 형태로 만듭니다. 재고를 바로 차감하지 않고 `RESERVED`로 두는 이유입니다.

### 1.7 Saga가 만드는 이상 현상과 대응

격리성이 없으므로 고전적 이상 현상이 다시 나타납니다.

| 이상 현상 | 사가에서의 모습 | 대응 |
| --- | --- | --- |
| 갱신 손실(Lost Update) | 두 사가가 같은 재고를 읽고 각자 차감 → 하나 유실 | **의미적 락(Semantic Lock)**, `@Version` 낙관적 락 |
| 더티 리드(Dirty Read) | 커밋되지 않은 중간 상태(주문 `PENDING`)를 다른 쪽이 읽음 | 중간 상태를 **조회에서 제외**(`WHERE status IN ('PAID')`) |
| 퍼지 리드(Fuzzy Read) | 사가 중간에 값이 바뀌어 앞뒤 조회가 다름 | **값 재확인(Reread Value)** 후 다르면 사가 중단 |
| | | **교환 가능한 업데이트(Commutative)** — `qty = qty - 1`처럼 순서 무관하게 |

**의미적 락**이 가장 실용적입니다.

```sql
-- 재고를 '차감'하지 않고 '예약'으로 표시한다. 사가가 끝나면 CONFIRMED, 실패하면 해제.
UPDATE inventory
   SET reserved_qty = reserved_qty + :qty,
       version = version + 1
 WHERE sku = :sku
   AND stock_qty - reserved_qty >= :qty      -- 조건부: 초과 예약을 DB가 막는다
   AND version = :expectedVersion;           -- 낙관적 락: 동시 사가 충돌 감지
-- 영향 행 수 0 → 재고 부족 또는 버전 충돌 → 사가 중단·재시도
```

- 조회 화면은 `stock_qty - reserved_qty`를 "구매 가능 수량"으로 보여줍니다. 중간 상태가 새지 않으면서 예약이 반영됩니다.
- **TTL이 있는 의미적 락**을 써야 합니다. 사가가 죽으면 예약이 영원히 남아 재고가 조용히 잠깁니다. `reserved_at`을 두고 배치로 회수합니다.

### 1.8 Transactional Outbox — 발행을 업무 트랜잭션 안으로 넣는다

```text
BEGIN (하나의 로컬 트랜잭션)
  INSERT INTO orders (...)                       -- 업무 데이터
  INSERT INTO outbox_event (aggregate, payload)  -- 발행할 사실
COMMIT                                          -- 둘 다 성공하거나 둘 다 실패

[별도 릴레이 프로세스]
  SELECT * FROM outbox_event WHERE status='PENDING' FOR UPDATE SKIP LOCKED LIMIT 100
  → Kafka 발행 → status='PUBLISHED'
```

| 보장 | 내용 |
| --- | --- |
| 발행 원자성 | 업무 커밋과 이벤트 기록이 원자적 → **"커밋됐는데 이벤트 없음"이 불가능** |
| 전달 보장 | **at-least-once**(발행 후 상태 갱신 전에 죽으면 재발행) |
| 순서 | 같은 파티션 키(집계 ID)면 순서 유지. 다른 집계 간 순서는 보장 안 됨 |
| 실패 처리 | 발행 실패 시 `PENDING` 유지 → 재시도. N회 초과 시 `FAILED` + DLQ·알림 |

- **"at-least-once이므로 exactly-once처럼 보이게 만드는 것은 컨슈머 몫"** 입니다. 이 문장이 1.9절과 세트입니다.
- 아웃박스는 **메시지 브로커뿐 아니라 외부 API 호출·이메일 발송**에도 그대로 씁니다(신뢰성 메시지). "발송할 편지"를 DB에 넣고 워커가 보냅니다.
- **이중 쓰기를 없애는 것이 아니라, 이중 쓰기 중 하나를 같은 DB 안으로 옮기는 것**입니다. 같은 DB 안의 두 INSERT는 로컬 트랜잭션으로 묶입니다.

### 1.9 Polling Publisher vs CDC(Debezium)

| | Polling Publisher | Transaction Log Tailing(CDC) |
| --- | --- | --- |
| 방식 | 주기적으로 아웃박스 테이블 조회 | MySQL binlog/Postgres WAL을 읽어 스트림으로 발행 |
| 지연 | 폴링 주기(수백 ms ~ 수 초) | 수십~수백 ms, 준실시간 |
| DB 부하 | 폴링 쿼리 부하(인덱스로 완화) | binlog 읽기(복제 슬롯), 쿼리 부하 거의 없음 |
| 구현 | 앱 코드 + 스케줄러 | Debezium 커넥터 + Kafka Connect |
| 순서 | 청크 단위라 뒤섞일 수 있음 | **binlog 순서 = 커밋 순서** |
| 운영 난이도 | 낮음(앱 안에서 끝남) | 높음(커넥터·스키마 변경·복제 권한·오프셋) |
| 정리 | 앱이 PUBLISHED 행 삭제 | 행 삭제를 이벤트로 오인하지 않도록 SMT 설정 필요 |

- **Debezium Outbox Event Router**(`io.debezium.transforms.outbox.EventRouter`)는 아웃박스 테이블의 `aggregatetype`·`aggregateid`·`type`·`payload` 컬럼을 읽어 **토픽 이름·파티션 키를 자동으로 만듭니다.** 토픽을 `outbox.event.{aggregatetype}` 규칙으로 나누고 `aggregateid`를 키로 쓰는 것이 표준 구성입니다.
- CDC의 함정: ① **행 삭제(DELETE)도 이벤트로 잡힙니다** → `op=d` 무시 또는 tombstone 설정, ② 스키마 변경 시 커넥터 재시작, ③ 복제 슬롯이 끊기면 **binlog 보존 기간이 지나 유실**(Postgres의 경우 슬롯 미소비로 WAL이 무한 증가).
- 소규모·초기에는 **폴링이 정답**입니다. 1초 지연이 문제가 되는 순간 CDC로 옮깁니다.

### 1.10 Inbox 패턴과 멱등 소비 — at-least-once를 exactly-once처럼

```text
컨슈머 처리 순서
1. BEGIN
2. INSERT INTO consumed_event (consumer, event_id)   -- 유니크 제약
   → 위반(duplicate)이면 즉시 롤백하고 오프셋만 커밋(이미 처리됨)
3. 업무 로직 수행 (같은 트랜잭션)
4. COMMIT
5. 오프셋 커밋
```

- **유니크 제약이 유일한 승자 결정자**입니다. `SELECT` 후 `if (exists)` 판정은 동시 소비에서 반드시 뚫립니다(11-19 노트 2.7절과 동일한 원리).
- **업무 처리와 소비 기록을 같은 트랜잭션**에 넣어야 합니다. 따로 두면 "업무는 커밋, 기록은 실패" → 재처리, 또는 그 반대 → 유실.
- 컨슈머가 업무를 커밋하고 오프셋을 커밋하기 전에 죽으면, **같은 메시지가 다시 옵니다.** 이 창은 없앨 수 없으므로 멱등성이 필요합니다.
- 이벤트 ID는 **프로듀서가 만든 UUID**여야 합니다. 브로커의 offset을 쓰면 재발행 시 값이 달라져 중복이 통과합니다.
- 3~5번 사이에서 업무가 실패하면 **예외를 던져 오프셋을 커밋하지 않습니다**(브로커가 재전송). 무한 재시도를 막으려면 재시도 횟수 제한 후 DLQ로 보냅니다.

### 1.11 나머지 패턴들 — TCC·신뢰성 메시지·최선 노력 통지

| 패턴 | 원리 | 대가 | 쓰는 곳 |
| --- | --- | --- | --- |
| **TCC**(Try-Confirm-Cancel) | Try에서 자원 **예약**, Confirm에서 확정, Cancel에서 해제 | 구현량 3배, 예약 자원의 TTL·빈 롤백 처리 필요 | 자원(재고·쿠폰·잔액)을 오래 잡는 예약 시스템 |
| **신뢰성 메시지(Reliable Message)** | 아웃박스와 사실상 동일(로컬 메시지 테이블 + 재발행) | 폴링 부하 | 일반 이벤트 발행 |
| **최선 노력 통지(Best-Effort Notification)** | 끝나고 나서 알림만 보내고, 실패하면 재시도·수동 보정 | 원자성 없음 | 알림·부가 정보 동기화 |
| **보상 트랜잭션(수동)** | 실패 건을 운영 도구로 정산 | 사람 비용 | 금액이 크고 빈도가 낮은 거래 |

- **TCC의 세 함정**: ① 빈 롤백(Cancel이 Try보다 먼저 올 수 있음) 허용, ② 멱등성(Confirm 두 번, Cancel 두 번), ③ **방치(hanging)** — Try만 하고 Confirm/Cancel이 안 오면 예약이 잠깁니다. TTL 회수 잡이 필수입니다.
- TCC는 Saga의 특수형으로 볼 수 있습니다. Try=보상 가능 단계의 의미적 락, Confirm=피벗, Cancel=보상.

### 1.12 패턴 선택표

| 상황 | 선택 |
| --- | --- |
| 같은 서비스, 같은 DB | 로컬 트랜잭션(고민할 필요 없음) |
| DB 커밋 + 브로커/외부 API 알림 | **Transactional Outbox** (+ 멱등 컨슈머) |
| 여러 서비스의 업무를 순차로 조합, 보상 조건 명확 | **오케스트레이션 Saga** |
| 단계 2~3개, 서비스 간 결합을 최소화 | 코레오그래피 + 아웃박스 이벤트 |
| 자원을 미리 잡아야 함(재고·쿠폰·잔액) | **TCC** 또는 의미적 락을 곁들인 Saga |
| 같은 벤더 DB 2대, 지연 허용, 사람 개입 가능 | 2PC/XA (예외적) |
| Kafka 내부 read-process-write만 | Kafka 트랜잭션(EOS) |

---

## 2. 실무 적용 및 주의사항 (Edge Cases)

### 2.1 아웃박스 테이블 설계

```sql
-- src/main/resources/db/migration/V21__create_outbox_event.sql
CREATE TABLE outbox_event (
    id             BIGINT       NOT NULL AUTO_INCREMENT,
    event_id       CHAR(36)     NOT NULL COMMENT 'UUID v4/v7. 컨슈머 멱등 키',
    aggregate_type VARCHAR(60)  NOT NULL COMMENT 'order / payment / inventory',
    aggregate_id   VARCHAR(64)  NOT NULL COMMENT '파티션 키로 사용',
    event_type     VARCHAR(80)  NOT NULL COMMENT 'ORDER_CREATED',
    payload        JSON         NOT NULL,
    status         VARCHAR(20)  NOT NULL COMMENT 'PENDING / PROCESSING / PUBLISHED / FAILED',
    retry_count    INT          NOT NULL DEFAULT 0,
    last_error     VARCHAR(500) NULL,
    occurred_at    DATETIME(6)  NOT NULL,
    published_at   DATETIME(6)  NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uk_outbox_event_id (event_id),
    KEY idx_outbox_status_id (status, id),
    KEY idx_outbox_published_at (published_at)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_0900_ai_ci;
```

- `(status, id)` 인덱스가 **릴레이 조회의 핵심**입니다. `WHERE status='PENDING' ORDER BY id LIMIT 100`이 인덱스만으로 정렬·제한되므로 테이블 스캔이 사라집니다.
- `event_id` 유니크는 릴레이가 같은 이벤트를 **논리적으로** 두 번 만들지 않게 하는 안전장치입니다. 재발행은 같은 `event_id`로 나가야 컨슈머가 중복을 걸러냅니다.
- `payload`를 `JSON` 타입으로 두면 스키마 변경에 유연하지만, **컬럼 길이 제한·인덱싱 제약**이 있습니다. 대용량 본문은 S3 등에 두고 참조만 넣습니다.
- MySQL에는 TTL 삭제가 없으므로 `published_at` 인덱스로 **보존 기간(예: 7일) 지난 PUBLISHED 행을 배치 삭제**합니다. 한 번에 지우면 락·undo가 커지므로 `LIMIT 5000` 루프로 나눕니다.

### 2.2 업무 코드 — 같은 트랜잭션에 쓴다

```java
// src/main/java/com/example/order/order/OrderService.java
package com.example.order.order;

import com.example.order.outbox.OutboxEvent;
import com.example.order.outbox.OutboxEventRepository;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Clock;
import java.time.LocalDateTime;
import java.util.UUID;

@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final OutboxEventRepository outboxRepository;
    private final ObjectMapper objectMapper;
    private final Clock clock;

    public OrderService(OrderRepository orderRepository,
                        OutboxEventRepository outboxRepository,
                        ObjectMapper objectMapper,
                        Clock clock) {
        this.orderRepository = orderRepository;
        this.outboxRepository = outboxRepository;
        this.objectMapper = objectMapper;
        this.clock = clock;
    }

    /**
     * 주문 저장과 이벤트 기록이 같은 트랜잭션. 둘 다 성공하거나 둘 다 실패한다.
     * 여기서 Kafka로 직접 보내면 안 된다(이중 쓰기).
     */
    @Transactional
    public Long placeOrder(PlaceOrderCommand command) {
        Order order = Order.create(command.memberId(), command.lines(), LocalDateTime.now(clock));
        orderRepository.save(order);

        outboxRepository.save(OutboxEvent.pending(
                UUID.randomUUID().toString(),
                "ORDER",
                String.valueOf(order.getId()),
                "ORDER_CREATED",
                toJson(new OrderCreatedPayload(order.getId(), order.getMemberId(), order.getTotalPrice())),
                LocalDateTime.now(clock)
        ));
        return order.getId();
    }

    private String toJson(Object payload) {
        try {
            return objectMapper.writeValueAsString(payload);
        } catch (JsonProcessingException e) {
            // 직렬화 실패는 업무 실패로 취급해 트랜잭션을 롤백한다.
            throw new IllegalStateException("outbox payload 직렬화 실패", e);
        }
    }

    public record PlaceOrderCommand(Long memberId, java.util.List<OrderLine> lines) { }
    public record OrderCreatedPayload(Long orderId, Long memberId, java.math.BigDecimal totalPrice) { }
}
```

```java
// src/main/java/com/example/order/outbox/OutboxEvent.java
package com.example.order.outbox;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

import java.time.LocalDateTime;

@Entity
@Table(name = "outbox_event")
public class OutboxEvent {

    public enum Status { PENDING, PROCESSING, PUBLISHED, FAILED }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "event_id", nullable = false, length = 36, updatable = false)
    private String eventId;

    @Column(name = "aggregate_type", nullable = false, length = 60)
    private String aggregateType;

    @Column(name = "aggregate_id", nullable = false, length = 64)
    private String aggregateId;

    @Column(name = "event_type", nullable = false, length = 80)
    private String eventType;

    @Column(name = "payload", nullable = false, columnDefinition = "JSON")
    private String payload;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private Status status;

    @Column(name = "retry_count", nullable = false)
    private int retryCount;

    @Column(name = "last_error", length = 500)
    private String lastError;

    @Column(name = "occurred_at", nullable = false)
    private LocalDateTime occurredAt;

    @Column(name = "published_at")
    private LocalDateTime publishedAt;

    protected OutboxEvent() {
    }

    public static OutboxEvent pending(String eventId, String aggregateType, String aggregateId,
                                      String eventType, String payload, LocalDateTime occurredAt) {
        OutboxEvent event = new OutboxEvent();
        event.eventId = eventId;
        event.aggregateType = aggregateType;
        event.aggregateId = aggregateId;
        event.eventType = eventType;
        event.payload = payload;
        event.status = Status.PENDING;
        event.occurredAt = occurredAt;
        return event;
    }

    public void markPublished(LocalDateTime now) {
        this.status = Status.PUBLISHED;
        this.publishedAt = now;
        this.lastError = null;
    }

    public void markFailed(String error) {
        this.status = Status.FAILED;
        this.lastError = truncate(error);
    }

    /** 일시 오류는 PENDING으로 되돌려 다음 폴링에서 다시 시도한다. */
    public void retryLater(String error) {
        this.retryCount++;
        this.lastError = truncate(error);
    }

    private String truncate(String error) {
        if (error == null) {
            return null;
        }
        return error.length() <= 500 ? error : error.substring(0, 500);
    }

    public Long getId() { return id; }
    public String getEventId() { return eventId; }
    public String getAggregateType() { return aggregateType; }
    public String getAggregateId() { return aggregateId; }
    public String getEventType() { return eventType; }
    public String getPayload() { return payload; }
    public Status getStatus() { return status; }
    public int getRetryCount() { return retryCount; }
    public LocalDateTime getOccurredAt() { return occurredAt; }
}
```

### 2.3 릴레이 — `SKIP LOCKED` 청크 발행 + ShedLock

```java
// src/main/java/com/example/order/outbox/OutboxEventRepository.java
package com.example.order.outbox;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.LocalDateTime;

public interface OutboxEventRepository extends JpaRepository<OutboxEvent, Long> {

    long countByStatus(OutboxEvent.Status status);

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("delete from OutboxEvent e where e.status = :status and e.publishedAt < :before")
    int deletePublishedBefore(@Param("status") OutboxEvent.Status status,
                              @Param("before") LocalDateTime before);
}

/*
 * 다중 노드에서 병렬 발행이 필요하면 선점은 아래 조건부 UPDATE로 한다.
 * SELECT ... FOR UPDATE SKIP LOCKED를 쿼리로 쓰려면 네이티브 쿼리여야 하고,
 * 트랜잭션이 닫히면 락이 풀려 중복 발행 창이 남는다(2.3절).
 *
 * UPDATE outbox_event
 *    SET status = 'PROCESSING', locked_at = NOW(6), owner = :ownerToken
 *  WHERE id = :id AND status = 'PENDING'      -- affected = 1 인 노드만 발행
 */
```

```java
// src/main/java/com/example/order/outbox/OutboxRelay.java
package com.example.order.outbox;

import net.javacrumbs.shedlock.spring.annotation.SchedulerLock;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.data.domain.Limit;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.time.Clock;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;
import java.util.concurrent.TimeUnit;

@Component
public class OutboxRelay {

    private static final Logger log = LoggerFactory.getLogger(OutboxRelay.class);
    private static final int BATCH_SIZE = 100;
    private static final int MAX_RETRY = 10;
    private static final String TOPIC_PREFIX = "outbox.event.";

    private final OutboxEventRepository repository;
    private final OutboxEventWriter writer;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final Clock clock;

    public OutboxRelay(OutboxEventRepository repository,
                       OutboxEventWriter writer,
                       KafkaTemplate<String, String> kafkaTemplate,
                       Clock clock) {
        this.repository = repository;
        this.writer = writer;
        this.kafkaTemplate = kafkaTemplate;
        this.clock = clock;
    }

    /** 고정 지연 스케줄. 실행 시간이 길어져도 겹치지 않게 fixedDelay를 쓴다. */
    @Scheduled(fixedDelayString = "${outbox.relay.delay-ms:1000}")
    @SchedulerLock(name = "outboxRelay", lockAtLeastFor = "500ms", lockAtMostFor = "30s")
    public void relay() {
        // 조회와 상태 변경을 한 트랜잭션으로 묶으면 발행(외부 I/O)이 트랜잭션 안에 들어간다.
        // 그래서 '선점(PROCESSING 마킹)'과 '발행'을 분리한다.
        List<OutboxEvent> batch = writer.claimBatch(BATCH_SIZE);
        for (OutboxEvent event : batch) {
            publishOne(event);
        }
    }

    private void publishOne(OutboxEvent event) {
        String topic = TOPIC_PREFIX + event.getAggregateType().toLowerCase();
        String key = event.getAggregateId();   // 같은 집계는 같은 파티션 → 순서 보장

        try {
            kafkaTemplate.send(topic, key, envelope(event))
                    .get(5, TimeUnit.SECONDS);   // 발행 확인까지 대기(비동기면 유실 판정 불가)
            writer.markPublished(event.getId(), LocalDateTime.now(clock));
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            writer.retryLater(event.getId(), "interrupted");
        } catch (Exception e) {
            log.warn("outbox publish failed. eventId={}, retry={}", event.getEventId(), event.getRetryCount(), e);
            if (event.getRetryCount() + 1 >= MAX_RETRY) {
                writer.markFailed(event.getId(), e.getMessage());   // DLQ·알림 대상
            } else {
                writer.retryLater(event.getId(), e.getMessage());
            }
        }
    }

    private String envelope(OutboxEvent event) {
        // 컨슈머가 eventId·eventType·occurredAt을 함께 받아야 멱등 처리와 순서 판정이 가능하다.
        return """
                {"eventId":"%s","eventType":"%s","aggregateId":"%s","occurredAt":"%s","payload":%s}
                """.formatted(event.getEventId(), event.getEventType(),
                event.getAggregateId(), event.getOccurredAt(), event.getPayload()).strip();
    }

    public Map<String, Object> stats() {
        return Map.of(
                "pending", repository.countByStatus(OutboxEvent.Status.PENDING),
                "failed", repository.countByStatus(OutboxEvent.Status.FAILED)
        );
    }
}
```

```java
// src/main/java/com/example/order/outbox/OutboxEventWriter.java
package com.example.order.outbox;

import jakarta.persistence.EntityManager;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.List;

/**
 * 아웃박스 상태 변경만 담당한다. 각 메서드가 짧은 트랜잭션이라
 * 외부 I/O(카프카 발행) 동안 DB 커넥션을 잡고 있지 않는다.
 */
@Component
public class OutboxEventWriter {

    private final EntityManager entityManager;
    private final OutboxEventRepository repository;

    public OutboxEventWriter(EntityManager entityManager, OutboxEventRepository repository) {
        this.entityManager = entityManager;
        this.repository = repository;
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public List<OutboxEvent> claimBatch(int size) {
        // SKIP LOCKED은 JPQL 락 모드로 표현이 안 되므로 네이티브 쿼리를 쓴다(MySQL 8.0+).
        // 다른 릴레이가 잡은 행은 기다리지 않고 건너뛴다.
        @SuppressWarnings("unchecked")
        List<OutboxEvent> events = entityManager.createNativeQuery("""
                        SELECT * FROM outbox_event
                         WHERE status = 'PENDING'
                         ORDER BY id
                         LIMIT :size
                         FOR UPDATE SKIP LOCKED
                        """, OutboxEvent.class)
                .setParameter("size", size)
                .getResultList();
        return events;   // 트랜잭션이 끝나면 락이 풀린다 → 중복 발행 창이 남는다(2.3절)
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void markPublished(Long id, LocalDateTime now) {
        repository.findById(id).ifPresent(event -> event.markPublished(now));
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void retryLater(Long id, String error) {
        repository.findById(id).ifPresent(event -> event.retryLater(error));
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void markFailed(Long id, String error) {
        repository.findById(id).ifPresent(event -> event.markFailed(error));
    }
}
```

- **`claimBatch`가 트랜잭션을 닫으면 `SKIP LOCKED`의 락도 풀립니다.** 같은 배치를 다른 노드가 잡으면 **중복 발행**이 됩니다. 그래서 위 구현은 `@SchedulerLock`으로 릴레이 자체를 한 노드로 제한하고, 다중 노드에서 병렬 발행이 필요하면 **`PROCESSING` 상태 + 소유자(UUID)·`locked_at` 조건부 UPDATE**로 선점해야 합니다(2.4절). 어느 쪽이든 결과는 at-least-once이므로 컨슈머 멱등성이 최종 방어선입니다.
- **발행 확인 대기(`.get()`)를 넣는 이유**: 비동기 `send()`만 쓰면 브로커 장애를 알 수 없어 `PUBLISHED`로 잘못 마킹합니다. 대기 시간(5초)은 브로커 `delivery.timeout.ms`보다 짧게 잡습니다.
- 배치 전체를 한 트랜잭션으로 잡고 그 안에서 발행하면 **커넥션을 수 초간 점유**합니다. HikariCP 풀이 10개인 서비스에서 이 실수 하나로 전체 API가 멈춥니다.
- `fixedRate`가 아니라 **`fixedDelay`** 를 씁니다. 앞 실행이 길어질 때 겹쳐서 같은 행을 두 번 잡는 것을 막습니다.

### 2.4 다중 노드 선점 — 조건부 UPDATE로 소유권 잡기

```sql
-- 1) 후보 조회 (락 없이 빠르게)
SELECT id FROM outbox_event
 WHERE status = 'PENDING'
   AND (status <> 'PROCESSING' OR locked_at < NOW(6) - INTERVAL 30 SECOND)
 ORDER BY id
 LIMIT 100;

-- 2) 소유권 선점: 조건부 UPDATE의 영향 행 수로 승자를 결정한다.
UPDATE outbox_event
   SET status = 'PROCESSING', locked_at = NOW(6), owner = :ownerToken
 WHERE id = :id
   AND status = 'PENDING';
-- affected = 1 인 행만 이 노드가 발행한다.
```

- **`SELECT ... FOR UPDATE`를 길게 잡는 것보다, 조건부 UPDATE 한 줄이 운영에 안전**합니다. 락 대기·데드락 없이 승자가 결정됩니다.
- `PROCESSING`에 갇힌 행(노드 사망)은 `locked_at` 기준 **회수(reaper) 잡**이 `PENDING`으로 되돌립니다. 회수도 조건부 UPDATE로 하지 않으면 두 노드가 동시에 회수합니다.
- 회수 후 재발행은 **같은 `event_id`** 로 나가야 합니다. 컨슈머가 중복을 걸러냅니다.

### 2.5 컨슈머 인박스 — 유니크 제약 기반 멱등 소비

```sql
CREATE TABLE consumed_event (
    consumer   VARCHAR(80)  NOT NULL COMMENT '서비스/핸들러 이름',
    event_id   CHAR(36)     NOT NULL,
    consumed_at DATETIME(6) NOT NULL,
    PRIMARY KEY (consumer, event_id)
) ENGINE = InnoDB;
```

```java
// src/main/java/com/example/inventory/messaging/OrderCreatedListener.java
package com.example.inventory.messaging;

import com.example.inventory.inbox.ConsumedEventRepository;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.time.Clock;
import java.time.LocalDateTime;

@Component
public class OrderCreatedListener {

    private final ConsumedEventRepository consumedEventRepository;
    private final InventoryService inventoryService;
    private final ObjectMapper objectMapper;
    private final Clock clock;

    public OrderCreatedListener(ConsumedEventRepository consumedEventRepository,
                                InventoryService inventoryService,
                                ObjectMapper objectMapper,
                                Clock clock) {
        this.consumedEventRepository = consumedEventRepository;
        this.inventoryService = inventoryService;
        this.objectMapper = objectMapper;
        this.clock = clock;
    }

    /**
     * 인박스 기록 + 재고 예약을 한 트랜잭션으로. 중복이면 DataIntegrityViolationException으로
     * 롤백되지만 업무 효과도 함께 롤백되므로 "이미 처리됨"과 동일한 결과가 된다.
     */
    @KafkaListener(topics = "outbox.event.order", groupId = "inventory-service")
    @Transactional
    public void onOrderCreated(String message) throws Exception {
        JsonNode root = objectMapper.readTree(message);
        String eventId = root.get("eventId").asText();

        consumedEventRepository.saveAndFlush(
                ConsumedEvent.of("inventory-service:order-created", eventId, LocalDateTime.now(clock)));

        JsonNode payload = root.get("payload");
        inventoryService.reserve(payload.get("orderId").asLong(), payload.get("linesJson").asText());
    }
}
```

- 예외를 그대로 던지면 **오프셋이 커밋되지 않아** 브로커가 재전송합니다. 중복(`DuplicateKeyException`)은 "정상적으로 이미 처리됨"이므로 **삼키고 오프셋을 커밋**해야 합니다. 위 구조는 예외를 던지지만 업무 효과가 함께 롤백되므로 결과는 같습니다. 로그 소음이 싫으면 `try { saveAndFlush } catch (DataIntegrityViolationException e) { return; }`로 처리합니다.
- **재시도 정책**: 역직렬화 실패·검증 실패처럼 재시도해도 안 되는 오류는 `DefaultErrorHandler`의 `addNotRetryableExceptions`로 즉시 DLQ로 보냅니다. 일시 오류(DB 락 타임아웃, 외부 타임아웃)만 지수 백오프로 재시도합니다.
- 컨슈머 그룹 리밸런싱 중에는 **오프셋 커밋 전 파티션이 넘어가** 같은 메시지가 재처리됩니다(11-16 노트). 인박스가 이 구간을 흡수합니다.
- 인박스 테이블도 무한히 큽니다. **보존 기간(예: 30일) 삭제 잡**을 두거나 `(consumer, consumed_at)` 인덱스로 파티셔닝합니다.

### 2.6 오케스트레이션 Saga 구현 — 상태를 DB에 남긴다

```java
// src/main/java/com/example/order/saga/OrderSaga.java
package com.example.order.saga;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import jakarta.persistence.Version;

import java.time.LocalDateTime;

@Entity
@Table(name = "order_saga")
public class OrderSaga {

    public enum Step {
        STARTED,
        INVENTORY_RESERVED,
        PAYMENT_APPROVED,      // 피벗: 이 뒤로는 보상하지 않는다
        COMPLETED,
        COMPENSATING,
        COMPENSATED,
        FAILED                 // 보상 실패 → 수동 개입
    }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "order_id", nullable = false)
    private Long orderId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 30)
    private Step step;

    @Version
    private Long version;   // 동시에 같은 사가를 진행하는 것을 낙관적 락으로 차단

    @Column(name = "last_error", length = 500)
    private String lastError;

    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;

    protected OrderSaga() {
    }

    public static OrderSaga start(Long orderId, LocalDateTime now) {
        OrderSaga saga = new OrderSaga();
        saga.orderId = orderId;
        saga.step = Step.STARTED;
        saga.updatedAt = now;
        return saga;
    }

    public boolean isBeforePivot() {
        return step == Step.STARTED || step == Step.INVENTORY_RESERVED;
    }

    public void moveTo(Step next, LocalDateTime now) {
        this.step = next;
        this.updatedAt = now;
        this.lastError = null;
    }

    /** 보상 실패·수동 개입이 필요한 상태를 남긴다. 다음 상태(next)는 호출자가 정한다. */
    public void fail(Step next, String error, LocalDateTime now) {
        this.step = next;
        this.lastError = error.length() <= 500 ? error : error.substring(0, 500);
        this.updatedAt = now;
    }

    public Long getId() { return id; }
    public Long getOrderId() { return orderId; }
    public Step getStep() { return step; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}
```

```java
// src/main/java/com/example/order/saga/OrderSagaOrchestrator.java
package com.example.order.saga;

import com.example.order.inventory.InventoryClient;
import com.example.order.payment.PaymentClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Clock;
import java.time.LocalDateTime;

@Service
public class OrderSagaOrchestrator {

    private static final Logger log = LoggerFactory.getLogger(OrderSagaOrchestrator.class);

    private final OrderSagaRepository sagaRepository;
    private final InventoryClient inventoryClient;
    private final PaymentClient paymentClient;
    private final Clock clock;

    public OrderSagaOrchestrator(OrderSagaRepository sagaRepository,
                                 InventoryClient inventoryClient,
                                 PaymentClient paymentClient,
                                 Clock clock) {
        this.sagaRepository = sagaRepository;
        this.inventoryClient = inventoryClient;
        this.paymentClient = paymentClient;
        this.clock = clock;
    }

    /**
     * 외부 호출은 트랜잭션 밖에서 수행한다. 트랜잭션 안에서 HTTP를 호출하면
     * 커넥션·락을 붙잡은 채 네트워크를 기다린다(2.8절).
     */
    public void execute(Long sagaId) {
        OrderSaga saga = load(sagaId);

        // 1) 보상 가능 단계: 재고 예약 (실패 시 보상할 것이 없다)
        if (saga.getStep() == OrderSaga.Step.STARTED) {
            try {
                inventoryClient.reserve(saga.getOrderId());
                advance(sagaId, OrderSaga.Step.INVENTORY_RESERVED, null);
            } catch (Exception e) {
                // 이 시점에는 되돌릴 단계가 없다. 주문을 취소 상태로 만든다.
                advance(sagaId, OrderSaga.Step.COMPENSATED, null);
                return;
            }
        }

        // 2) 피벗: 결제 승인. 여기서 실패하면 앞 단계를 보상한다.
        saga = load(sagaId);
        if (saga.getStep() == OrderSaga.Step.INVENTORY_RESERVED) {
            try {
                paymentClient.approve(saga.getOrderId());
                advance(sagaId, OrderSaga.Step.PAYMENT_APPROVED, null);
            } catch (PaymentRejectedException e) {
                compensate(sagaId, e.getMessage());
                return;
            } catch (Exception e) {
                // 타임아웃: 승인됐는지 알 수 없다 → 조회로 확인해야 한다(11-19 노트 2.9절).
                compensate(sagaId, "payment timeout: " + e.getMessage());
                return;
            }
        }

        // 3) 재시도 가능 단계: 배송 접수는 보상하지 않고 재시도한다.
        saga = load(sagaId);
        if (saga.getStep() == OrderSaga.Step.PAYMENT_APPROVED) {
            try {
                // 배송 접수는 별도 워커가 아웃박스 이벤트로 처리한다. 여기서는 완료 처리만.
                advance(sagaId, OrderSaga.Step.COMPLETED, null);
            } catch (Exception e) {
                log.warn("saga {} delivery scheduling failed. 재시도 대상", sagaId, e);
            }
        }
    }

    /** 보상은 역순으로, 각 단계는 멱등해야 한다. */
    private void compensate(Long sagaId, String reason) {
        OrderSaga saga = load(sagaId);
        advance(sagaId, OrderSaga.Step.COMPENSATING, reason);

        // 결제가 승인되지 않았으므로 결제 취소는 호출하지 않는다(빈 보상 허용).
        try {
            inventoryClient.release(saga.getOrderId());   // 재고 예약 해제(멱등)
        } catch (Exception e) {
            // 보상 실패는 사가를 FAILED로 남겨 운영자가 처리하게 한다. 재시도 잡이 먼저 시도한다.
            advance(sagaId, OrderSaga.Step.FAILED, "compensation failed: " + e.getMessage());
            return;
        }
        advance(sagaId, OrderSaga.Step.COMPENSATED, null);
    }

    @Transactional
    public OrderSaga load(Long sagaId) {
        return sagaRepository.findById(sagaId)
                .orElseThrow(() -> new IllegalStateException("saga not found: " + sagaId));
    }

    /** 상태 전이 자체를 짧은 트랜잭션으로 커밋한다. 재시작해도 여기까지가 남는다. */
    @Transactional
    public void advance(Long sagaId, OrderSaga.Step next, String error) {
        OrderSaga saga = sagaRepository.findById(sagaId)
                .orElseThrow(() -> new IllegalStateException("saga not found: " + sagaId));
        if (error == null) {
            saga.moveTo(next, LocalDateTime.now(clock));
        } else {
            saga.fail(next, error, LocalDateTime.now(clock));
        }
    }
}
```

- **사가 상태를 DB에 두는 이유**는 재시작 복구입니다. "어디까지 갔는지"가 남아 있어야 재시도 잡이 이어서 처리합니다. 메모리 상태 머신은 프로세스가 죽으면 사가가 증발합니다.
- **각 단계는 자체 로컬 트랜잭션**입니다. 사가 전체를 하나의 `@Transactional`로 감싸면 2PC를 쓰지 않는 한 의미가 없고, 외부 호출이 트랜잭션에 갇힙니다.
- 외부 클라이언트에는 **재시도·타임아웃·서킷브레이커**를 붙입니다(Resilience4j). 사가의 재시도 정책과 HTTP 재시도가 겹치면 중복 호출이 곱으로 늘어납니다. 외부 API에 **멱등 키**를 보내야 합니다.
- `compensate()`에서 결제 취소를 호출하지 않은 것은 의도적입니다. **승인되지 않은 결제를 취소하면 PG에서 오류**가 나고, 그 오류가 사가를 FAILED로 만듭니다. "보상할 것이 없으면 아무것도 하지 않는" 것이 정답입니다(빈 보상).

### 2.7 보상 설계 원칙

1. **역순으로 보상**합니다. T3 실패 → C2 → C1. 순서를 바꾸면 이미 해제된 자원을 다시 해제하려 듭니다.
2. **보상은 멱등**해야 합니다. `release()`는 예약이 없어도 200을 돌려주는 형태로 만듭니다(11-19 노트).
3. **비가역 단계는 뒤로 미룹니다.** 알림 발송·실물 출고는 사가의 마지막, 피벗 이후에 둡니다.
4. **보상 실패는 삼키지 않고 상태로 남깁니다.** `FAILED` + 알림 → 운영 도구에서 재처리. 보상이 실패하면 업무는 "부분 완료" 상태로 남습니다.
5. **보상에 타임아웃을 둡니다.** 보상이 무한 재시도되면 사가가 영원히 열려 있고, 의미적 락이 쌓입니다.
6. **자동 보상과 수동 정산의 경계를 정합니다.** 금액이 큰 환불은 자동 보상 대신 정산 큐로 보내는 편이 안전합니다.
7. **사가 로그를 남깁니다.** `saga_id`, `step`, `attempt`, `error`를 남기지 않으면 장애 분석이 불가능합니다.

### 2.8 트랜잭션 경계와 외부 호출

```java
// 잘못된 예: 외부 HTTP 호출이 트랜잭션 안에 있다
@Transactional
public void approve(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    PaymentResult result = pgClient.approve(order);   // 3초 대기 → 커넥션 3초 점유 + 락 유지
    order.approve(result.transactionId());
}
```

```java
// 나은 예: 조회 → 외부 호출(트랜잭션 밖) → 상태 반영(짧은 트랜잭션)
public void approve(Long orderId) {
    OrderSnapshot snapshot = orderReader.find(orderId);          // 트랜잭션 없이 읽기
    PaymentResult result = pgClient.approve(snapshot);           // 외부 I/O
    orderWriter.markApproved(orderId, result.transactionId());   // 짧은 트랜잭션
}
```

- **외부 API 호출을 트랜잭션 안에 두면** 커넥션 풀 고갈, 락 보유 시간 증가, `@Transactional` 타임아웃 초과가 한꺼번에 옵니다. 느린 PG 하나가 전체 서비스를 멈추는 전형적인 경로입니다.
- 대신 **중간 상태가 노출**됩니다(승인 전 상태로 조회됨). 이 상태를 `PENDING`으로 표시하고 화면에서 "처리 중"으로 보여주는 것이 정직한 설계입니다.

### 2.9 `@TransactionalEventListener`로는 왜 부족한가

```java
// 흔한 구현: 커밋 후 발행. '대부분' 동작하지만 원자적이지 않다.
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void publish(OrderCreatedEvent event) {
    kafkaTemplate.send("order-created", event.orderId().toString(), event);
}
```

| 문제 | 내용 |
| --- | --- |
| 유실 창 | 커밋 직후~리스너 실행 전 프로세스 사망 → 이벤트 영구 유실(재시작해도 기록이 없음) |
| 재시도 없음 | 브로커 일시 장애로 발행 실패하면 그대로 사라짐 |
| 순서 | 여러 스레드가 동시에 커밋하면 발행 순서가 뒤섞임 |
| 다중 인스턴스 | 각 인스턴스가 자기 커밋만 발행 → 문제 없지만, 유실은 노드 단위로 발생 |

- `AFTER_COMMIT`은 **"발행 실패 창을 좁히는"** 도구이지 보장이 아닙니다. 유실이 금전 손실로 이어지는 이벤트는 아웃박스로 가야 합니다.
- **Spring Modulith의 이벤트 발행 레지스트리**(`spring-modulith-events-jpa`)는 이 문제를 라이브러리로 해결합니다. `@ApplicationModuleListener`가 붙은 리스너를 위한 **이벤트 발행 테이블**을 자동 생성하고, 완료되지 않은 이벤트를 재발행하며, `@Transactional` 커밋과 이벤트 기록을 같은 트랜잭션에 묶습니다. 직접 아웃박스를 만들기 전에 검토할 만한 선택지입니다.
- 이미 Kafka·RabbitMQ를 쓰고 이벤트가 단순하다면 아웃박스 없이 **컨슈머 멱등성 + 재발행 배치**로 버티는 팀도 있습니다. 다만 "유실되면 안 되는 이벤트"에는 성립하지 않습니다.

### 2.10 Kafka 트랜잭션(EOS)의 경계

| 항목 | 내용 |
| --- | --- |
| 구성 | `transactional.id` 고정 + `enable.idempotence=true` + `isolation.level=read_committed` |
| 보장 | **Kafka 토픽 → Kafka 토픽** read-process-write가 원자적(`sendOffsetsToTransaction`) |
| 못 하는 것 | **DB 커밋과 Kafka 발행을 함께** 원자적으로 만들지 못함 |
| 다운스트림 | `read_committed` 컨슈머만 커밋된 레코드를 봄. `read_uncommitted`면 abort된 레코드도 보임 |

- "Kafka가 exactly-once를 지원하니 아웃박스는 필요 없다"는 흔한 오해입니다. **EOS는 Kafka 경계 안에서만** 성립하고, DB와 Kafka를 가로지르는 원자성은 여전히 아웃박스(또는 CDC)가 담당합니다.
- 멱등 프로듀서의 순서 보장: `enable.idempotence=true`면 in-flight 요청 5개까지 **재시도 시에도 순서가 유지**됩니다. 끄면 재시도로 순서가 뒤집힙니다.
- 프로듀서 `acks=all` + `min.insync.replicas=2`가 아니면 브로커 장애 시 발행 유실이 생깁니다. 아웃박스가 `PUBLISHED`로 마킹했는데 실제로는 유실되는 조합을 만들지 않도록 맞춰야 합니다.

### 2.11 운영 지표와 정리 작업

| 지표 | 의미 | 임계 판단 |
| --- | --- | --- |
| `outbox_pending_count` | 적체 이벤트 수 | 1분 이상 증가 추세 → 릴레이·브로커 장애 |
| `outbox_publish_latency` | `occurred_at` → `published_at` | p99가 SLO(예: 3초) 초과 시 알림 |
| `outbox_failed_count` | 재시도 초과 | 1건이라도 발생 시 Slack 알림 |
| `consumed_event_duplicate_rate` | 중복 소비 비율 | 급증 = 릴레이 중복 발행 또는 리밸런싱 폭주 |
| `saga_stuck_count` | `COMPENSATING`/`INVENTORY_RESERVED` 장기 체류 | 5분 초과 시 알림 |
| `inventory_reserved_stale` | TTL 지난 예약 | 재고 잠김 → 회수 잡 확인 |

- 정리 잡은 **ShedLock으로 한 노드만** 실행합니다(11-06 노트). PUBLISHED 행 삭제와 `PROCESSING` 회수를 같은 잡에 넣습니다.
- 아웃박스 테이블은 **쓰기 핫스팟**이 되기 쉽습니다. 초당 수천 건이면 `id` AUTO_INCREMENT 경합과 인덱스 증가가 문제이므로, `occurred_at` 기준 **월 단위 파티셔닝**과 파티션 드롭으로 정리합니다.
- **삭제는 배치로** 합니다. `DELETE FROM outbox_event WHERE ...`를 한 번에 수십만 건 실행하면 undo 로그·복제 지연이 발생합니다.

### 2.12 함정 체크리스트

- [ ] 업무 저장과 아웃박스 기록이 **같은 `@Transactional` 안**에 있는가 (별도 빈 호출로 트랜잭션이 갈라지지 않았는지 — 11-08 노트의 자가 호출)
- [ ] 이벤트 발행이 **업무 트랜잭션 밖**(릴레이)에서 일어나는가
- [ ] `send()` 성공을 **확인한 뒤에만** `PUBLISHED`로 마킹하는가
- [ ] 파티션 키가 **집계 ID**인가 (null 키는 라운드로빈 → 순서 깨짐)
- [ ] 컨슈머에 **`(consumer, event_id)` 유니크 제약**이 있는가
- [ ] `event_id`를 **프로듀서가 생성**했는가 (브로커 offset 아님)
- [ ] 사가 상태를 **DB에 저장**하고 재시작 복구가 되는가
- [ ] 보상이 **멱등**하고 **빈 보상**을 허용하는가
- [ ] 비가역 단계(메일·출고·PG)가 **피벗 뒤**에 있는가
- [ ] `PROCESSING`에 갇힌 아웃박스·사가를 **회수하는 잡**이 있는가
- [ ] 의미적 락에 **TTL과 회수 잡**이 있는가
- [ ] 아웃박스·인박스 **보존 기간과 파티셔닝** 계획이 있는가
- [ ] 외부 호출이 트랜잭션 **안에** 있지 않은가
- [ ] `acks=all`·`min.insync.replicas`가 아웃박스의 "발행 완료" 판정과 맞는가

### 2.13 테스트로 증명하는 방법

| 시나리오 | 기대 |
| --- | --- |
| 업무 저장 후 커밋 직전 예외 | 주문·아웃박스 **모두 없음** |
| 아웃박스 커밋 후 릴레이 실행 전 프로세스 종료 | 재시작 후 이벤트 **발행됨**(유실 없음) |
| 릴레이 2개 동시 실행 | 한 이벤트가 **한 번만** `PUBLISHED`(또는 중복 발행되어도 컨슈머가 1회 처리) |
| 같은 이벤트 2회 소비 | 업무 효과 **1회**, 중복은 정상 종료 |
| 사가 2단계 실패 | 1단계 보상 실행, 최종 상태 `COMPENSATED` |
| 보상 API가 2회 실패 | 사가 `FAILED`, 알림 발생, 수동 재처리 가능 |
| 결제 타임아웃 | 결제 조회로 상태 확인 후 보상 또는 완료(중복 승인 없음) |

- 아웃박스·인박스 동시성은 **Testcontainers MySQL**로 검증해야 합니다. H2는 `SKIP LOCKED`·유니크 제약의 락 동작이 달라 대체가 안 됩니다(11-21 노트).
- 사가 오케스트레이터는 외부 클라이언트를 **인터페이스로 분리해 목(Mock)으로 실패를 주입**합니다. 실패 주입 없이는 보상 경로가 한 번도 실행되지 않습니다.
- **카오스 지점**을 정해 둡니다: 발행 직후, 커밋 직후, 보상 호출 직전. 이 세 곳에서 프로세스를 죽이는 테스트가 통과하면 설계가 성립합니다.

---

## 3. 기술 면접 / 실전 예상 질문 & 답변 팁

- **Q. 마이크로서비스에서 트랜잭션은 어떻게 처리하나요?**
  - **A.** "로컬 트랜잭션으로 처리 가능한 경계를 먼저 최대한 넓히고, 경계를 넘는 부분은 2PC 대신 Saga와 Transactional Outbox로 풉니다. DB 저장과 이벤트 발행이 갈라지는 지점은 아웃박스로 원자성을 확보하고, 여러 서비스의 업무 조합은 보상 트랜잭션을 가진 Saga로 처리합니다. 전달은 at-least-once이므로 컨슈머 멱등성이 최종 방어선입니다."

- **Q. 2PC를 실무에서 안 쓰는 이유는?**
  - **A.** "원자성은 강하지만 코디네이터가 PREPARE 이후 죽으면 참여자가 락을 쥔 채 in-doubt로 멈춥니다. 락 보유 시간이 길어지고, 코디네이터가 단일 장애점이며, 2-RTT와 fsync 2회로 지연이 커집니다. 게다가 메시지 브로커나 PG 같은 외부 시스템을 참여자로 넣기 어렵습니다. 그래서 요청 경로에서는 쓰지 않고, 지연에 둔감하고 사람이 개입 가능한 배치 정도에 제한적으로 씁니다."

- **Q. Saga에서 '보상'과 '롤백'의 차이는?**
  - **A.** "롤백은 undo log로 없던 일로 만드는 것이고, 보상은 취소·환불 같은 새로운 업무 행위로 상쇄하는 것입니다. 그래서 결과 상태가 이전과 다르고, 보상 자체가 실패할 수 있어 재시도와 수동 개입 경로가 필요합니다. 메일 발송이나 PG 승인처럼 되돌릴 수 없는 단계는 사가의 뒤쪽에 배치합니다."

- **Q. Saga의 격리성 부재는 어떻게 보완하나요?**
  - **A.** "의미적 락으로 중간 상태를 명시적으로 관리합니다. 재고를 바로 차감하지 않고 예약 수량으로 잡아 '구매 가능 수량 = 재고 - 예약'으로 노출하고, 조건부 UPDATE로 초과 예약을 DB가 막게 합니다. 이와 함께 낙관적 락으로 갱신 손실을 막고, 조회에서 중간 상태를 제외하고, 값 재확인으로 퍼지 리드를 감지합니다. 예약에는 반드시 TTL과 회수 잡을 둡니다."

- **Q. 이중 쓰기 문제란 무엇이고 왜 생기나요?**
  - **A.** "DB와 메시지 브로커에 각각 쓰는데 두 저장소를 하나의 원자적 연산으로 묶을 수 없어서, 하나만 성공한 상태가 반드시 생깁니다. DB는 커밋됐는데 이벤트가 없으면 다운스트림이 영원히 모르고, 이벤트가 먼저 나가고 DB가 롤백되면 존재하지 않는 데이터를 알리게 됩니다. try-catch로도 해결되지 않습니다."

- **Q. Transactional Outbox는 그 문제를 어떻게 해결하나요?**
  - **A.** "발행할 사실을 outbox 테이블에 업무 데이터와 같은 로컬 트랜잭션으로 기록하고, 별도 릴레이가 그 테이블을 읽어 실제 발행합니다. 이중 쓰기 중 하나를 같은 DB 안으로 옮긴 것이라 원자성이 생깁니다. 발행 후 상태 갱신 전에 죽으면 재발행되므로 at-least-once이고, 중복은 컨슈머가 흡수합니다."

- **Q. 아웃박스 릴레이는 폴링과 CDC 중 무엇을 쓰나요?**
  - **A.** "초기에는 폴링이 낫습니다. 앱 안에서 끝나고 운영 부하가 적습니다. 지연이 문제가 되면 Debezium 같은 CDC로 binlog를 읽어 커밋 순서 그대로 발행합니다. CDC는 순서 보장이 좋지만 커넥터·복제 권한·스키마 변경·binlog 보존 같은 운영 부담이 늘고, 행 삭제 이벤트를 걸러내는 설정이 필요합니다."

- **Q. at-least-once 환경에서 중복 처리를 어떻게 막나요?**
  - **A.** "컨슈머에 (consumer, event_id) 유니크 제약을 가진 인박스 테이블을 두고, 인박스 기록과 업무 처리를 같은 트랜잭션에 넣습니다. 중복이면 유니크 위반으로 업무 효과까지 함께 롤백되므로 결과가 한 번 처리한 것과 같습니다. SELECT 후 판정하는 방식은 동시 소비에서 반드시 뚫리므로 DB 제약이 승자를 결정해야 합니다."

- **Q. Kafka의 exactly-once가 있으면 아웃박스가 필요 없나요?**
  - **A.** "아닙니다. Kafka EOS는 read-process-write가 모두 Kafka 토픽일 때 성립합니다. DB 커밋과 Kafka 발행을 함께 원자적으로 만들지는 못합니다. DB가 원본인 시스템에서는 아웃박스나 CDC가 필요하고, Kafka 내부 처리만 EOS로 묶습니다."

- **Q. 오케스트레이션과 코레오그래피 중 무엇을 선택하나요?**
  - **A.** "단계가 3개를 넘고 보상 조건이 복잡하면 오케스트레이션입니다. 흐름과 실패 지점이 한 곳에 보이고, 타임아웃·재시도·보상 순서를 통제할 수 있습니다. 코레오그래피는 결합도를 낮추지만 흐름이 여러 서비스에 흩어져 추적이 어렵고, 순환 이벤트로 무한 루프가 생기기 쉽습니다. 어느 쪽이든 사가 상태는 DB에 저장해 재시작 복구가 되어야 합니다."

- **Q. 사가 상태를 메모리에 두면 안 되는 이유는?**
  - **A.** "오케스트레이터가 재시작하면 진행 중인 사가가 증발해 부분 완료 상태로 남습니다. 상태를 DB에 두고 재시도 잡이 이어서 처리해야 합니다. 동시에 같은 사가를 두 워커가 진행하지 않도록 @Version 낙관적 락이나 조건부 UPDATE로 막습니다."

- **Q. 보상 트랜잭션이 실패하면 어떻게 하나요?**
  - **A.** "삼키지 않고 사가를 FAILED로 남기고 알림을 보냅니다. 재시도 잡이 먼저 지수 백오프로 시도하고, 최종 실패는 운영 도구에서 수동 정산합니다. 금액이 큰 환불은 처음부터 자동 보상 대신 정산 큐로 보내는 편이 안전합니다. 보상 실패를 조용히 넘기면 업무는 부분 완료 상태로 굳습니다."

- **Q. TCC 패턴의 함정은?**
  - **A.** "세 가지입니다. Cancel이 Try보다 먼저 올 수 있어 빈 롤백을 허용해야 하고, Confirm·Cancel 모두 멱등해야 하며, Try만 하고 응답이 오지 않는 방치 상태를 TTL로 회수해야 합니다. 구현량이 3배라 자원을 오래 잡아야 하는 재고·쿠폰·잔액 예약 같은 곳에만 씁니다."

- **Q. 외부 API 호출을 트랜잭션 안에 두면 어떤 문제가 생기나요?**
  - **A.** "커넥션과 락을 붙잡은 채 네트워크를 기다립니다. 느린 외부 시스템 하나가 커넥션 풀을 고갈시켜 전체 서비스가 멈추고, 트랜잭션 타임아웃과 락 대기가 늘어납니다. 외부 호출은 트랜잭션 밖에서 하고 상태 반영만 짧은 트랜잭션으로 처리하며, 중간 상태는 PENDING으로 노출합니다."

- **Q. @TransactionalEventListener(AFTER_COMMIT)로 충분하지 않나요?**
  - **A.** "창을 좁힐 뿐 원자적이지 않습니다. 커밋 직후 리스너 실행 전에 프로세스가 죽으면 이벤트가 영구 유실되고, 브로커 일시 장애 시 재시도도 없습니다. 유실이 금전 손실로 이어지는 이벤트는 아웃박스로 가야 합니다. Spring Modulith의 이벤트 발행 레지스트리를 쓰면 이 패턴을 라이브러리로 얻을 수 있습니다."

- **Q. 아웃박스 테이블이 계속 커지면 어떻게 관리하나요?**
  - **A.** "status와 id 복합 인덱스로 릴레이 조회를 인덱스 범위로 제한하고, 보존 기간이 지난 PUBLISHED 행을 배치로 나눠 삭제합니다. 초당 처리량이 크면 occurred_at 기준 월 단위 파티셔닝 후 파티션 드롭으로 정리합니다. 삭제 잡은 ShedLock으로 한 노드만 실행합니다."

---

## 참고

- Garcia-Molina & Salem, *Sagas* (SIGMOD 1987): https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf
- microservices.io — Saga 패턴: https://microservices.io/patterns/data/saga.html
- microservices.io — Transactional Outbox: https://microservices.io/patterns/data/transactional-outbox.html
- Debezium Outbox Event Router: https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html
- Spring Modulith — Event Publication Registry: https://docs.spring.io/spring-modulith/reference/events.html
- Apache Kafka — Exactly-Once Semantics / Transactions: https://kafka.apache.org/documentation/#semantics
- MySQL 8.0 Reference — XA Transactions: https://dev.mysql.com/doc/refman/8.0/en/xa.html
- MySQL 8.0 Reference — `SELECT ... FOR UPDATE SKIP LOCKED`: https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html
- Seata — 분산 트랜잭션 모드(AT/TCC/SAGA/XA): https://seata.apache.org/docs/overview/what-is-seata/
