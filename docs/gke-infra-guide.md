# 🐳 Docker & GKE 인프라 구축 및 클러스터 사용 가이드

팀 크레딧을 효율적으로 관리하고 팀원 간 배포 충돌을 방지하기 위해, **단일 GKE 클러스터 내에서 네임스페이스(Namespace)를 분리**하여 운영합니다.

---

## 1. 인프라 전체 구조

- **Container Registry:** Docker Hub (Public Repository, `latest` 태그 기준)
- **Kubernetes Cluster:** Google Kubernetes Engine (GKE)
  - **프로젝트 ID:** `iron-area-514720-j0`
  - **클러스터 이름:** `sungshin-find-cluster`
  - **리전 / 존:** `asia-northeast3-a` (서울)
- **네임스페이스(Namespace) 분리 정책**
  - `dev`: `main` 브랜치 머지 시 중앙 Jenkins가 자동 배포하는 **팀 공용 통합 환경** (접속 IP: `34.64.211.204`)
  - `prod`: 운영 배포용 환경
  - `dev-[이름]`: 팀원별 개인 개발 및 원격 테스트용 **격리 환경**
    - `dev-eunhee`, `dev-yuri`, `dev-yerin`, `dev-haram`, `dev-naeun`, `dev-sujin`

> 💡 **사전 세팅 완료 안내**  
> 각 개인 네임스페이스(`dev-[이름]`) 내부에는 `backend-config`(ConfigMap), `postgres-secret`(Secret), `backend`/`frontend`/`postgres` Service, 그리고 `PostgreSQL` DB가 미리 배포되어 있습니다. 별도의 초기 DB 세팅 없이 바로 애플리케이션 배포 테스트가 가능합니다.

---

## 2. 로컬 터미널에서 GKE 클러스터 연결하기

로컬 PC에서 클러스터 상태를 조회하거나 직접 배포하려면 **`gcloud` CLI**와 **`kubectl`**이 설치되어 있어야 합니다.

### ① Google 계정 로그인
```bash
gcloud auth login
```

### ② GKE 클러스터 인증 정보 등록
```bash
gcloud container clusters get-credentials sungshin-find-cluster \
  --zone asia-northeast3-a \
  --project iron-area-514720-j0
```

### ③ 연결 확인
```bash
kubectl get nodes
kubectl get pods -n dev
```

---

## 3. 개인 네임스페이스(`dev-[이름]`) 상태 확인 명령어

개인 Jenkins 배포 후 본인의 파드(Pod)가 정상적으로 실행되었는지 확인할 때는 명령어 뒤에 반드시 **`-n dev-[본인이름]`**을 붙여 조회합니다. *(예: `dev-yuri`)*

```bash
# 1. 내 네임스페이스의 모든 리소스(Pod, Service, Deployment 등) 조회
kubectl get all -n dev-[본인이름]

# 2. 파드 실시간 상태 모니터링 (종료: Ctrl + C)
kubectl get pods -n dev-[본인이름] -w

# 3. 백엔드/프론트엔드 에러 로그 확인
kubectl logs -l app=backend -n dev-[본인이름] --tail=100
kubectl logs -l app=frontend -n dev-[본인이름] --tail=100
```

> ⚠️ **주의사항:** `-n` 옵션을 생략하거나 `-n dev`로 명령어를 실행하면 공용 통합 환경이 조회/수정되므로, 개인 테스트 시 반드시 본인의 네임스페이스를 지정해 주세요.

---

## 4. GKE 노드 운영 및 트러블슈팅

### 🕒 클러스터 노드 운영 방식 (크레딧 절약)
- GCP 크레딧 절약을 위해 미사용 시간대에는 GKE 워커 노드를 `0`대로 축소(중지)해 둡니다.
- **가동 기준:** 작업 및 배포가 필요한 시간대에 유동적으로 가동하며, 노드가 꺼져 있을 때 배포가 필요하면 **팀 톡방에 노드 활성화를 요청**해 주세요.

### ❓ 배포 시 `Pending` 상태로 멈추고 타임아웃이 날 때
- **원인:** GKE 워커 노드가 `0`대로 꺼져 있는 상태에서 배포를 실행하면 새 파드가 할당될 서버가 없어 계속 `Pending` 상태에 머무르게 됩니다.
- **해결:** `kubectl get nodes`로 노드 가동 여부를 먼저 확인하고, 노드가 없다면 톡방에서 노드 활성화를 요청한 뒤 재배포해 주세요.