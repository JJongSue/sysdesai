# Materialized View Pattern

> 출처: https://www.sysdesai.com/learn/data-management-patterns/materialized-view

---

## Materialized View란 무엇일까?

**Materialized View**는 쿼리 결과를 미리 계산하여 저장해둔 것입니다. 접근할 때마다 기반 쿼리를 다시 실행하는 일반적인 SQL view와 달리, Materialized View는 디스크에 물리적으로 저장됩니다. 쿼리를 실행할 때 데이터가 이미 계산되어 있으므로 joins, aggregations, full table scans 과정이 필요 없습니다.

이 패턴은 SQL 데이터베이스에만 국한되지 않습니다. 분산 시스템에서 Materialized View는 CQRS 아키텍처의 **read-side projections**, 집계 데이터가 미리 채워진 **Redis caches**, 여러 소스 테이블의 데이터를 denormalize한 **Elasticsearch indexes**, 특정 쿼리 패턴을 위해 설계된 **Cassandra tables** 등의 형태로 나타납니다.

Materialized View는 여러 소스 테이블을 미리 join하여 하나의 빠른 접근용 projection으로 만듭니다.

## Refresh Strategies

Materialized View의 핵심 트레이드오프는 데이터의 **freshness**와 **cost** 사이의 균형입니다. 상황에 맞는 갱신 빈도를 선택해야 합니다.

| 전략 | 작동 방식 | Staleness | 유스케이스 |
| --- | --- | --- | --- |
| Full refresh | 정해진 일정에 따라 전체 뷰를 다시 계산 | 일정 간격만큼 발생 | 야간 보고서, 배치 분석 |
| Incremental refresh | 마지막 갱신 이후 변경된 행만 반영 | 일정 간격만큼 발생 | 자주 변경되는 대규모 뷰 |
| On-write (synchronous) | 기본 테이블 쓰기와 같은 transaction 내에서 뷰 업데이트 | 없음 (항상 최신) | 소규모 뷰, 엄격한 일관성 필요 |
| Event-driven (async) | 변경 이벤트를 구독하여 비동기적으로 뷰 업데이트 | 밀리초에서 초 단위 | CQRS read projections, CDC 기반 인덱스 |
| On-demand (lazy) | 쿼리 시점에 데이터가 오래되었고 TTL이 만료된 경우에만 재구축 | TTL만큼 발생 | 드물게 액세스되는 보고용 뷰 |

## CQRS에서의 Materialized View

CQRS 아키텍처에서 읽기 모델은 곧 **Materialized View**라고 할 수 있습니다. command가 이벤트를 생성할 때마다 projector가 하나 이상의 읽기 모델 테이블을 업데이트합니다. 이를 통해 각 읽기 테이블을 특정 쿼리에 맞게 설계할 수 있어 쿼리 시점에 join을 할 필요가 없습니다.

📌
Cassandra와 Materialized View 패턴

Cassandra에서는 데이터를 쿼리 형태와 정확히 일치하게 저장해야 합니다. join 기능이 없기 때문입니다. 그래서 같은 데이터를 여러 Cassandra 테이블에 저장하곤 합니다. 하나는 `userId`로, 다른 하나는 `orderId`로, 또 다른 하나는 `status`로 파티션을 나누는 식입니다. 각각은 동일한 논리적 데이터의 Materialized View이며 서로 다른 쿼리에 최적화되어 있습니다. 이를 **table-per-query** 방식이라고 부르며 Cassandra의 전형적인 설계 방식입니다.

## SQL Materialized Views 예시

```sql
-- 주문 요약 Materialized View 생성
CREATE MATERIALIZED VIEW order_summary AS
  SELECT
    o.id         AS order_id,
    u.name       AS customer_name,
    COUNT(i.id)  AS item_count,
    SUM(i.price * i.quantity) AS total,
    o.status,
    o.created_at
  FROM orders o
  JOIN users u ON o.user_id = u.id
  JOIN order_items i ON i.order_id = o.id
  GROUP BY o.id, u.name, o.status, o.created_at;

-- Materialized View 갱신 (PostgreSQL)
REFRESH MATERIALIZED VIEW CONCURRENTLY order_summary;

-- 이제 쿼리는 단일 테이블 스캔으로 매우 빠릅니다.
SELECT * FROM order_summary WHERE order_id = 12345;
```

## 일관성 고려 사항

> ⚠️
> Stale Reads는 피할 수 없습니다.
> 소스 쓰기와 동일한 transaction에서 동기적으로 갱신되지 않는 모든 Materialized View는 데이터가 오래될 수 있습니다. 애플리케이션은 이를 허용하도록 설계되어야 합니다. 분석용 뷰의 UI에는 마지막 업데이트 시간을 표시하고, 실시간에 가까운 데이터가 필요하다면 event-driven 방식을 사용하세요.

Materialized View가 무효화되었을 때(소스 데이터 변경) 세 가지 선택지가 있습니다. **mark-and-recompute** (오래된 것으로 표시하고 다음 읽기 때 계산), **eager refresh** (백그라운드에서 즉시 다시 계산), 혹은 **serve stale** (캐시 제어 힌트와 함께 이전 값을 반환)입니다. 사용자가 얼마나 오래된 데이터를 허용할 수 있는지에 따라 적절한 방식을 선택하세요.

## 실제 활용 사례

- **Twitter/X**는 팔로워 타임라인을 Redis에 Materialized View로 미리 계산해둡니다. 타임라인을 가져오는 작업은 수만 명의 팔로우 관계를 join하는 게 아니라 빠른 리스트 읽기 작업이 됩니다.
- **Google BigQuery**는 Materialized View를 자동으로 관리하고 소스 데이터가 변하면 알아서 갱신합니다. 이때 변경된 부분에 대한 계산 비용만 청구합니다.
- **Airbnb**는 Postgres의 숙소 데이터를 Elasticsearch에 Materialized View 형태로 유지합니다. 덕분에 관계형 DB에서는 성능이 떨어지는 전체 텍스트 검색이나 geo-queries를 빠르게 처리합니다.
- **Shopify**는 상점 검색을 위해 Elasticsearch에서 denormalize된 상품 뷰를 관리하며 CDC를 통해 Postgres 소스와 동기화합니다.

> 💡
> 면접 팁
> 면접관들은 읽기 성능 문제를 해결하는 방법으로 Materialized View를 좋아합니다. 시스템의 hot paths에서 복잡한 join 쿼리가 느리다면 CDC나 event-driven projection으로 관리되는 Materialized View를 제안해보세요. 이때 갱신 전략과 staleness 트레이드오프를 꼭 설명해야 합니다. 뷰의 신선도가 비즈니스 요구사항에 맞아야 한다는 점을 강조하세요. 분석 데이터는 몇 분의 지연도 괜찮지만 사용자의 잔액 정보는 그러면 안 되기 때문입니다.
