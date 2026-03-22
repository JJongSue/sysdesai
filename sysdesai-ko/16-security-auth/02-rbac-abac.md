# RBAC & ABAC Authorization

> 출처: https://www.sysdesai.com/learn/security-auth/rbac-abac

---

## Authentication vs Authorization

**Authentication**(AuthN)은 '당신은 누구인가?'에 답하며 OAuth 2.0 / OIDC에 의해 처리됩니다. **Authorization**(AuthZ)은 '당신은 무엇을 할 수 있는가?'에 답하며 이번 레슨의 주제입니다. 설계 시 이 두 가지 관심사를 분리해야 하며 서로 혼동해서는 안 됩니다. 사용자가 완벽하게 인증(AuthN)되었더라도 특정 리소스에 접근할 권한(AuthZ)은 전혀 없을 수 있습니다.

## Role-Based Access Control (RBAC)

**RBAC**에서는 권한(permissions)이 **Roles**(역할)에 할당되고, Roles가 사용자에게 할당됩니다. 한 사용자는 여러 Roles를 가질 수 있고, 하나의 Role은 여러 권한을 가질 수 있습니다. 주요 장점은 관리 용이성입니다. 새로운 결제 관리자를 고용하면 `billing-admin` Role을 부여하기만 하면 즉시 47개의 결제 관련 권한을 상속받게 됩니다. 규정이 변경되면 Role 하나만 업데이트하면 해당 Role을 가진 모든 사용자의 권한이 업데이트됩니다.

RBAC: 권한은 Roles에서 사용자로 흐릅니다 — Alice와 Bob은 인보이스를 작성할 수 있고, Carol은 읽기만 가능합니다

### Hierarchical RBAC

대부분의 실제 시스템에는 **Role Inheritance**(역할 상속)가 필요합니다. `super-admin`은 `admin`의 모든 권한을 상속받아야 하고, `admin`은 `editor`의 권한을 상속받는 식입니다. 이는 Roles의 Directed Acyclic Graph (DAG)를 형성합니다. 이를 구현하려면 신중한 순환 참조 감지와 상속 그래프를 탐색하는 권한 확인 알고리즘이 필요합니다. **Casbin**(Go, Python, Node) 및 **OPA**(Open Policy Agent)와 같은 라이브러리가 이를 대신 처리해 줍니다.

## Attribute-Based Access Control (ABAC)

**ABAC**는 네 가지 엔티티의 **Attributes**(속성)를 조합한 **Policies**(정책)를 평가하여 접근 결정을 내립니다. 네 가지 엔티티는 **Subject**(사용자 속성: 역할, 부서, 보안 등급), **Resource**(민감도, 소유자, 분류), **Action**(읽기, 쓰기, 삭제), 그리고 **Environment**(시간, IP 주소, 지리적 위치)입니다. 이를 통해 RBAC로는 표현할 수 없는 매우 정교한 정책을 만들 수 있습니다.

rego

```
# OPA (Open Policy Agent) — ABAC 정책 예시
# 사용자의 보안 등급 >= 리소스 민감도 등급이고 업무 시간 내일 때만 읽기 허용

package authz

default allow = false

allow {
    input.action == "read"
    input.user.clearance_level >= input.resource.sensitivity_level
    is_business_hours
    input.user.department == input.resource.owning_department
}

is_business_hours {
    hour := time.clock([time.now_ns(), "America/New_York"])[0]
    hour >= 9
    hour < 17
}
```

## RBAC vs ABAC: 적절한 모델 선택하기

| 차원 | RBAC | ABAC |
| --- | --- | --- |
| 복잡성 | 낮음 — Roles가 직관적임 | 높음 — Policies 설계에 신중한 엔지니어링 필요 |
| 유연성 | 낮음 — Coarse-grained (굵은 입자) | 높음 — Fine-grained (가는 입자), 컨텍스트 인식 |
| 감사 용이성 | 높음 — '누가 X 역할을 가졌나?' 확인이 쉬움 | 중간 — 정책 평가 과정을 추적해야 함 |
| 성능 | 높음 — 단순한 Role 조회 | 낮음 — 정책 평가 오버헤드 |
| Multi-tenancy | Tenant 범위의 Roles가 필요함 | 자연스러움 — Tenant ID를 속성으로 활용 |
| 동적 결정 | 새로운 Role 없이는 불가능 | 기본 제공 — 환경 속성 활용 |
| 적합한 분야 | 기업용 앱, 관리자 패널 | SaaS, 의료, 금융, 정부 기관 |

