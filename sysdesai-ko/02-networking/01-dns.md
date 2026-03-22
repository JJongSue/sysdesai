# DNS & 도메인 분석(Domain Resolution)

> Source: https://www.sysdesai.com/learn/networking/dns

---

## DNS란 무엇인가?

**도메인 네임 시스템(DNS, Domain Name System)**은 인터넷의 분산 전화번호부와 같습니다. 사람이 읽을 수 있는 `api.example.com`과 같은 호스트네임을 라우터가 실제로 사용할 수 있는 `93.184.216.34`와 같은 IP 주소로 변환합니다. DNS가 없다면 모든 사용자는 숫자로 된 IP 주소를 외워야 할 것이며, 회사가 서버를 이전할 때마다 그 주소들이 예고 없이 변경되어 큰 불편을 겪게 될 것입니다.

DNS는 계층 구조를 가진 글로벌 분산 시스템입니다. 단일 서버가 모든 도메인-IP 매핑 정보를 보유하지 않습니다. 대신 시스템의 속도를 유지하기 위해 모든 수준에서 공격적인 캐싱(Caching)을 수행하며, 권한 있는 서버(Authoritative server) 트리로 확인 절차를 위임합니다.

## DNS 분석 과정(The DNS Resolution Process)

브라우저가 `www.example.com`으로 이동할 때 일련의 조회 과정이 발생합니다. 전체 프로세스는 일반적으로 콜드 캐시(Cold cache) 상태에서 20-120ms 내에 완료되며, 캐시된 정보를 사용할 때는 거의 0에 가까운 시간이 소요됩니다.

1. **브라우저 캐시(Browser cache)**: 브라우저는 자신의 DNS 캐시를 가장 먼저 확인합니다 (Chrome은 TTL에 관계없이 최대 1분 동안 레코드를 저장합니다).
2. **OS 캐시 / hosts 파일**: 브라우저 캐시에 없는 경우, OS 리졸버(Resolver)는 자신의 캐시와 `/etc/hosts` 파일을 확인합니다.
3. **재귀적 리졸버(Recursive resolver, ISP 또는 8.8.8.8)**: OS는 쿼리를 설정된 재귀적 리졸버(보통 ISP 또는 Google의 `8.8.8.8`)로 전달합니다. 이 리졸버가 복잡한 조회 과정을 대신 처리합니다.
4. **루트 네임 서버(Root name servers)**: 재귀적 리졸버에 캐시 항목이 없으면, ICANN이나 VeriSign 등이 운영하는 13개의 루트 서버 클러스터 중 하나에 TLD 네임서버를 요청합니다.
5. **TLD 네임 서버(TLD name servers)**: 루트 서버는 리졸버를 `.com` TLD 서버로 안내하며, 이 서버는 `example.com`을 담당하는 권한 있는 서버가 어디인지 알고 있습니다.
6. **권한 있는 네임 서버(Authoritative name servers)**: 리졸버는 마침내 `example.com`의 권한 있는 서버에 도달하여 실제 A 레코드(IPv4) 또는 AAAA 레코드(IPv6)를 응답받습니다.
7. **캐싱 및 반환**: 각 단계는 레코드의 TTL과 함께 캐시됩니다. 최종 IP 주소가 브라우저로 반환됩니다.

## 주요 DNS 레코드 유형(Key DNS Record Types)

| 레코드 | 용도 | 예시 |
| --- | --- | --- |
| A | 호스트네임을 IPv4 주소로 매핑 | `example.com → 93.184.216.34` |
| AAAA | 호스트네임을 IPv6 주소로 매핑 | `example.com → 2606:2800::1` |
| CNAME | 한 호스트네임에서 다른 호스트네임으로의 별칭(Alias) | `www.example.com → example.com` |
| MX | 도메인의 메일 서버 | `example.com → mail.example.com` |
| TXT | 임의의 텍스트(SPF, DKIM, 검증용) | `v=spf1 include:sendgrid.net ~all` |
| NS | 도메인에 권한이 있는 네임 서버 | `example.com → ns1.example.com` |
| SOA | 권한 시작(Start of Authority) — 영역(Zone) 메타데이터 | Serial, refresh, retry intervals |
| SRV | 서비스 위치(포트 + 호스트네임) | `_grpc._tcp.example.com → svc:443` |

## TTL과 캐싱(TTL and Caching)

모든 DNS 레코드에는 초 단위로 측정되는 **유효 기간(TTL, Time To Live)**이 있습니다. 리졸버와 브라우저는 TTL이 만료된 후 캐시된 레코드를 폐기해야 합니다. 적절한 TTL을 선택하는 것은 엔지니어링의 트레이드오프(Trade-off) 문제입니다.

