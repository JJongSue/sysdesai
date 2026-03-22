# Vector Clocks & Logical Timestamps

> 출처: https://www.sysdesai.com/learn/distributed-systems/vector-clocks

---

## The Problem: Ordering Events Without a Global Clock

distributed system에서 nodes는 물리적 시계(physical clock)를 공유하지 않습니다. Network Time Protocol (NTP)은 시계를 약 100ms 이내로 동기화하지만, 수 밀리초 간격으로 발생하는 events의 순서를 정하기에는 정확도가 부족합니다. 만약 서로 다른 기기에서 node A가 12:00:00.001에 레코드를 쓰고 node B가 12:00:00.002에 동일한 레코드를 쓴다면, 각자의 시계는 어느 것이 먼저 발생했는지에 대해 서로 다를 수 있습니다. 따라서 wall-clock(실제 시간)을 이용한 순서 정렬은 신뢰할 수 없습니다.

**Logical clocks**는 실제 시간 대신 인과 관계(causal relationships, happened-before)를 추적하여 이 문제를 해결합니다. 만약 event A가 event B에 인과적으로 영향을 주었다면(예: A가 보낸 messages를 B가 받음), wall-clock 시간에 관계없이 A가 B보다 먼저 발생했다고 확실히 말할 수 있습니다.

## Lamport Timestamps

Leslie Lamport (1978)는 **happened-before** 관계(→)를 정의하고 logical clocks를 도입했습니다. 규칙은 간단합니다: 각 node는 counter `C`를 유지합니다. 로컬 event가 발생하면 `C`를 증가시킵니다. messages를 보낼 때 현재 `C`를 첨부합니다. messages를 받을 때는 `C = max(local_C, received_C) + 1`로 설정합니다.

pseudocode

```
// Lamport Timestamp rules
// Each node maintains integer counter C, initially 0

// Rule 1: Local event
function onLocalEvent():
  C = C + 1
  event.timestamp = C

// Rule 2: Send message
function onSend(message):
  C = C + 1
  message.timestamp = C
  send(message)

// Rule 3: Receive message
function onReceive(message):
  C = max(C, message.timestamp) + 1
  process(message)
```

> ⚠️
> Lamport timestamps는 인과 관계를 완전히 포착하지 못합니다
> Lamport timestamps는 다음을 보장합니다: 만약 A → B이면 ts(A) < ts(B)입니다. 하지만 그 역은 보장되지 않습니다: 즉, ts(A) < ts(B)라고 해서 A → B인 것은 아닙니다. messages 교환이 없는 서로 다른 nodes의 두 events(동시 발생 events, concurrent events)는 어떤 timestamp 순서도 가질 수 있습니다. 동시성(concurrency)을 감지하려면 vector clocks가 필요합니다.

## Vector Clocks

**Vector clocks**는 Lamport timestamps를 확장하여 전체 happened-before 관계를 포착합니다. 각 node는 시스템의 각 node당 하나씩 counter들의 vector를 유지합니다. node i의 vector clock `VC`는 다음과 같이 업데이트됩니다: 로컬 events 및 전송 시 `VC[i]`를 증가시킵니다. 수신 시에는 로컬 vector와 수신된 vector의 요소별 max(element-wise max)를 취한 후 `VC[i]`를 증가시킵니다.
Vector clock updates: 각 node는 다른 모든 node로부터 관찰한 최신 event를 추적합니다.
두 vector clocks `VC_a`와 `VC_b`를 비교할 때: `VC_a`의 모든 요소가 `VC_b`의 해당 요소보다 작거나 같다면(≤), a → b (a가 b보다 먼저 발생함)입니다. 만약 어느 쪽도 우세하지 않다면(어떤 요소는 `VC_a`가 크고, 어떤 요소는 `VC_b`가 크다면), 해당 events는 **concurrent**(동시 발생)하며 인과 관계가 없습니다.

typescript

```
type VectorClock = number[];

function happensBefore(a: VectorClock, b: VectorClock): boolean {
  // a → b: every element of a <= b AND at least one is strictly less
  const allLeq = a.every((val, i) => val <= b[i]);
  const someStrict = a.some((val, i) => val < b[i]);
  return allLeq && someStrict;
}

function concurrent(a: VectorClock, b: VectorClock): boolean {
  return !happensBefore(a, b) && !happensBefore(b, a);
}

// Example:
// VC_a = [2, 1, 0]  VC_b = [1, 2, 0]
// happensBefore(a, b) = false (a[0]=2 > b[0]=1)
// happensBefore(b, a) = false (b[1]=2 > a[1]=1)
// concurrent(a, b) = true — a and b are concurrent!
```

## Version Vectors and Conflict Detection

**Version vectors**는 특히 (event 인과 관계가 아닌) 데이터 버전의 인과 관계를 추적하는 데 사용되는 매우 유사한 개념입니다. DynamoDB와 Cassandra는 동일한 key에 대한 write conflicts를 감지하기 위해 version vectors(문서에서 종종 vector clocks라고 부름)를 사용합니다.

두 clients가 동시에 동일한 key를 업데이트할 때(그들의 version vectors가 concurrent할 때), 시스템은 자동으로 갈등을 해결할 수 없습니다. 시스템은 두 버전을 모두 애플리케이션에 노출하거나 last-write-wins 정책을 사용해야 합니다. Amazon의 DynamoDB 논문은 이를 실제 사례로 보여주었습니다: 두 명의 오프라인 clients가 장바구니에 아이템을 추가하는 경우 두 버전이 모두 보존됩니다.

| Concept | What It Tracks | Use Case |
| --- | --- | --- |
| Lamport timestamp | event당 단일 정수 | Total ordering (인과 관계 불완전) |
| Vector clock | N개 정수의 배열 (node당 하나) | Causal ordering, concurrency detection |
| Version vector | 데이터 항목 버전의 인과 관계 | Replicated KV stores에서의 Conflict detection |
| Hybrid Logical Clock (HLC) | 물리적 + 논리적 시간 | Distributed transactions (CockroachDB) |

> 💡
> Interview Tip
> "DynamoDB는 동일한 key에 대한 동시 쓰기를 어떻게 처리합니까?"라는 질문에 대한 좋은 답변은: version vectors (vector clocks)를 사용한다는 것입니다. 만약 vectors가 concurrent하다면(어느 쪽도 우세하지 않음), DynamoDB는 두 값을 siblings로 저장하고 읽기 시 둘 다 반환합니다. 그 후 애플리케이션이 갈등을 해결합니다. 이것이 Dynamo 논문의 장바구니 예시에서 두 장바구니 추가 항목이 모두 유지된 이유입니다: 고객의 장바구니에서 아이템을 잃어버리는 것보다 추가 항목을 보여주는 것이 더 안전하기 때문입니다.
