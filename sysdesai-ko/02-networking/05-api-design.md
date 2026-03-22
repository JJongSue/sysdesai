# API 디자인: REST, GraphQL & gRPC

> Source: https://www.sysdesai.com/learn/networking/api-design

---

## 세 가지 주요 API 패러다임

현대적인 분산 시스템은 주로 API를 통해 통신합니다. 다음 세 가지 패러다임이 주류를 이룹니다: **REST**(웹의 기본값), **GraphQL**(유연한 클라이언트 주도 쿼리), **gRPC**(고성능 바이너리 RPC). 각각은 단순성, 유연성, 성능 사이에서 서로 다른 트레이드오프(Trade-off)를 가지고 있습니다.

| 측면 | REST | GraphQL | gRPC |
| --- | --- | --- | --- |
| 프로토콜 | HTTP/1.1 또는 HTTP/2 | HTTP/1.1 또는 HTTP/2 | HTTP/2 |
| 데이터 형식 | JSON (주로), XML | JSON | Protocol Buffers (바이너리) |
| 스키마 | OpenAPI / 비정형 | 강한 타입의 GraphQL 스키마 | 강한 타입의 .proto 파일 |
| 데이터 가져오기 | 고정된 엔드포인트, 오버/언더페칭 발생 가능 | 클라이언트가 필요한 데이터만 정확히 요청 | proto에 정의된 고정 RPC 메서드 |
| 실시간 통신 | 폴링(Polling) 또는 웹훅(Webhooks) | 구독(Subscriptions, WebSocket 활용) | 서버 스트리밍, 양방향 스트리밍 |
| 도구 성숙도 | 매우 높음 — 보편적 지원 | 좋음 — Apollo, Relay | 좋음 — 10개 이상의 언어에서 스텁(Stub) 생성 지원 |
| 브라우저 지원 | 네이티브 지원 | 네이티브 지원 | gRPC-Web 프록시 필요 |
| 적합한 사례 | 공개 API, 단순 CRUD, 외부 클라이언트 | 유연한 클라이언트 요구사항, 다양한 클라이언트 타입(모바일/웹) | 내부 마이크로서비스, 저지연 통신, 스트리밍 |

## REST: Representational State Transfer

REST는 HTTP 관례를 기반으로 한 아키텍처 스타일입니다. 리소스는 URL로 식별되며, HTTP 동사(`GET`, `POST`, `PUT`, `PATCH`, `DELETE`)가 연산을 나타냅니다. 잘 설계된 REST API는 자기 서술적(Self-describing)이고 캐싱이 가능하며 무상태(Stateless)성을 유지합니다.

```http
# 사용자 생성
POST /users
Content-Type: application/json
{"name": "Alice", "email": "alice@example.com"}

# 사용자 조회
GET /users/123

# 사용자 정보 수정 (일부)
PATCH /users/123
{"name": "Alice Smith"}

# 사용자의 주문 목록 조회 (중첩 리소스)
GET /users/123/orders?status=pending&page=2&limit=20

# 사용자 삭제
DELETE /users/123
```

- **무상태(Stateless)**: 각 요청은 요청을 처리하는 데 필요한 모든 정보를 포함해야 합니다. 서버 측에 세션 상태를 저장하지 않습니다.
- **균일한 인터페이스(Uniform interface)**: 모든 리소스에 대해 일관된 URL 구조와 HTTP 동사 의미론을 적용합니다.
- **캐싱 가능(Cacheable)**: GET 응답은 캐싱될 수 있습니다. REST가 HTTP와 잘 맞기 때문에 CDN 통합이 자연스럽습니다.
- **계층화된 시스템(Layered system)**: 클라이언트는 서버와 직접 통신하는지 프록시를 거치는지 알 필요가 없습니다.

## GraphQL

GraphQL은 REST의 오버페칭(Over-fetching, 필요 이상의 데이터 수신)과 언더페칭(Under-fetching, 데이터 부족으로 인한 추가 요청) 문제를 해결하기 위해 Facebook에서 개발했습니다. 클라이언트는 필요한 데이터를 정확히 기술한 **쿼리(Query)**를 보내고, 서버는 정확히 그 데이터만 반환합니다.

```graphql
# 클라이언트가 필요한 필드만 정확히 요청
query {
  user(id: "123") {
    name
    email
    orders(status: PENDING) {
      id
      total
      items {
        productName
        quantity
      }
    }
  }
}

# 뮤테이션(Mutations)으로 데이터 변경
mutation {
  createUser(input: { name: "Alice", email: "alice@example.com" }) {
    id
    name
  }
}

# 실시간 업데이트를 위한 구독(Subscriptions)
subscription {
  orderStatusChanged(userId: "123") {
    orderId
    newStatus
  }
}
```

> ⚠️
> GraphQL의 트레이드오프
> GraphQL은 오버/언더페칭 문제를 해결하지만 복잡성을 가중시킵니다: N+1 쿼리 문제(DataLoader로 배치 처리 필요), 캐시 무효화의 어려움(URL 기반 캐싱 불가), 파일 업로드의 번거로움 등이 있습니다. 또한 클라이언트에게 전체 데이터 모델이 노출되므로 공개 API의 경우 보안 우려가 있을 수 있습니다. 모바일, 웹, 제3자 등 데이터 요구사항이 다양한 클라이언트가 많을 때 GraphQL을 사용하세요.

