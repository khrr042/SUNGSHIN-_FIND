# CI/CD 개인 구축 가이드

> 교수님 권장사항에 따라 **모든 팀원이 각자 CI/CD 파이프라인을 직접 구축**해보기 위한 가이드입니다.
> 강의자료 8주차(CI 환경 구축), 9주차(Kubernetes), 10주차(클라우드 배포) 방식을 기준으로 작성했습니다.
>
> - 작성: 고은희 (CI/CD 담당)
> - 기한: 11/28 (금)까지 개인 구축 완료
>
> 📌 **문서 역할 분담**
> - 이 문서: **Jenkins CI/CD 파이프라인 + 개인 Jenkins 실습** (Jenkins 설치, Credential, Jenkinsfile, Multibranch)
> - 수진님 **인프라 가이드** (`docs/`): 인프라 전체 구조, GKE 클러스터 접속 방법, 개인 네임스페이스(`dev-이름`) 사용·확인법, 노드 가동 시간 & 크레딧 관리
>
> 👉 GKE 접속 / 네임스페이스 / 노드 관련 내용은 **인프라 가이드를 먼저** 보고 오세요.

---

## 0. 전체 구조

### 중앙 레포 (이미 구축 완료)
```
GitHub (sungshin-find/SUNGSHIN-_FIND)
   │  PR / main 머지 → Webhook
   ▼
Jenkins (GCE VM jenkins-server)
   │  Build → Test → Push → Deploy
   ├──▶ Docker Hub (slwnt31/sungshin-find-*)
   └──▶ GKE sungshin-find-cluster / namespace: dev
```

### 개인 실습 (이 가이드로 만들 것)
```
GitHub (내계정/SUNGSHIN-_FIND  ← Fork)
   │  1분마다 Jenkins 가 GitHub 확인 (polling)
   ▼
Jenkins (내 컴퓨터 Docker Desktop)
   │  Build → Test → Push → Deploy
   ├──▶ Docker Hub (내아이디/sungshin-find-*)
   └──▶ GKE sungshin-find-cluster / namespace: dev-내이름
```

| 항목 | 중앙 레포 | 개인 실습 |
|---|---|---|
| 레포 | `sungshin-find/SUNGSHIN-_FIND` | 내 계정으로 Fork 한 레포 |
| Jenkins 위치 | GCP VM (고정 IP) | **내 컴퓨터** (`localhost:8080`) |
| GitHub → Jenkins 알림 | Webhook | **1분마다 polling** (GitHub 이 내 컴퓨터에 접속할 수 없어서) |
| Docker Hub | `slwnt31` | **내 Docker Hub 아이디** |
| 배포 네임스페이스 | `dev` (팀 공용 메인 환경) | **`dev-내이름`** (예: `dev-eunhee`, 개인 연습 공간) |
| 클러스터 | `sungshin-find-cluster` (공용) | 같은 클러스터 공유 |

