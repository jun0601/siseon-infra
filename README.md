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

## 시나리오

- **AWS 서울**: 본사. 한국 사용자 대상 메인 서비스
- **AWS 오하이오**: 미국 영업팀 대상 서비스 (멀티 리전 확장 예정)
- **온프레미스(한국)**: 센터/창고. 온도 센서 데이터 수집
- **Azure 서울**: 백업 데이터 + 상시 로그 저장 (재해 복구용)

**데이터**: AWS RDS Multi-AZ / 서울(Master) ↔ 오하이오(Slave) 동기화, 미국은 읽기 전용 / Azure에 백업·로그  
**트래픽**: Global Accelerator 지연 기반 라우팅 (한국→서울, 미국→오하이오), 리전 장애 시 페일오버  
**보안**: 미국 영업팀은 최소 권한으로 앱/DB 접근 (본사 영향 차단)

---

## 레포 구성

| 레포 | 담당 | 내용 |
|------|------|------|
| [siseon-infra](https://github.com/jun0601/siseon-infra) | 김진우 | Terraform IaC (modules + seoul) |
| [siseon-security](https://github.com/jun0601/siseon-security) | 이준형 | CloudTrail 보안/감사 모니터링 |
| [siseon-infra-monitoring](https://github.com/jun0601/siseon-infra-monitoring) | 이준형 | EKS 인프라 모니터링 (Prometheus + Grafana) |
| Stockops-Application | 이현수 | 앱 모노레포 + GitHub Actions CI/CD |

---

## 애플리케이션 컴포넌트

| 컴포넌트 | 기술 | 포트 | ALB 경로 |
|----------|------|------|----------|
| client-web | React + nginx | 80 | `/` (default) |
| admin-web | React + nginx | 80 | `/admin` |
| api-server | Spring Boot 3.2.12 / Java 21 | 8080 | `/api` |
| ai-module | FastAPI | 8000 | `/ai` |

---

## 인프라 모듈 구성

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

## 배포 방법

### 사전 준비
- AWS CLI 자격증명 설정 (`aws configure sso --profile siseon`)
- kubectl, terraform 설치
- `terraform.tfvars`에 `db_username`, `db_password`, `jwt_secret` 등 설정

### 1. 인프라 배포

```bash
cd seoul

# (A) 클러스터가 이미 있는 경우 — 한 방에
terraform apply -auto-approve

# (B) 완전 처음(클러스터 없음) — 단계 분리
terraform apply -auto-approve \
  -target=module.seoul_vpc \
  -target=module.seoul_alb \
  -target=module.seoul_eks \
  -target=module.seoul_db \
  -target=module.seoul_ecr

aws eks update-kubeconfig --region ap-northeast-2 --name seoul-cluster --profile siseon
terraform apply -auto-approve
```

> `wait_for_rollout = false` 설정으로 deployment는 이미지가 없어도 apply가 멈추지 않는다.

### 2. Kubernetes Secret 생성

```bash
kubectl create secret generic stockops-secret \
  --from-literal=JWT_SECRET="<랜덤32자이상>" \
  --from-literal=DB_USERNAME="<DB유저>" \
  --from-literal=DB_PASSWORD="<DB비번>" \
  -n stockops
```

### 3. GitHub Actions OIDC 설정

> ⚠️ **배포 전 필수 수정사항**  
> `modules/github-oidc/variables.tf` 에서 아래 값을 본인 레포 정보로 변경해야 합니다.

```hcl
github_org      = "jun0601"           # GitHub 계정명
github_repo     = "Stockops-Application"  # 애플리케이션 레포명
allowed_branches = ["main"]           # 허용할 브랜치
```

이후 Stockops-Application 레포의 GitHub Actions로 이미지를 빌드/푸시:

```bash
# main 브랜치 push 또는 수동 트리거
gh workflow run deploy.yml
```

이미지가 ECR에 올라오면 ImagePullBackOff 상태였던 Pod가 자동으로 다시 pull → Running.

### 4. 검증

```bash
kubectl get pods -n stockops
kubectl get targetgroupbinding -n stockops

# api 헬스체크
kubectl exec -it <api-pod> -n stockops -- curl -s localhost:8080/actuator/health

# ALB DNS 확인
aws elbv2 describe-load-balancers \
  --names seoul-alb \
  --query "LoadBalancers[0].DNSName" \
  --output text
```

### 5. 초기 로그인 계정

앱 기동 시 `AuthDataLoader`가 admin 계정을 자동 시드한다.
- 이메일: `admin@stockops.com`
- 비밀번호: `admin123`

---

## 인프라 모니터링 연동

EKS 배포 완료 후 `siseon-infra-monitoring` 레포를 배포하면 Grafana 대시보드가 자동으로 구성됩니다.

```bash
git clone https://github.com/jun0601/siseon-infra-monitoring.git
cd siseon-infra-monitoring
terraform init && terraform apply
```

---

## 종료 (destroy)

```bash
# TGB 먼저 (LBC 살아있을 때)
terraform destroy -auto-approve \
  -target=kubectl_manifest.client_tgb \
  -target=kubectl_manifest.admin_tgb \
  -target=kubectl_manifest.api_tgb \
  -target=kubectl_manifest.ai_tgb

# 전체
terraform destroy -auto-approve

# destroy가 막히면 TGB 수동 삭제 후 재시도
kubectl delete targetgroupbinding --all -n stockops
```

### destroy 후 잔재 확인

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

남을 수 있는 IAM: `seoul-eks-cluster-role`, `seoul-eks-node-role`, `seoul-lbc-role`, `seoul-lbc-policy`  
정책을 detach 후 role 삭제 필요.

---

## 추가 예정 (로드맵)

- **Route 53 + ACM**: `admin.도메인` / `client.도메인` 호스트 분리
- **멀티 리전**: 오하이오 리전 확장 + ECR replication
- **Secrets Manager**: ESO 연동으로 DB/JWT 시크릿 자동 동기화
- **온프레미스 연동**: Site-to-Site VPN
- **센서 파이프라인**: IoT Core → SQS → 백엔드 분석
- **Global Accelerator**: 멀티 리전 트래픽 라우팅

자세한 아키텍처는 `Architecture.md`, AWS 리소스 목록은 `AWS_resources.md` 참고.