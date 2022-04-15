# Assigning Pods to Nodes

Pod의 특성 및 목적에 따라서 특정한 노드에 Pod를 할당하여 고가용성, 보안 등을 향상 시킬 수 있다.
여러 방식으로 제어할 수 있으나 일반적으로 레이블을 이용하여 Pod 할당을 제어한다.

## nodeSelector

`nodeSelector` 는 가장 간단하고 권장되는 노드 선택 제약 조건의 형태이다. `nodeSelector` 는 PodSpec의 필드로써 키-값 쌍의 매핑으로 지정한다. 이를 활용하기 위해서는 노드에 키-값의 쌍으로 표시되는 레이블을 각자 가지고 있어야 한다.

### 1. Node에 label 할당

```Shell
# command
kubectl label nodes <node id> key=value

# e.g.
kubectl label nodes kubernetes-foo-node-1.c.a-robinson.internal disktype=ssd
```

- 참고

```Shell
# delete label
kubectl label node <nodename> <labelname>-

# e.g.
kubectl label nodes kubernetes-foo-node-1.c.a-robinson.internal disktype-
```

### 2. Pod에 nodeSelector 설정

`spec.nodeSelector`에 `key: value`형식으로 설정한다.

```yaml
# e.g.
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd         # key: value
```

## 내장 labels

Kubernetes는 노드에 표준 레이블을 기본으로 할당한다. 이를 [잘 알려진 레이블](https://kubernetes.io/ko/docs/reference/labels-annotations-taints/)이라고 하며 스케줄러 등에서 활용한다.

`잘 알려진 레이블`은 Kubernetes의 버전에 따라 삭제되거나(이 경우 대체 레이블로 변경) 새롭게 추가되기도 하므로 kubernetes 버전 업데이트 등의 작업을 할 경우 반드시 확인해야 한다.

## EKS labels

EKS 관리형 노드 그룹의 경우 다음의 레이블이 자동으로 할당된다.

- 접두사 `eks.amazonaws.com`로 시작되는 레이블
  - e.g. `eks.amazonaws.com/capacityType: ON_DEMAND`

### 참고: 필수 Tag

EKS의 경우 노드 레이블과 별개로 필수 Tag의 설정이 필요하다. 이는 `autoscaler`에 특정 노드가 특정 Cluster에 종속되어 있음을 알리기 위함이다.

관리형 노드 그룹의 경우, 자동으로 할당되지만 고객 관리형 노드 또는 노드 그룹의 경우 수동으로 설정해야 한다. (autoscaler 또는 launch template에 tag의 추가 등록이 필요하다.)

|Tag Key|Tag Value|
|---|---|
|kubernetes.io/cluster/\<cluster-name>|owned|
