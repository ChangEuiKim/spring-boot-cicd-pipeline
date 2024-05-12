# GitHub Actions로 CI/CD 파이프라인 설정하기

이 파이프라인은 변경 사항이 메인 브랜치에 푸시될 때마다 애플리케이션을 빌드, 테스트 및 AWS EC2 인스턴스에 배포하는 프로세스를 자동화합니다.

## CI/CD 워크플로 개요

지속적 통합(CI)과 지속적 배포(CD)는 최신 소프트웨어 개발 관행의 중요한 구성 요소입니다. CI는 코드 변경 사항을 자동으로 빌드하고 테스트하는 데 중점을 두며, CD는 이러한 변경 사항이 프로덕션 환경에 빠르고 안전하게 배포되도록 보장합니다.

### 지속적 통합 (CI):

1. 리포지토리 코드를 체크아웃합니다.
2. JDK 환경을 설정합니다.
3. 프로젝트를 빌드합니다.
4. Docker 이미지를 빌드하고 푸시합니다.

### 지속적 배포:

1. GitHub Actions 러너의 공인 IP를 검색합니다.
2. AWS 자격 증명을 구성합니다.
3. AWS 보안 그룹을 업데이트합니다.
4. Docker 컨테이너를 AWS EC2 인스턴스에 배포합니다.
5. 보안 그룹을 정리합니다.

## CI/CD 파이프라인 설정

GitHub Actions는 저장소의 루트 디렉토리에 `.github/workflows` 디렉토리가 있을 경우, 해당 디렉토리에 있는 YAML 파일을 읽어 CI/CD 파이프라인을 설정합니다.

파이프라인을 구성하고자하는 프로젝트가 있는 로컬 저장소의 루트 디렉토리에 `.github/workflows` 디렉토리를 생성하고, `spring-boot-ci-cd-pipeline.yml` 파일을 생성합니다. 워크 플로우로우가 반영된 프로젝트의 변경 사항을 커밋하고 푸시하면 GitHub Actions가 파이프라인을 실행합니다. 워크플로우 파일의 내용은 다음과 같습니다.

