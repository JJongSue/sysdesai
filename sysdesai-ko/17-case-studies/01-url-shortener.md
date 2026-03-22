# URL Shortener 설계

> 출처: https://www.sysdesai.com/learn/case-studies/url-shortener

---

## 문제 정의 (Problem Statement)

URL Shortener는 긴 URL을 입력받아 짧은 별칭(alias)을 반환합니다. 예를 들어, `https://www.example.com/very/long/path?query=foo`를 `https://sho.rt/aB3xQz`로 변환합니다. 사용자가 짧은 URL을 방문하면 서비스는 원래의 URL로 리다이렉트(redirect)합니다. 이 문제는 Hashing, Database 설계, Caching, 그리고 고처리량(high-throughput) Read 최적화를 모두 다루기 때문에 인터뷰에서 가장 자주 묻는 문제 중 하나입니다.

## 요구사항 (Requirements)

### 기능적 요구사항 (Functional Requirements)

- 긴 URL이 주어지면 고유한 짧은 URL(alias)을 생성합니다.
- 사용자를 짧은 URL에서 원래의 긴 URL로 리다이렉트합니다.
- 커스텀 별칭(custom aliases)을 지원합니다 (예: `sho.rt/my-brand`).
- 선택적으로 짧은 URL에 만료 시간(expiry time)을 설정할 수 있습니다.
- 클릭 분석(analytics)을 추적합니다 (count, geo, device, referrer).

### 비기능적 요구사항 (Non-Functional Requirements)

- **고가용성 (High availability)**: 시스템의 일부가 다운되더라도 리다이렉트가 성공해야 합니다.
- **낮은 지연 시간 (Low latency)**: 리다이렉트는 99th percentile에서 10ms 미만으로 완료되어야 합니다.
- **내구성 (Durability)**: 생성된 URL은 손실되지 않아야 합니다.
- **확장성 (Scale)**: 하루 1억 개의 새로운 URL 생성, 하루 100억 개의 리다이렉트를 지원해야 합니다.

## 용량 산정 (Capacity Estimation)

| 지표 (Metric) | 가정 (Assumption) | 결과 (Result) |
| --- | --- | --- |
| 새로운 URL/일 | 100 M | ~1,160 writes/sec |
| 리다이렉트/일 | 10 B (100:1 read/write) | ~115,740 reads/sec |
| URL당 저장 용량 | 500 bytes (URL + metadata) |  |
| 5년치 저장 용량 | 100 M × 365 × 5 × 500 B | ~91 TB |
| 캐시 크기 (20% hot) | 10 B × 0.2 × 500 B/day | ~1 TB/day hot set |

> 💡
> 산정 팁 (Estimation Tip)
> 항상 Read와 Write QPS를 분리하세요. URL Shortener는 읽기 작업이 매우 많습니다 (일반적으로 100:1 비율). 이는 Read Replicas, 공격적인 Caching, 그리고 CDN 우선 리다이렉트 전략으로 이어집니다.

## 상위 수준 설계 (High-Level Design)
URL Shortener를 위한 상위 수준 아키텍처 (High-level architecture for a URL shortener)

## 짧은 코드 생성 (Short Code Generation)

핵심 과제는 각 긴 URL에 대해 **고유하고, 간결하며, URL-safe한 식별자**를 생성하는 것입니다. 세 가지 일반적인 접근 방식이 있습니다:

| 접근 방식 (Approach) | 작동 방식 | 장점 | 단점 |
| --- | --- | --- | --- |
| MD5 / SHA-256 Truncation | 긴 URL을 해싱하고 앞의 7글자를 사용 | Stateless, 빠름 | 충돌 가능성; 동일한 URL은 동일한 코드를 가짐 |
| Base62 Auto-Increment ID | DB의 auto-increment ID를 base-62로 인코딩 | 충돌 없음, 예측 가능한 길이 | 순차적 ID 노출; 조정(coordination) 필요 |
| Snowflake / UUID + Base62 | 분산 고유 ID 생성기 사용 | 조정 불필요, 전역적으로 고유함 | 인프라 구조가 약간 더 복잡함 |

**권장 접근 방식**: 분산 ID 생성기(Snowflake 스타일)를 사용하고 ID를 base-62로 인코딩합니다. base-62로 인코딩된 64비트 ID는 충돌이 없고 불투명한(opaque) 7~11자 코드를 생성합니다. 커스텀 별칭의 경우 직접 저장하고 쓰기 시점에 충돌 여부를 확인합니다.

python

```python
import string

ALPHABET = string.ascii_lowercase + string.ascii_uppercase + string.digits  # 62 chars

def to_base62(num: int) -> str:
    if num == 0:
        return ALPHABET[0]
    result = []
    while num:
        result.append(ALPHABET[num % 62])
        num //= 62
    return "".join(reversed(result))

# 예시: ID 1_000_000_000 → "15ftgG" (6 chars)
print(to_base62(1_000_000_000))  # "15FtGG" (approx)
```

