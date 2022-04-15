# AWS CDK 실행환경 구성

AWS Cloud Development Kit(AWS CDK)는 익숙한 프로그래밍 언어를 사용하여 클라우드 애플리케이션 리소스를 정의할 수 있는 오픈 소스 소프트웨어 개발 프레임워크이며 다양한 개발언어(TypeScript, JavaScript, Python, Java, C#)를 지원한다.

TypeScript의 경우 많은 예제와 가이드가 있으므로 여기서는 TypeScript에 대한 실행환경 구성에 대해 설명한다.

## 실행환경 구성

### Node.js 설치

cdk는 LTS 버전의 Node.js을 지원하며 `Active LTS` 버전을 설치할 것을 권장한다. (Node.js 버전이 LTS 버전이 아닌 경우, cdk 실행 시 오류가 발행 할 수 있다)

LTS 버전에 관한 자세한 정보는 다음을 [참고](https://nodejs.org/ko/about/releases/)한다.

다양한 OS 환경에서의 설치는 다음을 [참조](https://nodejs.org/ko/download/package-manager/)한다.

- 예제에서는 ubuntu를 기준으로 기술한다. (다양한 Ubuntu 버전에서의 Node.js 설치 방법은 다음을 [참고](https://github.com/nodesource/distributions/blob/master/README.md)한다.)

  ```Shell
  # Using Ubuntu
  curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -
  sudo apt-get install -y nodejs

  # Using Debian, as root
  curl -fsSL https://deb.nodesource.com/setup_16.x | bash -
  apt-get install -y nodejs
  ```

- 설치 확인

  ```Shell
  # Node.js Version
  node -v

  # Npm Version
  npm -v
  ```

### cdk 설치

- 설치

  ```Shell
  sudo npm -g install typescript

  # 최신 cdk 버전을 설치
  sudo npm install -g aws-cdk

  # 특정 cdk 버전을 설치
  sudo npm install -g aws-cdk@1.x
  ```

- 설치확인

  ```Shell
  cdk --version
  ```

- 특정 라이브러리 설치

  ```Shell
  # aws-cdk-lib를 설치할 경우
  sudo npm install -g aws-cdk-lib
  ```

### CDK Bootstrapping

  CDK Bootstrapping을 위해서는 AWS configure 설정이 필요하다.

  ```Shell
  # aws configure
  aws configure

  # Bootstrapping
  cdk bootstrap aws://ACCOUNT-NUMBER/REGION
  ```

## ETC

- Typescript 버전 최신화

  ```Shell
  sudo npm update -g typescript
  ```
