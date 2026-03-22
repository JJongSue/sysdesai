# Immutable Infrastructure

> 원문: https://www.sysdesai.com/learn/deployment-operations/immutable-infrastructure

---

## Mutable 서버의 문제점

전통적인 운영 방식에서는 서버가 **실행 중인 상태에서 변경(mutated in place)**됩니다. 관리자가 SSH로 접속하여 패키지를 설치하고, 설정 파일을 업데이트하며, OS 패치를 수행합니다. 시간이 흐르면서 서버에는 수많은 변경 사항이 누적됩니다. 처음에는 동일하게 시작했던 두 서버가 점점 달라지게 되는데, 이러한 현상을 **설정 드리프트(configuration drift)** 또는 **스노우플레이크 서버(snowflake servers)**라고 부릅니다. 이는 많은 운영 문제의 근본 원인이 됩니다. 특정 서버에서는 잘 작동하는 배포가 다른 서버에서는 실패하거나, 운영 환경의 장애가 스테이징 환경에서 재현되지 않으며, 수동으로 변경한 사항들이 버전 관리 시스템에 기록되지 않는 등의 문제가 발생합니다.

📌
Snowflake Server 안티 패턴

'스노우플레이크 서버'란 수년에 걸쳐 수동으로 설정되어 내부 구조를 정확히 아는 사람이 아무도 없고, 감히 교체할 엄두도 내지 못하는 서버를 말합니다. 이러한 서버는 신뢰성이 낮아서가 아니라 **대체 불가능하기 때문에** 단일 장애점(single points of failure)이 됩니다. 팀은 이를 '반려동물(pets)'처럼 취급하며 이름을 붙여주고, 장애가 나면 정성껏 간호하여 복구시키려 애씁니다.

## Immutable Infrastructure: 핵심 원칙

**Immutable infrastructure**(불변 인프라)의 핵심은 일단 서버(또는 컨테이너)가 배포되면 **절대 수정하지 않는다**는 것입니다. 새로운 버전을 배포하거나 설정을 변경하려면 새로운 이미지를 빌드하여 기존 것과 함께 또는 기존 것을 대신하여 배포합니다. 그리고 이전 인스턴스는 즉시 종료(terminate)합니다. 실행 중인 서버에 패치를 적용하거나, 수정을 위해 SSH로 접속하거나, `apt-get upgrade`를 실행하는 일은 없습니다.

이미 많은 팀이 **Docker 컨테이너**를 사용하여 이러한 방식으로 작업하고 있습니다. 새 컨테이너 이미지를 빌드하고, 푸시하고, 배포한 뒤 이전 컨테이너를 삭제합니다. Immutable infrastructure는 이러한 규율을 VM 이미지(AWS의 AMI), Kubernetes 노드 및 모든 지원 인프라를 포함한 전체 인프라 계층으로 확장하는 것입니다.

Immutable infrastructure 파이프라인: 모든 변경사항은 새로운 이미지를 생성하며, 이전 인스턴스는 수정되지 않고 교체됩니다.

## Golden Images

**Golden image**(또는 golden AMI)는 OS, 런타임 의존성, 애플리케이션 코드 및 기본 설정 등 서버 부팅 즉시 트래픽을 처리할 수 있는 모든 요소를 포함하여 미리 구워진(pre-baked) 머신 이미지를 말합니다. Golden image는 전통적인 배포 방식에서의 **부트스트랩 시간(bootstrap time)**을 제거합니다(부팅 후 Ansible 플레이북이나 Chef 레시피를 실행할 필요가 없음).

- **Layer 1 — 베이스 OS 이미지:** 보안 강화된 OS (Amazon Linux 2023, Ubuntu 22.04), 빌드 시점에 보안 패치 적용
- **Layer 2 — 런타임 이미지:** 언어 런타임 (JDK 21, Node 22), 시스템 의존성 추가
- **Layer 3 — 애플리케이션 이미지:** 애플리케이션 JAR/바이너리, 시작 스크립트 추가
- **Layer 4 — 설정:** 이미지에 직접 포함하거나(드문 경우) 부팅 시 외부 설정 저장소(external config store)에서 가져옴

**HashiCorp Packer**는 golden AMI를 빌드하기 위한 표준 도구입니다. Packer는 EC2 인스턴스를 시작하고, 프로비저너(provisioner, 셸 스크립트나 Ansible 등)를 실행한 뒤 그 결과물로 AMI를 생성합니다. 생성된 AMI는 버전 메타데이터와 함께 AWS에 저장되며 Auto Scaling 런치 템플릿에서 사용됩니다.

