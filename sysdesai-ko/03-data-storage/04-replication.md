# Database Replication (데이터베이스 복제)

> Source: https://www.sysdesai.com/learn/data-storage/replication

---

## What Replication Solves (복제가 해결하는 문제)

**Replication(복제)**는 데이터를 한 노드(Primary, 프라이머리)에서 하나 이상의 다른 노드(Replica, 복제본)로 복사하는 것을 말합니다. 이는 두 가지 뚜렷한 목표를 가집니다: **High Availability(고가용성)** (프라이머리가 실패할 경우 복제본이 대신 처리함) 및 **Read Scaling(읽기 확장)** (읽기 트래픽을 여러 복제본에 분산함). *어떤 목표를 최적화하느냐*에 따라 선택할 복제 토폴로지가 결정됩니다.

## Single-Leader (Master-Replica) Replication (단일 리더 복제)

가장 일반적인 토폴로지입니다. 모든 쓰기는 **Leader(리더, Primary)**로 전달됩니다. 리더는 변경 사항을 **Followers(팔로워, Replicas)**에게 스트리밍하고, 팔로워들은 이를 비동기적으로 적용합니다. 읽기는 모든 복제본에서 처리할 수 있습니다. 이는 **PostgreSQL streaming replication**, **MySQL replica**, **MongoDB replica sets**의 기본 구성입니다.

### Replication Lag (복제 지연)

**Asynchronous Replication(비동기 복제)**에서 복제본은 리더가 이미 클라이언트에게 쓰기 완료를 응답한 후에 변경 사항을 적용합니다. 이로 인해 **Replication Lag(복제 지연)**이 발생합니다 — 복제본이 오래된(stale) 데이터를 제공하는 구간이 생기는 것입니다. 잘 구성된 시스템에서의 일반적인 지연은 밀리초에서 초 단위이지만, 쓰기 부하가 심할 경우 몇 분까지 늘어날 수 있습니다.

> ⚠️
> Read-After-Write Inconsistency (쓰기 후 읽기 불일치)
> 사용자가 프로필을 업데이트했는데, 다음 요청(읽기)이 지연된 복제본으로 전달되면 이전 데이터를 보게 됩니다. 이를 'read-after-write' 이상 현상이라고 합니다. 해결책: (1) 현재 사용자의 읽기 요청은 항상 프라이머리로 라우팅합니다. (2) 쓰기 작업의 복제 위치를 추적하여 최신 상태인 복제본으로만 읽기 요청을 보냅니다. (3) Sticky Session을 사용하여 항상 동일한 복제본을 바라보게 합니다.

### Synchronous vs Asynchronous Replication (동기 vs 비동기 복제)

| Mode(모드) | When Primary Acks(프라이머리 응답 시점) | Data Loss Risk(데이터 손실 위험) | Write Latency(쓰기 지연 시간) |
| --- | --- | --- | --- |
| Asynchronous(비동기) | 복제본 확인 전 | 있음 — 복제본 지연 가능성 | 낮음 (프라이머리만 처리) |
| Synchronous(동기) | 최소 하나의 복제본 확인 후 | 없음 — 최소 하나의 내구성 복사본 보장 | 높음 (복제본 확인 대기) |
| Semi-synchronous(반동기, MySQL) | 하나의 복제본 확인 후; 나머지는 비동기 | 최소화됨 — 하나의 복제본 보장 | 중간 |

## Failover and Leader Election (장애 조치 및 리더 선출)

프라이머리가 실패하면 복제본 중 하나가 승격되어야 합니다. 이 과정을 **Failover(장애 조치)**라고 합니다. 두 가지 핵심 과제: (1) 프라이머리가 실제로 죽었는지(단순히 느린 것이 아닌지) 판단하는 것 — **Split-brain(스플릿 브레인)** 문제. (2) 어떤 복제본을 승격시킬지 선택하는 것 — 보통 복제 지연이 가장 적은 것을 선택합니다. **Patroni**(PostgreSQL), **Orchestrator**(MySQL), **ZooKeeper**와 같은 도구들이 이를 자동화합니다.

