# 콘텐츠 전송 네트워크(CDN, Content Delivery Networks)

> Source: https://www.sysdesai.com/learn/networking/cdn

---

## CDN이란 무엇인가?

**콘텐츠 전송 네트워크(CDN, Content Delivery Network)**는 사용자에게 물리적으로 가까운 위치에서 콘텐츠를 캐싱하는 프록시 서버(이를 **에지 노드(Edge nodes)** 또는 **거점(PoP, Points of Presence)**이라고 부름)의 글로벌 분산 네트워크입니다. 모든 요청이 버지니아에 있는 원본 서버(Origin server)에 도달하기 위해 대서양을 건너는 대신, 도쿄에 있는 사용자는 오사카의 PoP에서 서비스를 제공받게 됩니다. 이를 통해 왕복 시간(Round-trip time)을 약 200ms에서 5ms로 대폭 단축할 수 있습니다.

CDN은 사실상 모든 고트래픽 웹사이트에서 사용됩니다. Cloudflare, Akamai, AWS CloudFront, Fastly는 전 세계 인터넷 트래픽의 상당 부분을 처리하고 있습니다. CDN은 지연 시간(Latency), 가용성(Availability), 원본 서버 부하 문제를 한 번에 해결할 수 있는 핵심 설계 요소이므로 시스템 디자인 면접에서 필수적으로 이해해야 합니다.

## Pull CDN vs Push CDN

CDN은 에지 캐시(Edge cache)를 채우는 방식에 따라 두 가지 기본 모드로 작동합니다.

| 측면 | Pull CDN | Push CDN |
| --- | --- | --- |
| 에지 전달 방식 | 캐시 미스(Cache miss)가 발생할 때 에지가 원본에서 데이터를 가져와 로컬에 캐싱함 | 사용자가 모든 에지 노드에 콘텐츠를 선제적으로 업로드함 |
| 설정 복잡도 | 낮음 — DNS가 CDN을 가리키도록 설정하면 끝 | 높음 — 업데이트가 있을 때마다 직접 푸시해야 함 |
| 적합한 사례 | 접속 패턴을 예측하기 어려운 동적 사이트 | 배포 시점을 알고 있는 대용량 정적 자산(비디오, 소프트웨어 다운로드) |
| 스토리지 비용 | 실제로 요청된 콘텐츠만 캐싱함 | 모든 PoP에 모든 콘텐츠를 저장해야 함 |
| 신선도 제어 | TTL 기반; TTL이 만료될 때까지 오래된 콘텐츠가 제공될 수 있음 | 콘텐츠가 배포되는 시점을 정확히 제어 가능 |
| 예시 | Cloudflare, CloudFront (기본 모드) | Akamai NetStorage, S3 푸시를 사용하는 CloudFront |

> 💡
> 면접에서 CDN을 추천해야 하는 시점
> 설계에서 정적 자산(JS, CSS, 이미지), 비디오 스트리밍, 대용량 파일 다운로드 또는 전 세계에 분산된 사용자 층을 다룰 때 CDN을 언급하세요. 또한 CDN은 원본 서버에 도달하기 전 에지에서 DDoS 트래픽을 흡수할 수도 있습니다. 순수 내부 서비스(마이크로서비스 간 호출)의 경우 CDN은 적합하지 않습니다.

## 캐시 제어 헤더(Cache Control Headers)

CDN의 동작은 주로 원본 서버가 설정한 HTTP 응답 헤더를 통해 제어됩니다. 가장 중요한 헤더는 `Cache-Control`과 `ETag`입니다.

```http
# CDN에서는 1시간 동안 캐싱하지만, 브라우저에서는 캐싱하지 않음
Cache-Control: public, max-age=3600, s-maxage=3600, no-store

# 파일 이름에 콘텐츠 해시가 포함된 불변(Immutable) 자산 — 영구 캐싱
Cache-Control: public, max-age=31536000, immutable

# 개인 사용자 데이터 — CDN에 절대 캐싱하지 않음
Cache-Control: private, no-cache, no-store

# 조건부 요청을 위한 ETag
ETag: "abc123def456"
Last-Modified: Wed, 19 Feb 2026 10:00:00 GMT
```

