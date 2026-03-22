# Relational Databases(관계형 데이터베이스) & ACID

> Source: https://www.sysdesai.com/learn/data-storage/relational-databases

---

## Why Relational Databases Still Dominate (관계형 데이터베이스가 여전히 우세한 이유)

Relational Databases(관계형 데이터베이스)는 50년 넘게 소프트웨어 시스템의 중추 역할을 해왔습니다. NoSQL의 유행에도 불구하고, 뱅킹, e-commerce, SaaS 등 대부분의 운영 트랜잭션 시스템은 여전히 **PostgreSQL**, **MySQL**, 또는 **SQL Server**에 의존합니다. 그 이유를 이해하려면 관계형 데이터베이스가 보장하는 **ACID** 속성을 이해해야 합니다.

## ACID Properties (ACID 속성)

ACID는 **Atomicity(원자성)**, **Consistency(일관성)**, **Isolation(고립성)**, **Durability(지속성)**의 약자입니다. 이 네 가지 보장은 관계형 데이터베이스에서 *transaction(트랜잭션)*이 의미하는 바를 정의합니다.

| Property(속성) | Meaning(의미) | Example(예시) |
| --- | --- | --- |
| Atomicity(원자성) | 트랜잭션 내의 모든 작업이 성공하거나 모두 롤백됩니다. 부분적인 상태는 허용되지 않습니다. | 은행 이체 시 한 계좌에서 출금되고 다른 계좌로 입금되는 과정 — 두 작업이 모두 일어나거나 아예 일어나지 않아야 합니다. |
| Consistency(일관성) | 트랜잭션은 모든 제약 조건을 준수하며 데이터베이스를 하나의 유효한 상태에서 다른 유효한 상태로 바꿉니다. | `NOT NULL` 제약 조건이나 Foreign Key(외래 키)가 강제되며, 유효하지 않은 데이터는 거부됩니다. |
| Isolation(고립성) | 동시에 실행되는 트랜잭션들이 마치 순차적으로 실행되는 것처럼 동작합니다. Dirty Read나 Phantom Read가 발생하지 않습니다. | 두 사용자가 비행기의 마지막 좌석을 예약할 때 — 오직 한 명만 성공합니다. |
| Durability(지속성) | 트랜잭션이 Commit(커밋)되면, 데이터는 시스템 장애(WAL을 통해 디스크에 기록됨)에도 살아남습니다. | `COMMIT` 이후에는 서버가 다운되더라도 주문 데이터가 손실되지 않습니다. |

### Isolation Levels (고립 수준)

완전한 Serializable(직렬화 가능) 고립은 비용이 많이 듭니다. SQL 데이터베이스는 성능과 정확성을 맞바꾼 다양한 Isolation Level(고립 수준)을 제공합니다. 인터뷰에서 이를 이해하는 것은 매우 중요합니다.

| Isolation Level(고립 수준) | Dirty Read | Non-Repeatable Read | Phantom Read |
| --- | --- | --- | --- |
| Read Uncommitted | 가능 | 가능 | 가능 |
| Read Committed (PG 기본값) | 방지됨 | 가능 | 가능 |
| Repeatable Read | 방지됨 | 방지됨 | 가능 |
| Serializable | 방지됨 | 방지됨 | 방지됨 |

> 💡
> PostgreSQL 기본값
> PostgreSQL은 기본적으로 **Read Committed** 고립 수준을 사용합니다. 엄격한 정확성이 필요한 금융 애플리케이션의 경우, 명시적으로 `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE`을 사용하세요.

## Indexing Strategies (인덱싱 전략)

Index(인덱스)는 관계형 데이터베이스에서 쿼리 성능을 높이는 가장 큰 수단입니다. 인덱스가 없으면 모든 쿼리는 **Full Table Scan(전체 테이블 스캔)**을 수행하며 O(n)의 비용이 발생합니다. 적절한 인덱스를 사용하면 B-tree를 통해 조회가 O(log n)이 됩니다.

### B-Tree Indexes (B-Tree 인덱스)

PostgreSQL과 MySQL의 기본 인덱스 유형입니다. B-tree는 데이터를 정렬된 순서로 저장하여 **Range Query(범위 쿼리)** (`WHERE created_at > '2024-01-01'`), **Equality Lookup(동등 조회)**, 그리고 추가 정렬 단계 없는 **ORDER BY** 작업을 가능하게 합니다. 이 트리는 수억 개의 행이 있는 테이블에서도 보통 3~4개 수준의 깊이를 가집니다.

### Hash Indexes (해시 인덱스)

Hash Index(해시 인덱스)는 O(1) 동등 조회를 제공하지만 **Range Query(범위 쿼리)를 지원할 수 없습니다**. 정확히 일치하는 워크로드에만 사용하세요. PostgreSQL은 해시 인덱스를 기본적으로 지원하며, MySQL에서는 MEMORY 스토리지 엔진에서만 존재합니다.

### Composite and Covering Indexes (복합 및 커버링 인덱스)

