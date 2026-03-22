# Database Sharding & Partitioning (데이터베이스 샤딩 및 파티셔닝)

> Source: https://www.sysdesai.com/learn/data-storage/sharding-partitioning

---

## Why Sharding Exists (샤딩이 존재하는 이유)

단일 데이터베이스 노드는 CPU, RAM, 디스크 I/O, 네트워크 처리량 등 하드웨어적인 한계가 있습니다. **Vertical Scaling(수직 확장, 더 큰 장비 도입)**은 결국 수확 체감의 법칙에 직면하고 비용 한계에 도달하게 됩니다. **Sharding(샤딩)** — **Horizontal Partitioning(수평 파티셔닝)**이라고도 함 — 은 전체 데이터셋의 하위 집합을 각각 소유하는 여러 노드(샤드)에 데이터를 분산시킵니다. 총 용량은 샤드 수에 따라 선형적으로 증가합니다.

## Vertical vs Horizontal Partitioning (수직 vs 수평 파티셔닝)

| Strategy(전략) | What Moves(이동 대상) | Best For(용도) | Limitation(한계) |
| --- | --- | --- | --- |
| Vertical Partitioning(수직 파티셔닝) | 컬럼을 테이블 간에 분할 (예: user_profiles와 user_preferences 분리) | 테이블 너비 감소, 자주 쓰는(hot) 컬럼과 덜 쓰는(cold) 컬럼 분리 | 여전히 단일 노드에 존재; 행(row)의 부피를 해결하지 못함 |
| Horizontal Partitioning(수평 파티셔닝, 샤딩) | Shard Key(샤드 키)를 기준으로 행을 여러 노드에 분할 | 단일 노드 용량을 넘어서는 확장 | 샤드 간(cross-shard) 쿼리가 비싸지거나 scatter-gather 방식이 필요함 |

## Sharding Strategies (샤딩 전략)

### Range-Based Sharding (범위 기반 샤딩)

Shard Key(샤드 키)의 연속된 범위를 기준으로 행을 샤드에 할당합니다. 예: 샤드 1은 `user_id` 1~1,000,000 보유, 샤드 2는 1,000,001~2,000,000 보유 등입니다. 범위 쿼리는 하나 또는 몇 개의 연속된 샤드만 건드리면 되므로 효율적입니다.

> ⚠️
> Hot Spot Problem (핫스팟 문제)
> 범위 샤딩은 쓰기가 편향될 때 핫스팟을 생성합니다. 사용자 ID가 순차적으로 증가(auto-increment)한다면 모든 새로운 사용자는 마지막 샤드에 저장됩니다. 다른 샤드들은 유휴 상태인데 하나의 샤드만 병목 현상이 발생합니다. 이는 고전적인 인터뷰 질문입니다: 'user_id 범위로 샤딩했는데 앱이 급격히 성장하면 어떤 일이 벌어질까요?'

### Hash-Based Sharding (해시 기반 샤딩)

Shard Key(샤드 키)에 해시 함수를 적용하고 `hash(key) % num_shards`를 수행하여 대상 샤드를 결정합니다. 이는 키의 분포와 상관없이 **데이터를 균등하게 분산**시켜 순차적 키로 인한 핫스팟을 제거합니다. 단점은 인접한 키들이 서로 다른 샤드로 해싱되기 때문에 **Range Query(범위 쿼리) 시 모든 샤드에 대해 scatter-gather가 필요**하다는 점입니다.

```python
def get_shard(user_id: int, num_shards: int) -> int:
    return hash(user_id) % num_shards

# user_id=1   → shard 1
# user_id=2   → shard 3
# user_id=3   → shard 0
# 키들이 흩어짐 — 핫스팟은 없으나 범위 지역성(locality)도 없음
```

### Directory-Based Sharding (디렉토리 기반 샤딩)

**Lookup Service(조회 서비스)**인 샤드 디렉토리가 키와 샤드 간의 매핑을 관리합니다. 이는 최고의 유연성을 제공합니다 — 재해싱 없이 개별 키를 샤드 간에 이동시킬 수 있습니다. 비용은 추가적인 네트워크 홉(hop)과 디렉토리 사용 불가능 시 발생하는 단일 장애점(Single Point of Failure) 위험입니다.

## Choosing a Shard Key (샤드 키 선택하기)

