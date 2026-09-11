# k8s-gitops-lab

쿠버네티스 클러스터에 애플리케이션을 GitOps 방식으로 배포하며 학습한 기록입니다.
이 저장소의 매니페스트를 ArgoCD가 읽어 클러스터에 반영하는 구조로 구성했습니다.

## 구성

    apps/
      jenkins/     Jenkins Helm 차트 (직접 작성)
      ubuntu/      StatefulSet + Headless Service + PVC
      web/         Deployment + Service
    cluster/
      argocd-panic-prevention-limitranges.yaml

## 작업 내용

### 1. Jenkins Helm 차트 직접 작성

공개 차트를 그대로 쓰지 않고 Chart.yaml, values.yaml, templates를 직접 구성했습니다.
차트의 각 파일이 어떤 역할을 하는지 직접 확인하기 위해서였습니다.

### 2. StatefulSet과 영속 볼륨

Headless Service(clusterIP: None)와 volumeClaimTemplates를 사용해
Pod가 재시작되어도 데이터가 유지되도록 구성했습니다.

### 3. Kyverno 정책 대응

클러스터에 외부 이미지 레지스트리를 제한하는 Kyverno 정책(restrict-image-registry)이
적용되어 있어, Docker Hub 이미지로는 Pod가 생성되지 않았습니다.
허용된 사내 Harbor 레지스트리 이미지로 교체해 해결했습니다.

### 4. ArgoCD 패닉 재발 방지

resources.requests가 선언되지 않은 Pod가 있을 때
ArgoCD가 nil map 패닉으로 재시작되는 문제가 있었습니다.
네임스페이스에 LimitRange를 적용해 defaultRequest가 자동으로 채워지도록 하여
같은 문제가 다시 발생하지 않게 했습니다.

### 5. 제한된 자원 환경에서의 운영

클러스터 자원이 제한된 환경이라, CI/CD 실습에 필요한 자원을 확보하기 위해
사용하지 않는 워크로드를 스케일 다운했습니다.

## 환경

- Kubernetes
- ArgoCD
- Helm
- Kyverno (이미지 레지스트리 정책)
- Harbor (프라이빗 레지스트리)

## 참고

학습 과정에서 작성한 저장소입니다.
harbor.kopoctc.kr 는 교육용 사내 레지스트리 주소로 외부에서는 접근할 수 없습니다.