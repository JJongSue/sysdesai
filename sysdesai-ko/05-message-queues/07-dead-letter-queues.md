# Dead Letter Queues & Retry Patterns(데드 레터 큐와 재시도 패턴)

> 출처: https://www.sysdesai.com/learn/message-queues/dead-letter-queues

---

## 메시지 처리 실패 시 어떤 일이 일어나나요?

Consumer(소비자)의 장애는 피할 수 없습니다. 데이터베이스를 일시적으로 사용할 수 없거나, 하류 API에서 속도 제한(Rate limit)이 걸리거나, 버그로 인해 처리되지 않은 예외가 발생할 수 있습니다. Consumer가 메시지에 대해 부정 확인 응답(Nack)을 보내거나 가시성 타임아웃(Visibility timeout) 전에 확인 응답(Ack)을 보내지 못하면, Broker(브로커)는 해당 메시지를 재전송합니다. 전략이 없다면 단 하나의 잘못된 메시지가 Consumer를 무한 재시도 루프에 빠뜨려 큐의 다른 모든 메시지를 차단할 수도 있습니다.

## 지수 백오프를 사용한 재시도(Retry with Exponential Backoff)

첫 번째 방어선은 **Exponential backoff(지수 백오프)와 Jitter(지터)를 포함한 재시도**입니다. 즉시 재시도하여 장애가 발생한 의존 시스템에 부하를 주는 대신, 시도 횟수에 따라 대기 시간을 기하급수적으로 늘립니다. **Jitter(지터)**(랜덤 오프셋)는 의존 시스템이 복구되었을 때 수백 개의 Consumer가 동시에 재시도하여 다시 시스템을 마비시키는 **Thundering herd(천둥 떼)** 문제를 방지합니다.

```python
import time
import random

def process_with_retry(message, max_retries=5):
    for attempt in range(max_retries):
        try:
            process_message(message)
            return  # 성공
        except TransientError as e:
            if attempt == max_retries - 1:
                raise  # 재시도 횟수 초과

            base_delay = 2 ** attempt       # 1, 2, 4, 8, 16 초
            jitter = random.uniform(0, 1)   # 랜덤 0-1 초
            delay = base_delay + jitter

            print(f"시도 {attempt + 1} 실패. {delay:.2f}초 후 재시도합니다.")
            time.sleep(delay)
        except PermanentError as e:
            # 영구적인 에러(예: 잘못된 메시지 형식)는 재시도하지 않음
            send_to_dead_letter_queue(message, error=str(e))
            return
```

> 💡
> 일시적 에러와 영구적 에러를 구분하세요.
> 모든 에러에 대해 재시도를 수행해서는 안 됩니다. **Transient error(일시적인 에러)**(네트워크 타임아웃, 속도 제한, DB 연결 거부 등)는 재시도할 가치가 있습니다. 하지만 **Permanent error(영구적인 에러)**(잘못된 메시지 형식, 스키마 검증 실패, 비즈니스 규칙 위반 등)는 아무리 재시도해도 성공하지 못하며 리소스만 낭비할 뿐입니다. 영구적인 실패는 즉시 DLQ로 보내야 합니다.

## Dead Letter Queues(DLQ, 데드 레터 큐)

**Dead Letter Queue(데드 레터 큐)**는 최대 재시도 횟수를 초과했거나 처리 불가능한 것으로 명시적으로 거부된 메시지들이 라우팅되는 특수한 큐입니다. DLQ는 격리 구역 역할을 합니다. 메시지가 유실되지 않으면서도 메인 큐를 차단하지 않게 해줍니다. 엔지니어는 나중에 DLQ의 메시지를 검사하고 버그나 데이터 문제를 수정한 뒤 다시 재생(Replay)할 수 있습니다.

DLQ 흐름: N번의 재시도 후에도 실패한 메시지는 검사 및 수동 재생을 위해 DLQ로 라우팅됩니다.

## 실무에서의 DLQ: SQS와 RabbitMQ

**Amazon SQS**에서는 메인 큐에 `maxReceiveCount`와 함께 DLQ를 구성합니다. 성공적인 Ack 없이 해당 횟수만큼 메시지가 수신되면 SQS는 자동으로 이를 DLQ로 옮깁니다. **RabbitMQ**에서는 큐에 `x-dead-letter-exchange`와 `x-max-length` 또는 `x-message-ttl`을 설정합니다. 만료되거나 다시 큐에 넣지 않고 Nack된 메시지는 데드 레터 익스체인지로 이동합니다.