**Composite Index(복합 인덱스)**는 여러 컬럼을 포함합니다. 순서가 중요합니다: `(user_id, created_at)`에 대한 인덱스는 `user_id`만으로 필터링하거나 `(user_id, created_at)`을 같이 사용하는 쿼리는 지원하지만, `created_at`만 사용하는 쿼리는 지원하지 **않습니다**. **Covering Index(커버링 인덱스)**는 쿼리에 필요한 모든 컬럼을 포함하여 데이터베이스가 메인 테이블에 접근할 필요가 없게 합니다 — 이를 Index-only Scan(인덱스 전용 스캔)이라고도 부릅니다.

```sql
-- Composite index: supports WHERE user_id = ? AND created_at > ?
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- Covering index: query can be answered entirely from the index
CREATE INDEX idx_orders_covering
ON orders(user_id, created_at)
INCLUDE (total_amount, status);

-- EXPLAIN to verify index usage
EXPLAIN ANALYZE
SELECT total_amount, status
FROM orders
WHERE user_id = 42 AND created_at > '2024-01-01';
```

## Normalization (정규화)

**Normalization(정규화)**는 테이블을 더 작게 분해하고 Foreign Key(외래 키)를 사용해 Join(조인)함으로써 중복을 제거합니다. 표준 형식으로는 1NF, 2NF, 3NF, BCNF가 있습니다. 대부분의 운영 스키마는 **3NF**를 목표로 합니다: 각 기본 키가 아닌 컬럼은 전체 기본 키에만 의존하며 그 외에는 의존하지 않습니다.

> ℹ️
> Normalization(정규화) vs Denormalization(비정규화)
> 정규화는 쓰기 일관성을 향상시키지만 조인 복잡성을 증가시킵니다. 대규모 환경에서 팀들은 종종 비싼 조인을 피하기 위해 데이터를 중복시키는 **Denormalization(비정규화)**를 선택합니다 — 쓰기 오버헤드를 감수하고 읽기 속도를 높이는 것입니다. 이는 우연이 아닌 의도적인 아키텍처 결정입니다.

## When to Choose a Relational Database (관계형 데이터베이스를 선택해야 할 때)

- **Complex Relationships(복잡한 관계)**: Foreign Key(외래 키) 제약 조건과 Join(조인)이 필요한 여러 엔티티 유형 (users → orders → products → inventory).
- **ACID Transactions Required(ACID 트랜잭션 필요)**: 금융 작업, 재고 관리 등 부분적인 쓰기가 치명적인 모든 경우.
- **Flexible Ad-hoc Queries(유연한 애드혹 쿼리)**: 모든 액세스 패턴을 미리 알 수 없고 SQL의 표현력이 필요한 경우.
- **Regulatory Compliance(규제 준수)**: 데이터베이스 엔진에 의해 감사 가능성, 제약 조건, 참조 무결성이 강제되어야 하는 경우.
- **Moderate Scale(적당한 규모)**: PostgreSQL은 적절한 인덱싱과 Connection Pooling(커넥션 풀링)을 통해 초당 수십만 개의 쿼리(QPS)를 처리할 수 있습니다.

> 💡
> Interview Tip (인터뷰 팁)
> 면접관이 시스템의 데이터 저장소에 대해 물으면 항상 "ACID 요구 사항이 무엇인가요?"로 시작하세요. 강력한 일관성과 관계적 무결성이 중요하다면(결제, 예약, 사용자 계정) PostgreSQL을 기본으로 선택하세요. 그런 다음 그 기준에서 벗어나는 경우 정당성을 부여하세요. 면접관들은 단순히 '확장성'이 있어 보인다는 이유로 NoSQL을 선택하는 것보다 사려 깊은 선택을 하는 것에 더 깊은 인상을 받습니다.

## Query Optimization Basics (쿼리 최적화 기초)

Query Optimizer(쿼리 옵티마이저)는 쿼리를 실행하기 위한 일련의 작업 순서인 **Execution Plan(실행 계획)**을 생성합니다. `EXPLAIN ANALYZE`를 사용하여 계획을 검사하세요. 주요 확인 사항: **Seq Scan**(대규모 테이블에서 주의), **Index Scan**(좋음), **Hash Join** 대 **Nested Loop**(테이블 크기에 따라 다름), 인덱스로 제거할 수 있는 **Sort Operation(정렬 작업)** 등입니다.

> ⚠️
> N+1 Query Problem (N+1 쿼리 문제)
> 가장 흔한 ORM 관련 성능 저하 요인입니다. 100명의 사용자를 가져온 다음 그들의 프로필을 위해 100개의 별도 쿼리를 실행하는 것은 O(n)번의 데이터베이스 라운드 트립을 발생시킵니다. 항상 JOIN이나 Eager Loading (`SELECT ... JOIN`)을 사용하세요. 코드 리뷰와 인터뷰에서 이를 포착하는 것이 시니어 엔지니어의 자질을 보여주는 척도입니다.
