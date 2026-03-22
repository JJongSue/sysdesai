# Application-Level Caching (애플리케이션 레벨 캐싱)

> Source: https://www.sysdesai.com/learn/caching/application-caching

---

## The Cache Hierarchy (캐시 계층 구조)

잘 설계된 시스템에서는 단일 캐시 계층만 존재하는 경우가 드뭅니다 — 크기, 지연 시간, 신선도 사이의 트레이드오프를 고려한 **Cache Hierarchy(캐시 계층 구조)**가 존재합니다. 가장 빠른 것부터 느린 순서대로: CPU L1/L2/L3 캐시, 인프로세스(In-process) 애플리케이션 메모리, 분산 캐시(Redis/Memcached), CDN 엣지 캐시, 그리고 마지막으로 오리진 데이터베이스나 객체 저장소입니다. 애플리케이션 레벨 캐싱은 애플리케이션 서버의 자체 메모리 힙에 저장되는 **In-process(인프로세스)** 및 **Request-scoped(요청 범위)** 계층을 의미합니다.

## In-Process (Local) Caching (인프로세스/로컬 캐싱)

**In-process cache(인프로세스 캐시)**는 데이터를 애플리케이션 서버의 힙(heap)에 직접 저장하므로 네트워크 라운드 트립이 필요 없습니다. 액세스 시간은 Redis로의 네트워크 홉(~1ms)이 아닌 나노초에서 마이크로초 단위입니다. 이는 수천 배 이상 빠릅니다.

일반적인 구현 사례: Java의 **Caffeine** (최적에 가까운 히트율을 가진 비동기 LW-W 캐시), Python의 `functools.lru_cache`, Node.js의 인메모리 맵, 또는 Rails의 `memory_store`와 같은 프레임워크 전용 캐시 등이 있습니다. Caffeine은 Netflix, Google, Spring Boot의 기본 캐시로 내부적으로 사용됩니다.

> ⚠️
> 인프로세스 캐시의 일관성 문제 (Consistency challenge)
> 10대의 애플리케이션 서버를 운영한다면, 각 서버는 자신만의 로컬 캐시를 가집니다. 레코드를 업데이트한 후에는 **모든 서버의 로컬 캐시를 무효화(invalidate)**하거나, TTL 만료 전까지 일부 서버가 오래된 데이터를 제공하는 것을 감수해야 합니다. 짧은 TTL(초 단위)의 경우 이는 대개 문제가 되지 않지만, 수명이 긴 데이터의 경우 분산 캐시를 사용하거나 Pub/Sub 무효화 신호(예: Redis pub/sub을 통한 브로드캐스트)를 사용해야 합니다.

## Memoization (메모이제이션)

**Memoization(메모이제이션)**은 함수 레벨의 캐싱입니다: 특정 인자와 함께 순수 함수(pure function)를 처음 호출하면 결과를 계산하여 저장하고, 동일한 인자로 다시 호출하면 저장된 결과를 반환합니다. 이는 인프로세스 캐싱의 가장 단순한 형태입니다.

```python
from functools import lru_cache

@lru_cache(maxsize=1024)
def get_tax_rate(country_code: str, product_category: str) -> float:
    """세율 규칙 데이터베이스에서 비싼 조회를 수행합니다. 인프로세스에 메모이제이션됩니다."""
    return db.query(
        "SELECT rate FROM tax_rules WHERE country=? AND category=?",
        country_code, product_category
    )

# 첫 번째 호출: DB 조회 발생
rate = get_tax_rate("US", "electronics")  # ~20ms

# 두 번째 호출: 캐시에서 반환
rate = get_tax_rate("US", "electronics")  # ~0.001ms
```

Python의 `lru_cache`는 스레드 안전하며 `maxsize` 파라미터로 LRU 축출을 구현합니다. 운영 환경에서는 항목이 만료되어 프로세스 재시작 전까지 낡은 데이터가 유지되지 않도록 TTL을 지원하는 라이브러리(예: `cachetools.TTLCache`)를 사용하는 것이 좋습니다.

## Request-Scoped Caching (요청 범위 캐싱)

**Request-scoped caching** (Per-request caching 또는 DataLoader 배칭이라고도 함)은 단일 요청의 수명 동안만 데이터를 저장합니다. 이는 **GraphQL 서버**나 복잡한 서비스 호출 그래프에서 특히 유용한데, 하나의 요청 내에서 서로 다른 리졸버(resolver)들이 동일한 엔티티를 여러 번 가져와야 하는 경우에 유용합니다.

