# Cache Invalidation Patterns (캐시 무효화 패턴)

> Source: https://www.sysdesai.com/learn/caching/cache-invalidation

---

## The Hardest Problem (가장 어려운 문제)

Phil Karlton의 유명한 명언이 있습니다: "컴퓨터 과학에는 오직 두 가지 어려운 것이 있다: 캐시 무효화와 이름 짓기." 캐시 무효화가 어려운 이유는 **언제 캐싱된 값을 만료시킬지** 알기 위해 원본 데이터가 언제 변경되었는지 알아야 하기 때문입니다. 그리고 분산 시스템에서는 그 변경이 다른 서버, 다른 서비스에서 발생하거나 캐시를 전혀 거치지 않는 경로를 통해 발생할 수 있습니다.

무효화를 잘못 처리했을 때의 결과는 명확하고 고통스럽습니다: 할인 적용 후에도 사용자가 이전 가격을 보거나, 수정한 후에도 예전 프로필 사진이 보이거나, 구매 후에도 잘못된 재고 수량이 표시되는 등의 현상입니다. 이 강의에서는 주요 무효화 전략과 운영 시의 실패 모드(failure modes)를 다룹니다.

## TTL-Based Invalidation (TTL 기반 무효화)

**TTL (Time-To-Live)**은 가장 단순하고 일반적인 무효화 메커니즘입니다. 모든 캐시 항목에 최대 수명이 부여되며, 만료 후 다음 요청 시 원본 데이터로부터 다시 데이터를 가져오게 됩니다. TTL 기반 무효화는 조율이 필요 없습니다 — 캐시가 독립적으로 만료를 처리하기 때문입니다.

적절한 TTL을 설정하는 것은 비즈니스 결정입니다: 어느 정도의 데이터 지연(staleness)이 허용되는가? 주식 가격: 1초. 제품 설명: 1시간. 국가 코드: 24시간. 정적 자산: 무제한 (버전이 지정된 URL 사용). 핵심은 **TTL이 단순함을 위해 지연 시간과 타협한다**는 점입니다 — 즉, 데이터가 최대 TTL 초만큼 최신이 아닐 수 있음을 수용하는 것입니다.

> ⚠️
> 캐시 스탬피드 (Cache stampede / Thundering herd)
> 인기 있는 캐시 키가 만료되면, 해당 키에 대한 모든 동시 요청이 동시에 캐시 미스를 겪고 이를 다시 생성하기 위해 경쟁합니다 — 각 요청이 데이터베이스에 비싼 쿼리를 날리게 됩니다. 이는 데이터베이스가 가장 취약한 시점(트래픽이 높을 때)에 과부하를 줄 수 있습니다. 이를 **Cache Stampede(캐시 스탬피드)** 또는 Thundering Herd(천둥 치는 들소 떼) 현상이라고 합니다.

## Cache Stampede Prevention (캐시 스탬피드 방지)

캐시 스탬피드를 방지하는 몇 가지 기법이 있습니다:

- **Mutex / Lock on cache miss**: 하나의 프로세스만 키를 다시 생성하고, 나머지는 대기하거나 기존의 낡은 데이터를 제공합니다. Redis의 `SET NX EX` (만료 시간이 있는 '존재하지 않을 때만 설정')는 표준적인 분산 락(distributed lock) 기본 명령입니다.
- **Probabilistic Early Expiry (확률적 조기 만료)**: TTL이 만료되기 전에 일부 요청이 선제적으로 캐시를 갱신합니다. TTL이 0에 가까워질수록 갱신 확률이 높아지는 'XFetch' 알고리즘이 있습니다.
- **Stale-while-revalidate**: 백그라운드 프로세스가 캐시를 갱신하는 동안 모든 호출자에게 기존의 낡은 값을 제공합니다. 사용자는 캐시 미스를 겪지 않으며, 백그라운드 갱신이 재계산 비용을 흡수합니다.
- **Jittered TTLs (지터링된 TTL)**: TTL에 무작위 지터(`ttl + random(0, jitter)`)를 추가하여 배치로 로드된 모든 항목이 동시에 만료되지 않도록 합니다.

```python
import redis
import time
import random

r = redis.Redis()

def get_with_mutex(key: str, recompute_fn, ttl: int):
    """캐시에서 값을 가져오되, 스탬피드 방지를 위해 Mutex 락을 사용합니다."""
    value = r.get(key)
    if value:
        return value

    # 캐시 미스 — 락 획득 시도
    lock_key = f"lock:{key}"
    acquired = r.set(lock_key, "1", nx=True, ex=10)  # 10초 락 타임아웃

    if acquired:
        try:
            # 락을 획득함 — 재계산 수행
            value = recompute_fn()
            jitter = random.randint(0, int(ttl * 0.1))  # 10% 지터 추가
            r.set(key, value, ex=ttl + jitter)
            return value
        finally:
            r.delete(lock_key)
    else:
        # 다른 프로세스가 재계산 중임 — 잠시 기다린 후 재시도
        time.sleep(0.05)
        return r.get(key) or recompute_fn()  # 락 소유자 실패 시 폴백
```

## Event-Driven Invalidation (이벤트 기반 무효화)

**Event-driven invalidation**은 TTL 만료를 기다리는 대신 데이터 변경 이벤트에 응답하여 캐시 항목을 무효화합니다. 사용자가 데이터베이스에서 프로필을 업데이트하면, 서비스는 `user.updated` 이벤트를 발행하고; 캐시 무효화 구독자(subscriber)가 해당 캐시 키를 삭제하거나 다시 생성합니다.

