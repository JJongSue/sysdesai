# 채팅 시스템 (WhatsApp) 설계

> 출처: https://www.sysdesai.com/learn/case-studies/chat-system

---

## 문제 정의 (Problem Statement)

채팅 시스템은 사용자가 1:1 및 그룹 대화에서 메시지를 실시간으로 주고받을 수 있게 하며, 메시지 전달, 순서 보장, 그리고 존재 확인(presence)을 보장해야 합니다. 대규모 환경(WhatsApp은 하루 1,000억 개의 메시지를 처리함)에서 핵심 과제는 수십억 대의 장치에서 지속적이고 낮은 지연 시간의 연결을 유지하면서 메시지 유실을 방지하는 것입니다.

## 요구사항 (Requirements)

| 기능적 요구사항 (Functional) | 비기능적 요구사항 (Non-Functional) |
| --- | --- |
| 1:1 및 그룹 메시징 (최대 500명) | 메시지 전달 지연 시간 100ms 미만 (P99) |
| 메시지 전달 확인 (sent, delivered, read) | 99.99% 가용성 |
| 온라인/오프라인 상태 표시 (Presence) | 수신자가 오프라인일 때도 메시지 내구성 보장 |
| 미디어 첨부 (이미지, 비디오, 오디오) | 일일 활성 사용자(DAU) 10억 명 이상 지원 |
| 메시지 기록 / 페이지네이션 | 종단간 암호화 (End-to-end encryption) |

## 용량 산정 (Capacity Estimation)

| 지표 (Metric) | 추정치 (Estimate) |
| --- | --- |
| 일일 활성 사용자 (DAU) | 500 M |
| 일일 메시지 수 | 100 B (사용자당 하루 200개) |
| 초당 메시지 수 (피크 시 평균의 2배) | ~2.3 M msg/sec |
| 평균 메시지 크기 | 100 bytes (텍스트) + metadata |
| 일일 저장 용량 (텍스트 전용) | ~10 TB/day |
| 활성 WebSocket 연결 수 | ~500 M 동시 접속 |

## 상위 수준 아키텍처 (High-Level Architecture)
채팅 시스템 상위 수준 아키텍처 (Chat system high-level architecture)

## WebSocket 연결 관리 (WebSocket Connection Management)

HTTP의 Request/Response 방식과 달리, 채팅은 **지속적인 양방향 연결**이 필요합니다. WebSockets가 표준적인 선택입니다. 각 Chat Server는 `userId → WebSocket connection`의 인메모리 맵을 유지합니다. 수신자에게 메시지가 도착하면 시스템은 해당 사용자의 연결을 보유한 Chat Server를 찾아야 하며, 이는 **라우팅 계층 (routing layer)**을 통해 해결됩니다.

> ℹ️
> 연결 라우팅 (Connection Routing)
> 사용자의 연결 정보는 Redis Hash에 등록됩니다: `chat:conn:{userId} → serverId`. 메시지를 전달할 때, 송신 서버는 수신자의 Server ID를 조회하고 Message Queue를 통해 라우팅합니다. 항목이 존재하지 않으면 사용자가 오프라인 상태인 것이므로 Push Notification을 트리거합니다.

## 메시지 전달 보장 (Message Delivery Guarantees)

WhatsApp은 **3단계 전달 모델**을 사용합니다: sent (서버 수신), delivered (장치 수신), read (사용자 확인). 이를 위해 명시적인 ACK 프로토콜이 필요합니다:

1. 송신자가 메시지 전송 → Chat Server가 `messageId`를 할당하고 Cassandra에 저장.
2. 서버가 수신자의 샤드(shard)에 해당하는 Kafka Topic에 발행.
3. 수신자의 Chat Server가 이를 소비(consume)하고 WebSocket을 통해 전달.
4. 장치가 전달을 확인(ACK) → 서버가 메시지 상태를 `delivered`로 업데이트.
5. 사용자가 대화를 열면 → 장치가 읽음 확인(read receipt) 전송 → 상태가 `read`로 업데이트.
영수증을 포함한 종단간 메시지 전달 (End-to-end message delivery with receipts)

## 그룹 메시징 (Group Messaging)

그룹 메시지는 하나의 메시지가 N명의 수신자에게 전달되어야 하는 **Fan-out 문제**를 야기합니다. 두 가지 전략이 있습니다:

