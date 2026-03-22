# Time-Series Databases (시계열 데이터베이스)

> Source: https://www.sysdesai.com/learn/data-storage/time-series-databases

---

## What Is Time-Series Data? (시계열 데이터란 무엇인가?)

**Time-series data(시계열 데이터)**는 시간에 따라 색인된 일련의 측정값입니다. 모든 데이터 포인트는 타임스탬프, 메트릭 이름, 값 그리고 선택적인 태그(메타데이터)를 가집니다. 예로는 10초마다의 서버 CPU 사용량, 밀리초 단위의 주식 가격, 분 단위의 IoT 센서 판독값, 시간에 따라 추적된 애플리케이션 요청 지연 시간 등이 있습니다. 가장 결정적인 특징은 **쓰기가 항상 새로운 타임스탬프를 추가하며, 오래된 데이터는 거의 업데이트되거나 삭제되지 않는다**는 점입니다.

## Why Not Use a General-Purpose Database? (왜 범용 데이터베이스를 사용하지 않는가?)

PostgreSQL에 시계열 데이터를 저장할 수 *있지만*, 규모가 커지면 고통스러워집니다. 초당 100만 개의 메트릭을 기록하는 모니터링 시스템을 생각해 보세요. PostgreSQL에서는 급격히 성장하는 테이블에 대한 B-tree 인덱스 유지 관리 비용이 매우 비싸지고, 특정 시간 윈도우에 대한 범위 스캔 성능이 저하되며, 수십억 개의 행을 저장하려면 지속적인 유지 관리(VACUUM, 인덱스 비대화 해결)가 필요합니다. 시계열 데이터베이스는 목적에 맞게 설계된 스토리지 엔진으로 이러한 문제들을 해결합니다.

## Key Time-Series Databases (주요 시계열 데이터베이스)

| Database(데이터베이스) | Model(모델) | Best For(용도) | Notes(참고) |
| --- | --- | --- | --- |
| InfluxDB | 커스텀 TSM 엔진 | DevOps 메트릭, IoT, 실시간 분석 | Flux 쿼리 언어; 기본 Retention Policy(보존 정책) 지원 |
| TimescaleDB | PostgreSQL 확장 (hypertables) | SQL 친숙도, 기존 PG 스택 활용 | 표준 SQL, 관계형 데이터와의 JOIN, 자동 파티셔닝 |
| Prometheus | Pull 방식, 인메모리 TSDB | Kubernetes / 인프라 모니터링 | 짧은 보존 기간; 장기 저장을 위해 Thanos/Cortex와 함께 사용 |
| Apache Druid | 컬럼형, 사전 집계 | 대규모 이벤트 스트림에 대한 초단위 분석 | Lyft, Airbnb에서 사용자 대상 분석에 사용 |
| Amazon Timestream | 서버리스 AWS TSDB | AWS 네이티브 IoT/운영 모니터링 | 메모리 → SSD → S3 자동 계층화 |

## Internal Architecture: Why TSDBs Are Fast (내부 아키텍처: TSDB가 빠른 이유)

시계열 데이터베이스는 **Sequential Writes(순차적 쓰기)** (항상 최신 시간 윈도우에 추가)와 **Time-range Reads(시간 범위 읽기)** (지난 1시간 동안의 모든 CPU 판독값 조회)에 최적화되어 있습니다. 이는 다음을 통해 달성됩니다:

- **Time-partitioned Storage(시간 분할 저장)**: 데이터가 시간 단위로 버킷화된 청크(예: 1시간 또는 1일 단위 청크)에 저장됩니다. 시간 범위 쿼리는 관련 청크만 건드리면 됩니다.
- **Columnar Compression(컬럼형 압축)**: 연속된 타임스탬프는 작은 차이(delta)만 가집니다 — 이는 델타 인코딩으로 고도로 압축 가능합니다. 값(부동 소수점)은 XOR 인코딩(Prometheus에서 사용되는 Gorilla 알고리즘)으로 잘 압축됩니다.
- **Automatic Chunk Expiration(자동 청크 만료)**: 오래된 청크는 행 단위 삭제가 아닌 파일 단위 삭제를 통해 원자적으로 제거될 수 있으며, 이는 인덱스 파편화를 방지합니다.

## Retention Policies and Downsampling (보존 정책 및 다운샘플링)

원시(raw) 시계열 데이터는 무한히 늘어납니다. 초당 100만 개의 메트릭을 수집하는 모니터링 시스템은 하루에 860억 개의 데이터 포인트를 생성합니다. **Retention Policies(보존 정책)**는 N일보다 오래된 데이터를 자동으로 만료시킵니다. **Downsampling(다운샘플링)** 또는 롤업(rollup)은 원시 데이터를 만료시키기 전에 세밀한 데이터를 더 거친 단위로 집계합니다 — 예를 들어, 원시 10초 데이터를 30일간 보관하고, 1분 평균은 1년, 1시간 평균은 5년간 보관하는 식입니다.

```sql
-- TimescaleDB: 시간별로 파티셔닝된 hypertable 생성
SELECT create_hypertable('cpu_metrics', 'time');

-- Continuous aggregate: 1분 평균을 사전 계산
CREATE MATERIALIZED VIEW cpu_1min
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 minute', time) AS bucket,
  host,
  AVG(cpu_percent) AS avg_cpu,
  MAX(cpu_percent) AS max_cpu
FROM cpu_metrics
GROUP BY bucket, host;

-- Retention policy: 30일보다 오래된 원시 데이터 삭제
SELECT add_retention_policy('cpu_metrics', INTERVAL '30 days');
```

> 💡
> SQL 팀을 위한 TimescaleDB
> 팀이 이미 PostgreSQL을 사용하고 있다면, **TimescaleDB**가 가장 마찰이 적은 시계열 솔루션입니다. PostgreSQL 확장 기능이므로 동일한 연결 문자열, 동일한 SQL, 동일한 도구를 사용합니다. Hypertable은 시간별로 자동 파티셔닝되며, Continuous Aggregate는 롤업을 사전에 계산합니다. 새로운 인프라 도입 없이 시계열 성능을 10~100배 향상시킬 수 있습니다.

> 💡
> Interview Tip (인터뷰 팁)
> 시계열 데이터베이스는 시스템 디자인 인터뷰에서 중심 주제로 등장하는 경우는 드물지만, 구성 요소로는 자주 등장합니다. 메트릭 플랫폼, 모니터링 시스템 또는 IoT 데이터 파이프라인 설계를 요청받으면 다음과 같이 언급하세요: "시계열 데이터의 경우, 30일 원시 데이터 보존 정책과 과거 트렌드 분석을 위한 사전 집계(Continuous Aggregate) 기능을 갖춘 InfluxDB나 TimescaleDB를 사용하겠습니다." 이는 가중치가 낮은 주제에 매몰되지 않으면서도 아키텍처적 인식을 보여줍니다.
