# BiddingGo 로컬 Kubernetes CI/CD 초기 구축 Runbook

이 문서는 Docker Desktop을 새로 설치한 상태에서 BiddingGo 프로젝트의 로컬 Kubernetes 기반 CI/CD 환경을 처음부터 구성하는 절차를 정리한 문서이다.

목표 흐름은 다음과 같다.

```text
Docker Desktop Kubernetes 활성화
  -> kubectl / helm 확인
  -> ingress-nginx 설치
  -> Argo CD 설치
  -> Jenkins 로컬 컨테이너 실행
  -> GHCR 인증 Secret 생성
  -> MariaDB / Redis 구성
  -> BiddingGo Manifest 적용
  -> Argo CD Application 생성
  -> Jenkins Pipeline으로 이미지 빌드 및 Manifest 갱
  -> Argo CD가 Kubernetes에 자동 배포
```

> 기준 환경
>
> - OS: macOS
> - Container Runtime: Docker Desktop
> - Kubernetes: Docker Desktop 내장 Kubernetes
> - CI: Jenkins
> - CD: Argo CD
> - Image Registry: GHCR
> - Manifest Repository: `beyond-sw-camp/be25-4th-biddingmate-biddinggo`
> - Backend Repository: `beyond-sw-camp/be25-2nd-biddingmate-biddinggo`
> - Service Name: `biddinggo`

---

## 0. 전체 구조

```text
Developer
  |
  | push
  v
GitHub Backend Repository
  - beyond-sw-camp/be25-2nd-biddingmate-biddinggo
  |
  | webhook or manual build
  v
Jenkins
  |
  | docker build
  | docker push
  v
GHCR
  - ghcr.io/beyond-sw-camp/be25-2nd-biddingmate-biddinggo:<BUILD_NUMBER>

Jenkins
  |
  | manifest repository clone
  | deployment.yaml image tag 수정
  | commit & push
  v
GitHub Manifest Repository
  - beyond-sw-camp/be25-4th-biddingmate-biddinggo
  |
  | watched by Argo CD
  v
Argo CD
  |
  | sync
  v
Docker Desktop Kubernetes
  - namespace: biddinggo
  - deployment: biddinggo-api-deploy
  - service: biddinggo-api-service
  - ingress: biddinggo-api-ingress
```

---

# 1. Docker Desktop Kubernetes 활성화

Docker Desktop을 새로 설치했다면 Kubernetes를 먼저 켠다.

1. Docker Desktop 실행
2. `Settings`
3. `Kubernetes`
4. `Enable Kubernetes` 체크
5. `Apply & Restart`

Kubernetes 활성화 후 터미널에서 확인한다.

```bash
docker version
kubectl version --client
kubectl config get-contexts
kubectl config use-context docker-desktop
kubectl cluster-info
kubectl get nodes
```

정상 예시:

```text
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane    ...   v...
```

---

# 2. 필수 CLI 설치 확인

macOS 기준으로 Homebrew가 설치되어 있다고 가정한다.

```bash
brew --version
```

필요한 CLI 설치:

```bash
brew install kubectl helm git gh
```

설치 확인:

```bash
kubectl version --client
helm version
git --version
gh --version
```

---

# 3. 작업 디렉토리 생성

```bash
mkdir -p ~/develop/biddinggo-cicd
cd ~/develop/biddinggo-cicd
```

---

# 4. Repository clone

## 4.1 Backend Repository clone

```bash
git clone https://github.com/beyond-sw-camp/be25-2nd-biddingmate-biddinggo.git
```

## 4.2 Manifest Repository clone

```bash
git clone https://github.com/beyond-sw-camp/be25-4th-biddingmate-biddinggo.git
```

확인:

```bash
ls
```

예상 결과:

```text
be25-2nd-biddingmate-biddinggo
be25-4th-biddingmate-biddinggo
```

---

# 5. Kubernetes Namespace 생성

BiddingGo 리소스를 배포할 namespace를 만든다.

```bash
kubectl create namespace biddinggo
```

이미 존재하면 아래처럼 나온다.

```text
Error from server (AlreadyExists): namespaces "biddinggo" already exists
```

그 경우 무시해도 된다.

확인:

```bash
kubectl get ns
kubectl get ns biddinggo
```

---

# 6. GHCR Image Pull Secret 생성

BiddingGo 이미지는 GHCR에서 pull한다.  
Kubernetes가 GHCR 이미지를 pull하려면 `ghcr-secret`이 필요하다.

## 6.1 GitHub username 확인

```bash
gh auth status
```

로그인이 안 되어 있다면:

```bash
gh auth login
```

본인 GitHub username 확인:

```bash
gh api user --jq .login
```

## 6.2 GHCR Token 준비

GHCR private package를 pull하려면 GitHub Token에 최소한 다음 권한이 필요하다.

```text
read:packages
```

Jenkins가 Manifest repository에 push까지 해야 한다면 Jenkins credential로 사용할 토큰에는 다음 권한이 필요하다.

```text
repo
write:packages
read:packages
```

환경변수로 저장한다.

```bash
export GITHUB_USERNAME="<GITHUB_USERNAME>"
export GITHUB_TOKEN="<GITHUB_TOKEN>"
export GITHUB_EMAIL="<GITHUB_EMAIL>"
```

예시:

```bash
export GITHUB_USERNAME="jin605"
export GITHUB_TOKEN="ghp_xxxxxxxxxxxxxxxxxxxx"
export GITHUB_EMAIL="jinddd3@gmail.com"
```

## 6.3 Docker에서 GHCR 로그인 확인

```bash
echo "$GITHUB_TOKEN" | docker login ghcr.io -u "$GITHUB_USERNAME" --password-stdin
```

확인 후 logout:

```bash
docker logout ghcr.io
```

## 6.4 Kubernetes Secret 생성

