# ⚙️ Kubernetes를 활용한 Spring Boot 애플리케이션 배포 및 관리

## 📜 개요

**Spring Boot 애플리케이션**을 **Kubernetes**를 통해 배포하고, **3개의 레플리카로 구성**하여 외부 통신이 가능한 **서비스를 생성**하는 과정을 알아보고자 합니다.

로컬 환경에서 개발한 Spring Boot 앱을 가상 환경(Ubuntu Linux)에서 Docker를 통해 실행하고, 이를 Kubernetes로 배포하여 애플리케이션의 확장성과 가용성을 극대화하는 것이 목표입니다. 또한, **멀티 아키텍처 Docker 이미지**를 빌드하고, **Minikube**를 사용해 외부에서 접근할 수 있도록 설정하는 과정을 다룹니다.

## 🔧 과정

## 1️⃣ Spring Boot 프로젝트 Ubuntu에서 Docker로 실행

1. https://start.spring.io/ 사이트에서 간단한 myapp 프로젝트를 생성합니다.
2. 해당 프로젝트에서 Dockerfile을 생성하고, 빌드하여 이미지를 만듭니다.

### **Dockerfile 생성**

```docker
FROM openjdk:17-jdk-slim  # alpine 버전은 m1 지원 안됨
ARG JAR_FILE=build/libs/*.jar # jar 파일 생성 위치 확인
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

이후 bootJar를 통해 jar 파일을 빌드하고, Docker 이미지화합니다.

**이미지 빌드**

```docker
docker buildx build --platform linux/amd64,linux/arm64 -t lisiant/myapp:latest --push .
```

이전에 platform 문제로 많이 어려움을 겪었기 때문에, 다중 아키텍처 이미지를 빌드하기 위해 `docker buildx` 를 사용하였습니다.

![image](https://github.com/user-attachments/assets/39449c68-5602-4e11-b805-e8030aeceffa)


DockerHub에 멀티 아키텍처 이미지가 push된 것을 확인할 수 있습니다.

### ⚠️ 트러블슈팅 : Docker buildx

**문제 상황**

명령어를 올바르게 입력하지 않아 이미지가 push되지 않는 현상이 발생했습니다.

1. `"docker buildx build" requires exactly 1 argument.`
    
    **원인**
    
    멀티 플랫폼을 기술할 때 콤마 사이에 띄어쓰기를 입력해서 발생했습니다.
    
    ```bash
    docker buildx build --platform linux/amd64, linux/arm64 -t myapp:latest .
    
    -> 수정
    docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest .
    
    ```
    
2. `authorization_failed`

    **오류 내용**
   
    ```bash
    docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest .
    
    ERROR: failed to solve: failed to push myapp:latest: push access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
    ```
    
    **원인**
    
    이미지에 태그를 부여할 시 DockerHub username이 있어야 합니다.
    
    **해결**
    
    ```bash
    docker buildx build --platform linux/amd64,linux/arm64 -t lisiant/myapp:latest .
    ```
    
4. 이미지 push가 안되는 문제
    
    ```bash
    docker buildx build --platform linux/amd64,linux/arm64 -t lisiant/myapp:latest .
    
    WARNING: No output specified with docker-container driver. Build result will only remain in the build cache. To push result image into registry use --push or to load image into docker use --load
    ```
    
    **원인**
    
    - `-push` 옵션을 추가하지 않아서 발생한 문제
    
    **해결**
    
    ```bash
    docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest --push .
    ```
    

### docker container 실행

```docker
docker pull <계정이름>:<이미지명>
docker run -p 8080:8080 --name myapp -d lisiant/myapp:latest

```
![image](https://github.com/user-attachments/assets/329a7507-0813-4200-98fb-b149bec0c6bb)

Container 위에서 Spring Boot 앱이 잘 동작하는 것을 확인하였습니다.

![image](https://github.com/user-attachments/assets/a656a0a8-f736-4116-8644-854673e9babe)

## 2️⃣ Kubernetes를 통한 Spring Boot 앱 배포 준비

기존 컨테이너는 종료하고, Kubernetes를 사용하여 앱 배포 과정을 알아보겠습니다.

### Kubernetes 클러스터 준비

Minikube를 사용하여 클러스터를 준비하고, Docker 이미지와 연동합니다.

```docker
minikube start
```

### Kubernetes에서 배포

이미지가 DockerHub에 있으므로 이미지를 빌드하거나 푸시할 필요 없이 Kubernetes자동으로 DockerHub에서 이미지를 가져옵니다.

이는 Deployment.yaml 파일을 작성하는 방법으로 진행할 수도 있고, `kubectl` 명령어를 통해 진행할 수도 있습니다.

### `kubectl` 사용 예제

```bash
kubectl create deployment myapp --image=lisiant/myapp:latest --replicas=3
kubectl get services

