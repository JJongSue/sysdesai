# Clean Architecture(클린 아키텍처)

> 출처: https://www.sysdesai.com/learn/architectural-styles/clean-architecture

---

## 클린 아키텍처 개요(Clean Architecture Overview)

2012년 로버트 C. 마틴(엉클 밥)에 의해 소개된 **Clean Architecture(클린 아키텍처)**는 소프트웨어를 동심원 형태의 계층으로 분리하는 원칙을 정형화했습니다. 가장 안쪽 원에는 가장 안정적인 비즈니스 규칙이 위치하고, 가장 바깥쪽 원에는 가장 자주 변하는 인프라 세부 사항이 위치합니다. 이는 헥사고날 아키텍처, 어니언 아키텍처, DCI(Data, Context, Interaction) 아키텍처를 하나로 결합한 형태입니다.

가장 핵심적인 원칙은 **Dependency Rule(의존성 규칙)**입니다: 소스 코드의 의존성은 반드시 안쪽 방향으로만 향해야 합니다. 안쪽 원은 인터페이스를 정의하고, 바깥쪽 원은 이를 구현합니다. 안쪽 원의 그 어떤 것도 바깥쪽 원에 대해 알 수 없습니다.

클린 아키텍처의 4개 원 — 모든 의존성은 엔티티를 향해 안쪽으로 향함.

## 4개 계층(The Four Rings)

| 계층 | 명칭 | 내용 | 변경 빈도 |
| --- | --- | --- | --- |
| 1 (가장 안쪽) | Entities(엔티티) | 애플리케이션에 무관하게 전사적으로 적용되는 핵심 비즈니스 객체와 규칙. 예: 검증 로직이 포함된 송장(Invoice) 엔티티. | 거의 없음 — 비즈니스의 근본적인 규칙임 |
| 2 | Use Cases(유스케이스) | 애플리케이션 전용 비즈니스 규칙. 엔티티를 조율하여 유스케이스를 수행함. 예: PlaceOrderUseCase, RefundPaymentUseCase. | 가끔 — 비즈니스 요구사항이 바뀔 때 |
| 3 | Interface Adapters(인터페이스 어댑터) | 유스케이스 포맷과 외부 포맷 간의 데이터 변환을 담당. 컨트롤러, 프리젠터(Presenters), 리포지토리 구현체 등. | 빈번함 — UI나 외부 API가 바뀔 때 |
| 4 (가장 바깥쪽) | Frameworks & Drivers | 웹 프레임워크(Express, Rails), DB 드라이버, UI 툴킷 등. 단순히 연결 고리 역할을 하는 코드. | 매우 빈번함 — 기술 선택이 진화함에 따라 |

## 실제 적용 사례: 의존성 규칙(The Dependency Rule in Practice)

```typescript
// 계층 1: Entity — 순수한 비즈니스 객체, 바깥 계층을 전혀 임포트하지 않음
export class Invoice {
  constructor(
    public readonly id: string,
    public readonly lineItems: LineItem[],
  ) {}

  get total(): number {
    return this.lineItems.reduce((sum, item) => sum + item.subtotal, 0);
  }

  validate(): void {
    if (this.lineItems.length === 0) throw new Error("송장에는 항목이 있어야 합니다");
    if (this.total <= 0) throw new Error("송장 총액은 0보다 커야 합니다");
  }
}

// 계층 2: Use Case — 인터페이스를 통해 필요한 것을 정의 (안쪽으로의 의존성만 존재)
interface InvoiceRepository {
  save(invoice: Invoice): Promise<void>;
  findById(id: string): Promise<Invoice | null>;
}

interface EmailNotifier {
  sendInvoice(email: string, invoice: Invoice): Promise<void>;
}

export class CreateInvoiceUseCase {
  constructor(
    private readonly repo: InvoiceRepository,    // 구현체가 아닌 인터페이스에 의존
    private readonly notifier: EmailNotifier,     // 구현체가 아닌 인터페이스에 의존
  ) {}

  async execute(request: CreateInvoiceRequest): Promise<string> {
    const invoice = new Invoice(generateId(), request.lineItems);
    invoice.validate();
    await this.repo.save(invoice);
    await this.notifier.sendInvoice(request.customerEmail, invoice);
    return invoice.id;
  }
}

// 계층 3: Interface Adapter — 계층 2의 리포지토리 인터페이스를 구현
export class PostgresInvoiceRepository implements InvoiceRepository {
  async save(invoice: Invoice): Promise<void> {
    await db.query("INSERT INTO invoices ...", [invoice.id, invoice.total]);
  }
  // ...
}

// 계층 4: Framework — 이들을 하나로 묶음 (Express 라우트)
app.post("/invoices", async (req, res) => {
  const useCase = new CreateInvoiceUseCase(
    new PostgresInvoiceRepository(db),
    new SendGridEmailNotifier(sgClient),
  );
  const id = await useCase.execute(req.body);
  res.status(201).json({ id });
});
```