## gRPC

gRPC(Google Remote Procedure Call)는 직렬화를 위해 **Protocol Buffers**를 사용하고 전송을 위해 **HTTP/2**를 사용합니다. `.proto` 스키마 파일로부터 10개 이상의 언어로 클라이언트 및 서버 스텁(Stub)을 생성하여, 서로 다른 언어 간의 서비스 통신을 타입 안전하고 효율적으로 만듭니다.

```protobuf
// users.proto
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User); // 서버 스트리밍
  rpc CreateUser (CreateUserRequest) returns (User);
}

message GetUserRequest {
  string user_id = 1;
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  int64 created_at = 4;
}
```

Protocol Buffers는 동일한 JSON보다 5~10배 작은 바이너리 페이로드를 생성합니다. HTTP/2 멀티플렉싱(Multiplexing)을 통해 단일 TCP 연결에서 여러 RPC 호출이 가능합니다. gRPC는 기본적으로 **네 가지 통신 패턴**을 지원합니다: 단항(Unary, 요청-응답), 서버 스트리밍, 클라이언트 스트리밍, 양방향 스트리밍.

## API 버전 관리 전략(API Versioning Strategies)

API는 기존 클라이언트를 중단시키지 않고 진화해야 합니다. 주요 버전 관리 전략은 다음과 같습니다.

| 전략 | 예시 | 장점 | 단점 |
| --- | --- | --- | --- |
| URL 경로 버전 관리 | `/v1/users`, `/v2/users` | 명시적이며 로그와 문서에서 확인하기 쉬움 | URL은 버전이 아닌 리소스를 식별해야 함 |
| 쿼리 파라미터 | `/users?version=2` | 깔끔한 URL 유지 | 잊어버리기 쉽고, 캐싱이 깔끔하지 않음 |
| 헤더 버전 관리 | `API-Version: 2` | 깔끔한 URL, HTTP 의미론 준수 | 브라우저 테스트가 어렵고, 발견 가능성이 낮음 |
| 콘텐츠 협상(Content negotiation) | `Accept: application/vnd.example.v2+json` | 완전한 RESTful 방식 | 복잡하고 도구 지원이 부족함 |
| Proto 하위 호환성 | 필드 추가만 허용; 삭제나 번호 변경 금지 | 사소한 변경 시 버전 번호가 필요 없음 | 스키마 관리 규칙 준수가 필수적임 |

## 페이지네이션 패턴(Pagination Patterns)

API가 목록을 반환할 때, 페이지네이션은 응답 크기가 무한정 커지는 것을 방지합니다. 세 가지 주요 접근 방식이 있습니다.

| 패턴 | 작동 방식 | 사용 사례 | 단점 |
| --- | --- | --- | --- |
| 오프셋/리밋(Offset/limit) | `?page=3&limit=20` → `OFFSET 40 LIMIT 20` | 단순함, 모든 SQL DB와 호환됨 | 높은 오프셋에서 성능 저하; 페이지 간 항목 누락/중복 발생 가능 |
| 커서 기반(Cursor-based) | `?cursor=` (마지막 확인한 ID 포함) | 실시간 피드 (Twitter, Facebook) | 임의의 페이지로 바로 점프할 수 없음 |
| 키셋(Keyset / seek) | `?after_id=1234&limit=20` → `WHERE id > 1234` | 고성능; 데이터 추가 시에도 안정적 | 정렬 컬럼에 인덱스가 필수적임 |

> 💡
> 피드 서비스에는 커서 기반 페이지네이션을 권장합니다.
> 오프셋 페이지네이션은 새로운 항목이 추가될 때 데이터가 밀려나면서 사용자가 중복된 항목을 보거나 일부 항목을 건너뛰게 되는 문제가 있습니다. 커서 기반 페이지네이션은 안정적입니다. 커서가 결과 집합의 위치를 인코딩하므로 숫자로 된 페이지 번호보다 정확합니다. 오프셋 방식은 사용자가 임의의 페이지로 점프해야 하는, 데이터 변경이 적은 경우에만 사용하세요.

> 💡
> 면접 팁
> API 디자인 면접에서는 항상 클라이언트의 상황을 먼저 파악하세요. 사용자가 내부 마이크로서비스(gRPC 적합)인지, 다양한 데이터 요구사항이 있는 모바일 앱(GraphQL)인지, 혹은 공개된 제3자 API(REST와 OpenAPI)인지 확인하세요. 그 다음 버전 관리와 페이지네이션을 논의하세요. 보너스 팁: 변경 연산의 안전성을 위해 멱등성 키(Idempotency keys)를 언급하세요. 주문 생성이나 카드 결제처럼 멱등성이 없는 연산은 클라이언트가 생성한 키를 받아 재시도 시 중복 처리가 되지 않도록 해야 합니다.
