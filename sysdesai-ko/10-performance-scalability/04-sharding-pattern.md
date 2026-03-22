# 확장성 패턴으로서의 Sharding

> 출처: https://www.sysdesai.com/learn/performance-scalability/sharding-pattern

---

## 왜 Sharding인가?

**Sharding** (또는 **horizontal partitioning**이라고도 함)은 하나의 큰 데이터셋을 shard라고 불리는 여러 독립적인 database node에 분산시킵니다. 각 shard는 데이터의 일부를 보유하고 해당 파티션의 읽기와 쓰기를 처리합니다. Sharding은 단일 머신이 처리할 수 있는 한계를 넘어 **쓰기**를 확장하는 주요 기법입니다 — 수직 확장 (더 큰 머신)은 결국 한계에 도달하지만, shard는 항상 더 추가할 수 있습니다.

MongoDB, Cassandra, DynamoDB, Vitess (YouTube와 Slack이 사용)는 모두 내부적으로 sharding을 구현합니다. Sharding을 이해하는 것은 시니어 시스템 디자인 인터뷰의 핵심 기대사항입니다.

## Shard Key

**Shard key**는 각 레코드를 특정 shard로 라우팅하는 데 사용되는 속성입니다. 잘못된 shard key를 선택하는 것이 가장 흔한 sharding 실수입니다. 좋은 shard key는 shard 전체에 데이터를 균등하게 분산시키고 (hotspot 방지), 대부분의 쿼리가 단일 shard를 대상으로 하도록 하며 (scatter-gather 방지), 불변입니다 (레코드에 할당된 후 절대 변경되지 않음).

> ⚠️
> Hot Shard 안티패턴
> 낮은 cardinality 또는 시간적으로 편향된 shard key를 사용하면 hot shard가 생성됩니다. 예를 들어, `status` (active/inactive)로 sharding하면 모든 쓰기가 'active' shard에 집중됩니다. `created_at` 날짜로 sharding하면 현재 날짜 shard가 모든 새 쓰기를 받는 반면 다른 모든 shard는 유휴 상태가 됩니다. Shard key를 선택하기 전에 항상 access 분포를 측정하세요.

## Range Sharding vs Hash Sharding

| 측면 | Range Sharding | Hash Sharding |
| --- | --- | --- |
| 작동 방식 | [A-M] 범위의 key를 가진 레코드는 shard 1로, [N-Z]는 shard 2로 | hash(key) % num_shards가 대상 shard를 결정 |
| Range 쿼리 | 효율적 — 데이터가 정렬되어 있어 단일 shard 또는 연속 shard | Scatter-gather — 범위가 예측 불가능하게 모든 shard에 걸침 |
| 쓰기 분산 | 새 key가 항상 높은 쪽에 있으면 편향될 수 있음 (예: time-series) | key 패턴에 관계없이 균등 분산 |
| 재균형 | 모든 데이터를 이동하지 않고 범위 분할 | Shard 추가 시 많은 key를 재매핑하고 이동해야 함 |
| 최적 사용 | Time-series, 순서 있는 lookup, prefix scan | Key-value access, user ID, 무작위 쓰기 |

## Sharding을 위한 Consistent Hashing

단순 모듈로 해싱 (`hash(key) % N`)은 shard를 추가하거나 제거할 때 심각하게 문제가 됩니다 — 거의 모든 key가 다른 shard에 매핑되어 대규모 데이터 마이그레이션이 필요합니다. **Consistent hashing**은 key와 shard 모두를 가상 링에 배치합니다. Shard를 추가하면 인접한 이웃의 key만 이동하여 마이그레이션을 데이터의 `1/N`으로 제한합니다. Redis Cluster와 Amazon DynamoDB는 더 부드러운 재균형을 위해 virtual node (vnode)를 사용한 consistent hashing을 사용합니다.
Hash sharding: router가 key를 해싱하여 올바른 shard로 라우팅
## Cross-Shard 쿼리

Sharding의 가장 큰 운영 과제는 **cross-shard 쿼리** — 단일 shard로 답할 수 없는 쿼리입니다. 집계 (`COUNT`, `SUM`), shard key 경계를 넘는 range scan, 다른 shard의 레코드 간 `JOIN`은 모두 **scatter-gather** 방식이 필요합니다: 모든 shard에 쿼리를 병렬로 fan out하고, 결과를 수집하고, 애플리케이션 레이어에서 병합 및 정렬합니다. 이는 느리고 복잡합니다.

- **Scatter-gather 방지**: 쿼리를 설계할 때 shard key가 항상 필터에 포함되도록 합니다. '12월에 주문된 모든 주문' (모든 shard에 접근)보다 '사용자 X의 모든 주문' (사용자 X의 shard로 라우팅)과 같은 access 패턴을 선호합니다.
- **Secondary index**: secondary key를 shard ID에 매핑하는 별도의 lookup 테이블을 유지합니다 (다음 레슨에서 다루는 Index Table 패턴).
- **Global 집계**: 백그라운드 작업으로 집계를 사전 계산하고 전용 analytics store (예: ClickHouse, Redshift)에 저장합니다.

## Resharding과 재균형

데이터가 증가함에 따라 shard가 가득 차고 분할해야 합니다. **Resharding**은 운영 비용이 높습니다: 데이터를 마이그레이션하고, 라우팅 테이블을 업데이트하고, 무중단을 보장하기 어렵습니다. 미리 계획하는 것이 중요합니다:

- 초기에 over-shard: 물리적 node보다 더 많은 논리적 shard (예: 1024개)를 생성합니다. 여러 논리적 shard를 각 물리적 node에 매핑합니다. 용량 추가는 데이터를 re-hashing하지 않고 논리적 shard를 새 node로 이동하기만 하면 됩니다.
- Directory service 사용: shard-to-node 매핑을 consistent store (ZooKeeper, etcd)에 저장합니다. 매핑 업데이트는 즉각적이고 데이터는 백그라운드에서 이동합니다.
- 마이그레이션 중 double-write: 쓰기를 이전 shard와 새 shard 모두에 라우팅하고, consistency를 확인한 후 읽기를 전환합니다.

> 💡
> 인터뷰 팁
> 인터뷰에서 sharding을 제안한 후, 항상 세 가지 후속 질문을 선제적으로 다루세요: (1) 'Shard key를 어떻게 선택했나요?' — access 패턴 분석으로 정당화; (2) 'Cross-shard 쿼리를 어떻게 처리하나요?' — scatter-gather를 언급하고 대안을 제안; (3) '재균형을 어떻게 하나요?' — 논리적/가상 shard 또는 consistent hashing을 언급. Sharding의 문제점을 알고 있다는 것을 보여주면 실제 프로덕션 경험을 어필할 수 있습니다.

## 실제 Sharding 사례

| 회사 | 시스템 | Shard Key 전략 |
| --- | --- | --- |
| Instagram | PostgreSQL에 저장된 사용자 사진 | 논리적 shard 맵을 사용하여 user ID로 shard |
| Uber | 여행 database | 지리적 locality를 위해 city ID로 shard |
| Discord | Cassandra의 메시지 | (channel_id, bucket)으로 파티션 — 시간 버킷 |
| DynamoDB | 모든 테이블 | Hash partition key + range를 위한 선택적 sort key |
