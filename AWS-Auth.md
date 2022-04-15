# aws-auth

EKS에서 Kubernetes의 RBAC과 IAM을 매핑하기 위한 configmap

## 사용자 또는 IAM Role 등록

- **최초 kubernetes 사용자 또는 Role에 권한을 할당하기 위해서는 반드시 EKS 클러스터를 생성한 AWS User로 작업**해야 한다.

### 권한 조회

```shell
kubectl get cm aws-auth -n kube-system -oyaml > aws-auth.yaml
```

노드 그룹을 생성하지 않은 경우 aws-auth는 생성되지 않으므로 조회되지 않으며 EKS 클러스터 및 노드 그룹을 생성 한 후 aws-auth를 조회 할 경우, 다음과 유사하다. (여러 노드 그룹을 생성한 경우, mapRoles에 그룹 만큼의 Role이 등록 되어 있다.)

```yaml
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::<ACCOUNT_ID>:role/<Node Instance Role>
      username: system:node:{{EC2PrivateDNSName}}
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
```

### 권한 등록

- 1. Bastion host 등 특정 Instance를 이용하여 Kubenetes를 관리할 경우

  다음과 같이 조회한 `aws-auth.yaml`의 `data`부분을 수정한 뒤 kubectl 명령어를 사용하여 적용한다. </br>
  kubernetes에서 제공하는 기본 역할 그룹에 대해서는 [사이트](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)를 참조한다. (예제의 system:masters는 `cluster-admin`이다.)

  ```yaml
  apiVersion: v1
  data:
    mapRoles: |
      - groups:
        - system:bootstrappers
        - system:nodes
        rolearn: arn:aws:iam::<ACCOUNT_ID>:role/<Node Instance Role>
        username: system:node:{{EC2PrivateDNSName}}
      ## groups에 IAM Role을 추가한다.
      - groups:
        - system:masters
        rolearn: arn:aws:iam::<ACCOUNT_ID>:role/<bastion instance profile>
        username: <BASTION_ADMIN>
  ... 생략 ...
  ```

  적용

  ```shell
  kubectl apply -f aws-auth.yaml
  ```

- 2. AWS User에게 권한을 부여하여 Kubernetes를 관리할 경우

  AWS User를 mapUsers에 추가한 후 kubectl 명령어를 사용하여 적용한다.

  ```yaml
  apiVersion: v1
  data:
    mapRoles: |
      - groups:
        - system:bootstrappers
        - system:nodes
        rolearn: arn:aws:iam::<ACCOUNT_ID>:role/<Node Instance Role>
        username: system:node:{{EC2PrivateDNSName}}
      ## groups에 IAM Role을 추가한다.
    mapUsers: |
      - groups:
        - system:masters
        rolearn: arn:aws:iam::<ACCOUNT_ID>:user/<AWS USER>
        username: <ADMIN>
  ... 생략 ...
  ```

## Service Account 및 Token을 이용하는 방법

- 1. Service account와 Kubeconfig을 통해 Kubernetes를 관리하는 경우

  이 경우는 주로 애플리케이션을 Kubernetes에 배포하는 `Spinnaker`와 같은 `CI/CD` 솔루션 또는 `Lens`등 등과 같은 Kubernetes dashboard에서 활용된다.

  kubeconfig 생성 방법

  1.1. 먼저 kubernetes의 cluster-admin 권한을 가진 사용자를 통해 `~/.kube/config`을 생성한다. (이미 존재할 경우 생략한다.)

  ```shell
  aws eks update-kubeconfig --name <CLUSTER NAME>
  ```

  1.2. Namespace 및 Service Account를 생성한 후 kubeconfig에 등록한다.

  ```shell
  #!/bin/bash
  export SERVICEACCOUNT=admin   # change service account name
  export NAMESPACE=admin        # change namespace

  CONTEXT=$(kubectl config current-context)
  kubectl create ns ${NAMESPACE}
  kubectl create sa ${SERVICEACCOUNT} -n ${NAMESPACE}
  kubectl create clusterrolebinding ${SERVICEACCOUNT}-cluster-admin-rolebinding \
      --clusterrole=cluster-admin \
      --serviceaccount=${NAMESPACE}:{SERVICEACCOUNT} \
      --namespace=${NAMESPACE}

  TOKEN=$(kubectl get secret --context ${CONTEXT} \
      $(kubectl get serviceaccount ${SERVICEACCOUNT} \
        --context ${CONTEXT} \
        -n ${NAMESPACE} \
        -o jsonpath='{.secrets[0].name}') \
      -n ${NAMESPACE} \
      -o jsonpath='{.data.token}' | base64 --decode)

  kubectl config set-credentials ${SERVICEACCOUNT}-token-user --token ${TOKEN}
  kubectl config set-context $CONTEXT --user ${SERVICEACCOUNT}-token-user
  ```

  1.3. 기존의 cluster/context/user를 kubeconfig에서 삭제한다.

  ```shell
  kubectl config get-clusters                 # 등록된 cluster 조회
  kubectl config delete-cluster <CLUSTER>     # 기존 cluster 삭제
  kubectl config get-contexts                 # 등록된 context 조회
  kubectl config delete-context <CONTEXT>     # 기존 context 삭제
  kubectl config get-users                    # 등록된 user 조회
  kubectl config delete-user <USER>           # 기존 user 삭제
  ```
