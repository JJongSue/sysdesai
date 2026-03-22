# NoSQL Database Types (NoSQL 데이터베이스 유형)

> Source: https://www.sysdesai.com/learn/data-storage/nosql-types

---

## The NoSQL Landscape (NoSQL의 지형)

"NoSQL"은 관계형 테이블 모델을 사용하지 않는 데이터베이스를 통칭하는 용어입니다. 이 용어는 다소 오해의 소지가 있습니다 — 이 데이터베이스들은 무엇이 *결여*되었느냐가 아니라, 특정 액세스 패턴에 최적화된 서로 다른 **Data Model(데이터 모델)**로 정의되기 때문입니다. 네 가지 주요 제품군이 있으며, 각각 근본적으로 다른 강점을 가지고 있습니다.

## 1. Key-Value Stores (키-값 저장소)

가장 단순한 NoSQL 모델인 분산 해시 맵입니다. 모든 레코드는 값(blob, 문자열 또는 구조화된 데이터)에 매핑된 키(문자열)입니다. **Redis**와 **DynamoDB**가 가장 일반적입니다. 조회는 O(1)로 매우 빠릅니다. 단점은 키로만 쿼리할 수 있다는 것입니다 — 값 필드로 필터링하는 개념이 없습니다.

| Database(데이터베이스) | Value Types(값 유형) | Best For(용도) | Persistence(지속성) |
| --- | --- | --- | --- |
| Redis | 문자열, 리스트, 셋, 해시, 정렬된 셋, 스트림 | 캐싱, 세션, 리더보드, Rate Limiting(속도 제한) | 선택 사항 (RDB 스냅샷 + AOF 로그) |
| DynamoDB | 파티션 + 정렬 키가 있는 JSON 문서 | 고처리량, 서버리스, 싱글 테이블 디자인 | 항상 보장 (SSD 기반) |
| etcd | 바이트(문자열) | 분산 설정, Service Discovery(서비스 탐색, Kubernetes) | 보장 (Raft 합의 알고리즘) |

## 2. Document Stores (문서 저장소)

문서 데이터베이스는 **자기 서술형 JSON (또는 BSON) 문서**를 저장합니다. Key-Value Stores(키-값 저장소)와 달리 문서 내부의 모든 필드를 쿼리하고 인덱싱할 수 있습니다. **MongoDB**가 대표적인 예입니다. 문서는 중첩될 수 있어 — 주문 문서가 라인 아이템을 포함할 수 있음 — 조인 없이 한 번의 읽기로 전체 엔티티를 가져올 수 있습니다.

```json
{
  "_id": "order_7821",
  "user_id": "user_42",
  "status": "shipped",
  "created_at": "2024-03-15T10:30:00Z",
  "line_items": [
    { "sku": "WIDGET-A", "qty": 2, "price": 9.99 },
    { "sku": "GADGET-B", "qty": 1, "price": 24.99 }
  ],
  "shipping_address": {
    "city": "New York",
    "zip": "10001"
  }
}
```

문서 모델은 데이터가 **Hierarchical(계층적)**이고 엔티티 경계가 명확할 때 빛을 발합니다. 주요 위험은 **Denormalization Drift(비정규화 드리프트)**입니다 — 만약 라인 아이템이 재고 문서에도 나타난다면, 가격을 업데이트할 때 여러 문서를 업데이트해야 합니다 (Foreign Key 제약 조건이 없음).

## 3. Wide-Column Stores (와이드 컬럼 저장소)

Wide-Column Stores (**Cassandra**, **HBase**, **Bigtable**)는 데이터를 행과 동적 컬럼으로 구성합니다. 모든 행이 동일한 컬럼을 갖는 관계형 테이블과 달리, 와이드 컬럼 저장소는 각 행이 서로 다른 컬럼 집합을 가질 수 있도록 허용합니다. 핵심 설계 개념은 **Partition Key(파티션 키)**입니다 — 파티션 키에 해당하는 모든 데이터는 동일한 노드에 거주하므로, 단일 파티션 쿼리가 매우 빠릅니다.