```yaml
name: Spring Boot CI/CD Pipeline

# STEP1: Main Branch에 push 또는 pull request가 발생할 때 실행한다.
on:
  push:
    branches: ['main']
  pull_request:
    branches: ['main']

jobs:
  # STEP2: SpringBoot_CI_CD_Pipeline이라는 Job을 실행한다.
  SpringBoot_CI_CD_Pipeline:
    # 최신 버전의 Ubuntu에서 실행한다.
    runs-on: ubuntu-latest

    steps:
      # STEP3: CI (1) 공식 Checkout Action을 사용해 저장소의 코드를 가져온다.
      - name: Checkout Repository
        uses: actions/checkout@v4

      # STEP4: CI (2) 공식 Setup Java Action을 사용해 Amazon Corretto 17을 설치한다.
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: 'corretto'
          java-version: '17'

      # STEP5: CI (3) Gradle Wrapper를 실행하기 위해 실행 권한을 부여한다.
      - name: Run chmod to make gradlew executable
        run: chmod +x ./gradlew

      # STEP6: CI (4) Gradle Wrapper를 사용해 Spring Boot 프로젝트를 빌드한다.
      - name: Build Spring Boot Project with Gradle Wrapper
        run: ./gradlew clean build

      # STEP7: CI (5) Dockerfile을 사용해 Docker 이미지를 빌드한다.
      - name: Build Docker Image
        run: docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/github-actions-demo .

      # STEP8: CI (6) 공식 Docker Login Action을 사용해 Docker Hub에 로그인한다.
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # STEP9: CI (7) Docker 이미지를 Docker Hub에 푸시한다.
      - name: docker Hub push
        run: docker push ${{ secrets.DOCKERHUB_USERNAME }}/github-actions-demo

      # STEP10: CD (1) haythem의 Public IP Action을 사용해 GitHub Actions Runner의 Public IP를 steps.ip.outputs.ipv4로 가져온다.
      - name: get Runner's Public IP
        id: ip
        uses: haythem/public-ip@v1.3

      # STEP11: CD (2) 공식의 "Configure AWS Credentials" Action을 사용해 IAM 계정의 AWS Credentials를 설정한다.
      - name: Configure AWS IAM credentials in repository secrets
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-2

      # STEP12: CD (3) AWS EC2의 보안그룹의 TCP 22번 포트에 GitHub Actions Runner의 Public IP를 추가한다.
      - name: Add GitHub IP to AWS
        run: |
          aws ec2 authorize-security-group-ingress \
          --group-id ${{ secrets.AWS_SG_ID }} \
          --protocol tcp --port 22 \
          --cidr ${{ steps.ip.outputs.ipv4 }}/32

      # STEP13: CD (4) appleboy의 SSH Remote Commands Action을 사용해 AWS EC2에 접속해 Docker 컨테이너를 실행한다.
      - name: Executing remote ssh commands using password with AWS EC2
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USERNAME }}
          password: ${{ secrets.EC2_PASSWORD }}
          port: ${{ secrets.EC2_SSH_PORT }}
          timeout: 60s
          script: |
            sudo docker stop github-actions-demo \
            && sudo docker rm github-actions-demo \
            && sudo docker rmi ${{ secrets.DOCKERHUB_USERNAME }}/github-actions-demo \
            || true
            sudo docker run -it -d -p 8080:8080 \
            --name github-actions-demo ${{ secrets.DOCKERHUB_USERNAME }}/github-actions-demo

      # STEP14: CD (5) AWS EC2의 보안그룹의 TCP 22번 포트에 GitHub Actions Runner의 Public IP를 삭제한다.
      - name: Remove IP FROM security group
        run: |
          aws ec2 revoke-security-group-ingress \
          --group-id ${{ secrets.AWS_SG_ID }} \
          --protocol tcp --port 22 \
          --cidr ${{ steps.ip.outputs.ipv4 }}/32
```

### STEP1: 워크플로우 트리거 설정하기

```yaml
on:
  push:
    branches: ['main']
  pull_request:
    branches: ['main']
```

- `on` 키워드는 워크플로우가 실행되는 이벤트를 정의합니다.
- `push` 이벤트는 코드가 푸시될 때 실행됩니다.
- `pull_request` 이벤트는 풀 리퀘스트가 생성될 때 실행됩니다.
- `branches` 키워드는 워크플로우가 실행되는 브랜치를 지정합니다.
- `main` 브랜치에 푸시 또는 풀 리퀘스트가 발생할 때 워크플로우가 실행됩니다.

### STEP2: Job 설정하기

```yaml
jobs:
  SpringBoot_CI_CD_Pipeline:
    runs-on: ubuntu-latest
```

- `ubuntu-latest`를 Runner로 사용하는 SpringBoot_CI_CD_Pipeline 작업을 정의합니다.
- `jobs` 키워드는 워크플로우의 작업들을 정의합니다. 각 작업은 병렬로 실행됩니다.
- `SpringBoot_CI_CD_Pipeline`은 작업의 이름을 정의합니다. 다른 작업에 대한 의존성을 설정할 때 `needs` 키워드를 사용할 수 있습니다.
- `runs-on` 키워드는 작업이 실행되는 Runner를 정의합니다.

### STEP3: 코드 체크아웃하기

```yaml
- name: Checkout Repository
  uses: actions/checkout@v4
```

- 공식 `actions/checkout@v4` 액션을 사용하여 저장소의 코드를 가져옵니다.
- `name` 키워드는 작업의 이름을 정의합니다.
- `uses` 키워드는 사용할 액션을 지정합니다.

### STEP4: JDK 환경 설정하기

```yaml
- name: Set up JDK 17
  uses: actions/setup-java@v4
  with:
    distribution: 'corretto'
    java-version: '17'
```

