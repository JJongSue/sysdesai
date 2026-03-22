# Rate Limiting & Throttling

> 출처: https://www.sysdesai.com/learn/security-auth/rate-limiting

---

## Rate Limiting이 중요한 이유

Rate limiting은 시스템을 **남용(abuse), 과부하(overload), 리소스 고갈(resource exhaustion)**로부터 보호하는 기본적인 방어 메커니즘입니다. 이것이 없다면 단 하나의 오작동하는 클라이언트(의도적이든 버그로 인해서든)가 모든 사용자의 서비스 품질을 저하시킬 수 있습니다. Rate limiting은 API 남용 방지, 공정 사용 정책(fair usage policies) 시행, Brute-force 공격 완화, 그리고 다운스트림 서비스가 압도당하지 않도록 보호하는 등 여러 목적을 수행합니다. 이는 시스템 디자인 인터뷰에서 가장 가치 있고 자주 등장하는 주제 중 하나이므로 알고리즘을 완벽히 숙지해야 합니다.

## Rate Limiting 알고리즘

### 1. Token Bucket

**Token Bucket**은 가장 널리 사용되는 알고리즘입니다(AWS API Gateway, Stripe 등에서 사용). 토큰은 정해진 Refill Rate에 따라 최대 용량(capacity)까지 버킷에 쌓입니다. 각 요청은 하나의 토큰을 소비합니다. 버킷에 토큰이 있으면 요청이 처리되고, 없으면 거부됩니다. 이 방식은 **제어된 버스트(controlled bursting)**를 허용합니다. 즉, 유휴 상태였던 클라이언트는 토큰을 축적했다가 버킷 용량만큼 한꺼번에 요청을 보낼 수 있습니다.

python

```python
import time

class TokenBucket:
    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity          # 최대 토큰 수 (버스트 크기)
        self.refill_rate = refill_rate    # 초당 추가되는 토큰 수
        self.tokens = capacity            # 처음에는 가득 찬 상태로 시작
        self.last_refill = time.time()

    def allow_request(self) -> bool:
        now = time.time()
        elapsed = now - self.last_refill

        # 경과 시간에 따라 토큰 리필
        self.tokens = min(
            self.capacity,
            self.tokens + elapsed * self.refill_rate
        )
        self.last_refill = now

        if self.tokens >= 1:
            self.tokens -= 1
            return True  # 요청 허용
        return False      # Rate limited (제한됨)

# 예시: 평상시 초당 100개 요청, 최대 200개까지 버스트 허용
limiter = TokenBucket(capacity=200, refill_rate=100)
```

### 2. Leaky Bucket

**Leaky Bucket**(큐 방식)은 입력 버스트와 상관없이 엄격하게 일정한 출력 속도를 유지합니다. 들어오는 요청은 고정된 크기의 큐에 들어갑니다. 프로세서는 일정한 속도로 큐에서 요청을 꺼내 처리합니다. 큐가 가득 차면 새로운 요청은 버려집니다. 이것은 **Traffic Shaper** 역할을 하며, 버스트를 처리할 수 없는 다운스트림 서비스(예: 결제 프로세서)를 보호하는 데 이상적입니다.

### 3. Fixed Window Counter

**Fixed Window**는 시간을 동일한 크기의 윈도우(예: 1분 간격)로 나눕니다. 각 요청마다 카운터가 증가합니다. 카운터가 한도를 초과하면 다음 윈도우가 시작될 때까지 요청이 거부됩니다. 매우 단순하지만(Redis의 `INCR`와 `EXPIRE` 하나로 구현 가능), **경계선 버스트 문제(boundary burst problem)**가 있습니다. 클라이언트가 한 윈도우의 끝과 다음 윈도우의 시작 부분에 요청을 집중하면 60초 동안 한도의 2배에 달하는 요청을 보낼 수 있습니다.

### 4. Sliding Window Log

**Sliding Window Log**는 모든 요청의 타임스탬프를 Sorted Set에 저장합니다. 요청을 확인할 때 `현재 시간 - window_size`보다 오래된 타임스탬프를 제거하고 남은 개수를 세어 한도 초과 시 거부합니다. 매우 정확하며 경계선 버스트 문제가 없지만, 요청마다 타임스탬프를 저장하므로 트래픽이 많은 API에서는 메모리 사용량이 많아집니다.

### 5. Sliding Window Counter

**Sliding Window Counter**는 가장 실용적인 절충안입니다. 두 개의 Fixed-window 카운터(현재와 이전 윈도우)를 사용하며, 비율을 계산하여 요청 수를 추정합니다: `count = 이전_윈도우_카운트 * (윈도우_남은_비율) + 현재_윈도우_카운트`. 이 근사치는 대부분의 트래픽 패턴에서 약 0.003% 이내의 정확도를 보이며 사용자당 두 개의 정수만 저장하면 됩니다.