```

![image](https://github.com/user-attachments/assets/287a202e-be7a-401b-9e3c-28f645fe331c)

아직 서비스를 노출하지 않았기 때문에 `kubectl get services` 로는 확인할 수 없습니다.

Replica 3개가 생성된 것을 `kubectl get deployments` 로 확인할 수 있습니다.

![image](https://github.com/user-attachments/assets/a4cd86f6-25bb-410c-b9a3-361bd7cd6eeb)

### 외부 로드밸런서를 통한 서비스 노출

`kubectl expose deployment` 명령어를 사용하여 Spring Boot 앱을 외부에서 접근 가능하도록 설정하겠습니다.

myapp이라는 이름의 배포를 외부 로드밸런서를 통해 포트 8080으로 노출하겠습니다.

```bash
kubectl expose deployment myapp --type=LoadBalancer --port=8080
kubectl get services

```

![image](https://github.com/user-attachments/assets/67481395-fbab-476e-9b0a-968bb0789e93)

## 3️⃣ Minikube를 통한 외부 IP 제공

Minikube를 통해 외부 IP를 생성하여 외부에서 애플리케이션에 접근할 수 있도록 합니다.

### Minikube 터널 실행

```bash
minikube tunnel

```

터널을 생성하고 외부 IP를 할당합니다.

![image](https://github.com/user-attachments/assets/642ef2b9-7a4d-4590-9f53-86219d61845a)

이후 서비스의 `EXTERNAL-IP` 항목이 할당되었는지 확인합니다.

minikube tunnel은 백그라운드로 실행되고 있지 않기 때문에, 다른 bash를 열어 해당 내용을 확인합니다.

```bash
kubectl get services

```

![image](https://github.com/user-attachments/assets/9b58c285-a98f-4428-8ea3-f662298284f4)

```bash
curl <http://10.97.96.21:8080/test>
test

```

### ⏭️ 트러블슈팅: 포트포워딩 설정

**문제 상황**

Ubuntu 환경에서 curl 명령어로 External IP에 접속이 가능하였으나, 로컬 Mac 환경의 브라우저에서 해당 IP로 접속이 불가능하였습니다.

**원인**

상단의 EXTERNAL-IP는 Kubernetes Pod와 Ubuntu가 통신하기 위한 IP입니다.

이를 Local 환경의 브라우저에서 사용하기 위해서는 포트포워딩 과정이 필요합니다.

**해결**

VSCode를 통해 간편하게 포트포워딩을 진행할 수 있습니다.

![image](https://github.com/user-attachments/assets/b774dfc4-62a6-4670-a9ea-389ab03f9046)

### **Minikube Dashboard**

![image](https://github.com/user-attachments/assets/1df4c733-a44d-46c7-b032-43586ef870be)

Dashboard를 통해 안정적으로 배포되는 과정을 확인할 수 있습니다.

![image](https://github.com/user-attachments/assets/3119751a-e0e1-494e-b170-c9670ae8ccc3)

포트포워딩을 통해 localhost로 접근 가능한 것을 확인할 수 있습니다.

## 🗼도식화

```bash
+-----------------+       External IP         +-------------------+     Port Forwarding    +-------------------+
|                 |  <------------------->    |                   |  <-------------------> |                   |
|                 |                           |                   |                        |                   |
| Kubernetes Pod  |        10.97.x.x          |  Virtual Machine  |       localhost:8080   |   Local MacBook   |
| (Spring Boot)   |    (External IP via LB)   |   (Linux VM)      |       (Tunnel)         |   (Mac)           |
|                 |                           |                   |                        |                   |
|  Cluster IP     | <-----------> Container   |                   |                        |                   |
|   10.96.x.x     |                           |                   |                        |                   |
+-----------------+                           +-------------------+                        +-------------------+
        |                                               |
        | <---------> Load Balancer (Kubernetes Service)|
        |                                               |
        |                                               |
  +--------------------+                                |
  |                    |                                |
  |   Kubernetes       |                                |
  |   Cluster Network  |                                |
  |   (Minikube, etc.) |                                |
  +--------------------+                                |
                                                        |
                                                +---------------------+
                                                |                     |
                                                |  Minikube Tunnel    |
                                                |  (Port Forwarding)  |
                                                +---------------------+

```

## 🏁 결론

이번 과정을 통해 **Spring Boot 애플리케이션**을 **Kubernetes** 상에서 **3개의 레플리카로 배포**하고, **외부 통신이 가능한 서비스**로 구성할 수 있었습니다. 특히, **멀티 아키텍처 지원**을 위한 **Docker buildx**와 **Kubernetes의 자동화된 배포** 및 **확장성**을 직접 경험할 수 있었습니다.

또한, **Minikube를 통한 외부 IP 제공**과 **포트 포워딩** 과정을 통해 로컬 환경에서 Kubernetes 클러스터의 Pod에 접근하는 방법도 확인하였습니다.

현재는 로드밸런싱만 고려하였지만, 추후 상태 동기화도 고려하는 과정을 진행하고자 합니다.
