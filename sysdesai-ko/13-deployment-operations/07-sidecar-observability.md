# Sidecar for Observability

> 원문: https://www.sysdesai.com/learn/deployment-operations/sidecar-observability

---

## Sidecar 패턴이란?

**Sidecar 패턴**은 동일한 Kubernetes pod 내에 메인 애플리케이션 컨테이너와 함께 실행되는 **보조 프로세스(helper process)**를 배치하는 방식입니다. 사이드카는 메인 컨테이너와 네트워크 네임스페이스(network namespace), 파일 시스템 볼륨, 수명 주기를 공유하지만 별도의 프로세스로 실행됩니다. 이를 통해 애플리케이션 코드를 수정하지 않고도 관측성(observability)과 같은 횡단 관심사(cross-cutting concerns)를 보완할 수 있습니다.

이 이름은 오토바이의 사이드카에서 유래했습니다. 메인 차량 옆에 붙어 함께 이동하면서도 각자의 고유한 목적을 가진 별도의 부착물입니다. 컨테이너 환경에서 pod는 오토바이이고, 사이드카는 여기에 연결된 보조 컨테이너입니다.

Sidecar OTel Collector: 애플리케이션 코드 수정 없이 텔레메트리 데이터를 수집하여 여러 백엔드(backends)로 Fan-Out합니다.

## 관측성의 세 가지 기둥 (The Three Pillars of Observability)

사이드카는 각 서비스를 개별적으로 수정하지 않고도 관측성의 세 가지 기둥을 구현하는 데 사용됩니다:

| 기둥 | 제공 정보 | 사이드카 도구 | 백엔드 |
| --- | --- | --- | --- |
| 로그(Logs) | 전체 맥락과 함께 발생한 사건 | Fluentd, Fluent Bit, Logstash | Elasticsearch, Loki, Splunk |
| 메리틱(Metrics) | 시스템의 시간별 동작 (빈도, 횟수, 기간) | Prometheus exporter, OTel Collector, StatsD | Prometheus, Datadog, CloudWatch |
| 트레이스(Traces) | 서비스 간 요청 흐름 (분산 컨텍스트) | OTel Collector, Jaeger agent, Zipkin | Jaeger, Tempo, AWS X-Ray, Datadog APM |

## Fluent Bit를 활용한 로그 수집

**Fluent Bit**는 사이드카로 흔히 사용되는 가벼운 로그 프로세서입니다. 메인 애플리케이션이 구조화된 JSON 로그를 stdout으로 출력하면, Kubernetes는 이를 노드의 로그 파일로 캡처합니다. Fluent Bit 사이드카(또는 노드 레벨의 DaemonSet)는 이 로그 파일을 읽어 변환(파싱, pod 메타데이터 추가)을 수행한 뒤 중앙 로그 저장소로 전송합니다.

yaml

```
# Fluent Bit 로그 사이드카가 포함된 Kubernetes pod 예시
apiVersion: v1
kind: Pod
spec:
  volumes:
    - name: app-logs
      emptyDir: {}
  containers:
    - name: app
      image: my-service:v2.0
      volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
      # 애플리케이션은 /var/log/app/service.log에 구조화된 JSON을 기록함

    - name: fluent-bit
      image: fluent/fluent-bit:2.2
      volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
          readOnly: true
      env:
        - name: ELASTICSEARCH_HOST
          value: "https://logs.internal.example.com"
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "100m"
          memory: "128Mi"
```

## OpenTelemetry Collector 사이드카

**OpenTelemetry Collector**는 관측성 사이드카의 표준으로 빠르게 자리 잡고 있습니다. 애플리케이션은 OpenTelemetry SDK를 사용하여 자체적으로 측정하고, OTLP 프로토콜을 통해 로컬 사이드카인 OTel Collector로 텔레메트리(트레이스, 메트릭, 로그)를 내보냅니다. 콜렉터는 fan-out 라우팅, 버퍼링, 재시도, 여러 백엔드를 위한 포맷 변환과 같은 복잡한 처리를 담당합니다.

OTel Collector 파이프라인은 세 단계로 구성됩니다:

- **Receivers:** 다양한 포맷(OTLP, Jaeger, Zipkin, Prometheus scrape, StatsD)의 데이터를 수락
- **Processors:** 데이터 변환, 샘플링, 필터링 및 보강 (예: tail-based sampling, 속성 추가, PII scrubbing)
- **Exporters:** 처리된 데이터를 백엔드(Jaeger, Prometheus, Datadog, AWS X-Ray, Splunk)로 전송

