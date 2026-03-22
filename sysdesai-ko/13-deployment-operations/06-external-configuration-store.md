# External Configuration Store

> 원문: https://www.sysdesai.com/learn/deployment-operations/external-configuration-store

---

## 왜 설정을 외부화해야 하는가?

Twelve-Factor App 방법론의 세 번째 원칙은 다음과 같습니다: **설정을 환경(environment)에 저장하라**. 환경마다 달라지는 설정값(데이터베이스 연결 문자열, API 키, 기능 임계치, 서드파티 서비스 URL 등)은 애플리케이션 코드나 컨테이너 이미지에 포함되어서는 안 됩니다. 설정이 코드에 포함되면 환경마다 별도의 빌드가 필요하게 되며, 이는 immutable infrastructure의 장점을 퇴색시킵니다.

Twelve-Factor의 근거 외에도 설정을 중앙 집중화해야 하는 운영상의 이유들이 있습니다: **감사 가능성(auditability)** (누가 언제 어떤 설정을 바꿨는가?), **핫 리로딩(hot reloading)** (재시작 없이 설정 변경), **접근 제어(access control)** (모든 엔지니어가 데이터베이스 비밀번호를 볼 필요는 없음), 그리고 **일관성(consistency)** (1,000개의 인스턴스에 걸쳐 단일 소스 오브 트루스 유지).

## 설정 계층 (Configuration Tiers)

모든 설정이 다 같은 것은 아닙니다. 다음 세 가지 계층으로 나누어 생각하면 유용합니다:

| 계층 | 유형 | 예시 | 저장 위치 |
| --- | --- | --- | --- |
| 비민감 설정 | 튜닝 파라미터, URL, 타임아웃 | DB_HOST, CACHE_TTL, API_URL | 환경 변수, ConfigMap, Parameter Store |
| 비밀 정보(Secrets) | 자격 증명, 개인 키, 토큰 | DB_PASSWORD, API_SECRET_KEY | AWS Secrets Manager, HashiCorp Vault, 암호화된 Kubernetes Secrets |
| 기능 설정 | 런타임 동작 플래그, 임계치 | MAX_RETRY_COUNT, CIRCUIT_BREAKER_THRESHOLD | 핫 리로딩을 지원하는 외부 설정 저장소 |

## 대중적인 외부 설정 저장소

| 서비스 | 유형 | 주요 특징 | 적합한 용도 |
| --- | --- | --- | --- |
| AWS Parameter Store (SSM) | 관리형 클라우드 서비스 | 계층적 경로, IAM 접근 제어, 무료 티어 제공 | AWS 환경에서의 단순 설정 및 가벼운 비밀 정보 |
| AWS Secrets Manager | 관리형 비밀 정보 서비스 | 자동 회전(rotation), 교차 계정 접근, 깊은 AWS 통합 | AWS 환경의 DB 자격 증명, API 키 |
| HashiCorp Consul | 분산 KV + 서비스 메시 | Watch 기반 변경 알림, 헬스 체크, multi-DC 지원 | 온프레미스 + 멀티 클라우드, 서비스 디스커버리 병행 |
| etcd | 분산 KV 저장소 | Watch API, 강력한 일관성(Raft), Kubernetes 내부 사용 | Kubernetes 설정, 강력한 일관성이 필요한 경우 |
| HashiCorp Vault | 비밀 정보 관리 | 동적 비밀 정보, PKI, 리스(lease) 기반 관리, 감사 로그 | 엔터프라이즈 급 비밀 정보 관리, 컴플라이언스 대응 |
| GCP Secret Manager / Azure Key Vault | 관리형 클라우드 비밀 정보 | 네이티브 클라우드 IAM, 버전 관리, 감사 로그 | GCP 또는 Azure 워크로드 |

## Hot Reloading: 재시작 없는 설정 변경

환경 변수와 비교했을 때 외부 설정 저장소의 핵심 장점은 **hot reloading**입니다. 애플리케이션을 재시작하거나 재배포하지 않고도 변경된 설정을 즉시 반영할 수 있습니다. 이는 운영상의 긴급 조치에 필수적입니다. 예를 들어, 장애 상황에서 배포 과정을 거치지 않고 즉시 타임아웃이나 서킷 브레이커(circuit breaker) 임계치를 조정할 수 있습니다.

Hot reload는 보통 두 가지 패턴 중 하나로 구현됩니다:

- **Polling (폴링):** 애플리케이션이 주기적으로(예: 30초마다) 저장소에서 설정을 가져옵니다. 구현이 간단하지만, 폴링 주기에 따라 약간의 지연(staleness)이 발생합니다.
- **Watch / Push (감시/푸시):** 저장소가 롱 폴링(long-polling), SSE, 또는 gRPC 스트림을 통해 변경 사항을 애플리케이션에 푸시합니다. Consul의 watch API, etcd의 Watch API, AWS AppConfig의 배포 전략 등이 이를 지원합니다. 거의 즉각적인 전파가 가능합니다.

typescript