- **`Cache-Control: public`**: CDN 및 공용 캐시가 응답을 저장할 수 있도록 허용합니다.
- **`s-maxage`**: 공용 캐시(CDN)에만 적용되는 `max-age` 값을 설정합니다. 이를 통해 브라우저와 에지의 TTL을 다르게 가져갈 수 있습니다.
- **`immutable`**: 브라우저에 `max-age`가 만료된 후에도 리소스를 재검증(Revalidate)하지 않도록 지시합니다. 콘텐츠 해시가 포함된 파일 이름과 함께 사용됩니다.
- **`ETag`**: 콘텐츠의 지문(Fingerprint)입니다. 브라우저나 CDN은 `If-None-Match`를 보내 내용이 변경되지 않았을 경우 다운로드 없이 업데이트 여부만 확인할 수 있습니다.
- **`Vary`**: 요청 헤더에 따라 다른 버전을 캐싱하도록 CDN에 지시합니다(예: gzip 버전과 br 버전을 구분하기 위한 `Vary: Accept-Encoding`).

## CDN 계층의 캐시 무효화(Cache Invalidation)

CDN 캐시 무효화는 운영상 가장 어려운 문제 중 하나입니다. 콘텐츠가 전 세계 수백 개의 에지 노드에 캐싱되고 나면, 오래된 데이터를 제거하기 위해 명시적인 퍼지(Purge) API가 필요합니다.

- **TTL 만료(TTL expiry)**: 가장 간단한 방법입니다. TTL이 지날 때까지 기다립니다. 운영 비용은 없지만 TTL 기간 동안 오래된 콘텐츠가 제공될 수 있습니다.
- **URL 퍼지 / 캐시 태그 퍼지(Cache tag purge)**: API를 통해 특정 URL이나 태그가 지정된 객체 그룹을 삭제하도록 CDN에 명시적으로 요청합니다. Cloudflare, Fastly, CloudFront 모두 이를 지원합니다.
- **버전 관리 URL을 통한 캐시 버스팅(Cache-busting)**: 파일 이름에 콘텐츠 해시를 포함시킵니다(`main.a3f8c2.js`). 파일이 변경되면 URL도 변경되므로 이전의 캐시된 버전은 무의미해집니다. 정적 자산을 위한 가장 안정적인 전략입니다.
- **서로게이트 키 / 캐시 태그(Surrogate keys / Cache tags)**: 객체 그룹을 태깅(예: 제품 ID 42가 포함된 모든 페이지)하고 제품 정보가 변경될 때 한 번에 무효화합니다. Fastly와 Cloudflare가 이를 기본적으로 지원합니다.

📌
실무 패턴: 불변 자산 + 짧은 수명의 HTML

Netflix 및 현대적인 웹 프레임워크는 2계층 전략을 사용합니다. JavaScript/CSS 번들은 콘텐츠 해시가 포함된 파일 이름을 사용하고 1년의 `max-age`(immutable)를 설정합니다. 해당 번들을 참조하는 HTML 페이지는 짧은 TTL(30-60초)을 갖거나 `no-cache`를 설정합니다. 이렇게 하면 자산은 에지 캐시에서 매우 빠르게 제공되면서도, HTML은 항상 최신 상태를 유지할 수 있습니다.

## 동적 콘텐츠를 위한 CDN(CDN for Dynamic Content)

현대적인 CDN은 캐싱할 수 없는 동적 콘텐츠도 가속화할 수 있습니다. 이는 **원본 실드(Origin shielding)**(캐시 미스를 단일 실드 PoP로 집중시켜 원본 부하를 줄임), **프로토콜 최적화**(원본과 지속적인 HTTP/2 또는 HTTP/3 연결 유지), **에지 컴퓨팅**(Cloudflare Workers나 Lambda@Edge를 통해 에지에서 인증, A/B 테스팅, 개인화 로직 실행)을 통해 이루어집니다.

> 💡
> 면접 팁
> 자주 나오는 추가 질문: '사용자 데이터가 개인화되어 있어도 CDN을 사용할 수 있나요?' 답은 '예'이지만 세부 사항이 중요합니다. 페이지 쉘(사용자 특정 데이터가 없는 HTML)은 CDN에 캐싱하고, 개인화된 섹션은 CDN을 우회하는 클라이언트 측 API 호출로 가져올 수 있습니다. 또는 에지 컴퓨팅을 사용하여 캐시 히트 후 개인화된 데이터를 주입할 수 있습니다. 이 패턴을 '에지 사이드 인클루드(ESI, Edge Side Includes)' 또는 '에지에서의 개인화를 동반한 stale-while-revalidate'라고 부릅니다.
