# Sequential Convoy Pattern

> 원문: https://www.sysdesai.com/learn/messaging-patterns/sequential-convoy

---

## 순서 보장 문제 (The Ordering Problem)

여러 소비자가 있는 메시지 큐는 메시지를 병렬로 처리하며, 이는 처리량을 획기적으로 향상시킵니다. 하지만 병렬 처리는 순서를 파괴합니다. 소비자 1이 메시지 3 처리를 끝내기 전에 소비자 2가 메시지 1 처리를 끝낼 수 있습니다. 분석, 알림, 캐시 업데이트 같은 많은 사용 사례에서는 괜찮지만, 어떤 사례에서는 치명적일 수 있습니다.

금융 계좌를 생각해 보세요. **$200 출금**(실패했어야 함)보다 **$100 입금**이 순서가 바뀌어 먼저 처리된다면, 잘못된 장부 상태가 됩니다. 또는 이커머스 주문 수명 주기인 `OrderPlaced → PaymentConfirmed → Shipped → Delivered`는 각 개별 주문에 대해 엄격한 순서대로 처리되어야 합니다. 다만 서로 다른 고객의 주문은 완전히 독립적이므로 자유롭게 병렬화할 수 있습니다.

> ℹ️
> 핵심 통찰
> 모든 메시지에 대해 전역적인 순서 보장이 필요한 경우는 드뭅니다. 거의 항상 특정 사용자 ID, 주문 ID, 계좌 ID, 또는 세션 ID와 같은 논리적 그룹 내에서의 순서 보장만 필요합니다. **Sequential Convoy** 패턴은 이를 활용합니다. 즉, 그룹 내 순서는 보장하면서 서로 다른 그룹은 병렬로 처리하는 것입니다.

## Sequential Convoy의 작동 방식

이 패턴은 모든 메시지를 순차적으로 처리되어야 하는 논리적 그룹인 **convoy**에 할당하는 방식으로 작동합니다. 각 메시지에는 **partition key**(예: `orderId`, `userId`, `accountId`)가 부여됩니다. 브로커는 이 키를 사용하여 동일한 키를 가진 모든 메시지를 동일한 파티션이나 큐로 라우팅합니다. 여기서 메시지들은 FIFO convoy를 형성하며 단일 소비자 인스턴스에 의해 처리됩니다.

Sequential Convoy: partition key가 동일한 orderId 메시지를 동일한 파티션으로 라우팅하여 순차적 처리를 보장합니다.

## 구현: Kafka Partitioning

Kafka는 **partition keys**를 통해 이를 네이티브하게 구현합니다. 메시지를 생성할 때 키(예: `orderId`)를 지정하면, Kafka는 이 키를 해싱하여 파티션 번호를 결정합니다. 동일한 키를 가진 모든 메시지는 항상 동일한 파티션에 도달합니다. 파티션 내에서 메시지는 offset에 의해 엄격하게 정렬됩니다. Consumer group은 각 파티션에 정확히 한 명의 소비자를 할당하므로, 추가적인 조정 없이도 순서가 보장됩니다.

typescript

```
// Kafka producer: orderId를 partition key로 사용
await producer.send({
  topic: "order-events",
  messages: [
    {
      key: order.id,          // ← partition key; 동일한 orderId → 동일한 파티션
      value: JSON.stringify({
        type: "OrderPlaced",
        orderId: order.id,
        customerId: order.customerId,
        timestamp: Date.now(),
      }),
    },
  ],
});

// "ord-123" 주문에 대한 모든 이벤트는 동일한 파티션에 도달하며
// 동일한 소비자 인스턴스에 의해 순서대로 처리됩니다.
```

## 구현: Message Group을 활용한 SQS FIFO Queue

AWS SQS FIFO 큐는 **Message Group IDs**를 통해 Sequential Convoy 패턴을 구현합니다. 동일한 `MessageGroupId`를 가진 메시지들은 엄격한 FIFO 순서로 처리됩니다. 서로 다른 메시지 그룹은 동시에 처리될 수 있습니다. 트레이드오프: SQS FIFO 큐는 초당 3,000건(배치 처리 미사용 시 300건)의 트랜잭션으로 제한되는 반면, 많은 파티션을 가진 Kafka는 훨씬 더 높은 확장이 가능합니다.

typescript

```
// SQS FIFO: MessageGroupId로 주문별 순서 강제
await sqs.sendMessage({
  QueueUrl: "https://sqs.us-east-1.amazonaws.com/123456789/orders.fifo",
  MessageBody: JSON.stringify({ type: "PaymentConfirmed", orderId: "ord-456" }),
  MessageGroupId: "ord-456",                     // ← 이 주문에 대한 모든 이벤트의 순서 보장
  MessageDeduplicationId: crypto.randomUUID(),   // ← FIFO에 필수; 중복 방지
}).promise();
```

## Hotspot / Hot Partition 문제

Sequential Convoy 패턴은 미묘한 실패 모드를 유발할 수 있습니다. 만약 특정 partition key가 극도로 빈번하게 사용된다면(예: 전체 이벤트의 90%를 생성하는 인기 판매자), 해당 키를 처리하는 파티션은 병목 지점이 되고 다른 파티션들은 유휴 상태가 됩니다. 이것이 **hot partition** 문제입니다.

| 완화 방법 | 설명 | 트레이드오프 |
| --- | --- | --- |
| Sub-partition keys | 키 뒤에 접미사 추가: `sellerId-0`, `sellerId-1` | 여러 파티션의 결과를 집계해야 함; sub-partition 내에서만 순서 보장 |
| 파티션 수 증가 | 더 많은 파티션 = 다양한 키에 대해 더 나은 분산 | 단일 hot key에는 도움이 되지 않음; 나중에 가동 중단 없이 파티션 수를 줄일 수 없음 |
| 별도의 topic/queue | hot key를 전용 고처리량 topic으로 라우팅 | 운영 복잡성 증가; 특수 케이스 처리가 필요함 |

## Sequential Convoy를 사용해야 하는 경우

- **금융 장부**: 동일 계좌에 대한 차변(debit)과 대변(credit)은 순서대로 적용되어야 함
- **주문 수명 주기 이벤트**: 각 주문에 대한 `Placed → Paid → Fulfilled → Delivered`
- **사용자 세션 이벤트**: 세션 내의 클릭스트림 이벤트는 순서대로 리플레이되어야 함
- **데이터베이스 변경 이벤트 (CDC)**: CDC 스트림의 행 수준 변경사항은 기본 키(primary key)별로 순서대로 적용되어야 함
- **게임 상태 업데이트**: 플레이어 상태 변화는 플레이어별로 순차적으로 적용되어야 함

> 💡
> 인터뷰 팁
> 면접관이 분산 메시지 시스템에서 순서를 보장하는 방법을 묻는다면, '소비자가 하나인 단일 큐를 사용한다'고 답하지 마세요. 이는 처리량을 죽이는 일입니다. 대신 **Sequential Convoy**를 설명하세요. 논리적 키로 파티셔닝하여 그룹 내 순서는 유지하면서 서로 다른 그룹은 병렬로 처리한다고 답하세요. 그런 다음 구체적인 구현 사례로 Kafka의 partition key나 SQS FIFO의 MessageGroupId를 언급하세요.
