# Redis & Memcached

> Source: https://www.sysdesai.com/learn/caching/redis-memcached

---

## The Two Dominant In-Memory Caches (두 개의 지배적인 인메모리 캐시)

엔지니어들이 "캐시 계층을 추가하자"고 말할 때, 대부분은 **Redis** 또는 **Memcached**를 의미합니다. 둘 다 1밀리초 미만의 액세스를 위해 데이터를 RAM에 저장합니다. 또한 둘 다 대규모 환경에서 검증되었습니다 — Redis는 GitHub, Twitter, Stack Overflow, Airbnb에서 사용되며, Memcached는 Facebook의 캐시 인프라를 뒷받침합니다. 하지만 이들은 서로 다른 설계 철학을 가지고 있으며, 잘못된 선택은 불필요한 운영 복잡성을 초래할 수 있습니다.

## Memcached: Simple, Fast, Multi-Threaded (단순함, 빠름, 멀티스레드)

Memcached는 의도적으로 최소한의 기능만 제공합니다. 이는 **RAM에 있는 분산 해시 테이블** 그 이상도 이하도 아닙니다. 데이터 모델은 키가 문자열(최대 250바이트)이고 값이 불투명한 블록(기본 최대 1MB)인 평면적인 키-값 저장소입니다. `get`과 `set` (그리고 `delete`, `add`, `replace`, 원자적 `incr`/`decr`)이라는 핵심 작업만 지원합니다.

Memcached는 **Multi-threaded(멀티스레드)** 방식으로 동작하여 단일 노드의 모든 코어를 효율적으로 사용합니다. **Slab Allocator(슬랩 할당자)**를 사용하여 메모리 파편화 없이 관리합니다 — 메모리는 고정된 청크 크기를 가진 슬랩 클래스로 미리 나누어집니다. 지속성(Persistence), 복제(Replication), Pub/Sub, 서버 측 스크립팅 기능은 없습니다. Memcached 노드가 재시작되면 캐시는 비어있는(cold) 상태가 됩니다. 이러한 단순함은 의도된 것입니다: Memcached 팀은 복제와 지속성은 애플리케이션이나 인프라 계층에서 처리해야 한다고 주장합니다.

> ℹ️
> Memcached Clustering (Memcached 클러스터링)
> Memcached 자체에는 클러스터링 기능이 없으며 단일 노드 데몬으로 동작합니다. 여러 Memcached 노드에 걸친 샤딩은 전적으로 **Client-side(클라이언트 측)**에서 이루어집니다: 애플리케이션 라이브러리가 키를 해싱하여 적절한 서버로 라우팅합니다. Facebook의 mcrouter 프록시는 이를 대규모 환경에서 투명하게 처리합니다.

## Redis: Feature-Rich, Persistent, Single-Threaded Core (풍부한 기능, 지속성, 싱글스레드 코어)

Redis (Remote Dictionary Server)는 캐시로 사용하기에 충분히 빠르면서도 다양한 기능을 갖춘 **데이터 구조 서버**입니다. 풍부한 데이터 모델이 가장 큰 장점입니다: 문자열, 리스트, 셋, 정렬된 셋(`ZSET`), 해시, 비트맵, HyperLogLog, 스트림, 지리 공간 인덱스 등을 O(1) 또는 O(log n) 작업으로 지원합니다. 덕분에 Redis는 단순한 캐시가 아니라 분산 시스템의 **맥가이버 칼(Swiss Army knife)** 같은 존재입니다.

- **String** — 단순 키-값, `SET user:42:name 'Alice'`, `INCR page_views`
- **Hash** — 키 내부의 중첩된 키-값, `HSET user:42 name Alice age 30`; 객체 저장에 적합
- **List** — 정렬된 연결 리스트, `LPUSH`/`RPUSH`/`LRANGE`; 큐와 활동 피드에 사용
- **Set** — 순서 없는 고유 멤버 집합, `SADD`/`SMEMBERS`/`SINTER`; 태그, 친구 목록에 사용
- **Sorted Set (ZSET)** — 점수(score)가 있는 셋, `ZADD`/`ZRANGE`; 리더보드, 우선순위 큐에 사용
- **Stream** — 추가 전용 로그, `XADD`/`XREAD`; 이벤트 소싱 및 메시지 전달에 사용

## Redis Persistence (Redis 지속성)

Memcached와 달리 Redis는 재시작 후에도 데이터를 유지할 수 있습니다. 두 가지 지속성 메커니즘이 있습니다:

| Mechanism(메커니즘) | How It Works(작동 방식) | Recovery Time(복구 시간) | Data Loss Risk(데이터 손실 위험) | Use When(사용 시점) |
| --- | --- | --- | --- | --- |
| RDB (Snapshot) | 일정 주기마다(예: 5분) 프로세스를 포크하여 시점 스냅샷을 디스크에 저장 | 빠름 (파일 하나 로드) | 스냅샷 간격만큼 발생 가능 | 빠른 재시작이 중요하고 약간의 데이터 유실이 허용될 때 |
| AOF (Append-Only File) | 모든 쓰기 명령을 로그로 기록하고 재시작 시 재생. fsync 설정 가능: 항상/1초마다/안 함 | 느림 (로그 재생) | 0 (always) ~ 약 1초 (everysec) | 데이터 유실 최소화가 필요할 때 |
| RDB + AOF | 둘 다 활성화; 복구에는 AOF, 빠른 백업에는 RDB 사용 | 중간 | 약 1초 | Write-Back 캐시를 위한 운영 환경 추천 설정 |

