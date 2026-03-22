# Cache Eviction Policies (캐시 축출 정책)

> Source: https://www.sysdesai.com/learn/caching/eviction-policies

---

## Why Eviction Policy Matters (축출 정책이 중요한 이유)

캐시는 유한한 메모리를 가집니다. 메모리가 가득 차면 새로운 항목을 위한 공간을 만들기 위해 **어떤 항목을 제거할지** 결정해야 합니다. 이 결정을 내리는 것이 바로 Eviction Policy(축출 정책)입니다. 잘못된 선택은 히트율(Hit Rate)을 망칠 수 있습니다: 잘못된 항목을 축출하면 다음 요청에서 데이터베이스를 거쳐야 하지만, 올바른 항목을 축출하면 훨씬 더 인기 있는 키를 위한 공간을 확보할 수 있습니다. 캐시 축출은 겉보기엔 단순해 보이지만 실제 성능에 엄청난 영향을 미치는 개념 중 하나입니다.

## LRU — Least Recently Used (최근 최소 사용)

**LRU**는 가장 오랫동안 **액세스되지 않은** 항목을 축출합니다. 직관적인 근거: 한동안 사용하지 않은 데이터는 앞으로도 곧 사용할 가능성이 낮다는 것입니다. LRU는 이중 연결 리스트(Doubly-linked list)와 해시 맵(Hash map)으로 구현됩니다. 모든 `get` 요청은 해당 항목을 리스트의 헤드(head)로 옮기며, 축출이 필요할 때는 테일(tail)에 있는 항목을 제거합니다.

```python
# LRU Cache — OrderedDict를 사용한 O(1) get 및 put 구현
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()   # 삽입/액세스 순서를 보존함

    def get(self, key: str):
        if key not in self.cache:
            return None
        self.cache.move_to_end(key)  # 최근 사용으로 표시
        return self.cache[key]

    def put(self, key: str, value) -> None:
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)  # LRU 항목(맨 앞) 축출
```

LRU는 대부분의 범용 캐시에서 잘 작동합니다. **Temporal Locality(시간적 지역성)** — 최근에 액세스된 데이터가 곧 다시 액세스되는 경향 — 이 흔하기 때문입니다. Redis는 샘플링을 통해 근사화된 LRU를 구현하며, 키당 `lru_clock`을 사용하여 정확한 LRU 변형도 지원합니다.

> ⚠️
> LRU 스캔 내성 (LRU scan resistance)
> LRU는 캐시 스캔 패턴에 취약합니다. 대규모 데이터셋에 대한 일회성 순차 스캔은 자주 쓰이는(hot) 키들의 작업 세트(working set) 전체를 캐시에서 몰아낼 수 있습니다. 가끔 발생하는 대규모 스캔 워크로드의 경우 LRU-K나 SLRU (Segmented LRU) 변형을 고려하세요.

## LFU — Least Frequently Used (최소 빈도 사용)

**LFU**는 **액세스 횟수가 가장 적은** 항목을 축출합니다. 직관적인 근거: 여러 번 조회된 항목은 아마도 인기가 높을 것이므로 계속 유지해야 한다는 것입니다. LFU는 LRU보다 스캔 내성이 강합니다 — 일회성 스캔으로 진정 인기 있는 키들을 몰아내지 않는데, 그 키들은 더 높은 빈도수를 가지고 있기 때문입니다.

**단점:** LFU는 구현 복잡도가 더 높습니다 (빈도 카운터와 최소 힙(Min-heap) 또는 빈도별 버킷 리스트가 필요). 더 중요한 것은 **Frequency Pollution(빈도 오염)** 현상을 겪는다는 점입니다: 과거에 매우 인기 있었으나 더 이상 액세스되지 않는 항목이 새로운 인기 항목을 밀어내고 계속 자리를 차지할 수 있습니다. 해결책은 **Aging(에이징)**입니다 — 시간이 지남에 따라 카운트 값을 감쇠시키는 것입니다. Redis의 `allkeys-lfu` 정책은 로그 함수적으로 감쇠하는 24비트 카운터를 사용하여 이를 근사화합니다.

## FIFO — First In, First Out (선입선출)

**FIFO**는 액세스 빈도나 시점과 상관없이 삽입된 지 **가장 오래된** 항목을 축출합니다. 큐(Queue)로 구현하기가 매우 간단합니다. 하지만 나이(age) 자체가 미래의 액세스를 예측하는 강력한 지표가 아니기 때문에 대부분의 실무 워크로드에서 히트율이 낮습니다. 주로 특수 하드웨어 캐시나 히트율보다 단순함이 더 중요한 경우에 사용됩니다.

## TTL — Time to Live (수명)

