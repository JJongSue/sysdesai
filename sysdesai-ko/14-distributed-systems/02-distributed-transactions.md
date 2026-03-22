# Distributed Transactions: 2PC & 3PC

> 출처: https://www.sysdesai.com/learn/distributed-systems/distributed-transactions

---

## Why Distributed Transactions Are Hard

Distributed transaction은 여러 databases, services, 또는 nodes에 걸쳐 실행됩니다. 근본적인 도전 과제는 각 참여자가 **모두 commit하거나 모두 abort**해야 한다는 것이지만, 어느 참여자(또는 network)든 어느 시점에서나 실패할 수 있다는 점입니다. coordination protocol이 없다면, 하나의 service는 commit하고 다른 service는 abort하여 데이터가 영구적으로 불일치(inconsistent)하게 될 위험이 있습니다.

> ⚠️
> ACID across services
> ACID 속성(Atomicity, Consistency, Isolation, Durability)은 단일 database 엔진 내에서는 구현하기 쉽습니다. 하지만 독립적인 여러 services에 걸쳐 이를 달성하려면 명시적인 coordination protocols가 필요하며, 이러한 프로토콜들은 상당한 performance 및 availability 비용을 수반합니다.

## Two-Phase Commit (2PC)

**Two-Phase Commit**은 고전적인 distributed transaction 프로토콜입니다. **coordinator** node가 두 단계를 조율합니다: Phase 1 (Prepare)에서 coordinator는 모든 **participants**에게 commit이 가능한지 묻습니다. Phase 2 (Commit/Abort)에서 모든 응답을 바탕으로 coordinator가 최종 결정을 보냅니다.
Two-Phase Commit (2PC) — 세 개의 databases에 걸친 coordinator 주도의 원자적(atomic) commit

### The Blocking Problem

2PC는 **blocking protocol**입니다. 만약 coordinator가 PREPARE를 보낸 후 commit 결정을 보내기 전에 crash되면, participants는 locks를 무기한 보유한 채 멈추게 됩니다. participants는 독자적으로 commit이나 abort를 결정할 수 없습니다. 다른 참여자가 이미 commit했을 경우 abort하면 atomicity를 위반할 수 있기 때문입니다. 이를 **in-doubt window**라고 하며, participants는 coordinator가 복구될 때까지 기다려야 합니다.

> ⚠️
> 2PC failure scenarios
> **prepare 이후 Coordinator 실패**: 모든 participants는 coordinator가 복구될 때까지 block됩니다. **YES 투표 이후 Participant 실패**: coordinator는 기다리거나 abort해야 합니다. **Network partition**: partition의 잘못된 쪽에 있는 participants는 진행할 수 없습니다. 이러한 모든 경우에 locks가 유지되어 throughput(처리량)이 크게 저하됩니다.

## Three-Phase Commit (3PC)

**Three-Phase Commit**은 2PC의 blocking 문제를 해결하기 위해 `pre-commit` 단계를 추가합니다. 세 단계는 다음과 같습니다: **CanCommit** (Prepare와 동일), **PreCommit** (coordinator와 participants가 준비 상태를 확인), 그리고 **DoCommit** (최종 commit). 핵심 아이디어는 모든 participants가 PreCommit을 확인한 후에는, coordinator와 연락이 끊기더라도 participant가 스스로 안전하게 commit할 수 있다는 것입니다. 다른 모든 참여자도 PreCommit 상태임을 알기 때문입니다.

| Property | 2PC | 3PC |
| --- | --- | --- |
| Phases | Prepare, Commit/Abort | CanCommit, PreCommit, DoCommit |
| Blocking on coordinator failure | 예 — 무기한 | 아니오 — 참여자들이 진행 가능 |
| Message complexity | 2N messages (prepare + commit) | 3N messages |
| Network partition safe | 아니오 | 아니오 (split-brain 발생 가능) |
| Latency | 낮음 (적은 messages) | 높음 (추가적인 round trip) |
| 실무 사례 | PostgreSQL FDW, XA transactions | 드묾 — 주로 이론적 |

> ⚠️
> 3PC는 partition-tolerant하지 않습니다
> 3PC가 blocking 문제를 해결하긴 하지만, network partitions 상황에서 split-brain을 유발할 수 있습니다. 만약 PreCommit과 DoCommit 사이에 partition이 발생하면, 두 파티션이 서로 다르게 결정할 수 있습니다 (하나는 timeout으로 abort, 다른 하나는 commit). 이러한 이유로 실무 distributed systems에서는 3PC가 거의 사용되지 않습니다.

## Modern Alternatives

대부분의 현대적인 microservice architectures는 distributed transactions를 완전히 피합니다. 대안은 다음과 같습니다:

- **Saga pattern**: 로컬 트랜잭션(local transactions)의 시퀀스로, 각각이 다음 트랜잭션을 트리거하는 events를 발행합니다. Compensating transactions가 rollback을 처리합니다. Uber, Amazon 및 대부분의 event-driven architectures에서 사용됩니다.
- **Outbox pattern**: 로컬 DB와 outbox table에 원자적으로 쓰고, relay process가 이벤트를 발행합니다. Distributed transactions 없이 at-least-once delivery를 보장합니다.
- **Eventual consistency**: 서로 다른 services가 일시적으로 불일치할 수 있음을 수용하고, 이를 처리하도록 UX를 설계합니다 (예: '대기 중' 상태 표시).
- **Google Spanner**: TrueTime 기반의 timestamps를 사용하여 Paxos와 함께 external consistency를 달성하며, 대규모 환경에서 전통적인 2PC coordinator의 병목 현상을 피합니다.

> 💡
> Interview Tip
> 면접관들은 종종 "두 개의 microservices에 걸친 트랜잭션을 어떻게 처리합니까?"라고 묻습니다. 정답은 "2PC"가 아닙니다. 2PC는 결합도(coupling)를 높이고 blocking coordinator를 도입한다고 설명하세요. 대신 Compensating transactions를 사용하는 **Saga pattern**을 설명하고, 신뢰할 수 있는 이벤트 발행을 위한 **Outbox pattern**을 언급하세요. 이는 여러분이 distributed systems의 운영상의 현실을 이해하고 있음을 보여줍니다.
