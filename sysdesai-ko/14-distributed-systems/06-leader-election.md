# Leader Election

> 출처: https://www.sysdesai.com/learn/distributed-systems/leader-election

---

## Why Leader Election?

많은 distributed systems는 충돌을 피하기 위해 한 번에 정확히 하나의 조정자(coordinator)가 필요합니다: primary database node, Kafka partition leader, jobs를 선택하는 scheduler 등이 그 예입니다. **Leader election**은 leader가 존재하지 않거나(시스템 시작 시) 현재 leader가 실패했을 때 자동으로 한 node를 leader로 지정하는 과정입니다. 올바르게 수행된다면, leader election은 **safety**(한 번에 최대 한 명의 leader만 존재함)와 **liveness**(결국 새로운 leader가 선출됨)를 보장합니다.

> ⚠️
> Safety vs Liveness tension
> network partition 상황에서 safety(오직 하나의 leader)와 liveness(항상 진행함)를 동시에 보장할 수는 없습니다 — 이는 CAP theorem의 직접적인 결과입니다. 대부분의 운영 시스템은 safety를 선택합니다: 두 명의 leader가 있는 것보다는 차라리 leader가 없는 편을 택합니다. 두 명의 leader가 존재하는 현상을 **split-brain**이라고 하며, 이는 데이터 오염을 유발할 수 있습니다.

## Bully Algorithm

**Bully algorithm** (Garcia-Molina 1982)은 가장 높은 ID를 가진 node를 leader로 선출합니다. node가 leader의 실패를 감지하거나(또는 시작 시), 자신보다 높은 ID를 가진 모든 nodes에게 `ELECTION` messages를 보내 선거를 시작합니다. 만약 더 높은 ID를 가진 node로부터 응답이 없다면, 스스로 leader임을 선언하고 자신보다 낮은 ID를 가진 모든 nodes에게 `COORDINATOR` messages를 보냅니다. 만약 더 높은 ID를 가진 node가 응답하면, 그 node가 선거를 인계받습니다.

1. Node P가 leader 실패를 감지하고, ID > P인 모든 nodes에게 ELECTION을 보냅니다.
2. timeout 내에 응답이 없으면 P가 승리합니다: 모든 nodes에게 COORDINATOR를 보냅니다.
3. 만약 node Q (ID > P)가 OK로 응답하면, Q가 선거를 인계받습니다.
4. Q는 ID > Q인 nodes를 대상으로 1단계를 반복합니다.
5. 살아있는 node 중 가장 높은 ID를 가진 node가 항상 승리합니다 — 그래서 'bully(괴롭히는 사람)'라고 불립니다.

> ℹ️
> Bully algorithm complexity
> 최악의 경우: O(N²) messages (가장 낮은 ID를 가진 node가 선거를 시작할 때). 최선의 경우: O(N) messages (두 번째로 높은 ID를 가진 node가 시작할 때). Bully 방식은 작은 cluster에서는 잘 작동하지만 수백 개의 nodes로 확장하기에는 무리가 있습니다.

## Ring Election Algorithm

**Ring algorithm**은 nodes를 논리적인 링(ring) 형태로 배치합니다. node가 선거를 시작할 때, 자신의 ID를 담은 election message를 링을 따라 시계 방향으로 보냅니다. 각 node는 도착한 ID와 자신의 ID를 비교하여 더 큰 값을 전달합니다. 메시지가 한 바퀴를 돌아 시작한 node에게 돌아오면, 가장 높은 ID가 승리하고 `LEADER` message가 링 전체에 전달됩니다.
Ring election: 메시지가 지금까지 확인된 가장 높은 ID를 운반합니다; Node D (ID=5)가 승리합니다.

## ZooKeeper-Based Leader Election

운영 시스템에서 순수 election algorithms를 직접 구현하는 경우는 드뭅니다. 대신 **ZooKeeper**나 **etcd**와 같은 **coordination service**를 사용합니다. 이들은 신뢰할 수 있는 기본 단위(primitive)를 제공합니다: **ephemeral sequential znodes** (ZooKeeper) 또는 **leases** (etcd).

1. 모든 후보 nodes는 `/election/` 아래에 ephemeral sequential znode를 생성합니다 (예: `/election/n_0000000001`, `/election/n_0000000002`).
2. 각 node는 모든 자식 노드들을 나열하고 자신이 가장 작은 시퀀스 번호를 가졌는지 확인합니다.
3. 맞다면 — leader가 됩니다.
4. 아니라면 — 바로 앞의(자신보다 한 단계 작은) node에 watch를 설정합니다 (herd effect를 피하기 위해 모든 node가 아닌 바로 앞 node만 감시합니다).
5. 현재 leader의 session이 만료되면(leader crash), 해당 ephemeral znode가 삭제되고 이를 감시하던 node가 알림을 받아 새로운 leader가 됩니다.

## Lease-Based Election with etcd

**etcd**는 `campaign` API를 제공하며, 이는 etcd의 Raft 기반 key-value store에 저장되는 시간 제한이 있는 잠금인 **leases**를 사용하여 leader election을 구현합니다. node는 TTL lease와 함께 key를 생성함으로써 leadership을 획득합니다. node는 주기적으로 lease를 갱신(renew)해야 합니다. 만약 crash가 발생하면 lease가 만료되고 다른 후보자가 이를 획득하게 됩니다.

go

```
// etcd leader election example (Go)
func runLeaderElection(client *clientv3.Client) {
    session, _ := concurrency.NewSession(client, concurrency.WithTTL(5))
    defer session.Close()

    election := concurrency.NewElection(session, "/leader")
    ctx := context.Background()

    // Campaign blocks until this node is elected
    if err := election.Campaign(ctx, "my-node-id"); err != nil {
        log.Fatal(err)
    }
    log.Println("I am the leader!")

    // Do leader work...
    // Resign when done or before shutdown
    election.Resign(ctx)
}
```

## Leader Election and Consensus

Leader Election은 곧 Consensus입니다: 어떤 node가 leader인지에 대해 모든 nodes가 합의하는 것이기 때문입니다. 이것이 Raft가 leader election을 consensus 프로토콜에 통합한 이유입니다 — 하나 없이는 다른 하나도 존재할 수 없습니다. ZooKeeper는 ZAB (ZooKeeper Atomic Broadcast)라는 Paxos 변형 프로토콜을 내부 consensus 프로토콜로 사용하여 leader election 결정을 신뢰성 있게 내립니다.

> 💡
> Interview Tip
> 흔한 면접 질문: '서비스에서 한 번에 정확히 하나의 인스턴스만 실행되어야 하는 경우(cron job, master process) 어떻게 구현합니까?' 답은 etcd나 ZooKeeper를 사용한 distributed leader election입니다. ephemeral key 패턴을 설명하세요: 승리한 node는 lease 기반의 key를 유지하고, 다른 모든 nodes는 이를 감시합니다. leader가 crash되면 key가 만료되고 새로운 선거가 트리거됩니다. Kubernetes가 자체 controller-manager와 scheduler의 leader election을 위해 etcd leases를 사용한다는 점을 언급하면 훌륭한 실무 근거가 됩니다.
