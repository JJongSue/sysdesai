# Publish/Subscribe Pattern (Deep Dive)

> 원문: https://www.sysdesai.com/learn/messaging-patterns/pub-sub-pattern

---

## Pub/Sub이란 무엇인가?

**Publish/Subscribe** (pub/sub) 패턴은 **topic**이나 **channel**이라는 중개자를 통해 메시지 생산자(producer)와 소비자(consumer)를 분리(decouple)합니다. Publisher는 메시지를 누가 받을지 모르는 상태에서 이벤트를 발행합니다. Subscriber는 특정 topic에 관심을 표명하고, 일치하는 모든 메시지를 수신합니다. 이는 Google Cloud Pub/Sub, AWS SNS, Apache Kafka, Redis Pub/Sub 같은 시스템의 핵심 패턴입니다.

핵심은 양측이 서로의 존재를 알 필요가 없다는 점입니다. 덕분에 publisher를 전혀 수정하지 않고도 새로운 소비자(감사 로그 기록, 알림 전송, 캐시 업데이트 등)를 추가할 수 있습니다. 이는 분산 시스템에 적용된 전형적인 **Open/Closed Principle**의 사례입니다.

## Core Pub/Sub Flow
기본적인 pub/sub: 하나의 publisher가 여러 독립적인 subscriber에게 메시지를 Fan-Out합니다.

## Topic Design

Topic의 세분화(granularity)는 pub/sub 시스템에서 가장 중요한 설계 결정 중 하나입니다. Topic이 너무 광범위하면 소비자가 불필요한 노이즈를 필터링해야 하고, 너무 좁으면 schema가 급증하고 운영 오버헤드가 발생합니다.

| 세분화 수준 | 예시 Topic | 장점 | 단점 |
| --- | --- | --- | --- |
| Coarse (entity-level) | `orders` | 심플한 publisher API, 관리할 topic 수가 적음 | 소비자가 직접 이벤트 타입을 필터링해야 함 |
| Medium (event-level) | `orders.placed`, `orders.shipped` | 소비자가 필요한 것만 수신함 | Topic 수가 많아짐; publisher가 정확히 라우팅해야 함 |
| Fine (instance-level) | `orders.placed.us-east-1` | 극도로 높은 선택성 | Topic 폭발; 관리가 어려움 |

> 💡
> 권장 네이밍 규칙
> `{domain}.{entity}.{event}`와 같이 점(.)으로 구분된 계층적 이름을 사용하세요. 예: `commerce.order.placed`, `commerce.order.shipped`, `payments.invoice.created`. 이를 통해 NATS나 RabbitMQ의 topic exchange처럼 와일드카드 구독을 지원하는 브로커에서 접두사 기반 구독이 가능해집니다.

## Subscription Filtering

최신 브로커들은 **server-side filtering**을 지원하여 소비자가 기준에 맞는 메시지만 수신할 수 있게 합니다. 이는 클라이언트 측에서 모든 메시지를 가져와 대부분을 버리는 것보다 훨씬 효율적입니다.

- **Topic-based filtering**: subscriber가 특정 topic 문자열(예: `orders.placed`)을 구독합니다.
- **Content-based filtering**: subscriber가 메시지 속성에 대한 조건(predicate)을 제공합니다(예: `region = 'us-east-1' AND amount > 1000`). AWS SNS와 Google Cloud Pub/Sub이 이를 지원합니다.
- **Wildcard subscriptions**: NATS는 `orders.*`(한 단계) 및 `orders.>`(하위 모든 단계)를 지원합니다. RabbitMQ topic exchange는 `#`와 `*`를 지원합니다.

## Delivery Guarantees

인터뷰에서 전송 의미론(delivery semantics)을 이해하는 것은 매우 중요합니다. 세 가지 수준이 있으며, 대부분의 실제 시스템은 기본적으로 **at-least-once**를 사용합니다.