> ℹ️
> RBAC와 ABAC의 결합
> 대부분의 프로덕션 시스템은 하이브리드 방식을 사용합니다. Coarse-grained 접근 제어에는 RBAC를 사용하고(예: 사용자가 쓰기 작업을 하려면 `editor`여야 함), Fine-grained 결정에는 ABAC를 사용합니다(예: 에디터는 자신이 소유한 리소스만 업무 시간 내에 승인된 IP 범위에서 수정 가능). 이렇게 하면 권한 모델의 감사 가능성을 유지하면서도 동적인 정책 적용이 가능합니다.

## Multi-Tenancy Authorization

**Multi-tenant SaaS** 시스템에서 모든 권한 결정은 Tenant 범위로 한정되어야 합니다. 가장 치명적인 버그 유형은 **Cross-tenant Data Leakage**(교차 테넌트 데이터 유출)로, TenantA의 사용자 Alice가 실수로 TenantB의 데이터를 읽게 되는 경우입니다. 두 가지 일반적인 접근 방식이 있습니다: **Tenant-scoped Roles**(사용자가 Tenant `T1` 내에서 `admin` 역할을 가짐, `T1:admin`으로 표현) 또는 **ABAC 속성으로서의 Tenant**(모든 리소스가 `tenant_id`를 가지고 모든 정책에서 `subject.tenant_id == resource.tenant_id`를 요구함).

## ReBAC: Relationship-Based Access Control

**Google Zanzibar**(Google Drive, Docs, YouTube의 권한 시스템)는 **Relationship-Based Access Control**을 도입했습니다. 접근 권한은 사용자와 객체 간의 관계 그래프에 의해 결정됩니다: '사용자 A는 폴더 B의 에디터이고, 폴더 B는 문서 C를 포함하므로, 사용자 A는 문서 C를 수정할 수 있다.' 이는 RBAC나 ABAC가 표현하기 어려운 복잡한 실제 소유권 및 위임 계층 구조를 모델링할 수 있을 만큼 표현력이 풍부합니다. 오픈 소스 구현체로는 **Authzed/SpiceDB**, **OpenFGA**, **Ory Keto** 등이 있습니다.

> 💡
> 면접 팁
> 면접관이 Google Drive 같은 시스템의 권한 설계를 요구하면 Google Zanzibar와 ReBAC를 언급하세요. Relationship tuples(user: alice, relation: editor, object: folder:marketing)가 필요하며 권한 쿼리가 이 그래프를 탐색한다고 설명하세요. 이는 기본적인 RBAC를 넘어 최신 권한 시스템에 대한 이해도를 보여줍니다.

## 권한 저장 및 적용

- **Centralized Policy Engine**: OPA, Casbin 또는 SpiceDB를 Sidecar 또는 마이크로서비스로 실행합니다. 모든 서비스는 요청을 처리하기 전에 권한 확인 호출을 수행합니다. 약간의 지연 시간(~1–5ms)이 발생하지만 일관된 정책 적용이 가능합니다.
- **JWT Claims**: Roles/Permissions를 JWT에 포함합니다. 빠르지만(추가 호출 없음), 토큰이 갱신될 때까지 권한 정보가 최신이 아닐 수 있습니다. 입자가 굵고 자주 변하지 않는 권한에만 적합합니다.
- **Middleware/Decorator**: API Gateway나 프레임워크 미들웨어에서 권한 로직을 처리합니다. 요청이 비즈니스 로직에 도달하기 전에 거부됩니다. 관심사의 깔끔한 분리가 가능합니다.
- **Database Row-Level Security**: 데이터 접근 시, PostgreSQL 등에서 지원하는 Row-level security 정책을 데이터베이스 계층에서 적용합니다. 이는 애플리케이션 버그에 대한 강력한 방어선이 됩니다.