- 공식 `actions/setup-java@v4` 액션을 사용하여 Amazon Corretto 17을 설치합니다.
- `with` 키워드는 액션의 입력 매개변수를 정의합니다.
- `distribution` 키워드는 사용할 JDK 배포판을 지정합니다.
- `java-version` 키워드는 사용할 JDK 버전을 지정합니다.

### STEP5: Gradle Wrapper 실행 권한 부여하기

```yaml
- name: Run chmod to make gradlew executable
  run: chmod +x ./gradlew
```

- `chmod +x ./gradlew`는 Gradle Wrapper를 실행할 수 있도록 실행 권한을 부여합니다.
- 리눅스에서 실행 권한이 없는 파일은 실행할 수 없습니다.
- `run` 키워드는 쉘 명령을 실행합니다.
- Gradle Wrapper는 Gradle을 설치하지 않고도 Gradle 프로젝트를 빌드할 수 있는 스크립트입니다.

### STEP6: Spring Boot 프로젝트 빌드하기

```yaml
- name: Build Spring Boot Project with Gradle Wrapper
  run: ./gradlew clean build
```

- `./gradlew clean build`는 Gradle Wrapper를 사용하여 Spring Boot 프로젝트를 빌드합니다.
- 빌드된 JAR 파일은 Docker 이미지를 빌드할 때 사용됩니다.
- `clean`은 이전 빌드 결과를 삭제합니다.
- 빌드 결과는 `build/libs` 디렉토리에 생성됩니다.

### STEP7: Docker 이미지 빌드하기

```yaml
- name: Build Docker Image
  run: docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/github-actions-demo .
```

- `docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/github-actions-demo .`는 Docker 이미지를 빌드합니다.
- 빌드된 Docker 이미지는 Docker Hub에 푸시됩니다.
- `${{ secrets.DOCKERHUB_USERNAME }}`는 GitHub에 저장된 DOCKERHUB_USERNAME을 참조합니다.
- Docker Hub의 username은 우측 상단의 프로필 이미지를 클릭하고 My Account > Account Settings에서 확인할 수 있습니다.
- secrets는 GitHub 저장소의 설정에서 관리할 수 있는 암호화된 환경 변수입니다.
- Repository secrets는 저장소의 Settings 탭에 들어가서 좌측 Security > Secrets and variables > Actions에서 관리할 수 있습니다.

### STEP8: Docker Hub에 로그인하기

```yaml
- name: Login to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
```

- `docker/login-action@v3` 액션을 사용하여 Docker Hub에 로그인합니다.
- Docker Hub의 레지스트리에 이미지를 푸시하려면 로그인이 필요합니다.
- `${{ secrets.DOCKERHUB_TOKEN }}`은 GitHub에 저장된 DOCKERHUB_TOKEN을 참조합니다.
- Docker Hub의 Access Token은 우측 상단의 프로필 이미지를 클릭하고 Account Settings > Security > New Access Token에서 생성할 수 있습니다.
- GitHub의 secrets에 저장된 환경 변수는 DOCKERHUB_USERNAME, DOCKERHUB_TOKEN으로 총 2개입니다.

### STEP9: Docker 이미지 푸시하기

```yaml
- name: docker Hub push
  run: docker push ${{ secrets.DOCKERHUB_USERNAME }}/github-actions-demo
```

- `docker push ${{ secrets.DOCKERHUB_USERNAME }}/github-actions-demo`는 Docker 이미지를 Docker Hub의 레지스트리에 푸시합니다.
- Docker 이미지는 AWS EC2 인스턴스에서 Docker 컨테이너를 생성할 때 사용됩니다.
- GitHub의 secrets에 저장된 환경 변수는 DOCKERHUB_USERNAME, DOCKERHUB_TOKEN으로 총 2개입니다.

### STEP10: GitHub Actions Runner의 Public IP 가져오기

