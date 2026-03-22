# SQL vs NoSQL Decision Framework (SQL 대 NoSQL 결정 프레임워크)

> Source: https://www.sysdesai.com/learn/data-storage/sql-vs-nosql

---

## The Real Question Is Not SQL vs NoSQL (진짜 질문은 SQL 대 NoSQL이 아닙니다)

SQL 대 NoSQL 논쟁은 흔히 잘못된 방식으로 프레임이 짜여 있습니다. 질문은 "어느 것이 더 나은가"가 아니라, **"어떤 데이터 모델이 이 액세스 패턴에 적합한가?"**여야 합니다. SQL과 NoSQL 데이터베이스 모두 거대한 규모로 운영될 수 있으며 고처리량의 쓰기를 처리할 수 있습니다. 결정은 데이터 모델, 쿼리 유연성, 일관성 요구 사항, 운영 제약이라는 네 가지 차원으로 귀결됩니다.

## Dimension 1: Data Model (차원 1: 데이터 모델)

다음 질문부터 시작하세요: **내 데이터는 자연적으로 어떻게 구조화되어 있는가?** 관계형 데이터는 개체(Entity) 간의 관계가 명확합니다(사용자, 주문, 제품). 문서형(Document) 데이터는 계층적이고 자기 완비적입니다(댓글이 포함된 블로그 포스트). 와이드 컬럼(Wide-column) 데이터는 이벤트 기반입니다(센서 판독값, 활동 로그). 그래프(Graph) 데이터는 연결에 의해 정의됩니다(소셜 네트워크, 사기 행위 그래프).

| Data Shape(데이터 형태) | Recommended Model(추천 모델) | Example(예시) |
| --- | --- | --- |
| 관계와 조인이 많은 개체 | Relational (PostgreSQL) | e-commerce: 사용자, 주문, 제품, 재고 |
| 계층적이고 스키마가 가변적인 문서 | Document (MongoDB) | 카테고리별로 속성이 다른 제품 카탈로그 |
| 대량의 시간 순서 이벤트 | Wide-column (Cassandra) 또는 Time-series | IoT 센서 로그, 활동 피드 |
| 복잡하게 연결된 데이터, 탐색 쿼리 | Graph (Neo4j) | 소셜 네트워크, 사기 탐지 |
| 단일 키를 통한 단순 조회 | Key-value (Redis, DynamoDB) | 세션 저장소, 피처 플래그(feature flags), 카운터 |

## Dimension 2: Query Patterns (차원 2: 쿼리 패턴)

**데이터베이스를 선택하기 전에 당신의 쿼리를 먼저 파악하세요.** SQL의 강점은 **Ad-hoc Query Flexibility(애드혹 쿼리 유연성)**입니다 — 어떤 컬럼에 대해서도 필터링, 정렬, 그룹화, 조인을 수행할 수 있습니다. NoSQL 데이터베이스는 쿼리 유연성을 희생하는 대신 **사전에 정의된 액세스 패턴에 대한 성능**을 얻습니다. 애플리케이션이 복잡하고 진화하는 분석 기능을 지원해야 한다면 SQL이 승리합니다. 상위 3개의 쿼리가 명확하고 변하지 않는다면 NoSQL이 순수 성능 면에서 우세할 수 있습니다.

- **SQL 선호**: 집계, 복잡한 필터, 개체 간 JOIN 또는 사전에 예측할 수 없는 보고용 쿼리가 필요한 경우.
- **NoSQL 선호**: 쿼리가 예측 가능하고 성능이 중요하며, 단순한 키 조회나 문서 페치(fetch)에 매핑되는 경우.
- **Wide-column 선호**: 쿼리에 항상 파티션 키가 포함되고 막대한 쓰기 처리량이 필요한 경우.

## Dimension 3: Consistency Requirements (차원 3: 일관성 요구 사항)

