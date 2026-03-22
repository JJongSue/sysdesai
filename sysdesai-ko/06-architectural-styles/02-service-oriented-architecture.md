# Service-Oriented Architecture(SOA, 서비스 지향 아키텍처)

> 출처: https://www.sysdesai.com/learn/architectural-styles/service-oriented-architecture

---

## SOA란 무엇인가요?

**Service-Oriented Architecture(SOA, 서비스 지향 아키텍처)**는 소프트웨어를 네트워크를 통해 통신하고 상호 운용 가능한 서비스들의 집합으로 구성하는 아키텍처 스타일입니다. 2000년대 초반, 기업들이 비즈니스 단위별로 산재한 서로 다른 시스템들을 통합하고자 하면서 등장했습니다. SOA는 정형화된 계약(WSDL)으로 서비스를 정의하고, 표준 프로토콜(SOAP over HTTP)을 통해 통신하며, 종종 중앙의 **Enterprise Service Bus(ESB, 엔터프라이즈 서비스 버스)**를 통해 통신을 라우팅했습니다.

SOA는 서로 통신할 수 없었던 COBOL 메인프레임, SAP 인스턴스, Oracle Financials와 같은 단단히 결합된 엔터프라이즈 애플리케이션에 대한 대응으로 나타났습니다. 각 시스템을 잘 정의된 인터페이스를 가진 서비스로 감싸고, ESB가 라우팅, 변환 및 오케스트레이션을 담당하게 한다는 아이디어였습니다.

전통적인 SOA: ESB가 모든 서비스 통신을 중재함.

## SOA의 핵심 원칙(Core SOA Principles)

- **표준화된 서비스 계약(Standardized service contracts)**: 서비스는 정형화된 계약(예: SOAP 서비스를 위한 WSDL)을 통해 기능을 노출합니다. 소비자는 구현이 아닌 계약에 의존합니다.
- **느슨한 결합(Loose coupling)**: 서비스는 서로의 구현 세부 사항에 대한 의존성을 최소화합니다.
- **추상화(Abstraction)**: 서비스는 내부 로직을 외부로부터 숨깁니다.
- **재사용성(Reusability)**: 서비스는 여러 소비자 및 비즈니스 프로세스에서 재사용될 수 있도록 설계됩니다.
- **조합성(Composability)**: 서비스들을 결합하여 더 높은 수준의 비즈니스 프로세스를 구축할 수 있습니다(오케스트레이션).
- **무상태성(Statelessness)**: 가능한 경우 서비스는 요청 간에 세션 상태를 유지하지 않습니다.
- **발견 가능성(Discoverability)**: 서비스는 자신의 기능을 발행하여 런타임에 발견될 수 있도록 합니다.

## Enterprise Service Bus(ESB)

**ESB**는 SOA 배포의 중추 신경계와 같았습니다. ESB는 프로토콜 변환(SOAP에서 JMS로, 다시 REST로), 메시지 변환(하나의 XML 스키마를 다른 스키마로 매핑), 라우팅(올바른 백엔드로의 콘텐츠 기반 라우팅), 오케스트레이션(다단계 비즈니스 프로세스 조정) 및 보안 강제를 처리했습니다. IBM MQ, MuleSoft, Oracle Service Bus와 같은 제품들이 이 시장을 주도했습니다.

> ⚠️
> ESB 안티 패턴
> 실무에서 ESB는 'God object'가 되어버렸습니다. ESB에 너무 많은 비즈니스 로직과 라우팅 규칙이 쌓이면서 엔터프라이즈에서 가장 변경하기 힘든 부분이 되었습니다. 마틴 파울러는 이를 'Smart pipes, dumb endpoints(똑똑한 파이프, 단순한 엔드포인트)'라고 불렀는데, 이는 우리가 원하는 방향의 정반대입니다. 마이크로서비스는 이를 뒤집어 'Dumb pipes, smart endpoints(단순한 파이프, 똑똑한 엔드포인트)'를 지향합니다.

## SOA vs Microservices(마이크로서비스)

마이크로서비스는 때때로 '올바르게 구현된 SOA' 또는 'ESB가 제거된 SOA'로 묘사되기도 합니다. 서비스 분해 원칙은 공유하지만, 범위, 통신 스타일 및 운영 모델 면에서 크게 다릅니다.

| 비교 항목 | SOA | Microservices(마이크로서비스) |
| --- | --- | --- |
| 서비스 세분성 | 큼(DB 공유, 넓은 인터페이스) | 작음(Bounded context, 좁은 인터페이스) |
| 통신 방식 | ESB / SOAP / WS-* 표준 | REST, gRPC, 가벼운 메시지 브로커 |
| 데이터 소유권 | 공통된 전사적 DB 공유가 일반적 | 서비스별 개별 DB (필수) |
| 배포 방식 | 주로 함께 배포 (SOA 스위트) | 독립적으로 배포 가능 |
| 거버넌스(관리) | 중앙 집중식 (ESB, WSDL 레지스트리) | 분산형 (팀별 자율) |
| 범위 | 시스템 간 전사적 통합 | 시스템 내 애플리케이션 아키텍처 |
| 기술 스택 | 벤더 주도 (Oracle, IBM, SAP) | 오픈 소스, 클라우드 네이티브 |

## 여전히 유효한 SOA 패턴들

마이크로서비스가 지배적인 아키텍처 스타일로 SOA를 대체했음에도 불구하고, 여러 SOA 패턴은 여전히 필수적입니다: **API contracts(API 계약)**(OpenAPI 사양은 현대판 WSDL입니다), **Service registries(서비스 레지스트리)**(Consul, AWS Service Discovery), **Message brokers(메시지 브로커)**(가벼운 ESB 역할을 하는 Kafka, RabbitMQ), 그리고 **Orchestration(오케스트레이션)**(Saga 조정을 위한 Temporal, AWS Step Functions). 용어는 바뀌었지만 해결하려는 문제는 동일합니다.

📌
실제 사례: Salesforce 통합 허브

Salesforce CRM, SAP ERP 및 커스텀 이커머스 플랫폼을 통합하는 대기업들은 종종 REST/이벤트 기반 인터페이스를 갖춘 현대적인 통합 플랫폼(MuleSoft, Boomi)을 사용하는데, 이는 개념적으로 ESB와 유사합니다. 서비스 계약, 변환, 라우팅과 같은 SOA적 사고는 현대의 클라우드 아키텍처에서도 여전히 설계를 주도합니다.

> 💡
> 인터뷰 팁
> 면접관이 SOA에 대해 묻는다면 핵심은 이것입니다: SOA와 마이크로서비스는 분해 원칙을 공유하지만 ESB에서 차이가 납니다. SOA의 실패 원인은 로직을 서비스가 아닌 버스(똑똑한 파이프)에 집중시켰기 때문입니다. 마이크로서비스는 이를 똑똑한 엔드포인트와 단순한 파이프로 뒤집었습니다. 기존 레거시 시스템 간의 전사적 통합이 포함된 시스템이라면 메시지 변환이나 프로토콜 변환과 같은 SOA 스타일의 패턴은 종종 피할 수 없는 선택이 됩니다.