| 알고리즘 | 메모리 | 정확도 | 버스트 허용 | 구현 복잡도 |
| --- | --- | --- | --- | --- |
| Token Bucket | 사용자당 O(1) | 높음 | 예 (용량까지) | 낮음 |
| Leaky Bucket | O(queue_size) | 높음 | 아니요 (트래픽 쉐이핑) | 중간 |
| Fixed Window Counter | 사용자당 O(1) | 중간 (경계선 버스트) | 예 (경계선에서 2배) | 매우 낮음 |
| Sliding Window Log | 사용자당 O(limit) | 정확함 | 아니요 | 중간 |
| Sliding Window Counter | 사용자당 O(1) | ~99.7% 정확 | 예 (제어된 방식) | 낮음 |

## Distributed Rate Limiting

API가 로드 밸런서 뒤의 여러 서버에서 서비스될 때, 각 서버가 로컬 카운터를 유지할 수 없습니다. 클라이언트가 서로 다른 서버로 라우팅되면 서버 수만큼 한도를 초과할 수 있기 때문입니다. 모든 서버가 업데이트하는 **공유된 원자적 카운터(shared, atomic counter)**가 필요합니다. **Redis**는 원자적 연산(`INCR`, Lua 스크립트)과 밀리초 미만의 지연 시간 덕분에 표준 솔루션으로 사용됩니다.

lua

```lua
-- 원자적 Sliding Window Counter Rate Limiting을 위한 Redis Lua 스크립트
-- 호출 방식: EVAL script 1 user:123 1000 60 current_time

local key = KEYS[1]             -- 예: "ratelimit:user:123"
local limit = tonumber(ARGV[1]) -- 예: 1000개 요청
local window = tonumber(ARGV[2]) -- 예: 60초
local now = tonumber(ARGV[3])   -- 현재 Unix 타임스탬프

local current_window = math.floor(now / window) * window
local prev_window = current_window - window

local curr_key = key .. ":" .. current_window
local prev_key = key .. ":" .. prev_window

local curr_count = tonumber(redis.call("GET", curr_key) or "0")
local prev_count = tonumber(redis.call("GET", prev_key) or "0")

-- 보간(Interpolation): 이전 윈도우에 현재 윈도우의 남은 시간 비율만큼 가중치 부여
local elapsed_in_window = now - current_window
local prev_weight = (window - elapsed_in_window) / window
local estimated_count = math.floor(prev_count * prev_weight) + curr_count

if estimated_count >= limit then
    return 0  -- Rate limited (제한됨)
end

-- 현재 윈도우 카운터 증가
redis.call("INCR", curr_key)
redis.call("EXPIRE", curr_key, window * 2)
return 1  -- 허용됨
```

## Rate Limiting 차원(Dimensions)

- **IP 주소 기준**: 인증되지 않은 엔드포인트를 보호합니다. 구현이 쉽지만, 단일 IP를 공유하는 NAT 뒤의 사용자들에게 불이익을 줄 수 있습니다.
- **사용자 ID 기준**: 인증된 API에 대해 더 정확합니다. JWT나 세션에서 사용자 ID를 추출해야 합니다.
- **API Key 기준**: 공개 API(예: Stripe, Twilio)의 표준입니다. 각 키마다 고유한 한도를 가질 수 있어 요금제별 차등 적용이 가능합니다.
- **엔드포인트 기준**: 일부 엔드포인트(예: SMS 전송)는 다른 엔드포인트(예: 읽기 작업)보다 더 엄격한 제한이 필요할 수 있습니다.
- **지리적 지역 기준**: 데이터 센터별로 별도의 Rate limit을 적용하여 한 지역의 트래픽이 글로벌 할당량을 모두 소모하는 것을 방지합니다.

> 💡
> 면접 팁
> Rate limiter 설계를 요청받으면 다음 순서로 설명하세요: (1) 어떤 알고리즘을 왜 선택했는지 (유연성을 위한 Token Bucket, 저메모리 고정밀을 위한 Sliding Window Counter), (2) 상태를 어디에 저장하는지 (원자적 Lua 스크립트를 사용하는 Redis), (3) Rate limit 키가 무엇인지 (사용자 ID, API 키, IP), (4) Redis 장애 시 어떻게 처리하는지 (Fail open vs Fail closed — 대부분은 정상 트래픽 차단을 막기 위해 Fail open을 선택), (5) 클라이언트에게 어떻게 정보를 노출하는지 (표준 `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After` 헤더).

연습해보기
[글로벌 API를 위한 분산 Rate Limiter 설계](https://www.sysdesai.com/design/new?prompt=Design%20a%20distributed%20rate%20limiter%20for%20a%20global%20API&mode=fast)