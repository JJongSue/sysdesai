# Fan-Out / Fan-In Pattern

> 원문: https://www.sysdesai.com/learn/messaging-patterns/fan-out-fan-in

---

## 핵심 개념

**Fan-Out**은 하나의 작업을 여러 워커(worker)에게 분산하여 하위 문제들을 병렬로 처리하게 하는 것입니다. **Fan-In**은 해당 워커들의 처리 결과를 다시 하나의 통합된 결과로 집계(aggregate)하는 것입니다. 이 둘은 함께 **Scatter-Gather** 패턴을 형성합니다. 즉, 작업을 흩뿌리고(scatter), 결과를 모으는(gather) 것입니다. 이는 메시징 세계에서의 병렬 처리 방식으로, MapReduce, 병렬 데이터베이스 쿼리 실행, 검색 엔진의 결과 머징과 동일한 개념입니다.

이 패턴은 개념적으로는 매우 간단해 보이지만, 실제 구현 시에는 여러 까다로운 과제가 있습니다: 모든 워커가 끝났는지 어떻게 아는가? 워커 하나가 실패하면 어떻게 하는가? 부분적인 결과를 어떻게 효율적으로 집계하는가? 지연되는 워커(stragglers)는 어떻게 처리하는가?

## Fan-Out / Fan-In Flow
Fan-Out / Fan-In: 하위 작업들을 병렬 워커에게 뿌리고(scatter), 결과를 애그리게이터(aggregator)에서 모읍니다(gather).

## 실전에서의 Fan-Out 패턴

Fan-out은 분산 대상에 따라 두 가지 주요 형태로 나타납니다:

| Fan-Out 유형 | 분산 대상 | 예시 | 매커니즘 |
| --- | --- | --- | --- |
| 알림(Notification) fan-out | 동일한 이벤트가 많은 수신자에게 전송됨 | Twitter/X 팔로워 알림, 수백만 사용자에게 푸시 알림 전송 | SNS → 리전/샤드별 SQS, Kafka 토픽 파티션, Redis Pub/Sub |
| 작업(Work) fan-out | 큰 작업이 병렬로 처리되는 하위 작업들로 분할됨 | 여러 인덱스 샤드에 걸친 검색, 배치 이미지 처리, MapReduce | 경쟁 소비자(competing consumers)가 있는 태스크 큐 (SQS/RabbitMQ), Kafka 파티션 |

## 대규모 알림 Fan-Out: 트위터 문제

1,000만 명의 팔로워를 가진 사용자가 트윗을 올린다고 가정해 봅시다. 단순히 생각해서 각 팔로워의 피드에 알림을 푸시한다면, 단 한 번의 사용자 액션에 대해 1,000만 번의 쓰기 작업이 발생합니다. 이것이 전형적인 fan-out 문제입니다. 두 가지 아키텍처적 접근 방식이 있습니다:

| 접근 방식 | 설명 | 장점 | 단점 |
| --- | --- | --- | --- |
| Fan-out on write (push 모델) | 포스팅 시점에 비동기 워커를 통해 모든 팔로워의 피드에 기록 | 읽기가 즉각적임 — 미리 계산된 피드 | 팔로워가 많은 사용자(연예인 등)에게 비용이 매우 많이 듦; 쓰기 증폭(write amplification) 발생 |
| Fan-out on read (pull 모델) | 읽기 시점에 팔로우한 사용자들의 타임라인을 머지하여 피드 계산 | 쓰기 증폭 없음 | 대량 팔로우 사용자의 경우 읽기 비용이 크고 느림; 캐싱 필요 |
| 하이브리드(Hybrid) | 일반 사용자는 fan-out on write, 연예인은 fan-out on read 사용 | 두 방식의 장점 결합 | 더 복잡함; 팔로워 수 임계치(threshold) 로직 필요 |

Fan-out on write: 트윗이 샤딩된 워커 큐를 통해 수백만 팔로워의 캐시된 피드에 비동기적으로 기록됩니다.

## 작업 Fan-Out: 집계(Aggregation)의 과제

Fan-out이 알림이 아닌 실제 작업을 분산할 때, fan-in 집계가 가장 까다로운 부분입니다. 오케스트레이터(orchestrator)는 최종 결과를 생성하기 전에 **모든** 하위 작업이 완료되었는지 알아야 합니다.

일반적인 집계 전략:

- **공유 상태의 카운터(Counter in shared state)**: 오케스트레이터가 Redis에 `expectedCount=N`을 기록하고, 각 워커가 원자적(atomic)으로 감소시킵니다. 카운터가 0이 되면 최종 조합을 트리거합니다. 빠르지만 원자적 연산이 필요합니다.
- **상관관계 추적 테이블(Correlation tracking table)**: 각 하위 작업을 상태와 함께 DB에 영구 저장하고 완료 여부를 쿼리합니다. 더 안정적이지만 느립니다.
- **Saga orchestrator 패턴**: 오케스트레이터가 명시적으로 어떤 하위 작업이 응답했는지 추적하고 fan-in 단계를 조정합니다.
- **Promise.all / Future 집계**: 코드 수준에서(메시징이 아닌 경우) 단순히 모든 병렬 future를 대기하고 결과를 수집합니다.

