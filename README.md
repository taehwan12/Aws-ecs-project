# 🚀 AWS ECS 기반 컨테이너 인프라 아키텍처 구축 프로젝트

> **AWS 가상 네트워크 인프라(VPC) 구축부터 Docker 컨테이너라이징, ECR 이미지 관리, 그리고 복합적인 서비스 배포를 위한 ECS(Fargate) 및 ELB(ALB) 연동까지 직접 설계하고 검증한 팀 프로젝트입니다.**

---

## 📄 프로젝트 보고서 원본
* [AWS ECS Project 보고서 (PDF) 보러가기](https://github.com/taehwan12/AWS-ECS-Project/blob/main/aws%20ecs%20project.pdf)

<br>

## 👥 Team Information
- **팀원:** 정태환, 이은석

<br>

## 🛠️ Tech Stack & Environment
- **Cloud Infrastructure:** `AWS (VPC, ECS, ECR, Route 53, ALB)`
- **Containerization:** `Docker`, `Dockerfile`
- **Languages & Tools:** `Shell CLI`, `Ubuntu Server 24.04 LTS`, `Xshell 8`

---

## 📌 주요 수행 내용 (Key Features)

### 1️⃣ AWS 기본 가상 네트워크 인프라(VPC) 구축
* **VPC 인프라 설계:** 전용 네트워크 대역인 `VEC-PRD-VPC`($10.250.0.0/16$)를 직접 생성 및 격리
* **고가용성 서브넷 구성:** 아시아 태평양(서울) 리전의 다중 가용 영역(AZ)을 활용하여 퍼블릭 서브넷 2개 분산 배치
  * `2A 영역`: `VEC-PRD-VPC-NGINX-PUB-2A` ($10.250.1.0/24$, 256 IPs)
  * `2C 영역`: `VEC-PRD-VPC-NGINX-PUB-2C` ($10.250.11.0/24$, 2551 IPs)
* **라우팅 및 외부 통신 확보:** 인터넷 게이트웨이(`VEC-PRD-IGW`)를 VPC에 연결(`Attached`)하고, 라우팅 테이블(`VEC-PRD-RT-PUB`)을 통해 외부 인터넷 트래픽($0.0.0.0/0$)이 두 서브넷으로 정상 소통하도록 명시적 매핑 완료

### 2️⃣ 도메인 매핑 및 가비아 네임서버(Route 53) 연동
* **퍼블릭 호스팅 영역 생성:** AWS Route 53에 구매한 도메인 `rmsid.store`를 등록하고 전용 NS(Name Server) 및 SOA 레코드 자동 생성
* **이기종 도메인 서비스 연동:** 생성된 4개의 AWS 네임서버 주소를 도메인을 구입한 외부 기관(가비아)의 네임서버 체계에 최종 포워딩 등록 완료

### 3️⃣ Docker 환경 구축 및 멀티 버전 이미지 빌드
* **호스트 가상 서버 운용:** `Ubuntu Server 24.04 LTS` 기반의 관리형 EC2 인스턴스(`EKS-Managed-server`)를 배포하고 Xshell을 통해 RSA 키 페어로 안전하게 원격 제어 환경 수립
* **AWS CLI 자격 증명:** IAM 관리자 엑세스 키(`rootkey.csv`)를 활용하여 인스턴스 내부에 `aws configure` 프로그래밍 방식 호출 권한 부여
* **애플리케이션 컨테이너라이징:** Docker 데몬 설치 후, Nginx 기반 커스텀 웹 페이지를 포함하는 멀티 버전 이미지 빌드 및 로컬 컨테이너 테스트 완료
  * `v1 버전`: 어두운 테마 배경 (#333) 웹 서비스 검증
  * `v2 버전`: 블루 테마 배경 웹 서비스 검증

### 4️⃣ Amazon ECR 이미지 관리 및 ECS 서버리스 배포
* **프라이빗 리포지토리 운용:** AWS ECR에 `ecs-nginx` 컨테이너 레지스트리를 개설하고 Docker 클라이언트 인증(`get-login-password`) 진행
* **이미지 푸시:** 로컬에서 검증된 `ecs-nginx:v1` 및 `v2` 빌드 이미지를 ECR 프라이빗 원격 저장소로 push 완료
* **ECS 서버리스 인프라 가동:** `AWS Fargate` 인프라를 사용하는 `VEC-PRD-ECR-Cluster` 클러스터 가동
* **태스크 정의(Task Definition):** Linux/X86_64 기반 하에 고정 자원(.5 vCPU, 1GB Memory) 및 컨테이너 포트(80/TCP)를 타겟팅하는 태스크 세트(`ecs-nginx-task`, `ecs-nginx-container`) 인프라 요구사항 선언

### 5️⃣ ELB(ALB) 로드 밸런싱 및 고가용성 서비스 배포
* **네트워크 가상 방화벽 설계:** 인프라 보호를 위한 레이어별 보안 그룹(SG) 세분화 설정
  * `ALB 보안 그룹`: 외부의 모든 HTTP 트래픽을 허용하는 `ECS-NGINX-ALB-SG` 수립
  * `컨테이너 보안 그룹`: ALB로부터의 접근을 제어하는 인바운드 규칙 `ECS-CONTAINER-SG` 지정
* **Application Load Balancer 연동:** 외부 인터넷 지향 체계인 `ECS-NGINX-ALB`를 다중 가용 영역 서브넷(2A, 2C)에 매핑
* **대상 그룹 및 리스너 규칙 바인딩:** 80번 포트로 수신되는 부하 트래픽을 타겟 대상 그룹(`ECS-NGINX-ALB-TARGET`)으로 전송하는 단순 라우팅 정책 수립
* **최종 서비스 자동화 배포:** ECS 클러스터 내부에서 복제본 스케줄링 전략(원하는 태스크 수: 2개)에 따라 컨테이너를 가동하고, Route 53 빠른 레코드 생성을 통해 `www.rmsid.store` 서브도메인을 ALB DNS 별칭(Alias)으로 묶어 고가용성 웹 서비스 최종 활성화 확인

---

## 📈 배운 점 및 성과 (Retrospective)
- **클라우드 네이티브 아키텍처 이해:** VPC 네트워크 컴포넌트 구성부터 컨테이너 기반 서버리스 아키텍처(Fargate)까지의 유기적인 연동 과정을 직접 수행하며 실무 트래픽 파이프라인의 전체 구조를 이해하게 되었습니다.
- **컨테이너 생태계 및 형상 관리 내재화:** Docker 로컬 빌드 환경과 AWS 가상 레지스트리(ECR) 간의 지속적인 이미지 푸시 및 태그 관리 기법을 마스터했습니다.
- **고가용성 부하 분산 설계:** 다중 AZ 기반 서브넷 설계와 리스너 규칙 연동을 통해 시스템 장애(Single Point of Failure)를 방지하는 고가용성 로드 밸런싱 자동화 능력을 배양했습니다.