이벤트 기반 무효화는 TTL보다 **정확합니다** — 변경 직후에 캐시 항목이 무효화되기 때문입니다. 하지만 인프라 복잡성이 추가됩니다: 신뢰할 수 있는 이벤트 버스가 필요하고, 무효화 구독자가 중복 이벤트(at-least-once delivery)를 처리해야 하며, 이벤트에는 영향을 받는 캐시 키를 식별할 수 있는 충분한 정보가 담겨 있어야 합니다.

> ⚠️
> 이벤트 순서 및 레이스 컨디션 (Event ordering and race conditions)
> 비동기 무효화 환경에서는 데이터베이스 쓰기와 캐시 삭제 사이에 낡은 데이터가 제공되는 구간이 존재합니다. 더 나쁜 경우: 삭제 이벤트가 도착하기 전에 캐시가 다시 생성되면, 업데이트 전의 낡은 값이 캐싱될 수 있습니다. 버전이 지정된 값을 사용하거나 조건부 쓰기를 통해 해결할 수 있습니다: 이벤트의 버전이 캐싱된 버전보다 최신일 때만 캐시에 씁니다.

## Versioned / Tag-Based Invalidation (버전/태그 기반 무효화)

**Versioned invalidation**은 캐시 키에 버전 번호를 포함합니다. 원본 데이터가 변경되면 버전이 증가하며; 모든 캐시 조회는 새로운 버전을 사용하여 미스가 발생하고 자연스럽게 데이터가 갱신됩니다. 이전 항목들은 TTL에 의해 만료되도록 둡니다 (더 이상 액세스되지 않으므로 고립됩니다).

```python
def get_user_cache_key(user_id: str) -> str:
    # 빠른 카운터(Redis나 DB)에 저장된 버전 정보
    version = get_user_version(user_id)   # 예: "v7"
    return f"user:{user_id}:{version}"

def invalidate_user(user_id: str):
    # 버전을 증가시킴 — 이전 키들은 모두 무효화됨
    increment_user_version(user_id)
    # 이전 캐시 키(예: user:42:v6)는 TTL에 의해 만료됨
    # 새로운 요청은 user:42:v7에서 미스가 발생하고 다시 채워짐
```

**Cache tags** (Fastly와 Cloudflare 지원)는 그룹화된 무효화의 CDN 레벨 형태입니다. 응답에 여러 식별자 태그를 붙입니다: `Surrogate-Key: product:42 category:electronics`. `product:42`에 대한 한 번의 Purge API 호출로 해당 제품 태그가 붙은 모든 응답을 무효화할 수 있습니다 — 수많은 서로 다른 URL에 걸쳐 있더라도 말이죠.

## Stale-While-Revalidate

**Stale-while-revalidate**는 백그라운드에서 캐시를 갱신하는 동안 낡은 캐시 응답을 계속 제공할 수 있게 하는 패턴(및 HTTP 지시어)입니다. 이는 약간의 데이터 지연을 허용하는 대신 최종 사용자의 캐시 미스 지연 시간을 사실상 제거합니다.

| Invalidation Strategy(무효화 전략) | Staleness(지연 시간) | Complexity(복잡도) | Infrastructure(필요 인프라) | Best For(용도) |
| --- | --- | --- | --- | --- |
| TTL expiry | 최대 TTL 초 | 없음 | 추가 필요 없음 | 허용 가능한 지연, 단순한 시스템 |
| Event-driven (pub/sub) | 초 단위 (비동기) | 중간 | 메시지 브로커 (Kafka/SQS) | 신속히 반영되어야 하는 사용자 수정 사항 |
| Versioned keys | 제로 (버전 변경 시) | 낮음 | 버전 카운터 저장소 | 모든 쓰기 경로를 통제할 수 있는 콘텐츠 |
| Cache tags (CDN) | 거의 제로 (API 호출 시) | 낮음 | 태그 지원 CDN | 제품/콘텐츠 페이지의 CDN 엣지 캐싱 |
| Stale-while-revalidate | 재검증 간격까지 | 낮음 | 추가 필요 없음 | 미스 지연을 허용할 수 없는 높은 읽기 QPS |

> 💡
> Interview Tip (인터뷰 팁)
> 캐시 무효화 질문은 시니어 후보자를 판별하는 리트머스 시험지입니다. 주니어의 답변: "TTL을 설정합니다." 시니어의 답변: "TTL이 기본이지만, 비즈니스의 지연 허용치에 따라 설정하겠습니다. 프로필 업데이트나 재고 변경 같이 사용자에게 즉시 보여야 하는 작업에는 Kafka를 통한 이벤트 기반 무효화를 추가하여 변경 사항이 분 단위가 아닌 초 단위로 전파되게 하겠습니다. 또한 핫 키(hot key)에 대해서는 Mutex 락을 사용하거나 Stale-while-revalidate를 적용하여 재생성 중에도 서비스를 지속함으로써 캐시 스탬피드를 방지하겠습니다. 백그라운드 갱신 동안의 짧은 지연이 허용되는지에 따라 방식을 선택하겠습니다." 이는 단순히 성공 케이스뿐만 아니라 실패 모드까지 알고 있음을 보여줍니다.