```bash
kubectl create secret docker-registry ghcr-secret \
  --namespace=biddinggo \
  --docker-server=ghcr.io \
  --docker-username="$GITHUB_USERNAME" \
  --docker-password="$GITHUB_TOKEN" \
  --docker-email="$GITHUB_EMAIL"
```

이미 만들어져 있다면 삭제 후 다시 생성한다.

```bash
kubectl delete secret ghcr-secret -n biddinggo

kubectl create secret docker-registry ghcr-secret \
  --namespace=biddinggo \
  --docker-server=ghcr.io \
  --docker-username="$GITHUB_USERNAME" \
  --docker-password="$GITHUB_TOKEN" \
  --docker-email="$GITHUB_EMAIL"
```

확인:

```bash
kubectl get secret ghcr-secret -n biddinggo
```

---

# 7. ingress-nginx 설치

Ingress를 사용하려면 Ingress Controller가 필요하다.  
Docker Desktop Kubernetes에서는 Helm으로 `ingress-nginx`를 설치하는 방식이 깔끔하다.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

설치:

```bash
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443
```

확인:

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

정상 확인:

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=Ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=180s
```

---

# 8. cert-manager 설치

BiddingGo Ingress manifest에는 `cert-manager.io/cluster-issuer: letsencrypt-prod` annotation이 있다.  
따라서 cert-manager가 없으면 Ingress TLS 인증서 발급 부분에서 문제가 생길 수 있다.

다만 로컬 Docker Desktop 환경에서는 실제 `bidding-go.shop`, `api.bidding-go.shop` 도메인이 내 로컬로 연결되지 않으면 Let's Encrypt 인증서 발급은 정상적으로 되지 않는다.

그래도 manifest 호환을 위해 cert-manager는 설치해둔다.

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

```bash
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

확인:

```bash
kubectl get pods -n cert-manager
```

---

# 9. 로컬용 ClusterIssuer 생성

실제 운영용 Let's Encrypt를 바로 쓰기 전에 로컬에서는 self-signed issuer를 먼저 쓰는 것이 안전하다.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  selfSigned: {}
EOF
```

확인:

```bash
kubectl get clusterissuer
```

> 주의
>
> 이 설정은 로컬 테스트용이다.  
> 실제 운영 환경에서는 Let's Encrypt HTTP-01 또는 DNS-01 방식의 ClusterIssuer를 따로 구성해야 한다.

---

# 10. 로컬 hosts 설정

Ingress host를 로컬에서 테스트하려면 `/etc/hosts`에 도메인을 등록한다.

```bash
sudo vi /etc/hosts
```

아래 내용 추가:

```text
127.0.0.1 bidding-go.shop
127.0.0.1 api.bidding-go.shop
```

확인:

```bash
cat /etc/hosts | grep bidding
```

Docker Desktop의 ingress-nginx를 NodePort로 구성했기 때문에 로컬 접속 주소는 다음과 같다.

```text
http://bidding-go.shop:30080
http://api.bidding-go.shop:30080
https://bidding-go.shop:30443
https://api.bidding-go.shop:30443
```

단, HTTPS는 self-signed 인증서라 브라우저에서 경고가 뜰 수 있다.

---

# 11. MariaDB 배포

BiddingGo backend configmap 기준 DB host는 다음과 같다.

```text
DB_HOST=mariadb-service
DB_PORT=3306
DB_NAME=biddinggo
DB_USERNAME=biddinggo
```

따라서 Kubernetes 안에 `mariadb-service` 이름으로 MariaDB를 띄운다.

## 11.1 MariaDB Secret 생성

이 repository의 `infra/db/mariadb-deployment.yaml` 기준으로 MariaDB root password는
`mariadb-secret` Secret의 `ROOT_PASSWORD` 키에서 읽는다.

먼저 `.env`에 MariaDB root password를 저장한다.

```bash
export MARIADB_ROOT_PASSWORD="<MARIADB_ROOT_PASSWORD>"
```

예시:

```bash
export MARIADB_ROOT_PASSWORD="root1234"
```

터미널에 적용한다.

```bash
source .env
```

값이 비어 있지 않은지 확인한다.

```bash
echo "${MARIADB_ROOT_PASSWORD:0:2}****"
```

Secret을 생성한다.

```bash
kubectl create secret generic mariadb-secret \
  -n biddinggo \
  --from-literal=ROOT_PASSWORD="$MARIADB_ROOT_PASSWORD"
```

이미 존재하면 삭제 후 다시 생성한다.

```bash
kubectl delete secret mariadb-secret -n biddinggo --ignore-not-found

kubectl create secret generic mariadb-secret \
  -n biddinggo \
  --from-literal=ROOT_PASSWORD="$MARIADB_ROOT_PASSWORD"
```

확인:

```bash
kubectl describe secret mariadb-secret -n biddinggo
```

정상이라면 `ROOT_PASSWORD`가 `0 bytes`가 아니어야 한다.

```text
Data
====
ROOT_PASSWORD:  8 bytes
```

> 참고
>
> 현재 `infra/db/mariadb-deployment.yaml`은 DB 이름, 일반 유저, 일반 유저 비밀번호를
> YAML 안에 직접 가지고 있다.
>
> ```yaml
> - name: MARIADB_DATABASE
>   value: "biddinggo"
> - name: MARIADB_USER
>   value: "biddinggo"
> - name: MARIADB_PASSWORD
>   value: "biddinggo"
> ```
>
> 따라서 현재 파일을 그대로 쓰는 경우 Secret에는 `ROOT_PASSWORD`만 있으면 된다.

## 11.2 MariaDB PV / PVC / Deployment / Service 생성

MariaDB 리소스는 repository에 있는 YAML 파일을 사용해서 생성한다.

현재 사용하는 파일:

```bash
infra/db/mariadb-pv.yaml
infra/db/mariadb-pvc.yaml
infra/db/mariadb-deployment.yaml
infra/db/mariadb-service.yaml
```

> Mac 환경 주의 (수정 완료)
>
> 현재 `infra/db/mariadb-pv.yaml`의 `hostPath.path`가 Windows Docker Desktop 기준 경로라면
> Mac 환경에 맞게 경로를 확인한 뒤 적용해야 한다.
>
> ```yaml
> hostPath:
>   path: /run/desktop/mnt/host/c/Users/Playdata/mariadb-biddinggo
> ```
>
> 위처럼 `c/Users/...` 경로라면 Windows 기준이다.

적용:

```bash
kubectl apply -f infra/db/mariadb-pv.yaml
kubectl apply -f infra/db/mariadb-pvc.yaml
kubectl apply -f infra/db/mariadb-deployment.yaml
kubectl apply -f infra/db/mariadb-service.yaml
```

확인:

```bash
kubectl get pv
kubectl get pvc -n biddinggo
kubectl get pod -n biddinggo -l app=mariadb-app
kubectl get svc -n biddinggo mariadb-service
```

MariaDB Pod가 준비될 때까지 기다린다.

```bash
kubectl wait pod -n biddinggo \
  --for=condition=Ready \
  --selector=app=mariadb-app \
  --timeout=180s
```

로그 확인:

```bash
kubectl logs -n biddinggo -l app=mariadb-app
```

---

# 12. Redis 배포

BiddingGo backend configmap 기준 Redis host는 다음과 같다.

```text
REDIS_HOST=redis-service
REDIS_PORT=6379
REDIS_USERNAME=default
```

## 12.1 Redis Secret 생성

이 repository의 `infra/db/redis-deployment.yaml` 기준으로 Redis password는
`biddinggo-env-secret` Secret의 `REDIS_PASSWORD` 키에서 읽는다.

먼저 `.env`에 Redis password를 저장한다.

```bash
export REDIS_PASSWORD="<REDIS_PASSWORD>"
```

예시:

```bash
export REDIS_PASSWORD="redis1234"
```

터미널에 적용한다.

```bash
source .env
```

값이 비어 있지 않은지 확인한다.

```bash
echo "${REDIS_PASSWORD:0:2}****"
```

`biddinggo-env-secret`은 Backend Deployment도 같이 참조한다.
따라서 이 Secret을 이미 만들었다면 `REDIS_PASSWORD`가 포함되어 있는지 확인해야 한다.

```bash
kubectl describe secret biddinggo-env-secret -n biddinggo
```

아직 만들지 않았거나 다시 만들려면 13장의 Backend Secret 생성 단계에서
`REDIS_PASSWORD`를 포함해 생성한다.

Redis만 먼저 테스트하려면 다음처럼 최소 Secret을 만들 수 있다.

```bash
kubectl delete secret biddinggo-env-secret -n biddinggo --ignore-not-found

kubectl create secret generic biddinggo-env-secret \
  -n biddinggo \
  --from-literal=REDIS_PASSWORD="$REDIS_PASSWORD"
```

## 12.2 Redis Deployment / Service 생성

Redis 리소스는 repository에 있는 YAML 파일을 사용해서 생성한다.

현재 사용하는 파일:

```bash
infra/db/redis-deployment.yaml
infra/db/redis-service.yaml
```

적용:

```bash
kubectl apply -f infra/db/redis-deployment.yaml
kubectl apply -f infra/db/redis-service.yaml
```

확인:

```bash
kubectl get pod -n biddinggo -l app=redis-app
kubectl get svc -n biddinggo redis-service
```

Redis Pod가 준비될 때까지 기다린다.

```bash
kubectl wait pod -n biddinggo \
  --for=condition=Ready \
  --selector=app=redis-app \
  --timeout=180s
```

Redis 로그 확인:

```bash
kubectl logs -n biddinggo -l app=redis-app
```

---

# 13. BiddingGo Backend Secret 생성

Backend Deployment는 `biddinggo-env-secret`을 참조한다.  
따라서 백엔드 실행에 필요한 민감 정보를 Secret으로 만든다.

아래 값은 로컬 테스트용 예시이다. 실제 값은 프로젝트 설정에 맞게 수정해야 한다.

```bash
kubectl create secret generic biddinggo-env-secret \
  -n biddinggo \
  --from-literal=DB_PASSWORD="biddinggo1234" \
  --from-literal=REDIS_PASSWORD="redis1234" \
  --from-literal=JWT_SECRET="local-jwt-secret-key-must-be-long-enough-for-test" \
  --from-literal=OPENAI_API_KEY="dummy" \
  --from-literal=GOOGLE_CLIENT_ID="dummy" \
  --from-literal=GOOGLE_CLIENT_SECRET="dummy" \
  --from-literal=KAKAO_CLIENT_ID="dummy" \
  --from-literal=KAKAO_CLIENT_SECRET="dummy" \
  --from-literal=TOSS_SECRET_KEY="dummy" \
  --from-literal=CLOUDFLARE_R2_ACCESS_KEY="dummy" \
  --from-literal=CLOUDFLARE_R2_SECRET_KEY="dummy"
