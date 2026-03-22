# Message Queue Fundamentals(메시지 큐 기본 개념)

> 출처: https://www.sysdesai.com/learn/message-queues/message-queue-fundamentals

---

## Message Queue(메시지 큐)란 무엇인가요?

**Message Queue(메시지 큐)**는 **Producer(생성자)**(작업을 생성하는 서비스)와 **Consumer(소비자)**(작업을 수행하는 서비스) 사이에 위치하는 내구성이 있는 버퍼입니다. Producer가 Consumer를 직접 호출하고 응답을 기다리는 대신, 메시지를 큐에 넣고 다음 작업을 계속합니다. Consumer는 자신의 속도에 맞춰 큐에서 메시지를 읽어 처리합니다. 이 단순한 개념은 건축학적으로 매우 큰 이점들을 제공합니다.

우편 시스템을 생각하면 쉽습니다. 편지(메시지)를 써서 우체통(큐)에 넣고 일상 생활을 계속합니다. 수신인(소비자)은 준비가 되었을 때 편지를 가져갑니다. 여러분은 수신인이 문을 열어줄 때까지 문 앞에 서서 기다리지 않습니다.

## 핵심 구성 요소(Core Components)

| 구성 요소 | 역할 | 예시 |
| --- | --- | --- |
| Producer(생성자) | 메시지를 생성하고 발행함 | 주문 생성 이벤트를 보내는 주문 서비스 |
| Broker(브로커) | 메시지를 저장하고 전달함 | RabbitMQ, SQS, Kafka 클러스터 |
| Queue(큐) / Topic(토픽) | 메시지를 보관하는 명명된 채널 | orders-queue, payment-events |
| Consumer(소비자) | 메시지를 읽고 처리함 | 주문 알림을 처리하는 이메일 서비스 |
| Message(메시지) | 데이터 단위: 헤더 + 바디 | orderId, userId, amount가 포함된 JSON 페이로드 |

## Message Queue(메시지 큐)를 사용하는 이유

- **Decoupling(결합도 낮추기)** — Producer와 Consumer가 독립적으로 발전할 수 있습니다. 새로운 Consumer를 추가할 때 Producer를 변경할 필요가 없습니다.
- **Load leveling(부하 분산)** — 트래픽 급증을 큐가 흡수합니다. Consumer는 압도당하지 않고 감당 가능한 속도로 메시지를 처리합니다.
- **Resilience(회복 탄력성)** — Consumer에 장애가 발생해도 메시지는 큐에서 기다립니다. 작업이 유실되지 않습니다.
- **Async processing(비동기 처리)** — 이미지 크기 조정, 이메일 전송, 리포트 생성과 같이 시간이 오래 걸리는 작업이 HTTP 응답을 차단하지 않습니다.
- **Rate limiting(속도 제한)** — 외부 API와 같이 하류 시스템으로 들어가는 작업의 속도를 제어할 수 있습니다.

## 전송 보장(Delivery Guarantees)

메시징에서 가장 중요한 개념 중 하나는 **Delivery guarantee(전송 보장)**입니다. 이는 Broker가 메시지를 전송할지, 그리고 몇 번 전송할지에 대해 제공하는 약속입니다.

| 보장 수준 | 설명 | 위험 요소 | 예시 시스템 |
| --- | --- | --- | --- |
| At-most-once(최대 한 번) | 메시지를 한 번만 보냄, 재시도 없음. 유실될 수 있음. | 메시지 유실 | UDP, 파이어 앤 포겟 로그 |
| At-least-once(최소 한 번) | 메시지를 한 번 이상 전송함. 유실 없음. | 중복 처리 | SQS, RabbitMQ(기본값), Kafka(기본값) |
| Exactly-once(정확히 한 번) | 메시지를 정확히 한 번만 전송함. 유실과 중복 없음. | 복잡성, 낮은 처리량 | 트랜잭션을 사용하는 Kafka, SQS FIFO + 중복 제거 |

> ⚠️
> Exactly-once는 비용이 많이 듭니다.
> Exactly-once 전송은 Broker와 Consumer 간의 조정이 필요하며, 종종 2단계 커밋이나 멱등성 키를 사용합니다. 실무에서는 대부분의 시스템이 **At-least-once(최소 한 번) 전송**을 사용하고 Consumer를 **Idempotent(멱등성)**하게 설계하여 중복 메시지를 안전하게 처리합니다.

## 확인 응답 모드(Acknowledgment Modes)

**Acknowledgment(확인 응답/Ack)**는 Consumer가 Broker에게 "메시지를 성공적으로 처리했으니 삭제해도 좋다"고 알리는 방법입니다. Consumer가 Ack를 보내기 전에 장애가 발생하면, Broker는 다른 Consumer에게 해당 메시지를 다시 전달합니다. 이것이 At-least-once 전송의 핵심 메커니즘입니다.

일부 시스템은 **Negative acknowledgment(부정 확인 응답/Nack)**를 지원합니다. 이는 Consumer가 Broker에게 메시지 처리에 실패했음을 알리는 것으로, Broker는 메시지를 다시 큐에 넣거나 Dead letter queue(데드 레터 큐)로 보낼 수 있습니다.

## 메시지 가시성 및 잠금(Message Visibility and Locking)

Amazon SQS와 같은 시스템에서는 Consumer가 메시지를 읽으면 설정된 **Visibility timeout(가시성 타임아웃)**(예: 30초) 동안 다른 Consumer에게 **보이지 않게(invisible)** 됩니다. 이는 두 Consumer가 동시에 같은 메시지를 처리하는 것을 방지합니다. Consumer가 타임아웃 내에 Ack를 보내지 않으면 메시지는 다시 다른 Consumer에게 나타납니다. 이것이 실제 작동하는 At-least-once 전송 방식입니다.

> 💡
> 가시성 타임아웃을 현명하게 설정하세요.
> 가시성 타임아웃을 예상 처리 시간의 최소 2~3배로 설정하십시오. 평균 처리 시간이 10초지만 최대 45초까지 늘어날 수 있는 경우, 30초 타임아웃을 설정하면 불필요한 재전송과 중복 처리가 발생하게 됩니다.

## 큐를 사용하지 말아야 할 때

- 클라이언트가 **즉각적인 응답을 필요로 할 때** — 동기식 HTTP가 더 단순하고 적절합니다.
- 작업이 **매우 빠르고 가벼울 때** — 1밀리초 미만의 작업에 큐 오버헤드를 감수할 가치가 없습니다.
- **강한 순서 보장이 필수적**이고 파티셔닝된 순서의 복잡성을 감당할 수 없을 때.
- 시스템 규모가 작아 Broker 운영 오버헤드를 정당화하기 어려울 때.

> 💡
> 인터뷰 팁
> 면접관들은 "서비스를 직접 호출하지 않고 왜 메시지 큐를 사용하나요?"라는 질문을 좋아합니다. 핵심 답변은 **Decoupling(결합도 낮추기) + Resilience(회복 탄력성) + Load leveling(부하 분산)**입니다. 또한 트레이드오프에 대해서도 설명할 준비를 하세요. 비동기 처리를 얻는 대신 동기식 응답 보장을 잃고 운영 복잡성이 증가합니다. At-least-once 전송에 대한 실무적인 해결책으로 Idempotency(멱등성)를 언급하세요.