| TTL 설정 | 트레이드오프 | 사용 사례 |
| --- | --- | --- |
| 낮은 TTL (30–300초) | 최신 데이터를 유지하지만, DNS 쿼리가 많아지고 권한 있는 서버의 부하가 증가함 | 빠른 전파가 필요한 배포, 이전 또는 장애 조치(Failover) 중 |
| 높은 TTL (3600–86400초) | 쿼리 수가 적고 성능이 향상되지만, 변경 사항 전파가 느림 | IP 변경이 거의 없는 안정적인 운영 서비스 |
| 매우 낮은 TTL (0–30초) | 거의 실시간 업데이트 가능, 리졸버 부하가 매우 큼 | 카나리 배포(Canary deployment), 블루/그린 전환 |

> ⚠️
> TTL은 즉각적인 전파를 의미하지 않습니다.
> TTL을 낮추거나 레코드를 변경하더라도, 기존 리졸버는 캐시된 항목이 만료될 때까지 계속해서 이전 데이터를 제공하며, 이는 **이전의**(더 높은) TTL을 반영할 수 있습니다. 계획된 이전 전에는 항상 미리 TTL을 낮춰두어야 합니다.

## DNS 기반 부하 분산(DNS-Based Load Balancing)

DNS는 단일 호스트네임에 대해 여러 개의 A 레코드를 반환함으로써 원시적인 부하 분산 메커니즘으로 사용될 수 있습니다. DNS 클라이언트는 일반적으로 주소를 순서대로 또는 무작위로 시도하여 트래픽을 여러 서버로 분산합니다.

- **라운드 로빈 DNS(Round-robin DNS)**: 여러 개의 A 레코드를 반환하며 클라이언트가 이를 순환하며 사용합니다. 단순하지만 상태 확인(Health awareness) 기능이 없어, TTL이 만료될 때까지 죽은 서버가 응답에 포함될 수 있습니다.
- **가중치 레코드(Weighted records)**: 일부 DNS 제공업체는 트래픽 분할을 위해 레코드에 가중치를 부여할 수 있게 합니다(예: Route 53 가중치 기반 라우팅). 카나리 릴리스(Canary release)에 유용합니다.
- **Geo DNS / 지연 시간 기반 라우팅(Latency-based routing)**: 쿼리를 요청한 클라이언트의 지리적 위치에 따라 다른 IP를 반환합니다. AWS Route 53과 Cloudflare가 이를 기본적으로 지원합니다.
- **장애 조치 DNS(Failover DNS)**: 기본 레코드를 정상적으로 제공하다가 상태 확인에 실패하면 자동으로 보조 IP로 전환합니다. 복구 속도는 TTL에 의존합니다.

> 💡
> 면접 팁
> 면접에서 DNS는 글로벌 트래픽 라우팅이나 장애 조치에 대해 논의할 때 자주 등장합니다. DNS 기반 부하 분산은 입도가 거칠고 TTL에 의존적이라는 점을 언급하세요. L7 부하 분산기처럼 실시간 서버 상태에 즉각 반응할 수 없습니다. 1초 미만의 장애 조치를 위해서는 낮은 TTL과 상태 확인 지원 라우팅(예: Route 53 health checks)의 조합이 필요합니다. 순수 DNS 라운드 로빈은 서버 상태를 전혀 감지하지 못합니다.

## DNSSEC 및 HTTPS를 통한 DNS(DNS over HTTPS)

전통적인 DNS는 인증되지 않았으며 암호화되지 않았습니다. **DNSSEC**은 DNS 레코드에 암호화 서명을 추가하여 리졸버가 응답이 변조되지 않았음을 검증할 수 있게 합니다(캐시 포이즈닝 공격 방지). **DoH(DNS over HTTPS)** 및 **DoT(DNS over TLS)**는 DNS 쿼리를 종단 간 암호화하여 ISP나 네트워크 공격자의 도청을 방지합니다.

> ℹ️
> 실무에서의 DNS
> Cloudflare(1.1.1.1), Google(8.8.8.8), AWS Route 53과 같은 주요 제공업체는 DoH와 DoT를 모두 지원합니다. 내부 마이크로서비스의 경우, 많은 팀이 공용 DNS 대신 Consul이나 CoreDNS와 같은 서비스 디스커버리(Service discovery) 도구를 사용하여 1초 미만의 상태 확인 지원 분석을 구현합니다.
