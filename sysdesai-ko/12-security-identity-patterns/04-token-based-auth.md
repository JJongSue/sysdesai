# Token-Based Auth (JWT & OAuth2)

> 원문: https://www.sysdesai.com/learn/security-identity-patterns/token-based-auth

---

## 아키텍처 수준에서의 Token-Based Authentication

세션 기반 인증(session-based auth)에서는 서버가 세션 상태를 저장하고 클라이언트는 세션 ID 쿠키만 보유합니다. 반면 **토큰 기반 인증(token-based auth)**에서 서버는 상태를 유지하지 않는 **stateless** 구조입니다. 필요한 모든 상태는 클라이언트가 매 요청마다 제시하는 서명된 토큰 내에 포함됩니다. 이러한 트레이드오프는 분산 시스템을 확장하는 데 매우 중요합니다. 중앙 집중식 세션 저장소가 없으므로 어떤 서버든 독립적으로 요청을 검증할 수 있기 때문입니다.

## JWT 구조

**JSON Web Token (JWT)**은 `header.payload.signature` 형태의 점(.)으로 구분된 base64url-encoded 문자열입니다. Header는 알고리즘(`alg`)을 명시하고, payload는 claim(데이터)을 포함하며, signature는 토큰이 신뢰할 수 있는 당사자에 의해 발행되었고 변조되지 않았음을 증명합니다.

text

```
// JWT Header (base64url 디코딩 결과)
{
  "alg": "RS256",   // 알고리즘: RSA + SHA-256 (비대칭 암호화, 권장됨)
  "typ": "JWT",
  "kid": "key-2024-01"  // Key ID — JWKS에서 공개 키를 찾는 데 사용됨
}

// JWT Payload (base64url 디코딩 결과)
{
  "iss": "https://auth.example.com",   // 발행자 (Issuer)
  "sub": "user_abc123",                // 대상 (Subject, 사용자 ID)
  "aud": "https://api.example.com",    // 수신자 (Audience, 이 토큰을 수락할 대상)
  "iat": 1700000000,                   // 발행 시간 (Issued at, Unix timestamp)
  "exp": 1700003600,                   // 만료 시간 (Expires at, iat + 1시간)
  "roles": ["read:orders", "write:orders"],
  "tenant_id": "acme-corp"
}

// Signature (서버 측 검증 의사코드)
signature = RS256(
  base64url(header) + "." + base64url(payload),
  privateKey  // 인증 서버만 보유함
)

// 전체 JWT (클라이언트에 전송되며 매 요청마다 제시됨)
eyJhbGci...  .  eyJpc3Mi...  .  signature_bytes_base64url
```

> ⚠️
> JWT 함정: 'alg: none' 공격
> 잘 알려진 JWT 취약점 중 하나는 공격자가 헤더에 `"alg": "none"`을 설정하고 서명을 완전히 제거하는 것입니다. 취약한 라이브러리는 알고리즘 허용 목록(allowlisting)을 강제하지 않을 경우 이를 유효한 것으로 수락합니다. 항상 JWT 검증 라이브러리에서 **허용된 알고리즘을 명시적으로 지정**하고 `alg: none`을 절대 수락하지 마세요.

## OAuth2 Grant Types — 아키텍처 관점

| Grant Type | 사용 사례 | 반환되는 토큰 | 비고 |
| --- | --- | --- | --- |
| Authorization Code + PKCE | 웹 앱, 모바일, SPA 사용자 로그인 | Access + ID + Refresh tokens | 모든 사용자 대상 플로우에 권장됨 |
| Client Credentials | Machine-to-machine (M2M), 서비스 계정 | Access token 전용 | 사용자가 개입하지 않음; client_secret으로 인증 |
| Refresh Token | 단기 access token의 자동 갱신 | 신규 access token (+ 신규 refresh token) | 매 사용 시마다 refresh token 회전(rotate) |
| Device Code | 입력이 제한된 장치 (TV, CLI) | Access + Refresh tokens | 폴링 기반; 사용자가 보조 장치에서 승인 |
| Implicit (deprecated) | 레거시 SPA | Access token 직접 반환 | 사용 금지 — URL 파편(fragment)에 토큰이 노출됨 |

