# Cache-Aside 패턴

> 출처: https://www.sysdesai.com/learn/performance-scalability/cache-aside

---

## Cache-Aside란?

**Cache-Aside** (또는 **Lazy Loading**이라고도 함)는 프로덕션 시스템에서 가장 널리 사용되는 caching 패턴입니다. 애플리케이션 코드가 cache를 직접 관리하며, 데이터는 실제로 요청될 때만 cache에 로드됩니다. 애플리케이션은 cache hit와 cache miss를 모두 명시적으로 처리해야 합니다. Cache와 data store는 서로를 알지 못하며, 애플리케이션이 중간에서 모든 읽기와 쓰기를 중재합니다.

이 패턴은 Netflix, Twitter, Airbnb 등의 기업에서 Redis 기반 caching layer의 기본 선택지입니다. 엔지니어가 cache에 무엇을 넣을지, 언제 만료시킬지, 어떻게 invalidate할지를 완전히 제어할 수 있기 때문입니다.

## Cache-Aside Read Flow

Read path는 일관된 3단계 확인 과정을 따릅니다. 모든 읽기 요청에서 애플리케이션은 먼저 cache를 확인합니다. 값이 존재하면 (**cache hit**) 즉시 반환됩니다. 값이 없으면 (**cache miss**) 애플리케이션은 primary data store에서 데이터를 가져오고, 적절한 TTL과 함께 결과를 cache에 저장한 후 호출자에게 값을 반환합니다.
Cache-Aside read flow: miss path는 DB에서 데이터를 가져와 cache를 채웁니다
## Cache-Aside Write Flow

쓰기 시, 애플리케이션은 **primary data store를 직접 업데이트**한 후, 해당 cache 항목을 업데이트하는 대신 **invalidate** (삭제)합니다. 업데이트된 값은 다음 읽기 요청 시 다시 채워집니다. 이렇게 하면 동시 요청이 새로운 데이터를 가져오는 동안 stale 값을 cache에 쓰는 race condition을 방지할 수 있습니다.

> ⚠️
> Invalidate, Don't Update
> 쓰기 시에는 새 값을 쓰는 대신 항상 cache key를 삭제하세요. 쓰기 시 cache를 업데이트하면 race condition이 발생합니다: 두 개의 동시 writer가 잘못된 순서로 cache를 업데이트하여 stale 데이터가 영구적으로 남을 수 있습니다.

python

```
def get_user(user_id: str) -> User:
    cache_key = f"user:{user_id}"

    # Step 1: Try cache first
    cached = redis.get(cache_key)
    if cached:
        return deserialize(cached)

    # Step 2: Cache miss — fetch from DB
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    if not user:
        return None

    # Step 3: Populate cache with TTL
    redis.setex(cache_key, ttl=3600, value=serialize(user))
    return user

def update_user(user_id: str, data: dict) -> None:
    # Step 1: Write to DB first
    db.execute("UPDATE users SET ... WHERE id = ?", user_id, data)

    # Step 2: Invalidate cache (delete, not update)
    redis.delete(f"user:{user_id}")
```

## Consistency 고려사항

Cache-Aside는 **eventual consistency**를 제공합니다. 쓰기가 cache를 invalidate한 시점과 reader가 다시 채우는 시점 사이에 모든 reader는 database로 직접 접근합니다. 이는 일반적으로 허용 가능하지만 알아두어야 할 두 가지 edge case가 있습니다:

- **쓰기 후 stale read**: Invalidation이 실패하면 (Redis에 대한 네트워크 오류), cache는 TTL 만료까지 stale 데이터를 보유합니다. 안전망으로 항상 적절한 TTL을 설정하세요.
- **Cold start 페널티**: 새로 배포된 서비스나 cache flush는 모든 요청이 database에 직접 접근하게 만듭니다. 예측 가능한 cold-start 시나리오에는 cache warming 전략을 사용하세요.
- **Cache stampede (thundering herd)**: 인기 있는 key가 만료되면 많은 동시 요청이 모두 miss되어 동시에 database를 쿼리하여 급격한 부하가 발생합니다.

## Cache Stampede 방지

트래픽이 많은 cache key가 만료되면 수백 개의 동시 요청이 한꺼번에 database를 공격할 수 있습니다 — **cache stampede** 또는 **thundering herd** 문제입니다. 세 가지 표준 완화 방법이 있습니다:

| 기법 | 작동 방식 | Trade-off |
| --- | --- | --- |
| Mutex / Lock | 첫 번째 miss가 distributed lock을 획득하고 재채움; 나머지는 대기 | 대기자에게 추가 latency 발생; 실패 시 lock 해제 필요 |
| Probabilistic Early Expiry | TTL 만료 직전에 일정 확률로 값을 재계산 | 약간의 과잉 계산이 있지만 lock contention 없음 |
| Background Refresh | stale 데이터를 즉시 제공; async worker가 백그라운드에서 갱신 | stale-ok window를 위한 별도 TTL 필요 |

> 💡
> 인터뷰 팁
> 면접관들은 cache stampede에 대해 자주 질문합니다. mutex lock 방식을 먼저 설명한 후, lock-free 대안으로 probabilistic early expiry ('XFetch'라고도 함)를 언급하세요. 두 가지 모두 알고 있다는 것을 보여주면 깊이 있는 지식을 어필할 수 있습니다. 또한 Redis 6.2+에는 distributed locking을 위한 내장 `GETDEL` + `SET NX` 패턴이 있다는 것도 언급하세요.

## Cache-Aside를 선택해야 할 때

| 상황 | Cache-Aside 적합? | 이유 |
| --- | --- | --- |
| 읽기 많고 쓰기 적은 경우 | 예 | Cache hit rate가 높고 invalidation이 적음 |
| 약간의 stale 데이터 허용 가능 | 예 | TTL 기반 consistency로 충분 |
| 예측 불가능한 access pattern | 예 | 인기 있는 데이터만 cache에 저장됨 |
| 쓰기 많은 workload | 아니오 | 지속적인 invalidation으로 hit rate 낮음 |
| 강한 consistency 필요 | 아니오 | 쓰기와 재채움 사이에 stale read 가능 |

📌
실제 사례: Twitter의 Timeline Cache

Twitter는 Cache-Aside를 사용하여 Redis에 사전 계산된 timeline을 저장합니다. 사용자가 피드를 로드하면 앱은 먼저 Redis를 확인합니다. Miss 시, fanout service가 Cassandra에서 timeline을 조합하여 다시 저장합니다. Timeline은 몇 초의 staleness를 허용할 수 있으므로 TTL 기반 방식이 강한 consistency 없이도 완벽하게 작동합니다.
