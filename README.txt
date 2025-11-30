1. 로컬 Kubernetes 배포
1.1 Docker Desktop Kubernetes 활성화
bash# Docker Desktop 설정 > Kubernetes > Enable Kubernetes 체크
# Apply & Restart
1.2 배포 스크립트 실행
bashchmod +x deploy.sh
./deploy.sh
1.3 수동 배포 (선택사항)
bash# 1. 이미지 빌드
docker build -t simple-board-backend:latest ./backend
docker build -t simple-board-frontend:latest ./frontend

# 2. Namespace 생성
kubectl create namespace monitoring

# 3. 애플리케이션 배포
kubectl apply -f k8s/backend-deployment.yaml
kubectl apply -f k8s/frontend-deployment.yaml

# 4. 모니터링 배포
kubectl apply -f monitoring/prometheus-config.yaml
kubectl apply -f monitoring/grafana-deployment.yaml

# 5. 배포 확인
kubectl get all -n default
kubectl get all -n monitoring
2. Helm을 사용한 배포
bash# Helm Chart 패키징
helm package helm/

# Helm으로 설치
helm install simple-board ./simple-board-1.0.0.tgz

# 설치 확인
helm list
kubectl get all

# 업그레이드
helm upgrade simple-board ./simple-board-1.0.0.tgz

# 삭제
helm uninstall simple-board
3. CI/CD 파이프라인 설정
3.1 GitHub Secrets 설정
GitHub 저장소 > Settings > Secrets and variables > Actions에서 다음 설정:

DOCKERHUB_USERNAME: Docker Hub 사용자명
DOCKERHUB_TOKEN: Docker Hub Access Token
GITOPS_REPO: GitOps 저장소 경로 (예: username/simple-board-gitops)
GITOPS_TOKEN: GitHub Personal Access Token (repo 권한)

3.2 GitOps 저장소 구조
simple-board-gitops/
├── k8s/
│   ├── backend-deployment.yaml
│   └── frontend-deployment.yaml
└── README.md
3.3 GitHub Actions 워크플로
.github/workflows/ci-cd.yaml 파일이 자동으로 다음을 수행:

main 브랜치 push 시 트리거
Docker 이미지 빌드 및 Docker Hub에 푸시
GitOps 저장소의 매니페스트 업데이트
ArgoCD가 자동으로 변경사항 감지 및 배포

4. ArgoCD 설정
4.1 ArgoCD 설치
bash# Namespace 생성
kubectl create namespace argocd

# ArgoCD 설치
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ArgoCD UI 접속 (Port Forward)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 초기 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
4.2 ArgoCD Application 생성
bash# GitOps 저장소 URL 수정 후 적용
kubectl apply -f argocd/argocd-application.yaml

# 또는 ArgoCD CLI 사용
argocd app create simple-board \
  --repo https://github.com/yourusername/simple-board-gitops \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated
4.3 ArgoCD UI 접속

브라우저에서 https://localhost:8080 접속
Username: admin
Password: 위에서 확인한 초기 비밀번호
Applications에서 simple-board 확인

5. 모니터링 설정
5.1 Prometheus 접속
bash# 브라우저에서 접속
http://localhost:30090

# 메트릭 확인
simple_app_requests_total
rate(simple_app_requests_total[5m])
5.2 Grafana 접속
bash# 브라우저에서 접속
http://localhost:30030

# 로그인
Username: admin
Password: admin123

# 대시보드 확인
Dashboards > Simple Board Metrics
5.3 주요 메트릭

simple_app_requests_total: 총 HTTP 요청 수
rate(simple_app_requests_total[5m]): 5분간 요청 비율
container_cpu_usage_seconds_total: CPU 사용량
container_memory_usage_bytes: 메모리 사용량

6. 서비스 접속

Frontend: http://localhost:30080
Backend API: http://localhost:8080/api/posts
Prometheus: http://localhost:30090
Grafana: http://localhost:30030
ArgoCD: https://localhost:8080 (port-forward 후)

7. 트러블슈팅
Pod가 시작되지 않는 경우
bash# Pod 상태 확인
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>

# 이벤트 확인
kubectl get events --sort-by=.metadata.creationTimestamp
이미지를 찾을 수 없는 경우
bash# 이미지 확인
docker images | grep simple-board

# 수동으로 이미지 로드 (minikube 사용 시)
minikube image load simple-board-backend:latest
minikube image load simple-board-frontend:latest
서비스에 접근할 수 없는 경우
bash# 서비스 확인
kubectl get svc

# Port Forward 사용
kubectl port-forward svc/frontend-service 3000:80
kubectl port-forward svc/backend-service 5000:5000
8. CI/CD 워크플로 테스트
bash# 1. 코드 변경
echo "# Test" >> README.md

# 2. Commit & Push
git add .
git commit -m "Test CI/CD pipeline"
git push origin main

# 3. GitHub Actions 확인
# GitHub 저장소 > Actions 탭에서 워크플로 실행 확인

# 4. Docker Hub 확인
# 새 이미지가 푸시되었는지 확인

# 5. ArgoCD 확인
# ArgoCD UI에서 자동 동기화 확인
9. 정리
bash# Helm 배포 삭제
helm uninstall simple-board

# Kubernetes 리소스 삭제
kubectl delete -f k8s/
kubectl delete -f monitoring/
kubectl delete namespace monitoring

# ArgoCD 삭제
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd