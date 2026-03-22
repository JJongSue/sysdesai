# API Gateway for Security

> 출처: https://www.sysdesai.com/learn/security-auth/api-gateway-security

---

## 보안 경계로서의 API Gateway

Microservices 아키텍처에서 HTTP 엔드포인트를 노출하는 모든 서비스는 잠재적인 공격 표면입니다. 각 서비스에서 개별적으로 Authentication, Rate limiting, TLS, Input validation을 관리하면 중복이 발생하고 일관성이 없어집니다. 결국 가장 약한 서비스가 공격자의 침입 경로가 됩니다. **API Gateway**는 이러한 횡단 관심사(cross-cutting concerns) 보안을 중앙 집중화하여 모든 외부 트래픽에 대한 단일 진입점 역할을 합니다. **AWS API Gateway**, **Kong**, **Nginx**, **Traefik**, **Apigee**, **Cloudflare API Shield** 등이 이 패턴을 구현합니다.

## Gateway의 핵심 보안 기능

### TLS Termination

API Gateway는 클라이언트로부터의 TLS 연결을 종료하고 트래픽을 복호화합니다. Backend 서비스는 평문 HTTP(또는 재암호화된 내부 TLS)를 수신합니다. 이를 통해 인증서 관리를 중앙 집중화할 수 있습니다. 모든 서비스 인스턴스가 아닌 Gateway에서만 인증서를 갱신하면 됩니다. **AWS Certificate Manager (ACM)**와 **Let's Encrypt**는 인증서 발급과 갱신을 자동화합니다. 내부 트래픽에는 양측이 모두 인증하는 mTLS (mutual TLS)를 사용할 수 있습니다.

### Gateway에서의 Authentication

Gateway는 요청을 전달하기 전에 자격 증명을 검증합니다. JWT 기반 Auth의 경우: `Authorization: Bearer ` 헤더를 추출하고, Authorization server의 Public key(JWKS 엔드포인트에서 가져옴)를 사용하여 Signature를 검증하며, `exp`, `iss`, `aud` Claim을 확인합니다. 유효하다면 사용자 ID와 Claim을 신뢰할 수 있는 헤더(예: `X-User-Id: 12345`, `X-User-Roles: admin`)로 변환하여 Backend 서비스에 전달합니다. Backend 서비스는 이 헤더가 Gateway에 의해서만 설정될 수 있으므로 이를 암묵적으로 신뢰합니다.

> ⚠️
> 내부 전용 헤더 보호
> 클라이언트가 `X-User-Id` 헤더를 직접 설정할 수 있다면 어떤 사용자로든 사칭할 수 있습니다. API Gateway는 Backend 서비스로 요청을 전달하기 전에 클라이언트가 보낸 헤더 중 내부 신뢰 헤더와 일치하는 것이 있다면 반드시 제거(strip)해야 합니다. 이는 권한 상승 취약점을 만드는 흔한 설정 오류 중 하나입니다.

### 요청 검증 및 Schema Enforcement

Gateway는 들어오는 요청을 **OpenAPI Specification**과 대조하여 검증할 수 있습니다. 필수 필드가 누락되었거나 데이터 타입이 틀린 경우, 또는 Payload가 크기 제한을 초과하는 요청을 거부합니다. 이는 많은 Injection 공격과 잘못된 형식의 데이터가 애플리케이션에 도달하기 전에 차단합니다. 또한 클라이언트 개발자가 신뢰할 수 있는 명확한 계약(contract)을 제공합니다. AWS API Gateway, Kong, Apigee 등은 모두 OpenAPI 기반의 요청 검증을 기본적으로 지원합니다.

## Web Application Firewall (WAF)

**WAF**는 Layer 7에서 동작하며 HTTP 요청 콘텐츠를 규칙 집합과 대조하여 공격을 탐지하고 차단합니다. 일반적으로 API Gateway 앞단에 배치되거나 Gateway와 통합되어 운영됩니다. **OWASP ModSecurity Core Rule Set (CRS)**은 OWASP Top 10을 아우르는 업계 표준 오픈 소스 규칙 집합입니다. **AWS WAF**, **Cloudflare WAF**, **Imperva**와 같은 관리형 WAF 서비스는 자동으로 업데이트되는 규칙 집합과 Geofencing 기능을 제공합니다.

| 공격 유형 | WAF 규칙 예시 | WAF가 없을 때의 위험 |
| --- | --- | --- |
| SQL Injection | `'; DROP TABLE`과 일치하는 요청 차단 | 데이터베이스 침해 |
| XSS | 쿼리 파라미터/바디 내의 `<script>` 태그 차단 | 세션 하이재킹 |
| Path Traversal | URL 내의 `../` 시퀀스 차단 | 파일 시스템 접근 |
| Rate-based DDoS | 분당 N회 요청을 초과하는 IP 차단 | 서비스 불능 |
| Geo-blocking | 금지된 국가로부터의 트래픽 차단 | 규정 준수 위반 |
| Bot detection | 인간과 유사한 헤더가 없는 요청 확인 | Credential stuffing |

## IP Allowlisting 및 Denylisting

**IP allowlisting**은 API 접근을 알려진 IP 범위로 제한합니다. 파트너 시스템이 고정 IP를 사용하는 B2B 통합이나 외부에서 절대 접근해서는 안 되는 관리자 API에 필수적입니다. **IP denylisting**은 위협 인텔리전스 피드(예: Tor 출구 노드와 관련된 IP 범위, 알려진 봇넷, 이전 공격자 등)를 통해 파악된 악성 IP를 차단합니다. 두 방식 모두 API Gateway나 WAF에서 쉽게 구현할 수 있지만, 유동 IP나 VPN을 사용하는 클라이언트가 차단될 수 있다는 운영상 부담이 있습니다.

> 💡
> 면접 팁
> 면접에서는 Gateway의 역할과 서비스의 역할을 구분해서 설명하세요. Gateway는 TLS termination, JWT validation, Rate limiting, Schema validation, WAF, IP filtering을 담당합니다. 서비스는 비즈니스 로직 기반의 Authorization('Alice가 이 특정 주문을 수정할 수 있는가?'), 도메인 특화 검증, 데이터 접근 제어를 담당합니다. 이러한 명확한 관심사 분리는 설계 역량을 보여주는 좋은 지표입니다.

## API Key 관리

공개 API의 경우 API Key는 흔한 Authentication 메커니즘입니다. Gateway는 모든 요청에서 키를 추출하여 빠른 저장소(Redis 또는 DynamoDB)와 대조해 검증합니다. 키 로테이션, 취소, 키별 Rate limiting, 사용량 분석 등이 중앙에서 관리됩니다. Best Practice: Access Log에 API Key를 절대 남기지 말고(Key ID 사용), Prefix 기반 키(예: `sk_live_xxx`)를 사용하여 코드 저장소의 Secret scanning이 가능하게 하며, 설정된 기간 후에는 키가 자동으로 만료되도록 하세요.