Cassandra는 높은 카디널리티를 가진 **Write-heavy, Append-heavy workload(쓰기 집약적, 추가 집약적 워크로드)**에 최적화되어 있습니다. 내부적으로 **LSM(Log-Structured Merge) Tree**를 사용하므로 쓰기가 매우 빠릅니다(순차적 디스크 쓰기). 단점은 읽기가 느릴 수 있고, Secondary Index(보조 인덱스) 없이는 파티션 간 쿼리가 비싸거나 불가능할 수 있다는 점입니다.

> ⚠️
> 데이터가 아닌 쿼리를 모델링하세요 (Model Your Queries, Not Your Data)
> Cassandra에서는 관계형 정규화와 반대로 쿼리 패턴에 맞춰 테이블을 설계합니다. 사용자 활동을 사용자별 그리고 날짜별로 가져와야 한다면, 두 개의 별도 테이블이 필요할 수 있습니다. 나중에 액세스 패턴을 변경하려면 데이터 마이그레이션이 필요합니다.

## 4. Graph Databases (그래프 데이터베이스)

Graph Databases (**Neo4j**, **Amazon Neptune**)는 데이터를 **Node(노드, 엔티티)**와 **Edge(엣지, 관계)**로 모델링하며, 양쪽 모두에 속성을 가질 수 있습니다. 이들은 관계를 탐색하는 쿼리에 탁월합니다: "친구의 친구", "사용자 간의 최단 경로", "X를 구매한 고객들이 구매한 모든 제품". 이러한 쿼리는 SQL(재귀적 CTE)에서는 매우 어렵고 문서 저장소에서는 사실상 불가능합니다.

```cypher
// Neo4j Cypher: 같은 밴드를 좋아하는 친구의 친구 찾기
MATCH (me:User {id: 'user_42'})
      -[:FOLLOWS]->(:User)
      -[:FOLLOWS]->(fof:User)
      -[:LIKES]->(b:Band {name: 'Radiohead'})
WHERE NOT (me)-[:FOLLOWS]->(fof)
RETURN fof.name, COUNT(*) AS mutual_friends
ORDER BY mutual_friends DESC
LIMIT 10;
```

## Quick Reference: Choosing a NoSQL Type (빠른 참조: NoSQL 유형 선택)

| Use Case(사용 사례) | Best NoSQL Type(최적 NoSQL 유형) | Example(예시) |
| --- | --- | --- |
| 사용자 세션, Rate Limiting(속도 제한) | Key-Value (Redis) | JWT 토큰 → 사용자 데이터 저장 |
| 제품 카탈로그, 콘텐츠 CMS | Document (MongoDB) | 제품 유형별 유연한 스키마 |
| 시계열 이벤트, IoT 원격 측정 | Wide-Column (Cassandra) | 디바이스별로 파티셔닝된 센서 판독값 |
| 소셜 그래프, 추천 | Graph (Neo4j) | 친구 추천, 사기 행위 네트워크(fraud rings) |
| 장바구니 (단일 사용자) | Key-Value 또는 Document | user_id를 키로 하는 장바구니 |

> 💡
> Interview Tip (인터뷰 팁)
> 면접관들은 '왜 모든 것에 MongoDB를 사용하지 않나요?'라는 질문을 좋아합니다. 명확한 답변: 문서 저장소는 엔티티들이 서로를 심하게 참조할 때(소셜 피드, 멀티 테넌트 권한) 불이익을 줍니다. 결국 애플리케이션 코드에서 조인을 다시 구현하게 됩니다. 적절한 도구를 사용하세요 — 일시적인 조회에는 Key-Value, 시계열 및 추가 전용 로그에는 Wide-Column, 관계 탐색이 주요 쿼리 패턴일 때는 Graph를 사용하세요.
