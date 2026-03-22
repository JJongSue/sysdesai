# Feature Flags

> 원문: https://www.sysdesai.com/learn/deployment-operations/feature-flags

---

## 핵심 개념: 배포(Deployment)는 출시(Release)가 아니다

**Feature flags**(feature toggles 또는 feature switches라고도 함)는 **배포**와 **출시**를 분리합니다. 배포는 코드가 실제 운영 서버로 전송되는 시점입니다. 출시는 사용자가 기능을 보고 사용할 수 있게 되는 시점입니다. Feature flags를 사용하면 코드를 100%의 서버에 배포하되 0%의 사용자에게만 활성화하거나, 내부 직원에게만 활성화하거나, 특정 지역의 사용자 10%에게만 활성화할 수 있습니다.

이는 팀이 소프트웨어를 제공하는 방식의 근본적인 변화입니다. 한 번에 모든 것을 공개하는 'big-bang' 출시를 조율하는 대신, 팀은 작은 코드 변경 사항을 플래그 뒤에 숨겨 지속적으로 머지합니다. 기능이 언제 '라이브(go live)'가 될지는 배포 파이프라인이 아닌 비즈니스 의사결정에 따라 결정됩니다.

## Feature Flags의 네 가지 유형

Pete Hodgson의 분류(Martin Fowler의 블로그에서 인용)에 따르면, 수명과 대상이 서로 다른 네 가지 유형의 플래그가 있습니다:

| 유형 | 목적 | 소유자 | 수명 | 예시 |
| --- | --- | --- | --- | --- |
| Release toggle | 개발 중 미완성 기능 숨기기 | 엔지니어링 | 짧음 (수 일~수 주) | 새로운 결제 플로우 — 기능 완성 전까지 비활성화 |
| Experiment toggle | 영향을 측정하기 위한 기능 변이 A/B 테스트 | 프로덕트 / 데이터 사이언스 | 중간 (수 주) | 두 가지 추천 알고리즘 테스트 |
| Ops toggle | 위험한 하위 시스템을 위한 운영용 서킷 브레이커(circuit breakers) | SRE / 운영팀 | 중간 ~ 영구적 | 부하 발생 시 ML inference를 끄고 규칙 기반 스코어링으로 대체 |
| Permission toggle | 특정 사용자 세그먼트에 기능 활성화 | 프로덕트 / 과금팀 | 긺 (수 개월~영구적) | 엔터프라이즈 티어 사용자에게만 고급 분석 대시보드 제공 |

> ℹ️
> 플래그 세분화 (Flag Granularity)
> 플래그는 개별 사용자(사용자 ID 기반), 사용자 비율, 사용자 세그먼트(속성 기반), 조직 또는 테넌트(tenant), 지리적 지역, 또는 기기 유형 등 다양한 기준으로 타겟팅할 수 있습니다. LaunchDarkly와 같은 최신 플래그 플랫폼은 이러한 차원들을 결합한 복잡한 불리언(boolean) 및 다변량(multi-variate) 규칙을 지원합니다.

## Feature Flag 시스템의 아키텍처

운영 수준의 feature flag 시스템은 세 가지 구성 요소로 이루어집니다:
Feature flag 시스템: 플래그는 중앙에서 정의되고, SDK 내에서 로컬로 평가되며, 분석을 위해 이벤트가 다시 스트리밍됩니다.

- **Flag management service:** 제품 매니저, 엔지니어, SRE가 플래그 규칙, 타겟 대상, 롤아웃 비율을 정의하는 대시보드입니다. 예: LaunchDarkly, Unleash, Split.io, AWS AppConfig, GrowthBook (오픈 소스).
- **SDK (In-process evaluation):** SDK는 애플리케이션 프로세스 내부에서 실행됩니다. 플래그 규칙을 로컬에 캐싱하고(스트리밍이나 폴링으로 업데이트), 네트워크 호출 없이 로컬에서 즉시 플래그를 평가합니다. 이를 통해 지연 시간(latency) 영향을 거의 0에 가깝게 유지합니다.
- **Analytics / audit log:** 디버깅, 감사 가능성, 실험 분석을 위해 모든 플래그 평가 결과가 스트리밍됩니다. 이를 통해 실험군(treatment group)의 전환율이 더 높았음을 증명할 수 있습니다.

## 코드 예시: 플래그 평가하기

typescript