```typescript
// DataLoader (Facebook의 오픈소스 라이브러리)
// 단일 요청 컨텍스트 내에서 DB 호출을 배치로 묶고 중복을 제거합니다.
import DataLoader from 'dataloader';

const userLoader = new DataLoader(async (ids: string[]) => {
  // 100개의 리졸버가 사용자를 요청하더라도 요청 주기당 한 번만 호출됩니다.
  const users = await db.query(
    'SELECT * FROM users WHERE id = ANY(?)', [ids]
  );
  // 결과는 ids와 동일한 순서로 반환되어야 합니다.
  return ids.map(id => users.find(u => u.id === id) ?? null);
});

// GraphQL 리졸버에서의 사용 — 100번의 호출이 1번의 DB 쿼리로 변합니다.
const user = await userLoader.load(userId);
```

요청 범위 캐싱이 없다면, 100개의 포스트와 그 작성자를 가져오는 GraphQL 쿼리는 100번의 개별 `SELECT * FROM users WHERE id = ?` 쿼리를 발생시킬 수 있습니다 (**N+1 문제**). DataLoader는 이를 하나의 `SELECT * FROM users WHERE id IN (...)`으로 묶어 처리하고 결과를 요청 내에서 캐싱하여 리졸버 코드에서는 이를 투명하게 이용할 수 있게 합니다.

## Computed / Derived Value Caching (계산/유도 값 캐싱)

일부 값들은 계산 비용은 비싸지만 저장 비용은 저렴합니다: 추천 점수, 집계 통계, 렌더링된 HTML 조각, 또는 직렬화된 API 응답 등이 그 예입니다. **Computed caching**은 이러한 값들을 미리 계산하여 저장해 둠으로써 매 요청마다 재계산하는 것을 피합니다.

- **핫 패스 최적화 (Hot path optimization)**: 매 요청마다 50개의 DB 쿼리로 조립하는 대신, 홈페이지 피드를 한 번 계산하여 30초 동안 캐싱합니다.
- **백그라운드 갱신 (Background refresh)**: 백그라운드 작업이 매분 비싼 집계 값을 재계산하여 캐시에 씁니다; 요청은 항상 미리 계산된 결과를 읽습니다.
- **프래그먼트 캐싱 (Fragment caching)**: 제품 카드와 같이 렌더링된 HTML/JSON 조각을 캐싱하여 템플릿 렌더링 오버헤드를 줄입니다.

## When Local Beats Distributed (로컬 캐시가 분산 캐시보다 나은 경우)

| Dimension(차원) | In-Process Cache(인프로세스 캐시) | Distributed Cache (Redis) |
| --- | --- | --- |
| Latency(지연 시간) | 나노초 (네트워크 없음) | ~0.3–2 ms (네트워크 RTT) |
| Consistency(일관성) | 서버별 — 복제본 간 불일치 발생 가능 | 모든 앱 서버 간 일관성 유지 |
| Capacity(용량) | 서버 힙 크기로 제한 (GB 단위) | 전용 노드 (수백 GB 단위) |
| Fault tolerance(장애 내성) | 재시작 시 유실; 복제 없음 | 지속성 지원 (AOF/RDB); 복제 지원 |
| Best For(용도) | 설정값, 조회 테이블, 요청당 중복 제거 | 세션 데이터, 공유 상태, 분산 락 |

> 💡
> 2계층 캐싱 (Two-tier caching)
> 두 계층을 결합하세요. 먼저 인프로세스 캐시를 확인(나노초)하고, 미스 발생 시 Redis를 확인(~1ms)하며, 거기서도 미스 발생 시 데이터베이스를 쿼리합니다. 데이터베이스 읽기 시 두 캐시를 모두 채웁니다. 이 2계층 접근 방식은 Facebook의 Memcache 아키텍처(지역 클러스터 + 로컬 인프로세스 캐시)에서 사용되며, Caffeine + Redis를 사용하는 고성능 Java 서비스에서도 흔히 볼 수 있는 패턴입니다.

> 💡
> Interview Tip (인터뷰 팁)
> 많은 지원자들이 즉시 "Redis를 추가하겠다"고 말하며 애플리케이션 레벨 캐싱을 간과하곤 합니다. 전체 계층 구조를 언급하여 깊이를 보여주세요: "Redis를 추가하기 전에, 짧은 TTL을 가진 인프로세스 캐싱으로 부하를 처리할 수 있는지 먼저 확인하겠습니다. 이는 지연 시간이 제로이며 Redis 부하를 줄여줍니다. GraphQL의 N+1 문제에는 DataLoader를 사용하겠습니다. 그 다음 공유 세션 상태나 분산 락을 위해 Redis를 사용하겠습니다." 이러한 계층적 사고는 엔지니어링적 성숙도를 나타냅니다.
