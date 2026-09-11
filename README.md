# k8s-gitops-lab

쿠버네티스 기반 GitOps 실습 저장소입니다.
한국폴리텍대학 융합기술교육원 DevOps 과정에서 직접 작성하고 클러스터에 배포한 매니페스트를 정리했습니다.

## 폴더 구성

| 폴더 | 내용 |
|---|---|
| `argocd/` | ArgoCD로 배포한 애플리케이션 매니페스트 |
| `helm/` | Helm 차트 (환경별 values 분리) |
| `kustomize/` | Kustomize base / overlays 구조 |
| `docs/` | 배포 계획 및 실습 정리 |

## argocd/

- **`apps/jenkins/`** — Jenkins를 Helm 차트로 패키징 (Deployment, Service, PVC)
- **`apps/web/`** — 웹 애플리케이션 Deployment + Service
- **`apps/ubuntu/`** — StatefulSet + PVC로 데이터 보존이 필요한 워크로드 구성
- **`cluster/`** — LimitRange로 네임스페이스 리소스 상한 지정
- **`applicationset/`** — ApplicationSet으로 여러 앱을 한 번에 배포
- **`nginx/`** — 기본 Deployment

## helm/

`web` 차트. `values.yaml`(기본)과 `values-prod.yaml`(운영)을 분리해
같은 차트로 환경별 설정을 다르게 배포합니다.

## kustomize/

`base/`에 공통 매니페스트를 두고, `overlays/dev`와 `overlays/prod`에서 환경별 차이만 덮어씁니다.
운영에는 replicas 조정과 Ingress가 추가됩니다.

## 사용 환경

Kubernetes(k3s/RKE2), ArgoCD, Helm, Kustomize, Docker
