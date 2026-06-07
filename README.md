# 🛒 StockOps - 식품 ERP 멀티 클라우드 인프라

![Terraform](https://img.shields.io/badge/terraform-%235843U9.svg?style=for-the-badge&logo=terraform&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-%23316192.svg?style=for-the-badge&logo=postgresql&logoColor=white)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/github%20actions-%232088FF.svg?style=for-the-badge&logo=githubactions&logoColor=white)

# StockOps — 멀티 하이브리드 클라우드 인프라

> **팀명**: 시선 (SISEON)  
> **주제**: AX 환경을 위한 ERP 솔루션 기반 멀티 하이브리드 클라우드 인프라 자동화 및 Observability 체계 구축

StockOps는 K-Food 수출 기업을 모델로 한 ERP/WMS 솔루션이다. 서울 본사가 운영을 총괄하고 미국 영업팀이 현지 영업을 지원하며, 멀티 리전·하이브리드 클라우드 위에서 동작한다.

---

## 📋 시나리오

- **AWS 서울**: 본사. 한국 사용자 대상 메인 서비스
- **AWS 오하이오**: 미국 영업팀 대상 서비스 (멀티 리전 확장 예정)
- **온프레미스(한국)**: 센터/창고. 온도 센서 데이터 수집
- **Azure 서울**: 백업 데이터 + 상시 로그 저장 (재해 복구용)

| 구분 | 내용 |
|------|------|
| 데이터 | RDS Multi-AZ / 서울(Master) ↔ 오하이오(Slave), Azure 백업 |
| 트래픽 | Global Accelerator 지연 기반 라우팅, 리전 장애 시 페일오버 |
| 보안 | 미국 영업팀 최소 권한 접근, IAM Identity Center 중앙 관리 |

---

## 👥 팀 구성 및 레포지토리

| 팀원 | 파트 | 레포 |
|------|------|------|
| 김진우 | 클라우드 인프라 & CI/CD | [siseon-infra](https://github.com/jun0601/siseon-infra) (현재) |
| 이준형 | 로그/모니터링 & 보안 | [siseon-security](https://github.com/jun0601/siseon-security) / [siseon-infra-monitoring](https://github.com/jun0601/siseon-infra-monitoring) |
| 김시온 | 데이터베이스 & 재해복구 | - |
| 이현수 | 풀스택 개발 & 온프레미스 | Stockops-Application |

---

## 🏗️ 인프라 모듈 구성

| 모듈 | 역할 |
|------|------|
| `modules/vpc` | Multi-AZ VPC, 퍼블릭/프라이빗/DB 서브넷 3-Tier |
| `modules/alb` | L7 ALB + 타겟 그룹 4개 (경로 기반 라우팅) |
| `modules/eks` | EKS 클러스터 + 노드그룹 + OIDC + LBC IAM |
| `modules/db` | RDS PostgreSQL (프라이빗, 스토리지 자동확장 100GB) |
| `modules/ecr` | ECR 레포 4개 (for_each 자동 생성) |
| `modules/github-oidc` | GitHub Actions OIDC 인증 (Access Key 없음) |
| `modules/iot` | AWS IoT Core + SQS + DLQ (센서 데이터 파이프라인) |

---

## 📦 애플리케이션 컴포넌트

| 컴포넌트 | 기술 | 포트 | ALB 경로 |
|----------|------|------|----------|
| client-web | React + nginx | 80 | `/` (default) |
| admin-web | React + nginx | 80 | `/admin` |
| api-server | Spring Boot 3.2.12 / Java 21 | 8080 | `/api` |
| ai-module | FastAPI | 8000 | `/ai` |

---

## 🚀 배포 방법

### 사전 준비

```bash
aws sso login --profile siseon
```

`terraform.tfvars` 에 아래 값 설정:
```hcl
db_username = "<DB유저>"
db_password = "<DB비번>"
jwt_secret  = "<랜덤32자이상>"
```

### 1단계: 기본 인프라 배포 (EKS 없을 때)

```bash
cd seoul

terraform init

terraform apply -auto-approve \
  -target=module.seoul_vpc \
  -target=module.seoul_alb \
  -target=module.seoul_eks \
  -target=module.seoul_db \
  -target=module.seoul_ecr
```

> EKS 없을 때 한 번에 apply하면 kubernetes provider 연결 오류 발생.  
> 반드시 단계 분리 후 kubeconfig 업데이트 필요.

### 2단계: kubeconfig 업데이트

```bash
aws eks update-kubeconfig --region ap-northeast-2 --name seoul-cluster --profile siseon
```

### 3단계: 전체 배포

```bash
terraform apply -auto-approve
```

### 4단계: Kubernetes Secret 생성

```bash
kubectl create secret generic stockops-secret \
  --from-literal=JWT_SECRET="<랜덤32자이상>" \
  --from-literal=DB_USERNAME="<DB유저>" \
  --from-literal=DB_PASSWORD="<DB비번>" \
  -n stockops
```

### 5단계: GitHub Actions OIDC 설정

`modules/github-oidc/variables.tf` 에서 레포 정보 수정:

```hcl
github_org       = "jinuuuKim"
github_repo      = "Stockops-Application"
allowed_branches = ["main"]
```

이후 Stockops-Application 레포에서 GitHub Actions 트리거 → ECR 이미지 푸시 → Pod 자동 실행

### 6단계: 인프라 모니터링 연동

```bash
git clone https://github.com/jun0601/siseon-infra-monitoring.git
cd siseon-infra-monitoring
terraform init && terraform apply
```

### 7단계: 검증

```bash
# Pod 상태 확인
kubectl get pods -n stockops

# ALB DNS 확인
aws elbv2 describe-load-balancers \
  --names seoul-alb \
  --query "LoadBalancers[0].DNSName" \
  --output text

# API 헬스체크
kubectl exec -it <api-pod> -n stockops -- curl -s localhost:8080/actuator/health
```

### 초기 로그인 계정

| 이메일 | 비밀번호 |
|--------|---------|
| `admin@stockops.com` | `admin123` |

---

## 🗑️ 종료 (Destroy)

### 1단계: 모니터링 스택 먼저 삭제

```bash
cd siseon-infra-monitoring
helm uninstall kube-prometheus-stack -n monitoring
terraform destroy -auto-approve
```

### 2단계: TGB 먼저 삭제 (LBC 살아있을 때)

```bash
cd siseon-infra/seoul

kubectl delete targetgroupbinding --all -n stockops

terraform destroy -auto-approve \
  -target=kubectl_manifest.client_tgb \
  -target=kubectl_manifest.admin_tgb \
  -target=kubectl_manifest.api_tgb \
  -target=kubectl_manifest.ai_tgb
```

### 3단계: 전체 삭제

```bash
terraform destroy -auto-approve
```

### 잔재 확인 (중요)

```bash
# IAM Role 확인
aws iam list-roles \
  --query "Roles[?contains(RoleName, 'seoul')].RoleName" \
  --output table

# 과금 리소스 확인
aws ec2 describe-nat-gateways \
  --filter "Name=state,Values=available" \
  --query "NatGateways[*].NatGatewayId" \
  --output table

aws rds describe-db-instances \
  --query "DBInstances[*].DBInstanceIdentifier" \
  --output table
```

> 남을 수 있는 IAM: `seoul-eks-cluster-role`, `seoul-eks-node-role`, `seoul-lbc-role`, `seoul-lbc-policy`  
> 정책 detach 후 role 삭제 필요.

---

## 📌 로드맵

- **Route 53 + ACM**: `admin.도메인` / `client.도메인` 호스트 분리
- **멀티 리전**: 오하이오 리전 확장 + ECR replication
- **Secrets Manager**: ESO 연동으로 DB/JWT 시크릿 자동 동기화
- **온프레미스 연동**: Site-to-Site VPN
- **센서 파이프라인**: IoT Core → SQS → 백엔드 분석
- **Global Accelerator**: 멀티 리전 트래픽 라우팅