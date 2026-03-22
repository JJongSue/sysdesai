# Message Broker Pattern

> 원문: https://www.sysdesai.com/learn/messaging-patterns/message-broker

---

## Message Broker란 무엇인가?

**Message broker**는 생산자(producer)로부터 메시지를 받아 안정적으로 보관(persist)하고, 적절한 소비자(consumer)에게 라우팅하는 중개자입니다. 직접적인 point-to-point HTTP 호출과 달리, 브로커는 시간, 공간 및 동기화 측면에서 송신자와 수신자를 분리(decouple)합니다. 생산자는 소비자가 실행 중인지, 몇 명인지, 어디에 있는지 알 필요가 없습니다.

브로커는 세 가지 핵심 기능을 제공합니다: **Routing** (메시지가 어느 큐나 토픽에 속할지 결정), **Persistence** (재시작이나 장애에도 견딜 수 있도록 메시지를 안전하게 저장), **Delivery guarantees** (메시지가 의도한 대로 소비자에게 도달하도록 보장). 순서 보장, 필터링, dead-letter 처리, 우선순위 등 다른 모든 기능은 이 세 가지 기본 요소 위에 구축됩니다.

## Broker Topologies
중앙 허브로서의 message broker: 생산자는 브로커에 메시지를 보내고, 브로커는 이를 소비자 큐로 라우팅합니다.

업계에는 두 가지 주요 브로커 토폴로지가 있습니다:

- **Queue-based (point-to-point)**: 각 메시지는 정확히 한 명의 소비자에 의해 소비됩니다. 동일한 큐의 여러 소비자는 메시지를 두고 경쟁합니다 (competing consumers 패턴). 작업 분산 및 작업 큐(work queue)에 가장 적합합니다. 예: SQS Standard Queue, RabbitMQ 기본 큐.
- **Topic-based (pub/sub)**: 각 메시지가 모든 구독자에게 전달됩니다. 각 구독자는 복사본을 하나씩 받습니다. 이벤트 알림 및 브로드캐스트에 가장 적합합니다. 예: Kafka topic, SNS, RabbitMQ fanout exchange.

## Message Persistence & Durability

**Message persistence**는 브로커가 생산자에게 응답하기 전에 메시지를 디스크에 기록하는 것을 의미합니다. 브로커가 충돌 후 재시작되더라도 메시지는 유실되지 않습니다. 대부분의 운영용 브로커가 이를 지원하지만, 지연 시간(latency) 비용이 발생합니다. 모든 메시지에 대해 `fsync`를 수행하는 것은 비용이 많이 들 수 있습니다. Kafka는 log segment에 쓰기를 일괄 처리(batching)하여 이 비용을 분담(amortize)하며, RabbitMQ는 노드 간 acknowledgement 기반 미러링을 사용합니다.

> ⚠️
> Transient vs Persistent 메시지
> RabbitMQ 및 AMQP 기반 브로커는 메시지별 내구성을 허용합니다. `delivery_mode = 2`는 메시지를 persistent로 표시합니다. 내구성 있는 큐에 `delivery_mode = 1`로 발행하면 브로커 재시작 시 메시지가 유실됩니다. 큐와 메시지 모두 내구성(durable) 있게 설정해야 합니다.

## Broker Comparison

| 브로커 | 모델 | 순서 보장 | 처리량 | 적합한 용도 |
| --- | --- | --- | --- | --- |
| Apache Kafka | Partitioned log (pull) | 파티션별 순서 보장 | 매우 높음 (초당 수백만 건) | 이벤트 스트리밍, 감사 로그(audit log), 리플레이, 분석 파이프라인 |
| RabbitMQ | Queue + exchange (push/pull) | 큐별 FIFO | 높음 (초당 수만 건) | 작업 큐, 복잡한 라우팅, request-reply 패턴 |
| AWS SQS | Managed queue (pull) | Standard: 보장 안 됨; FIFO: 엄격함 | Standard: 무제한; FIFO: 초당 3000건 | 서버리스 워크로드, 단순한 AWS 네이티브 분리 |
| AWS SNS | Managed topic (push) | 보장 안 됨 | 매우 높음 | SQS/Lambda/HTTP 엔드포인트로 Fan-out |
| Apache Pulsar | Segmented log + queue 하이브리드 (pull/push) | 파티션별 순서 보장 | 매우 높음 | Geo-replicated 스트리밍, 계층형 스토리지(tiered storage) |
| Redis Pub/Sub | Ephemeral channel (push) | 영속성 없음 | 극도로 낮은 지연 시간 | 유실이 허용되는 실시간 알림 |

