# Timeout 패턴

> Source: https://www.sysdesai.com/learn/reliability-resilience/timeout-pattern

---

## 타임아웃이 필수적인 이유

타임아웃이 없다면, 느린 응답을 기다리는 스레드는 **좀비(zombie)**가 됩니다. 이는 thread-pool 슬롯, 연결, 메모리를 영원히(또는 프로세스가 충돌할 때까지) 점유합니다. 실제로 몇십 개의 좀비 스레드만으로도 thread pool이 가득 차서 전체 서비스가 응답하지 않게 만들 수 있습니다. **모든** 네트워크 호출, 데이터베이스 쿼리, 프로세스 간 통신에는 명시적인 타임아웃이 있어야 합니다.

> ⚠️
> 기본 타임아웃은 대개 너무 깁니다.
> 많은 HTTP 클라이언트 라이브러리는 기본값을 60–120초로 설정하거나 타임아웃이 아예 없는 경우도 있습니다(예: Python의 `requests` 라이브러리). 항상 명시적인 타임아웃을 설정하세요: connection timeout(TCP 연결을 기다리는 시간) 및 read/socket timeout(연결 후 데이터를 기다리는 시간).

## 타임아웃의 종류 (Types of Timeouts)

| 타임아웃 유형 | 제한 대상 | 전형적인 값 |
| --- | --- | --- |
| Connection timeout | TCP 연결을 수립하는 데 걸리는 시간 | 1–5초 |
| Read / socket timeout | 연결 후 다음 바이트를 받는 데 걸리는 시간 | 작업에 따라 5–30초 |
| Request timeout | 전체 요청/응답 사이클의 종단 간(end-to-end) 시간 | SLO에서 유도됨 |
| Idle connection timeout | pool에 있는 연결이 미사용 상태로 유지될 수 있는 시간 | 30–90초 (서버측 유휴 타임아웃보다 짧아야 함) |

## 타임아웃 예산 및 데드라인 전파 (Timeout Budgets and Deadline Propagation)

마이크로서비스 호출 체인에서 각 홉(hop)은 전체 레이턴시 예산의 일부를 소비합니다. 서비스 A가 500ms 내에 응답해야 하고 서비스 B와 C를 순차적으로 호출한다면, 서비스 B의 타임아웃은 ~150ms, 서비스 C는 ~150ms로 설정하여 로컬 처리 및 오버헤드를 위해 ~200ms를 남겨두어야 합니다. 이를 **timeout budget** 또는 **deadline propagation**이라고 합니다.

Google의 gRPC는 `grpc-timeout` 헤더를 통해 데드라인을 자동으로 전파합니다. 각 서비스는 줄어든 데드라인을 하위 서비스에 전달합니다. 하위 서비스 호출이 시작될 때 이미 전체 데드라인이 만료되었다면, 성공 가능성이 없는 요청을 시작하는 대신 호출을 즉시 취소합니다.

Deadline propagation: 각 홉은 전체 예산의 일부를 할당받음

## 타임아웃 값 설정하기

추측이 아닌 **측정된 레이턴시 백분위수(measured latency percentiles)**를 기반으로 타임아웃을 설정하세요. 일반적인 가이드라인은 **p99 레이턴시 + 합리적인 버퍼**(예: 2배)로 설정하는 것입니다. p99 응답 시간이 200ms라면, 500ms 타임아웃은 가끔 발생하는 스파이크에 대비한 여유를 제공하면서도 실패 비용이 너무 크지 않게 해줍니다. 시스템이 진화함에 따라 검토하고 조정하세요. 건강한 서비스에 맞춰진 타임아웃은 저하된 서비스에는 너무 길 수 있습니다.

## 타임아웃과 Circuit Breaker의 결합

타임아웃과 circuit breaker는 상호 보완적입니다. 타임아웃은 **단일 느린 호출**을 처리합니다. 너무 오래 기다린 후 포기하고 에러를 반환합니다. 타임아웃이 빈번해지면 circuit breaker가 이 패턴을 감지하고 회로를 개방하여 이후의 호출들이 아예 기다리지 않도록 방지합니다. Circuit breaker의 slow-call threshold를 타임아웃 값과 일치하도록 설정하여 느린 호출이 브레이커의 통계에서 실패로 카운트되게 하세요.

> 💡
> 인터뷰 팁
> 면접관은 종종 '타임아웃을 설정하지 않으면 어떻게 되나요?'라고 묻습니다. 답은 좀비 스레드, connection pool 고갈, 그리고 연쇄 실패(cascading failures)입니다. 또한 타임아웃 이후에도 서버측 작업은 여전히 실행 중일 수 있음을 언급하세요. 취소 토큰(cancellation tokens)(gRPC deadline, HTTP/2 RST_STREAM)을 사용하여 서버에 버려진 요청의 처리를 중단하도록 신호를 보내야 합니다.

## 코드 예시: Node.js에서 타임아웃 설정하기

typescript

```
import axios from "axios";

const httpClient = axios.create({
  baseURL: "https://api.example.com",
  timeout: 3000, // 총 3초 요청 타임아웃
  // 참고: axios 타임아웃은 connection + read를 결합하여 커버합니다.
});

// 더 세밀한 제어를 위해 undici 또는 기본 fetch API를 사용하세요:
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 3000);

try {
  const res = await fetch("https://api.example.com/data", {
    signal: controller.signal,
  });
  const data = await res.json();
  return data;
} catch (err) {
  if (err instanceof DOMException && err.name === "AbortError") {
    throw new Error("3000ms 후 요청이 타임아웃되었습니다.");
  }
  throw err;
} finally {
  clearTimeout(timeoutId);
}
```
