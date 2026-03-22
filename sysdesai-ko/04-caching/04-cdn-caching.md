# CDN Caching (CDN 캐싱)

> Source: https://www.sysdesai.com/learn/caching/cdn-caching

---

## What Is a CDN? (CDN이란 무엇인가?)

**CDN (Content Delivery Network)**은 사용자 가까이에 콘텐츠를 캐싱하는 엣지 서버(Point of Presence, PoP)의 글로벌 분산 네트워크입니다. 도쿄의 사용자가 미국에 있는 오리진(Origin) 서버에 파일을 요청할 경우, CDN이 없다면 요청은 태평양을 왕복해야 하며 150ms 이상의 지연 시간이 추가됩니다. CDN을 사용하면 도쿄의 엣지 노드에서 요청을 받아 10ms 미만으로 캐싱된 파일을 제공할 수 있습니다.

**CloudFront**(AWS), **Fastly**, **Cloudflare**, **Akamai**와 같은 CDN은 대부분의 아키텍처에서 가장 바깥쪽 캐시 계층 역할을 합니다. 오리진 서버의 부하를 줄이고, 글로벌 성능을 향상시키며, 부수적으로 DDoS 방어 기능도 제공합니다. CDN 캐싱의 작동 방식과 실패 모드(failure modes)를 이해하는 것은 고트래픽 웹 시스템을 설계할 때 필수적입니다.

## Cache-Control Headers (Cache-Control 헤더)

CDN의 캐싱 동작은 오리진 서버가 반환하는 **HTTP Header(HTTP 헤더)**에 의해 제어됩니다. `Cache-Control`이 핵심 지시어입니다:

- `Cache-Control: max-age=86400` — 요청 시점으로부터 24시간 동안 캐싱
- `Cache-Control: s-maxage=3600` — `max-age`와 비슷하지만 공유 캐시(CDN)에만 적용됨; 브라우저는 `max-age`를 사용
- `Cache-Control: no-cache` — 콘텐츠를 제공하기 전에 반드시 오리진 서버에서 재검증(revalidate)해야 함 ('캐싱하지 마라'는 뜻이 아님)
- `Cache-Control: no-store` — 절대 캐싱하지 않음 (은행 계좌 페이지와 같은 민감한 데이터용)
- `Cache-Control: stale-while-revalidate=60` — 백그라운드에서 최신 데이터를 가져오는 동안 최대 60초간 낡은(stale) 콘텐츠를 제공
- `Cache-Control: private` — 브라우저만 캐싱 가능; CDN은 저장하면 안 됨

**ETag**는 검증 헤더입니다 — 응답의 해시나 버전 식별자입니다. 이후 요청에서 브라우저는 `If-None-Match: <etag>`를 보냅니다. 콘텐츠가 변경되지 않았다면 오리진은 본문 없이 `304 Not Modified`를 반환하여 대역폭을 절약합니다. CDN은 오리진과의 캐시 재검증을 위해 ETag를 사용합니다.

## Cache Keys and Vary (캐시 키와 Vary)

CDN은 **Cache Key(캐시 키)**를 매칭하여 캐싱된 응답을 제공할지 결정합니다 — 기본적으로 URL 경로 + 쿼리 스트링 조합입니다. `GET /product?id=42`에 대한 두 번의 요청은 동일한 캐시 항목을 공유합니다. 하지만 다른 차원에 따라 응답이 달라질 수 있으며, `Vary` 헤더는 CDN에게 어떤 요청 헤더가 별도의 캐시 항목을 생성해야 하는지 알려줍니다:

- `Vary: Accept-Encoding` — gzip/br/identity 방식별로 별도 버전 캐싱 (거의 항상 필수적)
- `Vary: Accept-Language` — 언어별 캐싱 (캐시 파편화를 유발할 수 있음)
- `Vary: Cookie` — 쿠키 값별 캐싱 (CDN에서는 대개 부적절하며 캐싱 효과를 사실상 없애버림)

> ⚠️
> CDN에서 Vary: Cookie를 피하세요 (Avoid Vary: Cookie at the CDN)
> `Vary: Cookie`는 고유한 쿠키 값마다 별도의 캐시 항목을 생성한다는 의미입니다. 대부분의 사용자는 고유한 세션 쿠키를 가지고 있으므로, 이는 CDN 캐싱을 완전히 무력화합니다. 대신 공개적으로 캐싱 가능한 경로에 대해서는 CDN 설정에서 쿠키를 제거(strip)하거나, 인증된 콘텐츠에는 별도의 도메인/경로를 사용하세요.

