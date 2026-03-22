# Async Request-Reply Pattern

> 원문: https://www.sysdesai.com/learn/messaging-patterns/async-request-reply

---

## 문제점: 오래 걸리는 작업들

동영상 트랜스코딩, 리포트 생성, 대규모 데이터셋에 대한 ML 추론, 배치 ETL 작업 등 일부 작업은 완료하는 데 수 초, 수 분, 또는 수 시간이 걸립니다. 이러한 시간 동안 HTTP 연결을 열어두는 것은 비실용적입니다. 클라이언트에서 timeout이 발생하거나, 로드 밸런서가 유휴(idle) 연결을 끊어버릴 수 있으며, 서버 스레드는 아무것도 하지 못한 채 대기 상태로 묶여 있게 됩니다.

**Async Request-Reply** 패턴은 클라이언트에 즉시 **job ID** (또는 상태 확인 URL)를 반환하고 백그라운드에서 작업을 처리함으로써 이 문제를 해결합니다. 클라이언트는 나중에 상태 엔드포인트를 **polling**하거나, 작업 완료 시 호출될 **webhook**을 등록하여 진행 상황을 확인할 수 있습니다.

## HTTP 202 Accepted: 진입점

전형적인 비동기 HTTP 진입점은 `202 Accepted` 상태 코드를 사용합니다. '완료'를 의미하는 `200 OK`와 달리 `202`는 '요청을 받았으며 처리 중이다'라는 의미입니다. 응답 본문(body)에는 클라이언트가 진행 상황을 확인할 때 사용할 수 있는 **status URL**이나 **job ID**가 포함되어야 합니다.

http

```
POST /api/reports/generate
Content-Type: application/json

{ "reportType": "annual", "year": 2025 }

──── Response ────
HTTP/1.1 202 Accepted
Location: /api/reports/status/job_7f3c2a
Retry-After: 5

{
  "jobId": "job_7f3c2a",
  "status": "queued",
  "statusUrl": "https://api.example.com/api/reports/status/job_7f3c2a",
  "estimatedSeconds": 30
}
```

## Full Async Request-Reply Flow
Polling을 사용한 Async Request-Reply: 클라이언트는 즉시 job ID를 받고 완료될 때까지 폴링합니다.

## Polling vs Webhooks

작업이 제출되면 클라이언트는 작업이 언제 끝나는지 알아야 합니다. 여기에는 두 가지 주요 접근 방식이 있습니다:

| 접근 방식 | 작동 방식 | 클라이언트 요구사항 | 적합한 용도 |
| --- | --- | --- | --- |
| Polling | 상태가 `complete` 또는 `failed`가 될 때까지 상태 엔드포인트를 반복적으로 호출 | 모든 HTTP 클라이언트; 인바운드 연결 불필요 | 브라우저 클라이언트, 모바일 앱, 단순 스크립트 |
| Webhook (callback) | 작업 완료 시 서버가 미리 등록된 콜백 URL로 결과를 POST | 공개적으로 접속 가능한 HTTPS 엔드포인트 노출 필요 | 서버 간 통합, CI/CD 파이프라인, 결제 처리 시스템 (Stripe, PayPal) |
| WebSocket / SSE | 서버가 지속적인 연결을 통해 상태 업데이트를 푸시 | 지속적인 연결 유지 필요 | 실시간 진행률 표시 바, 대시보드 UI |

> ℹ️
> Polling Interval 전략
> 폴링에는 exponential backoff를 사용하세요: 1초에서 시작하여 최대치(예: 30초)까지 시도마다 주기를 두 배로 늘립니다. 이는 작업이 예상보다 오래 걸릴 경우 상태 엔드포인트에 과부하를 주는 것을 방지합니다. 클라이언트에 힌트를 주기 위해 202 응답에 `Retry-After` 헤더를 포함하세요.

## Status Endpoint Design

상태 엔드포인트는 클라이언트가 기계적으로 파싱할 수 있는 일관된 구조를 반환해야 합니다. 최소한 `status` (queued, processing, complete, failed), `progress` (선택 사항, 0-100), `resultUrl` (완료 시), `error` (실패 시)를 포함해야 합니다.

typescript

```
// GET /api/jobs/:jobId
interface JobStatus {
  jobId: string;
  status: "queued" | "processing" | "complete" | "failed";
  progress?: number;          // 처리 중일 때 0-100
  resultUrl?: string;         // 완료 시 설정됨
  error?: string;             // 실패 시 설정됨
  createdAt: string;          // ISO8601
  completedAt?: string;       // 완료 시 ISO8601
}

// 응답 예시:
// 102 Processing — 실제 작업 중 (선택 사항)
// 200 { status: "complete", resultUrl: "/results/job_7f3c2a.pdf" }
// 200 { status: "failed", error: "2025년 리포트 데이터를 찾을 수 없습니다" }
```

## Job State Storage

작업 상태는 API 서버가 읽을 수 있고 워커(worker)가 업데이트할 수 있는 곳에 영구적으로 보관되어야 합니다. 일반적인 선택지는 다음과 같습니다:

- **Redis** (+ TTL): 빠른 읽기, 완료된 작업의 자동 만료. 수 분에서 수 시간 정도 유지되는 짧은 수명의 작업에 적합합니다.
- **관계형 DB 테이블** (`jobs` 테이블): 내구성 있고 쿼리가 가능하며 다양한 필터링을 지원합니다. 재시작 후에도 유지되어야 하거나 감사 이력(audit history)이 필요한 작업에 사용합니다.
- **DynamoDB / Firestore**: 서버리스이며 운영 오버헤드가 없습니다. 대규모 고부하 작업에 적합합니다.

## Idempotency Keys for Retried Submissions

클라이언트가 202 응답을 받지 못한 경우(네트워크 장애, timeout 등) 최초 POST 요청을 재시도할 수 있습니다. 보호 장치가 없다면 중복 작업이 생성될 것입니다. 해결책은 **idempotency key**입니다. 클라이언트가 UUID를 생성하여 헤더(`Idempotency-Key: `)로 보냅니다. 서버는 이 키를 저장하고, TTL 윈도우 내에 동일한 키가 다시 제출되면 기존 job ID를 반환합니다.

> 💡
> 인터뷰 팁
> 이 패턴은 거의 모든 대규모 시스템 설계 면접(ML 예측 API, 동영상 처리, 배치 리포팅, 결제 처리 등)에서 등장합니다. 202 Accepted 응답으로 시작하여, 상태 폴링 엔드포인트를 설명하고, 대안으로 webhook을 언급한 뒤, 재시도를 위한 idempotency key를 잊지 말고 설명하세요. 이 네 가지 포인트는 실제 운영 경험이 있음을 보여줍니다.
