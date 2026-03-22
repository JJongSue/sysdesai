# CQRS (Command Query Responsibility Segregation)

> 출처: https://www.sysdesai.com/learn/data-management-patterns/cqrs

---

## CQRS란 무엇일까?

**CQRS** (Command Query Responsibility Segregation)는 데이터를 읽는 작업(Queries)과 데이터를 수정하는 작업(Commands)을 분리하는 패턴입니다. Greg Young이 제안하고 Bertrand Meyer의 Command-Query Separation 원칙에서 영감을 받은 이 패턴은, 읽기와 쓰기를 위해 완전히 별개의 models를 사용하며 때로는 별도의 data stores를 두기도 합니다.

전통적인 CRUD architecture에서는 하나의 model이 읽기와 쓰기를 모두 처리합니다. 간단한 시스템에서는 잘 작동하지만, 복잡도가 높아지면 어려움이 생깁니다. 쓰기 작업에는 강력한 consistency와 transactional integrity가 필요한 반면, 읽기 작업에는 특정 UI 뷰에 최적화된 denormalized되고 빠른 접근이 가능한 projections가 필요하기 때문입니다. CQRS는 각 영역이 독립적으로 진화할 수 있게 하여 이런 문제를 해결합니다.

CQRS는 쓰기 경로와 읽기 경로를 분리하며, 각각 최적화된 data store를 가집니다.

## Commands vs Queries

| 항목 | Command | Query |
| --- | --- | --- |
| 의도 | 상태 변경 | 데이터 반환 |
| 반환 값 | 주로 void 또는 ID | 데이터 반환 (상태를 변경하지 않음) |
| 예시 | `PlaceOrder`, `CancelShipment` | `GetOrderById`, `ListUserOrders` |
| Consistency | Strong (transactions 사용) | Eventual (projection에서 읽음) |
| Scalability | 쓰기에 최적화된 DB | Read replicas, caches, search indexes |

## 구현 단계

CQRS는 단계별로 적용할 수 있습니다. 처음부터 모든 것을 도입할 필요는 없습니다.

1. **Simple CQRS** — 단일 데이터베이스를 사용하지만 코드상에서 command와 query 객체를 분리합니다. 오버헤드가 적고 관심사 분리가 잘 됩니다.
2. **Read replica CQRS** — 쓰기에는 동일한 기본 DB를 사용하지만, queries는 read replica로 보냅니다. 구현이 쉽지만 replica lag이 주요 고려 사항입니다.
3. **Full CQRS** — 읽기와 쓰기에 별도의 data stores를 사용합니다. 쓰기 쪽은 정규화된 관계형 DB를 사용하고, 읽기 쪽은 MongoDB, Elasticsearch, Redis 같은 denormalized store를 사용합니다. 성능을 극대화하지만 운영 복잡도가 높아집니다.

> ⚠️
> CQRS를 남용하지 마세요.
> CQRS는 별도의 models, eventual consistency, 동기화 로직 등 상당한 복잡성을 추가합니다. 읽기 및 쓰기 성능 요구사항이나 팀 확장성이 정말 필요한 경우에만 적용하세요. 단순한 CRUD 애플리케이션은 full CQRS로 얻는 이득이 없습니다.

## 읽기와 쓰기 사이의 Eventual Consistency

별도의 읽기 및 쓰기 저장소를 사용하면 읽기 모델은 **eventual consistency** 상태가 됩니다. 쓰기 command가 완료된 후, 시스템은 이벤트를 발행하거나 CDC를 사용하여 읽기 저장소를 업데이트합니다. 이 업데이트가 전파될 때까지 queries는 stale data를 반환할 수 있습니다. 이는 CQRS 시스템에서 가장 흔히 발생하는 버그의 원인입니다.

모델 간의 eventual consistency를 가지는 CQRS 쓰기 후 읽기 흐름입니다.

## 실무 적용 사례

**Amazon**은 주문 관리 시스템에 CQRS를 폭넓게 사용합니다. 쓰기 쪽은 ACID guarantees와 함께 주문을 처리하고, 읽기 projections는 주문 내역 페이지를 구동하는 최적화된 denormalized 뷰를 제공합니다. **Stack Overflow**도 비슷한 패턴을 사용합니다. 투표 수는 트랜잭션 방식으로 기록되지만, 빠른 태그 기반 검색을 위해 캐시되거나 투영됩니다. **Microsoft**는 Azure microservices의 핵심 패턴으로 CQRS를 설명합니다.

## 흔한 실수들

- **본인이 쓴 내용 바로 읽기** — command 직후 UI가 즉시 쿼리하여 stale data를 받는 경우입니다. optimistic UI 업데이트나 짧은 polling retry로 완화할 수 있습니다.
- **Projector lag** — 부하가 높을 때 event bus나 projector가 뒤처질 수 있습니다. 컨슈머 지연 시간을 주요 지표로 모니터링해야 합니다.
- **Schema coupling** — 읽기 모델 스키마가 이벤트와 강하게 결합되면 이벤트 스키마 변경이 어려워집니다. 이벤트 버전을 관리하세요.
- **분산 트랜잭션의 유혹** — 개발자들이 여러 서비스에 걸친 작업을 원자적으로 처리하려고 시도하는 경우가 있습니다. 이는 CQRS의 목적에 어긋나므로 대신 Saga를 사용하세요.

> 💡
> 면접 팁
> 면접에서 CQRS는 소셜 미디어 피드처럼 읽기와 쓰기 패턴이 매우 다른 경우(복잡한 집계 읽기, 단순한 쓰기 경로)를 지원해야 할 때 자주 언급됩니다. 읽기 및 쓰기 성능의 불균형, 독립적인 확장성, 또는 여러 읽기 모델 지원 등 왜 필요한지에 초점을 맞춰 답변하세요. Eventual consistency에 따른 트레이드오프를 먼저 언급하는 것이 좋습니다.
