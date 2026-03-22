# Write-Through & Write-Behind Caching

> 출처: https://www.sysdesai.com/learn/performance-scalability/write-through-write-behind

---

## Write 전략 스펙트럼

Cache-Aside가 읽기를 lazy하게 처리하는 반면, **write 전략**은 변경 사항이 cache와 primary data store 사이에서 어떻게 흐르는지를 결정합니다. 세 가지 전략이 있습니다: **write-through** (동기식, cache 우선, 강한 consistency), **write-behind** (비동기식, cache 우선, 고성능), **write-around** (쓰기 시 cache를 완전히 건너뜀). 각각 consistency와 성능 사이에서 서로 다른 trade-off를 만듭니다.

## Write-Through Caching

Write-through caching에서는 모든 쓰기가 **같은 작업 내에서 cache 먼저, 그리고 data store에 동기적으로** 이루어집니다. Cache와 database 모두 쓰기를 확인할 때까지 호출자에게 응답하지 않습니다. 이를 통해 cache와 database가 항상 동기화 상태를 유지합니다.
Write-Through: 성공을 반환하기 전에 cache와 DB 모두 확인해야 함
- **Consistency**: Cache와 DB가 항상 동기화됩니다. 쓰기 후 읽기는 항상 최신 데이터를 반환합니다.
- **높은 write latency**: 모든 쓰기가 두 번의 round trip (cache + DB)을 기다립니다.
- **Cache pollution**: 자주 읽히지 않는 데이터가 모든 쓰기 시 cache에 저장되어 메모리를 낭비합니다.
- **이상적인 경우**: 데이터가 쓰여지고 즉시 읽혀야 하는 읽기 많은 workload — 예: session store, leaderboard.

## Write-Behind (Write-Back) Caching

**Write-behind** (또는 **write-back**이라고도 함)는 쓰기를 즉시 cache에 저장하고 호출자에게 성공을 반환한 후, 백그라운드에서 비동기적으로 primary store에 데이터를 flush합니다. persistence 단계가 분리되어 있어 애플리케이션은 매우 낮은 write latency를 경험합니다. 이 기법은 SSD, CPU cache, database buffer pool 내부에서 사용됩니다.
Write-Behind: 쓰기는 즉시 반환; persistence는 비동기적
- **매우 낮은 write latency**: 호출자가 database I/O에 블로킹되지 않습니다.
- **Write coalescing**: 같은 key에 대한 여러 쓰기를 단일 DB 쓰기로 배치 처리할 수 있습니다.
- **데이터 손실 위험**: Cache node가 flush 전에 실패하면 진행 중인 쓰기가 손실됩니다.
- **Consistency gap**: DB에서의 읽기 (또는 다른 cache node)는 stale 데이터를 볼 수 있습니다.
- **이상적인 경우**: 일부 데이터 손실이 허용되는 쓰기 많은 workload — 예: analytics counter, IoT 센서 수집, 게임 leaderboard.

> ⚠️
> Write-Behind 내구성 위험
> Write-behind는 금융 거래, 재고 수량, 또는 손실이 허용되지 않는 데이터에는 안전하지 않습니다. Cache가 쓰기 확인과 database flush 사이에 충돌하면 해당 쓰기는 영구적으로 손실됩니다. 내구성이 필요한 경우 항상 write-behind와 durable queue 또는 WAL을 함께 사용하여 crash recovery를 보장하세요.

## Write-Around Caching

**Write-around**는 쓰기 시 cache를 완전히 건너뜁니다 — 데이터는 database로 직접 전달됩니다. Cache는 읽기 시에만 채워집니다 (Cache-Aside와 같은 cache miss path). 이는 거의 다시 읽히지 않을 쓰기 많은 데이터로 cache가 오염되는 것을 방지하지만, 쓰기 후 첫 번째 읽기에서 항상 cache miss가 발생하는 비용이 있습니다.

## 나란히 비교

| 전략 | Write Latency | Read Consistency | 데이터 손실 위험 | 최적 사용 사례 |
| --- | --- | --- | --- | --- |
| Write-Through | 높음 (DB에 동기) | 강함 | 없음 | 읽기 많고 consistency 중요 |
| Write-Behind | 낮음 (비동기 flush) | Eventual consistency | 있음 (flush 전 충돌) | 쓰기 많고 손실 허용 |
| Write-Around | 낮음 (DB 쓰기만) | 쓰기 후 첫 읽기에서 miss | 없음 | 쓰기 후 거의 다시 읽히지 않는 데이터 |
| Cache-Aside | 낮음 (DB 쓰기만) | Eventual consistency | 없음 | 범용, 읽기 많은 경우 |

## Write-Through와 Cache-Aside 결합

실제로 많은 시스템이 패턴을 결합합니다. 일반적인 hybrid는 **쓰기에는 write-through + 읽기에는 Cache-Aside**입니다. 쓰기는 항상 cache를 동기적으로 채우고, 읽기는 먼저 cache를 확인합니다. 이를 통해 쓰기 후 강한 consistency를 유지하면서도 write-through를 통해 쓰여지지 않은 key에 대해 Cache-Aside의 lazy-loading 의미론의 이점을 누릴 수 있습니다.

python

```
# Write-Through: write DB and cache together
def update_product_price(product_id: str, new_price: float) -> None:
    with db.transaction():
        db.execute(
            "UPDATE products SET price = ? WHERE id = ?",
            new_price, product_id
        )
        # Cache updated synchronously within the same logical operation
        cache_key = f"product:{product_id}"
        redis.setex(cache_key, ttl=3600, value=serialize({
            "id": product_id,
            "price": new_price,
        }))
    # Both succeed or we raise and roll back

# Write-Behind: write cache immediately, flush async
def record_page_view(page_id: str) -> None:
    # Increment an in-memory counter (write-behind)
    redis.incr(f"views:{page_id}")
    # A background job aggregates and flushes every 60 seconds
    # → some view counts may be lost on Redis crash
```

> 💡
> 인터뷰 팁
> 면접관이 'cache를 어떻게 일관성 있게 유지하나요?'라고 물으면, write 전략 선택을 중심으로 답변을 구성하세요. 이렇게 말하세요: '읽기는 많지만 쓰기가 적은 user profile service의 경우, 프로필 업데이트가 cache에 즉시 반영되도록 write-through를 사용하겠습니다. view counter의 경우, 몇 개의 count 손실이 허용되므로 주기적 flush를 사용하는 write-behind가 적합합니다.' 하나의 만능 선택이 아니라 데이터의 consistency 요구사항에 따라 전략을 선택한다는 것을 보여주세요.
