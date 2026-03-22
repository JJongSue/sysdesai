# Adapter Pattern in Distributed Systems

> 원문: https://www.sysdesai.com/learn/decomposition-integration/adapter-pattern

---

## Adapter Pattern: OOP에서 분산 시스템으로

**Adapter pattern**은 GoF(Gang of Four)의 구조 패턴 중 하나입니다. 객체 지향 설계에서 어댑터는 호환되지 않는 인터페이스를 래핑하여 클라이언트가 기대하는 인터페이스처럼 보이게 만듭니다. 분산 시스템에서도 동일한 개념이 더 큰 규모로 적용됩니다. 어댑터 서비스 또는 컴포넌트는 서로 호환되지 않는 프로토콜, 데이터 형식, 또는 API contract 사이를 변환하여 두 시스템이 서로를 변경하지 않고도 통신할 수 있게 합니다.

Anti-Corruption Layer(당신의 도메인 모델 보호가 목적)나 Ambassador(아웃바운드 연결의 복원력이 목적)와 달리, 어댑터는 **인터페이스 호환성(interface compatibility)**에 좁게 초점을 맞춥니다. 즉, 서로 맞지 않는 두 가지를 서로 맞게 만드는 역할을 합니다.

## 분산 시스템에서의 어댑터 유형

| 어댑터 유형 | 해결하는 문제 | 예시 |
| --- | --- | --- |
| Protocol Adapter | 클라이언트는 REST를 사용하고, 서비스는 gRPC나 SOAP를 사용할 때 | gRPC-gateway를 통한 REST → gRPC 변환 |
| Data Format Adapter | 시스템들이 서로 다른 직렬화 형식을 사용할 때 | Kafka 파이프라인에서의 JSON → Avro 변환기 |
| Schema Adapter | 개념은 같지만 필드 이름이나 구조가 다를 때 | `customer_id` → `user_guid` 변환 |
| Semantic Adapter | 필드 이름은 같지만 의미가 다를 때 | UTC 타임스탬프를 파트너사의 epoch 형식으로 변환 |
| Authentication Adapter | 호환되지 않는 인증 방식 | OAuth 2.0 토큰 → 레거시 Basic Auth 헤더로 변환 |

## Protocol Adapter 예시: gRPC-Gateway

흔한 시나리오: 내부 microservices는 고성능 서비스 간 통신을 위해 **gRPC**를 사용하지만, 외부 클라이언트(브라우저, 모바일 앱)는 REST를 사용합니다. 두 개의 API를 유지하는 대신, **gRPC-Gateway**를 배포합니다. 이 어댑터는 HTTP/JSON REST 요청을 gRPC 호출로 변환하고, protobuf 응답을 다시 JSON으로 변환합니다.

## 이벤트 기반 파이프라인의 Data Format Adapter

Kafka를 사용하는 이벤트 기반 아키텍처에서는 팀마다 서로 다른 형식으로 이벤트를 생성할 수 있습니다. **Kafka Streams adapter**(또는 Kafka Connect transformer)가 파이프라인 사이에 위치하여 한 형식의 이벤트를 소비하고 다른 형식으로 다시 발행할 수 있습니다. 예를 들어, 레거시 시스템이 XML 이벤트를 발행하면 어댑터가 이를 Avro로 변환하여 현대적인 소비자들이 받을 수 있게 합니다.

```typescript
// Data format adapter: 레거시 XML 이벤트를 현대적인 JSON 이벤트로 변환
interface LegacyOrderEvent {
  // 레거시 시스템의 XML 기반 구조
  OrderId: string;
  CustRef: string;
  ItemQty: number;
  OrderTimestamp: string; // "2024-01-15T10:30:00Z"
}

interface ModernOrderEvent {
  orderId: string;
  customerId: string;
  quantity: number;
  createdAt: number; // Unix timestamp in ms
}

class OrderEventAdapter {
  transform(legacy: LegacyOrderEvent): ModernOrderEvent {
    return {
      orderId: legacy.OrderId,
      customerId: legacy.CustRef,
      quantity: legacy.ItemQty,
      createdAt: new Date(legacy.OrderTimestamp).getTime(),
    };
  }
}
```

## Adapter vs Anti-Corruption Layer vs Facade

| 패턴 | 주요 초점 | 범위 |
| --- | --- | --- |
| Adapter | 인터페이스 호환성 (구조적/형식 변환) | 단일 클래스 또는 microservice 컴포넌트 |
| Anti-Corruption Layer | 도메인 모델 보호 + 변환 | Bounded contexts 간의 전체 경계 |
| Facade | 복잡한 하위 시스템 인터페이스 단순화 | 복잡성을 숨기는 단일 클래스 |
| Translator (in ACL) | 도메인 개념 간의 의미적 변환 | ACL의 일부 |

> ℹ️
> Adapter는 종종 ACL에 포함됩니다
> 실제로 어댑터는 Anti-Corruption Layer 내부의 구성 요소 중 하나인 경우가 많습니다. ACL은 어댑터(프로토콜/형식 변환용), translator(의미적 매핑용), facade(인터페이스 단순화용)를 함께 사용합니다. 누군가 ACL 설계를 요청하면, 어댑터를 그 내부 빌딩 블록 중 하나로 설명할 수 있습니다.

## 전용 어댑터 서비스를 도입해야 할 때

때로는 어댑터 로직이 충분히 방대하여 별도의 microservice로 만들 가치가 있을 때가 있습니다. 이를 흔히 **integration service** 또는 **adapter service**라고 부릅니다. 다음과 같은 경우에 적합합니다:

- 변환이 상태 저장(stateful) 방식이거나 조회가 필요할 때 (예: 매핑 테이블을 통해 외부 고객 ID를 내부 UUID로 매핑).
- 다운스트림 소비자에게 도달하기 전에 여러 업스트림 생산자의 데이터를 통합해야 할 때.
- 어댑터에 독립적인 확장이 필요할 때 (예: Kafka 파이프라인의 대량 이벤트 변환).
- 외부 API가 인증, 재시도 로직, 자체 서킷 브레이커를 필요로 할 때.

> 💡
> 면접 팁 (Interview Tip)
> Adapter 패턴 자체가 면접의 주인공이 될 가능성은 낮지만, 더 큰 답변을 구성하는 빌딩 블록으로서 중요합니다. 제3자 API 통합에 대해 논의할 때 다음과 같이 언급하십시오: '제3자 클라이언트를 어댑터로 래핑하여 그들의 인터페이스를 우리 도메인에 맞게 표준화하겠습니다. 이것이 우리 Anti-Corruption Layer의 첫 번째 계층이 될 것입니다.' 이는 패턴들이 어떻게 결합되고 연관되는지 이해하고 있음을 보여줍니다.
