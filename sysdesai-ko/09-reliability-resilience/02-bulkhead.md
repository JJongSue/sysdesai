# Bulkhead (벌크헤드) 패턴

> 출처: https://www.sysdesai.com/learn/reliability-resilience/bulkhead

---

## Bulkhead (벌크헤드)의 개념

선박의 선체는 벌크헤드(Bulkhead)라고 불리는 수밀 격실로 나뉩니다. 하나의 격실이 침수되어도 나머지는 마른 상태를 유지하여 배가 계속 떠 있을 수 있습니다. 소프트웨어의 **Bulkhead (벌크헤드) 패턴**도 동일한 원리를 적용합니다. 리소스(스레드 풀, 커넥션 풀, 세마포어)를 분할하여 한 영역의 실패나 지연이 다른 영역에 필요한 리소스를 고갈시키지 않도록 격리하는 것입니다.

Bulkhead (벌크헤드)가 없다면, 단 하나의 느린 하위 서비스가 공유 스레드 풀을 모두 소진시켜 해당 서비스와 관련 없는 기능을 포함한 애플리케이션 전체가 응답 불능 상태에 빠질 수 있습니다. Bulkhead (벌크헤드)는 전체 장애를 국소적이고 부분적인 기능 저하로 바꾸어 줍니다.

## 세 가지 격리 수준 (Three Levels of Bulkheading)

| 수준 | 메커니즘 | 격리 범위 |
| --- | --- | --- |
| Thread pool (스레드 풀) 격리 | 각 통합 지점마다 전용 고정 크기 스레드 풀 할당 | 의존성별 CPU 및 블로킹 I/O 리소스 |
| Semaphore (세마포어) 격리 | 세마포어로 동시 실행 호출 수를 제한 (별도 스레드 없음) | 낮은 오버헤드로 동시성 제한; 타임아웃 미지원 |
| Connection pool (커넥션 풀) 분할 | 소비자 또는 기능별로 별도의 DB/HTTP 커넥션 풀 운용 | 네트워크 연결 및 소켓 |
| 서비스 수준 Bulkhead (벌크헤드) | 테넌트나 기능 등급별로 별도의 마이크로서비스 인스턴스 배포 | 테넌트 간 완전한 컴퓨팅 격리 |

## 실무에서의 Thread Pool (스레드 풀) 격리
Thread pool (스레드 풀) 벌크헤드: 느린 InventoryService는 자신의 전용 풀(5개 스레드)만 소진시키며, Payment 및 Email 풀에는 영향을 주지 않습니다.
`InventoryService`가 느려지면 해당 풀이 10개의 스레드(또는 설정된 한도)로 가득 차고, 새로운 호출은 즉시 `BulkheadFullException`과 함께 거부됩니다. 이때 Payment와 Email 풀은 전혀 영향을 받지 않습니다. 애플리케이션은 전체적으로 무너지지 않고 우아하게 기능이 저하됩니다.

## Semaphore (세마포어) vs Thread Pool (스레드 풀) 격리

**Thread Pool (스레드 풀) 격리**는 각 호출을 별도의 스레드에서 실행하므로, 호출별 타임아웃 설정과 깔끔한 비동기 취소가 가능합니다. 오버헤드로는 컨텍스트 스위칭 비용과 스레드당 메모리 사용이 발생합니다. **Semaphore (세마포어) 격리**는 호출 스레드 자체에서 동시 호출 수를 카운트하므로 스레드 오버헤드가 거의 없지만, 실행 중인 차단된 호출을 도중에 타임아웃 시킬 수 없습니다. 엄격한 타임아웃이 필요한 경우 Thread Pool (스레드 풀) 격리를 사용하고, 처리량이 매우 높고 비차단(Non-blocking) 경로인 경우 Semaphore (세마포어) 격리를 사용하세요.

> 💡
> 적절한 풀 크기 설정
> 풀이 너무 크면 격리 효과가 떨어집니다. 반대로 너무 작으면 불필요한 요청 거부가 발생합니다. 리틀의 법칙(Little's Law)을 사용하여 추정하세요: 풀 크기 ≈ (평균 동시성) = (처리량) × (평균 지연 시간). 큐 깊이와 거부율을 모니터링하여 지속적으로 튜닝해야 합니다.

## 서비스 수준 Bulkhead (벌크헤드) (멀티 테넌시)

SaaS 플랫폼에서는 특정 테넌트의 과도한 사용이 다른 모든 테넌트의 서비스 품질을 저하시킬 수 있습니다. 서비스 수준 Bulkhead (벌크헤드)는 테넌트 등급별로 전용 인스턴스(또는 쿠버네티스 포드)를 배포합니다. 엔터프라이즈 고객은 전용 격리 풀을 할당받고, 무료 등급 사용자는 다른 풀을 공유합니다. 무료 등급 풀이 과부화되더라도 엔터프라이즈 사용자는 전혀 영향을 받지 않습니다. AWS는 이를 **Shuffle Sharding (셔플 샤딩)** 변형이라고 부릅니다. 각 고객에게 무작위로 선택된 샤드 하위 집합을 할당하여, 전체 샤드 장애가 발생하더라도 극히 일부의 고객만 영향을 받게 합니다.

## Resilience4j를 이용한 구현

java
```
// Resilience4j를 통한 Thread pool (스레드 풀) 벌크헤드
BulkheadConfig config = BulkheadConfig.custom()
    .maxConcurrentCalls(10)          // 최대 동시 호출 수
    .maxWaitDuration(Duration.ZERO)  // 가득 찼을 경우 즉시 거부
    .build();

Bulkhead bulkhead = Bulkhead.of("inventoryService", config);

// 호출 데코레이션
Supplier<Inventory> decoratedCall = Bulkhead.decorateSupplier(
    bulkhead, () -> inventoryClient.getStock(productId)
);

Try<Inventory> result = Try.ofSupplier(decoratedCall)
    .recover(BulkheadFullException.class, ex -> Inventory.unavailable());
```

> 💡
> 면접 팁
> 면접에서 Bulkhead (벌크헤드)를 논할 때는 항상 Circuit Breaker (서킷 브레이커)와 짝을 지어 설명하세요. Bulkhead (벌크헤드)는 느린 의존성의 폭발 반경(Blast radius)을 제한하여 리소스 고갈을 방지합니다. Circuit Breaker (서킷 브레이커)는 이미 실패한 것으로 알려진 의존성에 대한 호출을 중단하여 헛된 작업을 방지합니다. 두 패턴은 상호 보완적입니다. 전형적인 회복력(Resilience) 스택은 다음과 같습니다: Timeout (타임아웃) → Retry (재시도) → Circuit Breaker (서킷 브레이커) → Bulkhead (벌크헤드).

## Bulkhead (벌크헤드) vs Circuit Breaker (서킷 브레이커)

| 구분 | Bulkhead (벌크헤드) | Circuit Breaker (서킷 브레이커) |
| --- | --- | --- |
| 제한 대상 | 리소스 소비 (스레드, 커넥션) | 실패 중인 의존성에 대한 호출량 |
| 트리거 | 풀/세마포어가 가득 참 | 에러율 또는 느린 호출 비율이 임계값 초과 |
| 효과 | 풀이 가득 찼을 때 새 호출 즉시 거부 | Open 상태일 때 모든 호출 즉시 실패 |
| 회복 | 호출이 완료되어 풀에 여유가 생기면 자동 회복 | 리셋 타임아웃이 있는 상태 머신 기반 회복 |
| 주요 목표 | 폭발 반경(Blast radius) 억제 | 연쇄 장애 방지 |
