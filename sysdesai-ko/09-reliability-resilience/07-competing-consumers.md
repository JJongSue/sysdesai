# Competing Consumers 패턴

> Source: https://www.sysdesai.com/learn/reliability-resilience/competing-consumers

---

## Competing Consumers란 무엇인가?

**Competing Consumers** 패턴은 단일 큐에 대해 여러 소비자(consumer) 인스턴스를 실행합니다. 각 소비자는 독립적으로 메시지를 폴링(polling)하며, 큐는 각 메시지가 정확히 하나의 소비자에게 전달되도록 보장합니다('at most once' 간극을 메우는 멱등성 처리를 포함한 at-least-once 전달). 소비자들은 작업을 위해 서로 경쟁하며, 가장 빠른 소비자가 가장 많은 메시지를 처리하게 되어 풀(pool)이 자체적으로 부하 분산(self-load-balancing)을 수행합니다.

이 패턴은 메시지 처리의 수평적 확장(horizontal scaling)을 가능하게 합니다. 소비자 수를 두 배로 늘리면 처리량도 약 두 배가 됩니다(큐의 처리량 한계 내에서). 이는 작업 큐(task queue), 잡 프로세서(job processor), 비동기 워커 풀(async worker pool)에서 지배적인 패턴입니다.

## 메시지 가시성 및 승인 (Message Visibility and Acknowledgment)

소비자가 메시지를 받으면, 큐는 설정된 **visibility timeout**(예: Amazon SQS의 경우 30초) 동안 다른 소비자가 해당 메시지를 볼 수 없도록 **숨김(invisible)** 상태로 만듭니다. 소비자가 해당 시간 내에 메시지를 처리하고 승인(acknowledgment)하면 메시지는 큐에서 삭제됩니다. 소비자가 승인하기 전에 중단되거나 타임아웃되면, 메시지는 다시 가시화되어 다른 소비자가 이를 가져갈 수 있게 됩니다.

> ⚠️
> Visibility Timeout을 처리 시간보다 길게 설정하세요.
> 처리 시간이 25초인데 visibility timeout이 20초라면, 처리 도중에 메시지가 다시 가시화되어 중복 처리가 발생합니다. Visibility timeout을 예상 처리 시간의 3~5배로 설정하거나, 작업이 진행됨에 따라 프로그래밍 방식으로 이를 연장하세요.

Competing consumers: 각 메시지는 정확히 하나의 소비자에게 전달됨; ack 시 메시지 삭제

## 순서 보장 (Ordering Guarantees)

표준 큐(SQS Standard, 대부분의 RabbitMQ 설정)는 여러 소비자가 실행 중일 때 메시지 순서를 보장하지 **않습니다**. 더 빠른 소비자가 나중 메시지를 더 느린 소비자가 이전 메시지를 끝내기 전에 처리할 수 있기 때문입니다. 순서가 중요하다면(예: 사용자 이벤트가 순서대로 처리되어야 함) 다음 접근 방식 중 하나를 사용하세요.

1. **단일 소비자(Single consumer)** — 오직 하나의 소비자만 큐를 처리 (병렬성 상실)
2. **FIFO 큐** — SQS FIFO는 메시지 그룹 ID 내에서 순서를 보장 (하지만 처리량이 3,000 msg/s로 제한됨)
3. **파티션된 토픽(Partitioned topics)** — Kafka는 키(key)별로 메시지를 파티셔닝합니다. 동일한 키를 가진 모든 메시지는 동일한 파티션으로 가고, 한 소비자 그룹 멤버에 의해 순서대로 처리됩니다.
4. **애플리케이션 수준의 시퀀싱** — 시퀀스 번호를 포함하고 소비자가 버퍼를 사용하여 순서가 어긋난 처리를 핸들링합니다.

## 파티션 기반 병렬성 (Kafka Consumer Groups)

Apache Kafka는 다른 competing consumers 모델을 사용합니다. 개별 메시지가 아닌 파티션(partition)이 소비자에게 할당됩니다. 각 파티션은 한 번에 소비자 그룹 내의 정확히 하나의 소비자에 의해 소비됩니다. 이는 파티션 내의 순서를 보장하면서 파티션 간의 병렬 처리를 가능하게 합니다. 활성 소비자 수는 파티션 수로 제한되며, 파티션보다 소비자가 많으면 일부 소비자는 유휴 상태가 됩니다.

Kafka consumer group: 파티션이 소비자에게 할당됨; 각 파티션 내의 순서가 보장됨

## 멱등성 소비자 (Idempotent Consumers)

큐는 **at-least-once** 전달을 보장하므로(exactly-once가 아님), 메시지가 두 번 이상 처리될 수 있습니다(예: 소비자가 처리 후 승인 직전에 중단됨). 소비자는 반드시 **멱등성(idempotent)**을 가져야 합니다. 즉, 동일한 메시지를 두 번 처리해도 한 번 처리했을 때와 동일한 결과가 나와야 합니다. 기법: 데이터베이스에서 처리된 메시지 ID 추적, insert 대신 database upsert 사용, 또는 작업 자체를 멱등적으로 만들기(값을 설정하는 것은 멱등적이지만 증분시키는 것은 아님).

> 💡
> 인터뷰 팁
> 면접에서 competing consumers 패턴은 비동기 워커를 확장해야 할 때 등장합니다. 언급해야 할 핵심 사항: visibility timeout 보정, at-least-once 전달을 위한 멱등성, 순서 보장의 트레이드오프(standard vs FIFO vs Kafka partitions), 그리고 큐 깊이에 따른 소비자 자동 확장입니다. 또한 데드 레터 큐(dead letter queue, DLQ)도 언급하세요. 반복적으로 실패하는 메시지는 메인 큐를 막지 않도록 조사를 위해 DLQ로 이동시켜야 합니다.

## 소비자 자동 확장 (Auto-Scaling Consumers)

소비자 수는 큐의 깊이에 따라 확장되어야 합니다. 일반적인 접근 방식: CloudWatch 알람(AWS)이나 커스텀 메트릭을 사용하여 큐 깊이가 임계값을 초과할 때(예: depth > 1000) Auto Scaling Group을 확장(scale-out)하고, 깊이가 0으로 떨어지면 축소(scale-in)합니다. 커스텀 메트릭(큐 깊이 / 소비자 수)을 사용한 Target-tracking scaling은 목표하는 소비자당 메시지 비율을 유지하여 진동 없이 매끄러운 확장을 제공합니다.
