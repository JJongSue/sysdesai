# DDoS Protection

> 출처: https://www.sysdesai.com/learn/security-auth/ddos-protection

---

## DDoS 공격의 이해

**Distributed Denial of Service (DDoS)** 공격은 여러 소스에서 동시에 트래픽을 쏟아부어 서비스를 마비시키는 것을 목표로 합니다. 단일 IP에서 발생하는 DoS 공격(차단이 쉬움)과 달리, DDoS는 수천 또는 수백만 개의 감염된 장치로 구성된 **봇넷(botnets)**을 사용하므로 단순한 IP 차단으로는 효과가 없습니다. DDoS 공격은 표적으로 하는 OSI 계층에 따라 분류되며, 이에 따라 대응 전략이 달라집니다.

| 계층 | 공격 유형 | 예시 | 고갈 대상 | 대응 방안 |
| --- | --- | --- | --- | --- |
| 3 (Network) | Volumetric flood | UDP flood, ICMP flood | 네트워크 대역폭 | CDN scrubbing, anycast |
| 4 (Transport) | Protocol attack | SYN flood, ACK flood | TCP connection table, 방화벽 상태 | SYN cookies, stateless LB |
| 4 (Amplification) | Reflection/amplification | DNS amplification (70x), NTP (556x) | 희생자의 대역폭 | BCP38, RRL, anycast |
| 7 (Application) | HTTP flood | Slowloris, GET flood | 웹 서버 연결/CPU | WAF, rate limiting, CAPTCHA |
| 7 (Application) | Slow-and-low | Slow POST, slow read | 서버 스레드 풀 | 타임아웃 조정, 연결 제한 |

## 방어 전략: 계층적 완화 (Layered Mitigation)

단일 제어 수단으로 모든 유형의 DDoS를 막을 수는 없습니다. 효과적인 보호를 위해서는 여러 계층에 걸친 **심층 방어(Defense in depth)**가 필요하며, 각 계층은 서로 다른 공격 범주를 방어합니다.

## CDN 및 Anycast를 통한 흡수

Volumetric 공격에 대한 가장 효과적인 방어는 트래픽이 원본(origin) 서버에 도달하기 전에 에지(edge)에서 흡수하는 것입니다. Cloudflare(100+ Tbps 네트워크), Akamai, AWS CloudFront와 같은 **CDN**은 전 세계에 분산된 PoP(Points of Presence)를 보유하고 있으며, 단일 원본 서버보다 훨씬 더 큰 총 대역폭을 가지고 있습니다. **Anycast routing**을 사용하면 동일한 IP 주소를 여러 PoP에서 동시에 알릴 수 있어, 공격자의 패킷이 가장 가까운 PoP로 라우팅됩니다. 이를 통해 부하를 전 세계로 분산시키고 특정 지점이 마비되는 것을 방지합니다.

> ℹ️
> Origin IP를 비밀로 유지하세요
> 공격자가 DNS 히스토리, 인증서 투명성 로그, 이메일 헤더 등을 통해 원본 IP를 알아내면 CDN을 우회하여 직접 공격할 수 있습니다. 원본 IP를 비공개로 유지하세요: 방화벽 규칙을 설정하여 CDN의 IP 범위에서 오는 트래픽만 허용하고, 원본 IP를 외부로 노출될 수 있는 서비스(예: 직접 이메일 발송)에 사용하지 마세요.

## Layer 4 방어: SYN Cookies 및 연결 제한

**SYN Cookies**는 아직 연결이 확립되지 않은 세션에 대해 서버 측 상태를 유지하지 않음으로써 SYN flood 문제를 해결합니다. 서버는 SYN을 받았을 때 연결 테이블 항목을 할당하는 대신, 암호화 해시를 사용하여 연결 파라미터를 SYN-ACK의 ISN(초기 시퀀스 번호)에 인코딩합니다. 클라이언트가 올바른 ACK로 Handshake를 완료하면 서버가 상태를 재구성합니다. 공격(flood) 상황에서 클라이언트가 응답하지 않으면 리소스가 소비되지 않습니다. 리눅스 커널은 1996년부터 SYN cookies를 지원해 왔습니다.

## Layer 7 방어: WAF 및 봇 탐지

HTTP flood나 Slowloris 공격은 유효한 TCP 연결을 사용하므로 네트워크 계층 방어를 우회합니다. 대응 방안으로는 다음이 있습니다: **IP별 Rate limiting**(이전 레슨 참조), 봇 시그니처를 탐지하는 **WAF 규칙**(Accept-Language 헤더 누락, 비정상적인 User-Agent 패턴, 비정상적인 요청 타이밍 등), 의심스러운 트래픽에 대한 **CAPTCHA** 챌린지, 그리고 일정한 간격으로 요청을 보내는 패턴(인간과 다른 봇 특유의 패턴)을 식별하는 **행동 분석** 등이 있습니다.

## Auto-Scaling: 유용하지만 한계가 있음

Auto-scaling은 컴퓨팅 리소스에 부하를 주는 **Layer 7 Application flood**에 효과적입니다. 공격자가 비용이 많이 드는 데이터베이스 쿼리나 CPU 집약적인 작업을 보낼 때 애플리케이션 서버를 추가하여 부하를 분산할 수 있습니다. 하지만 Auto-scaling에는 한계가 있습니다: 인스턴스 실행에 시간(통상 2~5분)이 걸리고, 공격자도 이에 맞춰 공격 규모를 키울 수 있으며, 어느 시점에는 방어 비용이 감당할 수 없을 정도로 커집니다. Auto-scaling만으로는 충분하지 않으며 반드시 상단에서의 필터링과 병행되어야 합니다.

## 관리형 DDoS 보호 서비스

- **Cloudflare Magic Transit**: 테라비트급 공격을 흡수할 수 있는 Anycast 기반 L3/L4 DDoS 완화 서비스입니다. 모든 IP 트래픽을 Cloudflare 네트워크를 통해 라우팅합니다.
- **AWS Shield Standard**: 모든 AWS 고객에게 기본으로 제공됩니다. SYN flood, UDP flood 등 일반적인 L3/L4 공격을 추가 비용 없이 방어합니다.
- **AWS Shield Advanced**: 월 $3,000. L7 보호, 공격 포렌식, 24/7 DDoS 응답 팀(DRT), 그리고 DDoS로 인한 Auto-scaling 비용 보호 기능(AWS가 비용 부담)을 추가로 제공합니다.
- **Cloudflare DDoS Protection**: 모든 Cloudflare 요금제에 포함되어 있습니다. Rate limiting, 봇 관리, WAF 등은 부가 서비스로 이용 가능합니다.
- **Akamai Prolexic**: 엔터프라이즈급 스크러빙 센터(Scrubbing centers)를 운영합니다. 공격 중에도 99.999% 가동시간이 필요한 주요 금융 기관 등에서 사용합니다.

> 💡
> 면접 팁
> 면접에서는 계층적 접근 방식을 설명하세요: Volumetric L3/L4 공격에는 CDN/Anycast, 프로토콜 공격에는 SYN cookies, L7 flood에는 WAF + Rate limiting, 그리고 합법적인 트래픽 급증에는 Auto-scaling을 제안하세요. 또한 원본 IP를 비공개로 유지하는 것이 전제 조건임을 언급하세요. CDN을 우회할 수 있게 되면 모든 에지 방어는 무용지물이 되기 때문입니다.