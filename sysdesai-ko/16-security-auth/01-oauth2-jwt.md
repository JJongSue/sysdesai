# OAuth 2.0 & JWT

> 출처: https://www.sysdesai.com/learn/security-auth/oauth2-jwt

---

## Authentication이 대규모 환경에서 어려운 이유

Authentication(인증) — 사용자가 **누구인지** 확인하는 것 — 은 단일 세션 저장소를 가진 Monolith 환경에서는 간단해 보입니다. 하지만 Microservices, 모바일 클라이언트, 서드파티 통합, 그리고 수백만 명의 사용자가 생기면 문제는 매우 복잡해집니다. 이를 위해 **Stateless**(어느 서버에서나 중앙 저장소 호출 없이 토큰을 검증할 수 있어야 함), **Delegatable**(사용자가 서드파티 앱에 제한된 권한을 위임할 수 있어야 함), 그리고 **Revocable**(도난된 자격 증명을 빠르게 무효화할 수 있어야 함)한 프로토콜이 필요합니다. OAuth 2.0과 JWT는 이러한 요구사항을 함께 해결하며, 그렇기 때문에 사용자 데이터를 다루는 거의 모든 System Design 면접에서 등장합니다.

## OAuth 2.0: 프로토콜

**OAuth 2.0**은 Authentication 프로토콜이 아니라 Authorization(인가) 프레임워크입니다. 이는 **Resource Owner**(사용자)가 **Client**(사용자 앱)에게 credentials을 공유하지 않고도 **Resource Server**(예: Google Drive)에 저장된 리소스에 대한 제한된 접근 권한을 부여할 수 있게 해줍니다. **Authorization Server**(예: Auth0, Google Identity)는 토큰을 발급합니다. OpenID Connect (OIDC)는 OAuth 2.0 위에서 Authentication 기능을 추가합니다 — 이는 Access Token 위에 사용자에 대한 정보를 담은 `id_token`(JWT)을 도입합니다.

| Grant Type | 사용 대상 | 사용자 상호작용 | 권장 여부 |
| --- | --- | --- | --- |
| Authorization Code + PKCE | 웹 앱, SPAs, 모바일 | 있음 | 권장 — 현재 Best Practice |
| Client Credentials | Server-to-server (M2M) | 없음 | 권장 — 백엔드 서비스용 |
| Device Code | 스마트 TV, CLIs, IoT | 있음 (보조 기기) | 권장 — 리소스 제한 기기용 |
| Implicit | SPAs (레거시) | 있음 | 권장 안 함 — 폐기됨, URL에 토큰 노출 |
| Resource Owner Password | 신뢰할 수 있는 퍼스트파티 클라이언트 | 없음 (비밀번호 전송) | 권장 안 함 — OAuth의 목적을 저해함 |

### Authorization Code Flow with PKCE
OAuth 2.0 Authorization Code Flow with PKCE — 모든 사용자 대상 앱에 권장되는 Flow

## JWT: 구조 및 검증

**JSON Web Token (JWT)**은 클레임을 서명된 JSON 객체로 인코딩하는 콤팩트하고 URL-safe한 토큰 포맷입니다. 이는 점(.)으로 구분된 세 개의 Base64URL 인코딩된 세그먼트로 구성됩니다: `header.payload.signature`. Header는 서명 알고리즘(`alg`)을 지정합니다. Payload는 클레임을 포함합니다. Signature는 토큰이 신뢰할 수 있는 주체에 의해 발급되었으며 위변조되지 않았음을 증명합니다.

json

```
// Header (decoded)
{
  "alg": "RS256",   // SHA-256을 사용한 RSA 서명
  "typ": "JWT",
  "kid": "key-id-1" // 키 로테이션을 위한 Key ID
}

// Payload (decoded)
{
  "iss": "https://auth.example.com",  // 발급자 (Issuer)
  "sub": "user_12345",                // 주체 (Subject - 사용자 ID)
  "aud": "api.example.com",           // 대상 (Audience)
  "exp": 1740000000,                  // 만료 시간 (Unix timestamp)
  "iat": 1739996400,                  // 발급 시간
  "jti": "unique-token-id",           // JWT ID (블랙리스팅용)
  "roles": ["viewer", "commenter"],   // 커스텀 클레임
  "plan": "pro"
}

// Signature = RS256(base64url(header) + "." + base64url(payload), privateKey)
```

> ⚠️
> JWT는 암호화되는 것이 아니라 서명되는 것입니다
> 표준 JWT(JWS — JSON Web Signature)의 Payload는 암호화되는 것이 아니라 Base64URL로만 인코딩됩니다. 토큰을 획득한 사람은 누구나 클레임을 디코딩하고 읽을 수 있습니다. 비밀번호, PII(개인 식별 정보), 결제 정보와 같은 민감한 데이터를 JWT Payload에 절대 저장하지 마세요. 기밀 데이터가 필요한 경우 JWS 대신 JWE (JSON Web Encryption)를 사용하세요.

### 검증 체크리스트

