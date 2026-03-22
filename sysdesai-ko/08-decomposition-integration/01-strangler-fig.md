# Strangler Fig Pattern

> 원문: https://www.sysdesai.com/learn/decomposition-integration/strangler-fig

---

## Strangler Fig Pattern이란 무엇인가?

**Strangler Fig** pattern은 숙주 나무 주변에서 자라나 결국 숙주를 대체하는 열대 나무의 이름을 딴 패턴으로, 레거시 monolith를 microservices로 점진적으로 마이그레이션하는 가장 안전하고 검증된 접근 방식입니다. 위험한 "big bang" 재작성(rewrite)을 시도하는 대신, monolith가 계속 작동하는 동안 트래픽의 작은 일부분을 새로운 서비스로 라우팅합니다. 시간이 지나면서 새로운 서비스가 점점 더 많은 책임을 흡수하고, 마침내 기존 시스템을 중단(decommission)할 수 있게 됩니다.

Martin Fowler가 2004년에 이 패턴을 대중화했습니다. Amazon, Netflix, LinkedIn 모두 초기 monolith를 분해할 때 이 패턴의 변형을 사용했습니다. 면접관이 *"사이트 중단 없이 10년 된 Rails monolith를 microservices로 어떻게 마이그레이션하시겠습니까?"*라고 묻는다면 바로 이 답변이 정답입니다.

## 3단계 단계 (The Three Phases)

1. **Transform** — 새로운 서비스를 병렬로 구축합니다. 대상이 되는 기능에 대해 monolith와 동일한 외부 API contract를 유지해야 합니다.
2. **Co-exist** — 두 시스템 앞에 facade(reverse proxy 또는 API gateway)를 배치합니다. 트래픽의 하위 집합(feature flag, cookie, user segment 기준)을 새로운 서비스로 라우팅합니다. monolith는 나머지 모든 것을 계속 처리합니다.
3. **Eliminate** — 새로운 서비스의 안정성이 입증되면 트래픽의 100%를 전환하고 레거시 코드 경로를 제거합니다.

## Facade 구현 옵션

facade는 핵심적인 부분입니다. 트래픽 유형에 따라 여러 가지 구현 옵션이 있습니다:

| Facade 유형 | 적합한 용도 | 예시 도구 |
| --- | --- | --- |
| Reverse proxy (HTTP) | REST APIs, 웹 애플리케이션 | Nginx, Envoy, AWS ALB |
| API gateway | auth/rate-limiting이 포함된 공개 API | Kong, AWS API Gateway, Apigee |
| Message router | Event-driven / async 시스템 | Kafka topic routing, AWS EventBridge |
| DNS / feature flags | A/B testing, 점진적 배포 | LaunchDarkly, AWS CloudFront |

## 데이터 마이그레이션 전략 (Data Migration Strategy)

Strangler Fig 마이그레이션에서 가장 어려운 부분은 **data**입니다. monolith와 새로운 서비스가 장기적으로 데이터베이스를 공유해서는 안 되지만, 전환기 동안에는 일관성을 유지해야 합니다. 일반적인 접근 방식은 **sync bridge**를 활용한 **Parallel Run**입니다.

1. 새로운 서비스는 자체 데이터베이스를 가집니다(처음에는 비어 있거나 스냅샷으로 초기화됨).
2. **sync bridge**(Debezium을 통한 CDC — Change Data Capture, 또는 dual-write logic)가 마이그레이션 기간 동안 두 데이터베이스의 동기화를 유지합니다.
3. 트래픽이 새로운 서비스로 완전히 전환되면 sync bridge를 제거하고 레거시 테이블을 아카이브합니다.

> ⚠️
> Shared Databases를 피하십시오
> 새로운 microservice가 monolith의 데이터베이스를 직접 읽고 쓰게 하는 것은 흔한 anti-pattern입니다. 이는 추출의 목적을 무색하게 만드는 tight coupling을 유발합니다. 항상 첫날부터 깨끗한 데이터베이스 경계(boundary)를 목표로 하십시오.

## 안전한 배포를 위한 Feature Toggles

Strangler Fig facade를 **feature toggles**(feature flags)와 결합하여 퍼센트, user segment 또는 지역별로 트래픽을 새로운 서비스로 라우팅하십시오. 이를 통해 점진적인 노출이 가능해집니다. 처음에는 사용자의 1%부터 시작하여 error rates와 latency를 모니터링한 다음, 10%, 50%, 100%로 점차 늘려갑니다. 문제가 발생하면 즉시 토글을 뒤집어 monolith로 되돌릴 수 있으며, 별도의 배포는 필요하지 않습니다.

```nginx
# Nginx 예시: /orders는 새로운 서비스로, 나머지는 monolith로 라우팅
upstream monolith {
  server legacy-app:8080;
}

upstream orders_svc {
  server orders-service:8080;
}

server {
  listen 80;

  location /api/orders {
    proxy_pass http://orders_svc;
  }

  location / {
    proxy_pass http://monolith;
  }
}
```

## 복구 전략 (Rollback Strategy)

rollback 계획이 없는 마이그레이션은 도박과 같습니다. 트래픽 슬라이스를 전환하기 전에 rollback 트리거를 정의하십시오: error rate 임계값(예: > 0.5%), p99 latency 증가(예: baseline보다 > 200ms 높음), 또는 당직 엔지니어의 수동 개입 등. facade를 사용하면 단일 설정 변경만으로 트래픽을 monolith로 다시 리디렉션할 수 있어 rollback이 매우 간편합니다.

> 💡
> 면접 팁 (Interview Tip)
> 면접에서 레거시 시스템 마이그레이션이나 아키텍처 변경 시의 리스크 감소에 대한 질문을 받으면 언제든 Strangler Fig pattern을 언급하십시오. 세 가지 핵심을 강조하십시오: 트래픽 컨트롤러로서의 facade, 명시적인 데이터 마이그레이션 전략, 그리고 언제든 가능한 rollback 능력입니다. FAANG 및 fintech 기업의 면접관들은 단순한 기술 지식을 넘어 리스크 관리 능력을 보여주는 이 답변을 매우 선호합니다.

## 실제 사례: Netflix

Netflix는 2008년에서 2012년 사이에 DVD 시대의 monolith를 microservices로 마이그레이션했습니다. 그들은 strangler 접근 방식을 사용했습니다. 각 팀은 공유 API gateway(Zuul) 뒤에서 도메인(스트리밍, 추천, 결제)을 추출했습니다. 마이그레이션이 절정에 달했을 때, Netflix는 수백 개의 A/B 테스트를 동시에 실행하며 트래픽의 하위 집합을 새로운 서비스 버전으로 라우팅하는 동시에 monolith는 나머지 사용자들에게 서비스를 제공했습니다. 이 마이그레이션은 4년이 걸렸습니다. Strangler Fig는 sprint가 아니라 마라톤임을 상기시켜 줍니다.

> 💡
> 가장 위험이 적은 서비스부터 시작하십시오
> 첫 번째 추출 대상으로 다음과 같은 후보를 선택하십시오: 비교적 독립적이고(의존성이 적음), API 표면이 명확하며, 결제와 같은 핵심 경로(critical path)에 있지 않은 기능. 초기의 성공은 신뢰를 구축하고 핵심 도메인을 다루기 전에 마이그레이션 도구들을 다듬는 데 도움이 됩니다.
