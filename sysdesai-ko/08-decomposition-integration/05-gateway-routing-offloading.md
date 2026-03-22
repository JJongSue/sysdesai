# Gateway Routing & Gateway Offloading

> 원문: https://www.sysdesai.com/learn/decomposition-integration/gateway-routing-offloading

---

## Gateway Routing

**Gateway Routing**은 게이트웨이를 요청 속성에 따라 올바른 백엔드로 요청을 보내는 지능형 reverse proxy로 취급합니다. 클라이언트가 각 서비스의 주소를 아는 대신, 단일 게이트웨이 엔드포인트가 모든 트래픽을 받아 설정된 규칙에 따라 라우팅합니다. 집계(aggregation)와 구별하기 위해 이를 **Gateway as a Router** 패턴이라고 부르기도 합니다.

## 라우팅 전략 (Routing Strategies)

| 전략 | 라우팅 결정 기준 | 예시 |
| --- | --- | --- |
| Path-based routing | URL 경로 접두사 또는 정확한 매칭 | `/api/users/*` → User Service, `/api/orders/*` → Order Service |
| Header-based routing | HTTP 요청 헤더 값 | `X-API-Version: v2` → 새로운 서비스 버전 |
| Host-based routing | HTTP `Host` 헤더 / 서브도메인 | `mobile.api.example.com` → Mobile BFF |
| Method-based routing | HTTP 메서드 (GET, POST, DELETE) | GET `/items` → read replica, POST → write primary |
| Query param routing | URL 쿼리 파라미터 값 | `?region=eu` → EU 클러스터 |
| Weighted routing | 퍼센트 기반 분할 | 10% → canary 서비스, 90% → stable 서비스 |

## Gateway Offloading

**Gateway Offloading**은 모든 microservice에 중복되어 구현될 횡단 관심사들을 게이트웨이로 옮기는 것을 의미합니다. 이를 통해 각 서비스 팀은 인증 미들웨어, TLS 설정, 압축, 또는 분산 추적(distributed tracing)을 개별적으로 구현할 필요가 없게 됩니다. 이는 인프라 수준에서 적용된 DRY (Don't Repeat Yourself) 원칙입니다.

- **SSL/TLS Termination** — 게이트웨이에서 HTTPS를 복호화합니다; 서비스들은 신뢰할 수 있는 내부 네트워크에서 HTTP를 통해 통신합니다.
- **Authentication & Authorization** — JWT 또는 API 키를 중앙에서 검증하고, 서비스들을 위해 `X-User-Id` 헤더를 주입합니다.
- **Rate Limiting** — edge에서 API 키별, IP별, 또는 엔드포인트별 할당량을 강제합니다.
- **Request/Response Compression** — 각 서비스가 직접 구현하지 않아도 Gzip 또는 Brotli로 응답을 압축합니다.
- **Distributed Tracing** — 서비스 간 상관관계 분석을 위해 `X-Trace-Id` 헤더를 주입합니다 (OpenTelemetry, Zipkin).
- **Caching** — 백엔드 부하를 줄이기 위해 적절한 TTL로 `GET` 응답을 캐싱합니다.
- **CORS Headers** — 모든 서비스 대신 중앙에서 `Access-Control-Allow-Origin` 헤더를 추가합니다.
- **IP Allow/Deny Lists** — 요청이 서비스에 도달하기 전에 IP 대역별로 트래픽을 차단하거나 허용합니다.

> 💡
> Zero-Trust vs Perimeter Security
> 인증을 게이트웨이로 오프로딩하는 것은 경계 보안(perimeter security) 모델입니다. 제로 트러스트(zero-trust) 아키텍처에서는 게이트웨이가 검사한 후라도 서비스 내부에서 토큰을 다시 검증합니다. 보안 수준이 높은 시스템의 경우 두 계층을 모두 사용하십시오: 게이트웨이는 토큰이 진짜인지 검증하고, 개별 서비스는 특정 리소스에 대한 권한을 검증합니다.

## Weighted Routing을 통한 Canary Deployments

Gateway Routing은 새로운 서비스 버전을 배포하고 점진적으로 트래픽의 일부를 전환하는 **canary releases**를 가능하게 합니다. 1%의 트래픽에서 error rates와 latency를 모니터링하고, 지표가 건강하다면 10%, 50%, 100%로 전환합니다. 코드 변경 없이 게이트웨이 설정만으로 분할을 제어합니다. 이는 **AWS CodeDeploy**, **Argo Rollouts**, **Istio**가 실제로 canary 배포를 구현하는 방식입니다.

```yaml
# Kong Gateway 예시: canary 배포를 위한 weighted routing
services:
  - name: order-service-stable
    url: http://orders-v1:8080
  - name: order-service-canary
    url: http://orders-v2:8080

plugins:
  - name: request-termination
    route: orders-canary
    config:
      status_code: 200

routes:
  - name: orders-stable
    service: order-service-stable
    paths: ["/api/orders"]
    # traffic-splitter 플러그인 사용: 90%를 stable로, 10%를 canary로 전송
```

> 💡
> 면접 팁 (Interview Tip)
> 면접관이 '서비스의 새 버전을 가동 중단 없이(zero downtime) 어떻게 배포하시겠습니까?'라고 묻는다면, 그 메커니즘으로 게이트웨이 기반의 weighted routing(canary deployments)을 언급하십시오. 그런 다음 트레이드오프를 이해하고 있음을 보여주십시오: 게이트웨이가 weighted routing을 지원해야 하며(모든 게이트웨이가 기본적으로 지원하는 것은 아님), canary 퍼센트를 높일 시점을 판단하기 위한 observability가 필요하다는 점을 덧붙이십시오. Istio, Envoy, AWS ALB weighted target groups와 같은 도구들이 이를 네이티브하게 지원합니다.
