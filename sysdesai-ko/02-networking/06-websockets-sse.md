# 웹소켓(WebSockets) & 서버 전송 이벤트(SSE, Server-Sent Events)

> Source: https://www.sysdesai.com/learn/networking/websockets-sse

---

## 왜 실시간(Real-Time)인가?

HTTP의 요청-응답 모델은 클라이언트가 매번 상호작용을 시작해야 합니다. 하지만 채팅 메시지, 라이브 스코어, 공동 편집, 주식 시세, 주문 상태 업데이트처럼 서버가 데이터를 클라이언트에게 푸시(Push)해야 하는 애플리케이션의 경우, 클라이언트의 사전 요청 없이도 데이터를 보낼 수 있는 메커니즘이 필수적입니다.

## 웹소켓(WebSockets)

**웹소켓(WebSocket)**은 HTTP 연결을 전이중(Full-duplex), 지속적, 양방향 채널로 업그레이드하는 프로토콜(RFC 6455)입니다. 클라이언트나 서버 중 어느 쪽이든 언제든지 메시지를 보낼 수 있으며, 각 교환 시 HTTP 헤더에 따른 오버헤드가 발생하지 않습니다.

```javascript
// 클라이언트 측 웹소켓
const ws = new WebSocket('wss://chat.example.com/room/42');

ws.onopen = () => {
  ws.send(JSON.stringify({ type: 'join', userId: '123' }));
};

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  renderMessage(msg);
};

ws.onerror = (error) => console.error('WebSocket error:', error);
ws.onclose = (event) => {
  console.log('Disconnected. Code:', event.code);
  // 지수 백오프(Exponential backoff)를 이용한 재연결
};
```

- **전이중(Full duplex)**: 클라이언트와 서버가 독립적으로, 동시에 메시지를 보낼 수 있습니다.
- **낮은 오버헤드**: 핸드셰이크(Handshake) 이후 메시지는 약 800바이트의 HTTP 헤더 대신 2~14바이트의 프레이밍 오버헤드만 가집니다.
- **지속적 연결**: 메시지당 핸드셰이크 비용이 발생하지 않습니다 (HTTP/1.1 폴링과 대조적).
- **바이너리 지원**: 텍스트뿐만 아니라 바이너리 프레임(ArrayBuffer)도 전송 가능하여 게임 상태, 이미지, 오디오 전송에 유용합니다.

## 서버 전송 이벤트(SSE, Server-Sent Events)

**서버 전송 이벤트(SSE)**는 일반 HTTP를 기반으로 하는 더 단순한 프로토콜입니다. 서버는 수명이 긴 HTTP 응답을 열어두고 클라이언트에 텍스트 이벤트를 스트리밍합니다. 클라이언트는 데이터를 받기만 할 수 있는 단방향 통신이지만, 브라우저의 `EventSource` API를 통해 자동 재연결 및 이벤트 ID 추적 기능을 기본으로 제공받습니다.

```javascript
// 클라이언트: EventSource가 자동 재연결을 처리함
const evtSource = new EventSource('/api/stream/orders');

evtSource.addEventListener('order_update', (event) => {
  const update = JSON.parse(event.data);
  updateOrderStatus(update);
});

evtSource.onerror = () => {
  // EventSource는 일정 시간 후 자동으로 재연결을 시도함
  console.log('Connection lost, reconnecting...');
};
```

```text
# SSE 응답 형식 (일반 텍스트)
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

id: 1
event: order_update
data: {"orderId": "ABC", "status": "shipped"}

id: 2
event: order_update
data: {"orderId": "XYZ", "status": "delivered"}

: heartbeat comment (연결 유지를 위한 하트비트 주석)
```

## 웹소켓 vs SSE vs 폴링 비교

