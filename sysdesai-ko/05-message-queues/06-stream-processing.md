# Stream Processing(스트림 처리)

> 출처: https://www.sysdesai.com/learn/message-queues/stream-processing

---

## Batch Processing(배치 처리) vs Stream Processing(스트림 처리)

**Batch processing(배치 처리)**는 일정 기간(한 시간, 하루 등) 동안 데이터를 모아서 한꺼번에 처리합니다. 효율적이고 단순하지만 지연 시간이 발생합니다. 다음 배치가 실행될 때까지는 "지난 5분 동안 얼마나 많은 사용자가 가입했는가?"와 같은 질문에 답할 수 없습니다. **Stream processing(스트림 처리)**은 데이터가 도착하는 즉시 레코드별로 처리하여 실시간 분석, 알림 및 지속적인 연산을 가능하게 합니다.

| 비교 항목 | Batch Processing(배치 처리) | Stream Processing(스트림 처리) |
| --- | --- | --- |
| 지연 시간(Latency) | 분 ~ 시간 단위 | 밀리초 ~ 초 단위 |
| 처리량(Throughput) | 매우 높음 (대규모 데이터셋) | 높음 (지속적) |
| 복잡성 | 낮음 — 단순한 작업 | 높음 — 상태 유지, 윈도잉 필요 |
| 유스케이스 | 일간 리포트, ETL, ML 학습 | 실시간 대시보드, 부정 탐지, 알림 |
| 예시 | Spark batch, Hadoop MapReduce | Kafka Streams, Flink, Spark Streaming |

## 윈도잉(Windowing)

스트림 처리는 종종 "분당 클릭 수 계산", "시간당 평균 주문 금액 계산"과 같이 시간 윈도우를 기준으로 이벤트를 집계해야 합니다. **Windowing(윈도잉)**은 무한한 스트림을 집계를 위한 유한한 덩어리로 나눕니다.

| 윈도우 유형 | 설명 | 예시 |
| --- | --- | --- |
| Tumbling(텀블링) | 고정된 크기, 겹치지 않는 간격 | 5분마다 이벤트 수 계산 (0:00-0:05, 0:05-0:10) |
| Sliding(슬라이딩) | 고정된 크기, 슬라이드 간격만큼 이동 — 윈도우가 겹침 | 최근 5분간의 이벤트를 1분마다 업데이트하며 계산 |
| Session(세션) | 동적 — 활동 간의 공백(gap)을 기준으로 그룹화 | 30분 미만의 비활성 간격을 가진 사용자의 클릭을 모두 그룹화 |
| Hopping(호핑) | 고정된 크기지만 홉(hop) 간격으로 겹침 | 5분 윈도우를 1분마다 생성 — 겹치는 구간 발생 |

## Event Time(이벤트 시간) vs Processing Time(처리 시간)

**Processing time(처리 시간)**은 이벤트가 스트림 프로세서에 도착한 시점입니다. **Event time(이벤트 시간)**은 이벤트가 실제로 발생한 시점(이벤트 자체에 기록됨)입니다. 네트워크 지연, 모바일 기기의 배치 전송, 재시도 등으로 인해 이 둘은 서로 다를 수 있습니다. 정확한 분석을 위해서는 항상 이벤트 시간을 사용해야 하지만, 이는 **Late-arriving events(지연 도착 이벤트)**를 처리해야 함을 의미합니다.

> ℹ️
> Watermark(워터마크)가 지연 데이터를 처리합니다.
> **Watermark(워터마크)**는 휴리스틱 임계값입니다: "시간 T까지의 모든 이벤트가 도착했다고 판단하고 윈도우를 닫아 연산을 수행하겠다"는 의미입니다. 워터마크 이후에 도착하는 이벤트는 버려지거나, 지연 윈도우에 할당되거나, 윈도우 재계산을 트리거합니다. Apache Flink는 강력한 워터마크 기능을 지원합니다.

## Kafka Streams(카프카 스트림즈)

**Kafka Streams(카프카 스트림즈)**는 카프카 토픽에서 레코드를 처리하고 결과를 다시 카프카 토픽에 쓰는 클라이언트 라이브러리(별도의 클러스터가 아님)입니다. 애플리케이션 프로세스 내에서 실행되므로 카프카 클러스터 외에 관리할 별도의 인프라가 필요하지 않습니다.

