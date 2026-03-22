# Valet Key Pattern

> 원문: https://www.sysdesai.com/learn/security-identity-patterns/valet-key

---

## Valet Key 패턴이란?

발렛 파킹(valet parking) 서비스를 상상해 보세요. 여러분은 보안 요원에게 시동을 걸고 운전석 문을 열 수만 있는 **특수 열차(special key)**를 건넵니다. 이 키로는 귀중품이 들어 있는 글로브 박스나 트렁크를 열 수 없습니다. **Valet Key** 패턴은 이와 동일한 개념을 분산 시스템에 적용한 것입니다. 대규모 데이터 전송을 애플리케이션 서버(병목 지점)를 거치게 하는 대신, 클라이언트에게 저장소의 특정 리소스에 직접 접근할 수 있는 **일시적이고 범위가 제한된 자격 증명(scoped credential)**을 발행합니다.

가장 일반적인 구현 사례로는 **AWS S3 Pre-Signed URLs**, **Azure Blob Storage Shared Access Signature (SAS) tokens**, **Google Cloud Storage signed URLs**, 그리고 **AWS STS temporary credentials**가 있습니다. 이러한 자격 증명은 시간과 범위가 제한되어 있으며, 영구적인 권한을 부여하지 않습니다.

## 왜 Valet Key 패턴을 사용하는가?

Valet Key 패턴이 없다면 클라이언트는 애플리케이션 서버를 통해 대용량 파일을 업로드하거나 다운로드해야 합니다. 이는 **대역폭 병목 현상(bandwidth bottleneck)**을 유발하고, **서버의 CPU/메모리 부하를 증가**시키며, 애플리케이션 서버가 장기 저장소 자격 증명을 보유하게 되어 **리스크가 집중**됩니다. Valet Key 패턴은 이러한 프록시(proxy) 과정을 제거합니다.

| 항목 | Valet Key 미사용 (Proxy) | Valet Key 사용 (Direct) |
| --- | --- | --- |
| 데이터 경로 | 클라이언트 → 앱 서버 → 저장소 | 클라이언트 → 저장소 (직접 전송) |
| 서버 부하 | 앱 서버가 모든 I/O를 처리함 | 앱 서버는 토큰만 발행함 |
| 대역폭 비용 | 이중 데이터 전송 비용(egress) (저장소 → 앱 → 클라이언트) | 단일 데이터 전송 비용 (저장소 → 클라이언트) |
| 자격 증명 노출 | 앱 서버가 장기 저장소 키를 보유함 | 클라이언트는 수명이 짧고 범위가 제한된 토큰만 받음 |
| 확장성 | 앱 서버가 병목 지점이 됨 | 저장소가 독립적으로 확장됨 |

## Sequence: S3 Pre-Signed Upload URL
앱 서버는 메타데이터(metadata)만 처리하며, 모든 파일 I/O는 클라이언트와 S3 사이에서 직접 이루어집니다.

## 범위 지정 및 보안 제어 (Scoping and Security Controls)

Valet Key는 **최소한의 범위(minimally scoped)**로 지정되어야 합니다. 즉, 필요한 만큼의 접근 권한만 부여하고 그 이상은 허용하지 않아야 합니다. Valet key를 발행할 때 다음 제어 사항들을 적용해야 합니다:

- **Short TTL**: 수 분 내에 만료되도록 설정합니다 (업로드는 5~30분, 민감한 콘텐츠 다운로드는 수 초). 만료되지 않는 URL은 절대 발행하지 마세요.
- **Single resource scope**: URL/토큰은 전체 버킷이 아닌 특정 객체 키(object key) 하나만을 대상으로 해야 합니다.
- **HTTP method restriction**: PUT용 pre-signed URL은 GET에 사용할 수 없으며 그 반대도 마찬가지입니다.
- **IP allowlisting**: 지원되는 경우, 토큰 사용을 요청한 클라이언트의 IP 범위로 제한합니다.
- **Content-type enforcement**: S3 pre-signed URL 조건을 사용하여 저장소 레벨에서 예상되는 콘텐츠 타입과 최대 파일 크기를 강제합니다.
- **Audit trail**: 누가 어떤 리소스를 요청했는지 등 모든 토큰 발행 이력을 애플리케이션에 기록합니다.

> ⚠️
> Pre-Signed URL 유출 주의
> Pre-signed URL은 bearer 자격 증명입니다. 즉, 이를 가진 사람은 만료 전까지 누구나 사용할 수 있습니다. 애플리케이션 로그에 전체 URL을 기록하거나, 암호화되지 않은 채널로 전송하거나, 클라이언트 측 코드에 포함하지 마세요. 수명이 짧은 비밀번호처럼 취급해야 합니다.

## Real-World Implementations

| 플랫폼 | 매커니즘 | 주요 특징 |
| --- | --- | --- |
| AWS S3 | Pre-Signed URLs (GET/PUT) | HMAC 서명된 쿼리 파라미터, 설정 가능한 TTL, 조건 키 |
| Azure Blob Storage | Shared Access Signature (SAS) | 서비스, 컨테이너, 또는 계정 레벨; IP 제한, 권한 플래그 |
| Google Cloud Storage | Signed URLs (V4 signing) | 서비스 계정 서명, URL 범위 권한 |
| AWS STS | Assume Role / GetFederationToken | 정책 경계와 세션 태그가 있는 임시 IAM 자격 증명 |

애플리케이션 서버는 오직 제어 평면(control plane, 토큰 발행)에만 관여하며, 데이터 평면(data plane, I/O)에는 참여하지 않습니다.

> 💡
> 인터뷰 팁
> Valet Key 패턴은 '시스템에서 대용량 파일 업로드를 어떻게 처리할 것인가?'라는 질문에 대한 완벽한 답변입니다. 이는 성능(프록시 병목 없음)과 보안(수명이 짧고 범위가 제한된 토큰)을 모두 이해하고 있음을 보여줍니다. 답변할 때 반드시 TTL, 범위 제한, 그리고 업로드 후 후속 처리를 트리거하기 위한 알림(notify-on-complete) 패턴을 언급하세요. 추가로 S3의 버킷 정책을 통해 업로드 자격 증명과는 별개로 서버 측 암호화(server-side encryption at rest)를 강제할 수 있다는 점도 언급하면 좋습니다.
