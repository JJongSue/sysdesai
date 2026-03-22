# Pipes and Filters 패턴

> Source: https://www.sysdesai.com/learn/performance-scalability/pipes-and-filters

---

## Pipes and Filters란?

**Pipes and Filters 패턴**은 복잡한 처리 작업을 **필터(Filter)**라 부르는 일련의 독립적인 처리 단계로 분해하고, 이를 **파이프(Pipe)**라는 데이터 채널로 연결합니다. 각 필터는 입력 파이프에서 데이터를 받아 변환을 수행한 후 결과를 출력 파이프에 씁니다. 파이프라인 내에서 앞뒤로 무엇이 있는지는 알지 못합니다. 이 아키텍처 패턴은 Unix 셸 파이프라인, Apache Kafka Streams, AWS Glue ETL 작업, 현대 ML 피처 파이프라인의 근간입니다.

## 핵심 구성요소

| 구성요소 | 역할 | 예시 |
| --- | --- | --- |
| Source(소스) | 파이프라인에 입력 데이터 생성 | Kafka 토픽, S3 버킷, HTTP 웹훅 |
| Filter(필터) | 데이터 변환, 검증, 보강 또는 라우팅 | 중복 제거 단계, 스키마 검증기, ML 분류기 |
| Pipe(파이프) | 필터 간 데이터 전송 | 각 단계 사이의 Kafka 토픽, 인메모리 큐, gRPC 스트림 |
| Sink(싱크) | 마지막 필터의 최종 출력 수신 | 데이터베이스 쓰기, 다운스트림 API, 출력 Kafka 토픽 |

데이터 보강 파이프라인: 각 필터는 독립적으로 개발, 테스트, 확장 가능

## 핵심 특성

- **조합 가능성(Composability)**: 필터는 재사용 가능한 빌딩 블록입니다. 스키마 검증 필터는 수정 없이 여러 파이프라인에서 사용할 수 있습니다.
- **병렬성(Parallelism)**: 독립적인 필터는 동시에 실행될 수 있습니다. 보강(enrichment)이 느리다면 해당 필터 인스턴스만 추가하면 됩니다.
- **교체 가능성(Replaceability)**: 파이프라인의 나머지를 변경하지 않고 특정 필터를 새 구현으로 교체할 수 있습니다.
- **테스트 용이성(Testability)**: 각 필터를 완전히 독립적으로 단위 테스트할 수 있습니다 — 입력 데이터를 주고 출력을 검증하면 됩니다.

## 동기 vs 비동기 파이프

파이프는 동기식(단일 서비스 내의 인프로세스 함수 호출로 처리 체인 형성) 또는 비동기식(서비스 간 메시지 큐나 Kafka 토픽)일 수 있습니다. 비동기 파이프를 사용하면 각 필터를 독립적으로 수평 확장 가능한 서비스로 만들 수 있고, 필터마다 처리량이 다를 때 자연스러운 버퍼링을 제공합니다. 트레이드오프로는 추가 지연(브로커 왕복 시간)과 복잡성(최소 1회 전달 보장, 멱등성)이 있습니다.

```typescript
// 타입이 지정된 필터를 사용하는 동기 파이프라인
type Filter<TIn, TOut> = (input: TIn) => TOut | null; // null은 메시지 드롭을 의미

function createPipeline<T>(...filters: Filter<T, T>[]): Filter<T, T> {
  return (input: T): T | null => {
    let current: T | null = input;
    for (const filter of filters) {
      if (current === null) return null;
      current = filter(current);
    }
    return current;
  };
}

// 개별 필터 정의
const validateSchema: Filter<Event, Event> = (event) =>
  event.userId && event.type ? event : null;

const deduplicate: Filter<Event, Event> = (event) =>
  seenIds.has(event.id) ? null : (seenIds.add(event.id), event);

const enrichWithUser: Filter<Event, Event> = (event) => ({
  ...event,
  user: userCache.get(event.userId),
});

// 파이프라인 조합
const pipeline = createPipeline(
  validateSchema,
  deduplicate,
  enrichWithUser
);

// 이벤트 처리
for (const rawEvent of eventStream) {
  const result = pipeline(rawEvent);
  if (result !== null) {
    sink.write(result);
  }
}
```

## 파이프라인의 오류 처리

오류 처리는 파이프라인 설계에서 가장 까다로운 부분입니다. 잡히지 않은 예외를 던지는 필터는 전체 파이프라인을 중단시킬 수 있습니다. 일반적인 전략들:

- **Dead Letter Queue (DLQ)**: N번 재시도 후에도 필터를 통과하지 못한 메시지는 조용히 삭제되는 대신 수동 검토를 위해 DLQ로 라우팅됩니다.
- **오류 싱크(Error Sink)**: 실패한 레코드를 받는 별도 출력 파이프로, 다운스트림 분석 및 재처리를 가능하게 합니다.
- **건너뛰고 로깅(Skip and Log)**: 중요하지 않은 보강(예: 사용자 데이터 조회)의 경우, 파이프라인 전체를 블로킹하는 대신 부분 데이터로 계속 진행합니다.
- **체크포인트 및 재실행(Checkpoint and Replay)**: 처리 위치(예: Kafka offset)를 기록해 충돌 후 마지막 성공 지점부터 파이프라인을 재시작할 수 있도록 합니다.

> 📌
> 실제 사례: Apache Kafka Streams
> Kafka Streams는 Pipes and Filters 패턴을 네이티브로 구현합니다. 각 `KStream.map()`, `.filter()`, `.flatMap()`, `.join()` 호출이 하나의 필터입니다. 토픽이 파이프 역할을 합니다. 토폴로지는 파티션에 걸쳐 배포되는 스테이트풀 처리 그래프로 컴파일됩니다. Netflix는 수십 개의 필터 단계로 실시간 추천 시그널 처리에 Kafka Streams를 사용합니다.

> 💡
> 인터뷰 팁
> 데이터 처리 시스템(ETL 파이프라인, 이벤트 처리, 미디어 트랜스코딩)을 설계할 때 Pipes and Filters 아키텍처를 제안하면 탄탄한 엔지니어링 판단력을 보여줍니다. 각 필터의 독립적인 확장 가능성, 테스트 용이성 장점, 비동기 파이프(Kafka 토픽)가 버퍼링을 제공하고 각 단계를 독립적으로 확장할 수 있게 하는 방식을 강조하세요. 그리고 면접관이 묻기 전에 오류 처리 문제를 먼저 언급하세요.