```yaml
- name: get Runner's Public IP
  id: ip
  uses: haythem/public-ip@v1.3
```

- `haythem/public-ip@v1.3` 액션을 사용하여 GitHub Actions Runner의 공인 IP를 가져옵니다.
- Runner의 공인 IP는 `steps.ip.outputs.ipv4`로 참조할 수 있습니다.
- 공인 IP는 AWS EC2 인스턴스의 보안 그룹에 추가할 때 사용됩니다.
- `id` 키워드는 액션의 결괏값을 참조할 때 사용할 ID를 정의합니다.

### STEP11: AWS 자격 증명 구성하기

```yaml
- name: Configure AWS IAM credentials in repository secrets
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ap-northeast-2
```

- `aws-actions/configure-aws-credentials@v4` 액션을 사용하여 IAM 계정의 AWS 자격 증명을 설정합니다.
- AWS 자격 증명은 AWS CLI를 사용하여 EC2의 콘솔에 접근할 때 사용됩니다.
- `${{ secrets.AWS_ACCESS_KEY_ID }}`, `${{ secrets.AWS_SECRET_ACCESS_KEY }}`는 GitHub에 저장된 AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY를 참조합니다.
- GitHub의 secrets에 저장된 환경 변수는 DOCKERHUB_USERNAME, DOCKERHUB_TOKEN, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY으로 총 4개입니다.
- AWS 리전은 서울에 해당하는 `ap-northeast-2`로 설정합니다.

### STEP12: AWS EC2 보안 그룹 업데이트하기

```yaml
- name: Add GitHub IP to AWS
  run: |
    aws ec2 authorize-security-group-ingress \
    --group-id ${{ secrets.AWS_SG_ID }} \
    --protocol tcp --port 22 \
    --cidr ${{ steps.ip.outputs.ipv4 }}/32
```

- `aws ec2 authorize-security-group-ingress` 명령을 사용하여 AWS EC2 보안 그룹의 TCP 22번 포트에 GitHub Actions Runner의 공인 IP를 추가합니다.
- `${{ secrets.AWS_SG_ID }}`는 GitHub에 저장된 AWS_SG_ID를 참조합니다.
- GitHub의 secrets에 저장된 환경 변수는 DOCKERHUB_USERNAME, DOCKERHUB_TOKEN, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SG_ID으로 총 5개입니다.
- AWS 보안 그룹 ID는 AWS EC2 콘솔의 보안 그룹 탭에서 확인할 수 있습니다.
- 보안 그룹은 인바운드 및 아웃바운드 트래픽을 제어하는 방화벽 역할을 합니다.
- `--protocol tcp --port 22`는 TCP 프로토콜의 22번 포트에 대한 인바운드 트래픽을 허용합니다.
- `--cidr ${{ steps.ip.outputs.ipv4 }}/32`는 GitHub Actions Runner의 공인 IP를 CIDR 표기법으로 지정합니다.
- CIDR 표기법은 IP 주소와 서브넷 마스크를 결합하여 네트워크 주소를 나타냅니다.

### STEP13: AWS EC2 인스턴스에 Docker 컨테이너 배포하기

```yaml
- name: Executing remote ssh commands using password with AWS EC2
  uses: appleboy/ssh-action@v1.0.3
  with:
    host: ${{ secrets.EC2_HOST }}
    username: ${{ secrets.EC2_USERNAME }}
    password: ${{ secrets.EC2_PASSWORD }}
    port: ${{ secrets.EC2_SSH_PORT }}
    timeout: 60s
    script: |
      sudo docker stop github-actions-demo \
      && sudo docker rm github-actions-demo \
      && sudo docker rmi ${{ secrets.DOCKERHUB_USERNAME }}/github-actions-demo \
      || true
      sudo docker run -it -d -p 8080:8080 \
      --name github-actions-demo ${{ secrets.DOCKERHUB_USERNAME }}/github-actions-demo
```