json

```
// Golden AMI를 위한 Packer 템플릿 예시
{
  "variables": {
    "app_version": "{{env `APP_VERSION`}}"
  },
  "builders": [{
    "type": "amazon-ebs",
    "region": "us-east-1",
    "source_ami_filter": {
      "filters": { "name": "amzn2-ami-hvm-*-x86_64-gp2" },
      "owners": ["amazon"],
      "most_recent": true
    },
    "instance_type": "t3.medium",
    "ami_name": "my-app-{{user `app_version`}}-{{timestamp}}"
  }],
  "provisioners": [
    {
      "type": "shell",
      "scripts": ["scripts/install-deps.sh", "scripts/install-app.sh"]
    }
  ]
}
```

## 컨테이너와 Immutable Infrastructure

컨테이너는 immutable infrastructure의 **가장 자연스러운 표현 방식**입니다. Docker 이미지는 정의상 불변입니다. 한 번 빌드되고 푸시되면 변경할 수 없습니다(이미지 레이어는 SHA256 다이제스트에 의해 콘텐츠 주소 지정 방식으로 관리됨). Kubernetes는 이 모델을 강제합니다. Deployment spec에서 이미지 태그(image tag)를 변경하면 Kubernetes가 새 이미지로부터 새로운 pod들을 생성하여 배포합니다.

진정한 불변 컨테이너 인프라를 위한 주요 실천 사항:

- **운영 환경에서 `latest` 태그를 절대 사용하지 마세요.** 배포 재현성을 위해 항상 특정 다이제스트나 버전 태그를 고정(pin)하여 사용하세요.
- **가능한 경우 컨테이너를 읽기 전용으로 만드세요.** Kubernetes 보안 컨텍스트에서 `readOnlyRootFilesystem: true`를 설정하여 런타임 수정을 방지하세요.
- **모든 가변 상태를 외부화하세요.** 로그는 stdout/stderr로 보냅니다(사이드카(sidecar)나 노드 에이전트가 수집). 설정은 ConfigMaps나 외부 설정 저장소에서 가져옵니다. 데이터는 persistent volumes나 외부 데이터베이스에 저장합니다.
- **CI 파이프라인에서 이미지를 스캔하세요.** Trivy, Snyk, 또는 AWS ECR 이미지 스캐닝을 사용하여 런타임이 아닌 빌드 시점에 취약점을 잡아내세요.

## Infrastructure as Code (IaC): 실현 도구

Immutable infrastructure를 실제로 구현하려면 **Infrastructure as Code (IaC)**가 필수적입니다. 코드로 인프라를 재현할 수 없다면 이전 인스턴스를 안전하게 삭제할 수 없기 때문입니다. Terraform, AWS CloudFormation, Pulumi는 인프라를 선언적으로 정의합니다. 이를 불변 이미지와 결합하면 VPC 서브넷부터 애플리케이션 서버까지 전체 시스템을 소스 제어 도구로부터 다시 구축할 수 있습니다.

| Mutable Infrastructure | Immutable Infrastructure |
| --- | --- |
| 문제를 해결하기 위해 SSH 접속 | 새 이미지 빌드 후 배포, 이전 것 삭제 |
| 시간이 흐름에 따라 설정 드리프트 발생 | 모든 인스턴스가 동일 이미지 기반 — 드리프트 없음 |
| '내 서버(운영)에선 잘 되는데' 상황 발생 | 재현 가능 — 어디서나 동일한 이미지 사용 |
| 패치가 위험하고 임시방편적임 | 패치 = 이미지 재빌드 + 롤링 배포 |
| 롤백이 복잡함 | 롤백 = 이전 이미지 버전 배포 |

> 💡
> 인터뷰 팁
> Immutable infrastructure에 대해 질문받으면 이를 더 넓은 운영상의 이점과 연결하여 답변하세요: MTTR 감소(수리 대신 교체), 환경 재현성(스테이징과 운영 환경의 일치), 그리고 보안(패치를 수동 작업이 아닌 CI/CD의 핵심 프로세스로 처리). 또한 상태가 있는(stateful) 데이터는 반드시 외부화해야 한다는 제약 사항도 언급하세요. 애플리케이션 상태를 불변 이미지에 구워 넣을 수는 없기 때문입니다. 이는 여러분이 전체적인 그림을 이해하고 있음을 보여줍니다.