```java
// Kafka Streams: 5분 텀블링 윈도우로 제품별 주문 수 계산
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Order> orders = builder.stream("orders");

orders
  .groupBy((key, order) -> order.getProductId())
  .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
  .count()
  .toStream()
  .to("order-counts-per-product");

KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();
```

Kafka Streams는 집계나 조인과 같은 상태 유지(stateful) 작업을 위해 **State stores(상태 저장소)**(기본적으로 RocksDB)를 사용합니다. 상태는 카프카 체인지로그(changelog) 토픽에 백업되어 노드 장애 시 복구가 가능합니다.

## Apache Flink(아파치 플링크)

**Apache Flink(아파치 플링크)**는 대규모의 상태 유지 연산을 위해 설계된 모든 기능을 갖춘 스트림 처리 프레임워크입니다. 앱 내에서 실행되는 Kafka Streams와 달리, Flink는 자체 클러스터(또는 Kubernetes 배포)를 가진 분산 실행 엔진입니다. Flink는 복잡한 이벤트 처리, ML 추론 파이프라인, 그리고 다양한 소스 간의 Exactly-once(정확히 한 번) 처리에 탁월합니다.

| 기능 | Kafka Streams | Apache Flink |
| --- | --- | --- |
| 배포 방식 | 라이브러리 — 앱 내 실행 | 별도의 클러스터 또는 Kubernetes |
| 데이터 소스 | Kafka 전용 | Kafka, S3, DB, HTTP, 커스텀 등 |
| 상태 관리 | RocksDB (로컬) | RocksDB + 원격 체크포인트 |
| Exactly-once | 지원 (Kafka to Kafka) | 지원 (다양한 소스 간) |
| 복잡성 | 낮음 — 단순한 API | 높음 — 하지만 더 강력함 |
| 최적의 용도 | Kafka 네이티브 파이프라인 | 복잡한 이벤트 처리, 다중 소스 ETL |

## Stream-Table Duality(스트림-테이블 이원성)

스트림 처리의 심오한 개념: **모든 스트림은 테이블(현재 상태)로 볼 수 있고, 모든 테이블은 스트림(체인지로그)으로 볼 수 있습니다.** 사용자 업데이트 이벤트가 담긴 카프카 토픽은 스트림이지만, 이를 userId별로 최신 이벤트만 남기도록 압축하면 테이블(Kafka Streams의 **KTable**)이 됩니다. 이 이원성을 통해 스트림-테이블 조인이 가능해집니다. 즉, 주문 스트림을 KTable에 있는 최신 사용자 프로필 정보와 실시간으로 결합할 수 있습니다.

📌
실제 사례: 이벤트 강화(Enriching Events)

클릭 이벤트 스트림과 사용자 프로필 테이블(사용자 토픽에서 구체화됨)이 있다고 가정해 봅시다. 스트림-테이블 조인을 통해 각 클릭 이벤트에 사용자의 국가와 요금제 정보를 실시간으로 추가할 수 있습니다. 데이터베이스를 일일이 조회할 필요가 없습니다.

## 실시간 분석 아키텍처(Real-Time Analytics Architecture)

Lambda 아키텍처가 없는 실시간 분석: Kafka가 스트림 프로세서에 데이터를 공급하고, 프로세서는 집계된 데이터를 대시보드용 OLAP 저장소에 쓰고 원본 이벤트는 데이터 레이크에 저장합니다.

> 💡
> 인터뷰 팁
> 면접에서 스트림 처리는 "실시간 리더보드 설계"나 "라이브 이벤트 분석 설계" 질문에서 자주 등장합니다. 언급해야 할 핵심 개념: (1) 이벤트 소스로서의 Kafka. (2) 스트림 프로세서 (단순한 경우 Kafka Streams, 대규모인 경우 Flink). (3) 주기적 집계를 위한 텀블링 윈도우. (4) 저지연 쿼리를 위한 OLAP 데이터베이스 (ClickHouse, Druid, BigQuery). 트레이드오프를 설명하세요: Exactly-once는 비용이 많이 들며, 일반적인 집계에는 At-least-once + Idempotent(멱등성) 집계로도 충분합니다.
