### 1️⃣ 경매사이트 DAPANDA

중고 물품 경매를 통해 흥정 없이 간편하게 거래할 수 있는 환경을 제공합니다.

Kubernetes와 AWS EKS를 사용하여 안정적이고 고가용성이 보장되는 배포 환경을 구축하였습니다.

**프로젝트 기간:** 2024.05.29 ~ 2024.07.31

**인원**: 4명

Backend - Django Github : https://github.com/kyw613/dapanda-django

Terraform - Terraform Github : https://github.com/kyw613/dapanda-terraform

***

### **[ Cloud Architecture ]**

![Image](https://github.com/user-attachments/assets/1be29a5e-9240-4290-be7c-1d49d820bb8f)

### **[ EKS Architecture ]**

![Image](https://github.com/user-attachments/assets/7a06fc7b-4c61-4224-852b-518dcefe9780)

***

### 구현 기능

- Next.js를 활용한 웹페이지 구현 및 Django와 Spring Boot를 활용한 Backend 구현
- Amazon EKS와 Istio를 활용한 MSA 환경 구축
- Amazon Aurora MySQL의 CQRS 구성으로 읽기 전용과 쓰기 전용을 분리해 쓰기 성능 향상
- Amazon SQS FIFO를 통해 짧은 시간안에 발생한 대량의 입찰 요청을 순서를 보장
- Amazon SES를 통해 사용자에게 입찰, 낙찰 현황을 이메일을 통해 전달
- Karpenter를 통해 확장성 고려 및 비용 절감
- Terraform을 활용한 인프라 관리와 배포 자동화 - IAM, RBAC, Network Policy, WAF 등을 통한 보안 강화
- AWS CodePipeline과 ArgoCD를 통한 배포 자동화
- Grafana, Loki, Jaeger, Amazon CloudWatch를 사용한 지속적인 모니터링
- Chaos Mesh를 활용한 QA

***
### 📌 구현 기능 상세



### 1. CICD

### [ Flow Chart ]

<img src="https://github.com/user-attachments/assets/df45d838-c75f-4cad-9a8f-dd8d6686a382" width="50%">



### 2. Mornitoring

### [ DashBoard ] 

<img width="980" alt="Image" src="https://github.com/user-attachments/assets/b4b3b095-bd87-41d1-91b8-1b04786a0e2d" />

### [ Slack과 연동 ]

<img width="1073" alt="Image" src="https://github.com/user-attachments/assets/0a6b9c4a-af21-4643-854f-1d3f45b1cb3e" />



### 3. SES로 입찰 및 낙찰

<img width="1075" alt="Image" src="https://github.com/user-attachments/assets/bab8b1bf-b077-4753-a3e0-8804f2dc016e" />
