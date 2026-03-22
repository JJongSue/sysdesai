# Index Table 패턴

> 출처: https://www.sysdesai.com/learn/performance-scalability/index-table

---

## 문제: Primary Key 없이 쿼리하기

관계형 database에서 secondary index는 database 엔진이 자동으로 관리하는 일급 기능입니다. 하지만 **NoSQL database** (DynamoDB, Cassandra, Redis)와 **sharded database**에서는 non-primary-key 속성으로 lookup하는 것이 비용이 많이 듭니다. 데이터가 primary key로 파티션되어 있기 때문입니다. Primary key가 `user_id`일 때 email로 사용자를 가져오려면 모든 파티션을 스캔해야 합니다.

**Index Table 패턴**은 쿼리하려는 속성을 primary key로 하는 별도의 테이블을 수동으로 생성하여 이 문제를 해결합니다. 이 secondary 테이블은 쿼리 속성에서 primary key로의 매핑을 저장하여 전체 테이블 스캔 없이 O(1) lookup을 가능하게 합니다.

## Index Table 구조
Index table은 secondary key (email)를 primary key (user_id)에 매핑합니다
python

```
# Main table: users (primary key = user_id)
# DynamoDB put:
users_table.put_item(Item={
    "user_id": "u-123",
    "email": "alice@example.com",
    "name": "Alice"
})

# Index table: users_by_email (primary key = email)
# Maintained manually alongside the main table
users_by_email_table.put_item(Item={
    "email": "alice@example.com",
    "user_id": "u-123"
})

# Lookup by email:
def get_user_by_email(email: str) -> dict:
    # Step 1: Resolve primary key via index table
    index_row = users_by_email_table.get_item(
        Key={"email": email}
    ).get("Item")
    if not index_row:
        return None
    # Step 2: Fetch full record from main table
    user = users_table.get_item(
        Key={"user_id": index_row["user_id"]}
    ).get("Item")
    return user
```

## 세 가지 Index Table 변형

| 변형 | 저장 내용 | Trade-off |
| --- | --- | --- |
| Key-only index | Secondary key → primary key 매핑만 | Lookup당 두 번의 읽기; 최소한의 저장 오버헤드 |
| Covering index (비정규화) | Secondary key + 자주 쿼리되는 모든 속성 | Lookup당 한 번의 읽기; 데이터 중복; 동기화 유지 필요 |
| Composite index | 다중 속성 secondary key (예: city + last_name) | 복합 쿼리 가능; 부분 필터에 덜 유연 |

## Index Table 일관성 유지

Index table은 항상 main table의 현재 상태를 반영해야 합니다. 쓰기는 두 단계 작업이 됩니다: main table과 index table 모두에 쓰기. 이는 두 가지 consistency 과제를 도입합니다:

- **원자성**: Main table 업데이트 후 index table 업데이트 전에 충돌이 발생하면 동기화가 깨집니다. Datastore가 지원하는 경우 **transactional outbox** 또는 **two-phase commit**을 사용하세요 (DynamoDB TransactWriteItems).
- **Key 업데이트**: 인덱싱된 속성이 변경되면 (예: 사용자가 email을 변경), 이전 index 항목을 삭제하고 새 항목을 삽입해야 합니다. 여기서 실패하면 고아 index 항목이 남습니다.

> 💡
> DynamoDB Global Secondary Index
> DynamoDB의 관리형 Global Secondary Index (GSI)는 Index Table 패턴을 자동으로 구현합니다. 내부적으로 DynamoDB는 각 GSI에 대해 별도의 파티션을 유지하고 비동기적으로 쓰기를 전파합니다. Trade-off: GSI 읽기는 기본적으로 eventually consistent이며, GSI write throughput은 별도로 청구됩니다.

## Index Table vs 전문 검색

Index table은 구조화된 속성에 대한 정확한 일치 및 range 쿼리에 잘 작동합니다. **전문 검색**, prefix 매칭, 또는 fuzzy 매칭에는 전용 검색 엔진 (Elasticsearch, Algolia, Typesense)이 더 적합합니다. Index table과 검색 엔진 패턴은 상호 보완적입니다 — 많은 시스템이 둘 다 사용합니다.

> 💡
> 인터뷰 팁
> Index table은 NoSQL database를 제안하고 면접관이 '그런데 email로 사용자를 어떻게 조회하나요?'라고 물을 때마다 인터뷰에서 등장합니다. 답변은: 'email을 user_id에 매핑하는 secondary index table을 유지하겠습니다. 모든 쓰기는 두 테이블을 트랜잭션으로 업데이트합니다.' 그런 다음 consistency 문제를 선제적으로 다루세요. 이는 happy path뿐만 아니라 운영 정확성에 대해 생각한다는 것을 보여줍니다.