## Kafka Architecture Deep-Cut

Kafka의 아키텍처는 전통적인 브로커와 근본적으로 다릅니다. 메시지는 브로커에 걸쳐 파티셔닝된 **immutable append-only log**에 기록됩니다. 소비자는 로그에서의 위치(**offset**)를 독립적으로 추적합니다. 이를 통해 (1) 여러 독립적인 consumer group이 동일한 토픽을 서로 다른 offset에서 읽을 수 있고, (2) offset을 과거 위치로 재설정하여 메시지를 리플레이할 수 있으며, (3) 순차적 디스크 I/O를 통해 극도로 높은 처리량을 얻을 수 있습니다.

Kafka partitioned log: 두 개의 독립적인 consumer group이 동일한 토픽을 서로 다른 offset에서 읽습니다.

## RabbitMQ Exchange Types

RabbitMQ의 라우팅은 **exchanges**에 의해 구동됩니다. Exchange는 생산자로부터 메시지를 받아 바인딩 규칙(binding rules)에 따라 큐로 라우팅하는 객체입니다. 올바른 exchange 타입을 선택하는 것이 RabbitMQ를 사용할 때 가장 중요한 설계 결정입니다.

| Exchange 타입 | 라우팅 로직 | 사용 사례 |
| --- | --- | --- |
| `direct` | 라우팅 키가 정확히 일치할 때 라우팅 | 작업 큐, 일대일 라우팅 |
| `fanout` | 바인딩된 모든 큐에 브로드캐스트, 라우팅 키 무시 | Pub/sub fan-out |
| `topic` | 라우팅 키에 와일드카드 패턴 매칭 (`*`, `#`) | 선택적 라우팅이 필요한 다중 구독자 |
| `headers` | 메시지 헤더 속성으로 라우팅 | 키 인코딩 없는 콘텐츠 기반 라우팅 |

## Choosing a Broker

- **Kafka 사용**: 이벤트 리플레이, 감사 로그, 고처리량 스트리밍, 또는 서로 다른 속도로 동일한 이벤트를 읽는 여러 독립적인 consumer group이 필요한 경우.
- **RabbitMQ 사용**: 복잡한 라우팅 규칙, request-reply 패턴, 메시지별 우선순위, 또는 유연한 토폴로지를 가진 전통적인 작업 큐가 필요한 경우.
- **SQS + SNS 사용**: 이미 AWS를 사용 중이고, 관리 부담이 없는(zero-ops) 메시징을 원하며, 서버리스 소비자(Lambda)를 활용한 단순한 fan-out이 필요한 경우.
- **Redis Pub/Sub 사용**: 브로커 재시작 시 메시지 유실이 허용되는 일시적이고(ephemeral) 낮은 지연 시간이 필요한 알림(예: 실시간 접속 표시기)의 경우에만 사용.

> 💡
> 인터뷰 팁
> 면접관은 거의 항상 '여기서 Kafka를 쓸 건가요, SQS를 쓸 건가요?'라고 묻습니다. 답은 세 가지 질문에 달려 있습니다: (1) 메시지 리플레이나 독립적인 여러 consumer group이 필요한가? → Kafka. (2) AWS를 사용 중이며 관리의 편의성을 원하는가? → SQS/SNS. (3) 복잡한 라우팅 로직이 필요한가? → RabbitMQ. 승자를 고르기보다는 트레이드오프를 설명하세요.