> ⚠️ 개인 Jenkins 에서 **`dev` 로 배포하면 팀 공용 환경을 덮어씁니다.** Jenkinsfile 의 `NAMESPACE` 를 반드시 `dev-내이름` 으로 바꾸세요. ([8-1](#8-1-수정할-곳은-딱-2군데))
>
> `dev` / `dev-이름` 구조와 노드 가동 시간은 **인프라 가이드** 참고

---

## 1. 준비물

| 준비물 | 설치 / 확인 | 비고 |
|---|---|---|
| Git | `git --version` | |
| Docker Desktop | [docker.com](https://www.docker.com/products/docker-desktop/) 에서 설치 후 실행 | Jenkins 를 내 컴퓨터에서 띄우는 데 사용 |
| Google Cloud CLI | "gcloud cli install" 검색 → 공식 설치 프로그램 | 9주차 "Google Cloud SDK 설치" |
| kubectl | `gcloud components install kubectl gke-gcloud-auth-plugin` | |
| Docker Hub 계정 | [hub.docker.com](https://hub.docker.com) 가입 | 아이디는 영어 소문자 권장 |
| 클러스터 접속 + 내 네임스페이스 | **인프라 가이드** 참고 | `dev-이름` 은 수진님이 만들어주심 |
| **서비스 계정 JSON 키** | 수진님께 받기 | Jenkins 가 GKE 에 배포할 때 사용. **절대 커밋 금지** |

### 클러스터 접속 확인
내 컴퓨터에서 `kubectl get pods -n dev-내이름` 이 되는지 확인하세요. (설정 방법은 **인프라 가이드의 GKE 클러스터 접속 방법** 참고)

> ⚠️ **Windows Git Bash 에서 `gcloud --version` 이 "Python" 만 출력하고 끝나는 경우** → [10. 문제해결](#10-문제해결-모음) 1번 참고

---

## 2. 레포 Fork 하기

1. https://github.com/sungshin-find/SUNGSHIN-_FIND 접속
2. 오른쪽 위 **Fork** → Owner 를 **내 계정**으로 → **Create fork**
3. 내 컴퓨터에 클론
   ```bash
   git clone https://github.com/내GitHub아이디/SUNGSHIN-_FIND.git sungshin-find-fork
   cd sungshin-find-fork
   ```

> 💡 중앙 레포 클론과 헷갈리지 않게 폴더 이름을 `sungshin-find-fork` 로 따로 받았습니다.
> 💡 OneDrive 폴더 안에 클론하면 동기화 때문에 `.git` 이 꼬일 수 있어서 `C:\Users\내이름\dev` 같은 폴더를 권장합니다.

---

## 3. 내 컴퓨터에 Jenkins 띄우기 (📘 8주차 7~18p)

> 강의에서는 GCP VM 에 Docker 를 설치했지만, 개인 실습은 **내 컴퓨터의 Docker Desktop** 을 사용합니다.
> (VM 생성, 방화벽, 고정 IP 단계는 필요 없음)

### 3-1. Jenkins 이미지 빌드
Docker Desktop 이 실행 중인 상태에서, Fork 레포 폴더의 터미널에서:

```bash
docker build -t myjenkins ./jenkins
```
- `jenkins/Dockerfile` 은 강의 8주차 10p Dockerfile 과 같고, 베이스 이미지만 `lts-jdk11` → `lts-jdk17` 입니다. (jdk11 은 최신 플러그인 설치가 안 됨)

### 3-2. Jenkins 컨테이너 실행

**Windows → PowerShell 에서 실행** (Git Bash 는 경로가 바뀌는 문제가 있어서 PowerShell 권장)
```powershell
docker run -d --name myjenkins -p 8080:8080 --restart=always -u root -v jenkins-home:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock myjenkins
```

**Mac → 터미널**
```bash
docker run -d --name myjenkins -p 8080:8080 --restart=always -u root -v jenkins-home:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock myjenkins
```

> **강의 명령어와 다른 점:** `--group-add $(stat -c '%g' /var/run/docker.sock)` 대신 `-u root`
> → Docker Desktop 에서는 `stat` 명령이 없거나 docker.sock 권한 구조가 달라서, 개인 실습용으로는 root 로 실행합니다.
> (공용 서버가 아닌 내 컴퓨터라서 괜찮지만, 실제 서버에서는 강의 방식 사용)

확인:
```bash
docker ps
```
→ `myjenkins` 가 `Up` 이면 성공

### 3-3. 초기 설정 (📘 8주차 14~18p)
1. 초기 비밀번호 확인
   ```bash
   docker exec myjenkins cat /var/jenkins_home/secrets/initialAdminPassword
   ```
2. 브라우저에서 **http://localhost:8080** 접속 → 비밀번호 붙여넣기
3. **Install suggested plugins**
4. 관리자 계정 생성 (아이디/비번 메모)
5. Instance URL `http://localhost:8080/` 그대로 → **Save and Finish**

### 3-4. 플러그인 추가 설치
**Manage Jenkins → Plugins → Available plugins** 에서 검색해서 설치

| 플러그인 | 용도 | 강의 |
|---|---|---|
| Docker Pipeline | Jenkinsfile 에서 `docker.build()` 사용 | 8주차 37p |
| GitHub Checks | PR 에 빌드 결과 표시 | 8주차 56p |
| Google Kubernetes Engine | GKE 배포 (`KubernetesEngineBuilder`) | 10주차 31p |
| Kubernetes Credentials | GKE 인증 | 10주차 31p |

설치 후 "Restart Jenkins when installation is complete" 체크

---

## 4. Jenkins 컨테이너에 gcloud / kubectl 설치 (📘 10주차 25~30p)

> ⚠️ **강의자료 명령어 그대로 하면 실패합니다.**
> - `apt-key` 명령이 최신 Debian(Jenkins 이미지 기반)에서 삭제됨
> - kubectl 저장소 `apt.kubernetes.io` 가 2024년에 종료됨
>
> → 아래 수정된 명령어를 사용하세요.

### 4-1. 컨테이너에 들어가기
```bash
docker exec -it myjenkins bash
```
→ 프롬프트가 `root@...:/#` 으로 바뀜

### 4-2. 한 줄씩 실행
```bash
apt-get update && apt-get install -y apt-transport-https ca-certificates gnupg curl
```
```bash
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | gpg --dearmor --yes -o /usr/share/keyrings/cloud.google.gpg
```
```bash
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" > /etc/apt/sources.list.d/google-cloud-sdk.list
```
```bash
apt-get update && apt-get install -y google-cloud-cli kubectl google-cloud-cli-gke-gcloud-auth-plugin
```

### 4-3. 설치 확인
```bash
gcloud --version
kubectl version --client
```
→ 둘 다 버전이 나오면 성공

### 4-4. 클러스터 연결 (📘 10주차 28~30p)
```bash
gcloud init
```
1. `[2] Sign in with a new Google Account` 선택 → `Y`
2. 출력된 링크를 브라우저에서 열고 로그인 → 나온 **인증 코드**를 터미널에 붙여넣기
3. 프로젝트 `iron-area-514720-j0` 선택
4. 기본 zone 설정 `Y` → `asia-northeast3-a` 직접 입력

```bash
gcloud container clusters get-credentials sungshin-find-cluster --zone asia-northeast3-a --project iron-area-514720-j0
kubectl get pods -n dev-내이름
exit
```

> 💡 확인은 꼭 **컨테이너 안에서** 하세요. `exit` 후 바깥(내 컴퓨터)에서 `kubectl` 이 되는 건 컨테이너 설치와 무관합니다.

---

## 5. Docker Hub 설정 (📘 8주차 34p, 10주차 51~52p)

### 5-1. 저장소 2개 만들기
hub.docker.com → **Create a repository**
| Repository Name | Visibility |
|---|---|
| `sungshin-find-backend` | **Public** |
| `sungshin-find-frontend` | **Public** |

> Public 으로 해야 GKE 가 별도 인증 없이 이미지를 pull 할 수 있습니다.

### 5-2. Access Token 만들기
Docker Hub → 오른쪽 위 프로필 → **Account settings → Personal access tokens → Generate new token**
- Description: `jenkins`
- Access permissions: **Read & Write**
- 생성된 토큰(`dckr_pat_...`)은 **한 번만 보이니까** 바로 다음 단계에 붙여넣기

> ⚠️ 토큰은 **톡방, 코드, 커밋 어디에도 올리지 마세요.** Jenkins Credentials 에만 등록합니다.

### 5-3. Jenkins 에 등록
Jenkins → **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**
| 칸 | 값 |
|---|---|
| Kind | Username with password |
| Username | 내 Docker Hub 아이디 |
| Password | 5-2 에서 만든 토큰 |
| ID | **`dockerhub`** (Jenkinsfile 에 이 이름으로 적혀 있음) |

---

## 6. GKE Credential 등록 (📘 10주차 53~54p)

Jenkins → Credentials → Global → **Add Credentials**
| 칸 | 값 |
|---|---|
| Kind | **Google Service Account from private key** |
| Project Name | `iron-area-514720-j0` |
| JSON key | 수진님께 받은 서비스 계정 JSON 키 파일 업로드 |

→ **Create** 후 목록에 생긴 **ID** 를 메모 (보통 `iron-area-514720-j0`)

> ⚠️ JSON 키 파일은 **레포 폴더에 넣지 마세요.** (다운로드 폴더에 두기)

---

## 7. 내 네임스페이스 확인

> 네임스페이스 `dev-내이름` 준비·확인 방법은 **인프라 가이드** 참고

Jenkins 배포 단계는 `Deployment` 만 적용하기 때문에, 아래 리소스가 **내 네임스페이스 안에 먼저 있어야** 파드가 정상적으로 뜹니다. 배포 전에 한 번 확인하세요.
```bash
kubectl get configmap,secret,svc,statefulset -n dev-내이름
```
→ `backend-config`, `postgres-secret`, `backend`, `frontend`, `postgres` 가 보이면 OK.
없으면 수진님께 요청하세요.

> ⚠️ 개인 네임스페이스에 `k8s/ingress/` 는 적용하지 마세요. 중앙 `dev` 의 Ingress 와 경로(`/`, `/api`)가 겹칩니다. 개인 확인은 [9-3](#9-3-서비스-접속-확인) 의 `port-forward` 를 사용합니다.

---

## 8. Fork 레포 수정 → Jenkins 연결

### 8-1. 수정할 곳은 딱 2군데

**① `Jenkinsfile` 맨 위 environment**
```groovy
    CREDENTIALS_ID = 'iron-area-514720-j0'   // 6번에서 메모한 GKE Credential ID
    DOCKERHUB_ID   = '내도커허브아이디'        // slwnt31 → 내 아이디
    NAMESPACE      = 'dev-내이름'             // dev → dev-내이름
```

**② k8s 매니페스트 이미지 주소**
- `k8s/backend/deployment.yml`
  ```yaml
  image: 내도커허브아이디/sungshin-find-backend:latest
  ```
- `k8s/frontend/deployment.yml`
  ```yaml
  image: 내도커허브아이디/sungshin-find-frontend:latest
  ```

> 💡 Jenkinsfile 배포 단계가 `sungshin-find-backend:latest` → `sungshin-find-backend:빌드번호` 로 sed 치환하므로 **이미지 이름 뒤는 꼭 `:latest`** 로 두세요. (📘 10주차 48~49p)

Fork 의 main 에 바로 커밋/푸시:
```bash
git add Jenkinsfile k8s/backend/deployment.yml k8s/frontend/deployment.yml
git commit -m "chore: 개인 CI/CD 실습용 설정 변경"
git push origin main
```

> ⚠️ 이 커밋은 **내 Fork 전용**입니다. 중앙 레포로 PR 올리지 마세요.

### 8-2. GitHub App 만들기 (📘 8주차 60~68p)
GitHub → Settings → **Developer settings → GitHub Apps → New GitHub App**

| 항목 | 값 |
|---|---|
| GitHub App name | `sungshin-find-jenkins-내이름` (전 세계에서 유일해야 함) |
| Homepage URL | 내 Fork 레포 주소 |
| Webhook | **Active 체크 해제** (내 컴퓨터 Jenkins 는 GitHub 에서 접속 불가) |
| Repository permissions | Checks: **Read and write** / Commit statuses: **Read and write** / Contents: Read-only / Metadata: Read-only / Pull requests: Read-only |
| Where can this GitHub App be installed? | Only on this account |

1. 생성 후 **App ID** 메모
2. 페이지 맨 아래 **Private keys → Generate a private key** → `.pem` 다운로드
   (화면 변화 없이 다운로드만 될 수 있으니 다운로드 폴더 확인)
3. Jenkins 가 읽을 수 있는 형식으로 변환 (📘 65p) — 다운로드 폴더에서:
   ```bash
   openssl pkcs8 -topk8 -inform PEM -outform PEM -in 받은파일이름.pem -out converted-jenkins.pem -nocrypt
   ```
4. 왼쪽 **Install App** → 내 계정 → **Only select repositories** → 내 Fork 레포 선택 → Install

### 8-3. Jenkins 에 GitHub App 등록
Jenkins → Credentials → Global → **Add Credentials**
| 칸 | 값 |
|---|---|
| Kind | GitHub App |
| ID | `github-app` |
| App ID | 8-2 에서 메모한 번호 |
| Key | `converted-jenkins.pem` 을 메모장/VS Code 로 열어 **전체 복사** 후 붙여넣기 |

→ **Test connection** 성공 확인 → **Create**

### 8-4. Multibranch Pipeline 만들기 (📘 8주차 41~44p, 10주차 55~58p)
Jenkins → **New Item** → 이름 `sungshin-find` → **Multibranch Pipeline** → OK

| 설정 | 값 |
|---|---|
| Branch Sources → Add source | **GitHub** |
| Credentials | 8-3 의 GitHub App (`App ID/******`) |
| Repository HTTPS URL | `https://github.com/내아이디/SUNGSHIN-_FIND` → **Validate** |
| Behaviors → Add | **Status Checks Properties** (📘 8주차 57~59p) |
| Build Configuration | Mode `by Jenkinsfile`, Script Path `Jenkinsfile` |
| **Scan Multibranch Pipeline Triggers** | ✅ **Periodically if not otherwise run** → Interval **1 minute** |

> **강의와 다른 점:** 강의는 GitHub Webhook 으로 즉시 빌드하지만, 내 컴퓨터(`localhost`)는 GitHub 이 접근할 수 없어서 **Jenkins 가 1분마다 GitHub 을 확인**하는 방식으로 대체합니다.

→ **Save** 하면 레포를 스캔하고 `main` 빌드가 시작됩니다.

---

## 9. 동작 확인

### 9-1. main 빌드 (= 자동 배포)
Jenkins → `sungshin-find` → **Branches → main** → 최근 빌드
- 5단계 모두 초록이면 성공: `Checkout code` → `Build image` → `Test image` → `Push image` → `Deploy to GKE`
- 실패 시 빌드 번호 → **Console Output** 확인 → [10. 문제해결](#10-문제해결-모음)

```bash
kubectl get pods -n dev-내이름
kubectl get deployment backend -n dev-내이름 -o jsonpath="{.spec.template.spec.containers[0].image}"
```
→ `내아이디/sungshin-find-backend:빌드번호` 이면 자동 배포 성공 📸

Docker Hub → 내 저장소 → **Tags** 에 `latest`, `빌드번호` 가 있는지도 확인 📸

### 9-2. PR 검사
1. Fork 레포에서 브랜치 하나 만들고 아무 수정(예: README 한 줄) → push
2. **내 Fork 레포 안에서** `main` 으로 PR 생성 (base repository 가 중앙 레포가 아니라 **내 Fork** 인지 확인!)
3. 1분 이내로 PR 하단에 Jenkins 체크 표시 → PR 단계(Checkout/Build/Test)만 실행되는지 확인 📸
4. 머지 → main 빌드가 다시 돌면서 이미지 태그가 다음 번호로 바뀌는지 확인 📸

### 9-3. 서비스 접속 확인
Ingress 대신 port-forward 로 내 컴퓨터에서 접속:
```bash
kubectl port-forward svc/frontend 8081:80 -n dev-내이름
```
→ 브라우저에서 http://localhost:8081 (종료는 `Ctrl+C`)

### 9-4. 롤백 실습 (선택)
```bash
kubectl rollout history deployment/backend -n dev-내이름
kubectl rollout undo deployment/backend -n dev-내이름
```

---

## 10. 문제해결 모음

> 중앙 파이프라인 구축 중 실제로 겪은 문제들입니다. 새로 겪은 문제는 **문제 / 원인 / 해결** 로 정리해서 하람님께 공유해주세요.

| # | 문제 | 원인 | 해결 |
|---|---|---|---|
| 1 | Windows Git Bash 에서 `gcloud --version` 이 `Python` 만 출력 | Git Bash 가 Windows 스토어의 가짜 python 을 실행 | `echo 'export CLOUDSDK_PYTHON="/c/Users/내이름/AppData/Local/Google/Cloud SDK/google-cloud-sdk/platform/bundledpython/python.exe"' >> ~/.bashrc` 후 `source ~/.bashrc` (또는 `gcloud.cmd` 사용) |
| 2 | Jenkins 권장 플러그인 설치 실패 | 강의의 `jenkins/jenkins:lts-jdk11` 은 지원 종료, 최신 플러그인은 Java 17+ 필요 | 베이스 이미지 `lts-jdk17` 사용 (`jenkins/Dockerfile` 반영됨) |
| 3 | 컨테이너에 gcloud/kubectl 이 설치 안 됨 (`executable file not found`) | 최신 Debian 에서 `apt-key` 삭제, `apt.kubernetes.io` 저장소 종료 | `gpg --dearmor` 로 키 등록 + 구글 cloud-sdk 저장소에서 kubectl 설치 ([4-2](#4-2-한-줄씩-실행)) |
| 4 | `kubectl` 이 `not found` 인데 `gcloud` 는 됨 | `exit` 후 **컨테이너 바깥**에서 확인함 | `docker exec myjenkins kubectl version --client` 처럼 컨테이너 안에서 확인 |
| 5 | Git Bash 에 한글이 섞인 긴 내용 붙여넣기 시 글자가 깨짐 | Git Bash 터미널 붙여넣기 인코딩 문제 | 파일 내용은 **VS Code 편집기**로 작성 |
| 6 | `vi` 에서 `[1]+ Stopped` 가 뜨고 빠져나옴 | `Ctrl+Z` 로 vi 가 일시정지됨 | `kill %1` 후 `rm -f .파일명.swp`, 또는 `cat > 파일 <<'EOF' ... EOF` 방식 사용 |
| 7 | GitHub App 을 Organization 레포에 설치 불가 | App 설치 범위가 "Only on this account" | 중앙 레포용은 "Any account" 로 생성 (개인 Fork 는 Only on this account 로 OK) |
| 8 | PR 검사 중인 코드가 서비스에 뜰 위험 | 매니페스트가 `:latest` + `imagePullPolicy: Always` 인데 강의 방식은 PR 에서도 push | Jenkinsfile `Push image` 단계에 `when { branch 'main' }` 추가 |
| 9 | 배포됐는데 Ingress 로 접속이 안 됨 | 강의 예제는 namespace 지정이 없어 `default` 에 배포됨 | `KubernetesEngineBuilder` 에 `namespace` 지정 |
| 10 | main 빌드가 Deploy 단계에서 약 6분 후 실패, 파드가 계속 `Pending` | 크레딧 절약으로 노드를 0 대로 줄여둠 → 배포 검증(`verifyDeployments`) 시간 초과 | `kubectl get nodes` 확인 → 노드 가동 시간에 재빌드 (**인프라 가이드** 노드 가동 시간 참고) |

### 빌드 단계별 자주 나는 에러
| 실패 단계 | Console Output | 해결 |
|---|---|---|
| Build image | `permission denied ... docker.sock` | Jenkins 를 `-u root` 로 실행했는지 확인 (3-2) |
| Push image | `unauthorized` / `denied` | `dockerhub` credential 의 Username/토큰 확인, 토큰 권한이 Read & Write 인지 |
| Deploy to GKE | `Credentials ... not found` | Jenkinsfile `CREDENTIALS_ID` 와 Jenkins Credentials 목록의 ID 일치 여부 |
| Deploy to GKE | `Forbidden` | 서비스 계정에 내 네임스페이스 권한이 없음 → 수진님께 요청 |
| Deploy to GKE | `timeout` | `kubectl get pods -n dev-내이름` → `Pending`(노드 꺼짐) / `ImagePullBackOff`(이미지 주소·Public 여부) / `CreateContainerConfigError`(7번 리소스 누락) |

---

## 11. 체크리스트

- [ ] Fork + 클론
- [ ] 내 컴퓨터 Jenkins 실행 + 플러그인 4개
- [ ] 컨테이너에 gcloud / kubectl 설치 + 클러스터 연결
- [ ] Docker Hub 저장소 2개(Public) + `dockerhub` credential
- [ ] GKE credential
- [ ] 내 네임스페이스(`dev-내이름`) 리소스 확인 (인프라 가이드)
- [ ] Jenkinsfile 3개 값 + 매니페스트 이미지 주소 수정
- [ ] GitHub App + Multibranch Pipeline (polling 1분)
- [ ] main 빌드 성공 + `dev-내이름` 배포 확인 📸
- [ ] Fork 내부 PR → Jenkins 체크 확인 📸
- [ ] 새로 겪은 문제 정리 → 하람님께 공유
- [ ] 개선점 있으면 **중앙 레포에 PR** (12/1까지)
