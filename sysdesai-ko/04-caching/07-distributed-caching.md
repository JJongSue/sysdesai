# Distributed Caching (분산 캐싱)

> Source: https://www.sysdesai.com/learn/caching/distributed-caching

---

## Why Distribute a Cache? (캐시를 분산하는 이유)

단일 캐시 노드는 유한한 RAM을 가집니다. 작업 세트(Working set)가 한 대의 머신 메모리를 초과하거나, 처리해야 할 QPS가 한 대의 한계를 넘어서면 **캐시를 여러 노드에 걸쳐 샤딩(Sharding)**해야 합니다. 분산 캐싱은 분산 시스템의 복잡성을 그대로 가져옵니다: 요청을 어떻게 올바른 노드로 라우팅할 것인가? 노드가 실패하면 어떤 일이 벌어지는가? 전체 캐시를 무효화하지 않고 어떻게 용량을 증설할 것인가?

## Naive Sharding: Modulo Hashing (단순 샤딩: 나머지 연산 해싱)

가장 단순한 샤딩 방식은 `node_index = hash(key) % num_nodes`입니다. 모든 클라이언트가 동일한 공식을 계산하여 동일한 노드로 라우팅합니다. 문제는 노드를 추가하거나 제거할 때 `num_nodes`가 변하며, 거의 **모든 키가 다른 노드로 매핑**된다는 점입니다. 모든 캐시 데이터가 동시에 무효화되는 것과 같으며, 데이터베이스는 캐시 미스의 파도에 휩쓸리게 됩니다. 이를 **Cache Avalanche(캐시 눈사태)**라고 합니다.

> ⚠️
> 단순 재샤딩으로 인한 캐시 눈사태 (Cache avalanche from naive resharding)
> 10대의 노드 클러스터에서 나머지 연산 해싱을 사용할 경우, 노드를 10대에서 11대로 늘리면 약 (N/(N+1)) = ~91%의 키가 다른 노드로 재매핑됩니다. 이는 운영 시스템에 재앙과도 같습니다. **Consistent Hashing(일관된 해싱)**은 토폴로지 변경 시 재매핑을 최소화하여 이를 해결합니다.

## Consistent Hashing (일관된 해싱)

**Consistent Hashing(일관된 해싱)**은 노드와 키를 모두 원형의 해시 링(0 ~ 2^32) 위에 배치합니다. 각 키는 링 위에서 **시계 방향으로 가장 가까운 노드**에 할당됩니다. 노드가 추가되거나 제거될 때, 해당 노드와 링 위의 이전 노드 사이에 있는 키들만 재매핑됩니다 — N대의 노드 클러스터에서 대략 `1/N` 정도의 키만 이동합니다. 이는 확장 시 캐시 무효화를 최소화합니다.

**Virtual Nodes (vnodes, 가상 노드)**: 하나의 물리적 노드가 링 위에서 여러 위치(예: 물리 노드당 150개의 가상 노드)로 표현됩니다. 이는 부하 분산을 개선합니다 — 가상 노드가 없다면 불균형한 링 구조로 인해 특정 노드가 다른 노드보다 훨씬 넓은 키 범위를 담당하게 될 수 있습니다. 또한 가상 노드를 사용하면 성능이 더 좋은 노드에 더 많은 링 위치를 할당하기 쉬워집니다. Redis Cluster는 16,384개의 해시 슬롯(고정된 크기의 이산적인 링)을 일관된 해싱 매커니즘으로 사용합니다.

## Replication in Distributed Caches (분산 캐시의 복제)

샤딩은 키를 노드 간에 분할하지만 노드 장애로부터 보호해주지는 않습니다. **Replication(복제)**이 그 보호 기능을 제공합니다: 각 프라이머리(Primary) 노드는 동일한 키의 복사본을 가진 하나 이상의 복제본(Replica)을 가집니다. 읽기 요청은 복제본에서도 처리할 수 있어 처리량이 향상되며; 쓰기 요청은 프라이머리로 전달된 후 비동기적으로 복제됩니다.

| Replication Model(복제 모델) | Write Path(쓰기 경로) | Read Path(읽기 경로) | Failure Behavior(장애 시 동작) | Example(예시) |
| --- | --- | --- | --- | --- |
| Primary-Replica | 프라이머리에만 쓰기 | 프라이머리 또는 복제본 | 장애 조치 시 복제본을 프라이머리로 승격 | Redis Sentinel |
| Multi-Primary | 모든 프라이머리에 쓰기 | 모든 노드 | 파티션 내성; 충돌 해결 필요 | 다중 프라이머리 기반 Redis Cluster |
| Quorum Writes | W개 노드에 쓰기; W개 응답 시 성공 | R개 노드에서 읽기 | W+R > N을 통한 일관성 조절 가능 | Dynamo 스타일 시스템 |

## Cache Avalanche vs Cache Stampede (캐시 눈사태 vs 캐시 스탬피드)

자주 혼동되는 두 가지 치명적인 분산 캐시 실패 모드입니다:

| Failure Mode(실패 모드) | Cause(원인) | Effect(영향) | Prevention(방지책) |
| --- | --- | --- | --- |
| Cache Stampede | 하나의 인기 있는 키가 만료; 많은 동시 요청이 미스 발생 | 하나의 키를 위한 N개의 동시 쿼리가 DB에 집중됨 | Mutex 락, Stale-while-revalidate, 지터링된 TTL |
| Cache Avalanche | 많은 키가 동시에 만료 (예: 콜드 리스타트 또는 단순 재샤딩 후) | 모든 키에 걸친 거대한 미스의 파도가 DB로 몰림 | 분산된(staggered) TTL, 웜업 전략, 점진적 트래픽 전환 |
| Cache Penetration | 공격자가 존재하지 않는 키를 계속 요청 (항상 캐시 미스) | 모든 요청이 DB로 전달됨; 캐시의 보호 기능 무력화 | 존재하지 않는 키 차단을 위한 Bloom Filter; null 값 캐싱 |

