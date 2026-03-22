# Canary Release

> 원문: https://www.sysdesai.com/learn/deployment-operations/canary-release

---

## Canary Release란 무엇인가?

**Canary release**는 새로운 버전으로 **운영 트래픽의 소량(낮은 백분율)**만을 점진적으로 유도하고, 대다수의 사용자는 안정적인 기존 버전을 계속 사용하게 하는 방식입니다. 이 이름은 광부들이 가스 누출을 미리 감지하기 위해 탄광에 카나리아를 데려갔던 것에서 유래했습니다. 배포 측면에서 보면, 새 버전에 실제 트래픽이 유입되었을 때 문제가 발생하더라도 아주 적은 수의 사용자만 영향을 받으며 시스템이 자동으로 롤백(rollback)될 수 있습니다.

이진적인(all-or-nothing) 전환 방식인 blue-green과 달리, canary는 1% → 5% → 25% → 50% → 100%와 같이 **점진적인 트래픽 이동(progressive traffic shift)**을 수행합니다. 각 단계에서 지표(metrics)를 평가하여 기준 내에 있으면 배포를 진행하고, 지표가 나빠지면 카나리아를 즉시 중단하고 모든 트래픽을 기존 버전으로 되돌립니다.

5% 트래픽의 Canary: 지표에 따라 자동 승급(promotion) 또는 롤백이 결정됩니다.

## 트래픽 분할 전략

트래픽의 일부를 카나리아로 유도하는 방법에는 여러 가지가 있습니다:

| 전략 | 작동 방식 | 일관성 | 도구 예시 |
| --- | --- | --- | --- |
| Random weight | 로드 밸런서가 요청의 X%를 카나리아로 보냄 | 없음 — 동일 사용자가 두 버전을 모두 만날 수 있음 | AWS ALB 가중치 기반 타겟 그룹, Nginx upstream |
| User cohort (sticky) | 사용자 ID를 해싱하여 동일 사용자를 항상 동일 버전으로 유도 | 높음 — 일관된 사용자 경험 제공 | Istio VirtualService, Envoy |
| Header-based | 특정 헤더가 있는 요청만 카나리아로 유도 | 수동 — 특정 헤더를 가진 사용자나 테스터만 해당 | Nginx, API Gateway, Feature flag |
| Geography / segment | 특정 지역이나 세그먼트의 사용자만 유도 | 세그먼트 내에서 높음 | CloudFront, Akamai, Istio |

> 💡
> Canary를 위한 Sticky Sessions
> 사용자 대면 기능의 경우, 사용자 ID에 일관된 해싱(consistent hashing)을 적용하여 카나리아 기간 동안 동일한 사용자가 항상 동일한 버전을 만나도록 하세요. 한 세션 내에서 버전이 섞이면 사용자 경험(UX)이 혼란스러워지고 버그 진단이 어려워집니다.

## 자동 승급 및 롤백 (Automated Promotion and Rollback)

최신 카나리아 도구의 강점은 **자동화된 지표 기반 승급(metric-driven promotion)**에 있습니다. 성공 기준을 미리 정의하고, 동일한 시간대의 기존 안정 버전 트래픽 지표와 카나리아 지표를 비교합니다. **Flagger** (Kubernetes용)나 **AWS CodeDeploy(CloudWatch 알람 연동)** 같은 도구가 이를 자동으로 수행합니다.

- **에러율(Error rate):** 카나리아의 HTTP 5xx 에러율이 기존 버전보다 1% 이상 높지 않아야 함
- **지연 시간(Latency):** 카나리아의 p99 지연 시간이 기존 버전보다 50ms 이상 높지 않아야 함
- **비즈니스 지표:** 카나리아의 결제 전환율이 기준치 대비 2% 이상 떨어지지 않아야 함
- **커스텀 지표:** Datadog이나 Prometheus 같은 관측 도구(observability stack)의 모든 지표를 게이트(gate)로 사용할 수 있음

yaml

