# Claim Check 패턴

> Source: https://www.sysdesai.com/learn/performance-scalability/claim-check

---

## 대용량 메시지 문제

메시지 브로커(Message Broker)에는 크기 제한이 있습니다. AWS SQS는 **256 KB**, Azure Service Bus도 **256 KB**(프리미엄은 1 MB), Kafka의 기본값은 **1 MB**(설정 가능하지만 메시지가 클수록 처리량이 저하됩니다)로 제한됩니다. 서비스 간에 대용량 페이로드(payload) — 고해상도 이미지, PDF 문서, 수천 개의 항목을 담은 JSON 객체 등 — 를 주고받아야 할 때 이 제한이 블로킹 요소가 됩니다.

대용량 페이로드를 메시지에 직접 포함하면 브로커 성능도 저하됩니다. 큰 메시지는 브로커 버퍼에서 더 많은 메모리를 차지하고, 직렬화/역직렬화 비용이 증가하며, 처리가 느린 컨슈머(Consumer)가 파티션 전체를 블로킹할 수 있습니다.

## Claim Check 패턴 동작 원리

**Claim Check 패턴**은 메시지 페이로드와 메타데이터를 분리합니다. 대용량 페이로드는 외부 스토리지(S3, Azure Blob, GCS)에 저장하고, 브로커를 통해 전송되는 메시지에는 페이로드를 조회할 수 있는 **참조값**('claim check' — URL 또는 키)만 포함합니다. 컨슈머는 이 claim check를 사용해 처리가 필요할 때 외부 스토리지에서 페이로드를 직접 가져옵니다.

Claim Check: 브로커는 참조값만 전달하고, 실제 페이로드는 Blob 스토리지에 보관

## 구현 예시

```python
import boto3, uuid, json

s3 = boto3.client("s3")
sqs = boto3.client("sqs")
BUCKET = "my-payloads"
QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/123/my-queue"

# 프로듀서: 페이로드 저장 후 claim check 전송
def send_large_message(payload: dict) -> None:
    payload_key = f"payloads/{uuid.uuid4()}.json"

    # 1단계: 전체 페이로드를 S3에 업로드
    s3.put_object(
        Bucket=BUCKET,
        Key=payload_key,
        Body=json.dumps(payload),
        ContentType="application/json"
    )

    # 2단계: claim check만 포함한 경량 메시지 전송
    message = {
        "type": "order-submitted",
        "payload_location": f"s3://{BUCKET}/{payload_key}",
        "order_id": payload["order_id"],
    }
    sqs.send_message(
        QueueUrl=QUEUE_URL,
        MessageBody=json.dumps(message)
    )

# 컨슈머: claim check를 사용해 페이로드 조회
def process_message(sqs_message: dict) -> None:
    body = json.loads(sqs_message["Body"])
    bucket, key = parse_s3_uri(body["payload_location"])

    # S3에서 전체 페이로드 가져오기
    response = s3.get_object(Bucket=bucket, Key=key)
    payload = json.loads(response["Body"].read())

    # 전체 페이로드 처리
    process_order(payload)

    # 선택 사항: S3에서 페이로드 삭제
    s3.delete_object(Bucket=bucket, Key=key)
```

## 운영 시 고려사항

- **페이로드 생명주기**: 페이로드 삭제 시점을 결정해야 합니다. 소비 직후 삭제하면 스토리지 비용은 절감되지만 재처리(replay)가 불가합니다. S3 수명 주기 규칙으로 7일 후 만료시키는 방식이 일반적인 중간점입니다.
- **접근 제어**: 컨슈머가 Blob 스토리지에 읽기 권한을 갖도록 해야 합니다. 컨슈머가 신뢰할 수 없거나 크로스 계정인 경우 만료 시간이 짧은 Pre-signed URL을 사용하세요.
- **장애 처리**: 컨슈머가 페이로드를 가져올 때 S3를 사용할 수 없으면 메시지를 처리할 수 없습니다. 컨슈머는 NACK하고 브로커의 재시도 메커니즘에 의존해야 합니다.
- **멱등성(Idempotency)**: 컨슈머가 메시지를 처리한 후 Blob을 삭제하기 전에 충돌하면 재시도 시 동일 페이로드가 재처리될 수 있습니다. 다운스트림 작업이 멱등성을 보장하는지 확인하세요.

> ℹ️
> AWS S3 + SQS Extended Client Library
> AWS는 Claim Check 패턴을 투명하게 구현하는 공식 SQS Extended Client Library(Java, Python)를 제공합니다. 256 KB를 초과하는 메시지를 자동으로 S3에 업로드하고 메시지 본문을 S3 포인터로 대체합니다. 컨슈머도 동일한 라이브러리를 사용하며, 메시지를 반환하기 전에 S3에서 투명하게 가져옵니다. 이를 통해 애플리케이션 코드의 보일러플레이트(boilerplate)를 제거할 수 있습니다.

> 💡
> 인터뷰 팁
> Claim Check 패턴은 큐를 통해 서비스 간에 대용량 파일을 전달하는 시스템(비디오 처리 파이프라인, 문서 분석 워크플로, 이미지 리사이징 서비스)을 설계할 때 등장합니다. 메시지 페이로드가 브로커 한도를 초과할 가능성이 있으면 적극적으로 언급하세요. 스토리지 + 참조 분리 방식, 생명주기 관리 문제, 장애 처리 고려사항을 알고 있음을 보여주면 프로덕션 수준의 사고력을 입증할 수 있습니다.