```

이미 존재하면 삭제 후 다시 생성한다.

```bash
kubectl delete secret biddinggo-env-secret -n biddinggo
```

확인:

```bash
kubectl get secret biddinggo-env-secret -n biddinggo
```

> 주의
>
> 위 Secret 값은 로컬 실행용 placeholder이다.  
> 애플리케이션 실행 중 특정 환경 변수가 없다는 에러가 발생하면 `kubectl logs`로 확인한 뒤 해당 key를 `biddinggo-env-secret`에 추가해야 한다.

---

# 14. BiddingGo ConfigMap 적용

Manifest Repository로 이동한다.

```bash
cd ~/develop/biddinggo-cicd/be25-4th-biddingmate-biddinggo
```

Backend ConfigMap 적용:

```bash
kubectl apply -f infra/k8s/backend/configmap.yaml
```

확인:

```bash
kubectl get configmap -n biddinggo
kubectl describe configmap biddinggo-config -n biddinggo
```

---

# 14.1 GHCR 이미지 수동 빌드 및 Push

Backend / Frontend Deployment를 적용하기 전에 Kubernetes가 pull할 이미지가 GHCR에 먼저 올라가 있어야 한다.

이미지가 없거나 image tag가 잘못된 이미지를 가리키면 Pod가 다음 상태가 된다.

```text
ImagePullBackOff
CrashLoopBackOff
Startup probe failed
```

로컬 실습에서는 Jenkins가 이미지를 push하기 전이므로 한 번 수동으로 이미지를 빌드하고 GHCR에 push한다.

## 14.1.1 GHCR 로그인

Manifest repository의 `.env`를 적용한다.

```bash
cd /Users/jin605/develop/biddinggo/biddinggo-deploy
source .env
```

GHCR에 로그인한다.

```bash
echo "$GITHUB_TOKEN" | docker login ghcr.io -u "$GITHUB_USERNAME" --password-stdin
```

## 14.1.2 Backend 이미지 빌드 및 Push

Backend repository로 이동한다.

```bash
cd /Users/jin605/develop/biddinggo/biddinggo-backend
```

Backend 이미지를 빌드한다.

```bash
docker build \
  -t ghcr.io/jin605/biddinggo-backend:2 \
  -t ghcr.io/jin605/biddinggo-backend:latest \
  .
```

Backend Docker build 로그에는 Java / Gradle / jar 관련 단계가 보여야 한다.

```text
gradle
bootJar
app.jar
java
```

만약 `node`, `npm run build`, `nginx`, `/usr/share/nginx/html` 같은 로그가 보이면 Frontend directory에서 잘못 빌드한 것이다.

GHCR에 push한다.

```bash
docker push ghcr.io/jin605/biddinggo-backend:2
docker push ghcr.io/jin605/biddinggo-backend:latest
```

Manifest repository의 Backend Deployment image도 같은 tag로 맞춘다.

```yaml
image: ghcr.io/jin605/biddinggo-backend:2
```

파일 위치:

```bash
infra/k8s/backend/deployment.yaml
```

## 14.1.3 Frontend 이미지 빌드 및 Push

Frontend repository로 이동한다.

```bash
cd /Users/jin605/develop/biddinggo/biddinggo-frontend
```

Frontend는 `VITE_API_BASE_URL`이 build time에 JavaScript bundle 안으로 들어간다.
따라서 Docker build 때 API 주소를 반드시 build arg로 전달해야 한다.

```bash
docker build \
  --build-arg VITE_API_BASE_URL=https://api.bidding-go.shop:30443 \
  --build-arg VITE_TOSS_CLIENT_KEY="$VITE_TOSS_CLIENT_KEY" \
  -t ghcr.io/jin605/biddinggo-frontend:2 \
  -t ghcr.io/jin605/biddinggo-frontend:latest \
  .
```

Frontend Docker build 로그에는 Node / Vite / nginx 관련 단계가 보인다.

```text
node
npm ci
npm run build
nginx
/usr/share/nginx/html
```

GHCR에 push한다.

```bash
docker push ghcr.io/jin605/biddinggo-frontend:2
docker push ghcr.io/jin605/biddinggo-frontend:latest
```

Manifest repository의 Frontend Deployment image도 같은 tag로 맞춘다.

```yaml
image: ghcr.io/jin605/biddinggo-frontend:2
```

파일 위치:

```bash
infra/k8s/frontend/deployment.yaml
```

> 주의
>
> Backend와 Frontend image tag를 섞으면 안 된다.
>
> ```text
> ghcr.io/jin605/biddinggo-backend:<tag>   -> Spring Boot backend image
> ghcr.io/jin605/biddinggo-frontend:<tag>  -> Vite/nginx frontend image
> ```
>
> Frontend image를 backend tag로 push하면 backend Pod가 8080 포트를 열지 못해
> startup probe가 실패한다.

---

# 15. Backend Manifest 수동 적용

CI/CD를 연결하기 전, 먼저 현재 Manifest가 Kubernetes에서 뜨는지 수동으로 확인한다.

먼저 Backend Deployment image가 GHCR에 push한 tag와 일치하는지 확인한다.

```bash
grep "image:" infra/k8s/backend/deployment.yaml
```

```bash
kubectl apply -f infra/k8s/backend/deployment.yaml
kubectl apply -f infra/k8s/backend/service.yaml
kubectl apply -f infra/k8s/backend/ingress.yaml
```

확인:

```bash
kubectl get deploy -n biddinggo
kubectl get pods -n biddinggo
kubectl get svc -n biddinggo
kubectl get ingress -n biddinggo
```

Rollout 확인:

```bash
kubectl rollout status deployment/biddinggo-api-deploy -n biddinggo
```

로그 확인:

```bash
kubectl logs -f deployment/biddinggo-api-deploy -n biddinggo
```

---

# 16. Backend API 로컬 테스트

## 16.1 Service port-forward

Ingress보다 먼저 Service port-forward로 확인한다.

```bash
kubectl port-forward service/biddinggo-api-service 8080:8080 -n biddinggo
```

다른 터미널에서 확인:

```bash
curl http://localhost:8080/actuator/health
```

정상 예시:

```json
{ "status": "UP" }
```

## 16.2 Ingress로 확인

ingress-nginx를 NodePort `30080`, `30443`으로 설치했으므로 다음으로 확인한다.

```bash
curl http://api.bidding-go.shop:30080/actuator/health
```

HTTPS 확인:

```bash
curl -k https://api.bidding-go.shop:30443/actuator/health
```

---

# 17. Frontend Manifest 수동 적용

프론트엔드까지 같이 올릴 경우 적용한다.

먼저 Frontend Deployment image가 GHCR에 push한 tag와 일치하는지 확인한다.

```bash
grep "image:" infra/k8s/frontend/deployment.yaml
```

```bash
kubectl apply -f infra/k8s/frontend/deployment.yaml
kubectl apply -f infra/k8s/frontend/service.yaml
kubectl apply -f infra/k8s/frontend/ingress.yaml
```

확인:

```bash
kubectl get deploy -n biddinggo
kubectl get pods -n biddinggo
kubectl get svc -n biddinggo
kubectl get ingress -n biddinggo
```

프론트 Service port-forward:

```bash
kubectl port-forward service/biddinggo-web-service 8088:80 -n biddinggo
```

브라우저에서 접속:

```text
http://localhost:8088
```

Ingress 접속:

```text
http://bidding-go.shop:30080
https://bidding-go.shop:30443
```

---

# 18. Argo CD 설치

Argo CD namespace 생성:

```bash
kubectl create namespace argocd
```

이미 존재하면 무시한다.

Argo CD 설치:

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Pod 확인:

```bash
kubectl get pods -n argocd
```

Ready 대기:

```bash
kubectl wait --for=condition=Ready pod \
  --all \
  -n argocd \
  --timeout=300s
