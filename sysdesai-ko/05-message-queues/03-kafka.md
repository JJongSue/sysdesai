# Apache Kafka Deep Dive(아파치 카프카 심층 탐구)

> 출처: https://www.sysdesai.com/learn/message-queues/kafka

---

## Kafka(카프카)의 차별점

Apache Kafka(아파치 카프카)는 전통적인 Message Queue(메시지 큐)가 아닙니다. 이는 **Distributed commit log(분산 커밋 로그)**로, Append-only(추가 전용) 방식의 정렬되고 내구성이 있는 레코드 시퀀스입니다. 메시지 소비 후 삭제되는 RabbitMQ나 SQS와 달리, Kafka는 설정된 Retention period(보관 기간, 일 단위, 주 단위 또는 계층형 스토리지를 통한 영구 보관) 동안 메시지를 유지합니다. Consumer(소비자)는 **Offsets(오프셋)**을 사용하여 자신의 위치를 직접 추적합니다. 이는 메시지 재처리 및 재생 방식에 근본적인 변화를 가져옵니다.

## 핵심 아키텍처(Core Architecture)

Kafka 클러스터: Topic(토픽)은 Partition(파티션)으로 나뉘며, 각 파티션은 하나의 Leader(리더) 브로커와 N-1개의 Follower(팔로워)를 가집니다. Consumer group(소비자 그룹)은 각각 독립적인 오프셋을 가집니다.

## 토픽(Topics)과 파티션(Partitions)

Kafka **Topic(토픽)**은 하나 이상의 **Partition(파티션)**으로 나뉩니다. 각 파티션은 정렬되고 변경 불가능한 로그입니다. Producer(생성자)는 파티션에 데이터를 씁니다(기본적으로 라운드 로빈 방식 또는 파티션 키 기반). Consumer는 로그의 정수 인덱스인 **Offset(오프셋)**을 사용하여 순차적으로 파티션에서 데이터를 읽습니다.

파티션은 병렬성(Parallelism)의 단위입니다. 12개의 파티션과 그룹 내 12개의 Consumer 인스턴스가 있다면 12배의 병렬 소비가 가능합니다. 그룹 내의 활성 Consumer 수는 파티션 수보다 많을 수 없으며, 남는 Consumer는 대기 상태가 됩니다.

> 💡
> 파티션 키 선택이 중요합니다.
> 파티션 키(예: `userId`, `orderId`)를 제공하면, Kafka는 이를 해싱하여 동일한 키를 가진 레코드를 항상 동일한 파티션으로 라우팅합니다. 이는 키 범위 내에서의 순서 보장을 보장합니다. 키가 없으면 Kafka는 파티션 간에 라운드 로빈 방식으로 분산하여 처리량은 높이지만 키별 순서는 보장하지 않습니다.

## 소비자 그룹과 오프셋(Consumer Groups and Offsets)

**Consumer group(소비자 그룹)**은 토픽 읽기 작업을 공유하는 Consumer 인스턴스들의 집합입니다. Kafka는 각 파티션을 그룹 내 정확히 하나의 Consumer에게 할당합니다. 인스턴스에 장애가 발생하면 Kafka는 **Rebalances(리밸런싱)**을 수행하여 해당 파티션을 다른 인스턴스에 재배당합니다. 각 그룹은 `__consumer_offsets`라는 내부 카프카 토픽에서 파티션별 **Committed offset(커밋된 오프셋)**을 독립적으로 추적합니다.

| 시나리오 | 파티션 수 | Consumer 인스턴스 수 | 결과 |
| --- | --- | --- | --- |
| 리소스 부족 | 4 | 2 | 각 Consumer가 2개의 파티션을 읽음 — 정상 작동 |
| 완벽한 매칭 | 4 | 4 | 각 Consumer가 1개의 파티션을 읽음 — 최대 병렬성 |
| 리소스 과다 | 4 | 8 | 4개 Consumer만 활성, 4개는 대기 — 리소스 낭비 |

## 복제 및 내구성(Replication and Durability)

각 파티션은 **Replication factor(복제 계수)**(일반적으로 3)를 가집니다. 하나의 브로커가 해당 파티션의 **Leader(리더)**가 되어 모든 읽기 및 쓰기를 처리합니다. 다른 브로커들은 리더로부터 복제하는 **Followers(팔로워)**가 됩니다. 리더에 장애가 발생하면 팔로워 중 하나가 새 리더로 선출됩니다. Producer의 **`acks`** 설정은 내구성을 제어합니다.

| acks 설정 | 동작 방식 | 위험 요소 |
| --- | --- | --- |
| acks=0 | Fire and forget(발송 후 망각) — 브로커 응답을 기다리지 않음 | 브로커 장애 시 메시지 유실 |
| acks=1 | 리더의 확인 응답만 기다림 | 복제 전 리더 장애 시 유실 가능성 |
| acks=all (-1) | 모든 In-sync replicas(ISR, 동기화된 복제본) 응답을 기다림 | 가장 안전함 — 복제본이 2개 이상일 때 유실 없음 |

