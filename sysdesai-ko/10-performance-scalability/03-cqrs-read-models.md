# 성능을 위한 CQRS Read Model

> 출처: https://www.sysdesai.com/learn/performance-scalability/cqrs-read-models

---

## 공유 모델의 성능 문제

전통적인 아키텍처에서는 동일한 정규화된 database schema가 읽기와 쓰기를 모두 처리합니다. 정규화된 schema는 쓰기에 최적화되어 있습니다 — 중복을 제거하고 foreign key와 join을 통해 무결성을 강제합니다. 하지만 읽기는 일반적으로 많은 테이블을 join하고, 큰 결과 집합을 정렬하고, 데이터를 집계해야 합니다. 이러한 상충되는 요구사항은 지속적인 긴장을 만듭니다: 읽기를 위해 추가하는 모든 index는 쓰기를 느리게 하고, 쓰기에 도움이 되는 모든 정규화 결정은 읽기 성능을 저하시킵니다.

**CQRS (Command Query Responsibility Segregation)**는 write model (Command)과 read model (Query)을 완전히 분리함으로써 이 긴장을 해소합니다. 쓰기 측은 mutation에 최적화된 정규화된 무결성 강제 schema를 사용합니다. 읽기 측은 **비정규화된 projection** — UI가 답해야 하는 쿼리에 정확히 맞게 형성된 별도의 data store — 을 사용합니다.

## Read Model 아키텍처
CQRS 아키텍처: 쓰기는 정규화된 store로, event는 read model로 projection됨
Read model은 **projection** — 쓰기 측에서 파생된 데이터의 표현입니다. Command가 성공하고 상태가 변경되면, 쓰기 측은 domain event를 발행합니다. **Projection handler** (또는 projector라고도 함)는 이러한 event를 구독하고 그에 따라 read model을 업데이트합니다. Read model은 별도의 database, Redis hash, Elasticsearch index, 또는 동일한 database의 materialized view가 될 수 있습니다.

## 비정규화: 읽기를 위한 데이터 형성

CQRS read model의 강점은 **비정규화**에서 옵니다. UI가 쿼리 시점에 `orders`, `order_items`, `products`, `customers`를 join하게 하는 대신, projector가 이 데이터를 한 번 — 쓰기 시점에 — 조합하여 결과를 주문당 단일 document로 저장합니다. 그러면 쿼리는 단일 key lookup이 됩니다.

typescript

```
// Write side: normalized (source of truth)
// orders: { id, customer_id, status, created_at }
// order_items: { order_id, product_id, quantity, unit_price }
// customers: { id, name, email }
// products: { id, name, sku }

// Read model: denormalized document per order
interface OrderReadModel {
  orderId: string;
  status: string;
  createdAt: string;
  customer: {
    id: string;
    name: string;
    email: string;
  };
  items: Array<{
    productId: string;
    productName: string;
    sku: string;
    quantity: number;
    unitPrice: number;
    lineTotal: number;
  }>;
  orderTotal: number;
}

// Projector: listens to domain events and builds read model
async function handleOrderPlaced(event: OrderPlacedEvent): Promise<void> {
  const customer = await customerRepo.findById(event.customerId);
  const items = await Promise.all(
    event.items.map(async (item) => {
      const product = await productRepo.findById(item.productId);
      return {
        productId: item.productId,
        productName: product.name,
        sku: product.sku,
        quantity: item.quantity,
        unitPrice: item.unitPrice,
        lineTotal: item.quantity * item.unitPrice,
      };
    })
  );
  const orderDoc: OrderReadModel = {
    orderId: event.orderId,
    status: "placed",
    createdAt: event.occurredAt,
    customer: { id: customer.id, name: customer.name, email: customer.email },
    items,
    orderTotal: items.reduce((sum, i) => sum + i.lineTotal, 0),
  };
  await readStore.upsert("orders", event.orderId, orderDoc);
}
```

## 하나의 Write 측에서 여러 Read Model

단일 write store는 **여러 read model**로 projection될 수 있으며, 각각 다른 쿼리 패턴에 최적화됩니다. e-commerce 시스템은 다음을 가질 수 있습니다: 주문 상세 모델 (주문 ID별), 고객별 주문 모델 (날짜순 정렬), 제품 판매 모델 (제품별 집계), Elasticsearch index (전문 검색). 각각은 동일한 domain event를 구독하여 최신 상태를 유지합니다.

## Eventual Consistency와 Lag

CQRS read model의 근본적인 trade-off는 **eventual consistency**입니다. Command가 성공한 후, read model이 변경 사항을 반영하기까지 전파 지연 — 일반적으로 밀리초에서 초 단위 — 이 있습니다. 이 lag은 event 전파, projection 처리, write-to-read-store latency에서 발생합니다.

> 💡
> Read-After-Write 처리
> 사용자가 주문을 하고 즉시 확인하려 하면 stale 데이터를 볼 수 있습니다. 일반적인 완화 방법: (1) command payload를 기반으로 즉시 optimistic UI 상태를 표시합니다. (2) command의 event sequence number를 read API에 전달하고, read 측은 해당 sequence number까지 처리할 때까지 기다린 후 반환합니다. (3) 중요한 작업의 경우, command 후 짧은 시간 동안 write model에서 직접 읽습니다.

## Read Model 재구축

CQRS의 강점 중 하나는 read model이 **파생 가능**하다는 것입니다 — 처음부터 event를 재생하여 항상 재구축할 수 있습니다. 이는 언제든지 새로운 read model을 추가할 수 있고 (히스토리를 재생하여 backfill), projection 버그를 삭제하고 재구축하여 수정하고, write schema와 독립적으로 read schema를 발전시킬 수 있음을 의미합니다.

> 💡
> 인터뷰 팁
> 인터뷰에서 CQRS read model을 논의할 때 세 가지를 강조하세요: (1) 읽기와 쓰기가 독립적으로 확장됩니다 — write path를 건드리지 않고 read replica를 추가할 수 있습니다; (2) read model은 cache가 아닙니다 — 읽기의 source of truth이며, write DB가 아닌 event에서 재구축됩니다; (3) eventual consistency가 trade-off입니다. 후속 질문 '읽기-쓰기 후 consistency를 어떻게 처리하나요?'를 예상하고 구체적인 답변을 준비하세요.

| 측면 | 전통적 방식 (공유 모델) | CQRS (별도 Read Model) |
| --- | --- | --- |
| Schema 최적화 | 읽기와 쓰기 사이의 타협 | 각 측이 독립적으로 최적화 |
| 쿼리 복잡성 | 쿼리 시점에 join | 사전 join된 document, 단일 lookup |
| Consistency | 강함 | Eventual (초 단위 lag) |
| Schema 발전 | 쓰기 + 읽기가 함께 마이그레이션 | Read model이 독립적으로 재구축 |
| 확장성 | 전체 database 확장 | 읽기와 쓰기가 별도로 확장 |
