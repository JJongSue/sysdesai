# API Gateway & Gateway Aggregation

> 원문: https://www.sysdesai.com/learn/decomposition-integration/api-gateway

---

## API Gateway란 무엇인가?

**API Gateway**는 클라이언트와 백엔드 microservices 사이에 위치하는 단일 진입점(entry point)입니다. 인증(authentication), rate limiting, SSL termination, 로깅, 라우팅과 같은 횡단 관심사(cross-cutting concerns)를 처리하여, 개별 서비스들이 이를 독립적으로 구현할 필요가 없도록 해줍니다. 분산 시스템의 '정문'이라고 생각하면 됩니다.

게이트웨이가 없다면 모든 클라이언트는 모든 서비스의 주소를 알아야 하고, 각 서비스에 대한 인증을 직접 처리해야 하며, 개별 서비스의 파편화된 데이터 모델을 직접 다뤄야 합니다. 게이트웨이는 깨끗하고 통일된 façade를 제공합니다. 예시: **AWS API Gateway**, **Kong**, **Apigee**, **Nginx**, **Envoy**, **Traefik**, **Azure API Management**.

## 게이트웨이의 핵심 책임

API Gateway는 모든 횡단 관심사를 중앙 집중화하고 요청을 적절한 다운스트림 서비스로 라우팅합니다.

| 관심사 | 게이트웨이의 역할 |
| --- | --- |
| Request routing | 경로(path), 메서드(method) 또는 헤더를 기반으로 요청을 올바른 microservice로 전달 |
| Authentication | JWT 또는 API 키를 검증하여, 권한이 없는 요청이 서비스에 도달하기 전에 거부 |
| Rate limiting | 클라이언트 또는 엔드포인트별 요청 할당량 강제 |
| SSL termination | edge에서 TLS를 처리; 서비스 간 통신은 내부 HTTP를 통해 수행 |
| Load balancing | 서비스의 여러 인스턴스에 요청을 분산 |
| Response aggregation | 여러 서비스의 응답을 하나의 응답으로 결합 |
| Protocol translation | REST → gRPC, HTTP/1.1 → HTTP/2 등으로 변환 |
| Caching | 멱등성(idempotent)이 있는 응답을 캐싱하여 백엔드 부하 감소 |

## Gateway Aggregation 패턴

**Gateway Aggregation**은 게이트웨이가 단일 클라이언트 요청을 여러 다운스트림 서비스로 분산(fan out)시킨 후, 그 응답들을 하나의 페이로드로 병합하는 특정 기능입니다. 집계(aggregation)가 없다면, 제품 상세 페이지를 렌더링하는 클라이언트는 제품 정보, 재고, 리뷰를 위해 세 번의 별도 API 호출이 필요할 수 있습니다. 집계를 사용하면 한 번의 호출로 모든 정보를 반환받을 수 있습니다.

> 💡
> 항상 병렬로 Fan Out 하십시오
> 집계할 때 다운스트림 호출은 순차적이 아닌 항상 동시(병렬)에 수행해야 합니다. 순차적 호출은 지연 시간을 누적시킵니다: 각 100ms인 호출 3번 = 300ms. 병렬 호출은 max(100, 100, 100) = 100ms가 소요됩니다. 게이트웨이 구현 언어에서 `Promise.all` 또는 그에 상응하는 async/await 방식을 사용하십시오.

## 실제 사례: AWS API Gateway

AWS API Gateway는 가장 널리 사용되는 managed gateway입니다. AWS Lambda(serverless 백엔드용), Cognito(인증용), CloudWatch(로깅용)와 네이티브하게 통합됩니다. REST API, HTTP API(더 저렴하고 낮은 지연 시간), WebSocket API를 지원합니다. AWS는 또한 단순한 라우팅을 위한 **Application Load Balancer (ALB)**와 GraphQL 집계를 위한 **AWS AppSync**를 제공합니다.

## 게이트웨이 안티 패턴

- **Smart gateway, dumb services** — 비즈니스 로직(가격 책정, 자격 요건 규칙 등)이 게이트웨이로 옮겨가면, 게이트웨이는 병목 지점이자 배포 리스크가 됩니다. 게이트웨이는 가볍게(thin) 유지하십시오.
- **Single point of failure로서의 게이트웨이** — 상태 확인(health checks) 및 자동 장애 조치(failover)와 함께 여러 가용 영역(availability zones)에 게이트웨이를 배포하십시오.
- **과도하게 빈번한(chatty) 집계** — 15개의 다운스트림 호출을 동기적으로 집계하면 매우 긴 critical path가 생성됩니다. 복잡한 집계 요구사항에는 캐싱, 비동기 pre-fetching 또는 GraphQL을 사용하십시오.
- **버전 난립 (Version sprawl)** — 수십 개의 게이트웨이 라우트 버전이 쌓이지 않도록 하십시오. 명확한 중단(deprecation) 일정과 함께 버전 관리 정책(URI 버전 관리 `/v1/`, `/v2/` 등)을 수립하십시오.

> 💡
> 면접 팁 (Interview Tip)
> microservices가 포함된 모든 시스템 설계 면접에는 API gateway가 포함되어야 합니다. 다이어그램을 그릴 때 인터넷과 서비스 사이의 edge에 게이트웨이를 배치하십시오. 그리고 명시적으로 언급하십시오: '개별 서비스가 구현할 필요가 없도록 게이트웨이가 인증, rate limiting, SSL termination을 처리합니다.' 만약 질문에 모바일과 웹 클라이언트가 모두 포함된다면, 게이트웨이를 BFF 패턴과 결합하십시오. 이 조합은 깊이 있는 아키텍처 지식을 보여줍니다.
