# Circuit Breaker (서킷 브레이커) 패턴

> 출처: https://www.sysdesai.com/learn/reliability-resilience/circuit-breaker

---

## 서비스에 Circuit Breaker (서킷 브레이커)가 필요한 이유

분산 시스템에서는 모든 네트워크 호출이 실패할 수 있습니다. 업스트림 서비스가 느려지거나 중단되면, 계속해서 재시도하는 호출자들은 자신의 스레드 풀을 소진하고, 커넥션 풀을 가득 채우며, 결국 하위 서비스로 장애를 전파(Cascade)시킬 수 있습니다. **Circuit Breaker (서킷 브레이커)**는 최근의 호출 결과를 추적하고 상태가 악화된 의존성 서비스로의 요청 전달을 자동으로 중단하는 프록시입니다. 이를 통해 서비스가 회복할 시간을 주고, 호출자를 리소스 고갈로부터 보호합니다.

이 이름은 전류가 너무 높게 치솟을 때 회로를 차단하여 손상을 방지하는 전기 회로 차단기(Circuit Breaker)에서 유래했습니다. 소프트웨어 버전은 에러율(또는 느린 호출 비율)이 임계값을 넘으면 작동하며, 회로를 'Open' 상태로 만들어 이후의 모든 호출을 즉시 실패(Fast-fail) 처리합니다.

## 세 가지 상태 (The Three States)
Circuit Breaker (서킷 브레이커) 상태 머신
| 상태 | 동작 | 다음 전환 |
| --- | --- | --- |
| Closed (닫힘) | 요청이 정상적으로 통과합니다. 실패 횟수는 Sliding Window (슬라이딩 윈도우) 내에서 계산됩니다. | → 에러율이 임계값을 초과하면 Open 상태로 전환 |
| Open (열림) | 모든 요청이 Fallback (폴백) 에러와 함께 즉시 실패합니다. 의존성 서비스에 호출이 도달하지 않습니다. | → 리셋 타임아웃(예: 30초) 후 Half-Open 상태로 전환 |
| Half-Open (반열림) | 회복 여부를 테스트하기 위해 단일 프로브(Probe) 요청을 허용합니다. | → 성공 시 Closed 상태로 전환, 실패 시 Open 상태로 전환 |

## 주요 설정 파라미터 (Key Configuration Parameters)

- **Failure threshold (실패 임계값)** — 브레이커를 작동시키기 위해 윈도우 내에서 실패해야 하는 호출의 비율 (예: 50%)
- **Minimum request volume (최소 요청량)** — 2번 호출해서 2번 모두 실패했다고 브레이커를 작동시키지 않습니다. 평가를 위해 최소 N번의 호출(예: 20번)이 필요합니다.
- **Sliding window size (슬라이딩 윈도우 크기)** — 횟수 기반(최근 N번의 호출) 또는 시간 기반(최근 N초)
- **Slow-call threshold (느린 호출 임계값)** — X ms보다 느린 호출을 실패로 간주합니다 (지연 시간 급증이 감지되지 않는 것을 방지)
- **Reset timeout (리셋 타임아웃)** — 프로브를 시도하기 전까지 Open 상태에서 대기하는 시간 (예: 30초, 반복적인 차단 시 지수적으로 증가 가능)
- **Half-open permitted calls (반열림 허용 호출 수)** — Closed 또는 Open 여부를 결정하기 전에 허용할 프로브 호출의 수

## 시퀀스: 정상, 차단, 그리고 회복 (Sequence: Normal, Tripped, and Recovery)
Closed → Open → Half-Open → Closed에 걸친 Circuit Breaker (서킷 브레이커) 생명주기

## Fallback (폴백) 전략

회로가 열렸을 때 단순히 에러를 던지기보다는 우아하게 기능이 저하(Degrade)되어야 합니다. 일반적인 Fallback (폴백) 방식으로는 **캐시된 오래된 데이터(Stale data) 반환**, **빈 값 또는 기본 응답 반환** (예: 빈 추천 목록), 나중에 처리하기 위한 **요청 큐잉(Queueing)**, 또는 **보조 서비스로 리다이렉션** 등이 있습니다. Netflix Hystrix는 각 커맨드가 구현하는 `getFallback()` 메서드를 통해 이 패턴을 대중화했습니다.

> ⚠️
> Fallback (폴백)은 동일한 의존성을 호출해서는 안 됩니다.
> 실패 중인 동일한 서비스를 호출하는 Fallback (폴백)은 목적에 어긋납니다. Fallback (폴백)은 순수하게 로컬에서 처리(캐시 데이터 반환, 기본값 반환)하거나 완전히 다른 서비스를 호출해야 합니다.

## 구현 예시 (의사 코드)

typescript
```
class CircuitBreaker {
  private state: "closed" | "open" | "half-open" = "closed";
  private failureCount = 0;
  private lastFailureTime = 0;

  constructor(
    private readonly threshold = 5,       // 5번 실패 후 차단
    private readonly resetTimeoutMs = 30_000 // 30초 리셋
  ) {}

  async call<T>(fn: () => Promise<T>, fallback: () => T): Promise<T> {
    if (this.state === "open") {
      if (Date.now() - this.lastFailureTime > this.resetTimeoutMs) {
        this.state = "half-open";
      } else {
        return fallback(); // 즉시 실패 (Fast-fail)
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      return fallback();
    }
  }

  private onSuccess() {
    this.failureCount = 0;
    this.state = "closed";
  }

  private onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    if (this.failureCount >= this.threshold) {
      this.state = "open";
    }
  }
}
```

## 실제 사례 (Real-World Examples)

**Netflix Hystrix**(현재 유지보수 모드)는 모든 서비스 간 호출을 감싸는 최초의 대중적인 구현체였습니다. **Resilience4j**는 JVM 생태계를 위한 현대적인 후속 모델로, 횟수 기반 및 시간 기반의 Sliding Window (슬라이딩 윈도우)를 제공합니다. AWS App Mesh와 Istio 서비스 메쉬는 Envoy 프록시를 통해 인프라 계층에서 Circuit Breaking (서킷 브레이킹)을 구현하므로 라이브러리 수준의 코드가 필요하지 않습니다. Spring Cloud Gateway와 AWS API Gateway도 Circuit Breaker (서킷 브레이커) 필터를 노출합니다.

> 💡
> 면접 팁
> 면접관들은 Circuit Breaker (서킷 브레이커)와 Retry (재시도)의 차이점을 설명하는 것을 좋아합니다. Retry (재시도)는 개별 호출에서 발생하는 일시적인(Transient) 에러를 위한 것이고, Circuit Breaker (서킷 브레이커)는 하위 서비스의 지속적인 상태 악화로부터 보호하기 위한 것입니다. 두 가지를 함께 사용하세요 — Closed 상태에서는 내부에서 Retry (재시도)를 수행하고, Open 상태에서는 즉시 실패(Fast-fail) 처리합니다. 항상 Fallback (폴백) 전략과 Observability (관측성)(상태 전환, Open 지속 시간, Fallback 호출 횟수 등에 대한 메트릭)을 언급하세요.

## Observability (관측성)

모든 상태 전환은 메트릭이나 로그 이벤트를 발생시켜야 합니다. 주요 메트릭: `circuit_breaker_state` (게이지: 0=closed, 1=open, 2=half-open), `circuit_breaker_calls_total` (결과별 레이블: success/failure/fallback/rejected), `circuit_breaker_open_duration_seconds`. `circuit_breaker_state == 1` 상태가 60초 이상 지속되는 경우에 대한 알람은 일시적인 현상이 아닌 실제 장애임을 나타내는 경우가 많습니다.
