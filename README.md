# jenkins-ci-pipeline
>  Use Jenkins to build a Docker image locally and push it to Docker Hub with a versioned tag.
> Then, automatically update the corresponding values.yaml file in the GitOps repository to trigger a deployment via ArgoCD.

## 관련된 전체 리포지토리 구조
```
📦 전체 구조도
├── jenkins-ci-pipeline (도커파일 위치 + Jenkinsfile)
│   └── vvue-be/
│       ├── build/docker/
│       └── Jenkinsfile
├── vvue-be-refactoring (앱 소스코드)
│   └── build/libs/app.jar (Gradle 빌드 산출물)
└── k8s-gitops (배포 선언)
    └── apps/vvue-be/
        ├── dev/
        ├── staging/
        └── prod/
```

## 현재 리포지토리 구조  

```text
jenkins-ci-pipeline/
└── vvue-be/
    ├── build/docker
    └── Jenkinsfile
```

## 실행 흐름 요약
- Github에서 코드 clone
- Gradle로 JAR 빌드
- 별도 Dockerfile 레포지토리에서 Sparse Checkout
- Docker build + Push > DockerHub
- k8s-gitops 레포지토리에 Helm values 파일 업데이트

## Dockerfile
- `ARG IDLE_PROFILE`: Jenkins에서 전달받은 spring.profiles.active 값
- `COPY app.jar`: Gradle로 빌드한 결과물 주입
- `ENTRYPOINT`: 해당 프로파일로 Spring Boot 실행

## Jenkins 
### 주요기능
- 민감 정보는 Jenkins Credential Store를 통해 주입됨 
- 프로파일 기반으로 
    - Dodcker 이미지 빌드 및 푸시 : DockerHub에 `${APP_NAME}:${TAG}` 형식으로 업로드
    - 이미지 버전 변경 기록 : Helm values 파일도 업데이트됨

### 구축 과정
#### 1. VirtualBox에 Ubuntu Server 구축
- [VirtualBox 다운로드](https://www.virtualbox.org/wiki/Downloads)
- [Ubuntu Server 다운로드](https://ubuntu.com/download/server)


#### 2. Jenkins 설치 및 서버 설정
##### 2.1. 설치
- [우분투에 Jenkins 설치](https://www.jenkins.io/doc/book/installing/linux/#debianubuntu)

```bash
sudo journalctl -xeu jenkins.service #실패 로그 확인
sudo systemctl daemon-reexec 
sudo systemctl restart jenkins
```

- 소스코드 빌드용 Open JDK 17 설치
> Jenkins 구동에도 JDK가 필요해서 설치해야 한다.  
> readlink -f $(which java) #경로 확인하기

```bash
wget https://download.java.net/java/GA/jdk17.0.2/dfd4a8d0985749f896bed50d7138ee7f/8/GPL/openjdk-17.0.2_linux-x64_bin.tar.gz
tar -xvzf openjdk-17.0.2_linux-x64_bin.tar.gz
mv jdk-17.0.2 /usr/lib/jvm/jdk-17.0.2
```

- docker 설치 및 권한 부여
    - [설치 방법](https://docs.docker.com/engine/install/ubuntu/)
```bash
sudo su
chmod 666 /var/run/docker.sock
usermod -aG docker jenkins
su - jenkins -s /bin/bash
```

##### 2.2. Jenkins 서버 설정

- 서버 설정 (메뉴 - Jenkins관리 > Tools)
    - jdk 설정 : `/usr/lib/jvm/jdk-17.0.2/bin`
    - gradle 설정 : `gradle 8.2.1` install automatically로 설치
- Github, Docker Hub 계정 Secret 추가 (메뉴 - Jenkins 관리 > Credentials)
    - Github 계정
        - `Username and Password`
            - Scope : `Global(Jenkins, nodes, items, all child items, etc)`
            - Username : <Github-Username>
            - Password : <github-personal-token|contents:read&write>
    - Docker Hub 계정
        - `Username and Password`
            - Scope : `Global(Jenkins, nodes, items, all child items, etc)`
            - Username : <dockerhub-Username>
            - Password : <dockerhub-Password>
        
#### 3. Jenkins 프로젝트 생성

- 파이프라인 작성
    - 환경별 선택 (dev, staging, prod)
    - 도커 파일 빌드 및 푸시
    - Helm으로 구성된 k8s-gitops 프로젝트 내 values 파일 업데이트 
