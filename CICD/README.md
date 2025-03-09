## 🚀 AWS CI/CD 구축: CodeCommit, CodeBuild, CodePipeline, ECR, ArgoCD를 활용한 GitOps

AWS에서 **CodeCommit, CodeBuild, CodePipeline, ECR, ArgoCD**를 활용하여 CI/CD 파이프라인을 구축하는 방법을 정리합니다. 

목표는 **GitOps 방식으로 애플리케이션을 배포 및 관리하는 환경을 구성하는 것**입니다.



## 🎯 목표

1. **CodeCommit 레포지토리 생성 및 코드 업로드**
2. **CodeBuild를 사용하여 Docker 이미지를 빌드하고 ECR에 푸시**
3. **CodeCommit의 manifest 파일에 최신 이미지 태그 반영**
4. **ArgoCD가 CodeCommit을 모니터링하여 자동 배포 진행**

## **✨** 최종 결과

✔ **CodeCommit → CodeBuild → ECR → CodeCommit(manifest) → ArgoCD → Kubernetes배포**

✔ **완전한 GitOps 기반의 CI/CD 구축**

✔ **코드 변경이 자동으로 컨테이너 빌드 및 배포로 연결되는 파이프라인 완성**



## [ CICD Architecture ]

<img src="https://github.com/user-attachments/assets/df45d838-c75f-4cad-9a8f-dd8d6686a382" width="50%">



## 🏗️ 1. CodeCommit 레포지토리 생성 및 코드 업로드

### 🔹 **CodeCommit에 레포지토리 생성**

AWS CodeCommit에서 새로운 리포지토리를 생성합니다.

1. AWS 콘솔에서 CodeCommit 서비스로 이동
2. 새로운 리포지토리(`dpd-django-be`) 생성

<img width="703" alt="Image" src="https://github.com/user-attachments/assets/0b59e52d-f836-4d7f-8906-ea62c79df2dd" />

### 🔹 **로컬에서 Git 설정 후 코드 푸시**

기존의 GitHub 프로젝트를 CodeCommit으로 이동합니다.

```bash
# CodeCommit을 원격 저장소로 추가
git remote add codecommit https://git-codecommit.ap-northeast-2.amazonaws.com/v1/repos/dpd-django-be

# CodeCommit 자격 증명 설정
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true

# 코드 푸시
git add .
git commit -m "Initial commit"
git push codecommit main

```

- dpd-manifest-repo에 CD도구인 ArgoCD가 변화를 감지할 yaml 넣어주기

