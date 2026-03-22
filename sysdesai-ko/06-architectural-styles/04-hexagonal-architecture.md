# Hexagonal Architecture(헥사고날 아키텍처, Ports & Adapters)

> 출처: https://www.sysdesai.com/learn/architectural-styles/hexagonal-architecture

---

## 헥사고날 아키텍처가 해결하는 문제

**Hexagonal Architecture(헥사고날 아키텍처)**(또는 **Ports and Adapters(포트와 어댑터)** 패턴)는 2005년 알리스테어 콕번(Alistair Cockburn)에 의해 소개되었습니다. 이 아키텍처는 계층화 아키텍처의 근본적인 문제인 '도메인 로직이 데이터베이스에 의존한다'는 점을 해결합니다. 전형적인 3계층 앱에서 비즈니스 로직은 `UserRepository`를 임포트하고, 이는 다시 `pg`(PostgreSQL 라이브러리)를 임포트합니다. 즉, 도메인 로직이 실제 데이터베이스 없이는 실행될 수 없음을 의미합니다. 이는 테스트를 어렵게 만들고, 데이터베이스 교체를 힘들게 하며, 도메인이 진정으로 격리되지 못하게 합니다.

헥사고날 접근 방식은 이를 뒤집습니다: **Domain/application core(도메인/애플리케이션 코어)**가 중심에 위치하고 **Ports(포트)**(인터페이스)를 정의합니다. 인프라 구성 요소(데이터베이스, 메시지 큐, HTTP 클라이언트, UI 프레임워크 등)는 이러한 포트를 **Adapters(어댑터)**로서 구현합니다. 코어는 절대 인프라를 임포트하지 않으며, 반대로 인프라가 코어를 임포트합니다.

헥사고날 아키텍처: 코어가 포트를 정의하고, 가장자리의 어댑터들이 이를 구현함.

## 포트(Ports) vs 어댑터(Adapters)

| 개념 | 정의 | 예시 |
| --- | --- | --- |
| Inbound Port(인바운드 포트, Driving) | 외부 액터가 애플리케이션을 호출할 수 있도록 코어가 노출하는 인터페이스 | placeOrder(), cancelOrder() 메서드가 있는 OrderService 인터페이스 |
| Outbound Port(아웃바운드 포트, Driven) | 코어가 필요로 하는 인프라 기능을 호출하기 위해 정의하는 인터페이스 | findById(), save() 메서드가 있는 UserRepository 인터페이스 |
| Inbound Adapter(인바운드 어댑터) | 외부 입력을 변환하여 인바운드 포트를 호출하는 구현체 | REST 컨트롤러, CLI 커맨드 핸들러, 메시지 컨슈머 |
| Outbound Adapter(아웃바운드 어댑터) | 실제 인프라를 사용하여 아웃바운드 포트를 구현한 것 | PostgresUserRepository, SendGridEmailSender, StripePaymentGateway |

## 코드 예시: 의존성 역전(Dependency Inversion)

```typescript
// ── Outbound Port (코어에서 정의함) ──
// 코어는 이 인터페이스를 소유합니다. PostgreSQL을 임포트하지 않습니다.
interface UserRepository {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<void>;
}

// ── 도메인 엔티티(Domain Entity) ──
class User {
  constructor(public id: string, public email: string, private active: boolean) {}

  deactivate(): void {
    if (!this.active) throw new DomainError("사용자가 이미 비활성 상태입니다");
    this.active = false;
  }
}

// ── 애플리케이션 서비스 (Use Case) ──
// 어댑터가 아닌 포트 인터페이스에만 의존합니다.
class UserService {
  constructor(private readonly users: UserRepository) {}

  async deactivateUser(userId: string): Promise<void> {
    const user = await this.users.findById(userId);
    if (!user) throw new NotFoundError(userId);
    user.deactivate();
    await this.users.save(user);
  }
}

// ── Outbound Adapter (인프라 계층, 코어에 의존함) ──
class PostgresUserRepository implements UserRepository {
  async findById(id: string): Promise<User | null> {
    const row = await db.query("SELECT * FROM users WHERE id = $1", [id]);
    return row ? new User(row.id, row.email, row.active) : null;
  }
  async save(user: User): Promise<void> {
    await db.query("UPDATE users SET ...", [...]);
  }
}

// ── Inbound Adapter (HTTP 컨트롤러) ──
app.delete("/users/:id", async (req, res) => {
  await userService.deactivateUser(req.params.id);
  res.status(204).send();
});
```

## 테스트의 초능력(The Testing Superpower)

헥사고날 아키텍처의 가장 큰 이점은 **테스트 용이성(Testability)**입니다. 애플리케이션 코어는 오직 포트 인터페이스에만 의존하므로, 테스트를 위해 인메모리(in-memory) 구현체를 주입할 수 있습니다. 데이터베이스, HTTP 서버, 이메일 서비스 없이도 모든 비즈니스 로직에 대해 빠르고 결정론적인 유닛 테스트가 가능합니다.

```typescript
// 빠른 유닛 테스트 — 데이터베이스 불필요
class InMemoryUserRepository implements UserRepository {
  private users: Map<string, User> = new Map();
  async findById(id: string) { return this.users.get(id) ?? null; }
  async save(user: User) { this.users.set(user.id, user); }
}

describe("UserService.deactivateUser", () => {
  it("활성 사용자를 비활성화함", async () => {
    const repo = new InMemoryUserRepository();
    repo.save(new User("u1", "alice@example.com", true));
    const svc = new UserService(repo);

    await svc.deactivateUser("u1");

    const user = await repo.findById("u1");
    expect(user?.isActive).toBe(false);
  });
});
```

## 왜 '헥사고날(육각형)'인가요?

육각형 모양 자체는 임의적인 선택입니다. 콕번은 애플리케이션이 여러 측면(포트)을 가지고 있으며, 어떤 수의 어댑터라도 어느 포트에든 연결될 수 있음을 시각적으로 전달하기 위해 육각형을 선택했습니다. 포트가 정확히 6개여야 한다는 뜻이 아닙니다. 육각형은 계층형 다이어그램이 주는 상/하 편향을 피하기 위한 개념적인 형태입니다.

> ℹ️
> Hexagonal vs Clean vs Onion Architecture
> Hexagonal(헥사고날), Clean Architecture(클린 아키텍처 - 엉클 밥), Onion Architecture(어니언 아키텍처 - 제프리 팔레르모)는 모두 동일한 아이디어의 변형입니다: 의존성 역전(Dependency Inversion)을 사용하여 도메인을 인프라로부터 격리하는 것입니다. 이들은 명칭과 내부 계층을 나누는 방식에서만 차이가 납니다. 면접에서 이 용어들은 종종 '의존성이 역전된 도메인 중심 아키텍처'를 설명할 때 혼용됩니다.

> 💡
> 인터뷰 팁
> 설계에서 테스트 용이성에 대한 질문을 받으면 헥사고날 아키텍처를 언급하세요: "리포지토리와 서비스 인터페이스를 도메인 계층의 포트로 정의하고, 실제 PostgreSQL이나 Kafka 구현체는 인프라 계층의 어댑터로 구현하겠습니다. 이렇게 하면 모든 비즈니스 로직 테스트를 인메모리 어댑터를 사용하여 프로세스 내에서 빠르고 안정적으로 실행할 수 있습니다." 이는 아키텍처적 성숙도를 보여주는 신호가 됩니다.