typescript

```
// Redis 기반 fan-in 조정 예시
async function fanOutSearch(query: string, shards: string[]): Promise<SearchResult[]> {
  const jobId = crypto.randomUUID();
  const expectedCount = shards.length;

  // 카운터 초기화
  await redis.set(`job:${jobId}:remaining`, expectedCount, "EX", 60);
  await redis.set(`job:${jobId}:results`, JSON.stringify([]), "EX", 60);

  // Fan-out: 각 샤드 워커에게 하위 쿼리 발송
  for (const shard of shards) {
    await queue.publish(`search.shard.${shard}`, { jobId, query });
  }

  // Fan-in 대기: 카운터가 0이 될 때까지 폴링
  return waitForCompletion(jobId, 30_000 /* 30초 timeout */);
}

// 각 워커는 자신의 샤드 검색을 완료한 후 이를 호출합니다:
async function workerComplete(jobId: string, partialResults: SearchResult[]) {
  const pipeline = redis.pipeline();
  // 결과 원자적 추가
  const current = JSON.parse(await redis.get(`job:${jobId}:results`) ?? "[]");
  pipeline.set(`job:${jobId}:results`, JSON.stringify([...current, ...partialResults]), "EX", 60);
  pipeline.decr(`job:${jobId}:remaining`);
  await pipeline.exec();
}
```

## 지연 워커(Stragglers) 및 부분 실패 처리

대규모 fan-out에서는 일부 워커가 느려지거나(stragglers) 실패하기 마련입니다. 이를 처리하기 위한 전략들은 다음과 같습니다:

- **부분 결과를 포함한 타임아웃(Timeouts with partial results)**: timeout 발생 시 `partial: true` 플래그와 함께 지금까지 확보한 결과만 반환합니다. 검색 엔진이 이 방식을 사용합니다. 인덱스 샤드 하나가 누락되는 것이 전체 timeout보다 낫기 때문입니다.
- **투기적 실행(Speculative execution / hedging)**: 지연 임계치를 넘으면 동일한 하위 작업을 두 번째 워커에게 보냅니다. 먼저 끝나는 쪽의 결과를 사용합니다. 구글의 'The Tail At Scale' 논문에서 이 방식이 대중화되었습니다.
- **DLQ를 활용한 재시도**: 실패한 하위 작업은 Dead Letter Queue로 보내 재시도하고, 오케스트레이터는 별도로 통지받습니다.
- **Idempotent 워커**: 하위 작업 처리를 idempotent하게 만들어 재시도가 안전하게 이루어지도록 합니다.

> 💡
> 99번째 백분위수 문제 (The 99th Percentile Problem)
> 100개의 하위 작업을 fan-out할 때, 전체 지연 시간은 가장 느린 워커에 의해 제한됩니다. 즉, 단일 워커의 99번째 백분위수 지연 시간이 fan-out의 중앙값(median) 지연 시간이 됩니다. 이를 고려하여 설계하세요: 워커별 timeout을 공격적으로 설정하고, 핵심 경로에 투기적 실행을 사용하며, 무한정 기다리기보다 부분 결과를 선호하세요.

## AWS SNS + SQS Fan-Out

전형적인 AWS fan-out 패턴은 **SNS**를 브로드캐스터로 사용하고, 서비스별로 내구성 있는 수신자인 **SQS 큐**를 사용하는 것입니다. 하나의 SNS 토픽이 여러 SQS 큐에 연결되며, 각 큐는 서로 다른 다운스트림 서비스가 소유합니다. 이를 통해 각 소비자는 독립적이고 내구성 있는 큐를 가지면서 동시에 이벤트 브로드캐스트를 받을 수 있습니다.

AWS SNS + SQS fan-out: 단일 SNS 토픽으로부터 각 다운스트림 서비스가 자신만의 내구성 있는 SQS 큐를 통해 데이터를 받습니다.

> 💡
> 인터뷰 팁
> Fan-out/fan-in은 알림 시스템, 검색, 배치 처리, 분석 등 거의 모든 대규모 설계 질문에 등장합니다. 알림의 경우 push vs pull 모델과 연예인/핫스팟 문제를 언급하세요. 작업 fan-out의 경우 완료 추적 방법(counter 패턴), 지연 워커 처리(timeout + 부분 결과), 그리고 정확히 한 번의 집계 보장(idempotent 워커) 방법을 설명하세요. 이러한 세부 사항들이 우수한 후보자를 가르는 기준이 됩니다.