Shard Key(샤드 키)는 샤딩 시 가장 중대한 아키텍처 결정입니다. 잘못된 샤드 키는 핫스팟, 비싼 샤드 간 쿼리, 또는 두 문제 모두를 야기합니다. 좋은 샤드 키는 세 가지 속성을 가집니다:

1. **High Cardinality(높은 카디널리티)**: 데이터가 샤드 간에 고르게 분산되도록 다양한 구별된 값을 가져야 합니다 (값이 200개뿐인 `country_code` 같은 것은 부적절).
2. **Low Correlation with Write Patterns(쓰기 패턴과의 낮은 상관관계)**: 해시 샤딩의 경우 순차적으로 증가하는 키(auto-increment ID)를 피해야 하며, 범위 샤딩의 경우 시간적 핫스팟을 유발하므로 피해야 합니다.
3. **Query Alignment(쿼리 정렬)**: 샤드 키는 가장 빈번한 액세스 패턴과 일치해야 합니다. 쿼리의 90%가 `tenant_id`로 필터링한다면, `tenant_id`로 샤딩하세요.

| Shard Key(샤드 키) | Hot Spot Risk(핫스팟 위험) | Range Query(범위 쿼리) | Notes(참고) |
| --- | --- | --- | --- |
| user_id (hash) | 낮음 | Scatter-gather | 사용자 중심 시스템의 좋은 기본값 |
| created_at (range) | 높음 (최근 쓰기) | 매우 좋음 | 나쁨 — 모든 쓰기가 최신 샤드에 몰림 |
| tenant_id (hash) | 중간 (거대 테넌트) | 테넌트별 | 멀티 테넌트 SaaS에 적합 |
| random UUID (hash) | 없음 | 불가능 | 최대의 쓰기 분산, 지역성 없음 |

## Rebalancing Shards (샤드 재균형)

샤드를 추가할 때 기존 데이터를 다시 분산시켜야 하며, 이를 **Rebalancing(재균형)**이라고 합니다. 단순 해시 샤딩(`hash % n`)은 재균형 시 재항적입니다: `n`을 4에서 5로 바꾸면 전체 키의 약 80%가 다시 매핑되어 대규모 데이터 마이그레이션이 필요합니다.

**Consistent Hashing(일관된 해싱)**이 이를 해결합니다. 키와 노드가 링(ring) 위에 매핑됩니다. 노드가 추가될 때 새 노드와 그 이전 노드 사이의 키들만 이동합니다 — 보통 전체 데이터의 `1/n` 정도입니다. DynamoDB, Cassandra, Riak이 재균형을 처리하는 방식입니다.

> 💡
> Virtual Nodes (vnodes, 가상 노드)
> 분산의 균등함을 개선하기 위해 각 물리적 노드는 링 위에서 여러 개의 **Virtual Nodes(가상 노드)**로 표현됩니다. Cassandra는 노드당 기본 256개의 vnode를 사용합니다. 이는 노드가 실패하거나 추가될 때 부하가 인접한 노드에만 쏠리지 않고 남은 모든 노드에 균등하게 재분산되도록 보장합니다.

## Cross-Shard Queries and Joins (샤드 간 쿼리 및 조인)

샤딩에서 운영상 가장 고통스러운 부분은 **Cross-shard Queries(샤드 간 쿼리)**입니다. `SELECT * FROM orders WHERE status = 'pending'`과 같은 쿼리는 모든 샤드에 브로드캐스트(**Scatter**)되어야 하고, 결과를 수집(**Gather**)하여 병합해야 합니다. 이는 느리고 비싸며 복잡합니다. 연관된 데이터를 동일한 샤드에 유지함으로써 핫 패스(hot path)에서 샤드 간 쿼리가 발생하지 않도록 스키마를 설계하세요.

> 💡
> Interview Tip (인터뷰 팁)
> "데이터베이스를 어떻게 확장(scale)할 것인가?"라는 질문을 받으면 다음과 같이 답변을 구성하세요: (1) 먼저 Read Replica(읽기 복제본)를 추가합니다 — 대부분의 앱에서 부하의 80%를 처리할 수 있습니다. (2) 캐시(Redis)를 도입하여 DB 부하를 줄입니다. (3) 그 다음에야 샤딩을 고려합니다. 샤딩은 엄청난 운영 복잡성을 추가합니다. 면접관들은 샤딩을 첫 번째 본능이 아닌 최후의 수단으로 취급하는 후보자를 존중합니다.