```

---

# 19. Argo CD 접속

Argo CD 접속은 두 가지 방식 중 하나를 사용한다.

## 19.1 Ingress로 접속

이 repository에는 Argo CD Ingress manifest가 있다.

```bash
infra/argocd/ingress.yaml
```

로컬 hosts 파일에 Argo CD host를 추가한다.

```text
127.0.0.1 argocd.bidding-go.shop
```

Ingress manifest 적용:

```bash
kubectl apply -f infra/argocd/ingress.yaml
```

확인:

```bash
kubectl get ingress -n argocd
```

Docker Desktop의 ingress-nginx를 NodePort `30443`으로 설치했으므로 브라우저에서 접속한다.

```text
https://argocd.bidding-go.shop:30443
```

로컬 self-signed 인증서라 브라우저에서 인증서 경고가 뜰 수 있다.

## 19.2 Port-forward로 접속

Ingress를 쓰지 않고 임시로 접속하려면 port-forward를 사용한다.

```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

브라우저 접속:

```text
https://localhost:8081
```

## 19.3 초기 admin 비밀번호 확인

다른 터미널에서 실행:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

로그인 정보:

```text
username: admin
password: 위 명령어 결과
```

---

# 20. Argo CD Application 생성

Manifest Repository의 `infra/k8s` 경로를 Argo CD가 감시하도록 Application을 만든다.
이 repository에는 Application manifest가 이미 있으므로 CLI로 `argocd app create`를 실행하지 않고 YAML 파일을 적용한다.

```bash
infra/argocd/application.yaml
```

현재 Application manifest의 핵심 설정은 다음과 같다.

```yaml
source:
  repoURL: https://github.com/jin605/biddinggo-deploy.git
  targetRevision: main
  path: infra/k8s
  directory:
    recurse: true

destination:
  server: https://kubernetes.default.svc
  namespace: biddinggo

syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

적용:

```bash
kubectl apply -f infra/argocd/application.yaml
```

확인:

```bash
kubectl get application -n argocd
kubectl describe application biddinggo-app -n argocd
```

Kubernetes에서도 확인:

```bash
kubectl get all -n biddinggo
kubectl get ingress -n biddinggo
```

---

# 21. Jenkins 로컬 컨테이너 구성

현재 BiddingGo Jenkinsfile은 `docker build`, `docker push` 명령어를 직접 실행한다.  
따라서 Jenkins가 Docker CLI를 사용할 수 있어야 한다.

현재 repository에는 Jenkins용 Docker Compose 파일이 있다.

```bash
jenkins/docker-compose.yml
```

이 compose는 Jenkins를 로컬 Docker 컨테이너로 띄우고, host Docker socket을 mount해서 Jenkins 컨테이너 안에서 host Docker daemon을 사용하게 하는 구조다.

```text
Jenkins container
  -> /var/run/docker.sock mount
  -> host Docker daemon 사용
  -> docker build / docker push 수행
```

## 21.1 macOS Docker Desktop 주의사항

현재 `jenkins/docker-compose.yml`에는 Linux 기준 설정이 섞여 있다.

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
  - /usr/bin/docker:/usr/bin/docker

group_add:
  - "<docker_gid>"
```

macOS Docker Desktop에서는 보통 `/usr/bin/docker`가 없다.

확인:

```bash
which docker
```

Apple Silicon / Homebrew 기준으로는 보통 다음 중 하나다.

```text
/opt/homebrew/bin/docker
/usr/local/bin/docker
```

따라서 macOS에서는 다음 중 하나로 정리해야 한다.

```text
방법 A: Jenkins image 안에 Docker CLI가 포함된 custom Jenkins image를 만든다.
방법 B: compose의 docker binary mount 경로를 macOS의 실제 docker 경로로 바꾼다.
```

학습용으로는 방법 A가 더 안정적이다. host의 `/var/run/docker.sock`만 mount하고, Docker CLI는 Jenkins image 안에 설치해두는 방식이다.

> 주의
>
> `group_add: "<docker_gid>"`도 Linux host 기준이다.  
> macOS Docker Desktop에서는 그대로 쓰면 안 되므로 실제 실행 전 제거하거나 macOS에 맞게 조정해야 한다.

## 21.2 Jenkins Compose 실행

현재 compose 기준 Jenkins 접속 포트는 `8081`이다.

```yaml
ports:
  - "8081:8080"
```

실행:

```bash
cd /Users/jin605/develop/biddinggo/biddinggo-deploy
docker compose -f jenkins/docker-compose.yml up -d
```

확인:

```bash
docker ps
docker logs -f jenkins
```

브라우저 접속:

```text
http://localhost:8081
```

## 21.3 Jenkins 컨테이너에서 Docker 사용 확인

Jenkins 컨테이너 안에서 Docker CLI가 동작하는지 확인한다.

```bash
docker exec -it jenkins docker version
docker exec -it jenkins docker ps
```