```
// LaunchDarkly SDK — 서버 측 평가 예시
import * as ld from '@launchdarkly/node-server-sdk';

const client = ld.init(process.env.LD_SDK_KEY);
await client.waitForInitialization();

async function getRecommendations(userId: string): Promise<Recommendation[]> {
  const context = { kind: 'user', key: userId };

  // 다변량(multi-variate) 플래그 평가
  const algorithm = await client.variation(
    'recommendation-algorithm',  // 플래그 키
    context,
    'collaborative-filtering'    // SDK 호출 실패 시의 기본값
  );

  if (algorithm === 'llm-rerank') {
    return await llmRerankedRecommendations(userId);
  } else {
    return await collaborativeFilteringRecommendations(userId);
  }
}
```

## 플래그 수명 주기 관리

플래그는 시간이 지남에 따라 쌓입니다. 수백 개의 오래된 플래그가 남아 있는 코드베이스는 유지보수의 지옥이 됩니다. 모든 엔지니어가 어떤 브랜치가 죽은 코드인지 파악해야 하기 때문입니다. 이를 **플래그 부채(flag debt)**라고 하며, feature flag 도입 시 가장 흔히 겪는 함정 중 하나입니다.

- **플래그 생성 시 삭제 날짜를 설정하세요.** Release toggle은 최대 2주의 수명을 가져야 합니다. 플래그를 만들 때 이를 정리하기 위한 티켓도 함께 생성하세요.
- **플래그 정리를 일급 과제로 대우하세요.** 기능이 완전히 롤아웃되면 동일한 스프린트 내에서 플래그와 죽은 코드 경로를 제거해야 합니다.
- **유형별로 플래그를 태깅하세요.** 플래그 관리 플랫폼에서 플래그를 분류하여 오래된 플래그를 쉽게 식별할 수 있어야 합니다.
- **오래된 플래그에 대한 자동 알림을 추가하세요.** LaunchDarkly나 Unleash는 N일 동안 변경되지 않은 플래그에 대해 알림을 보내는 기능을 지원합니다.

> ⚠️
> 플래그 남용(Flag Sprawl)의 기술 부채
> 2012년 Knight Capital Group이 4억 4천만 달러의 거래 손실을 본 원인 중 하나는 오래된 죽은 코드를 트리거하는 휴면 상태의 feature flag가 다시 활성화되었기 때문입니다. 플래그 남용은 실질적인 운영 리스크입니다. 정리되지 않은 모든 플래그는 코드 복잡성을 증가시키고 잠재적인 장애 요인이 됩니다.

## Ops Toggles: 운영용 서킷 브레이커 (The Operational Circuit Breaker)

Ops toggles는 임시 방편이 아닌 **영구적인 인프라**이기 때문에 특별한 주의가 필요합니다. Ops toggle을 사용하면 SRE가 코드 배포 없이 수 초 만에 위험한 하위 시스템을 중단시킬 수 있습니다. 예시:

- GPU 추론 지연 시간이 급증할 때 ML inference 엔드포인트를 끄고 규칙 기반 스코어링으로 대체
- 서비스 제공업체 장애 시 비용이 많이 드는 서드파티 데이터 보강(enrichment) API 호출 중단
- 데이터베이스 장애 시 실시간 개인화 기능을 끄고 캐시된 추천 정보 제공
- 데이터베이스 경합을 일으키는 백그라운드 작업 중단

이것들이 바로 **단계적 기능 저하(graceful degradation)**를 위한 레버입니다. 잘 운영되는 시스템에서는 SRE가 운영 대시보드에서 30초 이내에 이러한 플래그를 조작하여 장애를 격리할 수 있습니다. 배포도, 롤백도, PR 머지를 위해 자고 있는 엔지니어를 깨울 필요도 없습니다.

> 💡
> 인터뷰 팁
> Feature flags는 맥가이버 칼과 같습니다. 용도에 맞게 사용하세요. 면접에서는 플래그 유형을 구분하여 답변하세요. Release toggle은 임시 장치이고, ops toggle은 영구적인 인프라입니다. 사용자에게 UI를 노출하지 않고 운영 시스템을 테스트하는 **다크 런치(dark launches)**를 논할 때, feature flags가 그 핵심 매커니즘임을 언급하세요. 그리고 항상 **플래그 부채** 문제를 언급하세요. 이는 즉각적인 이점뿐만 아니라 장기적인 운영 비용까지 고려하고 있음을 보여줍니다.
