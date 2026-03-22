# Federated Identity Pattern

> 원문: https://www.sysdesai.com/learn/security-identity-patterns/federated-identity

---

## Federated Identity란 무엇인가?

**Federated identity**는 시스템이 자체적으로 자격 증명(credentials)을 관리하는 대신 신뢰할 수 있는 외부 기관인 **Identity Provider (IdP)**에 **인증을 위임**하는 아키텍처 패턴입니다. 애플리케이션은 **Service Provider (SP)** 또는 **Relying Party (RP)**라고 불리며, 사용자가 누구인지 확인해 주는 IdP의 보증을 신뢰합니다. 이를 통해 애플리케이션은 비밀번호 저장, MFA 처리, 또는 비밀번호 정책 강제 등의 부담에서 벗어날 수 있습니다.

이 패턴은 **Single Sign-On (SSO)**의 근간이 됩니다. 사용자는 IdP에서 한 번만 인증하면 자격 증명을 다시 입력하지 않고도 여러 서비스 제공업체에 접근할 수 있습니다. 실제 사례로는 **Auth0**, **Okta**, **Azure Active Directory (Azure AD / Entra ID)**, **Google Identity**, **AWS Cognito** 등이 있습니다.

## Core Protocols

| 프로토콜 | 포맷 | 전형적인 사용 사례 | 토큰 타입 |
| --- | --- | --- | --- |
| SAML 2.0 | XML assertions | 엔터프라이즈 SSO, B2B 페더레이션 | XML Assertion |
| OpenID Connect (OIDC) | JSON/JWT | 소비자 및 클라우드 네이티브 앱 | ID Token (JWT) |
| OAuth 2.0 | JSON | 위임된 권한 부여 (ID 아님) | Access Token |
| WS-Federation | XML | 마이크로소프트 에코시스템, 레거시 엔터프라이즈 | XML Token |

> ℹ️
> OIDC vs OAuth 2.0
> OAuth 2.0은 '이 앱이 당신을 대신해 무엇을 할 수 있는가?'라는 질문에 답하는 **권한 부여(authorization)** 프레임워크입니다. OpenID Connect는 OAuth 2.0 위의 얇은 ID 계층으로, '당신은 누구인가?'라는 질문에 답하는 **인증(authentication)** 프로토콜입니다. 대부분의 최신 시스템에서는 로그인을 위해 OIDC를 사용하고, API 권한 부여를 위해 OAuth 2.0 access token을 사용합니다.

## The OIDC / SSO Flow
PKCE를 포함한 OIDC Authorization Code flow — 웹 및 모바일 앱에 권장되는 플로우입니다.

## Cross-Organizational Federation

두 조직이 접근 권한을 공유해야 할 때(예: 회사가 기업 고객의 직원들에게 로그인을 허용하는 경우), IdP 간에 **신뢰 관계**를 구축합니다. 이를 **identity federation** 또는 **B2B federation**이라고 합니다. 일반적인 흐름은 다음과 같습니다:

1. 조직 A(고객)는 자체 IdP(예: Azure AD)를 가지고 있습니다.
2. 조직 B(SaaS 벤더)는 자신의 IdP(예: Okta)가 조직 A의 IdP를 외부 ID 소스로 신뢰하도록 설정합니다.
3. 조직 A의 사용자가 조직 B의 앱에 접속하면, 인증을 위해 자신의 IdP로 리다이렉트됩니다.
4. 조직 A의 IdP는 조직 B의 IdP에 SAML assertion이나 OIDC 토큰을 발행하고, 조직 B의 IdP는 이를 로컬 사용자 계정에 매핑합니다.
5. 조직 B의 IdP가 애플리케이션에 토큰을 발행합니다.

## Architecture Diagram
조직 간 SAML/OIDC 페더레이션 — SaaS 앱은 Okta를 신뢰하고, Okta는 Azure AD를 신뢰합니다.

## Key Design Decisions

| 결정 사항 | 옵션 | 권장 사항 |
| --- | --- | --- |
| 프로토콜 선택 | SAML vs OIDC | 신규 구축 시 OIDC 선호; SAML을 요구하는 기업 고객을 위해 SAML 지원 |
| 세션 관리 | IdP 관리형 vs SP 관리형 세션 | SP가 짧은 세션을 관리; IdP는 backchannel 로그아웃을 통해 세션 취소 처리 |
| Just-in-time provisioning | 사전 프로비저닝 vs JIT | 첫 로그인 시 JIT provisioning을 통해 대기업 고객의 관리 오버헤드 감소 |
| Attribute mapping | 정적 vs 동적 Claim 매핑 | Claim 변환 규칙을 통해 IdP claim을 로컬 역할(role)로 동적 매핑 |
| 토큰 저장 | Cookie vs localStorage | HttpOnly, Secure, SameSite=Lax 쿠키 사용 — 민감한 토큰은 절대 localStorage에 저장하지 않음 |

> ⚠️
> 보안 경고: 모든 토큰을 검증하세요
> ID 토큰을 검증 없이 신뢰하지 마세요. 반드시 **서명**(IdP의 공개 JWKS 사용), **발행자**(`iss` claim), **대상**(`aud` claim), **만료**(`exp` claim)를 검증해야 합니다. 이 중 하나라도 누락하면 토큰 교체 공격(substitution attack)에 노출될 수 있습니다.

> 💡
> 인터뷰 팁
> 면접에서 SSO나 federated identity에 대해 질문을 받으면 OIDC Authorization Code flow를 단계별로 설명하세요. 면접관은 back-channel 토큰 교환 방식(왜 브라우저에 직접 토큰을 주지 않고 서버 측에서 코드를 교환하는지)과 인증(OIDC) 및 권한 부여(OAuth 2.0 scope)의 차이를 이해하고 있는지 확인하고 싶어 합니다. 공개 클라이언트(public clients)를 위한 PKCE도 언급하세요.