1. Public Key(RS256의 경우) 또는 Shared Secret(HS256의 경우)을 사용하여 Signature를 확인합니다. 유효하지 않으면 거부합니다.
2. `alg` 헤더를 확인합니다 — `alg: none` 또는 예상치 못한 알고리즘은 거부합니다(Algorithm Confusion 공격).
3. `exp` 클레임을 확인합니다 — 토큰이 만료되었으면 거부합니다.
4. `iss`(발급자)가 예상한 Authorization Server와 일치하는지 확인합니다.
5. `aud`(대상)가 자신의 서비스 식별자를 포함하고 있는지 확인합니다.
6. 필요한 경우 `jti`를 토큰 블랙리스트와 대조하여 취소 여부를 확인합니다.

## Access Token vs Refresh Token

| 속성 | Access Token | Refresh Token |
| --- | --- | --- |
| 수명 | 짧음 (5–60분) | 긺 (며칠에서 몇 달) |
| 전송 대상 | Resource Server (APIs) | Authorization Server 전용 |
| 저장 위치 | 메모리 또는 HttpOnly 쿠키 | 보안 HttpOnly 쿠키 |
| 취소 가능 여부 | 쉽지 않음 (Stateless) | 가능 — Auth Server DB에 저장됨 |
| 용도 | API 호출에 대한 권한 증명 | 새로운 Access Token 획득 |
| 탈취 시 위험 | 공격자가 제한된 시간 동안 권한 가짐 | 공격자가 무기한으로 토큰 생성 가능 |

표준 패턴은 다음과 같습니다: API 접근을 위해 **짧은 수명의 Access Token**(15–60분)을 발급하고, HttpOnly 쿠키에 **긴 수명의 Refresh Token**(7–30일)을 저장합니다. Access Token이 만료되면 클라이언트는 백그라운드에서 Refresh Token을 새 토큰 쌍으로 교환합니다. 이는 Access Token이 도난당했을 때의 노출을 제한하면서도 사용자가 재인증 없이 로그인 상태를 유지할 수 있게 해줍니다.

## 토큰 저장 보안

| 저장 위치 | XSS 위험 | CSRF 위험 | 권장 사항 |
| --- | --- | --- | --- |
| localStorage / sessionStorage | 높음 — JS가 읽을 수 있음 | 낮음 | 민감한 토큰에는 피할 것 |
| 메모리 (JS 변수) | 중간 — 새로고침 시 소실 | 낮음 | 짧은 수명의 Access Token에 적합 |
| HttpOnly 쿠키 | 낮음 — JS가 읽을 수 없음 | 높음 — 자동으로 전송됨 | CSRF 토큰 또는 SameSite=Strict와 함께 사용 |
| HttpOnly + SameSite=Strict 쿠키 | 낮음 | 낮음 | Refresh Token을 위한 Best Practice |

> 💡
> 면접 팁
> 면접에서 인증에 대한 질문을 받으면 Threat Model(위협 모델)부터 언급하세요. '이것이 모바일 앱인지, SPA인지, 아니면 서버 사이드 렌더링 앱인지' 물어보세요. SPAs는 Authorization Code + PKCE를 사용하고, Refresh Token은 HttpOnly 쿠키에, Access Token은 메모리에 저장해야 합니다. 쿠키 저장 방식의 XSS vs CSRF 트레이드오프를 언급하면 두 가지 공격 벡터를 모두 이해하고 있음을 면접관에게 어필할 수 있습니다.

## 일반적인 함정 및 공격

- **JWT Algorithm Confusion**: 공격자가 `alg`를 `RS256`에서 `HS256`으로 바꾸고 (자신이 가진) Public Key로 서명하는 공격입니다. 서버 측에서 예상되는 알고리즘을 항상 고정하세요.
- **Redirect URI의 Open Redirect**: `redirect_uri`를 엄격하게 검증하지 않으면 OAuth가 공격자가 제어하는 URL로 리다이렉트될 수 있습니다. 항상 정확한 Redirect URI 화이트리스트를 관리하세요.
- **Authorization Endpoint의 CSRF**: `state` 파라미터가 없으면 공격자가 피해자를 대신해 OAuth Flow를 시작할 수 있습니다. CSRF를 방지하기 위해 항상 `state` 파라미터를 검증하세요.
- **Referer 헤더를 통한 토큰 유출**: URL 쿼리 파라미터에 토큰이 포함되면(Implicit Flow 등) `Referer` 헤더를 통해 유출될 수 있습니다. 절대 URL에 토큰을 넣지 마세요.
- **Refresh Token 탈취**: 유출된 Refresh Token은 무기한 접근 권한을 부여합니다. Refresh Token Rotation(사용할 때마다 새 Refresh Token 발급, 기존 토큰 무효화)을 사용하고 재사용을 감지하여 침해 신호로 활용하세요.

> 💡
> 프로덕션에서는 Identity Provider를 사용하세요
> OAuth 2.0을 처음부터 직접 구축하는 것은 오류가 발생하기 쉽고 위험합니다. **Auth0**, **Okta**, **AWS Cognito**, **Google Identity Platform**, 또는 **Keycloak**(셀프 호스팅)과 같은 관리형 Identity Provider를 사용하세요. 이들은 토큰 저장, Refresh Rotation, MFA, 소셜 로그인, 컴플라이언스(SOC2, GDPR)를 즉시 제공합니다. 면접에서 이를 언급하며 커스텀 솔루션이 대체해야 할 기능들을 설명하세요.
