# Kubernetes Version Update Runbook

클라우드 환경에 구성된 kubernetes의 버전 업그레이드 절차를 가이드한다.

클러스터과 노드의 api는 상호간에 &plusmn;1 버전(minor) 차이만 지원하므로 한번에 2(minor)이상의 버전 업데이트는 지원하지 않는다. </br>
예를 들어 1.16 &#10132; 1.20으로 버전 업데이트가 필요할 경우, 1.17 &#10132; 1.18 &#10132; 1.19 &#10132; 1.20으로 순차적으로 버전을 업데이트 해야한다.
또는 신규 클러스터(v1.20)를 신규 생성한후, 애플리케이션을 재배포한다.

## Prerequisites

사전 준비는 버전 업데이트 작업을 수행하기전 최소 1주일 전에 수행한다.

### Tools

- kubectl
- [kube-no-trouble](https://github.com/doitintl/kube-no-trouble)

  ```Shell
  sh -c "$(curl -sSL https://git.io/install-kubent)"
  ```

- cli (e.g. aws-cli, azure-cli, etc)

### Check All helm Charts

- Helm Charts

  ```Shell
  # Installed Helm Charts
  helm list -A

  # Get helm values
  helm get values <RELEASE_NAME> -n <NAMESPACE> > RELEASE_NAME.yaml
  ```

### Check Deprecated APIs

- 업데이트 할 버전에서 지원하지 않는 API를 검색한다.
- 검출된 API가 존재할 경우, 해당 API가 적용된 모든 리소스에 대해 업그레이드를 수행한다.

#### How to check the APIs version

- `cluster-admin` 권한으로 실행
- 실행

  ```Shell
  /usr/local/bin/kubent
  ```

- 실행 결과는 다음 예제와 같이 출력된다.

  ```Shell
  $ kubent
  2:29PM INF >>> Kube No Trouble `kubent` <<<
  2:29PM INF version 0.5.1 (git sha a762ff3c6b5622650b86dc982652843cc2bd123c)
  2:29PM INF Initializing collectors and retrieving data
  2:29PM INF Target K8s version is 1.19.13-eks-8df270
  2:29PM INF Retrieved 47 resources from collector name=Cluster
  2:29PM INF Retrieved 25 resources from collector name="Helm v2"
  2:29PM INF Retrieved 44 resources from collector name="Helm v3"
  2:29PM INF Loaded ruleset name=custom.rego.tmpl
  2:29PM INF Loaded ruleset name=deprecated-1-16.rego
  2:29PM INF Loaded ruleset name=deprecated-1-22.rego
  2:29PM INF Loaded ruleset name=deprecated-1-25.rego
  __________________________________________________________________________________________
  >>> Deprecated APIs removed in 1.22 <<<
  ------------------------------------------------------------------------------------------
  KIND                           NAMESPACE     NAME                               API_VERSION                            REPLACE_WITH (SINCE)
  CustomResourceDefinition       <undefined>   eniconfigs.crd.k8s.amazonaws.com   apiextensions.k8s.io/v1beta1           apiextensions.k8s.io/v1 (1.16.0)
  Ingress                        mof2          sbs-gis-server-ingress             extensions/v1beta1                     networking.k8s.io/v1 (1.19.0)
  MutatingWebhookConfiguration   <undefined>   pod-identity-webhook               admissionregistration.k8s.io/v1beta1   admissionregistration.k8s.io/v1 (1.16.0)
  ```

- 예제의 경우, v1.21 이전 버전에서 나열된 api에 대해 반드시 업그레이드를 수행한 후, v1.22 업그레이드 작업을 수행해야 한다.
- 업데이트 대상 api가 helm으로 배포된 경우, helm charts의 버전을 업데이트한다.

  업데이트가 필요한 경우 `--dry-run`옵션을 사용하여 업데이트에 문제가 없는지 먼저 평가한다.

  ```Shell
  helm upgrade <RELEASE_NAME> <CHART> --dry-run \
  --version <TARGET_VERSION> \
  -f RELEASE_NAME.yaml
  ```

- 클러스터 버전별 적용 가능한 helm charts 버전은 Helm Chart 별 공식 홈페이지를 참고한다.

### ETC

- dockerhub를 사용할 경우, `Download rate limit`이 발생할 수 있다. (사전에 고객 관리형 image registry 전환 등을 검토한다.)
- 클러스터 업데이트 수행 시, 노드 또는 application 재배포에 따른 IP 부족이 발생할 있다.
- `PodDisruptionBudget`가 설정되어 있는 경우, 업데이트 전에 `pdb`를 삭제하거나 deployment의 `.spec.replicas` 설정을 pdb의 `.spec.minAvailable` 보다 크게 설정한다.

## Process

### 1. 클러스터 버전 업데이트

터미널 또는 웹 콘솔에서 클러스터의 버전을 업데이트한다.

- 터미널의 경우

  ```Shell
  # e.g. aws eks update-cluster-version --name example --kubernetes-version 1.13
  ```

### 2. 노드 버전 업데이트

#### 2.1 신규 버전의 노드 또는 노드그룹을 생성하여 업데이트하는 방법

사용자 관리형 노드/노드그룹일 경우 다음과 같은 절차를 통해 노드 또는 노드그룹을 업데이트한다.

- 터미널의 경우

  ```Shell
  export OLD_VERSION=1.19 # old version

  # add new node or nodegroup
  # e.g. eksctl create nodegroup --cluster=<clusterName> [--name=<nodegroupName>]

  # get node list
  kubectl get nodes

  # cordon nodes
  kubectl get nodes | awk -v ver="$OLD_VERSION" '$0 ~ ver {print $1}' | xargs kubectl cordon

  # evict pods
  # DO NOT DRAIN ALL NODE AT ONCE
  kubectl drain <node-to-drain> --ignore-daemonsets --delete-emptydir-data

  # delete old nodes or nodegroup
  # e.g. eksctl delete nodegroup --cluster=<clusterName> --name=<nodegroupName>
  ```

#### 2.2 기존 노드 또는 노드 그룹을 업데이트하는 방법

일반적으로 관리형 노드 그룹에서만 지원되는 기능(AWS EKS 경우)으로 2.1과 유사한 방식으로 노드 또는 노드그룹을 업데이트 하며 전체 업데이트 절차를 클라우드에서 관리한다

터미널 또는 웹 콘솔에서 노드 또는 노드그룹의 버전을 업데이트

- 터미널의 경우

  ```Shell
  # e.g. aws eks update-nodegroup-version --cluster-name <value> --nodegroup-name <value>
  ```

### 3. Validate

클러스터 및 노드 또는 노드그룹을 버전을 업데이트한 후, 정상적으로 업데이트 되었는지 또는 애플리케이션이 정상 동작하는 지 점검한다.

- Node's kubelet version

  ```Shell
  export VERSION='1.19' # target version
  kubectl get nodes | grep "v${VERSION}"
  ```

- Pod's running status

  ```Shell
  # 'Non-Running' Pods
  kubectl get pods -A --field-selector status.phase!=Running

  # Pads with Restart greater than zero
  kubectl get pods -A | awk '$5 > 0'
  ```

- Not normal events

  ```Shell
  kubectl get events --field-selector type!=Normal -A
  ```
