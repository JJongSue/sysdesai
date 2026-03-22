# Gossip Protocol

> 출처: https://www.sysdesai.com/learn/distributed-systems/gossip-protocol

---

## What Is a Gossip Protocol?

**Gossip protocol**(에피데믹 프로토콜, epidemic protocol이라고도 함)은 소셜 네트워크에서 소문이 퍼지는 방식에서 영감을 얻은 peer-to-peer 통신 메커니즘입니다. 각 node는 정기적인 간격으로 몇 개의 peers를 무작위로 선택하여 상태 정보를 교환합니다. 정보는 중앙 조정자(coordinator) 없이 전염병처럼 기하급수적으로 퍼져나갑니다.

주요 특징은 다음과 같습니다: **decentralization**(분산화; leader나 broker가 없음), **eventual consistency**(최종 일관성; 결국 모든 nodes가 동일한 상태로 수렴함), **scalability**(확장성; 전파 라운드당 message 복잡도는 O(log N)임), 그리고 **fault tolerance**(결함 허용; 어떤 node가 실패하더라도 프로토콜 진행에 영향을 주지 않음).

## How Gossip Spreads Information
Gossip은 기하급수적으로 퍼집니다 — 매 라운드마다 정보를 가진 nodes의 수가 두 배로 늘어납니다.
**Convergence**(수렴): N개의 nodes로 구성된 cluster에서 gossip은 O(log N) 라운드 내에 전체 전파를 완료합니다. 각 라운드에서 정보를 가진 node는 `fanout` 개수(보통 2-3개)의 peers에게 연락합니다. k 라운드 후에는 대략 `fanout^k`개의 nodes가 정보를 알게 됩니다. fanout이 3인 1,000개의 nodes가 있을 때: log₃(1000) ≈ 6 라운드면 충분합니다.

## Gossip for Failure Detection

**Failure detection**(장애 감지)은 gossip의 가장 중요한 응용 분야 중 하나입니다. 각 node는 모든 cluster members의 리스트와 그들의 **heartbeat counters**를 유지합니다. 각 gossip 라운드 동안 nodes는 membership lists를 교환합니다. 만약 특정 node의 counter가 임계치 이상의 라운드 동안 증가하지 않는다면, 해당 node는 실패한 것으로 의심받습니다. 이것을 **SWIM (Scalable Weakly-consistent Infection-style Membership)**이라고 하며, Consul과 Cassandra에서 사용하는 프로토콜입니다.

1. **Suspicion**(의심): node A가 node B에게 ping을 보냅니다. timeout 내에 ack가 없으면 A는 B를 suspected(의심됨) 상태로 표시합니다.
2. **Indirect ping**(간접 핑): A는 K개의 다른 nodes에게 자신을 대신해 B에게 ping을 보내달라고 요청합니다. 이는 오직 A-B 사이의 link만 끊어진 경우를 처리하기 위함입니다.
3. **Confirmation**(확인): 간접 핑이 모두 실패하면 B는 dead로 표시되고, 이 정보가 모든 nodes에 gossip으로 퍼집니다.

## Gossip in Production Systems

| System | Gossip Use Case |
| --- | --- |
| Apache Cassandra | Cluster membership, schema 변경, token ring 상태, failure detection |
| Consul | Service catalog, health check 전파 (SWIM 프로토콜) |
| Amazon DynamoDB | Node membership 및 ring 상태 (Dynamo 논문에 기술됨) |
| Bitcoin | 모든 full nodes에 transaction 및 block 전파 |
| Kubernetes | Calico CNI에서 BGP route 전파를 위해 gossip 사용 |

> ℹ️
> Push vs Pull vs Push-Pull gossip
> **Push**: 발신자가 무작위 peer를 선택해 자신의 상태를 밀어넣습니다. **Pull**: 발신자가 peer에게 상태를 요청합니다. **Push-Pull** (가장 효율적): 한 라운드 동안 발신자와 수신자가 양방향으로 상태를 교환하고 병합합니다. Push-Pull은 최소한의 추가 비용으로 Push 전용 방식에 비해 수렴 시간을 절반으로 줄여줍니다.

> 💡
> Interview Tip
> "Cassandra는 어떤 nodes가 살아있는지 어떻게 압니까?"라는 질문에 대한 답은 gossip/SWIM입니다. 단순히 direct ping timeout만으로는 link 단절과 node 장애를 구분할 수 없으므로, 다른 nodes를 통한 indirect ping 메커니즘을 설명하여 이해도를 보여주세요. 또한 gossip은 **eventually consistent**하므로 nodes가 일시적으로 membership에 대해 서로 다른 의견을 가질 수 있지만, 많은 경우에 이는 허용 가능하다는 점도 언급하세요.
