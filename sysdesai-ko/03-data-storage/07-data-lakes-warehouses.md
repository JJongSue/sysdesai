# Data Lakes & Data Warehouses (데이터 레이크 및 데이터 웨어하우스)

> Source: https://www.sysdesai.com/learn/data-storage/data-lakes-warehouses

---

## OLTP vs OLAP: Two Different Worlds (OLTP 대 OLAP: 두 개의 다른 세계)

운영 데이터베이스(PostgreSQL, MySQL, DynamoDB)는 **OLTP** — Online Transaction Processing(온라인 트랜잭션 처리) 시스템입니다. 이들은 개별 행에 대한 낮은 지연 시간과 높은 동시성의 읽기/쓰기에 최적화되어 있습니다. 분석 시스템은 **OLAP** — Online Analytical Processing(온라인 분석 처리) 시스템입니다. 이들은 "지난 분기 지역별 매출은 얼마인가?"와 같은 비즈니스 질문에 답하기 위해 수십억 개의 행을 스캔하는 복잡한 쿼리를 실행합니다. 이 두 워크로드는 근본적으로 호환되지 않는 액세스 패턴을 가지며 별도의 시스템을 필요로 합니다.

| Dimension(차원) | OLTP | OLAP |
| --- | --- | --- |
| Optimized for(최적화 대상) | 낮은 지연 시간의 행 읽기/쓰기 | 높은 처리량의 컬럼 스캔 |
| Query type(쿼리 유형) | 단순 조회, 삽입 | 복잡한 집계, JOIN |
| Data volume(데이터 볼륨) | GB ~ TB | TB ~ PB |
| Storage layout(저장 방식) | Row-oriented(행 지향) | Column-oriented(열 지향) |
| Examples(예시) | PostgreSQL, MySQL, DynamoDB | Redshift, BigQuery, Snowflake |

## Data Warehouses (데이터 웨어하우스)

**Data Warehouse(데이터 웨어하우스)**는 여러 OLTP 소스로부터 구조화된 schema-on-write 데이터를 저장합니다. 데이터는 **ETL (Extract, Transform, Load)** 파이프라인을 통해 정제되고 변환되어 로드됩니다. 웨어하우스는 **Columnar Storage(컬럼형 저장소)**를 사용합니다 — `revenue` 컬럼만 스캔하는 쿼리는 해당 컬럼의 데이터 블록만 읽고 다른 모든 컬럼은 건너뜁니다. 이는 행 지향 데이터베이스에 비해 분석 쿼리 속도를 10~100배 향상시킵니다.

- **Amazon Redshift**: 컬럼형 MPP (Massively Parallel Processing) 데이터베이스입니다. 노드들이 쿼리 실행을 분산 처리합니다. Sort Key와 Distribution Key가 데이터의 물리적 배치를 제어합니다.
- **Google BigQuery**: 서버리스 서비스이며 스캔한 바이트 수에 따라 쿼리당 비용을 지불합니다. Dremel 쿼리 엔진을 사용합니다. 페타바이트급 데이터셋에 대한 애드혹 분석에 탁월합니다.
- **Snowflake**: 클라우드에 구애받지 않으며 연산(Compute)과 저장(Storage)을 분리합니다. 연산 능력을 독립적으로 확장/축소할 수 있습니다. 멀티 클라우드 및 멀티 리전 분석에 인기가 많습니다.

## Data Lakes (데이터 레이크)

**Data Lake(데이터 레이크)**는 처리되지 않은 원시 데이터를 원래 형식(JSON 로그, Parquet 파일, CSV 엑스포트, 이미지, 비디오 등) 그대로 저렴한 Object Storage(객체 스토리지, S3, GCS, Azure Data Lake)에 저장합니다. 웨어하우스와 달리 데이터 레이크는 **Schema-on-read** 방식을 사용합니다 — 데이터가 저장될 때가 아니라 쿼리될 때 구조가 적용됩니다. 이는 아직 어떻게 사용할지 모르는 데이터도 일단 저장할 수 있게 해줍니다.

## ETL vs ELT

**ETL (Extract, Transform, Load)**은 데이터를 웨어하우스에 로드하기 전에 변환합니다. 이는 웨어하우스가 비싸고 저장 공간이 제한적이었을 때 — 정제되고 유용한 데이터만 로드해야 했던 시절에 필요했습니다. **ELT (Extract, Load, Transform)**는 원시 데이터를 먼저 로드한 다음, 웨어하우스 자체의 연산 능력을 사용해 변환합니다. 현대의 클라우드 웨어하우스는 충분히 강력하고 저렴하기 때문에 이제는 ELT가 선호됩니다 — 미래의 재처리를 위해 원시 데이터를 유지할 수 있기 때문입니다.

> ℹ️
> dbt: ELT의 표준
> **dbt (Data Build Tool)**는 업계 표준 ELT 변환 레이어입니다. 데이터 엔지니어는 SQL 변환 모델을 작성하고, dbt는 이를 컴파일하여 데이터 웨어하우스 내부에서 실행합니다. 의존성 순서 관리, 테스트, 문서화 및 증분 빌드를 처리합니다. 누군가 '현대적 데이터 스택(modern data stack)'을 언급한다면, dbt는 거의 항상 그 일부입니다.

## The Lakehouse Pattern (레이크하우스 패턴)

데이터 레이크와 웨어하우스는 각각 약점이 있습니다: 레이크는 ACID 트랜잭션과 쿼리 최적화가 부족하고, 웨어하우스는 비싸고 경직되어 있습니다. **Lakehouse(레이크하우스)** 패턴은 이 둘을 결합합니다 — 오픈 테이블 포맷(**Delta Lake**, **Apache Iceberg**, **Apache Hudi**)이 데이터 레이크 스토리지에 ACID 트랜잭션, 스키마 진화(Schema Evolution), 타임 트래블(Time Travel) 기능을 추가합니다. **Databricks**나 **Spark** 같은 연산 엔진은 이러한 테이블을 웨어하우스와 유사한 성능으로 쿼리합니다.

| Pattern(패턴) | Strength(강점) | Weakness(약점) |
| --- | --- | --- |
| Data Warehouse | 빠른 SQL 쿼리, 통제된 스키마, BI 친화적 | 대규모 환경에서 비용 증가, 경직된 스키마, ML용 원시 데이터에 부적합 |
| Data Lake | 저렴한 저장 비용, 원시 데이터 유지, ML 친화적 | ACID 부재, 최적화 없는 느린 쿼리, 스키마 혼돈 |
| Lakehouse | 저렴한 저장소 위에서의 ACID, BI + ML 통합 | 비교적 새로운 생태계, 더 높은 운영 복잡성 |

> 💡
> Interview Tip (인터뷰 팁)
> 분석 아키텍처 관련 질문은 시니어 시스템 디자인 인터뷰, 특히 데이터 플랫폼이나 분석 관련 직군에서 자주 등장합니다. 전달해야 할 핵심 포인트는 다음과 같습니다: "운영 OLTP 데이터베이스에서 직접 분석을 수행하지 않겠습니다. 이는 사용자 트래픽과 경합을 일으킬 수 있기 때문입니다. 대신 운영 데이터베이스에서 야간 ETL이나 실시간 CDC (Change Data Capture)를 통해 데이터를 공급받는 별도의 데이터 웨어하우스를 구축하여, 운영 환경에 영향을 주지 않고 분석가가 웨어하우스를 쿼리하도록 하겠습니다." 이것이 면접관들이 듣고 싶어 하는 정답입니다.
