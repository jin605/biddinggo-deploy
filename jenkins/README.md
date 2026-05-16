# Jenkins on NAS Host

Jenkins는 VM k3s가 아니라 NAS host Docker에서 실행한다.

```text
NAS host Docker Jenkins
  -> backend image build/push to GHCR
  -> update biddinggo-deploy manifests
  -> push manifest commit
  -> VM ArgoCD auto sync
```

## Run

NAS host에서 `biddinggo-deploy`를 받은 뒤 실행한다.

```bash
cd ~/biddinggo-deploy/jenkins
docker compose up -d --build
docker logs -f jenkins
```

초기 비밀번호:

```bash
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Docker CLI 확인:

```bash
docker exec -it jenkins docker version
docker exec -it jenkins docker ps
```

접속:

```text
http://192.168.45.115:8081
```

외부 도메인 노출이 필요하면 VM k3s에 `jenkins/ingress.yaml`을 적용한다.

```bash
kubectl apply -f jenkins/ingress.yaml
```

이 ingress는 VM ingress-nginx가 `jenkins.jinddd3.synology.me` 요청을 NAS host의 `192.168.45.115:8081`로 넘기는 구조다.

## Jenkins Credentials

최소로 필요한 credential:

```text
github-token
```

권장 권한:

```text
repo
read:packages
write:packages
```

현재 채팅/파일에 노출된 GitHub PAT는 작업 후 반드시 폐기하고 새 토큰으로 교체한다.