> ⚠️
> Split Brain (스플릿 브레인)
> 프라이머리가 일시적으로 도달 불가능해지고(네트워크 파티션) 새로운 프라이머리가 선출되면, 쓰기를 허용하는 두 개의 노드가 생기게 됩니다 — 이것이 스플릿 브레인입니다. 파티션이 해결될 때 이전 프라이머리에 기록된 쓰기는 유실됩니다. 이를 방지하려면 **Quorum(정족수)** 매커니즘이 필요합니다: 프라이머리는 과반수의 노드에 도달할 수 있을 때만 쓰기를 수용할 수 있습니다.

## Multi-Leader (Master-Master) Replication (다중 리더 복제)

여러 노드가 동시에 쓰기를 수용합니다. 여러 지역에서 낮은 지연 시간의 쓰기가 필요한 **Geo-distributed(지리적 분산)** 시스템에 유용합니다. 근본적인 과제는 **Write Conflict(쓰기 충돌)**입니다: 두 리더가 동시에 동일한 행에 대해 서로 다른 값을 수용하면 어떤 값이 이길까요?

- **Last-Write-Wins (LWW)**: 나중 타임스탬프를 가진 쓰기가 이깁니다. 단순하지만 순서가 바뀌어 도착한 동시 쓰기가 유실될 위험이 있습니다.
- **Conflict-free Replicated Data Types (CRDTs)**: 자동으로 병합되도록 설계된 데이터 구조(카운터, 셋 등). DynamoDB, Riak에서 사용됩니다.
- **Application-level Resolution(애플리케이션 레벨 해결)**: 충돌을 애플리케이션에 노출합니다(CouchDB 방식). 애플리케이션이 병합 로직을 결정합니다.
- **Avoid Conflicts(충돌 회피)**: 레코드 키에 일관된 해싱을 사용하여 해당 레코드의 모든 쓰기를 동일한 리더로 라우팅합니다.

## Quorum-Based Replication (Leaderless) (정족수 기반 복제, 리더리스)

**Leaderless(리더리스)** 시스템(Cassandra, DynamoDB, Riak)에서는 모든 노드가 쓰기를 수용할 수 있습니다. 일관성은 **Quorum Read/Write(정족수 읽기/쓰기)**를 통해 달성됩니다. 전체 복제본이 `N`개일 때, `W`개의 복제본이 쓰기를 확인하면 쓰기 성공이며, `R`개의 복제본이 응답하면 읽기 성공입니다. `W + R > N`일 때 일관성이 보장됩니다.

```text
Cassandra quorum 예시 (N=3 복제본):
  Write quorum (W=2): 3개 노드 중 2개에서 쓰기 성공해야 함
  Read quorum  (R=2): 3개 노드 중 2개에서 읽어와서 최신값 선택

  W + R = 4 > N = 3 → Consistent(일관적, 항상 최신 쓰기를 확인)

  강한 일관성: W=3, R=1 (또는 W=1, R=3)
  고가용성: W=1, R=1 (최종 일관성, eventual consistency)
  균형 잡힌 구성: W=2, R=2 (노드 1개의 장애를 견딤)
```

`W + R <= N`이면 **Eventual Consistency(최종 일관성)**를 갖게 됩니다 — 읽기와 쓰기는 빠르지만 정족수 오버랩에 도달하지 못해 오래된 데이터를 읽을 수도 있습니다. Cassandra는 기본적으로 `QUORUM` 일관성을 사용하지만 쿼리별로 오버라이드를 허용합니다.

> 💡
> Interview Tip (인터뷰 팁)
> 인터뷰에서 복제 관련 질문은 보통 다음 패턴을 따릅니다: "프라이머리가 다운되면 어떻게 처리하나요?" 답변 순서: 장애 감지(하트비트 + 타임아웃), 새 리더 선출(Raft/Paxos 또는 Sentinel), 쓰기 리다이렉션, 진행 중인 트랜잭션 처리. 스플릿 브레인 위험과 정족수가 이를 어떻게 방지하는지 언급하세요. 이는 단순히 교과서적인 토폴로지 다이어그램이 아니라 운영 실무를 이해하고 있음을 보여줍니다.