- `appleboy/ssh-action@v1.0.3` 액션을 사용하여 AWS EC2 인스턴스에 SSH로 접속하여 Docker 컨테이너를 실행합니다.
- `${{ secrets.EC2_HOST }}`, `${{ secrets.EC2_USERNAME }}`, `${{ secrets.EC2_PASSWORD }}`, `${{ secrets.EC2_SSH_PORT }}`는 GitHub에 저장된 EC2_HOST, EC2_USERNAME, EC2_PASSWORD, EC2_SSH_PORT를 참조합니다.
- GitHub의 secrets에 저장된 환경 변수는 DOCKERHUB_USERNAME, DOCKERHUB_TOKEN, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SG_ID, EC2_HOST, EC2_USERNAME, EC2_PASSWORD, EC2_SSH_PORT으로 총 9개입니다.
- `sudo docker stop github-actions-demo`는 실행 중인 Docker 컨테이너를 중지합니다.
- `sudo docker rm github-actions-demo`는 중지된 Docker 컨테이너를 제거합니다.
- `sudo docker rmi ${{ secrets.DOCKERHUB_USERNAME }}/github-actions-demo`는 Docker 이미지를 제거합니다.
- 세 개의 명령을 `&&`로 연결하여 순차적으로 실행합니다.
- `|| true`는 이전 명령이 실패해도 다음 명령을 실행합니다.
- 따라서 최초 실행 시 Docker 컨테이너가 없어서 `docker stop`이 실패하더라도 다음 명령을 통해 Docker hub에서 이미지를 다운로드하고 컨테이너를 실행합니다.
- `sudo docker run -it -d -p 8080:8080 --name github-actions-demo ${{ secrets.DOCKERHUB_USERNAME }}/github-actions-demo`는 Docker 컨테이너를 실행합니다.
- `-it`는 상호작용 모드로 실행합니다.
- `-d`는 백그라운드 모드로 실행합니다.
- `-p 8080:8080`은 호스트의 8080 포트와 컨테이너의 8080 포트를 연결합니다.
- `--name github-actions-demo`은 컨테이너의 이름을 지정합니다.
- `${{ secrets.DOCKERHUB_USERNAME }}/github-actions-demo`는 Docker Hub에 푸시된 이미지를 사용합니다.

### STEP14: AWS EC2 보안 그룹 정리하기

```yaml
- name: Remove IP FROM security group
  run: |
    aws ec2 revoke-security-group-ingress \
    --group-id ${{ secrets.AWS_SG_ID }} \
    --protocol tcp --port 22 \
    --cidr ${{ steps.ip.outputs.ipv4 }}/32
```

- `aws ec2 revoke-security-group-ingress` 명령을 사용하여 AWS EC2 보안 그룹의 TCP 22번 포트에 GitHub Actions Runner의 공인 IP를 삭제합니다.
- CI/CD 파이프라인이 실행될 때마다 보안 그룹에 새로운 IP가 추가되므로, 작업이 완료되면 IP를 삭제합니다.

## CI/CD 파이프라인 사전 준비

### GitHub 저장소 생성하기

GitHub Actions는 GitHub 저장소에서 코드를 가져와 CI/CD 파이프라인을 실행합니다. 따라서 먼저 GitHub 저장소를 생성해야 합니다.

- 저장소에는 Spring Boot 프로젝트의 소스 코드가 포함되어 있어야 합니다.
- 저장소의 루트 디렉토리에 `.github/workflows` 디렉토리와 워크 플로우 파일이 있어야합니다.
- 저장소의 루트 디렉토리에 **STEP7: Docker 이미지 빌드하기**에서 사용할 `Dockerfile`이 있어야합니다.
- 저장소에는 DOCKERHUB_USERNAME, DOCKERHUB_TOKEN, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SG_ID, EC2_HOST, EC2_USERNAME, EC2_PASSWORD, EC2_SSH_PORT 총 9개의 secrets가 설정되어 있어야합니다.

