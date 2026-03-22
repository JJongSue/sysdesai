# Event Sourcing

> 출처: https://www.sysdesai.com/learn/data-management-patterns/event-sourcing

---

## 핵심 아이디어

전통적인 시스템은 엔티티의 **current state**만 저장합니다. 데이터베이스의 행을 업데이트하면 이전 상태는 사라집니다. **Event Sourcing**은 이 방식을 뒤집습니다. 현재 상태를 저장하는 대신, 지금까지 발생한 모든 **event**를 순서대로 기록한 로그를 저장합니다. 현재 상태는 처음부터 이벤트를 replaying하여 도출합니다.

은행 계좌를 예로 들어보겠습니다. 잔액을 직접 저장하지 않고 모든 입금, 출금, 송금 내역을 다시 실행하여 계산하는 방식입니다. Git도 좋은 예시입니다. 코드베이스의 현재 상태는 커밋 히스토리로부터 나옵니다. Event Sourcing은 이런 감사 기능과 시간 여행 기능을 애플리케이션 데이터에 제공합니다.

Event store는 append-only log로 작동하며 현재 상태는 이벤트를 다시 실행하여 얻습니다.

## 주요 구성 요소

| 구성 요소 | 역할 | 예시 |
| --- | --- | --- |
| Event | 발생한 일에 대한 불변의 사실 | `OrderPlaced`, `PaymentFailed` |
| Event Store | aggregate당 이벤트의 append-only log | EventStoreDB, Kafka, DynamoDB streams |
| Aggregate | 이벤트를 생성하고 처리하는 도메인 엔티티 | `Order`, `Account`, `Inventory` |
| Projection | 이벤트를 다시 실행하여 구축된 파생 읽기 모델 | SQL view, Redis hash, Elasticsearch doc |
| Snapshot | 재생 속도를 높이기 위해 상태를 저장해둔 체크포인트 | 매 N번째 이벤트마다 저장 |
| Command Handler | 입력을 검증하고 aggregate를 로드하며 이벤트를 적용함 | 도메인 서비스 계층 |

## Event Sourcing 흐름

전체 Event Sourcing 흐름: 이벤트를 다시 실행하여 aggregate 로드, 새 이벤트 추가, projections 업데이트.

## Snapshots

aggregate에 수천 개의 이벤트가 쌓이면 매번 처음부터 다시 실행하기엔 너무 느려집니다. **Snapshots**가 이 문제를 해결합니다. 주기적으로 aggregate 상태를 직렬화하여 버전 번호와 함께 저장합니다. 다음번에 데이터를 불러올 때는 최신 스냅샷을 가져온 뒤 그 버전 **이후**의 이벤트만 다시 실행하면 됩니다.

```typescript
// 의사 코드: snapshot 최적화를 사용한 aggregate 로딩
async function loadOrder(orderId: string): Promise<Order> {
  // 1. 최신 snapshot 로드 시도
  const snapshot = await eventStore.getSnapshot(orderId);

  let order: Order;
  let fromVersion: number;

  if (snapshot) {
    order = Order.fromSnapshot(snapshot.state);
    fromVersion = snapshot.version + 1;
  } else {
    order = new Order();
    fromVersion = 0;
  }

  // 2. snapshot 이후의 이벤트만 다시 실행
  const events = await eventStore.getEvents(orderId, fromVersion);
  for (const event of events) {
    order.apply(event);
  }

  return order;
}
```

## Event Sourcing + CQRS

Event Sourcing은 CQRS와 훌륭하게 어울립니다. 쓰기 쪽은 event store를 source of truth로 사용합니다. 추가된 모든 이벤트는 쿼리에 필요한 형태의 denormalized 읽기 모델을 유지하는 projectors로 흘러갑니다. 서로 다른 projector는 동일한 이벤트로부터 완전히 다른 뷰를 만들 수 있습니다. 주문 이벤트 하나가 Postgres의 `order_summary` 테이블, 검색용 Elasticsearch 문서, 배송 대시보드용 Redis sorted set을 동시에 구동할 수도 있습니다.

> ℹ️
> Event Versioning은 매우 중요합니다.
> 이벤트는 불변이며 수년 후에 다시 실행될 수도 있습니다. 도메인이 발전하더라도 예전 이벤트를 바꿀 수는 없습니다. 대신 `OrderPlaced_v1`, `OrderPlaced_v2`처럼 버전을 관리하세요. Upcasters는 재생 중에 이전 버전의 이벤트를 새로운 형식으로 변환해줍니다. 이벤트 스키마 진화 전략을 미리 계획하세요. 나중에 수정하려면 훨씬 고생합니다.

## 장점과 비용

| 장점 | 비용 |
| --- | --- |
| 완전한 감사 로그 확보 (누가 무엇을 언제 변경했는지 확인 가능) | event store가 정리 없이 매우 커질 수 있음 |
| 시간 여행 디버깅 가능 (어느 시점으로든 상태 재현 가능) | eventually consistent 읽기 모델 |
| 특정 시점 쿼리 가능 ('특정 날짜 X의 상태는 어떠했는가?') | 개발자의 학습 곡선이 높음 |
| 분리된 projections (쓰기 로직 수정 없이 새 읽기 모델 추가 가능) | event versioning 및 upcasting의 복잡성 |
| 다운스트림 컨슈머를 위한 자연스러운 통합 이벤트 제공 | snapshot 없이는 상태 복구를 위해 수백만 개의 이벤트를 재생하는 것이 느림 |

## 실제 활용 사례

**LinkedIn**은 활동 피드에 Event Sourcing을 사용합니다. 좋아요, 공유, 댓글 같은 모든 활동은 스트림에 추가되는 이벤트입니다. **Walmart**는 재고 관리에 이를 활용하여 감사나 분쟁 해결을 위해 상태를 다시 실행할 수 있게 합니다. **Axon Framework**나 **EventStoreDB**는 인기 있는 프로덕션 구현체입니다. **Apache Kafka**도 event store로 자주 쓰이지만 aggregate 범위의 스트림이나 optimistic concurrency 기능은 직접 구현해야 하는 경우가 많습니다.

> 💡
> 면접 팁
> 면접관이 Event Sourcing에 대해 물어보면 구현 방법보다는 감사 추적, 시간 여행, 분리된 projections 같은 비즈니스 이점부터 설명하세요. 그런 다음 eventual consistency, event versioning, snapshot 전략 같은 트레이드오프를 논의하는 게 좋습니다. CQRS와 함께 사용하며 Kafka나 EventStoreDB를 백엔드 저장소로 활용한다고 덧붙여보세요. 스키마 변경 처리에 대한 질문이 나오면 버전화된 이벤트와 upcasters로 답변하면 완벽합니다.
