# Backend for Frontend (BFF)

> 원문: https://www.sysdesai.com/learn/decomposition-integration/backend-for-frontend

---

## 천편일률적인 API (One-Size-Fits-All API) 문제

단일 API가 모바일 앱과 웹 대시보드 모두에 서비스를 제공할 때, 트레이드오프(trade-offs)가 빠르게 누적됩니다. 웹 앱은 복잡한 UI를 위해 풍부하고 깊게 중첩된 데이터가 필요합니다. 반면 모바일 앱은 배터리에 효율적인 가벼운 페이로드가 필요합니다. 결국 어떤 클라이언트도 필요한 것을 정확히 얻지 못합니다. 웹 앱은 여러 번의 round trips를 수행해야 하고, 모바일 앱은 표시하지도 않을 필드들을 다운로드하게 되며, 한쪽 클라이언트의 요구사항이 바뀔 때마다 다른 쪽이 깨질 위험을 감수해야 합니다.

*Building Microservices*의 저자인 Sam Newman이 대중화한 **Backend for Frontend (BFF)** 패턴은 각 프론트엔드 클라이언트 유형마다 전용 백엔드 서비스를 제공하여 이 문제를 해결합니다. BFF는 특정 클라이언트의 요구사항에 정확히 맞춘 API composition 계층 역할을 합니다.

## BFF 아키텍처

각 프론트엔드 유형은 다운스트림 microservices 호출을 집계하고 해당 클라이언트에 맞게 응답 형식을 조정하는 자체 BFF를 가집니다.

## BFF의 역할

- **API Composition** — 여러 다운스트림 서비스 호출을 단일 응답으로 집계하여, 클라이언트 측의 chattiness(번거로운 통신)를 제거합니다.
- **Response Shaping** — 클라이언트가 필요한 필드만 반환합니다 (over-fetching 또는 under-fetching 방지).
- **Protocol Translation** — 웹 BFF는 REST/JSON을 사용하고, 모바일 BFF는 더 작은 페이로드를 위해 Protocol Buffers를 사용할 수 있습니다.
- **Auth Token Handling** — BFF는 OAuth 토큰 교환, 세션 갱신을 처리하고 클라이언트에 보내기 전 민감한 필드를 제거할 수 있습니다.
- **Rate Limiting / Throttling** — 클라이언트별 할당량을 독립적으로 강제할 수 있습니다.

## BFF vs API Gateway

이는 면접에서 흔히 혼동하는 지점입니다. API Gateway와 BFF는 서로 다른 목적을 가지며 종종 함께 사용됩니다:

| 구분 | API Gateway | BFF |
| --- | --- | --- |
| 소유권 | 플랫폼 / 인프라 팀 | 프론트엔드 제품 팀 |
| 범위 | 모든 클라이언트, 모든 서비스 | 하나의 특정 클라이언트 유형 |
| 로직 | Thin — 라우팅, 인증, rate limiting | Rich — 집계, 변환, 비즈니스 로직 |
| 커스터마이징 | 일반적인 횡단 관심사 (cross-cutting concerns) | 특정 클라이언트 UI 요구사항에 고도로 최적화 |
| 배포 | 환경당 하나의 게이트웨이 | 클라이언트 유형당 하나의 BFF |

> ℹ️
> 일반적인 토폴로지 (Typical Topology)
> 실제 적용 시: Client → API Gateway (인증, SSL 종료, rate limiting) → BFF (집계, 형식 조정) → Microservices 순으로 구성됩니다. API Gateway는 'edge' 관련 문제를 처리하고, BFF는 '클라이언트 전용' 문제를 처리합니다.

## 실제 사례: SoundCloud

SoundCloud는 BFF 패턴의 기원으로 널리 알려져 있습니다. 웹 앱과 함께 네이티브 모바일 클라이언트를 추가했을 때, 그들은 단일 API가 양쪽 모두를 제대로 만족시키지 못한다는 것을 깨달았습니다. 그들은 각 프론트엔드와 가장 가까운 팀이 소유하고 발전시키는 별도의 백엔드 서비스를 만들었습니다. 웹 팀은 모바일 팀과 조율할 필요 없이 자신들의 속도에 맞춰 API 변경사항을 배포할 수 있었고, 모바일 팀도 마찬가지였습니다.

## 함정과 트레이드오프

- **코드 중복 (Code duplication)** — 다운스트림 서비스에 있어야 할 비즈니스 로직이 여러 BFF로 스며들 수 있습니다. BFF는 가능한 가볍게(thin) 유지하고, 공유 로직은 다운스트림으로 밀어내십시오.
- **팀 소유권 (Team ownership)** — BFF는 별도의 백엔드 팀이 아닌, 해당 서비스를 사용하는 프론트엔드 팀이 소유해야 합니다. 소유권이 일치하지 않으면 BFF는 병목 지점이 됩니다.
- **난립 (Proliferation)** — 기능이나 화면마다 BFF를 만드는 것은 지양하십시오. 주요 클라이언트 유형(웹, 모바일, 제3자 파트너 등)당 하나의 BFF가 적절한 단위입니다.
- **지연 시간 (Latency)** — 추가적인 hop이 발생합니다. BFF가 다운스트림 호출을 할 때 가능한 한 병렬로(`Promise.all` 등 사용) 처리하도록 하십시오.

> ⚠️
> BFF에 비즈니스 로직을 넣지 마십시오
> 비즈니스 규칙이 쌓인 BFF는 새로운 monolith가 됩니다. 만약 BFF에서 '프리미엄 고객이면 20% 할인 적용'과 같은 코드를 작성하고 있다면, 그 로직은 다운스트림 서비스에 있어야 합니다. BFF는 결정하는 곳이 아니라 구성하고 형식을 맞추는 곳이어야 합니다.

> 💡
> 면접 팁 (Interview Tip)
> 모바일 앱과 웹 대시보드가 모두 있는 시스템(예: Uber, Instagram, Airbnb)을 설계하는 면접에서 BFF 패턴을 선제적으로 언급하십시오. "모바일에서의 over-fetching과 웹에서의 under-fetching을 피하기 위해, 공유 API gateway 뒤에 각 프론트엔드 팀이 소유하는 클라이언트 유형별 BFF를 도입하겠습니다"라고 말하십시오. 이는 단순히 '앞단에 API gateway를 둔다'는 것 이상의 제품 이해도와 아키텍처 깊이를 보여줍니다.
