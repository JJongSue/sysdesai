# Request-Reply Pattern

> 원문: https://www.sysdesai.com/learn/messaging-patterns/request-reply

---

## 왜 메시징을 통한 Request-Reply인가?

마이크로서비스에서 대부분의 서비스 간 통신은 동기식(synchronous)입니다. 서비스 A가 서비스 B를 호출하고 응답을 기다립니다. 보통 이는 HTTP나 gRPC로 이루어집니다. 하지만 때로는 이러한 요청-응답 상호작용을 **message broker**를 통해 라우팅하고 싶을 때가 있습니다. 이를 통해 브로커의 장점(내구성, 라우팅, 재시도)을 얻으면서 호출자가 필요로 하는 call-and-response 의미론(semantics)을 유지할 수 있습니다.

**Request-Reply** 패턴은 요청자(requester)가 request queue에 메시지를 발행하고, 전용 **reply queue**에서 응답 메시지를 기다리는 방식으로 이를 구현합니다. **Correlation ID**는 응답을 원래의 요청과 연결해 줍니다. 이 패턴은 **RPC over messaging** 패턴이라고도 불립니다.

## Pattern Flow
메시징을 통한 Request-Reply: Correlation ID가 응답을 원래의 요청과 연결합니다.

## Correlation ID

**Correlation ID**는 메시지를 보낼 때 요청자가 생성하는 UUID(또는 유사한 고유 토큰)입니다. 응답자(responder)는 응답 메시지에 이 ID를 그대로 포함하여 돌려보냅니다. 요청자는 `correlationId → pending promise/callback` 맵을 유지하며, 들어오는 응답의 correlation ID를 사용하여 올바른 대기자(waiter)를 찾아 해결(resolve)합니다.

typescript

```
// 요청자 측 — 간략한 예시
const pending = new Map<string, (result: unknown) => void>();

async function sendRequest(payload: unknown): Promise<unknown> {
  const correlationId = crypto.randomUUID();
  const replyTo = "replies.service-a." + correlationId;

  return new Promise((resolve) => {
    pending.set(correlationId, resolve);

    broker.publish("requests.service-b", {
      correlationId,
      replyTo,
      payload,
    });

    // Timeout guard — 영원히 기다리지 않음
    setTimeout(() => {
      if (pending.has(correlationId)) {
        pending.delete(correlationId);
        resolve({ error: "timeout" });
      }
    }, 5000);
  });
}

// Reply queue로부터 응답이 올 때:
broker.subscribe(myReplyQueue, (msg) => {
  const resolve = pending.get(msg.correlationId);
  if (resolve) {
    pending.delete(msg.correlationId);
    resolve(msg.payload);
  }
});
```

## Reply Queue Strategies

Reply queue에는 두 가지 일반적인 전략이 있습니다:

| 전략 | 설명 | 장점 | 단점 |
| --- | --- | --- | --- |
| Per-request temporary queue | 요청마다 새로운 일시적 큐를 생성; 응답 후 삭제 | 깔끔한 격리; 라우팅 로직 불필요 | 요청당 큐 생성 오버헤드; 높은 처리량에는 부적합 |
| Shared reply queue per service | 서비스 인스턴스당 하나의 영구적 reply queue 유지; correlation ID로 내부 라우팅 | 낮은 오버헤드; 재사용 가능 | 모든 응답이 하나의 큐로 들어옴 — correlation ID 기반의 클라이언트 측 디먹싱(demux) 필요 |

> 💡
> RabbitMQ Direct Reply-To
> RabbitMQ에는 `amq.rabbitmq.reply-to`라는 내장 최적화 기능이 있습니다. 이는 실제 큐를 생성하지 않고도 응답을 소비 중인 연결로 직접 라우팅하는 가상 큐(pseudo-queue)입니다. 이를 통해 요청당 큐 생성 오버헤드를 피하면서 패턴을 깔끔하게 유지할 수 있습니다.

## Timeouts Are Non-Negotiable

연결 종료가 실패 신호가 되는 HTTP와 달리, 큐에서 응답을 기다리는 요청은 응답자가 죽었는지 알 수 있는 내재된 신호가 없습니다. 따라서 반드시 클라이언트 측에서 **timeout**을 구현해야 합니다. 그렇지 않으면 `pending` 맵이 무한히 커지고 호출자는 무기한 차단(block)될 것입니다. 응답자의 P99 처리 지연 시간(processing latency)에 안전 여유분(보통 예상 응답 시간의 2~5배)을 더해 timeout을 설정하세요.

## Request-Reply vs Fire-and-Forget

| 항목 | Request-Reply | Fire-and-Forget |
| --- | --- | --- |
| 결합도 | 시간적 결합 (Temporal coupling, 요청자가 대기) | 완전히 분리됨 |
| 사용 사례 | 호출자가 다음 단계를 위해 결과가 필요한 경우 | 호출자가 결과가 필요 없거나 기다릴 수 없는 경우 |
| 복잡성 | 높음 (Correlation ID, reply queue, timeout) | 낮음 |
| 장애 처리 | 호출자가 즉시 장애를 인지함 (timeout을 통해) | 호출자가 처리에 성공했는지 알 수 없음 |
| 예시 | 결제 승인, 주문 유효성 검사 | 이메일 알림, 감사 로그 기록, 분석 이벤트 |

> 💡
> 인터뷰 팁
> 면접관이 '서비스 A가 다음 단계를 진행하기 위해 서비스 B의 연산 결과가 필요하다'는 시나리오를 설명한다면, 이는 request-reply 시나리오입니다. 이때 "이 과정을 비동기로 만들 수 있을까요(async request-reply)?"라고 질문해 보세요. 작업이 100ms 이상 걸린다면, 콜백을 사용하는 async request-reply가 동기식 request-reply로 호출자를 차단하는 것보다 보통 더 낫습니다.
