# Bloom Filter & HyperLogLog

> Source: https://www.sysdesai.com/learn/distributed-systems/bloom-filters-hyperloglog

---

## 확률적 자료구조가 필요한 이유

대규모 시스템에서 **정확한 답은 비쌉니다**. 웹 크롤러가 중복 확인을 위해 방문한 모든 URL을 저장하면 수 테라바이트의 메모리가 필요할 수 있습니다. 수십억 건의 이벤트에서 고유 사용자를 집계하는 것은 해시 셋으로는 비현실적입니다. **확률적 자료구조(Probabilistic Data Structures)**는 작고 제한된 오류율을 받아들이는 대신 메모리와 CPU 사용량을 극적으로 — 종종 수 자릿수 단위로 — 줄입니다.

## Bloom Filter

**Bloom Filter**는 멤버십 질의(Membership Query)에 답하는 공간 효율적인 확률적 자료구조입니다: 'X가 집합에 있나요?' **거짓 양성(False Positive)**(없는데 있다고 답함)은 발생할 수 있지만, **거짓 음성(False Negative)**(없다고 했는데 실제로 있음)은 절대 발생하지 않습니다. 이 비대칭성은 많은 시스템에서 활용할 수 있습니다.

Bloom Filter는 크기 m의 **비트 배열**(초기값 모두 0)과 **k개의 해시 함수**로 구성됩니다. 원소를 추가할 때는 k개의 함수로 해시하여 해당 비트 위치를 1로 설정합니다. 조회 시에는 k개의 함수로 해시하여 해당 비트 위치가 모두 1인지 확인합니다 — 하나라도 0이라면 해당 원소는 집합에 확실히 없습니다.

Bloom Filter: 삽입 시 비트 설정, 조회 시 k개 비트 모두 확인

### 거짓 양성률(False Positive Rate)

거짓 양성률은 비트 배열 크기 m, 해시 함수 수 k, 삽입된 원소 수 n에 따라 결정됩니다. 최적 거짓 양성률은 대략 `(1 - e^(-kn/m))^k`입니다. 실제로: 원소당 m=10 비트, k=7 해시 함수를 사용하면 거짓 양성률은 약 **1%**입니다. m을 20 비트/원소로 두 배 늘리면 **0.001%**로 떨어집니다.

```typescript
class BloomFilter {
  private bits: Uint8Array;
  private m: number;       // 비트 배열 크기
  private k: number;       // 해시 함수 수

  constructor(expectedItems: number, falsePositiveRate: number) {
    // 최적 m, k 공식
    this.m = Math.ceil(-(expectedItems * Math.log(falsePositiveRate)) / Math.LN2 ** 2);
    this.k = Math.round((this.m / expectedItems) * Math.LN2);
    this.bits = new Uint8Array(Math.ceil(this.m / 8));
  }

  add(item: string): void {
    for (const pos of this.hashPositions(item)) {
      this.bits[Math.floor(pos / 8)] |= 1 << (pos % 8);
    }
  }

  mightContain(item: string): boolean {
    return this.hashPositions(item).every(pos =>
      (this.bits[Math.floor(pos / 8)] >> (pos % 8)) & 1
    );
  }

  private hashPositions(item: string): number[] {
    // 프로덕션에서는 다른 시드로 MurmurHash3 사용
    return Array.from({ length: this.k }, (_, i) =>
      Math.abs(this.simpleHash(item, i)) % this.m
    );
  }

  private simpleHash(s: string, seed: number): number {
    let h = seed;
    for (const c of s) h = Math.imul(h ^ c.charCodeAt(0), 0x9e3779b9);
    return h;
  }
}

// 활용: 확실히 없는 키에 대한 DB 조회 회피
const filter = new BloomFilter(1_000_000, 0.01); // 100만 항목, 1% FPR
// ~1.2 MB vs 100만 문자열 해시 셋의 ~8 MB
```

## 실제 Bloom Filter 활용 사례

