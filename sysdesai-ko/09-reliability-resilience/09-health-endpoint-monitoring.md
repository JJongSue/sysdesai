# Health Endpoint Monitoring

> Source: https://www.sysdesai.com/learn/reliability-resilience/health-endpoint-monitoring

---

## Health Endpoint란 무엇인가?

**Health endpoint**는 서비스 인스턴스가 올바르게 동작하고 있는지를 보고하는 전용 HTTP 엔드포인트(일반적으로 `GET /health` 또는 `GET /healthz`)입니다. 로드 밸런서(load balancer), Kubernetes 및 모니터링 시스템은 이 엔드포인트를 폴링(polling)하여 해당 인스턴스로 트래픽을 라우팅할지, 재시작할지, 아니면 당직 엔지니어에게 알람을 보낼지 결정합니다.

잘 설계된 health endpoint는 단순히 프로세스가 실행 중인지를 확인하는 것을 넘어, 데이터베이스, 캐시, 메시지 브로커 및 기타 주요 종속성(dependency)과의 연결성을 검증함으로써 서비스의 실제 요청 처리 능력을 확인합니다.

## Liveness vs Readiness Probes

Kubernetes는 의미와 결과가 다른 두 가지 유형의 health probe를 구분합니다.

| Probe | 대답하는 질문 | 실패 시 조치 | 확인 사항 |
| --- | --- | --- | --- |
| Liveness | 이 컨테이너가 살아 있는가 (데드락에 걸리지 않았는가)? | 컨테이너를 종료하고 재시작 | 프로세스 응답성, 데드락 유무. 무거운 종속성 확인은 피해야 함 — DB 장애로 컨테이너를 재시작해서는 안 됨 |
| Readiness | 이 컨테이너가 트래픽을 처리할 준비가 되었는가? | 로드 밸런서 풀에서 제거 (재시작하지 않음) | 데이터베이스 연결 가능성, 마이그레이션 완료 여부, 웜업(warm-up) 완료, 종속성 상태 허용 범위 내 |
| Startup (선택사항) | 컨테이너가 초기화를 마쳤는가? | 시작이 너무 오래 걸리면 종료 | 일회성 확인; liveness가 느리게 시작되는 컨테이너를 죽이는 것을 방지 |

> ⚠️
> Liveness Probes에서 외부 종속성을 확인하지 마세요.
> Liveness probe가 데이터베이스를 확인하고 있는데 데이터베이스가 다운되면, Kubernetes는 모든 pod를 재시작할 것입니다. 이는 복구를 훨씬 어렵게 만드는 thundering-herd restart 폭풍을 유발합니다. Liveness probe는 오직 애플리케이션 프로세스 자체의 응답성(예: HTTP 연결을 수락하고 단순한 엔드포인트에 응답할 수 있는지)만 확인해야 합니다.

## Deep Health Check의 구조

**Deep health check**는 각 핵심 종속성의 상태를 개별적으로 반환하여 운영자가 어느 컴포넌트가 실패하고 있는지 신속하게 파악할 수 있게 합니다. 응답에는 전체 상태와 컴포넌트별 상태가 포함되어야 합니다.

json

```
// GET /health/ready 응답
{
  "status": "degraded",
  "version": "2.3.1",
  "uptime": 3642,
  "checks": {
    "database": {
      "status": "healthy",
      "latency_ms": 4
    },
    "redis": {
      "status": "healthy",
      "latency_ms": 1
    },
    "payment_api": {
      "status": "unhealthy",
      "error": "Connection refused",
      "latency_ms": null
    },
    "disk_space": {
      "status": "healthy",
      "free_gb": 45.2
    }
  }
}
```

결제 API에 연결할 수 없기 때문에 전체 상태는 `degraded`(완전한 `healthy`가 아님)입니다. 서비스가 여전히 읽기 요청은 처리할 수 있을지 모르므로, 자신을 완전히 unhealthy로 표시하는 대신 치명적이지 않은 상태를 반환합니다. 컴포넌트별 중요도를 정의하세요: 핵심 종속성(DB) 실패 = `unhealthy`; 선택적 종속성 실패 = `degraded`.

## Health Check 구현

typescript

```
import express from "express";
import { db } from "./db";
import { redis } from "./cache";

const app = express();

// Liveness: 프로세스가 살아 있는가?
app.get("/health/live", (req, res) => {
  res.json({ status: "ok" });
});

// Readiness: 이 인스턴스가 트래픽을 처리할 수 있는가?
app.get("/health/ready", async (req, res) => {
  const checks: Record<string, object> = {};
  let overall: "healthy" | "degraded" | "unhealthy" = "healthy";

  // 데이터베이스 확인
  try {
    const start = Date.now();
    await db.query("SELECT 1");
    checks.database = { status: "healthy", latency_ms: Date.now() - start };
  } catch (err) {
    checks.database = { status: "unhealthy", error: String(err) };
    overall = "unhealthy"; // 핵심 종속성
  }

  // Redis 확인
  try {
    const start = Date.now();
    await redis.ping();
    checks.redis = { status: "healthy", latency_ms: Date.now() - start };
  } catch (err) {
    checks.redis = { status: "degraded", error: String(err) };
    if (overall === "healthy") overall = "degraded"; // 비핵심
  }

  const statusCode = overall === "unhealthy" ? 503 : 200;
  res.status(statusCode).json({ status: overall, checks });
});
```

## 로드 밸런서에서의 Health Check

AWS ALB와 NLB는 설정된 health check 엔드포인트를 N초마다 폴링합니다. 인스턴스는 M번의 연속적인 확인을 통과하면 healthy로 간주되고, N번의 연속적인 확인을 실패하면 unhealthy로 간주됩니다(히스테리시스(hysteresis)를 통해 상태가 요동치는 것을 방지). Unhealthy 인스턴스는 대상 그룹(target group)에서 제거되어 트래픽이 전송되지 않습니다. Health check 경로, 포트, 프로토콜, 간격(기본 30초), 타임아웃 및 임계값은 모두 설정 가능합니다.

로드 밸런서 health check 사이클: 실패 시 자동 트래픽 제거 및 복구 시 재추가

## Health Check 표준

Microsoft Azure 패턴의 **Health Check API pattern**과 **IETF draft for `application/health+json`**(draft-inadarei-api-health-check) 모두 표준 응답 형식을 정의합니다. Spring Boot Actuator(`/actuator/health`)와 Node.js `@godaddy/terminus`는 Kubernetes probe와 즉시 연동되는 풍부한 health check 프레임워크를 제공합니다.

> 💡
> 인터뷰 팁
> Health 엔드포인트는 운영의 우수성(operational excellence)과 Kubernetes 배포를 논할 때 등장합니다. 강조해야 할 차이점: liveness(pod 재시작) vs readiness(LB 풀에서 제거), liveness probe에서 외부 종속성을 확인하면 안 되는 이유, 그리고 health check가 어떻게 무중단 배포(zero-downtime deployments)를 가능하게 하는지(readiness probe가 새로운 pod가 완전히 초기화될 때까지 트래픽을 차단함) 등입니다. 'thundering herd restart' 안티 패턴을 구체적인 실패 모드로 언급하세요.
