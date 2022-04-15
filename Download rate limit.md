# Download rate limit

Docker Hub는 Docker에서 관리하는 호스팅된 Docker 레지스트리로써 소프트웨어 공급업체, 오픈 소스 프로젝트 및 커뮤니티에서 제공하는 100,000개 이상의 컨테이너 이미지가 존재한다. Docker Hub에는 NGINX, Logstash, Apache HTTP, Grafana, MySQL, Ubuntu 및 Oracle Linux와 같은 공식 리포지토리의 소프트웨어 및 애플리케이션이 포함되어 있어 많은 Helm charts 및 애플리케이션 배포에서 이미지를 기본적으로 공용 Docker Hub에서 자동으로 가져온다.(로컬에서 사용할 수 없는 경우).

애플리케이션 배포 시, 특정 제약 횟수를 초과하여 이미지를 Docker hub에서 다운로드 할 경우(대부분 일정 규모 이상의 신규 배포 시, 발생한다.) 이미지 다운로드 오류가 발생하여 일시적인 서비스 장애가 발생할 수 있다.

이 경우 다음과 같은 알람이 log 등에 출력된다.

```text
You have reached your pull rate limit. You may increase the limit by authenticating and upgrading: https://www.docker.com/increase-rate-limits
```

Dowonload rate limit은 Docker inc.의 고유 정책으로 시기에 따라서 달라지므로 다운로드 제약 사항은 도커 사이트를 참고한다.
[Docker Pricing](https://www.docker.com/pricing)

- 2022년 1월 기준으로 대략적인 다운로드 제약 정책은 다음과 같다.

|        |Personal|Pro     |Team (minimum 5 users)   |Business|
|:------:|:------:|:------:|:------:|:------:|
|Price   | 0$     |$5  per month    |$7 per user/month |$21 per user/month |
|Download limits|200 image pulls per 6 hours|5,000 image pulls per day|unlimited|unlimited|

## Download rate limits 회피방안

### 1. 임시 방안

애플리케이션 배포 설정 파일(yaml)의 imagePullPolicy 변경. ***단, 노드가 많은 경우 임시 방안으로 해결이 안될 수 있음***

- `IfNotPresent`: 이미지가 로컬에 없는 경우에만 내려받음

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-test-1
spec:
  containers:
    - name: uses-private-image
      image: $PRIVATE_IMAGE_NAME
      imagePullPolicy: Always         # IfNotPresent로 변경
      command: [ "echo", "SUCCESS" ]
```

### 2. 영구 방안

영구 회피를 위해서는 다음의 2가지 안에서 **`유지 보수 편의성`**, **`시스템 환경`** 및 **`비용 효율성`** 등을 고려하여 하나 또는 모든 안를 선택 적용한다.

#### 2.1. Docker Hub Subscriptions

- Docker Hub에 계정을 생성하고 구독한다.
- Kubernetes에 Docker hub `imagePullSecret`을 생성한다.</br> ***Notice! `imagePullSecret`은 Namespace 별 생성해야 한다.***

  ```Shell
  kubectl create secret docker-registry <docker-secret-name> \
  --docker-server=DOCKER_REGISTRY_SERVER \
  --docker-username=DOCKER_USER \
  --docker-password=DOCKER_PASSWORD \
  --docker-email=DOCKER_EMAIL \
  --namespace NAMESPACE
  ```

- 애플리케이션 배포 설정(yaml)의 `imagePullSecret`에 생성한 docker-registry `secret`을 설정한 후 배포한다.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: foo
    namespace: awesomeapps
  spec:
    containers:
      - name: foo
        image: janedoe/awesomeapp:v1
    imagePullSecrets:
      - name: <docker-secret-name>
  ```

#### 2.2. 관리형 Image Registry 구성

다양한 Image Registry 솔루션이 있으나 특정 요구사항이 없으며 또한 EKS에 워크로드를 구성한 경우, `Image Registry`로 `ECR`을 권장한다. ***(여기서는 ECR에 관해서만 기술하며 다른 솔루션의 설치 구성에 대해서는 해당 솔루션의 문서를 참조한다.)***

- ECR을 적용할 시의 이점

  - 완전 관리형
  - EKS(Node)에서 Image Registry 접근을 위한 별도의 구성이 필요없다. (ECR에 대한 IAM Role이 기본적으로 부여되어 있다.)

- ECR에 이미지를 등록하는 방법

  - ECR login

    ```Shell
    # ECR login
    aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${REPO_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com
    ```

  - ECR Repository 생성

    ```Shell
    aws ecr create-repository --repository-name ${REPO_PREFIX}/csi-provisioner
    ```

  - Image Download

    ```Shell
    # image download (local)
    docker pull k8s.gcr.io/sig-storage/csi-provisioner:v2.1.1
    ```

  - Image Tagging

    ```Shell
    # Tagging downloaded images for ECR push (local)
    docker tag k8s.gcr.io/sig-storage/csi-provisioner:v2.1.1                    ${REPO_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${REPO_PREFIX}/csi-provisioner:v2.1.1
    ```

  - Image push

    ```Shell
    # Push the image from local to ECR
    docker push ${REPO_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${REPO_PREFIX}/csi-provisioner:v2.1.1
    ```

  - Variables

| Key | Description |
|-----|-------------|
|`REPO_ACCOUNT_ID`|AWS Account ID|
|`REPO_PREFIX`|Prefix (path/to/image)|
|`REGION`|ECR Region (e.g. ap-northeast-2)|

## 참고

- `.spec.containers[*].image`의 구성은 `registry_url(optional)/prefix(account)/image_name:tags(version:optional)`으로 구성된다.
- `registry_url`이 생략된 경우 `Docker Hub`를 의미한다.
- `prefix`가 생략된 경우 `/`를 의미한다.
- `tags`가 생략된 경우 `latest`을 의미한다.
- e.g. `image: janedoe/awesomeapp:v1`는 `docker hub`의 `janedoe계정`의 이미지 `awesomeapp` 버전 `v1`를 의미한다.
- registiry에 `immutable`를 설정한 경우, 동일한 tag(e.g. latest)를 사용할 수 없다.
