# TLS/SSL & Encryption

> 출처: https://www.sysdesai.com/learn/security-auth/tls-encryption

---

## 암호화(Encryption) 기초

암호화는 암호화 알고리즘과 키를 사용하여 평문(plaintext)을 암호문(ciphertext)으로 변환하는 과정입니다. 분산 시스템에서는 두 가지 상황에서 암호화가 필요합니다: **In transit**(네트워크를 통해 이동 중인 데이터 보호)과 **At rest**(디스크나 데이터베이스에 저장된 데이터 보호)입니다. 세 번째 시나리오인 **In use**(동형 암호화, Homomorphic encryption)가 등장하고 있지만 아직 대부분의 운영 시스템에서는 실용적이지 않습니다.

| 구분 | Encryption in Transit | Encryption at Rest |
| --- | --- | --- |
| 프로토콜/표준 | TLS 1.2 / 1.3 | AES-256-GCM, ChaCha20-Poly1305 |
| 보호 대상 | 네트워크 도청, MITM (중간자 공격) | 디스크 도난, 무단 저장소 접근 |
| 키 수명 | 임시 세션 키 (Forward secrecy) | 장기 수명 키, 주기적 로테이션 |
| 관리 주체 | TLS 인증서, CDN/LB | KMS (AWS KMS, GCP KMS, HashiCorp Vault) |
| 성능 영향 | TLS 1.3 + 하드웨어 가속 시 낮음 | AES-NI CPU 명령어 사용 시 낮음 |

## TLS 1.3 Handshake

**TLS 1.3** (RFC 8446, 2018)은 현재의 표준입니다. 키 교환을 첫 번째 메시지에 통합하여 Handshake를 2 RTT에서 **1 RTT**로 간소화했습니다. 또한 서버의 Private key가 나중에 유출되더라도 과거 세션을 복호화할 수 없도록 하는 **Forward Secrecy**(임시 Diffie-Hellman 키 교환)를 의무화했습니다. 취약하거나 더 이상 사용되지 않는 암호화 알고리즘(RC4, MD5, SHA-1, DES, 3DES)을 완전히 제거했습니다.

## Mutual TLS (mTLS)

표준 TLS는 클라이언트에게 **서버**만 인증합니다. **Mutual TLS (mTLS)**는 클라이언트 인증서 인증을 추가하여 양측이 모두 인증서를 제시하도록 합니다. 이는 Microservices 환경에서 **서비스 간 인증(Service-to-service authentication)**의 골드 스탠다드입니다. 내부 서비스 간에 API Key나 토큰이 필요 없게 됩니다. **Istio**, **Linkerd**, **Consul Connect**와 같은 Service mesh는 애플리케이션 코드 변경 없이 mTLS 인증서를 투명하게 주입하고 로테이션할 수 있습니다.

## Encryption at Rest: 키 관리

Encryption at rest의 강도는 키 관리 능력에 달려 있습니다. 암호화 키를 암호화된 데이터 옆에 저장하거나 소스 코드에 하드코딩하는 것은 보호 효과가 거의 없습니다. 업계 표준은 **Key Management Service (KMS)**를 사용하는 것입니다: AWS KMS, GCP Cloud KMS, Azure Key Vault 또는 HashiCorp Vault 등이 있습니다. 이러한 서비스는 키를 생성하고 평문으로 내보내지 않는 **Hardware Security Modules (HSMs)**라는 변조 방지 하드웨어 장치에 마스터 키를 저장합니다.

### Envelope Encryption 패턴

대량의 데이터를 KMS로 직접 암호화하는 것은 비효율적입니다(KMS에는 크기 제한과 API 호출 제한이 있음). 해결책은 **Envelope Encryption**입니다: 각 데이터 조각을 위해 무작위 **Data Encryption Key (DEK)**를 생성하고, 데이터를 DEK로 암호화한 다음, KMS에 저장된 **Key Encryption Key (KEK)**로 DEK를 암호화합니다. 암호화된 DEK를 암호문과 함께 저장합니다. 복호화할 때는 KMS를 호출하여 DEK를 복호화한 후, 그 DEK로 데이터를 복호화합니다.

## 인증서 관리

- **Let's Encrypt + ACME 프로토콜**: 무료이며 자동화된 인증서 발급 및 갱신을 지원합니다. 대부분의 리버스 프록시(Nginx, Caddy, Traefik)에서 지원됩니다. 인증서가 90일마다 만료되므로 자동화가 필수적입니다.
- **AWS Certificate Manager (ACM)**: AWS 리소스(ALB, CloudFront)를 위한 관리형 TLS 인증서입니다. 자동으로 갱신되며 AWS 통합 리소스에 대해 무료입니다.
- **Certificate Pinning**: 클라이언트가 특정 인증서나 Public key만 수락하도록 하여 가짜 CA 서명 인증서를 이용한 중간자 공격을 방지합니다. 위험성: 로테이션이 어렵고 설정 오류 시 서비스 중단이 발생할 수 있습니다.
- **HSTS (HTTP Strict Transport Security)**: 브라우저가 해당 도메인에 대해 항상 HTTPS를 사용하도록 강제합니다. `includeSubDomains`와 `preload` 지시어를 포함하고, 최대 보호를 위해 HSTS Preload list에 등록하세요.

> ⚠️
> 암호화 알고리즘을 직접 만들지 마세요
> 암호화 알고리즘을 직접 구현하지 마세요. 검증된 라이브러리를 사용하세요: **libsodium**(현대적이고 직관적이며 오용하기 어려움), **OpenSSL**(어디에나 있지만 API가 복잡함), 또는 **Bouncy Castle**(JVM). 암호화 버그는 매우 미묘합니다. Heartbleed, POODLE, BEAST 공격 등은 수학적 결함이 아니라 운영 시스템에서의 구현 오류였습니다.

> 💡
> 면접 팁
> 면접에서 TLS가 언급되면 TLS 1.3의 1-RTT Handshake와 취약한 암호문 제거에 대해 언급하세요. 내부 서비스에는 Service mesh를 통한 mTLS를 추천하고, 키 관리에는 AWS KMS나 HashiCorp Vault를 사용한 Envelope Encryption을 언급하세요. 이러한 폭넓은 이해는 프로토콜 수준과 운영 실무를 모두 파악하고 있음을 보여줍니다.