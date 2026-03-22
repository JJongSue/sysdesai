# 역방향 프록시(Reverse Proxy)

> Source: https://www.sysdesai.com/learn/networking/reverse-proxy

---

## 순방향 프록시(Forward Proxy) vs 역방향 프록시(Reverse Proxy)

'프록시(Proxy)'라는 용어는 중의적으로 사용됩니다. **순방향 프록시**는 클라이언트와 인터넷 사이에 위치하며 클라이언트를 대신해 동작합니다(예: 외부 트래픽을 필터링하는 기업용 프록시나 VPN). 반면 **역방향 프록시**는 서버 앞에 위치하며 서버를 대신해 동작합니다. 클라이언트는 프록시와 통신하고, 프록시는 요청을 백엔드 서비스로 전달한 후 그 응답을 받아 클라이언트에게 반환합니다.

## 역방향 프록시 vs 부하 분산기(Load Balancer)

많은 도구(NGINX, HAProxy, AWS ALB 등)가 두 기능을 모두 수행하기 때문에 이 용어들은 자주 혼용됩니다. 개념적인 차이는 다음과 같습니다.

| 기능 | 역방향 프록시 | 부하 분산기 |
| --- | --- | --- |
| 주요 역할 | 중재자 — 서버를 대신해 요청 처리 | 분배자 — 여러 서버로 요청 분산 |
| SSL/TLS 종료 | 지원 — 핵심 기능 | 선택적 지원 (L7 LB) |
| 콘텐츠 기반 라우팅 | 지원 — 경로, 호스트, 헤더 기반 규칙 | 지원 (L7 LB 전용) |
| 트래픽 분산 | 지원 가능 | 핵심 기능 |
| 캐싱 | 자주 지원 (NGINX 프록시 캐시 등) | 거의 지원하지 않음 |
| 속도 제한(Rate limiting) | 지원 | 거의 지원하지 않음 |
| 요청/응답 변환 | 지원 (헤더, 압축, 인증 등) | 거의 지원하지 않음 |
| 예시 | NGINX, Envoy, Traefik, Caddy | AWS ALB/NLB, HAProxy, F5 |

> ℹ️
> 실무에서는 기능이 겹칩니다.
> NGINX는 기술적으로 부하 분산 기능도 갖춘 역방향 프록시입니다. AWS ALB는 기술적으로 역방향 프록시 작업(SSL 종료, 경로 기반 라우팅 등)도 수행하는 부하 분산기입니다. 개념적 구분은 중요하지만, 면접에서는 용어 자체보다 어떤 기능이 필요한지에 집중하세요.

## 역방향 프록시의 핵심 사용 사례

### SSL/TLS 종료(SSL/TLS Termination)

역방향 프록시에서 TLS를 종료하면 백엔드 서버는 평문 HTTP를 수신하게 됩니다. 이를 통해 암호화/복호화에 따른 CPU 비용을 줄이고 인증서 관리를 중앙 집중화할 수 있습니다. 프록시는 클라이언트와 HTTPS 핸드셰이크를 처리한 후, 신뢰할 수 있는 내부 네트워크를 통해 백엔드에 암호화되지 않은 HTTP(또는 재암호화된 HTTPS)로 요청을 전달합니다.

### 압축(Compression)

역방향 프록시는 응답을 클라이언트에 보내기 전에 압축(gzip, Brotli 등)하여 대역폭을 절약할 수 있습니다. 이는 텍스트 기반 API에서 특히 효과적이며, JSON을 압축하면 페이로드 크기를 60~90%까지 줄일 수 있습니다. 프록시는 클라이언트의 `Accept-Encoding` 헤더를 확인하고 그에 맞춰 압축을 수행합니다.

### 속도 제한(Rate Limiting)

역방향 프록시는 요청이 백엔드 서비스에 도달하기 전에 속도 제한을 적용하여 남용을 방지하고 공정한 리소스 사용을 보장할 수 있습니다.

```nginx
# NGINX 속도 제한: IP당 초당 10개 요청
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
  location /api/ {
    limit_req zone=api burst=20 nodelay;
    proxy_pass http://backend;
  }
}
```

### 요청 라우팅 및 경로 재작성(Request Routing and Path Rewriting)

역방향 프록시는 URL 경로, 호스트네임 또는 헤더를 기반으로 요청을 서로 다른 백엔드 서비스로 라우팅할 수 있습니다. 이를 통해 여러 서비스에 대해 단일 공개 진입점(Entry point)을 가질 수 있습니다.

```nginx
server {
  server_name api.example.com;

  # /users 경로를 사용자 서비스로 라우팅
  location /users/ {
    proxy_pass http://user-service:8080/;
  }

  # /orders 경로를 주문 서비스로 라우팅
  location /orders/ {
    proxy_pass http://order-service:8081/;
  }

  # 모든 응답에 보안 헤더 추가
  add_header X-Frame-Options DENY;
  add_header X-Content-Type-Options nosniff;
}
```

### 캐싱(Caching)

NGINX와 같은 역방향 프록시는 백엔드 응답을 로컬에 캐싱하여 데이터 센터 내에서 미니 CDN 역할을 할 수 있습니다. 이는 자주 변경되지 않는 멱등(Idempotent) API 응답에 유용하며, 전체 CDN 설정 없이도 백엔드 부하를 줄여줍니다.

- **인증/인가 게이트웨이**: 요청을 전달하기 전 프록시 계층에서 JWT나 API 키를 검증합니다.
- **요청 버퍼링(Request buffering)**: 느린 클라이언트의 업로드를 버퍼링한 후 백엔드에 전달하여, 백엔드가 느린 연결을 기다리며 유휴 상태로 남는 것을 방지합니다.
- **서킷 브레이킹(Circuit breaking)**: Envoy와 같은 프록시는 서킷 브레이커 패턴을 기본적으로 구현합니다.
- **가시성(Observability)**: 중앙 집중식 액세스 로그, 메트릭 수집 및 분산 추적 주입(추적 헤더 추가)을 수행합니다.

> 💡
> 면접 팁
> 마이크로서비스 아키텍처를 논의할 때 **API 게이트웨이(API Gateway)**는 인증, 인가, 요청 변환, API 버전 관리, 개발자 포털 통합 등 추가 기능이 포함된 정교한 역방향 프록시라는 점을 언급하세요. AWS API Gateway, Kong, Apigee가 대표적인 예입니다. 면접에서 "인증이나 속도 제한 같은 횡단 관심사(Cross-cutting concerns)를 처리하기 위해 서비스 앞에 API 게이트웨이를 두어, 각 서비스가 비즈니스 로직에만 집중할 수 있게 하겠습니다"라고 답변할 수 있습니다.
