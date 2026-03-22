# Publish/Subscribe(발행/구독) 패턴

> 출처: https://www.sysdesai.com/learn/message-queues/pub-sub

---

## Point-to-Point(점대점) vs Pub/Sub(발행/구독)

메시징에는 두 가지 기본 패턴이 있습니다. **Point-to-Point(점대점)**(큐 기반) 메시징에서는 각 메시지가 정확히 하나의 Consumer(소비자)에 의해 소비됩니다. 여러 워커가 작업을 위해 경쟁하는 태스크 큐를 생각하면 됩니다. 반면 **Publish/Subscribe(발행/구독, Pub/Sub)** 메시징에서는 **Topic(토픽)**에 발행된 메시지가 해당 토픽의 **모든** Subscriber(구독자)에게 전달됩니다. 하나의 이벤트가 여러 수신자에게 전달되는 방식입니다.

| 비교 항목 | Point-to-Point(점대점) | Pub/Sub(발행/구독) |
| --- | --- | --- |
| 전달 대상 | 하나의 Consumer(경쟁 방식) | 모든 Subscriber(구독자) |
| 주요 유스케이스 | 태스크/작업 분산 | 이벤트 알림 / Fan-out(팬아웃) |
| 결합도 | Producer가 큐를 알고 있음 | Producer가 Subscriber를 모름 |
| 예시 | SQS, RabbitMQ 기본 큐 | SNS, Kafka 토픽, Google Pub/Sub |
| Consumer 확장성 | 워커 증설 = 처리량 증가 | 각 Subscriber 그룹이 독립적으로 확장됨 |

## 토픽(Topics)과 구독(Subscriptions)

**Topic(토픽)**은 명명된 채널입니다. Producer는 이벤트를 토픽에 발행(publish)합니다. **Subscription(구독)**은 토픽과 Consumer 사이의 지속적인 연결입니다. Broker는 각 Subscriber가 어떤 메시지까지 수신했는지 추적합니다. 새로운 Subscriber가 추가되면, (Kafka와 같은 시스템에서는) 히스토리의 처음부터 메시지를 받을 수도 있고, 이후에 생성되는 새로운 메시지만 받을 수도 있습니다.

## Fan-Out(팬아웃) 패턴

**Fan-out(팬아웃)**은 Pub/Sub의 핵심 기능입니다. 주문 서비스에서 발생한 단일 이벤트가 이메일 확인, 재고 차감, 분석 기록, 포인트 계산을 동시에 트리거할 수 있습니다. 이때 주문 서비스는 이러한 하류 시스템들의 존재 여부를 알 필요가 없습니다. 새로운 하류 작업을 추가하더라도 Producer를 수정할 필요가 전혀 없습니다.

📌
실제 사례: AWS SNS + SQS Fan-out

일반적인 AWS 패턴은 SNS 토픽에 발행하고, 이를 여러 개의 SQS 큐(하류 서비스당 하나씩)로 팬아웃하는 것입니다. 각 서비스는 고유한 재시도 및 Dead-letter(데드 레터) 설정을 가진 자신만의 큐를 갖게 됩니다. 이를 통해 Pub/Sub의 팬아웃 이점과 Point-to-Point 큐의 내구성 및 서비스별 격리 이점을 모두 얻을 수 있습니다.

## 메시지 필터링(Message Filtering)

하나의 토픽이 다양한 이벤트 유형을 전달할 때, Consumer는 그중 일부에만 관심을 가질 수 있습니다. **Message filtering(메시지 필터링)**을 통해 Subscriber는 자신이 원하는 메시지를 선언할 수 있습니다. AWS SNS는 메시지 속성에 대한 필터 정책을 지원합니다. Kafka Consumer는 특정 토픽을 구독하거나 레코드 헤더를 사용할 수 있습니다. Google Pub/Sub은 속성 표현식을 사용한 필터링된 구독을 지원합니다.

```json
// 구독에 대한 AWS SNS 필터 정책 예시
// eventType이 "order-placed"이고 amount가 100보다 큰 메시지만 전달
{
  "eventType": ["order-placed"],
  "amount": [{ "numeric": [">", 100] }]
}
```

## Durable(지속성) vs Non-Durable(비지속성) 구독

**Durable subscription(지속성 구독)**은 Subscriber가 오프라인 상태일 때 메시지를 보관했다가 다시 연결되면 전달합니다. **Non-durable subscription(비지속성 구독)**은 활성 상태로 연결되어 있을 때 발행된 메시지만 수신하며, 다운타임 동안 발행된 메시지는 유실됩니다. 운영 시스템에서는 거의 항상 지속성 구독을 사용합니다.

## 구독자 그룹 내의 경쟁 소비자(Competing Consumers Within a Subscriber Group)

Pub/Sub과 Point-to-Point는 상호 배타적이지 않습니다. Kafka의 **Consumer group(소비자 그룹)**은 이 두 가지를 우아하게 결합합니다. 서로 다른 그룹의 모든 Subscriber는 모든 메시지를 받지만(Pub/Sub 팬아웃), 한 그룹 내에서는 각 메시지가 하나의 인스턴스에 의해서만 처리됩니다(Point-to-Point 로드 밸런싱). 이를 통해 전체 팬아웃을 유지하면서 Subscriber를 수평적으로 확장할 수 있습니다.

## Pub/Sub을 선택해야 할 때

- 여러 독립적인 서비스가 동일한 이벤트에 반응해야 할 때
- Producer를 수정하지 않고 새로운 하류 서비스를 추가하고 싶을 때
- 서비스들이 완전히 분리된 Event-driven architecture(이벤트 기반 아키텍처)가 필요할 때
- 브로드캐스트 알림(연결된 모든 사용자나 모든 지역 노드로 전송)이 필요할 때

> 💡
> 인터뷰 팁
> 면접관이 여러 서비스에 이벤트를 어떻게 알릴 것인지 묻는다면 즉시 Pub/Sub을 언급하세요. 핵심 문구: "토픽에 이벤트를 발행하여 각 하류 서비스가 독립적으로 구독하게 함으로써 결합도를 없애고, Producer 변경 없이 새로운 서비스를 추가할 수 있게 하겠습니다." 그런 다음 내구성을 위해 SNS+SQS 팬아웃 패턴을 언급하십시오.
