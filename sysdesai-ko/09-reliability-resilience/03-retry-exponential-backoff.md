# Exponential Backoff을 이용한 Retry

> Source: https://www.sysdesai.com/learn/reliability-resilience/retry-exponential-backoff

---

## Retry가 필요한 이유

네트워크는 신뢰할 수 없습니다. 패킷이 유실되거나, 연결이 타임아웃되거나, 가비지 컬렉션(garbage collection) 일시 중지, 일시적인 과부하 또는 롤링 재시작으로 인해 서비스가 일시적인 지장(transient blips)을 겪을 수 있습니다. 단순한 **retry**(실패 후 동일한 요청을 다시 시도하는 것)를 통해 사용자가 문제를 전혀 눈치채지 못하게 이러한 대다수의 일시적인 에러로부터 투명하게 복구할 수 있습니다.

과제는 retry를 안전하게 수행하는 것입니다. 이미 고전 중인 서비스에 과부하를 주지 않아야 하고, 복구 불가능한 에러를 재시도하지 않아야 하며, 이미 성공했을 수도 있는 비멱등(non-idempotent) 작업을 재시도하지 않아야 합니다.

## 지수 백오프 (Exponential Backoff)

단순한 retry는 요청을 즉시 다시 보내는데, 이는 과부하된 서비스를 더 악화시킬 수 있습니다. **Exponential backoff**는 재시도 사이의 대기 시간을 기하급수적으로 늘립니다. 1초 대기 후, 2초, 4초, 8초 등으로 최대 상한선까지 늘립니다. 이를 통해 서비스가 복구될 시간을 벌어주고 저하된 기간 동안 전체 부하를 줄여줍니다.

typescript

```
async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries?: number;
    baseDelayMs?: number;
    maxDelayMs?: number;
    jitter?: boolean;
    retryOn?: (err: unknown) => boolean;
  } = {}
): Promise<T> {
  const {
    maxRetries = 3,
    baseDelayMs = 500,
    maxDelayMs = 30_000,
    jitter = true,
    retryOn = () => true,
  } = options;

  let attempt = 0;
  while (true) {
    try {
      return await fn();
    } catch (err) {
      if (attempt >= maxRetries || !retryOn(err)) throw err;

      // Exponential backoff: 500ms, 1000ms, 2000ms, ...
      let delay = Math.min(baseDelayMs * 2 ** attempt, maxDelayMs);

      // Jitter 추가: 지연 시간의 ±30%
      if (jitter) delay *= 0.7 + Math.random() * 0.6;

      await sleep(delay);
      attempt++;
    }
  }
}
```

## 지터 추가 (Adding Jitter)

지터가 없다면, 대략 같은 시간에 에러를 받은 모든 클라이언트들이 지수적으로 백오프된 동일한 순간에 재시도를 하게 됩니다. 이는 복구 중인 서비스에 동기화된 파동 형태로 다시 충격을 주는 **thundering herd** 문제를 야기합니다. **Jitter**는 백오프 지연 시간에 무작위 노이즈를 추가하여 재시도가 시간상으로 분산되도록 합니다. AWS는 **full jitter**(0과 상한선 사이의 랜덤) 또는 **decorrelated jitter**(지연 시간 = base와 prev_delay × 3 사이의 랜덤)를 권장합니다.

Jitter는 재시도를 시간상으로 분산시켜 thundering-herd의 재동기화를 방지합니다

## Retry를 하지 말아야 할 때

> ⚠️
> 복구할 수 없는 에러를 재시도하는 것은 상황을 악화시킵니다.
> 다음의 경우에는 절대 재시도하지 마세요: (1) 4xx 클라이언트 에러 (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found) — 요청 자체가 잘못되었거나 권한이 없으므로 재시도해도 얻는 것이 없습니다. (2) 비멱등(non-idempotent) 작업 (결제를 생성하는 POST 등), 단 idempotency keys가 사용되는 경우는 제외입니다. (3) circuit breaker가 열려 있을 때 — 브레이커는 알려진 장애 동안 재시도를 중단하기 위해 존재합니다.

| 에러 유형 | Retry 여부 | 근거 |
| --- | --- | --- |
| Network timeout / connection reset | Yes | 일시적인 네트워크 실패 |
| 503 Service Unavailable | Yes (백오프와 함께) | 서비스가 일시적으로 과부하됨 |
| 429 Too Many Requests | Yes (Retry-After 헤더 준수) | Rate limited 상태 — 대기 후 재시도 |
| 500 Internal Server Error | Yes (주의해서) | 일시적일 수 있음 |
| 400 Bad Request | No | 클라이언트 에러 — 요청을 수정해야 함 |
| 401 / 403 | No | 인증/권한 에러 — 재시도가 도움이 안 됨 |
| 404 Not Found | No | 리소스가 존재하지 않음 |

## Idempotency Keys

결제 청구와 같은 비멱등 작업을 안전하게 재시도하려면, 각 논리적 작업에 대해 클라이언트가 생성한 고유 ID인 **idempotency key**를 포함하세요. 서버는 이 ID를 키로 결과를 저장합니다. 재시도 시 서버는 작업을 다시 실행하는 대신 저장된 결과를 반환합니다. Stripe, Braintree 및 대부분의 결제 API는 `Idempotency-Key: uuid-v4`와 같은 요청 헤더를 통해 idempotency keys를 지원합니다.

## Retry Budgets

대규모 시스템에서는 원래 요청당 3번의 재시도만으로도 장애 시 부하가 4배로 늘어날 수 있습니다. Google의 SRE 서적은 **retry budgets**를 권장합니다. 클라이언트는 각 재시도가 토큰을 소비하는 토큰 버킷(token bucket)을 유지하며, 예산은 원래 요청 대비 재시도의 비율을 제한합니다(예: 재시도가 트래픽의 10%를 초과할 수 없음). 이는 재시도가 실패를 연쇄적인 과부하(cascading overload)로 증폭시키는 것을 방지합니다.

> 💡
> 인터뷰 팁
> 면접에서 retry를 논할 때는 항상 jitter를 언급하세요. 이는 thundering-herd 문제를 이해하고 있다는 신호입니다. Retry를 circuit breaker와 짝을 지으세요. Retry는 일시적인 에러를 처리하고, circuit breaker는 서비스가 시스템적으로 저하되었을 때 retry를 중단합니다. 회복성 고려 사항의 순서는 다음과 같아야 합니다: 일시적인 에러 retry → 지속적인 실패 circuit break → 폭발 반경 제한을 위한 bulkhead.