**TTL**은 LRU/LFU의 대체재가 아니라 추가적인 만료 메커니즘입니다. 모든 캐시 항목에 최대 수명이 할당됩니다. TTL이 만료되면 해당 항목은 낡은(stale) 것으로 간주되어 제거됩니다 (다음 액세스 시 게으르게 제거되거나 백그라운드 스윕에 의해 능동적으로 제거됨). TTL은 대부분의 시스템에서 **주요 캐시 무효화 도구(Cache Invalidation Tool)**로 사용됩니다.

적절한 TTL 설정은 일종의 예술입니다. 너무 짧으면: 빈번한 캐시 미스와 데이터베이스 과부하 발생. 너무 길면: 원본 데이터가 변경된 후에도 사용자에게 낡은 데이터를 제공. 일반적인 패턴: 주식 가격처럼 변동성이 큰 데이터는 짧은 TTL(초 단위), 사용자 세션 데이터는 중간 TTL(분 단위), 국가 코드와 같은 정적 데이터는 긴 TTL(시간/일 단위).

## Random Eviction (무작위 축출)

**Random Eviction**은 무작위로 희생자를 선택합니다. 놀랍게도 많은 워크로드에서 LRU와 비슷한 성능을 보이면서도 구현 비용은 더 저렴합니다. Redis에서는 `allkeys-random`으로 사용됩니다. 액세스 패턴을 예측할 수 없고 오버헤드를 최소화하고 싶을 때 합리적인 선택입니다.

## Eviction Policy Comparison (축출 정책 비교)

| Policy(정책) | Evicts(축출 대상) | Memory Overhead(메모리 오버헤드) | Scan Resistant(스캔 내성) | Best For(용도) |
| --- | --- | --- | --- | --- |
| LRU | 최근 사용되지 않은 것 | 낮음 (이중 연결 리스트 + 해시맵) | 아니요 | 시간적 지역성이 있는 일반 워크로드 |
| LFU | 빈도가 가장 낮은 것 | 중간 (빈도 카운터 필요) | 예 | 안정적인 핫스팟, 추천 캐시 |
| FIFO | 가장 먼저 들어온 것 | 매우 낮음 (큐) | 아니요 | 단순하고 예측 가능한 패턴 |
| TTL | 만료된 항목 | 추가 비용 없음 | N/A | 시간 기반의 신선도가 필요한 변동성 데이터 |
| Random | 무작위 항목 | 추가 비용 없음 | 부분적 | 패턴을 모르거나 균등할 때 |

## Redis Eviction Policies in Practice (실무에서의 Redis 축출 정책)

Redis는 `maxmemory-policy` 설정 옵션을 통해 축출 정책을 제공합니다. 사용 가능한 정책들은 범위(모든 키 vs TTL이 설정된 키만)와 알고리즘(LRU, LFU, random, TTL-nearest)을 결합한 형태입니다:

- `noeviction` — 메모리가 가득 차면 에러를 반환 (기본값, 내구성에 가장 안전함)
- `allkeys-lru` — 모든 키에 대해 LRU 적용 (순수 캐시 용도로 가장 일반적)
- `volatile-lru` — 만료 시간이 설정된 키들 사이에서만 LRU 적용
- `allkeys-lfu` — 모든 키에 대해 LFU 적용 (편향된 액세스 패턴에 최적)
- `volatile-ttl` — 만료 시간이 가장 가까운 키부터 축출
- `allkeys-random` — 모든 키에 대해 무작위 축출

> 💡
> Production recommendation (운영 추천 사항)
> Redis를 (주 저장소가 아닌) 캐시로 사용할 때는 `allkeys-lru` 또는 `allkeys-lfu`를 사용하세요. 이를 통해 Redis가 어떤 키든 축출할 수 있게 하여 메모리 부족 에러를 방지할 수 있습니다. `noeviction`은 데이터 유실이 허용되지 않는 주 저장소 용도의 Redis 인스턴스를 위해 남겨두세요.

> 💡
> Interview Tip (인터뷰 팁)
> 축출에 대해 질문을 받으면 단순히 LRU만 나열하지 마세요. 트레이드오프를 이해하고 있음을 보여주세요: "대부분의 액세스 패턴에 시간적 지역성이 적용되므로 LRU로 시작하겠지만, 작업 세트가 캐시보다 커서 LRU가 계속 교체(thrashing)되는 것이 관찰된다면 LFU를 고려하거나 캐시 크기를 늘리겠습니다. 또한 모든 정책에 적절한 TTL을 병행하여 데이터의 신선도를 보장하겠습니다." 이러한 논리는 운영적 성숙도를 잘 보여줍니다.
