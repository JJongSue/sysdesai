# Zero Trust Architecture

> 출처: https://www.sysdesai.com/learn/security-auth/zero-trust

---

## 경계 보안(Perimeter Security)의 문제점

전통적인 보안은 **성벽과 해자(castle-and-moat)** 모델에 기반했습니다: 강력한 경계(방화벽, VPN)를 구축하고 내부의 모든 것은 신뢰하는 방식입니다. 하지만 현대의 위협 환경에서 이 모델은 처참하게 실패했습니다. 공격자들은 피싱이나 사회 공학 기법을 통해 사용자 계정을 탈취하여 '성' 안으로 들어오는 정식 자격 증명을 점점 더 많이 얻고 있습니다. 일단 내부에 들어오면, 평평한(flat) 내부 네트워크 덕분에 핵심 시스템에 도달하기 위해 자유롭게 측면 이동(lateral movement)을 할 수 있습니다. 2020년 SolarWinds 침해 사고와 2021년 Colonial Pipeline 랜섬웨어 공격 모두 이 모델의 약점을 이용했습니다.

**Zero Trust Architecture (ZTA)** — 2010년 Forrester Research의 John Kindervag에 의해 대중화되었고 Google (BeyondCorp), Microsoft 등이 채택했으며 미국 정부도 의무화(행정명령 14028)한 방식 — 는 경계 모델을 **'절대 신뢰하지 말고, 항상 검증하라(Never trust, always verify)'**는 철학으로 대체합니다. 네트워크 위치는 어떠한 암묵적 신뢰도 부여하지 않습니다. 모든 요청은 위치와 상관없이 반드시 인증, 인가, 검증을 거쳐야 합니다.

> ℹ️
> Google BeyondCorp: 운영 환경에서의 Zero Trust
> 2011년부터 2017년 사이에 구축된 Google의 BeyondCorp 프로그램은 모든 Google 직원의 접속을 VPN 없이 공개 인터넷으로 옮기고 매 요청마다 신원을 확인하도록 했습니다. 직원들은 VPN을 사용하지 않으며, 크롬북의 장치 인증서와 Google 계정이 모든 요청에서 검증됩니다. 이는 대규모로 구현된 가장 유명한 실제 Zero Trust 사례입니다.

## Zero Trust의 세 가지 기둥

- **명시적 검증(Verify Explicitly)**: 사용 가능한 모든 신호를 사용하여 모든 요청을 인증하고 인가합니다: 사용자 신원(강력한 MFA), 장치 상태(관리되는 장치이며 보안 정책을 준수하는가?), 위치(IP가 의심스럽지 않은가?), 시간(평소와 다른 시간대의 접근인가?) 등을 모두 확인합니다.
- **최소 권한 접근(Use Least Privilege Access)**: 필요한 만큼만, 필요한 시간 동안만 접근 권한을 부여합니다. 권한이 필요한 작업에는 JIT (Just-In-Time) 접근 방식을 사용합니다. 와일드카드 권한보다는 범위가 제한된(scoped) 자격 증명을 선호하며, 접근 권한은 자동으로 만료되도록 합니다.
- **침해 가정(Assume Breach)**: 공격자가 이미 내부에 있다고 가정하고 설계합니다. 네트워크를 세분화(segmentation)합니다. 내부 서비스 간 통신(East-West)을 포함한 모든 트래픽을 암호화합니다. 모든 활동을 기록하고 이상 징후를 탐지합니다. 공격의 폭발 반경(blast radius)을 최소화합니다.

## Microsegmentation

**Microsegmentation**은 평평한 내부 네트워크 문제에 대한 Zero Trust의 해답입니다. 모든 서비스가 '내부니까' 서로 통신할 수 있게 두는 대신, 명확하고 세밀한 네트워크 정책을 정의합니다: `orders-service`는 `inventory-service`와 `orders-db`에 요청을 보낼 수 있지만, `payments-db`나 `auth-service`에는 접근할 수 없습니다. 정책은 네트워크 경계가 아닌 워크로드 수준에서 강제됩니다.

Microsegmentation은 **네트워크 정책**(Kubernetes NetworkPolicy, AWS Security Groups, GCP VPC Firewall Rules) 및/또는 사이드카 프록시 계층에서 mTLS 기반 ID 정책을 강제하는 **Service mesh**(Istio, Linkerd, Consul Connect)를 통해 구현됩니다. Service mesh 방식은 네트워크 토폴로지에 의존하지 않고 같은 호스트에서 실행되는 서비스 간에도 작동하므로 특히 강력합니다.

