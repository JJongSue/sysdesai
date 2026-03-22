# 롱 폴링(Long Polling) vs 숏 폴링(Short Polling)

> Source: https://www.sysdesai.com/learn/networking/polling-patterns

---

## 왜 폴링(Polling)인가?

웹소켓(WebSockets)과 SSE are powerful, but they require persistent connections and specialized infrastructure. 폴링은 실시간에 가까운 데이터를 얻기 위한 가장 오래되고 단순한 방식입니다. 클라이언트가 주기적으로 서버에 변경 사항이 있는지 묻는 방식이죠. 푸시(Push) 기반 방식보다 효율성은 떨어지지만, 폴링은 여전히 많은 실무 상황에서 유효합니다.

> ℹ️
> 폴링이 적합한 경우
> 다음과 같은 상황에서 폴링이 적합합니다: 기업용 방화벽이나 프록시로 인해 웹소켓이 차단된 경우, 업데이트 빈도가 낮은 경우(몇 분 단위), 효율성보다 인프라의 단순함이 중요한 간단한 내부 도구를 구축하는 경우, 혹은 매우 오래된 브라우저 환경을 지원해야 하는 경우. 유의미한 트래픽이 발생하면서 30초보다 빠른 업데이트가 필요한 모든 경우에는 SSE나 웹소켓을 권장합니다.

## 숏 폴링(Short Polling)

**숏 폴링(Short Polling)**은 정해진 간격으로 서버에 HTTP 요청을 보냅니다. 서버는 변경 사항 유무와 관계없이 즉시 응답하며, 클라이언트는 다음 요청을 예약합니다.

```javascript
// 숏 폴링: 5초마다 확인
function startShortPolling(userId) {
  const poll = async () => {
    try {
      const response = await fetch(`/api/notifications/${userId}`);
      const data = await response.json();
      if (data.notifications.length > 0) {
        renderNotifications(data.notifications);
      }
    } catch (err) {
      console.error('Poll failed:', err);
    }
  };

  poll(); // 즉시 첫 번째 호출
  return setInterval(poll, 5000); // 이후 5초마다 반복
}
```

- **단순함**: 표준 HTTP 요청을 사용하므로 어디서나 작동하고 구현 및 캐싱이 쉽습니다.
- **예측 가능한 서버 부하**: 요청이 일정한 비율로 들어오므로 용량 계획(Capacity planning)이 명확합니다.
- **비효율적**: 대부분의 요청이 빈 응답을 반환하여 서버 CPU, 네트워크 대역폭, 클라이언트 배터리를 낭비합니다.
- **지연 시간(Latency)**: 평균적으로 이벤트 발생 후 `주기 / 2`만큼 기다려야 확인할 수 있습니다. 5초 간격일 경우 평균 지연 시간은 2.5초입니다.

## 롱 폴링(Long Polling)

**롱 폴링(Long Polling)**은 서버가 새로운 데이터가 생길 때까지 요청을 열어두어 숏 폴링의 단점을 개선한 방식입니다. 클라이언트가 요청을 보내면, 서버는 보낼 데이터가 생기거나 타임아웃(보통 30~60초)이 발생할 때까지 응답하지 않습니다. 응답이 도착하면 클라이언트는 즉시 다음 요청을 보냅니다.

```javascript
// 롱 폴링 클라이언트 구현
async function longPoll(userId, lastEventId = null) {
  while (true) {
    try {
      const url = new URL(`/api/events/${userId}`);
      if (lastEventId) url.searchParams.set('since', lastEventId);

      const response = await fetch(url, {
        signal: AbortSignal.timeout(65000) // 서버 타임아웃보다 약간 길게 설정
      });

      if (response.ok) {
        const data = await response.json();
        if (data.events.length > 0) {
          lastEventId = data.events.at(-1).id;
          processEvents(data.events);
        }
        // 즉시 재연결
      }
    } catch (err) {
      if (err.name !== 'TimeoutError') {
        // 실제 오류 발생 시 지수 백오프 적용
        await sleep(Math.min(retries++ * 1000, 30000));
      }
    }
  }
}
```

## 모든 방식 비교

| 측면 | 숏 폴링 | 롱 폴링 | SSE | 웹소켓(WebSocket) |
| --- | --- | --- | --- | --- |
| 지연 시간 | 최대 주기만큼 발생 (높음) | 실시간에 가까움 (낮음) | 실시간에 가까움 (낮음) | 실시간 (가장 낮음) |
| 1만 명 접속 시 서버 연결 | 낮음 (간헐적 요청) | 높음 (1만 개 연결 유지) | 높음 (1만 개 연결 유지) | 높음 (1만 개 연결 유지) |
| 서버 복잡도 | 매우 낮음 | 보통 | 낮음 | 높음 |
| 프록시/방화벽 호환성 | 보편적 | 대부분 양호 | 대부분 양호 | 가끔 차단됨 |
| 자동 재연결 | 해당 없음 (클라이언트가 제어) | 수동 구현 | 내장됨 | 수동 구현 |
| 양방향 통신 | 불가 | 불가 (우회책: 별도 POST 요청) | 불가 | 가능 |
| 브라우저 지원 | 보편적 | 보편적 | 모든 현대적 브라우저 | 모든 현대적 브라우저 |

## 의사 결정 프레임워크(Decision Framework)

면접에서 실시간 메커니즘을 선택할 때 다음의 사고 모델을 활용하세요:

1. **1초 미만의 양방향 통신이 필요한가?** → 웹소켓 (게임, 공동 편집, 사용자 입력이 잦은 라이브 채팅).
2. **실시간성이 필요하면서 서버에서 클라이언트로만 보내는가?** → SSE (알림, 라이브 대시보드, 주문 추적).
3. **웹소켓/SSE가 차단되거나 지원되지 않는가?** → 롱 폴링 (기업 환경, 레거시 시스템).
4. **업데이트 빈도가 낮은가(1분 이상)?** → 숏 폴링 (빌드 상태, 이메일 동기화, 날씨 업데이트).
5. **푸시 방식이 과한가?** → 웹훅(Webhooks) + 필요 시 수동 조회 (제3자 통합, 배치 작업).

📌
과거 사례: Gmail의 롱 폴링 활용

웹소켓이 표준화되기 전, Gmail은 새 이메일 알림을 전달하기 위해 롱 폴링을 사용했습니다. 클라이언트가 HTTP 요청을 보내면 서버는 최대 60초 동안 이를 열어두었습니다. 새 이메일이 도착하면 서버가 응답하고 클라이언트는 즉시 재연결했습니다. 이를 통해 지속적인 연결 없이도 실시간에 가까운 이메일 전달이 가능했습니다. 오늘날 Gmail은 다양한 기능에 웹소켓과 SSE를 혼합하여 사용합니다.

> 💡
> 면접 팁
> 면접관이 폴링을 직접 구현하라고 하는 경우는 드물지만, '실시간 업데이트를 어떻게 처리할 것인가?'라는 질문은 자주 나옵니다. 이때 숏 폴링(가장 단순하지만 낭비가 심함), 롱 폴링(지연 시간 개선, 서버 리소스 사용), SSE(서버 푸시의 단순함과 자동 재연결), 웹소켓(강력한 전이중 통신)의 트레이드오프 스펙트럼을 알고 있음을 보여주세요. 모든 상황에 웹소켓을 적용하기보다 양방향성, 지연 시간, 클라이언트 제약 사항 등 요구사항에 근거하여 선택하는 모습을 보여주는 것이 중요합니다.