## API 설계 (API Design)

| 메서드 (Method) | 엔드포인트 (Endpoint) | 요청 본문 (Request Body) | 응답 (Response) |
| --- | --- | --- | --- |
| POST | /api/v1/urls | `{ longUrl, customAlias?, expiresAt? }` | `201 { shortCode, shortUrl, expiresAt }` |
| GET | /:shortCode | — | `301/302 Location: ` |
| GET | /api/v1/urls/:shortCode/stats | — | `200 { clicks, geo, devices }` |
| DELETE | /api/v1/urls/:shortCode | — | `204 No Content` |

> ℹ️
> 301 vs 302 리다이렉트
> 모든 클릭을 서버 측에서 추적하고 싶다면 **302 Found** (임시)를 사용하세요. 브라우저가 이를 캐시하지 않습니다. 브라우저와 CDN이 캐시하여 원본 서버의 부하를 줄이고 싶다면 **301 Moved Permanently**를 사용하세요. 분석 중심의 URL Shortener는 대부분 302를 사용합니다.

## 핵심 흐름: 리다이렉트 (Key Flow: Redirect)
CDN 및 Redis 캐시 계층을 포함한 리다이렉트 흐름 (Redirect flow with CDN and Redis cache layers)

## 데이터베이스 스키마 (Database Schema)

sql

```sql
CREATE TABLE urls (
  id          BIGINT PRIMARY KEY,          -- Snowflake ID
  short_code  VARCHAR(16) UNIQUE NOT NULL, -- base-62 encoded ID or custom alias
  long_url    TEXT NOT NULL,
  user_id     BIGINT,                      -- nullable for anonymous
  created_at  TIMESTAMP DEFAULT NOW(),
  expires_at  TIMESTAMP,                   -- NULL = never expires
  click_count BIGINT DEFAULT 0
);

CREATE INDEX idx_short_code ON urls(short_code);  -- primary lookup path

CREATE TABLE clicks (
  id          BIGSERIAL PRIMARY KEY,
  short_code  VARCHAR(16) NOT NULL,
  clicked_at  TIMESTAMP DEFAULT NOW(),
  ip_hash     VARCHAR(64),
  country     VARCHAR(2),
  user_agent  TEXT
);
```

## 확장성 고려 사항 (Scaling Considerations)

- **Read Replicas**: 트래픽의 99%는 읽기(리다이렉트)입니다. 모든 읽기는 복제본(replicas)으로, 쓰기는 기본(primary) 서버로 라우팅합니다.
- **Redis Cluster**: 자주 사용되는 짧은 코드를 URL 만료 시간과 동일한 TTL로 Redis에 캐싱합니다. 제거 정책(Eviction policy)은 `allkeys-lru`를 사용합니다.
- **CDN Edge Caching**: 만료되지 않는 URL의 경우 CDN Edge에서 301 리다이렉트를 캐싱하여 원본 요청을 완전히 제거합니다.
- **Sharding**: 수평적 확장을 위해 `short_code` 해시를 기준으로 `urls` 테이블을 샤딩합니다. Consistent Hashing을 사용하면 재균형(rebalancing)의 고통을 피할 수 있습니다.
- **분석 파이프라인 (Analytics pipeline)**: 클릭 이벤트를 Kafka에 비동기적으로 기록합니다. Flink나 Spark가 이를 데이터 웨어하우스로 집계합니다. 리다이렉트 경로에서 직접 분석 데이터를 기록하지 마세요.
- **Rate Limiting**: 남용을 방지하기 위해 API Gateway에서 사용자당 및 IP당 제한을 적용합니다.

> ⚠️
> 흔한 실수: 블룸 필터 (Bloom Filters)
> 일부 후보자는 DB 조회 전에 기존 짧은 코드를 확인하기 위해 Bloom Filter를 사용할 것을 제안합니다. 창의적이지만, Redis가 이미 모든 활성 URL을 캐싱하고 있다면 큰 이득 없이 복잡성만 더합니다. 매우 큰 규모에서 고려할 수 있는 최적화로 언급하되, 주된 해결책으로 내세우지는 마세요.

> 💡
> 인터뷰 팁 (Interview Tip)
> 면접관은 이 문제에서 세 가지를 확인합니다: (1) 방어 가능한 충돌 없는 ID 전략, (2) 캐싱 계층 구조와 각 계층의 존재 이유에 대한 명확한 설명, (3) 301 vs 302 트레이드오프에 대한 인식. 분석을 비동기적인 관심사로 언급하는 것은 실무적인 성숙도를 보여줍니다.

이 패턴 연습하기
[Design a URL shortener like bit.ly](https://www.sysdesai.com/design/new?prompt=Design%20a%20URL%20shortener%20like%20bit.ly&mode=fast)