yaml

```
# OpenTelemetry Collector 사이드카 설정 예시 (otel-collector-config.yaml)
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  memory_limiter:
    limit_mib: 100

exporters:
  jaeger:
    endpoint: jaeger-collector.tracing.svc.cluster.local:14250
  prometheusremotewrite:
    endpoint: http://prometheus.monitoring.svc.cluster.local/api/v1/write
  logging:
    verbosity: normal

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [jaeger]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheusremotewrite]
```

## 서비스 메시 사이드카: Envoy

**Envoy** 프록시는 Istio 및 기타 서비스 메시에서 사용하는 사이드카입니다. 로그/메트릭 사이드카와 달리 Envoy는 **네트워크 경로(network path)**에 위치합니다. 즉, pod로 들어오고 나가는 모든 트래픽이 Envoy를 거칩니다. 이를 통해 서비스 메시는 애플리케이션 코드 변경 없이도 깊은 가시성과 제어권을 얻습니다:

- **분산 트레이싱:** Envoy는 모든 요청에 대해 자동으로 span을 생성하고 서비스 간에 트레이스 컨텍스트 헤더(B3, W3C TraceContext)를 전파합니다.
- **메트릭:** Envoy는 서비스별, 경로별 상세 L7 지표(요청 수, 성공률, 지연 시간 히스토그램)를 Prometheus로 내보냅니다.
- **액세스 로그:** 업스트림 서비스 메타데이터를 포함한 모든 요청에 대한 구조화된 액세스 로그를 생성합니다.
- **mTLS:** 모든 서비스 간 상호 TLS(mTLS)를 적용합니다 — 애플리케이션은 인증서를 직접 다룰 필요가 없습니다.
- **트래픽 관리:** 메시 레벨에서 설정된 재시도, 타임아웃, 서킷 브레이킹을 수행합니다.

> 💡
> 로그 수집을 위한 DaemonSet vs Sidecar
> 로그 수집에는 두 가지 토폴로지가 있습니다: pod당 Fluent Bit 사이드카를 두는 방식(세밀한 제어 가능, 리소스 오버헤드 큼)과 노드당 하나의 콜렉터를 두는 Fluent Bit DaemonSet 방식(효율적이나 유연성 낮음)입니다. 로그 포맷이 일정한 경우 DaemonSet 패턴이 선호됩니다. pod마다 로그 전송 대상이 다르거나 테넌트(tenant) 간 로그 처리 격리가 필요한 경우 사이드카 주입 방식을 사용하세요.

## 리소스 오버헤드 및 사이드카 주입

사이드카는 공짜가 아닙니다. 각 사이드카 컨테이너는 CPU와 메모리 리소스를 소비합니다. 수천 개의 pod가 있는 클러스터에서 사이드카 오버헤드의 총합은 상당할 수 있습니다. 설계 가이드라인:

- 사이드카에 리소스 request와 limit을 설정하세요. Fluent Bit 사이드카는 보통 50m CPU와 64Mi 메모리가 필요합니다.
- Istio처럼 Kubernetes admission webhooks를 통한 사이드카 주입(injection)을 사용하여 팀들이 매번 수동으로 사이드카 설정을 추가하지 않게 하세요.
- 고밀도 워크로드의 경우 노드 레벨(DaemonSet)로 통합하는 것을 고려하세요.
- Kubernetes 1.29 이상에서는 **네이티브 사이드카 컨테이너**(`restartPolicy: Always`가 설정된 init containers)를 지원합니다. 이는 일반 컨테이너보다 수명 주기 관리가 용이합니다 (메인 컨테이너보다 먼저 시작되고 나중에 종료됨).

> 💡
> 인터뷰 팁
> 시스템 설계 면접에서 관측성을 논할 때, 텔레메트리 수집을 위한 현대적인 접근 방식으로 사이드카/OTel Collector 패턴을 언급하세요. 핵심 포인트: (1) 애플리케이션 코드는 로컬 OTLP 엔드포인트와만 통신하면 되므로 백엔드에 대해 알 필요가 없다. (2) 콜렉터가 fan-out, 샘플링, 포맷 변환을 처리한다. (3) 이를 통해 운영적 관심사(트레이스를 어디로 보낼 것인가?)와 개발적 관심사(코드에 어떻게 측정 코드를 심을 것인가?)를 분리할 수 있다. 또한 Istio와 같은 서비스 메시를 사용하면 애플리케이션 수정 없이도 L7 메트릭과 트레이스를 무료로 얻을 수 있다는 점도 덧붙이세요.
