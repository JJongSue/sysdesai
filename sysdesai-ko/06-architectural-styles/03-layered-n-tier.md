# Layered / N-Tier(계층화 / N-계층) 아키텍처

> 출처: https://www.sysdesai.com/learn/architectural-styles/layered-n-tier

---

## 계층 패턴(The Layered Pattern)

**Layered (N-Tier) architecture(계층화 / N-계층 아키텍처)**는 코드를 수평적인 계층으로 구성하며, 각 계층은 뚜렷한 책임을 가집니다. 계층은 아래 방향으로만 통신합니다. 즉, 위의 계층은 아래의 계층에 의존합니다. 이러한 분리는 코드베이스를 예측 가능하게 만듭니다. 표현 로직이 어디에 있는지, 비즈니스 규칙이 어디에 사는지, 데이터베이스 호출이 어디서 이루어지는지 항상 알 수 있습니다.

전형적인 3계층 분해는 **Presentation → Business Logic → Data Access → Database**입니다. N-Tier에서 'N'은 필요에 따라 계층을 더 추가할 수 있음을 의미합니다. 비즈니스 로직과 데이터 접근 계층 사이에 유스케이스를 캡슐화하기 위한 **Service(서비스) 계층**을 추가하는 것이 흔한 확장 사례입니다.

전형적인 3계층 아키텍처 — 의존성은 아래 방향으로만 흐름.

## 계층별 책임(Layer Responsibilities)

| 계층 | 책임 | 예시 |
| --- | --- | --- |
| Presentation(표현) | 사용자 입력 처리, 출력 렌더링, HTTP 라우팅 | Express.js 라우트, React 컴포넌트, Django 뷰 |
| Business Logic(비즈니스 로직) | 도메인 규칙, 검증, 유스케이스 오케스트레이션 | OrderService.placeOrder(), PricingEngine.calculate() |
| Data Access(데이터 접근) | DB 호출 추상화, 쿼리 빌딩, 매핑 | UserRepository.findById(), ActiveRecord 모델 |
| Database(데이터베이스) | 데이터 저장 및 조회 | PostgreSQL, MongoDB, Redis |

## Strict vs Relaxed 계층화 규칙(The Strict vs Relaxed Layering Rule)

**Strict layering(엄격한 계층화)**에서는 각 계층이 바로 아래 계층하고만 통신할 수 있습니다. 이는 경계를 명확히 강제하지만, 표현 계층이 특별한 비즈니스 로직 변환 없이 데이터베이스의 데이터만 필요로 할 때도 비즈니스 계층을 반드시 거쳐야 하므로 번거로울 수 있습니다. **Relaxed layering(완화된 계층화)**은 단순 읽기 작업의 경우 계층을 건너뛰는 것을 허용하여, 순수성 대신 실용성을 선택합니다.

## 장점

- **관심사 분리(Separation of concerns)**: 각 계층은 하나의 일만 수행합니다. UI 개발자는 데이터베이스 코드를 건드릴 필요가 없습니다.
- **교체 가능성(Replaceability)**: 비즈니스 로직을 건드리지 않고도 표현 계층을 교체할 수 있습니다(예: 웹 UI를 모바일 API로 교체).
- **테스트 용이성**: 데이터 접근 계층을 모킹(mocking)함으로써 데이터베이스 없이 비즈니스 로직만 유닛 테스트할 수 있습니다.
- **친숙함**: 가장 널리 알려진 패턴입니다. Rails, Django, Spring MVC, ASP.NET 등이 기본적으로 이 패턴을 따릅니다.

## 단점과 'Lasagna Code(라자냐 코드)' 문제

계층화 아키텍처의 주요 실패 모드는 **'Lasagna code(라자냐 코드)'**입니다. 모든 변경 사항이 모든 계층을 건드려야 하는 상황을 말합니다. 사용자 프로필에 새로운 필드 하나를 추가하려면 데이터베이스 스키마, ORM 모델, 리포지토리, 서비스, DTO, 컨트롤러, 뷰를 모두 수정해야 합니다. 아키텍처는 수평적이어서 이해하기 쉽지만, 변경 사항은 모든 계층을 수직적으로 관통하며 퍼져나갑니다.

> ⚠️
> 빈약한 도메인 모델(Anemic Domain Model)의 함정
> 계층화 아키텍처는 종종 빈약한 도메인 모델을 만들어냅니다. 도메인 객체(User, Order 등)가 단순히 Getter/Setter만 있는 데이터 컨테이너로 전락하면서, 비즈니스 로직이 서비스 계층이나 컨트롤러로 새어 나갑니다. '비즈니스 로직 계층'은 결국 절차지향적인 스크립트가 되어버립니다. 이는 도메인이 제대로 모델링되지 않았다는 신호입니다.

```typescript
// 안티 패턴: Anemic domain model(빈약한 도메인 모델)
// Order 클래스에 동작은 없고 데이터만 있음
class Order {
  id: string;
  status: string;   // "pending" | "shipped" | "cancelled"
  items: OrderItem[];
  totalPrice: number;
}

// 모든 로직이 서비스 계층에 있음 (절차지향적)
class OrderService {
  async cancelOrder(orderId: string): Promise<void> {
    const order = await this.repo.findById(orderId);
    if (order.status !== "pending") throw new Error("취소할 수 없습니다");
    order.status = "cancelled";
    await this.repo.save(order);
    // ...계속...
  }
}

// 개선안: Rich domain model(풍부한 도메인 모델)
class Order {
  cancel(): void {
    if (this.status !== "pending") throw new DomainError("취소할 수 없습니다");
    this.status = "cancelled";
    this.addEvent(new OrderCancelledEvent(this.id));
  }
}
```

## 계층화 아키텍처를 사용해야 할 때

계층화 아키텍처는 다음과 같은 경우에 훌륭한 선택입니다: 도메인 로직이 얇은 CRUD 중심 애플리케이션, 프런트엔드/백엔드/데이터베이스 전문가가 명확히 나뉜 팀, 도메인이 잘 이해된 엔터프라이즈 애플리케이션, 그리고 모놀리스나 개별 마이크로서비스의 내부 구조를 설계할 때.

> 💡
> 인터뷰 팁
> 면접관들은 백엔드 서비스의 내부 구조를 설명할 때 계층화 아키텍처를 기본값으로 기대하는 경우가 많습니다. 계층의 이름(컨트롤러 → 서비스 → 리포지토리 → 데이터베이스)을 대고, 의존성 방향을 설명하며, 각 계층을 독립적으로 테스트하는 방법을 설명할 준비를 하세요. 보너스: Hexagonal(헥사고날)이나 Clean(클린) 아키텍처는 '데이터베이스 중심'의 함정을 피하기 위해 계층화 사고를 발전시킨 형태라고 언급하십시오.
