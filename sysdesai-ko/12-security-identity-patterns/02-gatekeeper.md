# Gatekeeper Pattern

> 원문: https://www.sysdesai.com/learn/security-identity-patterns/gatekeeper

---

## Gatekeeper 패턴이란?

**Gatekeeper 패턴**은 클라이언트와 backend 서비스 사이에 전용 보안 호스트를 배치하는 방식입니다. 이 호스트의 유일한 책임은 들어오는 모든 요청을 실제 애플리케이션 서버로 전달하기 전에 **검증(validate), 인증(authenticate), 및 정리(sanitize)**하는 것입니다. Backend 서비스는 보호된 네트워크 구역에서 운영되며 외부 세계에서 직접 접근할 수 없습니다.

이는 **방어 정책(defense-in-depth)**의 일환입니다. 설령 공격자가 애플리케이션 로직에서 제로데이(zero-day) 취약점을 찾아내더라도, 최소한의 코드만 실행하고 공격 표면(attack surface)이 매우 작으며 잘못된 요청을 거부하도록 설정된 gatekeeper를 먼저 뚫어야 합니다. 이는 클럽의 보안 요원과 같습니다. 아무리 옷을 잘 차려입었어도 신분증(ID) 없이는 누구도 통과할 수 없습니다.

## Architecture Overview
Gatekeeper는 DMZ에 위치하며, backend 서비스는 public IP가 없고 직접 접근이 불가능합니다.

## Gatekeeper의 역할

- **Authentication & authorization**: 토큰(JWT, API keys, OAuth)을 검증하고, 인증되지 않은 요청이 backend 서비스에 도달하기 전에 거부합니다.
- **Input validation & sanitization**: 콘텐츠 타입(content-type), 페이로드 크기 제한, 스키마 준수 여부를 확인하고 위험한 문자를 제거하여 인젝션 공격을 방지합니다.
- **Rate limiting & throttling**: 클라이언트 또는 IP별 요청 제한을 적용하여 backend 리소스를 소모하기 전에 초과 요청을 차단합니다.
- **TLS termination**: HTTPS 처리를 담당하여 backend 서비스의 암호화 오버헤드를 줄여줍니다.
- **Request routing**: 검증된 요청을 경로(path), 호스트, 또는 헤더를 기반으로 적절한 backend 서비스로 전달합니다.
- **Audit logging**: 보안 포렌식(forensics)을 위해 거부된 요청을 포함한 모든 요청을 기록합니다.

## Gatekeeper vs API Gateway

Gatekeeper 패턴은 종종 API Gateway(예: AWS API Gateway, Kong, Nginx, Envoy)를 통해 구현됩니다. 차이점은 강조하는 지점에 있습니다. API gateway는 라우팅과 조합(composition)에 중점을 두는 반면, gatekeeper 패턴은 보안 검증을 최우선 과제로 삼습니다. 실제로는 잘 설정된 API gateway가 gatekeeper의 책임을 수행합니다.

| 항목 | API Gateway | Strict Gatekeeper |
| --- | --- | --- |
| 주요 초점 | 라우팅, 집계, 변환 | 보안 검증 및 거부 |
| 신뢰 모델 | 부분적으로 신뢰된 요청이 통과할 수 있음 | 제로 트러스트(Zero-trust): 명시적으로 허용되지 않은 모든 것을 거부 |
| 배포 | 공유 인프라 | 종종 전용 보안 호스트로 배포 |
| 일반적인 도구 | AWS API Gateway, Kong, Nginx | WAF + API Gateway, 커스텀 프록시(proxy) |

## Sequence: Request Validation Flow
Gatekeeper가 요청을 검증한 뒤, 거부하거나 인증 컨텍스트를 주입하여 전달합니다.

> ⚠️
> 외부로부터의 내부 호출 헤더를 절대 신뢰하지 마세요
> 흔히 저지르는 실수 중 하나는 클라이언트가 `X-User-Id`나 `X-Roles` 같은 내부 헤더를 직접 설정하도록 허용하는 것입니다. Gatekeeper는 클라이언트가 제공한 신뢰 헤더를 반드시 **제거(strip)**한 뒤, 자신이 검증한 값을 주입해야 합니다. 그렇지 않으면 공격자가 단순히 `X-User-Id: admin`을 보내 권한 부여를 완전히 우회할 수 있습니다.

## Trusted Subsystem 모델

요청이 Gatekeeper를 통과하면, backend 서비스는 **trusted subsystem 모델**로 작동합니다. 즉, Gatekeeper가 이미 인증과 권한 부여를 수행했음을 신뢰하고, 주입된 ID 헤더(`X-User-Id`, `X-Roles`, `X-Tenant-Id`)를 재검증 없이 직접 사용합니다. 이는 backend 서비스를 단순화하고 지연 시간을 줄여줍니다. 단, backend 서비스가 **오직 Gatekeeper를 통해서만 접근 가능**해야 하며 퍼블릭 인터넷에서는 절대 접근할 수 없어야 한다는 전제가 필요합니다.

> 💡
> 방어 정책 (Defense in Depth)
> Gatekeeper 패턴은 그 앞에 웹 애플리케이션 방화벽(WAF)을 두고, backend 서비스 간에는 mTLS를 강제하는 서비스 메시(예: Istio)와 결합할 때 효과가 극대화됩니다. 각 계층이 독립적인 보안 제어를 제공하므로, 하나의 계층이 뚫리더라도 전체 시스템의 권한이 노출되지 않습니다.

> 💡
> 인터뷰 팁
> 면접관이 API 보안 방법을 묻는다면, Gatekeeper 패턴을 언급하여 계층별 보안 사고를 보여주세요. 핵심 포인트: (1) Gatekeeper는 공격 표면이 최소화되어 있으며 보안이 강화된 코드만 실행한다. (2) Backend 서비스는 퍼블릭 네트워크에 노출되지 않는다. (3) Trusted subsystem 모델을 통해 backend에서 토큰 재검증을 생략할 수 있지만, 이는 오직 gatekeeper가 네트워크 경로를 보장하기 때문이라는 점을 명시하세요. 이것이 단순히 개별 Lambda에서 토큰을 검증하는 것보다 AWS API Gateway + IAM 정책을 사용하는 것이 더 안전한 이유와 같습니다.
