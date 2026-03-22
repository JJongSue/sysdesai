# RabbitMQ & Traditional Message Brokers(RabbitMQ와 전통적인 메시지 브로커)

> 출처: https://www.sysdesai.com/learn/message-queues/rabbitmq

---

## RabbitMQ 아키텍처(RabbitMQ Architecture)

RabbitMQ는 강력한 라우팅 모델을 기반으로 설계된 **AMQP 기반**의 메시지 브로커입니다. Kafka의 Producer(생성자)가 Partition(파티션)에 직접 데이터를 쓰는 모델과 달리, RabbitMQ는 Indirection layer(간접 계층)를 도입했습니다. Producer는 **Exchanges(익스체인지)**에 메시지를 발행하고, Exchange는 **Bindings(바인딩)**를 통해 메시지를 **Queues(큐)**로 라우팅하며, Consumer(소비자)는 큐를 구독합니다. 이러한 라우팅 유연성이 RabbitMQ의 가장 큰 특징이자 강점입니다.

RabbitMQ Topic exchange: 라우팅 키 패턴(*: 단어 하나, #: 0개 이상의 단어)을 기반으로 메시지를 라우팅함.

## 익스체인지 유형(Exchange Types)

| 익스체인지 유형 | 라우팅 로직 | 유스케이스 |
| --- | --- | --- |
| Direct(다이렉트) | 라우팅 키의 정확한 일치로 라우팅 | 단순 큐 분산, Point-to-Point |
| Fanout(팬아웃) | 바인딩된 모든 큐에 브로드캐스트(키 무시) | 알림, 캐시 무효화 |
| Topic(토픽) | 패턴 매칭(*, #)으로 라우팅 | 카테고리 기반 라우팅, 멀티 테넌트 시스템 |
| Headers(헤더) | 메시지 헤더 속성으로 라우팅 | 키 구조 없는 복잡한 필터링 |

## 메시지 확인 응답 및 내구성(Message Acknowledgment and Durability)

RabbitMQ는 **Manual acknowledgment(수동 확인 응답)**(Consumer가 처리를 마친 후 명시적으로 Ack를 보냄)와 **Automatic acknowledgment(자동 확인 응답)**(전달 시점에 Ack 처리됨)를 지원합니다. 내구성을 확보하려면 **Durable queues(지속성 큐)**(브로커 재시작 시 유지됨)와 **Persistent messages(지속성 메시지)**(디스크에 기록됨)가 모두 필요합니다. 둘 중 하나라도 누락되면 재시작 시 메시지가 유실됩니다.

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# 지속성 큐 선언 (브로커 재시작 시 유지됨)
channel.queue_declare(queue='tasks', durable=True)

def callback(ch, method, properties, body):
    print(f"Processing: {body}")
    # ... 작업 수행 ...
    # 성공적으로 처리된 후 수동으로 Ack를 보냄
    ch.basic_ack(delivery_tag=method.delivery_tag)

# prefetch_count=1: 현재 메시지에 대한 Ack가 올 때까지 새 메시지를 보내지 않음
channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue='tasks', on_message_callback=callback)
channel.start_consuming()
```

> 💡
> 공정한 배분(Fair dispatch)을 위해 basic_qos(prefetch_count=1)를 사용하세요.
> 기본적으로 RabbitMQ는 Consumer의 작업 상태를 모른 채 라운드 로빈 방식으로 메시지를 배분합니다. prefetch_count=1을 설정하면 RabbitMQ는 Consumer에게 아직 확인 응답이 오지 않은 메시지를 한 개 이상 보내지 않습니다. 이를 통해 바쁜 Consumer에게 작업이 쌓이고 한가한 Consumer가 노는 현상을 방지할 수 있습니다.

## RabbitMQ vs Kafka: 정면 비교

| 비교 항목 | RabbitMQ | Kafka |
| --- | --- | --- |
| 패러다임 | 똑똑한 브로커, 단순한 소비자 | 단순한 브로커, 똑똑한 소비자 |
| 메시지 보관 | 확인 응답 후 삭제 | 설정된 기간 동안 유지 |
| 순서 보장 | 큐별 FIFO | 파티션별 순서 보장 |
| 처리량 | 노드당 약 초당 5만 개 | 초당 수백만 개 |
| 라우팅 | 풍부함(익스체인지, 바인딩) | 단순함(토픽 + 파티션 키) |
| 재생(Replay) | 지원하지 않음 | 보관 기간 내 전체 재생 가능 |
| Consumer 추적 | 브로커가 Ack를 추적함 | Consumer가 오프셋을 추적함 |
| 최적의 용도 | 태스크 큐, 복잡한 라우팅, RPC | 이벤트 스트리밍, 높은 처리량, 재생 |

## Amazon SQS 및 Azure Service Bus

브로커를 직접 운영하지 않고 관리형 전통 큐를 사용하려는 팀에게는 클라우드 네이티브 옵션이 훌륭한 선택입니다. **Amazon SQS**는 표준형(최소 한 번 전달, 최선형 순서 보장)과 FIFO(정확히 한 번 처리, 엄격한 순서 보장, 초당 최대 300건) 변체를 제공하는 완전 관리형 큐 서비스입니다. **Azure Service Bus**는 세션, 데드 레터링, 메시지 지연, 예약 전달과 같은 풍부한 기능을 가진 큐와 토픽을 제공합니다.

> ℹ️
> SQS Standard(표준) vs FIFO
> SQS Standard 큐는 거의 무제한의 처리량을 제공하지만 메시지 순서가 바뀔 수 있고 가끔 중복이 발생할 수 있습니다. SQS FIFO 큐는 정확히 한 번 처리와 메시지 그룹 내의 엄격한 순서를 보장하지만 초당 300건(배치 처리 시 3,000건)으로 제한됩니다. 순서가 중요하지 않은 대량 처리 작업에는 Standard를, 금융 거래나 주문 순서 보장이 필요한 경우에는 FIFO를 선택하세요.

## RabbitMQ나 SQS를 선택해야 할 때

- **Task queues(태스크 큐)** — 각 작업이 한 번만 처리되어야 하는 백그라운드 작업(이미지 처리, 이메일 전송, PDF 생성)
- **Complex routing(복잡한 라우팅)** — 커스텀 코드 없이 내용이나 유형에 따라 메시지를 다른 Consumer에게 라우팅해야 할 때
- **RPC over messaging** — Consumer로부터 응답이 필요한 요청/응답 패턴
- **Moderate throughput(적당한 처리량)** — 초당 수만 개의 메시지 수준
- **Operational simplicity(운영 단순성)** — SQS는 브로커 관리가 전혀 필요 없으며, RabbitMQ는 Kafka보다 운영이 단순합니다.

> 💡
> 인터뷰 팁
> 면접에서 RabbitMQ와 Kafka를 비교할 때 핵심은 이렇습니다. RabbitMQ는 메시지 라우팅과 전달 확인을 수행하는 'Smart broker(똑똑한 브로커)'이고, Kafka는 로그에 데이터를 추가만 하고 Consumer가 스스로의 위치를 추적하게 하는 'Dumb broker(단순한 브로커)'입니다. 이로 인해 Kafka는 확장성이 더 뛰어나고, RabbitMQ는 라우팅 유연성이 더 높습니다. "태스크 큐"나 "백그라운드 작업"이라는 단어가 들리면 RabbitMQ/SQS를 제안하고, "이벤트 스트리밍", "높은 처리량", "재생"이 필요하다면 Kafka를 제안하세요.