정상이라면 Jenkins 컨테이너 안에서 host Docker daemon 정보를 볼 수 있다.

만약 다음과 같은 에러가 나면 Docker CLI가 Jenkins 컨테이너 안에 없거나 socket 권한 문제가 있는 것이다.

```text
docker: command not found
permission denied while trying to connect to the Docker daemon socket
```

이 경우 `jenkins/docker-compose.yml`의 macOS Docker CLI 경로와 socket mount 설정을 다시 확인한다.

## 21.4 Jenkins 초기 비밀번호 확인

```bash
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

## 21.5 Jenkins Ingress 사용 시 주의사항

현재 repository에는 Jenkins용 Ingress manifest도 있다.

```bash
jenkins/ingress.yaml
```

이 파일은 Kubernetes 안에 Jenkins Pod를 띄우는 방식이 아니라,
Kubernetes Ingress가 로컬 Docker Compose로 떠 있는 Jenkins로 요청을 넘기는 구조다.

```text
Browser
  -> https://jenkins.bidding-go.shop:30443
  -> ingress-nginx
  -> Service jenkins-external
  -> Endpoints JENKINS_SERVER_IP:8081
  -> Mac host의 Jenkins compose container
```

따라서 `jenkins/ingress.yaml`의 `JENKINS_SERVER_IP`는 실제 Mac host IP로 바꿔야 한다.

```yaml
subsets:
  - addresses:
      - ip: JENKINS_SERVER_IP
```

Mac에서 현재 IP 확인:

```bash
ipconfig getifaddr en0
```

유선/환경에 따라 `en0`가 비어 있으면 다음도 확인한다.

```bash
ipconfig getifaddr en1
```

Ingress host도 `/etc/hosts`에 추가한다.

```text
127.0.0.1 jenkins.bidding-go.shop
```

적용:

```bash
kubectl apply -f jenkins/ingress.yaml
```

접속:

```text
https://jenkins.bidding-go.shop:30443
```

> 주의
>
> `JENKINS_SERVER_IP` placeholder를 그대로 두면 동작하지 않는다.  
> 또한 macOS 방화벽이나 Docker Desktop 네트워크 설정에 따라 Kubernetes Pod가 Mac host의 `8081` 포트에 접근하지 못할 수 있다.
> 이 경우 ngrok 또는 port-forward 방식으로 Jenkins에 접속한다.

---

# 22. Jenkins 초기 설정

Jenkins 화면에서 다음 순서로 설정한다.

```text
1. 초기 비밀번호 입력
2. Install suggested plugins 선택
3. 관리자 계정 생성
4. Jenkins URL: http://localhost:8081
```

추가로 필요한 플러그인이 없다면 기본 추천 플러그인으로 충분하다.  
Pipeline, Git, Credentials Binding, GitHub 관련 플러그인이 설치되어 있어야 한다.

---

# 23. Jenkins Credential 등록

BiddingGo Jenkinsfile은 credential id로 `github-token`을 사용한다.  
따라서 Jenkins에 같은 ID로 credential을 등록해야 한다.

Jenkins UI에서:

```text
Manage Jenkins
  -> Credentials
  -> System
  -> Global credentials
  -> Add Credentials
```

설정:

```text
Kind: Username with password
ID: github-token
Username: <GITHUB_USERNAME>
Password: <GITHUB_TOKEN>
Description: GitHub token for GHCR and manifest repository
```

중요:

```text
ID는 반드시 github-token
```

Jenkinsfile에서 다음처럼 사용하기 때문이다.

```groovy
withCredentials([usernamePassword(
  credentialsId: 'github-token',
  usernameVariable: 'GITHUB_USER',
  passwordVariable: 'GITHUB_TOKEN'
)])
```

---

# 24. Jenkins Backend Pipeline Job 생성

Jenkins UI에서 새 Job을 만든다.

```text
New Item
  -> Item name: biddinggo-backend
  -> Pipeline 선택
  -> OK
```

Pipeline 설정:

```text
Definition: Pipeline script from SCM
SCM: Git
Repository URL: https://github.com/jin605/<BACKEND_REPOSITORY>.git
Branch Specifier: */main
Script Path: Jenkinsfile
```

저장 후 `Build Now`를 실행한다.

---

# 25. Jenkins Pipeline 동작 확인

Jenkins 빌드가 실행되면 다음 단계가 수행된다.

```text
Docker Build
  -> ghcr.io/jin605/biddinggo-backend:<BUILD_NUMBER> 생성

Push to GHCR
  -> GHCR에 이미지 push

Update Backend Deployment
  -> Manifest repository clone
  -> infra/k8s/backend/deployment.yaml image tag 수정
  -> commit & push
```

Jenkins 로그에서 확인할 부분:

```text
docker build 성공 여부
docker push ghcr.io/jin605/biddinggo-backend:<BUILD_NUMBER> 성공 여부
git clone 성공 여부
sed로 image tag 수정 여부
git commit / push 성공 여부
```

> 주의
>
> Backend Jenkinsfile도 개인 GHCR 기준으로 맞아 있어야 한다.
>
> ```groovy
> GHCR_OWNER = 'jin605'
> IMAGE_NAME = 'biddinggo-backend'
> ```
>
> Manifest repository URL도 현재 deploy repository와 맞아야 한다.
>
> ```groovy
> CICD_REPO_URL = 'github.com/jin605/biddinggo-deploy.git'
> ```

---

# 26. Jenkins 빌드 후 Argo CD 확인

Jenkins가 Manifest Repository에 push하면 Argo CD가 변경을 감지한다.

Argo CD Application 확인:

```bash
kubectl get application biddinggo-app -n argocd
kubectl describe application biddinggo-app -n argocd
```

Kubernetes Deployment image 확인:

```bash
kubectl get deployment biddinggo-api-deploy -n biddinggo \
  -o=jsonpath='{.spec.template.spec.containers[0].image}'; echo