## 서비스 아이덴티티: SPIFFE 및 SPIRE

Zero Trust 모델에서 서비스는 위조 불가능하고 자동으로 로테이션되는 **암호화된 아이덴티티**가 필요합니다. **SPIFFE (Secure Production Identity Framework For Everyone)**는 CNCF 표준으로, URI 기반의 워크로드 아이덴티티 포맷을 정의합니다: `spiffe://trust-domain/namespace/workload-name`. **SPIRE**는 플랫폼 증명(Kubernetes service account, cloud IAM role 등)을 기반으로 각 워크로드에 짧은 수명의 X.509 SVID(SPIFFE Verifiable Identity Document)를 발급하는 레퍼런스 구현체입니다.

## Zero Trust 점진적 구현하기

Zero Trust는 한 번에 구매할 수 있는 제품이 아니라 여정입니다. 성숙한 조직을 하룻밤 사이에 Zero Trust로 전환할 수는 없습니다. 위험도에 따라 단계적으로 구현하세요:

1. **인벤토리(Inventory)**: 모든 사용자, 장치, 서비스, 데이터를 카탈로그화합니다. 보이지 않는 것은 보호할 수 없습니다.
2. **강력한 아이덴티티**: SSO를 도입하고 모든 사용자, 특히 관리자에게 MFA를 강제합니다. 이것만으로도 대다수의 침해 위험을 제거할 수 있습니다.
3. **장치 신뢰(Device Trust)**: MDM (Intune, Jamf)에 장치를 등록합니다. 모든 접속 요청 시 장치의 보안 준수 여부를 확인합니다.
4. **최소 권한**: 과도하게 권한이 부여된 서비스 계정을 감사하고 축소합니다. 권한 작업에 대해 JIT 접근(예: AWS IAM Roles Anywhere, HashiCorp Boundary)을 구현합니다.
5. **네트워크 세분화**: 보안 그룹과 네트워크 정책을 구현하여 측면 이동을 제한합니다. 가장 민감한 서비스부터 시작하세요.
6. **내부 서비스용 mTLS**: Service mesh를 도입합니다. 모든 내부 트래픽을 암호화하고 인증합니다.
7. **지속적인 모니터링**: SIEM을 구축합니다. 비정상적인 접근 패턴에 대해 알림을 설정합니다. 침해를 가정하고 측면 이동 흔적을 능동적으로 추적합니다.

| 성숙도 단계 | 제어 수단 | 관련 도구 예시 |
| --- | --- | --- |
| 1단계: 전통적 | VPN, 방화벽 경계, 비밀번호 인증 | Cisco VPN, 온프레미스 AD |
| 2단계: 기초 ZT | SSO + MFA, 기본 IAM 정책 | Okta, AWS IAM, Google Workspace |
| 3단계: 심화 ZT | 장치 신뢰, 네트워크 세분화, SIEM | Intune, AWS Security Hub, Splunk |
| 4단계: 최적 ZT | 모든 구간 mTLS, 지속적 검증, JIT 접근 | Istio, SPIRE, HashiCorp Boundary |

> 💡
> Zero Trust는 제품이 아니라 아키텍처입니다
> 벤더들은 'Zero Trust' 제품을 홍보하지만, Zero Trust는 철학이자 아키텍처적 접근 방식입니다. 단일 제품으로 Zero Trust를 구현할 수는 없습니다. 완전한 ZTA는 아이덴티티(IdP), 장치(MDM/EDR), 네트워크(Microsegmentation/Service mesh), 애플리케이션(매 요청별 인가), 데이터(분류/DLP) 전반에 걸친 제어가 필요합니다. 벤더를 평가할 때 마케팅 문구가 아닌 그들이 어떤 기둥을 해결해 주는지를 확인하세요.

> 💡
> 면접 팁
> 시스템 디자인 면접에서 Zero Trust가 나오면 세 가지 기둥(명시적 검증, 최소 권한 사용, 침해 가정)을 중심으로 답변을 구성하세요. Service mesh(예: Istio)를 통한 mTLS가 어떻게 서비스 아이덴티티를 제공하고 내부 트래픽을 암호화하는지 설명하세요. 폭발 반경 제한을 위한 Microsegmentation을 언급하세요. Zero Trust가 VPN 모델을 대체하며, 어떤 네트워크에 있든 장치가 동일한 수준의 정밀 검사를 받는다는 점을 강조하세요. 이는 단순히 'HTTPS 사용'을 넘어선 아키텍처적 사고를 보여줍니다.