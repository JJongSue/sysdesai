# Ambassador Pattern

> 원문: https://www.sysdesai.com/learn/decomposition-integration/ambassador-pattern

---

## Ambassador: 아웃바운드 사이드카의 특화 (Outbound Sidecar Specialization)

**Ambassador pattern**은 Sidecar pattern의 변형으로, 오직 **아웃바운드(outbound)** 연결에만 집중합니다. 일반적인 사이드카가 인바운드와 아웃바운드 문제를 모두 처리하는 반면, 앰배서더는 메인 서비스를 대신하여 외부 서비스와의 통신에 수반되는 모든 복잡한 작업들(retries, timeouts, circuit breaking, connection pooling, service discovery, TLS 등)을 관리하는 아웃바운드 프록시 역할을 합니다.

이 이름은 매우 적절합니다. 외교관인 대사(ambassador)가 외국 정부를 상대할 때 자국을 대표하듯이, 앰배서더 프록시는 외부 세계와 통신할 때 당신의 서비스를 대표합니다. 메인 애플리케이션은 단순히 `localhost:8080/some-service`를 호출하고, 앰배서더가 모든 까다로운 부분들을 처리합니다.

## Ambassador 아키텍처

메인 애플리케이션은 localhost에 있는 앰배서더를 호출합니다. 앰배서더는 애플리케이션을 대신하여 모든 아웃바운드 통신 복잡성을 처리합니다.

## Ambassador가 처리하는 항목들

- **Retries with backoff** — 실패한 요청에 대해 지수 백오프(exponential backoff)를 적용하여 자동으로 재시도합니다. 앱은 별도의 재시도 로직을 구현할 필요가 없습니다.
- **Circuit breaking** — 다운스트림 서비스가 지속적으로 실패할 경우 회로를 열어(open) 장애가 연쇄적으로 확산되는 것을 방지합니다.
- **Connection pooling** — 다운스트림 서비스에 대한 지속적인 연결(persistent connections) 풀을 유지하여, 요청마다 발생하는 TCP 핸드셰이크 오버헤드를 피합니다.
- **Service discovery** — 클러스터의 서비스 레지스트리(Consul, Kubernetes DNS 등)를 사용하여 서비스 이름을 IP 주소로 변환합니다.
- **TLS termination (outbound)** — 아웃바운드 호출을 위한 mTLS 핸드셰이크를 처리합니다. 메인 앱은 로컬에서 일반 HTTP를 보냅니다.
- **Observability** — 대시보드를 위해 업스트림 서비스별 지연 시간(latency), 성공률, 트래픽 양을 기록합니다.

## Ambassador vs Sidecar vs API Gateway

| 패턴 | 방향 | 집중 분야 |
| --- | --- | --- |
| API Gateway | 인바운드 (클라이언트 → 서비스) | 모든 인바운드 외부 트래픽에 대한 횡단 관심사 |
| Sidecar | 인바운드 + 아웃바운드 | 메인 컨테이너 옆의 범용 도우미 |
| Ambassador | 아웃바운드 (서비스 → 의존성) | 아웃바운드 연결성: 외부 서비스에 대한 retries, circuit breaking, TLS |

## 실제 사례: Azure Ambassador Pattern

Microsoft Azure Cloud Design Patterns에서는 Ambassador 패턴을 정식으로 문서화하고 있습니다. 일반적인 Azure 배포 환경에서는 **Dapr (Distributed Application Runtime)** 사이드카를 사용하는데, 여기에는 앰배서더 스타일의 아웃바운드 프록시(`dapr-http://payment-service`)가 포함되어 있습니다. 이는 .NET, Java, Python, Go 서비스 모두에 대해 retries, circuit breaking, mTLS, service discovery를 투명하게 처리해 줍니다.

> ℹ️
> 서비스 메쉬 맥락에서의 Ambassador
> Istio나 Linkerd에서 앰배서더 역할은 Envoy 사이드카의 아웃바운드 리스너(outbound listener)가 수행합니다. '앰배서더'와 '사이드카'의 구분은 개념적인 것입니다. 실제로는 종종 동일한 바이너리(Envoy)가 두 역할을 모두 수행합니다. 중요한 점은 아웃바운드 트래픽 관리가 인바운드 트래픽 관리와는 별개의 관심사임을 이해하는 것입니다.

## Ambassador Pattern을 사용해야 할 때

- 서비스가 많은 외부 또는 제3자 API를 호출하며, 여러 서비스에 걸쳐 로직을 중복시키지 않고 일관된 retry/circuit-breaking 동작을 원할 때.
- Polyglot 환경이라서 모든 언어로 된 서비스에 대해 복원력(resilience) 로직을 공유 라이브러리로 사용할 수 없을 때.
- 점진적으로 서비스 메쉬를 도입하면서 아웃바운드 프록시 기능부터 먼저 시작하고 싶을 때.
- 애플리케이션 코드 변경 없이 아웃바운드 호출에 mTLS를 강제하고 싶을 때.

> 💡
> 면접 팁 (Interview Tip)
> Ambassador 패턴은 Circuit Breaker나 Sidecar만큼 직접적으로 질문을 받지는 않지만, 이를 언급하면 깊이 있는 아키텍처 지식을 갖추고 있음을 보여줄 수 있습니다. 만약 '서비스 B가 불안정할 때 서비스 A가 어떻게 안전하게 B를 호출할 수 있습니까?'라는 질문을 받는다면, 지수 백오프를 이용한 재시도, 서킷 브레이커, 타임아웃을 포함한 완전한 답변을 제시하십시오. 그리고 이러한 기능들이 각 서비스에 코딩되는 대신 앰배서더 프록시로 외부화될 수 있음을 덧붙이십시오.