## Access Token + Refresh Token Flow
수명이 짧은 access token + 회전되는 refresh token: 표준 OAuth2 토큰 수명 주기입니다.

## Token Storage: 토큰을 어디에 보관할 것인가

| 저장 위치 | XSS 리스크 | CSRF 리스크 | 권장 사항 |
| --- | --- | --- | --- |
| localStorage / sessionStorage | 높음 — JS가 읽을 수 있음 | 없음 | 민감한 토큰을 여기에 저장하지 마세요 |
| 메모리 (JS 변수) | 낮음 — 지속되지 않음 | 없음 | SPA의 access token에 적합; 새로고침 시 소멸됨 |
| HttpOnly, Secure 쿠키 | 없음 — JS가 읽을 수 없음 | 존재함 (SameSite로 완화) | Refresh token에 가장 적합; SameSite=Lax 또는 Strict 사용 |
| Backend-for-Frontend (BFF) | 없음 — 브라우저는 세션 쿠키만 보유 | BFF 계층에서 완화됨 | SPA 보안의 골드 스탠다드 |

## 서비스 간 인증 (Service-to-Service Auth, M2M)

마이크로서비스 아키텍처에서 서비스들은 사용자뿐만 아니라 **서로에 대해서도** 인증해야 합니다. OAuth2 **Client Credentials grant**가 표준 패턴입니다. 각 서비스는 `client_id`와 `client_secret`(또는 인증서)를 가지며 이를 인증 서버의 access token과 교환합니다. 호출하는 서비스는 이 토큰을 요청에 첨부하고, 받는 서비스는 사용자 토큰을 검증하는 것과 동일한 방식으로 이를 검증합니다.

typescript

```
// M2M Client Credentials flow — 마이크로서비스용 의사코드
async function getServiceToken(): Promise<string> {
  const response = await fetch("https://auth.example.com/oauth/token", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      grant_type: "client_credentials",
      client_id: process.env.SERVICE_CLIENT_ID,
      client_secret: process.env.SERVICE_CLIENT_SECRET,
      scope: "orders:read inventory:write",
    }),
  });
  const { access_token, expires_in } = await response.json();
  // (만료 시간 - 버퍼) 초 동안 토큰 캐싱
  cacheToken(access_token, expires_in - 60);
  return access_token;
}

// 서비스 토큰을 사용하여 하위 서비스 호출
async function getInventory(itemId: string) {
  const token = await getCachedOrFetchToken();
  return fetch(`https://inventory-service/items/${itemId}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
}
```

> 💡
> Token Introspection vs Offline Validation
> JWT 검증은 **오프라인**(캐시된 공개 키를 사용한 암호화 검증)으로 이루어지며 빠르고 확장성이 좋습니다. 토큰 인트로스펙션(introspection)은 인증 서버의 `/introspect` 엔드포인트를 호출하여 토큰이 여전히 활성 상태인지 확인합니다(취소된 토큰을 실시간으로 감지). 더 느리지만 높은 보안이 필요한 시나리오에서는 필수적입니다. 일반적인 하이브리드 방식은 다음과 같습니다. 매 요청마다 JWT를 오프라인으로 검증하되, 장기 실행 세션이거나 계정 탈취 보고가 있을 때 주기적으로 introspection을 호출합니다.

> 💡
> 인터뷰 팁
> 시스템 설계 면접에서 인증을 논할 때는 다음 세 가지 시나리오를 명확히 구분하여 답변하세요: (1) 브라우저-API 간 (OIDC + 메모리 내 access token 또는 BFF 패턴), (2) 모바일-API 간 (Authorization Code + PKCE), (3) 서비스 간 (캐시된 토큰을 사용하는 Client Credentials). Refresh token 회전과 'alg: none' 취약점을 언급하면 시니어 수준의 보안 인식을 보여줄 수 있습니다. 또한 유출된 토큰의 피해 범위(blast radius)를 제한하기 위해 access token의 수명을 짧게(15분) 유지해야 한다는 점과, refresh token 회전 및 취소 목록(revocation list)을 통해 탈취된 refresh token을 잡아낼 수 있다는 점도 덧붙이세요.
