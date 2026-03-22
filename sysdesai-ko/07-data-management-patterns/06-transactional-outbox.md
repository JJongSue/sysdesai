# Transactional Outbox Pattern (트랜잭셔널 아웃박스 패턴)

> 출처: https://www.sysdesai.com/learn/data-management-patterns/transactional-outbox

---

## 이중 쓰기(Dual-Write) 문제

마이크로서비스에서 흔히 볼 수 있는 패턴은 서비스가 데이터베이스를 업데이트한 다음 메시지 브로커에 이벤트를 발행하는 것입니다. 하지만 이는 **이중 쓰기(Dual-Write)**입니다. 즉, 원자성(Atomicity)이 보장되지 않는 두 개의 개별 작업입니다. 만약 데이터베이스 쓰기는 성공했지만 메시지 발행이 실패하거나, 두 작업 사이에서 서비스가 중단되면 상태는 변경되었으나 아무에게도 알림이 가지 않는 불일치 상태가 발생합니다.

반대로 메시지를 먼저 발행하고 데이터베이스 쓰기가 실패하면, 다운스트림 서비스들은 커밋되지 않은 이벤트에 반응하게 됩니다. **Transactional Outbox Pattern**(트랜잭셔널 아웃박스 패턴)은 메시지 발행을 상태 변경과 동일한 데이터베이스 트랜잭션의 일부로 만들어 이 문제를 해결합니다.

> ⚠️
> 프로덕션 환경에서 이중 쓰기를 절대 하지 마세요
> `db.save(entity)`를 수행한 뒤 이어서 `kafka.publish(event)`를 호출하고 싶은 유혹은 어디에나 있습니다. 하지만 두 작업 사이의 충돌, 네트워크 타임아웃, 또는 브로커 장애는 시스템을 불일치 상태로 만듭니다. Transactional Outbox는 이를 해결하기 위한 운영 등급의 솔루션입니다.

## Outbox Pattern (아웃박스 패턴)

이 패턴은 비즈니스 데이터와 **동일한 데이터베이스**에 `outbox`(아웃박스) 테이블을 도입합니다. 서비스가 상태 변경을 기록할 때, **동일한 트랜잭션 내에서** `outbox` 테이블에도 레코드를 삽입합니다. 별도의 백그라운드 프로세스인 **Message Relay**(메시지 릴레이)는 발행되지 않은 아웃박스 레코드를 읽어 메시지 브로커에 발행한 후 발행 완료로 표시합니다.

트랜잭셔널 아웃박스: 상태 변경과 아웃박스 삽입은 원자적이며, 릴레이가 비동기적으로 발행합니다.

## Outbox 테이블 스키마 예시

```sql
-- Outbox 테이블
CREATE TABLE outbox (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aggregate_id  VARCHAR(255) NOT NULL,   -- 예: orderId
  aggregate_type VARCHAR(255) NOT NULL,  -- 예: 'Order'
  event_type    VARCHAR(255) NOT NULL,   -- 예: 'OrderPlaced'
  payload       JSONB NOT NULL,          -- 전체 이벤트 페이로드
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  published_at  TIMESTAMPTZ             -- NULL이면 아직 발행되지 않음
);

-- 서비스 코드: 단일 트랜잭션 내에서 쓰기
BEGIN;
  INSERT INTO orders (id, user_id, status, ...) VALUES (...);
  INSERT INTO outbox (aggregate_id, aggregate_type, event_type, payload)
    VALUES ('order-123', 'Order', 'OrderPlaced', '{"orderId": "order-123", ...}');
COMMIT;
```

## Message Relay (메시지 릴레이) 구현 방식

메시지 릴레이를 구현하는 두 가지 주요 방법이 있습니다:

-   **폴링 퍼블리셔 (Polling Publisher)** — 백그라운드 작업이 일정 간격(예: 100ms마다)으로 아웃박스 테이블을 폴링하여 발행되지 않은 행을 찾아 브로커에 발행하고 발행 완료로 표시합니다. 구현이 간단하지만 폴링 간격에 비례하는 지연 시간이 발생하고 DB 부하를 줄 수 있습니다.
-   **로그 테일링 (Log-Tailing, CDC)** — CDC(Change Data Capture)를 사용하여 아웃박스 테이블로의 삽입 작업을 데이터베이스의 트랜잭션 로그(Postgres의 WAL, MySQL의 binlog 등)에서 감시합니다. **Debezium**과 같은 CDC 도구는 이러한 변경 사항을 캡처하여 거의 실시간으로 Kafka에 발행합니다. 폴링 오버헤드가 없고 지연 시간이 매우 짧지만 CDC 인프라가 필요합니다.

CDC(Debezium)를 활용한 아웃박스: 폴링 없이 실시간으로 이벤트를 전달합니다.

## 최소 한 번(At-Least-Once)과 멱등성

아웃박스 패턴은 **최소 한 번(At-Least-Once)** 전달을 보장합니다. 릴레이가 메시지를 발행한 후 발행 완료 표시를 하기 전에 중단되면 동일한 메시지를 다시 발행할 수 있기 때문입니다. 따라서 컨슈머(Consumer)는 반드시 **멱등성(Idempotent)**을 가져야 합니다. 각 이벤트는 고유 ID를 가져야 하며, 컨슈머는 처리된 ID를 추적하여 중복을 건너뛰어야 합니다. 완전한 **정확히 한 번(Exactly-once)** 처리는 프로듀서와 컨슈머 양쪽 수준에서의 조정이 필요합니다(예: Kafka 트랜잭셔널 프로듀서 + 컨슈머 멱등성).

> 💡
> 아웃박스 라이브러리
> 아웃박스를 처음부터 직접 구현할 필요는 거의 없습니다. **Debezium**이 CDC 릴레이를 처리해 줍니다. Spring Boot, .NET, Go 등을 위한 **Transactional Outbox** 라이브러리들이 존재합니다. Debezium의 Outbox 커넥터는 이벤트를 애그리거트별 Kafka 토픽으로 자동 라우팅해 주는 운영 준비가 된 솔루션입니다.

> 💡
> 인터뷰 팁
> 이중 쓰기(Dual-write) 문제는 고전적인 인터뷰 함정입니다. 많은 지원자가 장애 발생 가능성을 깨닫지 못하고 DB 업데이트와 Kafka 발행을 두 단계로 제안하곤 합니다. 이 문제를 선제적으로 식별하고 Transactional Outbox를 제안하면 프로덕션 경험이 풍부하다는 신호를 줄 수 있습니다. 폴링 퍼블리셔(단순함, 높은 지연 시간)와 CDC/Debezium 접근 방식(추가 인프라, 실시간성)을 모두 설명하고, 컨슈머 멱등성을 위한 최소 한 번 전달 요구 사항을 언급하세요.