| 시스템 | DLQ 구성 방식 | 핵심 설정 |
| --- | --- | --- |
| Amazon SQS | 소스 큐에 DLQ ARN 설정 | maxReceiveCount (예: 5) |
| RabbitMQ | 큐 선언 시 x-dead-letter-exchange 설정 | x-max-length 또는 재처리 없는 nack |
| Kafka | 앱 수준: 예외 처리 후 DLQ 토픽으로 발행 | 내장 DLQ 없음 — Consumer에서 구현 |
| Azure Service Bus | 내장된 데드 레터 서브 큐 | MaxDeliveryCount (기본 10) |

> ⚠️
> Kafka는 기본 DLQ 기능이 없습니다.
> Kafka에는 내장된 데드 레터 메커니즘이 없습니다. 애플리케이션이 예외를 잡아 실패한 메시지를 별도의 DLQ 토픽(예: `orders-dlq`)으로 명시적으로 발행해야 합니다. Spring Kafka의 `DefaultErrorHandler`나 Confluent의 `DeadLetterPublishingRecoverer`와 같은 라이브러리가 이 패턴을 기본적으로 제공합니다.

## 포이즌 메시지 처리(Poison Message Handling)

**Poison message(포이즌 메시지)**는 버그나 손상된 데이터로 인해 Consumer를 항상 충돌하게 만드는 메시지입니다. 보호 장치가 없다면 포이즌 메시지는 무한한 '충돌-재시작' 루프를 유발하여 다른 메시지 처리를 방해합니다. 완화 전략은 다음과 같습니다:

- **최대 수신 횟수/최대 전달 횟수 설정** — N번 실패 후 DLQ로 라우팅
- **수집 시점의 스키마 검증** — 처리 큐에 들어가기 전에 잘못된 형식의 메시지 거부
- **모든 예외 처리(Catch all)** — 처리되지 않은 예외로 인해 의도치 않게 메시지가 재전송되지 않도록 함
- **Consumer의 서킷 브레이커(Circuit breaker)** — 에러율이 급증하면 소비를 일시 중단하여 연쇄 장애 방지

## 멱등성: 안전한 재시도의 초석(Idempotency: The Cornerstone of Safe Retry)

At-least-once(최소 한 번) 전달 방식은 중복이 발생할 수 있음을 의미하므로, Consumer는 반드시 **Idempotent(멱등성)**를 가져야 합니다. 즉, 동일한 메시지를 여러 번 처리해도 한 번 처리한 것과 결과가 같아야 합니다. 접근 방식은 다음과 같습니다:

- **DB의 멱등성 키** — 처리된 이벤트 테이블에 `eventId`를 저장하고 처리 전 확인합니다. 이미 처리되었다면 건너뛰고 Ack만 보냅니다.
- **Insert 대신 Upsert 사용** — `INSERT ... ON CONFLICT DO NOTHING`이나 `MERGE`를 사용하여 행 중복을 방지합니다.
- **자연스러운 멱등성** — 일부 작업은 본질적으로 멱등성을 가집니다: 상태 필드를 두 번 `CONFIRMED`로 설정해도 결과는 변하지 않습니다.
- **조건부 쓰기(Conditional writes)** — 쓰기 조건에 예상되는 상태를 포함합니다(DynamoDB의 낙관적 잠금 / 조건부 Put).

멱등성 소비자 패턴: At-least-once 재전송을 안전하게 처리하기 위해 처리 전 이벤트 ID를 확인합니다.

## 완벽한 재시도 아키텍처(Complete Retry Architecture)

운영 수준의 재시도 전략은 일반적으로 대기 시간이 늘어나는 **여러 개의 재시도 큐**를 사용합니다. Consumer 내부에서 sleep을 사용하여 스레드를 점유하는 대신, 메시지를 `retry-30s` 큐로 보내고 일정 시간 후 소비하여 재시도합니다. 계속 실패하면 `retry-5m`, `retry-1h`를 거쳐 마지막으로 DLQ로 이동합니다.

📌
SQS를 사용한 지연 재시도

SQS는 **Delay Queue(지연 큐)**(0-900초 지연)와 **메시지 타이머**(메시지별 지연)를 지원합니다. 점진적 백오프를 위해 여러 개의 SQS 큐와 Consumer 내의 라우팅 로직을 사용하세요: 실패 → retry-queue-30s → retry-queue-5m → dlq.

> 💡
> 인터뷰 팁
> 데드 레터 큐와 재시도 패턴은 사실 '신뢰성'에 관한 질문입니다. 면접관은 여러분이 실패 시나리오를 고려하는지 테스트하고 있습니다. 다음 사항을 반드시 언급하세요: (1) 지터가 포함된 지수 백오프, (2) 재시도 횟수 초과 시 DLQ 사용, (3) Consumer의 멱등성 보장, (4) DLQ 깊이에 대한 알람 설정. 가산점 포인트: 일시적 에러와 영구적 에러의 차이점을 언급하고, Kafka는 앱 수준에서 DLQ를 구현해야 한다는 점을 덧붙이세요.
