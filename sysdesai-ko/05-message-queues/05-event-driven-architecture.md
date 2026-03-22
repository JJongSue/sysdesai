# Event-Driven Architecture(EDA, 이벤트 기반 아키텍처)

> 출처: https://www.sysdesai.com/learn/message-queues/event-driven-architecture

---

## Event-Driven Architecture(EDA)란 무엇인가요?

**Event-Driven Architecture(EDA, 이벤트 기반 아키텍처)**는 서비스들이 **Events(이벤트)**—이미 발생한 일들에 대한 불변의 기록—를 생성하고 소비함으로써 통신하는 디자인 패러다임입니다. 서비스 A가 서비스 B를 직접 호출하는 대신, A는 버스(Bus)에 `OrderPlaced` 이벤트를 발행하고, 주문에 관심 있는 여러 서비스가 이를 구독하여 반응합니다. Producer(생성자)는 자신의 Consumer(소비자)에 대해 전혀 알지 못합니다.

EDA는 Message Queue(메시지 큐)와 Pub/Sub(발행/구독) 시스템이 가능하게 하는 아키텍처 패턴입니다. EDA를 이해한다는 것은 단순한 메시징 메커니즘뿐만 아니라 그 뒤에 숨겨진 디자인 철학을 이해하는 것을 의미합니다.

## 이벤트(Events) vs 명령(Commands) vs 쿼리(Queries)

| 유형 | 의미 | 방향 | 예시 |
| --- | --- | --- | --- |
| Event(이벤트) | 무언가 발생했음 (과거형, 사실) | 브로드캐스트 — 모든 소비자 | OrderPlaced, PaymentFailed, UserRegistered |
| Command(명명) | 특정 작업을 수행하라 (명령형) | 지정된 대상 — 특정 목표 | SendEmail, ProcessPayment, ResizeImage |
| Query(쿼리) | 정보를 달라 | 요청/응답 | GetOrderStatus, FetchUserProfile |

> 💡
> 이벤트를 과거 시제로 명명하세요.
> 이벤트는 이미 발생한 사실을 나타냅니다. 항상 과거형으로 이름을 지으세요: `PlaceOrder` 대신 `OrderPlaced`, `FailPayment` 대신 `PaymentFailed`. 이는 소비자에게 자신이 어떤 명령을 받는 것이 아니라, 과거의 사실에 반응하고 있다는 신호를 보냅니다.

## 이벤트 스키마 설계(Event Schema Design)

잘 설계된 이벤트 스키마는 매우 중요합니다. 이벤트는 사실상 공용 API의 일부입니다. 소비자가 이를 기반으로 구축을 시작하면, 변경 사항은 파괴적인 영향(breaking changes)을 미칩니다. 다음 원칙을 따르세요:

- **Event ID 포함** — 중복 제거 및 멱등성을 위한 고유 식별자
- **Timestamp 포함** — 언제 발생했는가? (ISO 8601 / Epoch 밀리초 사용)
- **Source(출처) 포함** — 어떤 서비스가 이 이벤트를 생성했는가?
- **Schema version 포함** — 하위 호환성을 위해 필요 (예: `"version": 2`)
- **페이로드를 자급자족형으로 유지** — 소비자가 다시 호출할 필요가 없도록 충분한 컨텍스트 포함
- **내부 DB ID만 포함하는 것 지양** — 충분한 비즈니스 컨텍스트 포함

```json
// 잘 설계된 이벤트 스키마 예시
{
  "eventId": "evt_01HX9KMVB3FGQZ",
  "eventType": "order.placed",
  "version": "1.0",
  "timestamp": "2025-11-15T14:32:00Z",
  "source": "order-service",
  "data": {
    "orderId": "ord_7821",
    "userId": "usr_4421",
    "userEmail": "alice@example.com",
    "items": [
      { "productId": "prod_99", "name": "Widget", "quantity": 2, "price": 29.99 }
    ],
    "total": 59.98,
    "currency": "USD"
  }
}
```

## 이벤트 버스 vs 메시지 브로커(Event Bus vs Message Broker)

**Event bus(이벤트 버스)**는 논리적인 개념으로, 이벤트를 위한 공유 채널입니다. **Message broker(메시지 브로커)**(Kafka, SNS, EventBridge 등)는 그 구현체입니다. AWS EventBridge는 특히 이벤트 버스로 설계되었습니다. 스키마 레지스트리, 콘텐츠 기반 라우팅(이벤트 패턴), 그리고 200개 이상의 AWS 서비스 및 SaaS 애플리케이션과의 기본 통합을 지원합니다.

