# JSP(Tomcat) 웹 애플리케이션 쿠버네티스 배포 종합 계획서

> 대상: Tomcat 기반 JSP 애플리케이션(`target/app.war`)
> 환경: 사내 교육/운영 쿠버네티스 클러스터(`*.std.kopoctc.kr`)
> 최종 목표: GitLab에서 소스 관리 → 이미지 빌드 → Harbor 저장 → Helm으로 쿠버네티스 배포 → HTTPS 외부 공개 + 정기 백업

---

## 목차

1. [전체 아키텍처 흐름도](#1-전체-아키텍처-흐름도)
2. [폴더 / 파일 구조](#2-폴더--파일-구조)
3. [사전 준비 사항](#3-사전-준비-사항)
4. [단계별 작업 순서](#4-단계별-작업-순서)
5. [쿠버네티스 리소스 목록](#5-쿠버네티스-리소스-목록)
6. [보안 고려사항(Secret 관리)](#6-보안-고려사항secret-관리)
7. [백업 전략](#7-백업-전략)
8. [배포 및 유지보수 명령어 예시](#8-배포-및-유지보수-명령어-예시)
9. [점검 체크리스트 / 문제 해결](#9-점검-체크리스트--문제-해결)

---

## 1. 전체 아키텍처 흐름도

```
 ┌──────────────┐      git push       ┌──────────────────────────────────────┐
 │  개발자 PC   │ ─────────────────▶  │            GitLab (SCM)              │
 │  (JSP/Maven) │  ◀───────────────── │  - 소스코드, Dockerfile, Helm 차트    │
 └──────────────┘   clone/CI 트리거   │  - .gitlab-ci.yml (CI/CD 파이프라인) │
                                     └───────────────┬──────────────────────┘
                                                     │ 1) mvn package (WAR 빌드)
                                                     │ 2) docker build (Tomcat 이미지)
                                                     ▼
                                     ┌──────────────────────────────┐
                                     │   Harbor (Private Registry)  │
                                     │   harbor.kopoctc.kr/         │
                                     │     <studentId>/<app>:<tag>  │
                                     └───────────────┬──────────────┘
                                                     │ docker push (CI 자동화)
                                                     │  ▲
                                                     │  │ docker pull (배포 시)
                                                     ▼  │
 ┌───────────────────────────────────────────────────────────────────────────┐
 │                        쿠버네티스 클러스터 (std)                            │
 │                                                                           │
 │   ┌─────────────┐    ┌─────────────────────────────────────────────────┐  │
 │   │  Ingress    │───▶│  Deployment (Pod: Tomcat)                        │  │
 │   │ (nginx/TLS) │    │  - image: harbor.../<studentId>/<app>:tag         │  │
 │   │ Host:       │    │  - imagePullSecrets: [harbor-creds] (Secret)     │  │
 │   │ <id>-<app>  │    │  - envFrom: db-secret (Secret) / app-config(CM)  │  │
 │   │ .std.kopctc │    │  - volume: data-pvc (PVC, nfs-std-1)             │  │
 │   └─────────────┘    └────────────────┬────────────────────────────────┘  │
 │                                       │ Service(ClusterIP:8080)           │
 │                                       ▼                                   │
 │                        (필요 시) MariaDB/MySQL Pod  ◀── db-secret          │
 │                                                                           │
 │   ┌─────────────────────────────┐        ┌────────────────────────────┐   │
 │   │ CronJob (정기 백업)          │ ────▶  │ backup-pvc (PVC, nfs-std-1) │   │
 │   │ schedule: "0 2 * * *"        │ mysqldump │  /backup/dump-YYYYmmdd.sql│   │
 │   │ historyLimit: 3/1            │        └────────────────────────────┘   │
 │   └─────────────────────────────┘                                          │
 └───────────────────────────────────────────────────────────────────────────┘
                            ▲
                            │ HTTPS (TLS: 와일드카드 인증서 *.std.kopoctc.kr)
                            │
                        외부 사용자 브라우저
```

**핵심 흐름 한 줄 요약:**
`GitLab push → CI가 WAR 빌드 → Docker 이미지화 → Harbor push → Helm install이 Harbor에서 pull → Ingress로 HTTPS 노출 → CronJob이 매일 DB/데이터를 PVC에 백업`

---

## 2. 폴더 / 파일 구조

```
project-root/                         # GitLab 저장소 루트
├── src/                              # JSP/Java 소스 (Maven 프로젝트)
├── pom.xml                           # Maven 빌드 설정 → target/app.war 산출
├── Dockerfile                        # tomcat:10.1 + app.war → 컨테이너 이미지
├── .gitlab-ci.yml                    # build → push → (선택) deploy 파이프라인
│
├── chart/                            # ★ Helm 차트 (본 계획서 위치)
│   ├── DEPLOY-PLAN.md                # ← 이 문서
│   ├── Chart.yaml                    # 차트 메타데이터 (이름/버전)
│   ├── values.yaml                   # 기본값 (studentId/appName/image 등)
│   ├── .helmignore
│   └── templates/
│       ├── _helpers.tpl              # host 자동 조립 등 헬퍼 함수
│       ├── deployment.yaml           # Tomcat Pod
│       ├── service.yaml              # ClusterIP (8080)
│       ├── ingress.yaml              # HTTPS 외부 공개
│       ├── secret.yaml               # DB 비밀번호 등 (base64)
│       ├── configmap.yaml            # 비민감 설정값
│       ├── pvc.yaml                  # data-pvc / backup-pvc
│       ├── cronjob-backup.yaml       # 정기 백업 (history limit 포함)
│       ├── serviceaccount.yaml       # (선택) 전용 SA
│       └── NOTES.txt                 # helm install 후 안내 메시지
│
└── .git/                             # GitLab 원격 추적
```

> **참고**: 기존 `myapp/` 차트(`helm create` 기본 스캐폴드)의 `_helpers.tpl`, `deployment.yaml`, `ingress.yaml` 패턴을 그대로 계승하되, `studentId`/`appName` 주입과 Harbor/Secret/백업을 추가한 형태로 `chart/`에 새로 구성한다.

---

## 3. 사전 준비 사항

| 항목 | 내용 | 비고 |
|------|------|------|
| 클러스터 접속 | `kubectl` + `~/.kube/config` 구성 완료 | `kubectl get nodes` 확인 |
| 스토리지 | `storageClassName: nfs-std-1` 사용 가능 | `kubectl get sc` 확인 |
| Ingress 컨트롤러 | nginx(또는 traefik) 설치됨 | `kubectl get ingressclass` |
| TLS 인증서 | `*.std.kopoctc.kr` 와일드카드 인증서 (Secret) | 와일드카드 1장으로 모든 학생 호스트 커버 |
| Harbor 계정 | `<studentId>` 프로젝트 + push 권한 | `harbor.kopoctc.kr` |
| GitLab 계정 | 저장소 생성 + CI Runner 등록 | 그룹/프로젝트 생성 |
| 빌드 도구 | Maven(`mvn`), Docker, Helm(v3) | 로컬 또는 CI Runner |

---

## 4. 단계별 작업 순서

### Step 1 — GitLab 저장소 준비
1. GitLab에 신규 프로젝트 생성 (예: `<studentId>-jsp-app`).
2. 로컬에서 `git init` 후 원격 연결:
   ```bash
   git init
   git remote add origin git@gitlab.kopoctc.kr:<studentId>/jsp-app.git
   ```
3. `.gitignore`에 `target/`, `*.war`, `.env`, 시크릿 파일 추가 → **빌드 산출물/비밀번호 커밋 금지**.

### Step 2 — JSP 앱 빌드 (Maven → WAR)
```bash
mvn clean package        # target/app.war 생성
```
- 컨텍스트 경로 주의: `webapps/app.war` → 접속 경로 `/app/`. 루트(`/`)로 서비스하려면 `ROOT.war`로 rename.

### Step 3 — Dockerfile / 이미지 빌드
`Dockerfile` (이미 존재하는 패턴 유지):
```dockerfile
FROM tomcat:10.1
COPY target/app.war webapps/
# (선택) 루트 컨텍스트: COPY target/app.war webapps/ROOT.war
```
```bash
docker build -t <studentId>-<app>:<tag> .
```

### Step 4 — Harbor 로그인 및 이미지 Push
```bash
# Harbor 로그인 (한 번; 자격증명은 ~/.docker/config.json 에 저장됨)
docker login harbor.kopoctc.kr -u <studentId>

# 태깅: <registry>/<project>/<image>:<tag>
docker tag <studentId>-<app>:<tag> harbor.kopoctc.kr/<studentId>/<app>:<tag>
docker push harbor.kopoctc.kr/<studentId>/<app>:<tag>
```
- **CI 자동화**: `.gitlab-ci.yml`의 `build`/`push` 잡에서 동일 작업 수행 (아래 부록).

### Step 5 — Helm 차트 작성
`chart/` 하위에 `Chart.yaml`, `values.yaml`, `templates/` 작성 (상세 템플릿은 본문 예시 참조).

### Step 6 — Secret 생성 (DB 비밀번호 등)
```bash
kubectl create secret generic db-secret \
  --from-literal=DB_USER=<dbuser> \
  --from-literal=DB_PASSWORD='<강한비밀번호>' \
  --from-literal=DB_HOST=mariadb.<namespace>.svc.cluster.local
```
> CLI 히스토리에 비밀번호가 남지 않도록 `--from-file` 또는 `kubectl apply -f secret.yaml`(base64) 사용 권장 (6절 참조).

### Step 7 — Harbor Pull용 Secret (imagePullSecret)
```bash
kubectl create secret docker-registry harbor-creds \
  --docker-server=harbor.kopoctc.kr \
  --docker-username=<studentId> \
  --docker-password='<Harbor비밀번호>' \
  --docker-email=<studentId>@kopoctc.kr
```

### Step 8 — 배포 (helm install)
```bash
helm install <app> ./chart \
  --namespace <studentId> --create-namespace \
  --set studentId=<studentId> \
  --set appName=<app> \
  --set image.repository=harbor.kopoctc.kr/<studentId>/<app> \
  --set image.tag=<tag>
```

### Step 9 — Ingress / HTTPS 확인
```bash
kubectl get ingress -n <studentId>
curl -k https://<studentId>-<app>.std.kopoctc.kr/
```
- 호스트는 차트가 자동 조립 (아래 `_helpers.tpl` 참조).

### Step 10 — CronJob 백업 배포
- 차트에 포함된 `cronjob-backup.yaml`이 `backup-pvc`에 매일 덤프 저장 (7절 참조).
- 배포 후 임의 1회 수동 실행으로 검증:
  ```bash
  kubectl -n <studentId> create job --from=cronjob/<app>-backup manual-backup-test
  ```

---

## 5. 쿠버네티스 리소스 목록

| 리소스 | 종류 | 용도 | 소스 |
|--------|------|------|------|
| `Deployment` | apps/v1 | Tomcat Pod 구동 | `deployment.yaml` |
| `Service` | v1 (ClusterIP) | Pod 서비스 디스커버리 (8080) | `service.yaml` |
| `Ingress` | networking.k8s.io/v1 | 외부 HTTPS 라우팅 (호스트 + TLS) | `ingress.yaml` |
| `Secret(db-secret)` | v1 | DB 접속정보(민감) | `secret.yaml` / CLI |
| `Secret(harbor-creds)` | v1 (docker-registry) | Harbor 이미지 pull 자격 | CLI |
| `Secret(tls)` | v1 (tls) | `*.std.kopoctc.kr` 인증서 | 클러스터 공통 |
| `ConfigMap` | v1 | 비민감 설정(GREETING/MODE 등) | `configmap.yaml` |
| `PersistentVolumeClaim` | v1 | `data-pvc`(2Gi) / `backup-pvc`(3Gi) | `pvc.yaml` |
| `CronJob` | batch/v1 | 정기 백업 + history limit | `cronjob-backup.yaml` |
| `ServiceAccount` | v1 | (선택) 전용 SA / RBAC | `serviceaccount.yaml` |

---

## 6. 보안 고려사항 (Secret 관리)

> **원칙**: 비밀번호는 평문 YAML에 두지 않고, Secret(base64) + 출처 관리. `base64`는 암호화가 아님(인코딩) → RBAC와 외부 보관이 핵심.

### 6.1 무엇을 Secret에 넣는가
- DB 비밀번호, DB 사용자, DB 호스트 (또는 JDBC URL)
- Harbor 토큰/비밀번호 (imagePullSecret)
- API 키, 외부 서비스 자격증명
- TLS 개인키 (`kubernetes.io/tls` 타입)

### 6.2 Secret 생성 방법 (안전한 순서)
1. **kubectl create(권장)** — CLI 히스토리 노출 주의, `--from-file`로 파일에서 읽기:
   ```bash
   kubectl create secret generic db-secret \
     --from-env-file=./db.env     # db.env는 .gitignore, 배포 후 삭제
   ```
2. **매니페스트(base64)** — `secret.yaml`은 Git에 넣되 **값은 비워두고** CI/Sealed-Secrets로 주입:
   ```yaml
   # secret.yaml (템플릿 - 값은 별도 주입)
   apiVersion: v1
   kind: Secret
   metadata:
     name: db-secret
   type: Opaque
   stringData:          # stringData는 base64 인코딩 불필요 (kubectl이 인코딩)
     DB_USER: "<studentId>_user"
     DB_PASSWORD: "CHANGE_ME"
     DB_HOST: "mariadb.<namespace>.svc.cluster.local"
   ```

### 6.3 권장 추가 조치
- **RBAC**: 앱 전용 ServiceAccount에 최소 권한 부여, Secret 읽기는 해당 SA만 허용.
- **Sealed-Secrets / External Secrets / SOPS**: Git에 암호화된 Secret을 두고 클러스터에서 복호화 (GitOps 운영 시).
- **Pod 단위**: `envFrom: secretRef`로 주입, 볼륨 마운트 시 `readOnly: true`.
- **감사**: `kubectl get secret` 권한 자체를 제한, `kubectl auth can-i get secrets`로 검증.
- **이미지 서명**: Harbor의 Cosign 서명/취약점 스캔(Robots) 활성화.

---

## 7. 백업 전략

### 7.1 대상
- **DB 데이터**: `mysqldump` / `mariadb-dump` → SQL 파일.
- **앱 업로드 데이터**: `data-pvc`에 저장된 사용자 업로드 파일(있으면) → tar 복사.

### 7.2 보관 정책 (2계층)
- **K8s 잡 히스토리(history limit)**: 실패한 잡 원인 추적용. 클러스터 내 잡 개수 제한.
- **PVC 내 파일 로테이션**: 실제 백업 파일은 보존일수(N일) 초과 시 자동 삭제.

### 7.3 CronJob 템플릿 (`templates/cronjob-backup.yaml`)
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: {{ .Values.appName }}-backup
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  schedule: {{ .Values.backup.schedule | quote }}        # 기본 "0 18 * * *" (매일 18:00 KST)
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: {{ .Values.backup.successfulJobsHistoryLimit }}   # 3
  failedJobsHistoryLimit: {{ .Values.backup.failedJobsHistoryLimit }}           # 1
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: mariadb:10.11
              envFrom:
                - secretRef: { name: db-secret }          # DB_USER/PASSWORD/HOST
              command:
                - /bin/sh
                - -c
                - |
                  set -e
                  TS=$(date +%Y%m%d-%H%M%S)
                  OUT=/backup/dump-${TS}.sql.gz
                  echo "[backup] start -> ${OUT}"
                  mysqldump -h"$DB_HOST" -u"$DB_USER" -p"$DB_PASSWORD" \
                    --all-databases --single-transaction --routines --triggers | gzip > "$OUT"
                  # PVC 내 14일 초과 파일 자동 삭제(용량 관리)
                  find /backup -name 'dump-*.sql.gz' -mtime +14 -delete
                  echo "[backup] done. files:"; ls -lh /backup
              volumeMounts:
                - name: backup-vol
                  mountPath: /backup
          volumes:
            - name: backup-vol
              persistentVolumeClaim:
                claimName: backup-pvc
```

### 7.4 복원 절차
```bash
# 최신 덤프 파일 확인
kubectl -n <studentId> exec deploy/<app>-backup-... -- ls -lh /backup

# 임시 Pod로 PVC 마운트 후 복원
kubectl -n <studentId> run restore --rm -i --tty --restart=Never \
  --image=mariadb:10.11 --overrides='{
    "spec":{"volumes":[{"name":"b","persistentVolumeClaim":{"claimName":"backup-pvc"}}],
    "containers":[{"name":"restore","image":"mariadb:10.11",
      "stdin":true,"tty":true,
      "volumeMounts":[{"name":"b","mountPath":"/backup"}]}]}}' \
  -- bash -c "zcat /backup/dump-<TS>.sql.gz | mariadb -h<DB_HOST> -u<DB_USER> -p<DB_PASSWORD>"
```

---

## 8. 배포 및 유지보수 명령어 예시

> 예시 변수: `studentId=hong`, `appName=diary`, `tag=1.0.0`, `ns=hong`

### 8.1 사전 검증
```bash
# 차트 문법/렌더링 검사 (실제 배포 없이)
helm lint ./chart
helm template diary ./chart \
  --set studentId=hong --set appName=diary \
  --set image.repository=harbor.kopoctc.kr/hong/diary \
  --set image.tag=1.0.0 | less
```

### 8.2 배포
```bash
# 신규 배포
helm install diary ./chart -n hong --create-namespace \
  --set studentId=hong \
  --set appName=diary \
  --set image.repository=harbor.kopoctc.kr/hong/diary \
  --set image.tag=1.0.0

# 업데이트 (이미지 태그/설정 변경 시)
helm upgrade diary ./chart -n hong \
  --set studentId=hong --set appName=diary \
  --set image.repository=harbor.kopoctc.kr/hong/diary \
  --set image.tag=1.0.1
```

### 8.3 값 주입을 파일로 관리 (반복 입력 회피)
```bash
# values-prod.yaml 에 studentId/appName/image 등 기록 후
helm upgrade --install diary ./chart -n hong -f values-prod.yaml
```

### 8.4 이미지 갱신(새 빌드) → 롤아웃
```bash
docker build -t harbor.kopoctc.kr/hong/diary:1.0.2 .
docker push harbor.kopoctc.kr/hong/diary:1.0.2
helm upgrade diary ./chart -n hong --set image.tag=1.0.2 \
  --reuse-values
kubectl -n hong rollout status deploy/diary
```

### 8.5 상태 조회
```bash
helm list -n hong
helm status diary -n hong
kubectl -n hong get all,ingress,pvc,secret,configmap
kubectl -n hong get pods -w
kubectl -n hong logs -f deploy/diary
kubectl -n hong describe ingress diary
```

### 8.6 백업 수동 실행 / 확인
```bash
kubectl -n hong create job --from=cronjob/diary-backup manual-backup-$(date +%s)
kubectl -n hong get jobs -w
kubectl -n hong logs job/manual-backup-<TS>
```

### 8.7 롤백
```bash
helm history diary -n hong
helm rollback diary 1 -n hong          # 이전 리비전으로 복귀
kubectl -n hong rollout undo deploy/diary
```

### 8.8 삭제
```bash
helm uninstall diary -n hong
# PVC는 보존(데이터 손실 방지) → 명시적 삭제 시에만
kubectl -n hong delete pvc data-pvc backup-pvc
```

### 8.9 부록: `.gitlab-ci.yml` (CI 자동화)
```yaml
stages: [build, push]
variables:
  HARBOR: harbor.kopoctc.kr
  IMG: $HARBOR/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME

build-war:
  stage: build
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn -B clean package
  artifacts:
    paths: [target/app.war]

build-push-image:
  stage: push
  image: docker:24
  services: [{ name: docker:24-dind }]
  script:
    - echo "$HARBOR_TOKEN" | docker login $HARBOR -u "$HARBOR_USER" --password-stdin
    - docker build -t $IMG:$CI_COMMIT_SHORT_SHA .
    - docker tag  $IMG:$CI_COMMIT_SHORT_SHA $IMG:latest
    - docker push $IMG:$CI_COMMIT_SHORT_SHA
    - docker push $IMG:latest
```

---

## 9. 점검 체크리스트 / 문제 해결

### 9.1 배포 전 체크리스트
- [ ] `target/app.war` 정상 생성 (`mvn package`)
- [ ] Harbor에 로그인 성공, 프로젝트에 push 권한
- [ ] `db-secret`, `harbor-creds`, TLS Secret 존재 확인
- [ ] `nfs-std-1` StorageClass로 PVC Pending 없이 Bound
- [ ] `helm lint ./chart` 통과
- [ ] Ingress 호스트 = `<studentId>-<app>.std.kopoctc.kr` 일치 (DNS/와일드카드)

### 9.2 자주 발생하는 장애와 해결
| 증상 | 원인 | 해결 |
|------|------|------|
| Pod `ImagePullBackOff` | `harbor-creds` 미설정 / 태그 오타 | `imagePullSecrets` 확인, 태그 재확인 |
| `CrashLoopBackOff` | DB 연결 실패 / WAR 컨텍스트 경로 | `logs` 확인, `db-secret` 값/호스트 점검 |
| 503/502 Ingress | Service/Pod 미준비 | `kubectl get endpoints`, readinessProbe 경로 점검 |
| TLS 브라우저 경고 | 인증서 Secret 누락/호스트 불일치 | Ingress `tls.hosts`와 호스트 일치, 인증서 유효기간 |
| PVC `Pending` | StorageClass 오타/용량 | `storageClassName: nfs-std-1`, 가용 용량 확인 |
| 백업 실패 | DB 자격증호/네트워크 | CronJob Pod `logs`, `db-secret` 재확인 |
| CronJob 잔류 | history limit 미지정 | `successful/failed JobsHistoryLimit` 설정 |

### 9.3 운영 권장
- 이미지 태그는 `latest` 외에 **커밋 SHA/버전** 고정 (롤백 용이).
- `helm` 값은 `values-<env>.yaml`로 환경 분리.
- 백업 파일 주기적 외부 스토리지(NAS/S3)로 2차 복사 권장.
- 리소스 requests/limits 설정, HPA로 트래픽 대응.
- 정기적으로 `kubectl auth can-i` 로 권한/접근 감사.

---

## 부록 A: 핵심 차트 파일 예시

### A.1 `chart/Chart.yaml`
```yaml
apiVersion: v2
name: jsp-app
description: JSP(Tomcat) 웹앱 배포 차트 (studentId/appName 주입형)
type: application
version: 0.1.0
appVersion: "1.0.0"
```

### A.2 `chart/values.yaml`
```yaml
# === 주입형 식별자 (--set 필수) ===
studentId: ""
appName: ""

domain:
  base: std.kopoctc.kr

# 호스트 자동 조립: <studentId>-<appName>.<domain.base>
#   예) hong-diary.std.kopoctc.kr

image:
  repository: ""          # harbor.kopoctc.kr/<studentId>/<appName>
  pullPolicy: IfNotPresent
  tag: ""

imagePullSecrets:
  - name: harbor-creds

replicaCount: 1

service:
  type: ClusterIP
  port: 8080              # Tomcat 기본 포트

ingress:
  enabled: true
  className: nginx
  tls:
    - secretName: std-kopoctc-kr-tls     # 와일드카드 인증서 Secret
      hosts:
        - <studentId>-<app>.std.kopoctc.kr   # _helpers.tpl 로 자동 치환

resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }

backup:
  schedule: "0 18 * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  retentionDays: 14

persistence:
  data:
    enabled: true
    size: 2Gi
    storageClass: nfs-std-1
  backup:
    enabled: true
    size: 3Gi
    storageClass: nfs-std-1
```

### A.3 `chart/templates/_helpers.tpl` (host 자동 조립 핵심)
```go
{{/* 차트 이름 */}}
{{- define "app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{/* 풀네임 (릴리스명 우선) */}}
{{- define "app.fullname" -}}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- if contains $name .Release.Name -}}
{{- .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}

{{/* ★ host 자동 조립: <studentId>-<appName>.<domain.base> */}}
{{- define "app.host" -}}
{{- required "studentId is required" .Values.studentId -}}-{{- required "appName is required" .Values.appName -}}.{{- .Values.domain.base -}}
{{- end -}}

{{/* 전체 이미지명 */}}
{{- define "app.image" -}}
{{- printf "%s:%s" .Values.image.repository (.Values.image.tag | default .Chart.AppVersion) -}}
{{- end -}}

{{/* 공통 라벨 */}}
{{- define "app.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version }}
{{ include "app.selectorLabels" . }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
app.kubernetes.io/part-of: {{ .Values.studentId }}-{{ .Values.appName }}
{{- end -}}

{{- define "app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
```

> `required` 함수 사용으로 `--set studentId/appName` 누락 시 배포가 즉시 실패 → 잘못된 호스트/이미지 사전 차단.

### A.4 `chart/templates/ingress.yaml` (HTTPS, 호스트 자동 치환)
```yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "app.fullname" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: {{ .Values.ingress.className }}
  tls:
    - hosts:
        - {{ include "app.host" . | quote }}
      secretName: {{ .Values.ingress.tls[0].secretName }}
  rules:
    - host: {{ include "app.host" . | quote }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "app.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end -}}
```

### A.5 `chart/templates/deployment.yaml` (Secret + imagePullSecret + PVC)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
  labels: {{- include "app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels: {{- include "app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels: {{- include "app.labels" . | nindent 8 }}
    spec:
      imagePullSecrets:
        {{- toYaml .Values.imagePullSecrets | nindent 8 }}
      containers:
        - name: {{ include "app.name" . }}
          image: {{ include "app.image" . }}
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
          envFrom:
            - secretRef: { name: db-secret }        # DB 비밀번호 주입
            - configMapRef: { name: app-config }    # 비민감 설정
          readinessProbe:
            httpGet: { path: /, port: http }
            initialDelaySeconds: 15
          livenessProbe:
            httpGet: { path: /, port: http }
            initialDelaySeconds: 30
          resources: {{- toYaml .Values.resources | nindent 12 }}
          volumeMounts:
            - name: data
              mountPath: /usr/local/tomcat/uploads   # 업로드 데이터 보존 경로
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: data-pvc
```

---

**문서 버전**: v1.0 · 작성일: 2026-06-17
**대상 클러스터**: `std` (도메인 `*.std.kopoctc.kr`, StorageClass `nfs-std-1`)