![Image](https://github.com/user-attachments/assets/fb15a5c2-78b2-4e9f-bfce-5568b3bf3971)



## 🏗️ 2. CodeBuild를 사용하여 Docker 이미지 생성 및 ECR에 푸시

### 🔹 **ECR (Elastic Container Registry) 생성**

1. AWS ECR에서 **Private Repository** 생성 (`dpd-django-be`)
2. IAM 역할 설정: CodeBuild가 ECR에 접근할 수 있도록 `AmazonEC2ContainerRegistryFullAccess` 권한 추가

![Image](https://github.com/user-attachments/assets/3103e7fe-ff80-47fa-8725-41d994a8b680)

### 🔹 **CodeBuild 프로젝트 생성**

1. CodeBuild에서 새로운 프로젝트 생성
2. 소스: CodeCommit (`dpd-django-be`) 연결
3. 빌드 사양: `buildspec.yml` 사용
4. IAM 역할에 다음과 같은 정책 추가
    - `AmazonEC2ContainerRegistryFullAccess`
    - `AWSCodeCommitFullAccess`
    - `AmazonSSMFullAccess`

### 🔹 **buildspec.yml 작성**

CodeBuild가 실행할 빌드 스크립트를 정의합니다.

| • **`build` 단계:** Docker 이미지를 빌드하고, ECR에 새로운 태그로 푸시한 후, manifest 저장소를 클론하여 최신 이미지 태그를 생성했다.
• **`post_build` 단계:** manifest 파일의 이미지 태그를 업데이트하고, 변경 사항을 Git에 커밋 및 푸시하여 ArgoCD가 자동으로 배포하도록 했다. |
| --- |

```yaml
version: 0.2

env:
  parameter-store:
    AWS_REGION: "/dapanda/AWS_REGION"
    ECR_REGISTRY: "/dapanda/django/ECR_REGISTRY"
    IMAGE_REPO_NAME: "/dapanda/django/IMAGE_REPO_NAME"
    REPO_URL: "/dapanda/django/REPO_URL"

phases:
  install:
    commands:
      - echo "Installing required dependencies..."
      - apt-get update && apt-get install -y git jq
      - git config --global credential.helper '!aws codecommit credential-helper $@'
      - git config --global credential.UseHttpPath true

  pre_build:
    commands:
      - echo "Logging into Amazon ECR..."
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY

  build:
    commands:
      - echo "Cloning manifest repository..."
      - git clone $REPO_URL
      - cd dpd-manifest-repo
      - git checkout main
      - CURRENT_TAG=$(grep -oP 'image: .*:\K([0-9]+)' django-backend.yaml)
      - NEW_TAG=$((CURRENT_TAG + 1))
      - IMAGE_TAG="$ECR_REGISTRY/$IMAGE_REPO_NAME:$NEW_TAG"
      - cd ..
      - echo "Building and tagging Docker image..."
      - docker build -t $IMAGE_REPO_NAME:$NEW_TAG -f Dockerfile .
      - docker tag $IMAGE_REPO_NAME:$NEW_TAG $ECR_REGISTRY/$IMAGE_REPO_NAME:$NEW_TAG
      - docker tag $IMAGE_REPO_NAME:$NEW_TAG $ECR_REGISTRY/$IMAGE_REPO_NAME:latest
      - echo "Pushing Docker image..."
      - docker push $ECR_REGISTRY/$IMAGE_REPO_NAME:$NEW_TAG
      - docker push $ECR_REGISTRY/$IMAGE_REPO_NAME:latest

  post_build:
    commands:
      - cd dpd-manifest-repo
      - git config user.email "your-email@example.com"
      - git config user.name "your-git-username"
      - echo "Updating image tag in manifest file..."
      - sed -i "s|image: .*|image: $IMAGE_TAG|g" django-backend.yaml
      - git add django-backend.yaml
      - git commit -m "Update image version to $NEW_TAG"
      - git push origin main
      - echo "Manifest updated successfully!"

```

### 🛠️ CodeBuild 역할 설정 및 환경 변수 등록

### 🔹 **CodeBuild 역할에 필요한 권한 추가**

1. **IAM 콘솔 이동** → `역할(Role)` 클릭
2. CodeBuild 생성 후 **해당 역할 선택**
3. **아래 권한 추가**

![Image](https://github.com/user-attachments/assets/d0c0d69e-a2ba-4328-9808-f6b673a8afc7)

### 🔹 **AWS SSM Parameter Store에 환경 변수 등록**

CodeBuild에서 사용할 주요 환경 변수를 AWS Systems Manager (SSM)에 저장합니다.

![Image](https://github.com/user-attachments/assets/16008507-de98-458e-a486-182cec854c4d)



## 🏗️ 3. CodePipeline을 활용한 자동화

### 🔹 **CodePipeline 생성**

1. CodeCommit (`dpd-django-be`) → CodeBuild → CodeCommit (`dpd-manifest-repo`) 업데이트
2. 빌드 완료 후 **ArgoCD가 자동으로 최신 이미지를 배포**하도록 설정

![Image](https://github.com/user-attachments/assets/5dd72556-14bd-4ecf-9297-a5e975061d72)

![Image](https://github.com/user-attachments/assets/614a339f-2c94-42e1-a47a-e375bff6dd7b)

![Image](https://github.com/user-attachments/assets/571e77f3-7f78-4db3-92bb-5178ce80ef70)

![Image](https://github.com/user-attachments/assets/18c0eeb7-a3e5-4924-b4d1-22298861e1eb)

### 🔹이후 codecommit repo에 내용 변경시 동작 양호

![Image](https://github.com/user-attachments/assets/671b52ab-37da-4f78-94ad-94f005acdef7)

![Image](https://github.com/user-attachments/assets/ef9c140c-f86c-4f6d-b5b6-6418218c3a98)



## 🏗️ 4. ArgoCD를 이용한 GitOps 배포

### 🔹 **ArgoCD 설치**

```c
sudo snap install helm --classic
helm repo add stable https://charts.helm.sh/stable

kubectl create ns argocd

helm repo add argo https://argoproj.github.io/argo-helm -n argocd
===
"argo" has been added to your repositories
===

helm repo list
===
NAME    URL
argo    https://argoproj.github.io/argo-helm
k8s-master@dapanda-k8s-master:~$ helm search repo argo
NAME                            CHART VERSION   APP VERSION     DESCRIPTION
argo/argo                       1.0.0           v2.12.5         A Helm chart for Argo Workflows
argo/argo-cd                    7.1.3           v2.11.3         A Helm chart for Argo CD, a declarative, GitOps...
argo/argo-ci                    1.0.0           v1.0.0-alpha2   A Helm chart for Argo-CI
===

helm pull argo/argo-cd
tar xvzf argo-cd-7.1.3.tgz
rm argo-cd-7.1.3.tgz 
cd argo-cd/
cp values.yaml my-values.yaml
vi my-values.yaml

2048   service:
2049     # -- Server service annotations
2050     annotations: {}
2051     # -- Server service labels
2052     labels: {}
2053     # -- Server service type
2054     type: NodePort
2055     # -- Server service http port for NodePort service type (only if `server.service.type` is set to "NodePort")
2056     nodePortHttp: 32080
2057     # -- Server service https port for NodePort service type (only if `server.service.type` is set to "NodePort")
2058     nodePortHttps: 32443
2059     # -- Server service http port
2060     servicePortHttp: 80
2061     # -- Server service https port
2062     servicePortHttps: 443

 helm install argocd -n argocd -f my-values.yaml .
 
 kubectl get svc -n argocd -o wide
 
 #초기 비밀번호
 kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

## 🔹 **PuTTY를 활용한 Tunneling (로컬에서 ArgoCD 접속하기)**

AWS 환경에서 ArgoCD 서버가 Private Subnet에 위치해 있을 경우, **PuTTY를 이용해 포트 포워딩을 설정하여 로컬에서 접근**할 수 있습니다.

### **1️⃣ `.pem` 파일을 `.ppk`로 변환 (PuTTY 사용 시)**

1. **PuTTYgen 실행 → Load 클릭 → `.pem` 파일 선택**
2. **Save private key** 버튼을 눌러 `.ppk` 형식으로 저장

![Image](https://github.com/user-attachments/assets/87d4d114-4fd0-4199-b8f5-b4be07a0c55e)

![Image](https://github.com/user-attachments/assets/11983528-143b-481c-8eb2-1350ea21d919)

- ppk키 넣어주기

![Image](https://github.com/user-attachments/assets/a483ccba-49ce-495b-9261-14c7e1e1d4ee)

![Image](https://github.com/user-attachments/assets/06ca9bb2-85cc-439a-90f6-0645ad1049c7)

```c
kubectl -n argocd port-forward svc/argocd-server 8080:80
```

✅ **이제 로컬에서 `http://localhost:8080`으로 ArgoCD 접속 가능!**

![Image](https://github.com/user-attachments/assets/00fb9182-4071-42b6-ac10-1ec965702849)

## 🔹 **Private Subnet에서 ArgoCD가 CodeCommit 감시하도록 설정**

ArgoCD가 AWS CodeCommit을 모니터링할 수 있도록 **IAM User 권한을 설정**해야 합니다.

![Image](https://github.com/user-attachments/assets/2870291c-3366-4c14-b94e-1c2023b54afb)

![Image](https://github.com/user-attachments/assets/5aa95c47-e6e1-4dc8-984e-3879ddf74e7d)

- 보안자격증명 생성

![Image](https://github.com/user-attachments/assets/60add49c-e630-4c18-b4bb-234a9d49588c)

- ArgoCD Repository 설정

<img width="700" alt="Image" src="https://github.com/user-attachments/assets/aecbd60b-8f45-42ad-b944-adfe0314a34e" />

- ArgoCD가 **CodeCommit의 `dpd-manifest-repo`를 감시하여 자동 배포**할 수 있도록 애플리케이션을 생성합니다.

<img width="714" alt="Image" src="https://github.com/user-attachments/assets/27dae9be-b42a-42b0-86f5-1873448ad225" />

### 🔹 **ArgoCD가 배포 자동화**

ArgoCD가 `dpd-manifest-repo`를 감시하면서 `django-backend.yaml`이 변경될 때마다 자동으로 배포합니다.