```
// hot reload를 지원하는 AWS AppConfig 사용 예시 (Node.js)
import { AppConfigDataClient, StartConfigurationSessionCommand, GetLatestConfigurationCommand } from '@aws-sdk/client-appconfigdata';

class ConfigManager {
  private client = new AppConfigDataClient({ region: 'us-east-1' });
  private sessionToken: string | undefined;
  private config: Record<string, unknown> = {};

  async init() {
    const session = await this.client.send(new StartConfigurationSessionCommand({
      ApplicationIdentifier: 'my-app',
      EnvironmentIdentifier: process.env.ENV,
      ConfigurationProfileIdentifier: 'app-config',
    }));
    this.sessionToken = session.InitialConfigurationToken;
    await this.poll(); // 최초 조회
    setInterval(() => this.poll(), 30_000); // 30초마다 폴링
  }

  private async poll() {
    const result = await this.client.send(new GetLatestConfigurationCommand({
      ConfigurationToken: this.sessionToken,
    }));
    this.sessionToken = result.NextPollConfigurationToken;
    if (result.Configuration?.length) {
      this.config = JSON.parse(Buffer.from(result.Configuration).toString());
    }
  }

  get<T>(key: string, defaultVal: T): T {
    return (this.config[key] as T) ?? defaultVal;
  }
}

const cfg = new ConfigManager();
await cfg.init();

// 요청 핸들러에서 — 재시작 없이 현재 설정값을 읽음
const timeout = cfg.get<number>('upstream_timeout_ms', 5000);
```

## Secrets 관리 모범 사례

비밀 정보(Secrets)는 일반 설정값보다 더 엄격한 제어가 필요합니다:

- **비밀 정보를 절대 버전 관리 시스템에 커밋하지 마세요.** 나중에 삭제하더라도 git 히스토리에 남게 됩니다. 실수로 인한 커밋을 방지하기 위해 `git-secrets`나 pre-commit hooks를 사용하세요.
- **가능한 경우 동적 비밀 정보(dynamic secrets)를 사용하세요.** AWS Secrets Manager와 Vault는 요청 시 짧은 수명의 DB 자격 증명을 생성할 수 있습니다. 애플리케이션은 1시간 동안만 유효한 권한을 부여받으므로, 유출되더라도 피해가 제한적입니다.
- **정기적으로 비밀 정보를 회전(rotate)시키세요.** 자동화된 회전(AWS Secrets Manager의 Lambda 기반 자동 회전 등)은 수동 작업에 따른 리스크를 제거합니다.
- **IAM 및 정책을 통해 접근 범위를 제한하세요.** 각 인스턴스는 오직 자신에게 필요한 비밀 정보만 읽을 수 있어야 합니다(최소 권한 원칙).
- **비밀 정보 접근 이력을 감사(audit)하세요.** AWS CloudTrail이나 Vault 감사 로그는 모든 읽기 작업을 기록합니다. 자격 증명이 유출되었을 때 누가 언제 접근했는지 파악할 수 있습니다.

> ⚠️
> Kubernetes Secrets는 기본적으로 암호화되지 않습니다
> Kubernetes Secrets는 단순히 base64-encoded된 것이지 암호화된 것이 아닙니다. etcd에 접근할 수 있는 사람은 누구나 읽을 수 있습니다. 운영 환경에서는 반드시 etcd의 저장 데이터 암호화(encryption at rest)를 활성화하고 KMS 제공자를 사용하세요. 더 좋은 방법은 외부 비밀 정보 운영 도구(External Secrets Operator + AWS Secrets Manager 조합 등)를 사용하여 비밀 정보가 etcd에 아예 저장되지 않게 하는 것입니다.

## Kubernetes에서의 설정: ConfigMaps와 그 너머

Kubernetes는 비민감 설정을 위한 `ConfigMap`과 자격 증명을 위한 `Secret`을 제공합니다. 둘 다 파일로 마운트하거나 환경 변수로 주입할 수 있습니다. 하지만 환경 변수로 주입된 ConfigMap과 Secret은 자동으로 hot-reload되지 않습니다. 환경 변수를 변경하려면 pod를 재시작해야 합니다.

Kubernetes에서 hot-reload를 구현하려면 다음 방식 중 하나를 사용하세요: 설정을 파일로 마운트하고(ConfigMap 업데이트 시 약 1분 내에 자동 업데이트됨) 애플리케이션이 파일 변경을 감시하게 하거나, **Reloader**와 같은 오퍼레이터를 사용하여 참조된 ConfigMap/Secret이 변경될 때 자동으로 배포를 롤링(rolling)하게 만드세요.

> 💡
> 인터뷰 팁
> 외부 설정 저장소를 논할 때 일반 설정(비민감, 로그 기록 가능, 환경별 오버라이드와 함께 버전 관리 가능)과 비밀 정보(로그 및 커밋 금지, 최소 접근 권한)를 명확히 구분하세요. 데이터베이스를 위한 **동적 비밀 정보(dynamic secrets)** 개념을 언급하면 면접관에게 깊은 인상을 줄 수 있습니다. 그리고 Kubernetes Secrets와 관련하여 'base64는 암호화가 아니다'라는 점을 잊지 말고 지적하세요.
