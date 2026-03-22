# Sidecar Pattern

> 원문: https://www.sysdesai.com/learn/decomposition-integration/sidecar-pattern

---

## Sidecar Pattern이란 무엇인가?

**Sidecar pattern**은 동일한 배포 단위(일반적으로 Kubernetes Pod) 내에서 기본 애플리케이션 컨테이너와 함께 도우미 프로세스를 배포하는 방식입니다. 사이드카는 메인 컨테이너와 동일한 네트워크 네임스페이스와 파일 시스템을 공유하므로, 메인 애플리케이션이 모르는 사이에 트래픽을 가로채거나 로그 파일을 읽고 설정을 관리할 수 있습니다. 이 이름은 오토바이의 사이드카에서 유래했습니다. 메인 차량에 부착되어 여정을 함께하지만, 서로 다른 목적을 수행한다는 의미를 담고 있습니다.

이 패턴은 **service meshes**(Istio, Linkerd, Consul Connect)의 아키텍처적 기반입니다. 서비스 메쉬에서는 모든 Pod에 Envoy proxy 사이드카가 자동으로 주입(injected)됩니다. 이러한 사이드카들이 모여 모든 east-west traffic(서비스 간 호출)을 처리하는 **data plane**을 형성합니다.

## Sidecar 아키텍처

사이드카와 메인 컨테이너는 네트워크 네임스페이스를 공유합니다. 모든 인바운드 및 아웃바운드 트래픽은 iptables 규칙을 통해 사이드카 프록시를 통과합니다.

## 일반적인 Sidecar 활용 사례

| 활용 사례 | 사이드카의 역할 | 예시 |
| --- | --- | --- |
| Proxy / Service Mesh | 모든 인바운드/아웃바운드 트래픽을 가로채서 mTLS, retries, circuit breaking 처리 | Envoy (Istio), Linkerd proxy |
| Logging | 공유 볼륨의 로그 파일을 읽어(tail) 중앙 저장소로 전송 | Fluentd, Filebeat, Fluent Bit |
| Metrics Collection | 앱 메트릭을 수집하여 Prometheus 형식으로 노출 | Prometheus exporter sidecars |
| Config / Secret Sync | 설정 저장소를 감시하고 업데이트 내용을 공유 볼륨에 기록 | Vault Agent, AWS Secrets Manager sidecar |
| Security (mTLS) | TLS 인증서를 관리하고 모든 서비스 간 트래픽을 암호화 | Istio's Pilot-agent |
| Ambassador | 메인 앱을 대신해 아웃바운드 호출 처리 (다음 레슨 참고) | Envoy, custom HTTP proxies |

## Sidecar 트래픽 가로채기(Interception)의 작동 원리

Istio가 설치된 Kubernetes에서는 네임스페이스에 라벨(`istio-injection=enabled`)을 지정하면 사이드카가 자동으로 주입됩니다. 메인 앱이 실행되기 전에 **init container**가 실행되어 모든 인바운드 트래픽(사이드카 자체 트래픽 제외)을 Envoy 프록시 포트(15006)로, 모든 아웃바운드 트래픽을 Envoy의 아웃바운드 포트(15001)로 리디렉션하는 **iptables rules**를 설치합니다. 메인 애플리케이션은 이를 전혀 인지하지 못하며, localhost나 원격 서비스와 직접 통신하고 있다고 생각합니다.

```yaml
# Kubernetes Pod spec: Istio가 이 사이드카를 자동으로 주입함
# (직접 작성하지 않아도 mutating webhook이 처리함)
apiVersion: v1
kind: Pod
metadata:
  name: order-service
  annotations:
    sidecar.istio.io/inject: "true"
spec:
  containers:
    - name: order-service          # 메인 애플리케이션
      image: mycompany/orders:1.4
      ports:
        - containerPort: 8080
    # Istio가 자동으로 주입하는 내용:
    - name: istio-proxy            # Envoy sidecar
      image: docker.io/istio/proxyv2:1.18
      ports:
        - containerPort: 15090     # Prometheus metrics
        - containerPort: 15020     # Istio health check
```

## Sidecar Pattern의 장점

- **언어에 구애받지 않음 (Language agnostic)** — 사이드카는 별도의 프로세스이므로, Java 서비스와 Go 서비스가 동일한 동작을 하는 동일한 Envoy 사이드카를 공유할 수 있습니다.
- **애플리케이션 변경 불필요** — 기존 서비스들은 코드 변경 없이도 observability, mTLS, retries 기능을 얻을 수 있습니다.
- **독립적인 생명주기 (Independent lifecycle)** — 사이드카는 메인 애플리케이션과 별개로 업그레이드될 수 있습니다.
- **일관된 정책 강제 (Uniform policy enforcement)** — 보안 정책, rate limits, 추적(tracing) 등이 제어 평면(control plane)에 의해 모든 서비스에 걸쳐 일관되게 적용됩니다.

## 트레이드오프

> ⚠️
> Sidecar는 지연 시간을 추가합니다
> Envoy 사이드카를 통과하는 모든 요청은 한 번의 hop당(수신 및 발신 모두 포함) 약 1~5ms의 오버헤드를 추가합니다. 대부분의 서비스에서는 무시할 수 있는 수준이지만, 초당 수백만 개의 작은 메시지를 처리하는 초저지연 시스템의 경우 전체 서비스 메쉬의 오버헤드가 받아들여지지 않을 수 있습니다. 도입 전 반드시 측정하십시오.

- **복잡성** — 서비스 메쉬 제어 평면(Istiod) 운영은 상당한 투자입니다. 간단한 로깅 사이드카부터 시작하여 점진적으로 전체 메쉬로 확장하십시오.
- **리소스 오버헤드** — 각 Envoy 사이드카는 약 50MB의 RAM과 일정 수준의 CPU를 소비합니다. 500개의 Pod가 있는 클러스터라면 사이드카용으로만 25GB의 RAM이 필요합니다.
- **디버깅의 어려움** — 요청이 실패했을 때 결함이 애플리케이션에 있는지 사이드카 프록시에 있는지 명확하지 않을 수 있습니다.

> 💡
> 면접 팁 (Interview Tip)
> Sidecar pattern은 observability('200개 서비스에서 어떻게 일관된 로깅을 구현합니까?'), 보안('서비스 코드 변경 없이 어떻게 mTLS를 강제합니까?'), 인프라('서비스 메쉬란 무엇입니까?')에 관한 면접 질문에서 등장합니다. 전달해야 할 핵심 통찰은 다음과 같습니다: 사이드카는 인프라 관련 관심사를 애플리케이션 코드에서 외부화함으로써, 모든 서비스가 자동으로 프로덕션 급의 observability와 보안을 갖추는 polyglot 아키텍처를 가능하게 합니다.