```

Frontend Deployment image 확인:

```bash
kubectl get deployment biddinggo-web-deploy -n biddinggo \
  -o=jsonpath='{.spec.template.spec.containers[0].image}'; echo
```

Pod rollout 확인:

```bash
kubectl rollout status deployment/biddinggo-api-deploy -n biddinggo
```

Pod 확인:

```bash
kubectl get pods -n biddinggo
```

로그 확인:

```bash
kubectl logs -f deployment/biddinggo-api-deploy -n biddinggo
```

---

# 27. GitHub Webhook 연결

처음에는 Jenkins에서 `Build Now`로 수동 실행해도 된다.  
자동화를 하려면 GitHub Webhook을 연결한다.

로컬 Jenkins는 외부 GitHub에서 직접 접근할 수 없다.  
따라서 ngrok 같은 터널링 도구가 필요하다.

## 27.1 ngrok 설치

```bash
brew install ngrok
```

로그인과 토큰 설정은 ngrok 계정에서 발급받은 값을 사용한다.

```bash
ngrok config add-authtoken <NGROK_AUTH_TOKEN>
```

## 27.2 Jenkins 포트 공개

Jenkins가 `http://localhost:8081`에서 실행 중이므로 다음을 실행한다.

```bash
ngrok http 8081
```

ngrok이 제공하는 HTTPS 주소를 확인한다.

예시:

```text
https://abc123.ngrok-free.app
```

## 27.3 GitHub Webhook 등록

Backend Repository로 이동:

```text
https://github.com/jin605/<BACKEND_REPOSITORY>
```

설정:

```text
Settings
  -> Webhooks
  -> Add webhook
```

입력:

```text
Payload URL: https://abc123.ngrok-free.app/github-webhook/
Content type: application/json
Events: Just the push event
Active: checked
```

Jenkins Job 설정에서 GitHub hook trigger를 활성화한다.

```text
Configure
  -> Build Triggers
  -> GitHub hook trigger for GITScm polling 체크
```

이후 backend repo에 push하면 Jenkins가 자동 실행된다.

---

# 28. 전체 확인 명령어 모음

## 28.1 Cluster 확인

```bash
kubectl config current-context
kubectl get nodes
kubectl get ns
```

## 28.2 BiddingGo 리소스 확인

```bash
kubectl get all -n biddinggo
kubectl get configmap -n biddinggo
kubectl get secret -n biddinggo
kubectl get ingress -n biddinggo
```

## 28.3 Backend 확인

```bash
kubectl get deploy biddinggo-api-deploy -n biddinggo
kubectl get pods -n biddinggo
kubectl logs -f deployment/biddinggo-api-deploy -n biddinggo
```

## 28.4 현재 이미지 확인

```bash
kubectl get deployment biddinggo-api-deploy -n biddinggo \
  -o=jsonpath='{.spec.template.spec.containers[0].image}'; echo
```

## 28.5 Rollout 확인

```bash
kubectl rollout status deployment/biddinggo-api-deploy -n biddinggo
kubectl rollout history deployment/biddinggo-api-deploy -n biddinggo
```

## 28.6 Rollback

```bash
kubectl rollout undo deployment/biddinggo-api-deploy -n biddinggo
```

특정 revision으로 rollback:

```bash
kubectl rollout undo deployment/biddinggo-api-deploy -n biddinggo --to-revision=<REVISION_NUMBER>
```

## 28.7 Argo CD 확인

```bash
kubectl get application -n argocd
kubectl describe application biddinggo-app -n argocd
```

## 28.8 Jenkins 확인

```bash
docker ps
docker logs -f jenkins
```

---

# 29. 자주 발생하는 문제

## 29.1 `ImagePullBackOff`

확인:

```bash
kubectl describe pod <pod-name> -n biddinggo
```

원인 후보:

```text
GHCR image tag가 없음
ghcr-secret이 없음
GitHub token에 read:packages 권한이 없음
GHCR package가 private인데 권한이 없음
image 이름이 잘못됨
```

해결:

```bash
kubectl get secret ghcr-secret -n biddinggo
kubectl delete secret ghcr-secret -n biddinggo
```

그 후 다시 생성:

```bash
kubectl create secret docker-registry ghcr-secret \
  --namespace=biddinggo \
  --docker-server=ghcr.io \
  --docker-username="$GITHUB_USERNAME" \
  --docker-password="$GITHUB_TOKEN" \
  --docker-email="$GITHUB_EMAIL"
```

---

## 29.2 Backend Pod가 계속 재시작됨

확인:

```bash
kubectl get pods -n biddinggo
kubectl logs <pod-name> -n biddinggo
kubectl describe pod <pod-name> -n biddinggo
```

원인 후보:

```text
DB 연결 실패
Redis 연결 실패
biddinggo-env-secret 누락
환경 변수 누락
JWT secret 형식 문제
application.yml에서 요구하는 외부 API key 누락
```

---

## 29.3 MariaDB 연결 실패

확인:

```bash
kubectl get svc mariadb-service -n biddinggo
kubectl logs deploy/mariadb-deploy -n biddinggo
```

Backend ConfigMap 확인:

```bash
kubectl describe configmap biddinggo-config -n biddinggo
```

DB 접속 테스트:

```bash
kubectl run mariadb-client \
  -n biddinggo \
  --rm -it \
  --image=mariadb:11 \
  --restart=Never \
  -- mariadb -h mariadb-service -u biddinggo -p
```

비밀번호:

```text
biddinggo1234
```

---

## 29.4 Redis 연결 실패

확인:

```bash
kubectl get svc redis-service -n biddinggo
kubectl logs deploy/redis-deploy -n biddinggo
```

Redis 접속 테스트:

```bash
kubectl run redis-client \
  -n biddinggo \
  --rm -it \
  --image=redis:7-alpine \
  --restart=Never \
  -- redis-cli -h redis-service -a redis1234 ping
```

정상 결과:

```text
PONG
```

---

## 29.5 Argo CD가 변경을 반영하지 않음

확인:

```bash
kubectl get application biddinggo-app -n argocd
kubectl describe application biddinggo-app -n argocd
```

Argo CD는 GitHub repository의 `main` branch를 기준으로 동작한다.
따라서 로컬 manifest만 수정한 상태라면 Argo CD가 변경을 볼 수 없다.

```bash
git status
git push origin main
```

Kubernetes 상태 확인:

```bash
kubectl get deploy -n biddinggo
kubectl get pods -n biddinggo
```

---

## 29.6 Jenkins에서 Docker 명령어 실패

Jenkins 컨테이너 안에서 Docker 접근 확인:

```bash
docker exec -it jenkins docker ps
```

정상적으로 Docker 컨테이너 목록이 보여야 한다.

안 되면 Jenkins 컨테이너를 삭제하고 다시 실행한다.

```bash
docker compose -f jenkins/docker-compose.yml down
docker compose -f jenkins/docker-compose.yml up -d
```

macOS에서는 `jenkins/docker-compose.yml`의 Docker CLI 경로와 `group_add` 설정이 Linux 기준으로 남아 있지 않은지 확인한다.

---

# 30. 전체 명령어 빠른 실행 순서

아래는 핵심 흐름만 모은 버전이다.  
처음 구축할 때는 위 설명을 읽고, 익숙해진 뒤 이 순서만 보면 된다.

```bash
# 1. Docker Desktop Kubernetes 확인
kubectl config use-context docker-desktop
kubectl get nodes

# 2. 작업 디렉토리
mkdir -p ~/develop/biddinggo-cicd
cd ~/develop/biddinggo-cicd

# 3. 레포 clone
git clone https://github.com/jin605/<BACKEND_REPOSITORY>.git
git clone https://github.com/jin605/biddinggo-deploy.git

# 4. namespace
kubectl create namespace biddinggo

# 5. GHCR secret
export GITHUB_USERNAME="<GITHUB_USERNAME>"
export GITHUB_TOKEN="<GITHUB_TOKEN>"
export GITHUB_EMAIL="<GITHUB_EMAIL>"

kubectl create secret docker-registry ghcr-secret \
  --namespace=biddinggo \
  --docker-server=ghcr.io \
  --docker-username="$GITHUB_USERNAME" \
  --docker-password="$GITHUB_TOKEN" \
  --docker-email="$GITHUB_EMAIL"

# 6. ingress-nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443

# 7. cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true

cat <<'EOF' | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  selfSigned: {}
EOF

# 8. MariaDB
kubectl create secret generic mariadb-secret \
  -n biddinggo \
  --from-literal=MARIADB_ROOT_PASSWORD="root1234" \
  --from-literal=MARIADB_DATABASE="biddinggo" \
  --from-literal=MARIADB_USER="biddinggo" \
  --from-literal=MARIADB_PASSWORD="biddinggo1234"

# 9. Redis
kubectl create secret generic redis-secret \
  -n biddinggo \
  --from-literal=REDIS_PASSWORD="redis1234"

# 10. Backend Secret
kubectl create secret generic biddinggo-env-secret \
  -n biddinggo \
  --from-literal=DB_PASSWORD="biddinggo1234" \
  --from-literal=REDIS_PASSWORD="redis1234" \
  --from-literal=JWT_SECRET="local-jwt-secret-key-must-be-long-enough-for-test" \
  --from-literal=OPENAI_API_KEY="dummy" \
  --from-literal=GOOGLE_CLIENT_ID="dummy" \
  --from-literal=GOOGLE_CLIENT_SECRET="dummy" \
  --from-literal=KAKAO_CLIENT_ID="dummy" \
  --from-literal=KAKAO_CLIENT_SECRET="dummy" \
  --from-literal=TOSS_SECRET_KEY="dummy" \
  --from-literal=CLOUDFLARE_R2_ACCESS_KEY="dummy" \
  --from-literal=CLOUDFLARE_R2_SECRET_KEY="dummy"

# 11. Manifest 적용
cd ~/develop/biddinggo-cicd/be25-4th-biddingmate-biddinggo

kubectl apply -f infra/k8s/backend/configmap.yaml
kubectl apply -f infra/k8s/backend/deployment.yaml
kubectl apply -f infra/k8s/backend/service.yaml
kubectl apply -f infra/k8s/backend/ingress.yaml

# 12. 확인
kubectl get all -n biddinggo
kubectl rollout status deployment/biddinggo-api-deploy -n biddinggo
kubectl logs -f deployment/biddinggo-api-deploy -n biddinggo
```

---

# 31. 정리

처음부터 로컬에서 BiddingGo CI/CD를 구성할 때 핵심은 다음 순서이다.

```text
1. Docker Desktop Kubernetes 활성화
2. kubectl context를 docker-desktop으로 변경
3. ingress-nginx 설치
4. cert-manager 설치
5. biddinggo namespace 생성
6. GHCR pull secret 생성
7. MariaDB / Redis 배포
8. BiddingGo backend secret 생성
9. Manifest 수동 적용으로 Kubernetes 배포 확인
10. Argo CD 설치
11. Argo CD Application 생성
12. Jenkins 로컬 컨테이너 실행
13. Jenkins credential 등록
14. Jenkins backend pipeline 생성
15. Jenkins build 실행
16. GHCR image push 확인
17. Manifest image tag 변경 확인
18. Argo CD sync 확인
19. Kubernetes rollout 확인
```

이 구조에서 역할은 명확히 나뉜다.

```text
Jenkins:
  애플리케이션 빌드
  Docker image 생성
  GHCR push
  Manifest repository image tag 수정

Argo CD:
  Manifest repository 변경 감지
  Kubernetes에 sync
  Deployment rollout 수행

Kubernetes:
  Pod 생성
  GHCR에서 image pull
  Service / Ingress로 트래픽 연결
```