## Cache Purging Strategies (캐시 제거 전략)

오리진에서 콘텐츠가 변경되면 CDN 캐시를 무효화해야 합니다. 몇 가지 전략이 있습니다:

| Strategy(전략) | How It Works(작동 방식) | Pros(장점) | Cons(단점) |
| --- | --- | --- | --- |
| TTL Expiry | 캐시 항목이 자연스럽게 만료되기를 기다림 | 운영 복잡도 제로 | TTL 만료 전까지 낡은 콘텐츠 제공 |
| Purge by URL | 특정 URL에 대해 제거 API 호출 | 정밀하고 즉각적임 | 모든 캐시된 URL을 알아야 함; 호출이 많아질 수 있음 |
| Surrogate Keys (Cache Tags) | 응답에 콘텐츠 ID 태그를 부여; 태그별로 제거 | 리소스의 모든 변체를 원자적으로 제거 가능 (Fastly, Cloudflare) | 오리진에서 태깅을 구현해야 함 |
| Cache Busting (versioned URLs) | URL에 콘텐츠 해시 포함 (예: `main.a3f9b.js`) | 무한 TTL 가능; 불변 자산에 완벽함 | 배포마다 URL 변경; 빌드 파이프라인 필요 |

## Dynamic Content Caching (동적 콘텐츠 캐싱)

CDN은 정적 자산만을 위한 것이 아닙니다. 현대적인 CDN은 적절한 헤더만 있다면 API 응답과 HTML 페이지도 캐싱할 수 있습니다. '동적' 콘텐츠를 캐싱하는 기법들은 다음과 같습니다:

- **Edge Side Includes (ESI)**: 내비게이션 바나 헤더 같은 정적 부분과 장바구니 같은 동적 부분으로 구성된 페이지를 엣지에서 조합합니다. Fastly와 Varnish가 ESI를 지원합니다.
- **Stale-While-Revalidate**: 낡은 캐시 버전을 즉시 제공하고 백그라운드에서 재검증합니다. 사용자는 항상 빠른 응답을 받으며, 최종 일관성을 유지합니다.
- **Short TTL Caching**: API 응답을 5초 동안 캐싱합니다. 초당 10만 건의 요청(100k RPS)이 들어올 때, 이는 5초마다 단 1번의 오리진 요청으로 변합니다 — 요청량을 50만 배 줄이는 효과가 있습니다.
- **Segment by Authentication**: 공개 엔드포인트에는 `s-maxage`를 사용하고, 인증된 응답에는 `Cache-Control: private`를 사용합니다. 공개 경로에 대해서는 CDN에서 인증 쿠키를 제거합니다.

📌
Real-world example: Cache-busting for static assets (실사례: 정적 자산의 캐시 버스팅)

배포 시 Webpack/Vite는 JS/CSS 파일 이름에 콘텐츠 해시를 추가합니다: `app.a3f9b2.js`. 이 파일들은 `Cache-Control: max-age=31536000, immutable` (1년 TTL) 헤더와 함께 제공됩니다. 배포마다 URL이 변경되므로 이전 URL이 낡은 코드를 제공할 일이 없습니다. 새로운 URL은 처음에는 캐시 미스로 시작하여 전 세계적으로 첫 번째 히트 때 CDN 캐시를 채우게 됩니다. 이것이 정적 자산 배포의 골드 스탠다드입니다.

> 💡
> Interview Tip (인터뷰 팁)
> CDN 관련 질문은 미디어 플랫폼, e-commerce 사이트 또는 글로벌 서비스를 설계할 때 등장합니다. 핵심 포인트: (1) 공개 API 응답에는 아주 짧은(5~60초) 시간이라도 `s-maxage`를 설정하세요. 대규모 환경에서 이는 혁신적인 변화를 가져옵니다. (2) 불변의 정적 자산에는 긴 TTL과 함께 Cache-busting(버전이 지정된 URL)을 사용하세요. (3) 항상 캐시 무효화 전략을 언급하세요 — 업데이트를 배포할 때 어떤 일이 벌어지나요? (4) 인증을 위해 별도의 도메인을 사용하거나 쿠키를 제거하여 CDN이 공개 콘텐츠를 캐싱할 수 있도록 하세요.
