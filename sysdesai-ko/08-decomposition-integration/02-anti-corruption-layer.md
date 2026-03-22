# Anti-Corruption Layer

> 원문: https://www.sysdesai.com/learn/decomposition-integration/anti-corruption-layer

---

## 문제점: 외부 모델이 당신의 모델을 오염시킬 때

모든 시스템은 결국 다루기 까다로운 대상과 통합하게 됩니다. 암호 같은 데이터 모델을 가진 레거시 ERP 시스템, 자신들만의 'transaction' 개념을 가진 제3자 payment gateway, 또는 완전히 다른 용어를 사용하는 인수된 회사의 API 등이 그 예입니다. 만약 그들의 개념을 당신의 깨끗한 domain model에 직접 매핑한다면, 그들의 복잡함이 유출되어 당신의 추상화를 오염시킬 것입니다. 바로 이때 **Anti-Corruption Layer (ACL)**이 필요합니다.

ACL은 Eric Evans가 소개한 **Domain-Driven Design (DDD)**의 용어입니다. 이는 당신의 bounded context와 외부 시스템 사이에 위치하는 변환 계층입니다. 외부 시스템의 데이터 구조와 개념을 당신의 도메인 언어로 변환하여, 반대편에 무엇이 있든 당신의 domain model이 깨끗하게 유지되도록 합니다.

## ACL 아키텍처

ACL은 외부 모델로부터 당신의 도메인을 보호하는 facade, translator, adapter를 포함합니다.
ACL은 세 가지 내부 빌딩 블록으로 구성됩니다:

- **Facade** — 외부 시스템의 인터페이스를 단순화하여 도메인에서 복잡성을 숨깁니다.
- **Translator** — 두 모델 간의 데이터 구조와 개념을 변환합니다 (예: `ExternalOrder` → `PurchaseOrder`).
- **Adapter** — 프로토콜이나 기술적 차이를 처리합니다 (예: SOAP → REST, XML → JSON).

## 구체적인 예: Payment Gateway 통합

당신의 domain model에 `amount`, `currency`, `customerId`, `status` 필드를 가진 `Payment` 엔티티가 있다고 가정해 봅시다. 당신의 payment gateway(예: Stripe)는 `PaymentIntent`, `charge`, `metadata`, 그리고 `requires_payment_method`와 같은 상태 코드를 사용합니다. 당신의 도메인을 Stripe의 모델에 결합하는 대신, ACL을 구축합니다:

```typescript
// 당신의 깨끗한 domain model
interface Payment {
  id: string;
  customerId: string;
  amount: Money;
  status: "pending" | "completed" | "failed";
}

// Anti-Corruption Layer: Stripe의 모델을 당신의 모델로 변환
class StripeACL {
  async charge(payment: Payment): Promise<PaymentResult> {
    // 당신의 domain model → Stripe의 API 형식으로 변환
    const intent = await stripe.paymentIntents.create({
      amount: payment.amount.toCents(),        // Stripe는 정수 cents를 사용
      currency: payment.amount.currency.toLowerCase(),
      metadata: { customerId: payment.customerId },
    });

    // Stripe의 응답 → 당신의 domain model로 변환
    return {
      success: intent.status === "succeeded",
      paymentId: intent.id,
      domainStatus: this.translateStatus(intent.status),
    };
  }

  private translateStatus(stripeStatus: string): Payment["status"] {
    const map: Record<string, Payment["status"]> = {
      succeeded: "completed",
      requires_payment_method: "pending",
      canceled: "failed",
    };
    return map[stripeStatus] ?? "failed";
  }
}
```

## ACL vs 기타 통합 패턴

| 패턴 | 목적 | 변환이 일어나는 위치 |
| --- | --- | --- |
| Anti-Corruption Layer | 외부 모델로부터 당신의 도메인을 보호 | 당신의 bounded context 경계에서 |
| Adapter | 인터페이스 변환 (구조적) | 단일 클래스 / 컴포넌트 내부 |
| Facade | 복잡한 하위 시스템 단순화 | 단순화된 API를 노출하는 단일 클래스 |
| Open Host Service | 공표된 프로토콜을 통해 당신의 도메인을 타인에게 노출 | 외부 경계 (당신이 제어함) |

> ℹ️
> Microservices에서의 ACL
> microservices에서 모든 서비스 간 통합은 잠재적인 ACL 경계입니다. 서비스 A가 서비스 B를 호출할 때, B의 데이터 구조를 자신의 도메인으로 그대로 통과시키기보다 B의 응답을 자신의 도메인 언어로 변환해야 합니다. 이를 통해 bounded contexts를 진정으로 격리된 상태로 유지할 수 있습니다.

## ACL을 사용해야 할 때

- 데이터 모델이 불량하거나 일관성이 없는 레거시 시스템과 통합할 때.
- 제3자 API(payment gateways, 배송업체, CRM 시스템 등)를 사용할 때.
- 인수 합병 후 통합 과정에서 인수된 회사의 시스템과 통합할 때.
- 업스트림 팀의 모델이 당신의 모델과 갈라지는 DDD 시스템에서 bounded context 경계를 넘을 때.

> 💡
> 면접 팁 (Interview Tip)
> 면접관이 직접적으로 ACL에 대해 묻는 경우는 드물지만, '레거시 결제 시스템과 어떻게 통합하시겠습니까?' 또는 'microservices 간의 결합을 어떻게 피합니까?'와 같은 질문에 암시적으로 나타납니다. 업스트림 모델이 도메인으로 유출되는 것을 방지하기 위해 변환 계층(ACL)을 도입하겠다고 언급하는 것은 DDD 지식과 아키텍처적 성숙도를 보여줍니다. 적절하다면 Eric Evans의 이름을 언급하십시오.

> ⚠️
> 작은 통합을 과도하게 설계하지 마십시오
> 안정적인 contract를 가진 잘 설계된 단일 REST API와 통합하는 경우, 간단한 adapter 클래스로 충분할 수 있습니다. 외부 모델이 당신의 모델과 진정으로 다르거나, 자주 변경되거나, 동일한 인터페이스 뒤에서 여러 제공업체와 통합해야 할 때 비로소 전체 ACL이 정당화됩니다.