## 코레오그래피 vs 오케스트레이션(Choreography vs Orchestration)

EDA에서 다단계 워크플로우를 조정하는 두 가지 패턴은 다음과 같습니다.

| 측면 | Choreography(코레오그래피/안무) | Orchestration(오케스트레이션/지휘) |
| --- | --- | --- |
| 제어 | 분산형 — 각 서비스가 이벤트를 들었을 때 무엇을 할지 스스로 앎 | 중앙 집중형 — 오케스트레이터가 서비스들에 할 일을 지시함 |
| 결합도 | 낮음 — 서비스들은 이벤트만 알 뿐, 서로를 모름 | 높음 — 오케스트레이터가 모든 참여자를 알고 있음 |
| 가시성 | 트레이싱 없이는 전체 흐름을 보기 어려움 | 쉬움 — 오케스트레이터가 전체 뷰를 가짐 |
| 장애 처리 | 각 서비스가 자신의 장애를 처리함 | 오케스트레이터가 재시도, 보상, 롤백 수행 가능 |
| 적합한 경우 | 단순하고 참여자가 적은 안정적인 워크플로우 | Saga나 보상 트랜잭션이 포함된 복잡한 워크플로우 |
| 예시 | Kafka/SNS를 사용한 EDA | AWS Step Functions, Temporal, Camunda |

Choreography: 각 서비스가 이벤트를 듣고 새 이벤트를 발행함 — 중앙 조정자 없음.

## 이벤트 기반 아키텍처에서의 일반적인 주의점(Pitfalls)

> ⚠️
> 함정: Event schema coupling(이벤트 스키마 결합)
> 소비자가 이벤트 페이로드의 특정 필드에 의존하게 되면, 스키마 변경 시 모든 소비자가 망가집니다. 스키마 버전을 관리하고, 발행 전에 호환성 체크를 강제할 수 있는 **Event schema registry(스키마 레지스트리)**(Confluent Schema Registry, AWS Glue Schema Registry 등) 사용을 고려하세요.

> ⚠️
> 함정: Event ordering assumptions(이벤트 순서 가정)
> 분산 시스템에서는 이벤트가 순서대로 도착하지 않을 수 있습니다. 서로 다른 파티션을 사용하는 경우 `PaymentFailed`가 `OrderPlaced`보다 먼저 도착할 수 있습니다. 소비자가 순서가 바뀐 배달을 견딜 수 있도록 설계하거나, 파티션 키를 사용하여 엔티티 내의 순서를 보장하십시오.

> ⚠️
> 함정: Invisible workflows(보이지 않는 워크플로우)
> 코레오그래피 방식에서 전체 비즈니스 흐름은 수십 개의 이벤트 핸들러에 흩어져 있어 암시적입니다. 실패한 주문을 디버깅할 때 10개의 서비스를 추적해야 할 수도 있습니다. 초기부터 분산 트레이싱(Jaeger, AWS X-Ray)과 모든 이벤트에 상관관계 ID(Correlation ID)를 도입하는 데 투자하세요.

## Outbox Pattern(아웃박스 패턴)

고전적인 이중 쓰기(dual-write) 문제: 어떻게 데이터베이스에 저장하고 동시에 이벤트를 발행하는 작업을 원자적으로 처리할 수 있을까요? DB 저장은 성공했는데 발행이 실패하거나 그 반대의 경우, 시스템은 불일치 상태가 됩니다. **Outbox Pattern(아웃박스 패턴)**은 이를 해결합니다. 비즈니스 데이터와 동일한 DB 트랜잭션 내에서 `outbox` 테이블에 이벤트를 기록합니다. 별도의 **Relay process(릴레이 프로세스)**가 아웃박스를 읽어 메시지 버스에 발행하고, 전송 완료 표시를 합니다.

아웃박스 패턴은 DB 쓰기와 이벤트 발행 간의 원자성을 보장합니다 — 이중 쓰기 문제가 발생하지 않습니다.

> 💡
> 인터뷰 팁
> 아웃박스 패턴은 면접관에게 즉각적으로 깊은 인상을 줄 수 있는 고급 주제입니다. "DB와 메시지 큐의 동기화를 어떻게 보장하나요?"라고 묻는다면 아웃박스를 설명하세요: "비즈니스 데이터와 동일한 트랜잭션에서 아웃박스 테이블에 이벤트를 씁니다. 릴레이 서비스가 이를 비동기적으로 Kafka/SNS에 발행합니다. 이는 DB 관점에서는 Exactly-once(정확히 한 번), 메시지 버스 관점에서는 At-least-once(최소 한 번)이므로 소비자는 멱등성을 가져야 합니다."