| 측면 | 웹소켓(WebSocket) | SSE | 롱 폴링(Long Polling) |
| --- | --- | --- | --- |
| 방향성 | 전이중 (양방향) | 서버에서 클라이언트로만 (단방향) | 서버에서 클라이언트로만 (단방향) |
| 프로토콜 | ws:// / wss:// | 일반 HTTP | 일반 HTTP |
| 브라우저 지원 | 모든 현대적 브라우저 | 모든 현대적 브라우저 (IE 제외) | 범용적 |
| 자동 재연결 | 직접 구현해야 함 | EventSource에 내장됨 | 직접 구현해야 함 |
| 프록시/방화벽 친화성 | 가끔 차단될 수 있음 | 높음 — 일반 HTTP 사용 | 높음 — 일반 HTTP 사용 |
| HTTP/2 멀티플렉싱 | 미지원 (별도 연결 필요) | 지원 | 지원 |
| 사용 사례 | 채팅, 공동 편집, 게임 | 알림, 라이브 피드, 대시보드 | WS/SSE가 차단된 환경의 대체 수단 |
| 서버 부하 | 높음 (지속적 연결 유지) | 높음 (지속적 연결 유지) | 보통 (연결 순환) |

## 웹소켓 연결 확장하기(Scaling WebSocket Connections)

웹소켓은 상태 유지(Stateful) 방식입니다. 즉, 연결이 특정 서버에 고정됩니다. 이는 수평 확장을 할 때 다음과 같은 도전 과제를 안겨줍니다.

- **세션 고정 필요**: 부하 분산기는 특정 웹소켓 클라이언트의 모든 요청을 동일한 서버로 라우팅해야 합니다. 서버가 재시작되면 연결이 끊깁니다.
- **메시지 팬아웃(Fan-out)**: 서로 다른 서버에 접속한 사용자들에게 메시지를 전달해야 하는 경우(예: 다른 서버의 사용자들이 포함된 채팅방), 서버 인스턴스 간에 메시지를 브로드캐스트하기 위한 **Redis Pub/Sub**이나 **Kafka**와 같은 펍/섭 백본이 필요합니다.
- **연결 제한**: 단일 서버는 현실적으로 10,000~100,000개의 동시 웹소켓 연결을 처리할 수 있습니다(메시지 빈도와 메모리에 따라 다름). 수평 확장을 고려하여 설계해야 합니다.
- **하트비트 및 킵얼라이브(Heartbeats and Keepalives)**: 연결 타임아웃이 있는 NAT 장치나 방화벽에서 연결이 끊어지지 않도록 주기적으로 핑(Ping) 프레임을 보내 죽은 연결을 감지하고 유지해야 합니다.

📌
실무 사례: Slack의 웹소켓 아키텍처

Slack은 실시간 메시지 전달을 위해 웹소켓을 사용합니다. 각 서버는 수만 개의 지속적 연결을 처리합니다. Redis Pub/Sub은 워크스페이스의 연결 풀에 있는 모든 서버로 메시지를 확산시킵니다. 서버에 부하가 걸리거나 재시작되면 클라이언트는 지수 백오프를 사용하여 자동으로 재연결하고, 메시지 ID를 통해 연결이 끊겼던 시점부터 놓친 메시지를 다시 받아옵니다.

> 💡
> 면접 팁
> 실시간 시스템(채팅, 알림, 실시간 협업) 설계를 요청받으면 다음 네 가지를 명확히 언급하세요: (1) 연결 관리 — 양방향성 필요에 따른 웹소켓 또는 SSE 선택, (2) 메시지 지속성 — 재연결된 클라이언트가 놓친 이벤트를 재생할 수 있도록 메시지를 저장하는 방법, (3) 팬아웃 — 여러 서버에 접속된 사용자들에게 메시지를 전달하는 방법, (4) 수평 확장 — 서버 간 메시지 전달을 위한 Redis Pub/Sub이나 Kafka 활용. 이 네 가지 포인트는 아키텍처적 성숙도를 보여줍니다.

연습해보기
[실시간 공동 문서 편집기 설계](https://www.sysdesai.com/design/new?prompt=Design%20a%20real-time%20collaborative%20document%20editor&mode=fast)