### Docker Hub에서 secrets 값 가져오기

**STEP7: Docker 이미지 빌드하기**와 **STEP8: Docker Hub에 로그인하기**에 사용되는 Docker Hub의 username과 token을 가져오기 위해 Docker Hub에 로그인합니다.

1. Docker Hub에 로그인합니다.
2. 우측 상단의 프로필 이미지를 클릭하고 `My Account`로 이동합니다.
3. 좌측 상단의 `user` 위에 있는 계정명이 **DOCKERHUB_USERNAME**에 사용될 값입니다.
4. 좌측의 `Security` 탭을 클릭하고 우측의 `New Access Token`을 클릭합니다.
5. `Access Token Description`에 임의의 값을 입력하고 `Generate` 버튼을 클릭합니다.
6. `2. At the password prompt, enter the personal access token.` 하단의 값이 **DOCKERHUB_TOKEN**에 사용될 값입니다.

### Github Repository에 secrets 추가하기

1. GitHub 저장소로 이동합니다.
2. 우측 상단의 `settings`를 클릭합니다.
3. 좌측 하단의 `Secrets and variables > Actions`를 클릭합니다.
4. 우측의 `New repository secret`을 클릭합니다.
5. Name에 **DOCKERHUB_USERNAME**을 입력하고 Value에 docker hub에서 가져온 값을 입력합니다.
6. **DOCKERHUB_TOKEN**에 대해서도 동일한 방법으로 하나씩 추가합니다.
7. 총 2개의 secrets가 저장되었습니다.

### AWS IAM 계정 생성하기

CI/CD 파이프라인이 AWS EC2 인스턴스에 접속하고 보안 그룹을 업데이트하려면 AWS IAM 계정이 필요합니다.

1. Amazon Web Services에 로그인합니다.
2. IAM 대시보드로 이동합니다.
3. 좌측에서 사용자 탭을 선택하고 우측 상단의 노란색 `사용자 생성` 버튼을 클릭합니다.
4. 사용자 이름에 임의의 값을 입력하고 노란색 다음 버튼을 클릭합니다.
5. `직접 정책 연결`을 선택하고 권한 정책에서 `EC2FullAccess`를 검색하여 체크합니다.
6. 노란색 `사용자 생성` 버튼을 클릭합니다.

### AWS IAM 계정에서 secrets 값 가져오기

**STEP11: AWS 자격 증명 구성하기**에 사용되는 AWS IAM 계정의 액세스 키와 비밀 액세스 키를 가져오기 위해 액세스 키를 생성합니다.

1. 사용자를 선택하고 `보안 자격 증명` 탭을 선택합니다.
2. `액세스 키 만들기`를 클릭합니다.
3. 액세스 키의 값이 **AWS_ACCESS_KEY_ID**에 사용됩니다.
4. 비밀 액세스 키의 값이 **AWS_SECRET_ACCESS_KEY**에 사용됩니다.
5. GitHub 저장소의 secrets에 저장합니다.
6. 총 4개의 secrets가 저장되었습니다.

### AWS EC2 인스턴스 생성하기

CI/CD 파이프라인을 실행할 AWS EC2 인스턴스를 생성합니다.

1. Amazon Web Services에 로그인합니다.
2. EC2 대시보드로 이동합니다.
3. 노란색 `인스턴스 시작` 버튼을 클릭합니다.
4. `이름 및 태그`에 임의의 값을 입력합니다.
5. 애플리케이션 및 OS 이미지, 인스턴스 유형, 스토리지 구성, 고급 세부 정보는 기본값을 사용합니다.
6. `키 페어(로그인)`은 `키 페어 없이 계속 진행`을 선택합니다. 키 페어는 SSH로 접속할 때 사용되는 인증 수단입니다.
7. `네트워크 설정`의 우측 상단에서 `편집` 버튼을 누르고 좌측 하단의 `보안 그룹 규칙 추가` 버튼을 클릭합니다. `포트 범위`를 22번으로 설정하고 `소스 유형`을 위치 무관으로 설정합니다.
8. 최하단의 노란색 `인스턴스 시작` 버튼을 클릭합니다.

