# 이커머스 플랫폼 설계

> Source: https://www.sysdesai.com/learn/case-studies/e-commerce

---

## 문제 정의

이커머스 플랫폼은 구매자가 상품을 탐색하고, 장바구니에 담고, 구매를 완료할 수 있게 합니다. 고유한 과제는: (1) 빠른 검색으로 초당 수백만 건의 쿼리를 처리해야 하는 **상품 카탈로그**, (2) 초과 판매를 방지하는 **재고 관리**, (3) 극심한 트래픽 급증을 일으키는 **플래시 세일**입니다. 결제와 재고에 대한 정확성 요구 사항으로 인해 일반적인 읽기 중심 시스템보다 더 까다롭습니다.

## 요구 사항

| 기능 요구 사항 | 비기능 요구 사항 |
| --- | --- |
| 상품 카탈로그 탐색 및 검색 | 페이지 로드 < 100ms (P95) |
| 장바구니에 항목 추가/제거 | 재고는 절대 음수가 되면 안 됨 (초과 판매 금지) |
| 체크아웃: 주소, 결제, 주문 확인 | 체크아웃 흐름 99.99% 가용성 |
| 주문 내역 및 상태 추적 | 플래시 세일 중 100배 트래픽 급증 처리 |
| 판매자 카탈로그 관리 | 상품 1천만 개, 일간 활성 사용자 1억 명 |
| 리뷰 및 평점 | PCI DSS 준수 결제 처리 |

## 고수준 아키텍처

이커머스 플랫폼 고수준 아키텍처

## 상품 카탈로그

상품 데이터는 **MySQL**(구조화: SKU, 가격, 판매자, 카테고리)에 저장되고 패싯 필터링을 포함한 전문 검색을 위해 **Elasticsearch**에 인덱싱됩니다. 상품 페이지는 Redis(TTL: 5분)에 적극적으로 캐시되고, 정적 에셋은 CDN 레이어에서 캐시됩니다. 읽기:쓰기 비율은 일반적으로 100:1 이상이므로 읽기에 집중적으로 최적화하세요.

```sql
CREATE TABLE products (
  product_id  BIGINT PRIMARY KEY,
  seller_id   BIGINT NOT NULL,
  title       VARCHAR(500) NOT NULL,
  description TEXT,
  price_cents INT NOT NULL,       -- 금액은 정수(센트)로 저장
  category_id INT NOT NULL,
  status      ENUM('active','inactive','draft') DEFAULT 'active',
  created_at  TIMESTAMP DEFAULT NOW(),
  updated_at  TIMESTAMP DEFAULT NOW() ON UPDATE NOW()
);

CREATE TABLE inventory (
  product_id    BIGINT PRIMARY KEY,
  warehouse_id  INT NOT NULL,
  quantity      INT NOT NULL DEFAULT 0,
  reserved      INT NOT NULL DEFAULT 0,  -- 활성 장바구니/체크아웃의 항목
  CHECK (quantity >= 0),
  CHECK (reserved >= 0),
  CHECK (quantity >= reserved)
);
```

## 장바구니(Shopping Cart)

장바구니는 임시적이고 사용자별인 데이터 구조입니다. **Redis Hashes**가 이상적입니다: `HSET cart:{userId} productId quantity`. 주요 설계 결정:

- **게스트 장바구니**: 쿠키 기반 `sessionId`를 키로 사용; 로그인 시 사용자 장바구니에 병합.
- **장바구니 TTL**: 게스트 장바구니는 7일 후 만료 (`EXPIRE cart:{sessionId} 604800`).
- **장바구니 시점에 재고 예약 안 함**: 방치된 장바구니를 위한 재고 보관을 피하기 위해 체크아웃 시작 시에만 예약.
- **멱등성 업데이트**: 중복 요청 처리를 위해 수량에 `HINCRBY` (증분) 대신 `HSET` (설정) 사용.

## 재고 및 체크아웃 — 초과 판매 방지

초과 판매 방지는 가장 중요한 정확성 요구 사항입니다. 세 가지 접근법이 있습니다:

| 접근법 | 메커니즘 | 일관성 | 성능 |
| --- | --- | --- | --- |
| 비관적 잠금(Pessimistic Locking) | 재고 행에 SELECT ... FOR UPDATE | 강함 — 경쟁 조건 없음 | 낮음 — 잠금이 체크아웃 직렬화 |
| 낙관적 동시성(Optimistic Concurrency, CAS) | UPDATE inventory SET qty = qty-N WHERE qty >= N AND version = V | 강함 — 충돌 시 재시도 | 높음 — 잠금 없음 |
| Redis 원자적 감소 | Redis에서 DECRBY; DB에 비동기 동기화 | 최종 일관성 | 가장 높음 — 밀리초 미만 |

**권장**: Redis `DECRBY`를 빠른 예약 게이트로 사용합니다. Redis 수량이 요청된 양 이상이면 원자적으로 감소하고 진행합니다. 실패하면 DB를 건드리지 않고 즉시 '품절'을 반환합니다. Redis 재고를 MySQL에 비동기로 동기화합니다. 최종 DB 커밋에는 낙관적 동시성을 사용하여 드문 불일치를 잡아냅니다.

## 체크아웃 흐름

재고 예약 및 결제를 포함한 체크아웃 흐름

## 플래시 세일(Flash Sales)

플래시 세일(예: 블랙 프라이데이, 신제품 출시)은 정상의 100배 트래픽 급증을 일으킵니다. 처리 전략:

- **사전 관심 등록**: 대기실/가상 큐 페이지로 급증을 흡수합니다. 타임스탬프가 있는 토큰을 발급하고 배치로 처리합니다.
- **Redis 재고 게이트**: 판매 재고 수량을 Redis에 사전 로드합니다. 경쟁 조건 방지를 위해 원자적 확인-감소에 Lua 스크립트 사용.
- **속도 제한**: 플래시 세일 중 체크아웃 엔드포인트에 사용자당 적극적인 속도 제한 적용.
- **사전 확장**: 판매 시작 30분 전에 오토스케일링 또는 사전 워밍된 인스턴스로 추가 용량 프로비저닝.
- **정적 상품 페이지**: 피크 시 오리진을 건드리지 않고 CDN에서 상품 페이지 제공.

> 📌
> Lua를 사용한 원자적 재고 감소
> Redis Lua 스크립트는 원자적으로 실행됩니다. 수량이 요청된 양 이상인지 확인하고 감소하는 스크립트는 명시적 잠금 없이도 10만 개의 동시 요청에서 경쟁 조건을 방지합니다.

```lua
-- 원자적 재고 예약을 위한 Redis Lua 스크립트
local key = KEYS[1]           -- 예: "inventory:product:42"
local requested = tonumber(ARGV[1])
local current = tonumber(redis.call("GET", key) or "0")

if current >= requested then
  redis.call("DECRBY", key, requested)
  return 1  -- 성공
else
  return 0  -- 품절
end
```

> 💡
> 인터뷰 팁
> 재고 초과 판매 문제는 이커머스 시스템 설계의 핵심 기술 과제입니다. 세 가지 접근법(비관적 잠금, 낙관적 CAS, Redis 원자적)을 모두 설명하고 플래시 세일에 Redis + 원자적 Lua가 최선인 이유를 정당화하세요. 면접관들은 또한 2단계 커밋 문제에 대한 인식도 봅니다: 결제는 성공했지만 재고 커밋이 실패하면 어떻게 될까요? 답은 멱등성 키와 보상 트랜잭션(환불)입니다.