```
# Flagger Canary 리소스 (Kubernetes / Istio)
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: payment-service
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  progressDeadlineSeconds: 3600
  service:
    port: 8080
  analysis:
    interval: 2m          # 2분마다 평가
    threshold: 5          # 롤백 전 최대 5회 체크 실패 허용
    maxWeight: 50         # 카나리아 트래픽이 50%를 넘지 않도록 설정
    stepWeight: 10        # 주기마다 10%씩 증가
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99         # 99% 이상이어야 함
        interval: 1m
      - name: request-duration
        thresholdRange:
          max: 500        # p99가 500ms 이하여야 함
        interval: 1m
```

## Canary vs A/B Testing

Canary release와 A/B 테스팅은 서로 다른 버전으로 트래픽을 유도한다는 점에서는 비슷해 보이지만 목적이 다릅니다:

| 항목 | Canary Release | A/B Testing |
| --- | --- | --- |
| 목표 | 새 코드의 신뢰성/안정성 검증 | 사용자 행동 및 비즈니스 지표 측정 |
| 결정 기준 | 기술적 지표 (에러, 지연 시간) | 비즈니스 지표 (전환율, 참여도) |
| 소요 기간 | 수 시간 ~ 수 일 | 수 일 ~ 수 주 |
| 트래픽 분할 | 초기에는 소량 (1-5%) | 통계적 유의성을 위해 보통 50/50 |
| 롤백 트리거 | 자동 (지표 임계치 도달 시) | 수동 (비즈니스적 결정) |
| 담당 팀 | 엔지니어링 / SRE | 프로덕트 / 데이터 사이언스 |

실제로는 팀들이 이 두 가지를 **결합**하여 사용하기도 합니다. 먼저 카나리아를 통해 기술적 지표를 검증하고, 카나리아 트래픽이 100%가 된 후에 별도의 A/B 실험을 통해 새 기능이 비즈니스에 미치는 영향을 측정합니다.

## Kubernetes 구현

Kubernetes에서 서비스 메시 없이 순수하게 카나리아를 구현하려면, 서로 다른 복제본 수(replica counts)를 가진 두 개의 `Deployment` 객체와 하나의 `Service`를 사용합니다. 기존 버전이 9개, 카나리아 버전이 1개의 복제본을 가진다면 대략 10%의 트래픽이 카나리아로 유입됩니다. 하지만 이 방식은 트래픽 비율을 `1/전체 복제본 수` 단위로만 조절할 수 있다는 한계가 있습니다.

**Istio**나 **Linkerd** 같은 서비스 메시를 사용하면 복제본 수와 관계없이 서비스 메시 계층에서 정확한 백분율 기반의 라우팅이 가능합니다. 예를 들어 카나리아 복제본이 1개뿐이라도 정확히 5%의 트래픽만 받게 하고, 나머지 95%는 기존 버전의 19개 복제본이 받도록 설정할 수 있습니다. 이것이 운영 환경에서 선호되는 방식입니다.

> ⚠️
> Canary 및 데이터베이스 마이그레이션
> Blue-green과 마찬가지로 canary 배포도 버전 간에 데이터베이스를 공유합니다. 카나리아 기간 동안 v1.0과 v2.0이 동시에 동일한 데이터베이스에 기록을 남깁니다. 따라서 모든 스키마 변경은 v1.0과 완벽하게 하위 호환되어야 합니다. expand-contract 패턴을 사용하고, 카나리아가 진행 중일 때는 절대 컬럼을 삭제하거나 타입을 변경하지 마세요.

> 💡
> 인터뷰 팁
> '어떻게 안전하게 배포할 것인가?'라는 질문에 canary는 아주 좋은 답변입니다. 단순히 '트래픽을 점진적으로 늘린다'고만 하지 말고, 지표 기반의 승급 게이트(promotion gates)를 설명하여 깊이를 보여주세요. 절대적인 임계치뿐만 아니라 기존 버전(baseline)과의 비교가 중요하다는 점, 그리고 Kubernetes에서는 Flagger나 Argo Rollouts가 이를 자동화한다는 점을 언급하세요. 추가 점수를 얻으려면 canary와 feature flags가 서로 다른 문제를 해결하며 종종 함께 사용된다는 점도 덧붙이세요.