## 경계 넘어가기: Humble Object Pattern(험블 객체 패턴)

데이터가 바깥쪽으로 흘러야 할 때(예: 유스케이스가 데이터를 컨트롤러로 반환할 때), 클린 아키텍처는 경계에서 **Data Transfer Objects(DTO, 데이터 전송 객체)**를 사용합니다. 유스케이스는 도메인 엔티티를 컨트롤러에 직접 반환하지 않고, 단순한 데이터 구조(일반 객체, 값 객체)를 반환합니다. 이를 통해 바깥쪽 원이 안쪽 원의 엔티티 구현 세부 사항에 의존하는 것을 방지합니다.

> 💡
> 작은 프로젝트를 과하게 설계하지 마세요.
> 클린 아키텍처는 간접성(Indirection)과 보일러플레이트(Boilerplate)를 추가합니다. 복잡한 비즈니스 규칙이 없는 작은 CRUD 서비스의 경우, 단순한 3계층 구조(컨트롤러 → 서비스 → 리포지토리)가 더 실용적입니다. 클린 아키텍처는 다음과 같은 경우에 가치를 발휘합니다: 도메인 로직이 복잡하고 자주 진화할 때, 기술적 결정(어떤 DB를 쓸지, 어떤 프레임워크를 쓸지)을 미루고 싶을 때, 도메인 로직을 독립적으로 테스트하는 것이 매우 중요할 때.

## Clean Architecture vs Hexagonal Architecture

두 아키텍처 모두 **Dependency Inversion Principle(의존성 역전 원칙)**을 핵심 메커니즘으로 강제합니다. 주요 차이점은 다음과 같습니다: 클린 아키텍처는 안쪽 계층에 구체적인 이름(Entities, Use Cases)을 부여하고 코어 내부의 계층화에 대해 더 규범적입니다. 반면 헥사고날 아키텍처는 코어 내부 구조를 규정하지 않고 포트와 어댑터 메커니즘에 집중합니다. 실무에서는 종종 이 둘을 결합하여 사용합니다. 외부 경계에는 헥사고날 방식을, 내부 구조에는 클린 아키텍처 계층을 적용하는 식입니다.

> 💡
> 인터뷰 팁
> 클린 아키텍처에 대해 질문을 받으면 의존성 규칙(Dependency Rule)부터 설명하세요: "모든 소스 코드 의존성은 안쪽을 향하며, 도메인의 그 무엇도 프레임워크나 데이터베이스에 대해 알지 못합니다." 그런 다음 4개 계층을 설명하세요. 실제 프로젝트에서 사용해 본 경험이 있다면 구체적인 이점을 언급하십시오: "유스케이스 계층의 수정 없이 특정 서비스의 DB를 PostgreSQL에서 MongoDB로 교체할 수 있었습니다." 이러한 구체적인 사례가 교과서적인 정의보다 면접관들에게 더 큰 인상을 남깁니다.
