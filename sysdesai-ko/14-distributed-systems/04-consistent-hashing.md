# Consistent Hashing

> 출처: https://www.sysdesai.com/learn/distributed-systems/consistent-hashing

---

## The Problem with Naive Hashing

N개의 nodes에 keys를 분산하는 가장 단순한 방법은 `node = hash(key) % N`입니다. 이 방식은 N이 고정되어 있을 때는 잘 작동합니다. 하지만 node를 추가하거나 제거하면 N이 변하게 되고, 거의 모든 key가 다른 node로 매핑됩니다. 만약 100만 개의 keys가 있는 상태에서 node를 하나 추가하면(N이 N+1이 됨), 거의 모든 keys를 다시 매핑해야 하며, 이는 대규모 cache miss storm을 일으키거나 몇 시간씩 걸리는 data reshuffling(데이터 재배치)을 초래할 수 있습니다.

## The Hash Ring

**Consistent hashing**은 해시 공간(hash space)을 링(ring) 형태로 구성하여 이 문제를 해결합니다 (보통 0에서 2^32 - 1 또는 0에서 2^64 - 1 범위). nodes와 keys 모두 이 링 위에 해싱됩니다. 각 key는 링 위에서 해당 key의 위치로부터 **시계 방향으로 첫 번째 node**(first node clockwise)가 소유합니다. node가 추가되거나 제거될 때, 새로 추가된/제거된 node와 그 이전 node 사이에 있는 keys만 다시 매핑하면 됩니다.
Consistent hash ring — 각 key는 시계 방향으로 가장 가까운 node에 매핑됩니다.
**hash=250 위치에 node D 추가**: hash=200 (Node B)과 hash=250 (Node D) 사이에 있는 keys만 Node C에서 Node D로 이동하면 됩니다. 다른 모든 keys는 영향을 받지 않습니다. 4개의 nodes가 있는 링에서 node 하나를 추가하면 modulo hashing과 달리 평균적으로 1/4의 keys만 다시 매핑됩니다.

## Virtual Nodes (Vnodes)

기본적인 consistent hash ring은 두 가지 문제가 있습니다: **불균형한 부하 분산**(nodes가 링 위에 균일하지 않은 위치에 배치됨)과 **node 실패 시의 hot spots**(실패한 node의 모든 keys가 하나의 후속 node로 이동함)입니다. **Virtual nodes**는 이 두 문제를 모두 해결합니다: 각 물리적 node는 링 위의 여러 위치에 배치됩니다 (예: Cassandra에서는 256개 위치). 이러한 가상 위치들이 링 전체에 퍼져 부하가 균등하게 분산되도록 보장합니다.

- **Even load**: node당 256개의 vnodes를 사용하면, node당 1개의 위치를 가질 때보다 부하 편차가 훨씬 작아집니다.
- **Graceful failure**: node가 실패하면, 해당 node의 256개 가상 위치들은 그들이 가진 keys를 여러 다른 nodes로 재분배합니다. 즉, 단 하나의 후속 node가 모든 부하를 떠안지 않습니다.
- **Heterogeneous hardware**: 성능이 더 좋은 nodes에는 더 많은 가상 위치를 할당하여 비례적으로 더 많은 부하를 처리하게 할 수 있습니다.

typescript

```
class ConsistentHashRing {
  private ring: Map<number, string> = new Map(); // hash -> nodeId
  private sortedHashes: number[] = [];
  private replicationFactor: number;
  private vnodes: number;

  constructor(replicationFactor = 3, vnodes = 256) {
    this.replicationFactor = replicationFactor;
    this.vnodes = vnodes;
  }

  addNode(nodeId: string): void {
    for (let i = 0; i < this.vnodes; i++) {
      const virtualKey = `${nodeId}#${i}`;
      const hash = this.hash(virtualKey);
      this.ring.set(hash, nodeId);
      this.sortedHashes.push(hash);
    }
    this.sortedHashes.sort((a, b) => a - b);
  }

  getNode(key: string): string {
    const hash = this.hash(key);
    // Find first node clockwise from key's hash
    const idx = this.sortedHashes.findIndex(h => h >= hash);
    const ringIdx = idx === -1 ? 0 : idx; // wrap around
    return this.ring.get(this.sortedHashes[ringIdx])!;
  }

  getReplicaNodes(key: string): string[] {
    // Returns replicationFactor distinct nodes for this key
    const hash = this.hash(key);
    const seen = new Set<string>();
    const nodes: string[] = [];
    let idx = this.sortedHashes.findIndex(h => h >= hash);
    if (idx === -1) idx = 0;

    while (nodes.length < this.replicationFactor) {
      const nodeId = this.ring.get(this.sortedHashes[idx % this.sortedHashes.length])!;
      if (!seen.has(nodeId)) { seen.add(nodeId); nodes.push(nodeId); }
      idx++;
    }
    return nodes;
  }

  private hash(key: string): number {
    // In production: use MurmurHash3 or xxHash
    let h = 0;
    for (const c of key) h = Math.imul(31, h) + c.charCodeAt(0) | 0;
    return Math.abs(h);
  }
}
```

## Real-World Usage

| System | Usage | Notes |
| --- | --- | --- |
| Apache Cassandra | nodes 간의 데이터 파티셔닝 | 기본적으로 node당 256 vnodes 사용; Murmur3 해시 사용 |
| Amazon DynamoDB | Partition key → storage node 매핑 | 관리형 서비스; 내부적으로 consistent hashing 사용 |
| Memcached clients | Client-side key → server 매핑 | Ketama 알고리즘은 consistent hash의 변형입니다 |
| Amazon ElastiCache | Cluster slot 할당 | Redis Cluster는 16384개의 고정 슬롯을 사용합니다 (순수 consistent hashing은 아님) |
| Discord | 사용자 연결을 위한 shard 선택 | Consistent hash가 user ID를 shard에 매핑합니다 |

> ℹ️
> Consistent hashing vs Redis Cluster slots
> Redis Cluster는 각 key의 CRC16 값을 16384로 나눈 나머지(modulo)를 취하는 16,384개의 고정 슬롯 공간을 사용합니다. 이는 순수한 consistent hashing은 아니지만 유사한 목표를 달성합니다. nodes가 추가되거나 제거될 때 슬롯이 재할당되며, 이동된 슬롯에 있는 keys만 영향을 받습니다. 고정 슬롯 공간은 cluster membership gossip을 단순화합니다.

> 💡
> Interview Tip
> Consistent hashing은 거의 모든 분산 저장 시스템 설계 질문에서 등장합니다. 전달해야 할 핵심 포인트: (1) 단순한 `hash % N` 방식은 위상(topology) 변화 시 모든 keys를 재매핑합니다; (2) consistent hashing은 평균적으로 K/N개의 keys만 재매핑합니다; (3) virtual nodes는 불균등한 분산과 hot failover 문제를 해결합니다; (4) 실제 시스템(Cassandra, DynamoDB)에서 이를 사용합니다. 분산 캐시 설계를 요청받으면 항상 vnodes를 포함한 consistent hashing을 파티셔닝 전략으로 언급하세요.