## Cache Penetration and Bloom Filters (캐시 관통과 블룸 필터)

**Cache Penetration(캐시 관통)**은 서비스 거부(DoS) 패턴의 일종입니다: 공격자가 존재하지 않는 ID(예: `user:-1`, `product:99999999`)를 계속 쿼리합니다. 모든 요청은 캐시 미스가 되어 데이터베이스로 향합니다. 캐시는 아무런 보호 기능을 제공하지 못합니다. 해결책: **Bloom Filter(블룸 필터)**를 사용하는 것입니다 — 이는 '확실히 존재하지 않음' 또는 '아마도 존재함'을 빠르게 알려주는 확률적 데이터 구조입니다. 모든 유효한 사용자 ID에 대한 블룸 필터를 애플리케이션에 로드해 두고 캐시나 DB 호출 전에 먼저 확인할 수 있습니다.

```python
# 캐시/DB 조회 전 블룸 필터 확인
# pybloom_live 등의 라이브러리 사용 가정
from pybloom_live import ScalableBloomFilter

# 시작 시 DB에서 데이터를 가져와 필터링 채우기
user_bloom = ScalableBloomFilter()
for user_id in db.query("SELECT id FROM users"):
    user_bloom.add(user_id)

def get_user(user_id: str) -> User:
    # 캐시나 DB를 건드리지 않고 불가능한 ID를 거절
    if user_id not in user_bloom:
        raise NotFoundError("User does not exist")

    # 이제 안심하고 캐시 확인
    cached = cache.get(f"user:{user_id}")
    if cached:
        return deserialize(cached)

    # 캐시 미스 — DB 조회
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    if user:
        cache.set(f"user:{user_id}", serialize(user), ttl=300)
    return user
```

## Hot Key Problem (핫 키 문제)

일관된 해싱은 키를 **평균적으로** 고르게 분산시키지만, 특정 키는 연예인의 프로필 페이지, 화제의 뉴스 기사, 또는 반짝 세일 제품처럼 불균형하게 많은 트래픽을 받습니다. 하나의 키에 대해 초당 수백만 건의 요청을 처리하는 단일 노드는 다른 키들이 아무리 잘 분산되어 있어도 **Hot Spot(핫스팟)**이 됩니다.

핫 키 문제의 해결책:

- **Local in-process cache(로컬 인프로세스 캐시)**: 핫 키를 모든 애플리케이션 서버의 인프로세스 캐시에 넣습니다. 읽기 요청이 프로세스를 벗어나지 않으므로 네트워크 오버헤드가 제로입니다.
- **Key Replication(키 복제)**: 핫 키를 여러 개의 복사본(`product:trending#0`, `product:trending#1` ... `#N`)으로 쪼개어 여러 노드에 분산시킵니다. 읽기 요청 시 무작위로 하나의 샤드를 선택합니다.
- **Read Replicas(읽기 복제본)**: 핫 키가 있는 프라이머리 노드의 읽기 트래픽을 복제본으로 돌려 부하를 여러 물리 장비로 분산시킵니다.
- **Redis Cluster read-from-replica**: 복제본 연결에 `READONLY` 설정을 하여 읽기를 허용함으로써 프라이머리 부하를 줄입니다.

## Gradual Cache Warm-Up (점진적 캐시 웜업)

콜드 스타트(클러스터 재시작, 대규모 재샤딩 또는 확장) 이후 캐시는 비어 있습니다. 모든 운영 트래픽을 즉시 보내면 데이터베이스 눈사태(avalanche)가 발생합니다. **Gradual Warm-up(점진적 웜업)**을 사용하세요: 처음에는 트래픽의 아주 적은 비율만 새 클러스터로 보내고 나머지는 기존 클러스터가 처리하게 합니다. 히트율이 올라감에 따라(`cache_hits / (cache_hits + cache_misses)` 모니터링) 더 많은 트래픽을 전환합니다. 또는 트래픽 전환 전에 최근의 데이터베이스 읽기 기록을 새 클러스터에 재현(replay)하여 미리 캐시를 채울 수도 있습니다.

> 💡
> Interview Tip (인터뷰 팁)
> 분산 캐싱 질문은 대규모 데이터셋을 다루는 제품의 시니어 시스템 디자인 인터뷰에서 자주 등장합니다. 다음 네 가지 영역을 다루세요: (1) **Sharding(샤딩)** — 재매핑을 최소화하는 일관된 해싱; (2) **Replication(복제)** — 결함 내성을 위한 프라이머리 + 복제본; (3) **Failure Modes(실패 모드)** — 캐시 스탬피드(단일 키), 캐시 눈사태(대량 만료), 캐시 관통(유효하지 않은 키); (4) **Hot Keys(핫 키)** — 로컬 인프로세스 캐싱 또는 키 복제. 캐시 관통 해결을 위해 블룸 필터를 언급하는 것은 대부분의 후보자가 하지 못하는 높은 수준의 답변입니다.

Practice this pattern
[Design a distributed caching system for a social media feed](https://www.sysdesai.com/design/new?prompt=Design%20a%20distributed%20caching%20system%20for%20a%20social%20media%20feed&mode=fast)
