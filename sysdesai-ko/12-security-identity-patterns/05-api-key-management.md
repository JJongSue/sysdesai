# API Key Management

> 원문: https://www.sysdesai.com/learn/security-identity-patterns/api-key-management

---

## API Key — 단순하지만 중요한 관리 요소

API key는 호출하는 애플리케이션이나 사용자를 식별하고 인증하는 긴 무작위 문자열(**opaque tokens**)입니다. OAuth2 access token과 달리 보통 **수명이 길고(long-lived)** 내부에 정보(claim)를 포함하지 않습니다. 서버는 데이터베이스에서 키를 조회하여 권한을 결정합니다. 이러한 단순함 덕분에 개발자용 API(Stripe, Twilio, OpenAI, GitHub 등)에서 인기가 많지만, 그만큼 보안 사고를 피하기 위한 세심한 관리가 필요합니다.

API key 관리는 단순히 키를 생성하는 것만이 아닙니다. **생성(creation), 범위 지정(scoping), 배포(distribution), 회전(rotation), 모니터링(monitoring), 및 취소(revocation)**를 아우르는 전체 수명 주기를 관리해야 합니다. 어느 한 단계라도 부실하게 관리되면 리스크가 발생합니다.

## API Key Architecture
API key 수명 주기: 포털에서의 생성, 해시(hash) 형태의 저장, 게이트웨이에서의 검증, 사용량 분석.

## 보안이 강화된 API Key 생성

보안상 안전한 API key는 **암호학적으로 무작위(cryptographically random)**여야 하며 **무차별 대입 공격(brute-force)에 견딜 수 있을 만큼 충분히 길어야** 합니다. 256비트(32바이트)의 엔트로피(entropy)를 16진수(hex)나 base62로 인코딩하는 것이 표준 기준입니다. 키 포맷의 모범 사례는 다음과 같습니다:

typescript

```
import crypto from "crypto";

// 암호학적으로 안전한 API key 생성
function generateApiKey(prefix: string = "sk_live"): string {
  // 32바이트 = 256비트의 엔트로피
  const rawBytes = crypto.randomBytes(32);
  // URL-safe base64로 인코딩 (16진수보다 짧음)
  const keyBody = rawBytes.toString("base64url");
  // 포맷: prefix_version_body  예: sk_live_v1_abc123...
  return `${prefix}_v1_${keyBody}`;
}

// 원본 키를 절대 저장하지 마세요 — 해시(hash)만 저장합니다.
async function storeApiKey(rawKey: string, userId: string, scopes: string[]) {
  // 빠른 조회를 위해 SHA-256 사용; 매 요청마다 해싱하기에 bcrypt는 너무 느림
  const keyHash = crypto.createHash("sha256").update(rawKey).digest("hex");

  await db.apiKeys.create({
    keyHash,          // 저장 및 조회 시 비교 대상
    keyPrefix: rawKey.slice(0, 12) + "...",  // 화면 표시용 (전체 키 노출 금지)
    userId,
    scopes,
    createdAt: new Date(),
    lastUsedAt: null,
    revokedAt: null,
    expiresAt: null,  // 또는 명시적 만료일 설정
  });

  // 원본 키를 사용자에게 단 한 번만 반환 — 나중에 복구할 수 없음
  return rawKey;
}

// 들어오는 API key 검증
async function validateApiKey(incomingKey: string) {
  const incomingHash = crypto.createHash("sha256").update(incomingKey).digest("hex");
  const keyRecord = await db.apiKeys.findByHash(incomingHash);

  if (!keyRecord || keyRecord.revokedAt || keyRecord.expiresAt < new Date()) {
    throw new Error("유효하지 않거나 만료된 API key입니다.");
  }

  await db.apiKeys.updateLastUsed(keyRecord.id); // 비동기, non-blocking
  return keyRecord;
}
```

> ⚠️
> API Key를 Plaintext로 저장하지 마세요
> 데이터베이스에는 API key의 **SHA-256 해시(hash)**만 저장하세요. 원본 키는 생성 시 사용자에게 딱 한 번만 보여주며 이후에는 복구할 수 없게 해야 합니다. 이는 데이터베이스가 유출되었을 때의 피해 범위(**blast radius**)를 제한합니다. 공격자는 사용할 수 있는 키 대신 해시값만 얻게 되기 때문입니다. API key 해싱에는 bcrypt 대신 SHA-256을 사용하세요. bcrypt의 느린 해싱 속도는 매 요청 검증 시 허용할 수 없는 수준의 지연 시간(latency)을 추가하기 때문입니다.

## Key Scoping and Hierarchies

**최소 권한 원칙(principle of least privilege)**을 따르세요. 모든 API key는 그 목적에 필요한 권한만 부여받아야 합니다. 잘 설계된 키 시스템은 **계층적 범위(hierarchical scopes)**를 지원합니다:

