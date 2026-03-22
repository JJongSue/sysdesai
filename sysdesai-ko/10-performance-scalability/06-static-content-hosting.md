# Static Content Hosting 패턴

> 출처: https://www.sysdesai.com/learn/performance-scalability/static-content-hosting

---

## Static Content Hosting이란?

**Static Content Hosting 패턴**은 정적 asset — HTML, CSS, JavaScript 번들, 이미지, 폰트, 동영상 — 을 애플리케이션 서버에서 목적에 맞게 구축된 인프라로 이동시킵니다: object storage (S3, GCS)와 CDN (CloudFront, Cloudflare, Fastly)의 조합입니다. 애플리케이션 서버는 바이트 전송에서 해방되어 완전히 연산에 집중할 수 있습니다. 이는 모든 웹 시스템에서 사용할 수 있는 가장 높은 레버리지의 성능 최적화 중 하나입니다.

Netflix는 S3 + CloudFront에서 프론트엔드 asset을 제공하여 origin 트래픽을 95% 이상 줄입니다. GitHub Pages, Vercel, Netlify는 이 패턴으로 완전히 구축되어 있습니다 — 저장소의 정적 출력이 전 세계 edge node에 배포됩니다.

## 아키텍처
정적 asset 흐름: CDN edge가 캐시된 asset을 제공; miss 시 object storage에서 가져옴
## Cache-Control 헤더

CDN caching 동작은 각 asset에 설정된 HTTP `Cache-Control` 헤더로 제어됩니다. 최적 전략은 **content-hash 기반 버전 관리**입니다: 번들 파일명에 콘텐츠의 hash가 포함됩니다 (예: `main.a3f7b2.js`). Hash는 콘텐츠가 변경될 때만 바뀌므로 이 파일들은 **영구적으로** 캐시될 수 있습니다 (`max-age=31536000, immutable`). Hash된 asset을 참조하는 HTML 파일은 매우 짧은 시간 (`max-age=60`) 동안 또는 전혀 캐시되지 않습니다.

| Asset 유형 | Cache-Control 헤더 | 이유 |
| --- | --- | --- |
| Hash된 JS/CSS 번들 | max-age=31536000, immutable | 주어진 hash에 대해 콘텐츠가 절대 변경되지 않음; 영구 캐시 |
| HTML 파일 | max-age=60 또는 no-cache | HTML이 hash된 asset을 참조; 새 배포가 전파되도록 최신 상태 유지 필요 |
| 이름에 hash가 있는 이미지 | max-age=31536000, immutable | JS/CSS와 동일 — hash가 콘텐츠 무결성 보장 |
| API 응답 | no-cache 또는 private | 동적 콘텐츠; CDN edge에서 캐시되어서는 안 됨 |

## CDN Cache Invalidation

Asset이 hash 버전 관리되지 않은 경우, 배포 시 CDN cache invalidation이 필요합니다. 대부분의 CDN은 경로 기반 invalidation을 지원합니다 (`CloudFront: /static/*`). Invalidation은 모든 edge node에 전파되지만 몇 분이 걸릴 수 있습니다. 이것이 **hash 기반 버전 관리가 선호되는** 이유입니다 — 이전 URL이 여전히 유효하고 (이전 빌드를 가리킴) 새 URL이 자동으로 새 파일을 가져오기 때문에 invalidation이 필요 없습니다.

> 💡
> 불변 URL을 사용한 원자적 배포
> Hash된 asset URL은 불변이므로 애플리케이션의 이전 버전과 새 버전이 CDN에서 동시에 공존할 수 있습니다. 배포 중에 이전 HTML을 로드한 사용자는 이전 hash된 asset을 계속 사용합니다 (여전히 edge에 캐시됨). 새 사용자는 새 HTML을 로드하고 새 hash된 asset을 받습니다. 사용자가 불일치하는 HTML과 JS를 받는 시간이 없습니다 — 이는 사용자 관점에서 배포를 원자적으로 만듭니다.

## Edge Computing 확장

현대 CDN은 **edge computing** (Cloudflare Workers, CloudFront Functions, Vercel Edge Runtime)으로 순수 파일 제공을 넘어 확장됩니다. Edge node에서 경량 JavaScript를 실행하여 다음을 처리할 수 있습니다: A/B 테스트 라우팅, auth 쿠키 검증, 지리적 리다이렉트, 심지어 server-side rendering. 이는 더 많은 로직을 edge로 이동시켜 origin 부하를 더욱 줄입니다.

> 💡
> 인터뷰 팁
> 디자인 인터뷰에서 웹 프론트엔드를 그리는 즉시 모든 정적 asset이 CDN으로 간다고 언급하세요. 이는 실제 프로덕션 아키텍처에 대한 인식을 보여줍니다. Hash 기반 버전 관리와 immutable cache-control 헤더를 언급하여 '앞에 CDN을 두세요'를 넘어서는 깊이를 보여주세요. 면접관이 cache invalidation에 대해 질문하면, hash 버전 방식이 invalidation의 필요성을 없앤다고 설명하세요.