| 전략 (Strategy) | 메커니즘 | 적합한 경우 |
| --- | --- | --- |
| Fan-out on write | 전송 시 각 멤버의 Inbox에 메시지 복사 | 소규모 그룹 (100명 미만) |
| Fan-out on read | 한 개의 복사본만 저장; 클라이언트가 열 때 가져옴 | 대규모 그룹 (100~500명 이상) |

WhatsApp은 하이브리드 방식을 사용합니다: 소규모 그룹은 Kafka를 통해 Fan-out on write를 수행하고(그룹당 하나의 파티션), 매우 큰 그룹은 Pull 모델을 사용합니다. 그룹 멤버십 목록은 ZooKeeper나 MySQL을 사용하는 전용 Group Service에 저장됩니다.

## 상태 서비스 (Presence Service)

상태(online/offline/last seen)는 Redis를 사용하는 전용 서비스에서 관리됩니다. 장치가 연결되면 5초마다 Heartbeat를 보냅니다. 30초 동안 Heartbeat가 수신되지 않으면 사용자는 오프라인으로 표시됩니다. 핵심 과제는 **Thundering Herd** 문제입니다: 유명인이 온라인이 되면 수백만 명의 구독자에게 알림이 가야 합니다. 해결책으로는 Queue로 Fan-out on write를 하거나, 구독자가 대화를 열 때만 지연 전달(lazy delivery)하는 방식이 있습니다.

## 메시지 저장소: 왜 Cassandra인가? (Message Storage: Why Cassandra?)

Cassandra는 다음과 같은 이유로 메시지 저장에 이상적입니다: (1) Clustering Keys를 통해 **Time-series 워크로드**를 기본적으로 지원합니다. (2) 선형적으로 확장 가능합니다. (3) 일관성(consistency)을 조정할 수 있습니다. 스키마는 `(userId, conversationId)`를 Partition Key로, `messageId`를 Clustering Key로 사용하여 페이지네이션을 위한 효율적인 범위 스캔(range scan)을 가능하게 합니다.

sql

```sql
-- Cassandra CQL 스키마 (간략화)
CREATE TABLE messages (
  conversation_id UUID,
  message_id      TIMEUUID,    -- 정렬을 위한 시간 순서 UUID
  sender_id       UUID,
  content         TEXT,
  media_url       TEXT,
  status          TEXT,        -- 'sent' | 'delivered' | 'read'
  PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

## 확장성 고려 사항 (Scaling Considerations)

- **수평적 Chat Servers**: 인메모리 연결 맵을 제외하고는 Stateless입니다. 연결을 인식하는 Load Balancer(userId 기준 Consistent Hashing) 뒤에서 독립적으로 확장합니다.
- **Kafka Partitioning**: 대화 내 메시지 순서를 유지하기 위해 `conversationId`를 기준으로 파티셔닝합니다.
- **CDN을 통한 미디어 처리**: Chat Server를 통해 미디어를 서빙하지 마세요. S3에 업로드하고 CloudFront를 통해 서빙합니다. 보안 액세스를 위해 Pre-signed URLs를 생성합니다.
- **종단간 암호화 (End-to-end encryption)**: 장치에서 키 쌍을 생성하며, 서버는 평문을 볼 수 없습니다. Signal Protocol (Double Ratchet + X3DH key agreement)을 사용합니다.
- **Rate Limiting**: 스팸 방지를 위해 Chat Server 계층에서 사용자당 메시지 전송 속도 제한을 적용합니다.

> 💡
> 인터뷰 팁 (Interview Tip)
> 이 문제에서 많은 후보자가 WebSockets 대신 HTTP Polling이나 SSE를 사용하려다 실수합니다. 왜 WebSockets인지 명확히 하세요: 양방향, 지속성, 낮은 오버헤드의 프레이밍. 또한 연결 맵(userId → server)은 로컬이 아닌 Redis에 있어야 서버 간 메시지 라우팅이 가능하다는 점을 언급하세요.

이 패턴 연습하기
[Design a real-time messaging app like WhatsApp](https://www.sysdesai.com/design/new?prompt=Design%20a%20real-time%20messaging%20app%20like%20WhatsApp&mode=fast)