| 시스템 | Bloom Filter 활용 |
| --- | --- |
| Apache Cassandra | 각 SSTable에 Bloom Filter 보유; 해당 SSTable에 없는 키의 디스크 읽기 방지 |
| Google Bigtable | 태블릿별 Bloom Filter로 존재하지 않는 행의 디스크 탐색 감소 |
| PostgreSQL | 쿼리 플래너가 다중 컬럼 조회에 Bloom Filter 기반 인덱스 사용 |
| 웹 브라우저 | Safe Browsing API: 악성 URL에 로컬 Bloom Filter 사용; 양성일 때만 서버 조회 |
| Akamai CDN | One-hit-wonder 감지: 최소 두 번 이상 요청된 객체만 캐시(캐시 오염 방지) |

> 💡
> Counting Bloom Filter
> 표준 Bloom Filter는 삭제를 지원하지 않습니다 — 여러 원소가 동일한 비트 위치를 공유할 수 있어 비트를 해제할 수 없습니다. **Counting Bloom Filter**는 각 비트를 작은 카운터(일반적으로 4비트)로 대체합니다. 추가 시 증가, 삭제 시 감소합니다. 4배의 메모리 비용으로 삭제를 지원합니다. 네트워크 라우터의 플로우 카운팅에 사용됩니다.

## HyperLogLog

**HyperLogLog (HLL)**은 고정된 소량의 메모리를 사용하여 멀티셋의 **카디널리티(Cardinality)**(고유 원소 수)를 추정합니다. 핵심 아이디어: 모든 원소를 해시하고 임의의 해시에서 앞에 오는 0의 최대 개수를 관찰하면 카디널리티를 추정할 수 있습니다. 레지스터로 p비트 접두사를 사용하면(2^p 레지스터), HLL은 `1.04 / sqrt(2^p)`의 표준 오차를 달성합니다.

- **12 KB 메모리**로 ~0.81% 표준 오차로 수십억까지 카디널리티 추정 가능 (p=14).
- **HLL++ (Google)**: 소규모 및 대규모 카디널리티에 대한 바이어스 보정이 개선된 HLL. BigQuery에서 사용.
- **Redis HyperLogLog**: `PFADD key element...` / `PFCOUNT key`. 키당 ~12 KB 사용, 0.81% 표준 오차.
- **병합 가능성(Mergeability)**: 두 HLL 스케치를 원소별 최댓값으로 병합 가능 — 통신 오버헤드 없이 샤드 전체에서 분산 카운팅 가능.

| 자료구조 | 사용 사례 | 메모리 (100만 항목) | 오류율 |
| --- | --- | --- | --- |
| 해시 셋(Hash Set) | 정확한 멤버십 | ~32 MB | 0% (정확) |
| Bloom Filter | 멤버십 테스트 | ~1.2 MB (1% FPR) | 1% 거짓 양성, 0% 거짓 음성 |
| Counting Bloom Filter | 멤버십 + 삭제 | ~4.8 MB | ~1% 거짓 양성 |
| HyperLogLog | 카디널리티 추정 | 12 KB (고정!) | ~0.81% 표준 오차 |

> 💡
> 인터뷰 팁
> Bloom Filter는 스토리지 시스템 설계 질문에서 등장합니다('Cassandra가 없는 키의 디스크 읽기를 어떻게 방지하나요?'). HyperLogLog는 분석 질문에서 등장합니다('YouTube가 모든 사용자 ID를 저장하지 않고 고유 비디오 조회수를 어떻게 셀까요?'). 두 자료구조의 핵심 면접 포인트: 이들은 수학적으로 제한된 오류를 가진 공간 최적 근사치입니다. Redis가 HyperLogLog를 네이티브로 지원하고(`PFADD`, `PFCOUNT`), Cassandra가 불필요한 디스크 I/O를 방지하기 위해 SSTable별로 Bloom Filter를 사용한다는 점을 언급하세요.