## Redis High Availability: Sentinel & Cluster (Redis 고가용성: 센티널 및 클러스터)

**Redis Sentinel**은 단일 프라이머리 + 복제본 구성에서 자동 장애 조치(failover)를 제공합니다. 센티널이 프라이머리를 모니터링하고, 실패 시 새로운 프라이머리를 선출하며, 클라이언트를 재구성합니다. 데이터 샤딩 기능은 없으며 모든 데이터는 하나의 샤드에 존재합니다. 중간 규모의 데이터에 적합합니다.

**Redis Cluster**는 여러 프라이머리 노드에 분산된 16,384개의 해시 슬롯을 통해 자동 샤딩을 제공합니다. 각 노드는 선택적으로 복제본을 가질 수 있습니다. 클라이언트는 CLUSTER SLOTS 맵을 사용하여 명령을 라우팅합니다. 이는 수평 확장을 가능하게 하지만 복잡성을 더합니다: 여러 키를 사용하는 명령은 모든 키가 동일한 슬롯에 있어야 합니다 (해시 태그 `{user:42}:profile`를 사용하여 키를 동일 위치에 배치).

## Redis Pub/Sub and Other Features (Redis Pub/Sub 및 기타 기능)

Redis의 `PUBLISH`/`SUBSCRIBE`는 실시간 메시징을 가능하게 합니다 — 무효화 알림, 라이브 스코어 업데이트, 채팅 등에 유용합니다. 하지만 Redis의 Pub/Sub은 **Fire-and-forget(전송 후 망각)** 방식입니다: 지속성이 없으므로 오프라인 상태인 구독자는 메시지를 놓칩니다. 신뢰할 수 있는 메시징이 필요하다면 Redis Streams를 사용하세요.

`EVAL`을 이용한 **Lua scripting**은 여러 명령을 원자적으로 실행할 수 있게 해줍니다. `MULTI`/`EXEC`을 통한 **Transaction**은 명령들을 배치로 묶어 원자적으로 처리합니다. **Pipelining**은 각 응답을 기다리지 않고 여러 명령을 한꺼번에 보내어, 지연 시간이 긴 연결에서도 처리량을 획기적으로 향상시킵니다.

## Redis vs Memcached: Decision Guide (결정 가이드)

| Dimension(차원) | Redis | Memcached |
| --- | --- | --- |
| 데이터 모델 | 풍부함 (strings, lists, sets, hashes, ZSET, streams) | 평면적인 키-값 블록만 가능 |
| 지속성 | 지원 (RDB, AOF 또는 둘 다) | 미지원 — 메모리 전용 |
| 복제 | 내장 (Sentinel, Cluster) | 클라이언트 측 샤딩만 가능 |
| 스레딩 | 싱글스레드 명령 처리 (v6+부터 I/O 스레드 도입) | 멀티스레드, 모든 코어 활용 |
| Pub/Sub | 지원 (PUBLISH/SUBSCRIBE, Streams) | 미지원 |
| Lua Scripting | 지원 (EVAL) | 미지원 |
| 메모리 효율 | 키당 오버헤드가 더 높음 | 오버헤드가 낮음, 슬랩 할당자 사용 |
| 단순함 | 비교적 복잡함 | 운영상 더 단순함 |
| 용도 | 리더보드, 세션, 큐, 속도 제한, Pub/Sub | 극한의 규모에서 단순 객체 캐싱 |

> 💡
> Default choice (기본 선택)
> 2024년 이후에는 특별한 이유가 없다면 Redis를 선택하세요. Redis의 풍부한 기능이 운영 오버헤드를 크게 늘리지 않으면서도 캐싱, 큐잉, 속도 제한, Pub/Sub 인프라를 하나로 통합할 수 있게 해줍니다. Memcached의 주요 장점인 멀티스레딩은 많은 코어를 가진 장비에서 매우 높은 QPS(초당 100만 회 이상)를 처리해야 할 때 주로 유의미합니다.

> 💡
> Interview Tip (인터뷰 팁)
> 면접관이 "어떤 캐시를 사용하시겠습니까?"라고 물으면 그냥 Redis라고 답하지 마세요. "리더보드를 위한 Sorted Set과 캐시 무효화 알림을 위한 Pub/Sub 기능이 필요하므로 Redis를 사용하겠습니다. 시스템을 하나로 통합하면 운영 오버헤드를 줄일 수 있습니다. 만약 Facebook 수준의 QPS 환경에서 순수 키-값 캐싱만 수행하고 단일 노드의 모든 CPU 코어에서 최대 처리량을 뽑아내야 한다면 Memcached의 멀티스레딩이 매력적일 것입니다."라고 답하세요. 두 기술을 모두 알고 있으며 상황에 맞는 선택을 할 수 있음을 보여주는 것이 목표입니다.