| 키 유형 | 범위 예시 | 사용 사례 |
| --- | --- | --- |
| Master / Root Key | 모든 권한 | 코드에서 절대 사용 금지; 하위 키 생성용으로만 사용. vault에 보관. |
| Environment Key | sk_live_... / sk_test_... | 운영 환경과 스테이징/테스트 환경용 키를 분리 |
| Scoped Service Key | read:orders, write:inventory | 최소한의 API 엔드포인트로 범위가 제한된 서비스별 키 |
| Per-User Key | user_id:123, read:own_data | 사용자가 자신의 데이터에만 접근하기 위해 생성한 키 |
| Webhook Signing Key | sign:webhooks | API 접근용이 아닌, 나가는 웹훅 페이로드를 HMAC-sign하기 위한 키 |

## API Key별 Rate Limiting

Rate limit은 단순히 IP 레벨이 아닌 **API key 레벨**에서 강제되어야 합니다. 단일 키가 여러 IP(모바일 클라이언트, 분산 서비스 등)에서 사용될 수 있고, 반대로 단일 IP가 합법적으로 여러 개의 서로 다른 키를 호스팅할 수도 있기 때문입니다. 일반적인 전략은 다음과 같습니다:

- **Redis의 sliding window counter**: `INCR key:api_key_hash:minute_bucket`, `EXPIRE`를 60초로 설정. 빠르고 정확합니다.
- **키별 Token bucket**: 일시적인 대량 트래픽(`burst_size`)은 허용하되 지속적인 속도는 `rate_per_second`로 제한합니다.
- **플랜별 Tiered limits**: 무료 티어(100 req/min), 프로 티어(1000 req/min), 엔터프라이즈 티어(커스텀) 등.
- **엔드포인트별 제한**: `POST /generate`(10 req/min)와 `GET /status`(1000 req/min) 분리 — 연산 집약적인 엔드포인트는 더 엄격한 제한이 필요합니다.

## Key Rotation and Revocation
가동 중단 없는(zero-downtime) 키 회전: 새 키 생성, 배포 후 기존 키 취소.

## 즉각적인 식별을 위한 Key Prefix

`sk_live_v1_...`이나 `ghp_...`(GitHub Personal Access Tokens)와 같은 키 포맷은 미관상의 이유를 넘어선 보안상의 장점이 있습니다. **비밀 스캐너(secret scanners)**(GitHub, GitLab, Trufflehog, Gitleaks 등)가 이러한 접두사를 인식하여 유출된 키가 포함된 커밋을 자동으로 알리거나 차단할 수 있기 때문입니다. 자동 스캔 도구가 쉽게 인식할 수 있도록 키 포맷을 설계하세요.

📌
실제 사례
Stripe: `sk_live_...` / `pk_live_...` | GitHub: `ghp_...` (개인용) / `ghs_...` (서비스용) | OpenAI: `sk-...` | Twilio: `ACxxxxxxx` (Account SID) + 별도의 인증 토큰 | Anthropic: `sk-ant-...`. 각 접두사는 GitHub의 비밀 스캐닝 파트너 프로그램에 등록되어 유출 시 자동 알림을 트리거합니다.

## 클라이언트를 위한 안전한 키 저장소

| 환경 | 권장 저장소 | 피해야 할 곳 |
| --- | --- | --- |
| 서버 / 컨테이너 | 비밀 관리자(secrets manager)의 환경 변수 (AWS Secrets Manager, HashiCorp Vault, Doppler 등) | 소스 코드에 하드코딩, .env를 git에 커밋 |
| CI/CD 파이프라인 | CI 비밀 변수 (GitHub Actions Secrets, GitLab CI 변수 등) | 파이프라인 로그, 아티팩트 메타데이터 |
| 모바일 앱 | 보안 엔클레이브(Secure enclave) / 키체인(keychain), 인증서 핀닝(certificate pinning) | 앱 바이너리에 포함, 기기 내 plaintext 저장 |
| 클라이언트 측 브라우저 | 서버 API key를 브라우저 JS에 두지 마세요 — 백엔드 프록시(proxy) 사용 | localStorage, window 객체, 소스 코드 |

> 💡
> 인터뷰 팁
> API key 관리 질문은 개발자용 API(예: '결제 API 설계', 'AI 추론 API 설계')를 설계할 때 자주 나옵니다. 핵심 포인트: (1) 저장 전 키를 해싱할 것 — 이때 bcrypt를 사용하지 않는 이유를 설명하세요. (2) 환경과 권한에 따라 키의 범위를 제한할 것. (3) Redis sliding window를 통해 키 레벨에서 rate limit을 적용할 것. (4) 비밀 스캐닝이 가능한 키 접두사를 사용할 것. 회전(rotation)에 대해 질문받으면 두 개의 키가 겹치는 윈도우(two-key overlapping window) 방식을 설명하세요. 이는 가동 중단 없는 회전 방식이며 운영 성숙도를 보여줍니다.