## Exactly-Once Semantics(정확히 한 번 처리)

Kafka는 재시도로 인한 중복 쓰기를 방지하기 위해 **Idempotent producers(멱등성 생성자)**(`enable.idempotence=true`로 활성화)를 지원합니다. 엔드 투 엔드 정확히 한 번 처리를 위해, Kafka **Transactions(트랜잭션)**를 사용하면 여러 파티션에 원자적으로 쓰고 오프셋 커밋을 단일 원자 작업으로 수행할 수 있습니다. 이를 통해 Exactly-once(정확히 한 번) 보장과 함께 읽기-처리-쓰기 파이프라인(Kafka Streams 스타일)을 구현할 수 있습니다.

```java
// Exactly-once semantics를 사용하는 Kafka producer (Java 예시)
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9092");
props.put("enable.idempotence", "true");
props.put("acks", "all");
props.put("transactional.id", "my-transactional-id");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.initTransactions();

try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("output-topic", key, value));
    // 처리된 오프셋과 새 레코드를 원자적으로 커밋
    producer.sendOffsetsToTransaction(offsets, consumerGroupId);
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

## Kafka의 보관 모델 vs 전통적인 큐(Kafka's Retention Model vs Traditional Queue)

이 부분이 Kafka가 근본적으로 다른 점입니다. 전통적인 큐는 확인 응답 후 메시지를 삭제하지만, Kafka는 설정된 기간(기본 7일) 동안 메시지를 유지합니다. 이를 통해 다음과 같은 것들이 가능해집니다:

- **Replay(재생)** — 새로운 Consumer 서비스를 배포할 때 처음부터 모든 이벤트를 재처리할 수 있습니다.
- **다중 독립 소비자** — 각 Consumer group은 자신만의 오프셋을 가집니다. 10개의 서로 다른 팀이 동일한 토픽을 독립적으로 읽을 수 있습니다.
- **Time-travel debugging(시점 이동 디버깅)** — 운영 문제를 진단하기 위해 특정 타임스탬프로 Consumer를 되돌릴 수 있습니다.
- **Event sourcing(이벤트 소싱)** — Kafka 토픽 자체가 Source of truth(진실의 원천)가 되며, 데이터베이스는 하나의 투영(projection)이 됩니다.

## Kafka를 선택해야 할 때

| 이럴 때 Kafka를 선택하세요... | 이럴 때 전통적인 큐를 선택하세요... |
| --- | --- |
| 높은 처리량(초당 수백만 개 이벤트) | 적당한 처리량으로 충분할 때 |
| 여러 독립적인 Consumer가 동일한 이벤트 필요 | 메시지당 하나의 Consumer면 충분할 때 |
| 재생(Replay) 및 재처리가 필요할 때 | 처리 후 메시지를 버려도 될 때 |
| 이벤트 소싱 / 감사 로그 유스케이스 | 단순한 태스크 큐(작업, 백그라운드 작업) |
| Kafka Streams를 통한 스트림 처리 | 복잡한 라우팅 로직(RabbitMQ exchanges) 필요 |

> ⚠️
> Kafka는 운영 부담이 큽니다.
> Kafka는 ZooKeeper(또는 최신 버전의 KRaft), 신중한 파티션 설계, Consumer lag(소비자 지연) 모니터링, 보관 및 복제 튜닝이 필요합니다. 관리형 서비스(Amazon MSK, Confluent Cloud)를 통해 이 부담을 크게 줄일 수 있지만, 단순한 유스케이스에서는 여전히 SQS나 RabbitMQ보다 무겁습니다.

> 💡
> 인터뷰 팁
> Kafka는 시스템 디자인 면접에서 단골 질문입니다. 강조해야 할 핵심 포인트: (1) 단순한 큐가 아니라 메시지가 유지되는 분산 커밋 로그이다. (2) 파티션은 병렬성의 단위이며, 파티션을 추가하여 Consumer를 확장한다. (3) Consumer group을 통해 여러 독립적인 서비스가 동일한 이벤트를 읽을 수 있다. (4) 키 범위 내의 순서 보장을 위해 파티션 키를 사용한다. "Kafka vs SQS" 질문을 받는다면: Kafka는 처리량, 재생 가능성, 다중 소비자 측면에서 유리하고, SQS는 단순성과 관리 편의성 측면에서 유리하다고 답변하세요.

연습해 보기
[Kafka를 사용한 실시간 이벤트 처리 파이프라인 설계](https://www.sysdesai.com/design/new?prompt=Design%20a%20real-time%20event%20processing%20pipeline%20using%20Kafka&mode=fast)