| 보장 수준 | 설명 | 리스크 | 사용 예시 |
| --- | --- | --- | --- |
| At-most-once | 전송 후 망각(Fire and forget); 메시지 유실 가능성 있음 | 장애 시 데이터 유실 | 지연이 허용되는 메트릭이나 실시간 텔레메트리 |
| At-least-once | 메시지가 최소 한 번 전달됨; 중복 가능성 있음 | 소비자가 Idempotent해야 함 | 대부분의 메시징 시스템 (Kafka 기본값, SQS, SNS, RabbitMQ) |
| Exactly-once | 정확히 한 번 전달됨; 중복이나 유실 없음 | 높은 복잡성과 비용 | 트랜잭션과 idempotent producer를 사용하는 Kafka (EOS) |

> ⚠️
> Exactly-Once는 비용이 많이 듭니다
> Exactly-once 전송은 내부적으로 분산 트랜잭션이나 two-phase commit이 필요합니다. 실제로는 대부분의 아키텍트가 소비자를 **idempotent**하게 설계하고 at-least-once 전송을 받아들입니다. 제대로 구현한다면 이것이 더 단순하고 성능이 뛰어나며 안전합니다.

## Push vs Pull Delivery

브로커는 메시지를 subscriber에게 **push**하거나, subscriber가 자신의 속도에 맞춰 **pull**할 수 있게 합니다. 각 모델에는 고유한 트레이드오프가 있습니다.

| 모델 | 작동 방식 | Back-Pressure | 예시 |
| --- | --- | --- | --- |
| Push | 메시지 도착 시 브로커가 즉시 전달 | Subscriber가 압도당할 수 있음; 흐름 제어 필요 | SNS HTTP 구독, WebSocket push |
| Pull | Subscriber가 자신의 일정에 따라 브로커를 폴링 | 자연스러운 back-pressure; 소비자가 처리 속도 제어 | Kafka consumer, SQS 폴링, Google Cloud Pub/Sub pull |

## Durable vs Ephemeral Subscriptions

**Durable subscription**은 subscriber가 오프라인일 때 메시지를 보관했다가 다시 연결될 때 전달합니다. **Ephemeral** (non-durable) subscription은 subscriber가 없을 때 메시지를 버립니다. Kafka의 consumer group은 본질적으로 durable하며, offset이 브로커에 저장됩니다. Redis Pub/Sub은 ephemeral합니다. 오프라인 subscriber는 메시지를 놓치게 됩니다.

Durable subscriptions: 각 subscriber는 메시지를 독립적으로 보관하는 자신만의 큐를 가집니다.

## Scaling Pub/Sub Systems

단일 소비자가 topic의 메시지 속도를 따라가지 못할 때, **consumer groups** (Kafka)나 공유 큐 뒤의 **competing consumers** 패턴을 사용하여 확장할 수 있습니다. 주요 차이점은 다음과 같습니다. Kafka consumer group의 모든 소비자는 partition을 공유하며, 각 partition은 한 번에 정확히 한 명의 그룹 멤버에 의해 소비됩니다. 이를 통해 partition 수에 비례하는 병렬성을 얻을 수 있습니다.

SNS/SQS의 경우 표준 패턴은 다음과 같습니다. SNS topic이 여러 **SQS queues**로 fan-out되며, 각 큐는 서로 다른 서비스가 소유합니다. 각 서비스 내에서는 여러 EC2나 Lambda 소비자가 해당 SQS 큐의 메시지를 두고 경쟁(compete)합니다. 이것이 전형적인 AWS fan-out 아키처입니다.

> 💡
> 인터뷰 팁
> 알림 시스템이나 이벤트 기반 마이크로서비스 설계를 요청받으면, pub/sub으로 시작하여 다음 사항들을 즉시 언급하세요: (1) topic 네이밍, (2) delivery guarantee (at-least-once + idempotent consumers), (3) 소비자 확장 방법 (consumer groups / competing consumers), (4) subscriber 장애 시 대응 방법 (durable queue vs DLQ). 이는 면접관이 가장 중요하게 생각하는 네 가지 질문을 모두 다룹니다.
