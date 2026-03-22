# Throttling 패턴

> Source: https://www.sysdesai.com/learn/reliability-resilience/throttling

---

## Throttling과 Rate Limiting

**Throttling**은 소비자가 서비스에 요청을 보내는 속도를 제어합니다. 이를 통해 서비스를 과부하로부터 보호하고, 클라이언트 간의 공정한 리소스 공유를 보장하며, 비즈니스 정책(예: API 티어 제한)을 강제합니다. **Rate limiting**은 throttling을 구현하는 데 사용되는 메커니즘입니다. 클라이언트가 제한을 초과하면 서버는 `429 Too Many Requests` 응답과 선택적으로 `Retry-After` 헤더를 반환합니다.

## 알고리즘 1: Fixed Window Counter

가장 단순한 알고리즘입니다. 고정된 시간 윈도우(예: 분당 100개 요청) 내에서 요청을 카운트합니다. 각 윈도우가 시작될 때 카운터를 초기화합니다. 약점은 **경계 버스트 문제(boundary burst problem)**입니다. 클라이언트가 12:00:59에 100개 요청을 보내고 12:01:00에 100개를 더 보내면, 약 2초 만에 200개의 요청이 발생하여 의도한 속도의 두 배가 될 수 있습니다.

## 알고리즘 2: Sliding Window Log

각 요청의 타임스탬프 로그를 저장합니다. 새로운 요청이 올 때마다 윈도우보다 오래된 항목을 제거하고 남은 개수를 셉니다. 어떤 롤링 윈도우에서도 정확히 N개의 요청만 허용합니다. 정확하지만 메모리 집약적입니다. 모든 요청의 타임스탬프를 저장하므로 트래픽이 많은 API에서는 확장성이 떨어집니다.

## 알고리즘 3: Sliding Window Counter

실용적인 하이브리드 방식입니다. 현재 윈도우의 카운트와 이전 윈도우의 카운트를 현재 윈도우가 경과한 정도에 따라 가중치를 두어 결합합니다. 예: `allowed = prev_count × (1 - elapsed/window) + curr_count`. 매끄럽고 메모리 효율적이며 Cloudflare의 rate limiter와 같은 프로덕션 시스템에서 사용됩니다.

## 알고리즘 4: Token Bucket

버킷은 최대 N개의 토큰을 담습니다. 토큰은 일정한 속도(예: 초당 10개)로 추가됩니다. 각 요청은 토큰 하나를 소비합니다. 버킷이 비어 있으면 요청은 거부됩니다. 이 버킷은 평균 속도를 유지하면서도 버킷 용량까지 **버스트(bursting)**를 허용합니다. AWS API Gateway, NGINX 및 대부분의 클라우드 throttling 시스템에서 사용됩니다.

Token bucket: 요청은 토큰을 소비함; 토큰은 일정한 속도로 리필됨; 버킷 용량까지 버스트 허용

## 알고리즘 5: Leaky Bucket

요청은 큐('버킷')에 들어가고 고정된 속도로 처리됩니다('새어 나감'). 큐가 가득 차면 새로운 요청은 버려집니다. 이는 **버스트를 매끄럽게(smooths bursts)** 하여 일정한 출력 속도로 만듭니다. 하위 서비스가 버스트를 처리할 수 없을 때 유용합니다. Token bucket과의 주요 차이점은 leaky bucket은 엄격한 출력 속도를 강제하는 반면, token bucket은 버스트를 허용하고 시간이 지남에 따라 매끄럽게 만듭니다.

| 알고리즘 | 버스트 처리 | 메모리 | 정확도 | 유스케이스 |
| --- | --- | --- | --- | --- |
| Fixed Window | 경계에서 2배 버스트 허용 | O(1) | 낮음 | 단순 API, 높은 정밀도 불필요 |
| Sliding Window Log | 버스트 불허 (정확함) | 클라이언트당 O(N) | 높음 | 저트래픽, 고정밀 필요 |
| Sliding Window Counter | 부드러운 근사치 | O(1) | 좋음 | 프로덕션 rate limiters (Cloudflare) |
| Token Bucket | 용량까지 버스트 허용 | O(1) | 좋음 | AWS, NGINX, 대부분의 클라우드 시스템 |
| Leaky Bucket | 버스트 불허 (평활화된 출력) | O(queue size) | 높은 출력 | 하위 서비스로의 트래픽 평활화 |

## 분산 Rate Limiting (Distributed Rate Limiting)

멀티 노드 배포 환경에서 각 노드가 자체 카운터를 유지하면 클라이언트가 10개 노드 각각에 N개씩 요청을 보내 총 10×N개를 보낼 수 있습니다. 분산 rate limiting은 **공유 저장소**를 필요로 합니다. 원자적(atomic) Lua 스크립트(또는 `INCR` + `EXPIRE`)를 사용하는 Redis가 가장 일반적인 접근 방식입니다. Redis의 단일 스레드 모델은 원자성을 보장합니다. 또는 서비스 메쉬 사이드카(Envoy)나 전용 rate-limiting 서비스(예: Lyft의 Ratelimit, Kong)를 사용하세요.

lua

```
-- sliding-window-counter rate limiting을 위한 Redis Lua 스크립트
-- EVAL을 통해 원자적으로 호출됨

local key = KEYS[1]           -- 예: "ratelimit:user:123"
local now = tonumber(ARGV[1]) -- 현재 타임스탬프 (ms)
local window = tonumber(ARGV[2]) -- 윈도우 크기 (ms)
local limit = tonumber(ARGV[3])  -- 최대 요청 수

-- 윈도우보다 오래된 항목 제거
redis.call("ZREMRANGEBYSCORE", key, 0, now - window)

-- 남은 항목 카운트
local count = redis.call("ZCARD", key)

if count < limit then
  -- 현재 요청 추가
  redis.call("ZADD", key, now, now)
  redis.call("PEXPIRE", key, window)
  return 1  -- 허용됨
else
  return 0  -- 거부됨
end
```

## Client-Side vs Server-Side Throttling

**Server-side throttling**은 API gateway나 서비스에서 제한을 강제하며 429로 응답합니다. 서버를 보호하지만 낭비되는 네트워크 호출을 막지는 못합니다. **Client-side throttling**은 알려진 제한 내에 머물기 위해 나가는 요청 속도를 선제적으로 제한하여, 거부될 요청을 아예 보내지 않음으로써 효율성을 높입니다. Google의 클라이언트 라이브러리들은 관찰된 에러율을 바탕으로 자체적으로 throttling을 수행하고 429 응답의 `Retry-After` 헤더를 준수하는 등 두 가지 방식을 모두 구현합니다.

> 💡
> 인터뷰 팁
> 'Rate Limiter 설계' 질문에 대해: 요구사항(사용자별 vs 글로벌, 엔드포인트별, 하드 vs 소프트 제한)부터 시작하여, 알고리즘을 선택하고(유연성을 위해 대개 token bucket이 정답임), 분산 처리 방식(Redis 원자적 스크립트)을 다룬 후, 엣지 케이스(분산 시스템에서의 clock skew, 클라이언트 측의 Retry-After 처리, 응답의 X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset과 같은 rate limit 헤더)를 논의하세요.

Practice this pattern
[Design a rate limiter for an API gateway](https://www.sysdesai.com/design/new?prompt=Design%20a%20rate%20limiter%20for%20an%20API%20gateway&mode=fast)