일부 분야에서 ACID 트랜잭션은 타협의 대상이 아닙니다. 시스템이 **돈, 재고 또는 법적 기록**을 다룬다면 강한 일관성(Strong Consistency)이 필수입니다. **소셜 피드, 분석 또는 콘텐츠 추천**을 다룬다면 최종 일관성(Eventual Consistency)으로도 충분하며 이는 거대한 확장성을 가능하게 합니다.

| Use Case(사용 사례) | Consistency Need(일관성 요구 사항) | Recommended DB(추천 DB) |
| --- | --- | --- |
| 은행 송금, 이중 지불 방지 | Strong (ACID) | PostgreSQL, MySQL |
| 비행기 좌석 예약 | Strong (Serializable) | `SELECT FOR UPDATE`를 사용하는 PostgreSQL |
| 사용자 프로필 업데이트 | Read-your-writes로 충분함 | MongoDB (Majority Write Concern 사용) |
| 소셜 미디어 좋아요 카운터 | Eventual | Cassandra, Redis INCR |
| 제품 카탈로그 읽기 | Eventual | DynamoDB, MongoDB |

## Dimension 4: Scale Requirements (차원 4: 확장성 요구 사항)

**가지고 있지 않은 규모를 위해 과잉 설계(over-engineer)하지 마세요.** 적절한 인덱싱을 갖춘 PostgreSQL은 100,000 QPS를 처리할 수 있습니다. Amazon은 PostgreSQL 기반의 Aurora를 거대한 규모로 사용합니다. NoSQL 데이터베이스가 자동으로 "더 확장성이 좋은" 것은 아닙니다 — 단순히 일관성과 쿼리 유연성을 대가로 수평 확장을 가능하게 하는 다른 트레이드오프를 선택한 것일 뿐입니다.

> ℹ️
> SQL로 시작하고 필요할 때 마이그레이션하세요 (Start with SQL, Migrate if Needed)
> 새로운 시스템을 위한 가장 실용적인 조언: PostgreSQL로 시작하세요. 검증되었고 도구 생태계가 훌륭하며, 반구조화된 데이터를 위한 JSON 컬럼을 지원하고, 대부분의 팀이 생각하는 것보다 훨씬 더 큰 규모까지 확장 가능합니다. SQL이 샤딩과 읽기 복제본으로도 해결할 수 없는 구체적인 병목 지점을 발견했을 때만 NoSQL로 마이그레이션하세요.

## Decision Flowchart (결정 플로우차트)
데이터베이스 선택을 위한 실용적인 결정 플로우차트

## Using Both: Polyglot Persistence (둘 다 사용하기: 폴리글랏 퍼시스턴스)

운영 시스템에서 단일 데이터베이스만 사용하는 경우는 드뭅니다. **Polyglot Persistence(폴리글랏 퍼시스턴스)**는 시스템의 각 부분에 대해 그 강점에 맞는 서로 다른 데이터베이스를 사용하는 것을 의미합니다. 전형적인 e-commerce 스택은 주문 처리에 PostgreSQL, 세션과 장바구니에 Redis, 제품 검색에 Elasticsearch, 사용자 활동 피드에 Cassandra를 사용할 수 있습니다. 각 데이터베이스는 특정 사용 사례에 맞춰 선택됩니다.

> 💡
> Interview Tip (인터뷰 팁)
> 시스템 디자인 인터뷰에서 데이터베이스를 선택한 후에는 항상 그 선택을 정당화하세요. 마법의 문장은 다음과 같습니다: '주요 액세스 패턴이 Y이고 Z 일관성이 필요하기 때문에 X를 선택했습니다.' 예를 들어: '활동 피드에는 Cassandra를 선택하겠습니다. 주요 액세스 패턴이 user_id 기준 시간 순서 읽기이고 초당 수백만 개의 이벤트를 기록해야 하며, 사용자가 자신의 게시물을 즉시 보지 않아도 되므로 최종 일관성이 허용되기 때문입니다.' 이는 암기된 답변이 아닌 신중한 논리적 사고를 보여줍니다.
