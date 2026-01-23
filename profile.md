# 🏦 Hybrid Cloud Financial Banking System (Woori-Next-PoC)

## 📖 Project Overview
본 프로젝트는 **금융권 차세대 시스템**을 위한 **Hybrid MSA 아키텍처** 검증(PoC) 모델입니다.
**AWS Public Cloud(채널계)**의 확장성과 **On-Premise(계정계)**의 보안성을 결합하여, 실제 금융 서비스에서 발생하는 트래픽 폭주, 분산 트랜잭션, 네트워크 장애 상황에 대한 기술적 해결책을 제시합니다.

## 🏗 Architecture
![Architecture Diagram](./docs/images/architecture_overview.png)
*(여기에 아키텍처 다이어그램 이미지를 삽입하세요)*

### 1. Zone & Components
| Zone | Role | Infrastructure | Key Tech Stack |
| :--- | :--- | :--- | :--- |
| **☁️ Cloud (AWS)** | 채널/이벤트 | **AWS EKS** (Managed K8s) | Spring Boot, Aurora MySQL, **Redis**, Kafka |
| **🏢 On-Premise** | 코어/원장 | **Ubuntu Metal** (Kubeadm) | Spring Boot, **Oracle DB**, Docker |
| **🌉 Bridge** | 보안 연결 | **Tailscale** (Mesh VPN) | Spring Cloud Gateway, mTLS |

## 🚀 Key Scenarios (Design Patterns)

### 1. 🎟️ Event & Coupon Service (High Traffic)
* **Goal:** 선착순 쿠폰 발급 트래픽 분산
* **Pattern:** **Redis Atomic Counter** & **MQ(Kafka) Buffering**
* **Flow:** Client → API Gateway → Redis(재고 확인) → Kafka(발급 요청) → Async Consumer → DB Save
* **Why:** RDBMS Lock 대기 시간을 제거하여 초당 처리량(TPS) 극대화

### 2. 💳 Composite Payment (Distributed Transaction)
* **Goal:** 포인트(Cloud) + 현금(On-Prem) 복합 결제의 데이터 정합성 보장
* **Pattern:** **Saga Pattern (Choreography)**
* **Flow:**
    1. 포인트 차감 (Cloud) ✅
    2. VPN 통신 ⚡️
    3. 현금 출금 (On-Prem) ❌ **(실패 발생)**
    4. 보상 이벤트 발행 (Rollback Event) 
    5. 포인트 환불 처리 (Compensating Transaction) ✅

### 3. 🔌 Resilience (Network Partition)
* **Goal:** 하이브리드 클라우드 간 VPN 연결 유실 시 서비스 생존
* **Pattern:** **Circuit Breaker** & **Event Sourcing**
* **Flow:** 즉시 실패 처리 대신 요청을 Queue에 적재하고 사용자에게는 "접수됨(Processing)" 응답 반환 후, 네트워크 복구 시 순차 처리.

## 🛠 Tech Stack Details

### Backend (Microservices)
* **Language:** Java 17, Kotlin
* **Framework:** Spring Boot 3.2, Spring Cloud
* **Build Tool:** Gradle

### Infrastructure
* **On-Premise Node:**
    * OS: Ubuntu 22.04 LTS
    * K8s: v1.28 (via Kubeadm)
    * DB: Oracle 19c XE
* **Cloud Node:**
    * Platform: AWS EKS
    * DB: Amazon Aurora MySQL (Serverless v2)

## 🔧 Installation & Setup

### Prerequisites
* AWS CLI configured
* Docker & Kubernetes CLI (kubectl)
* Tailscale Auth Key

### 1. On-Premise Setup (Core)
```bash
# Install K8s Cluster via Kubeadm
cd infra/on-prem/k8s
./install_k8s.sh

# Deploy Oracle & Core Services
kubectl apply -f core-db-statefulset.yaml
kubectl apply -f core-services.yaml
```

### 2. Cloud Setup (Channel)

```bash
# Provision EKS & Aurora (Terraform)
cd infra/cloud/terraform
terraform init && terraform apply

# Deploy Channel Services
kubectl apply -f channel-services.yaml
```

📂 Project Structure

woori-next-poc/
├── 📂 docs/                  # 아키텍처 다이어그램 및 발표 자료
├── 📂 infra/                 # IaC 및 K8s 설정
│   ├── 📂 cloud/             # AWS Terraform, EKS Manifests
│   └── 📂 on-prem/           # Kubeadm scripts, Oracle Pod specs
├── 📂 backend/               # 마이크로서비스 소스코드
│   ├── 📂 channel-api/       # [Cloud] 가입, 조회, Gateway
│   ├── 📂 event-api/         # [Cloud] 쿠폰, Redis 로직
│   ├── 📂 point-api/         # [Cloud] 포인트 관리
│   ├── 📂 core-payment/      # [On-Prem] 계정계 결제
│   └── 📂 core-user/         # [On-Prem] 고객 원장
└── 📜 docker-compose.local.yml # 로컬 테스트용 환경


---

### 3. 정리할 전체 프로젝트 목차 (Directory Structure)

실제 개발 시 폴더 구조는 아래와 같이 잡으시면, MSA 서비스와 인프라 코드가 명확히 분리되어 보여주기 좋습니다.

```text
woori-next-poc/
├── README.md                 # 메인 설명서
├── LICENSE
├── .gitignore
│
├── 📁 docs/                  # 문서화
│   ├── architecture.png      # 전체 구성도
│   ├── sequence_saga.png     # 사가 패턴 흐름도
│   └── seminar_ppt.pdf       # 세미나 발표 자료
│
├── 📁 infra/                 # 인프라 코드 (DevOps)
│   ├── 📁 aws-cloud/         # AWS 관련
│   │   ├── 📁 terraform/     # EKS, RDS, VPC 생성 코드
│   │   └── 📁 k8s-manifests/ # Deployment, Service, HPA yaml
│   │
│   ├── 📁 on-premise/        # 온프레미스 관련
│   │   ├── 📁 vagrant/       # 로컬 VM 테스트용 (옵션)
│   │   ├── 📁 k8s-setup/     # kubeadm 설치 스크립트
│   │   └── 📁 oracle-db/     # Oracle StatefulSet, PV, PVC yaml
│   │
│   └── 📁 network/           # 네트워크 연결
│       └── tailscale-subnet.yaml # VPN 라우팅 설정
│
└── 📁 services/              # 애플리케이션 코드 (Backend)
    ├── 📁 common-library/    # DTO, Util, Exception 공통 모듈
    │
    ├── 📁 cloud-zone/        # AWS 배포용 서비스
    │   ├── 📁 api-gateway/   # Spring Cloud Gateway
    │   ├── 📁 member-svc/    # Channel-User Service
    │   ├── 📁 coupon-svc/    # Event-Coupon Service (Redis)
    │   └── 📁 point-svc/     # Point Service
    │
    └── 📁 core-zone/         # On-Prem 배포용 서비스
        ├── 📁 core-banking/  # Core-Payment + Core-User (모듈형 모놀리스 권장)
        └── 📁 batch-job/     # 정산용 Spring Batch
```
