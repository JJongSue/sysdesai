# Rolling Update

> 원문: https://www.sysdesai.com/learn/deployment-operations/rolling-update

---

## Rolling Update란 무엇인가?

**Rolling update**는 이전 버전의 인스턴스를 새로운 버전으로 **한 번에 한 배치(batch)씩** 점진적으로 교체하는 방식입니다. 배포가 진행되는 동안 서비스가 완전히 중단되는 시점은 없으며, 항상 이전 버전이나 새 버전 중 일부 인스턴스가 트래픽을 처리하고 있습니다. 이는 blue-green 방식처럼 인프라 비용을 두 배로 쓰지 않고도 제로 다운타임을 달성할 수 있게 해줍니다.

Rolling update는 **Kubernetes의 기본 배포 전략**(`Deployment`의 기본 전략이 `RollingUpdate`)입니다. 또한 AWS Auto Scaling Groups, Elastic Beanstalk 및 대부분의 PaaS 플랫폼에서도 네이티브하게 지원됩니다. 백엔드나 플랫폼 엔지니어 인터뷰를 준비한다면 rolling update의 한계점까지 포함하여 깊이 있게 이해하는 것이 필수적입니다.

배치 크기 2의 Rolling update: 이전 pod들이 점진적으로 교체되며, 각 배치마다 헬스 체크(health checks)를 거칩니다.

## 주요 파라미터: maxSurge와 maxUnavailable

Kubernetes의 rolling update 전략은 두 가지 파라미터로 제어됩니다:

| 파라미터 | 정의 | 기본값 | 증가 시 효과 |
| --- | --- | --- | --- |
| `maxUnavailable` | 업데이트 중 사용할 수 없는 상태가 될 수 있는 최대 pod 수 | 25% | 배포 속도 향상, 일시적 처리량 감소 |
| `maxSurge` | 설정된 복제본 수보다 초과하여 생성될 수 있는 최대 pod 수 | 25% | 배포 속도 향상, 일시적 인프라 비용 증가 |

yaml

```
# Kubernetes Deployment의 rolling update 설정 예시
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # 최대 12개까지 pod 허용 (10 + 2)
      maxUnavailable: 0  # 10개 미만으로 줄어들지 않도록 설정
  template:
    spec:
      containers:
        - name: payment-service
          image: payment-service:v2.0
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
```

`maxUnavailable: 0`과 `maxSurge: N`으로 설정하는 것이 운영 환경에서 **가장 안전한 설정**입니다. 전체 처리 용량이 설정값 아래로 절대 떨어지지 않기 때문입니다. 새 pod가 먼저 생성되고(surge 제한까지), **readiness checks**를 통과한 후에만 이전 pod가 종료됩니다. 이를 통해 배포 중 서비스 저하를 방지할 수 있습니다.

## Health Checks의 중요성

제대로 설정된 health checks 없는 rolling update는 위험합니다. Kubernetes는 두 가지 타입의 프로브(probes)를 사용합니다:

- **Readiness probe:** pod가 트래픽을 받기 시작해도 되는지 결정합니다. 이 프로브가 실패하면 Kubernetes는 서비스 엔드포인트에서 해당 pod를 제거(트래픽 전송 중단)하지만, pod를 재시작하지는 않습니다. Rolling update에서 새 pod는 이전 pod가 종료되기 전에 반드시 이 검사를 통과해야 합니다.
- **Liveness probe:** pod가 정상인지 확인하고 재시작이 필요한지 감지합니다. 이 프로브가 실패하면 Kubernetes는 해당 pod를 강제로 종료하고 재시작합니다. 너무 공격적으로 설정하면 부팅 속도가 느린 애플리케이션의 경우 무한 재시작 루프에 빠질 수 있습니다.

> ⚠️
> Readiness Probe 타이밍 갭
> Kubernetes에는 미묘한 레이스 컨디션(race condition)이 있습니다. pod가 readiness probe를 통과하여 엔드포인트에 추가되더라도, 모든 kube-proxies가 iptables 규칙을 업데이트하기까지 약간의 시간이 걸립니다. 이전 pod에 `terminationGracePeriodSeconds`를 최소 30초 이상 설정하여, pod가 종료되기 전에 진행 중인 연결(in-flight connections)들이 안전하게 처리되도록 하세요.

## 하위 호환성 문제 (The Backward Compatibility Problem)

Rolling update가 진행되는 동안 **두 버전의 인스턴스가 동시에 트래픽을 처리**합니다. 이는 rolling update를 blue-green보다 복잡하게 만드는 근본적인 제약 사항입니다. 다음 사항들을 반드시 보장해야 합니다:

- **API 하위 호환성:** v2.0에서 API 응답 포맷을 변경한다면, 배포 중에 클라이언트는 이전 포맷과 새 포맷을 무작위로 받게 될 수 있습니다. API는 반드시 하위 호환이 가능하도록(필드 추가는 허용, 이름 변경이나 삭제는 금지) 설계해야 합니다.
- **메시지 포맷 호환성:** 서비스 간에 큐나 이벤트 스트림으로 통신한다면, v1.0 소비자가 v2.0 생산자가 만든 메시지를 받을 수 있어야 합니다(반대의 경우도 마찬가지). Avro나 Protobuf 같은 스키마 진화(evolution) 패턴을 사용하세요.
- **데이터베이스 호환성:** 배포 중에 적용되는 모든 스키마 변경은 v1.0과 v2.0 모두에서 동시에 작동해야 합니다. 여기에서도 expand-contract 패턴을 사용하세요.

## 롤백 (Rollback)

Rolling update도 롤백을 지원하지만, 롤백 역시 **역방향으로 진행되는 또 다른 rolling update**일 뿐이며 즉각적이지 않습니다. 이미 80%의 pod가 v2.0으로 교체되었다면, 이를 다시 v1.0으로 되돌리는 데에도 비슷한 시간이 소요됩니다. 진정한 의미의 즉각적인 롤백이 필요하다면 blue-green이 답입니다. `kubectl rollout undo deployment/my-app` 명령어로 롤백을 트리거할 수 있습니다.

| 전략 | 롤백 속도 | 추가 비용 오버헤드 | 버전 혼용 윈도우 |
| --- | --- | --- | --- |
| Rolling Update | 느림 (인스턴스 수에 비례) | 없음 — 추가 인프라 불필요 | 있음 — 두 버전이 동시에 가동됨 |
| Blue-Green | 즉시 (수 초) | 2배의 인프라 비용 | 없음 — 깔끔한 전환 |
| Canary | 빠름 (카나리아 비율을 0%로 조정) | 적음 (카나리아용 소량 인프라) | 있음 — 하지만 제어된 비율만큼만 |

> 💡
> 인터뷰 팁
> Rolling update는 대부분의 엔지니어가 처음 접하는 배포 전략입니다(Kubernetes의 기본값이기 때문). 면접에서 숙련도를 보여주려면 즉시 '버전 혼용 윈도우(mixed-version window)' 문제를 언급하세요. "Rolling update 중에는 이전 버전과 새 버전이 동시에 살아있습니다. 이는 모든 API 규약이나 스키마 변경이 반드시 하위 호환되어야 함을 의미하며, 때로는 2단계 마이그레이션이 필요하다는 뜻입니다. 이것이 rolling update를 겉보기보다 어렵게 만드는 핵심 제약 사항입니다."라고 답변하세요.