### AWS EC2 인스턴스 요약에서 secrets 값 가져오기

**STEP12: AWS EC2 보안 그룹 업데이트하기**와 **STEP13: AWS EC2 인스턴스에 Docker 컨테이너 배포하기**에 사용되는 EC2의 보안 그룹 이름, 퍼블릭 IPv4 DNS를 가져오기 위해 EC2 인스턴스의 요약을 확인합니다.

1. EC2 대시보드로 이동합니다.
2. 인스턴스(실행중)을 클릭합니다.
3. 인스턴스 ID를 클릭하면 인스턴스의 요약이 표시됩니다.
4. 하단의 `보안` 탭을 클릭하고 `보안 그룹`의 복사버튼을 클릭해 값을 복사합니다. 이 값이 **AWS_SG_ID**에 사용됩니다.
5. 상단의 `퍼블릭 IPv4 DNS`를 복사합니다. 이 값이 **EC2_HOST**에 사용됩니다.
6. 총 6개의 secrets가 저장되었습니다.

### AWS EC2 인스턴스에 접속해서 secrets 값 가져오기

**STEP13: AWS EC2 인스턴스에 Docker 컨테이너 배포하기**에 사용되는 EC2 인스턴스의 username, password를 가져오기 위해 EC2 인스턴스에 접속합니다.

1. EC2 대시보드로 이동합니다.
2. 인스턴스(실행중)을 클릭합니다.
3. 인스턴스를 체크하고 상단의 `연결` 버튼을 클릭합니다.
4. 하단에 보이는 `사용자 이름`의 `ec2-user`가 **EC2_USERNAME**에 사용됩니다.
5. 하단의 노란색 `연결` 버튼을 클릭하면 새 창에서 EC2 인스턴스가 SSH로 연결됩니다.
6. RHEL 계열의 패키지 관리자인 yum을 업그레이드하고 Docker를 설치 후 실행합니다.

   ```bash
   sudo yum upgrade -y && yum update -y

   sudo yum install docker -y

   sudo service docker start
   ```

7. 관리자 권한으로 `ec2-user`의 비밀번호를 설정합니다. 여기서 입력한 값이 **EC2_PASSWORD**에 사용됩니다.

   ```bash
   sudo passwd ec2-user
   ```

    - `passwd` 명령어는 사용자의 비밀번호를 변경하는 명령어입니다.

8. **STEP13: AWS EC2 인스턴스에 Docker 컨테이너 배포하기**에서 SSH 키 페어가 아닌 비밀번호로 접속하기 위해 설정파일을 수정합니다.

   ```bash
   sudo vim /etc/ssh/sshd_config
   ```

    - `vim`은 리눅스에서 사용하는 텍스트 편집기입니다.
    - `:`을 누르고 `set nu`를 입력하여 줄 번호를 표시합니다.
    - 65번 줄로 가서 i를 누르고 편집모드에 진입합니다.
    - `PasswordAuthentication yes`를 입력하고 ESC를 눌러 편집모드를 종료합니다.
    - `:wq`를 입력하여 저장하고 종료합니다.

9. 설정 변경을 반영하기 위해 SSH 서비스를 재시작합니다.

   ```bash
   sudo systemctl sshd restart
   ```

    - systemctl 명령은 시스템 서비스를 관리하는 명령어입니다.
    - `restart`는 서비스를 재시작하는 옵션입니다.
    - `sshd`는 SSH 서비스의 이름입니다.

10. GitHub 저장소의 secrets에 저장합니다.
11. 총 8개의 secrets가 저장되었습니다.
12. 마지막 **EC2_SSH_PORT**는 SSH로 접속할 때 사용되는 22를 저장합니다.
