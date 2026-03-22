# Distributed Consensus: Paxos & Raft

> 출처: https://www.sysdesai.com/learn/distributed-systems/distributed-consensus

---

## The Consensus Problem

**Distributed consensus**는 여러 nodes가 하나의 값에 대해 합의를 이루는 과정에서 발생하는 도전 과제입니다. nodes의 crash, messages 지연, 또는 network partitions가 발생하더라도 합의가 이루어져야 합니다. 이는 모든 replicated system의 근간이 됩니다: distributed databases, coordination services (ZooKeeper, etcd), 그리고 leader election 등이 여기에 해당합니다. 올바른 consensus protocol이 없다면, replicated state machines는 서로 달라지게(diverge) 되고 데이터 손실이나 오염이 발생할 수 있습니다.

FLP impossibility result (Fischer, Lynch, Paterson 1985)는 단 하나의 process만 crash되더라도 비동기 network(asynchronous network)에서는 어떤 deterministic consensus algorithm도 종료(termination)를 보장할 수 없음을 증명했습니다. 실제 시스템에서는 **timeouts**와 **partial synchrony** 가정을 통해 이 문제를 우회합니다. 즉, network가 느려질 수는 있지만 결국 messages는 도착한다는 가정입니다.

> ℹ️
> consensus가 만족해야 하는 세 가지 속성
> **Agreement**: 모든 정상적인 nodes는 동일한 값으로 결정합니다. **Validity**: 결정된 값은 반드시 어떤 node에 의해 제안된 값이어야 합니다. **Termination**: 모든 정상적인 nodes는 결국 결정을 내립니다. 실제 프로토콜들은 partition 상황에서 safety를 보장하기 위해 termination을 포기하기도 합니다 (CAP theorem 참고).

## Paxos

Paxos (Lamport 1989, 1998년 발표)는 널리 연구된 최초의 실용적인 consensus algorithm입니다. 두 단계(two phases)로 작동하며 세 가지 역할을 사용합니다: **Proposers**는 consensus를 시작하고, **Acceptors**는 제안에 투표하며, **Learners**는 선택된 값을 학습합니다. 진행을 위해서는 acceptors의 majority quorum 응답이 필요합니다.
Single-decree Paxos: 하나의 값에 합의하기 위한 2단계 프로토콜
**Multi-Paxos**는 이를 확장하여 값의 시퀀스(log)에 합의합니다. 안정적인 leader를 선출하여 이후 entries에 대해 Phase 1을 건너뛸 수 있게 함으로써 message 복잡도를 줄입니다. 하지만 Paxos는 올바르게 구현하기가 매우 까다롭기로 유명합니다. Lamport 자신도 혼란을 줄이기 위해 2001년에 "Paxos Made Simple"이라는 논문을 썼을 정도입니다.

## Raft: 이해하기 쉬운 시스템을 위한 Consensus

Raft (Ongaro & Ousterhout 2014)는 명확하게 **understandability**(이해 가능성)를 목표로 설계되었습니다. consensus를 세 가지 상대적으로 독립적인 하위 문제로 분해합니다: **leader election**, **log replication**, 그리고 **safety**. Raft는 한 번에 하나의 강력한 leader를 선출하며, 모든 writes는 leader를 통과하므로 시스템에 대한 추론이 단순해집니다.
Raft node state machine — nodes는 Follower, Candidate, Leader 상태를 순환합니다.
**Terms**는 Raft의 logical clock 역할을 합니다. 각 term은 election과 함께 시작됩니다. follower가 election timeout (보통 150–300 ms) 내에 heartbeat를 받지 못하면, term을 증가시키고 candidate가 되어 votes를 요청합니다. node는 candidate의 log가 최소한 자신의 것만큼 최신 상태일 때 vote를 부여합니다. 과반수(majority)를 먼저 확보한 candidate가 leader가 됩니다.

### Log Replication

leader가 선출되면 모든 client writes는 leader에게 전달됩니다. leader는 해당 entry를 자신의 log에 추가하고 병렬로 followers에게 `AppendEntries` RPCs를 보냅니다. 대다수의 nodes가 해당 entry를 log에 기록하면 그 entry는 **committed** 상태가 됩니다. 그 후 leader는 해당 entry를 자신의 state machine에 적용하고 client에게 응답합니다. followers는 다음 `AppendEntries` heartbeat 중에 committed entries를 적용합니다.

pseudocode

```
// Raft Leader — handling a client write
function handleClientWrite(value):
  entry = { term: currentTerm, index: log.length + 1, value }
  log.append(entry)

  // Send AppendEntries to all followers in parallel
  responses = parallel_send_all(followers, AppendEntries {
    term: currentTerm,
    leaderId: self.id,
    prevLogIndex: entry.index - 1,
    prevLogTerm: log[entry.index - 1].term,
    entries: [entry],
    leaderCommit: commitIndex
  })

  // Commit when majority acknowledges
  successCount = 1  // count self
  for r in responses:
    if r.success: successCount++

  if successCount > (clusterSize / 2):
    commitIndex = entry.index
    applyToStateMachine(entry)
    return SUCCESS
  else:
    return RETRY
```

## Paxos vs Raft Comparison

| Property | Paxos (Multi-Paxos) | Raft |
| --- | --- | --- |
| Leadership | 암시적(Implicit) — 어떤 node든 제안 가능 | 명시적인 강력한 leader |
| Log holes | 발생 가능 — entries가 듬성듬성할 수 있음 | 구멍 없음 — 순차적 |
| Understandability | 매우 복잡하기로 유명함 | 단순함을 목표로 설계됨 |
| Leader election | Ad-hoc 방식, 내장된 메커니즘 없음 | Term 기반, randomized timeout |
| Reads from follower | 복잡함 — lease/quorum 필요 | lease reads로 지원됨 |
| 사용 사례 | Google Chubby, Apache Zookeeper (ZAB 변형) | etcd, CockroachDB, TiKV, Consul |

## Real-World Systems

- **etcd** (Kubernetes): cluster state 저장을 위해 Raft를 사용합니다. Kubernetes control-plane의 모든 결정은 etcd consensus를 거칩니다.
- **CockroachDB**: per-range (keyspace의 64 MB shards) 단위로 Raft를 사용하여 per-shard fault tolerance를 제공합니다.
- **Apache Kafka** (3.0 이후): 자체 metadata management를 위해 ZooKeeper를 KRaft (Kafka Raft)로 대체했습니다.
- **Google Spanner**: shard별로 Paxos를 사용하며 TrueTime과 결합하여 externally consistent transactions를 구현합니다.
- **Consul**: service catalog 및 key-value storage를 위해 Raft를 사용하며, 최대 약 3개의 data centers를 기본적으로 지원합니다.

> 💡
> Interview Tip
> 면접에서 consensus에 대해 질문을 받으면 단순히 Raft/Paxos의 이름만 언급하는 것 이상의 답변을 준비하세요. 핵심 통찰을 설명하세요: **majority quorum** (N/2 + 1 nodes)은 어떤 두 quorums라도 최소한 하나의 node가 겹친다는 것을 보장하며, 따라서 최신 committed value는 항상 알 수 있다는 점입니다. Raft가 etcd에서 사용되며 모든 Kubernetes cluster의 기반이 된다는 점도 언급하세요. 더 깊은 질문이 나온다면, read-heavy 시스템에서 reads 시 quorum round trips를 피하기 위해 **lease-based reads**를 어떻게 사용하는지 설명해 보세요.
