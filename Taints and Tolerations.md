# Taints and Tolerations

`Taints`는 `Affinity`와 반대로, 노드가 파드 셋을 제외하는 데 적용한다.

`Tolerations`은 파드에 설정하며, `Tolerations`과 일치하는 `Taints`가 있는 노드에 파드가 스케줄 될 수 있게 한다.

`Taints` 설정을 기반으로 `Dedicated 노드`를 구성할 수 있다.

## Taints Command

Taints 조회

```Shell
kubectl get nodes -o json | jq '.items[]|{name:.metadata.name, taints:.spec.taints}'
```

Taints 추가

```Shell
kubectl taint nodes <Node Name> key1=value1:NoSchedule
```

Taints 삭제

```Shell
kubectl taint nodes <Node Name> key1=value1:NoSchedule-
```

## Tolerations Configure

- 특정 value 값

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
```

- 모든 value 값

```yaml
tolerations:
- key: "key1"
  operator: "Exists"
  effect: "NoSchedule"
```

설정 예시

```yaml
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
  tolerations:
  - key: "example-key"
    operator: "Exists"
    effect: "NoSchedule"
```

- `operator`가 `Exists`에 key가 생략된 경우, 모든 `Taints`에 대해 `tolerations` 된다.
- `effect`가 생략된 경우, 모든 이펙트를 키 key1 와 일치시킨다.

## Taints 기반 Pod 축출

`NoExecute` Taints 효과를 설정하여 실행 중인 Pod를 노드에서 축출할 수 있다.

- `Taints`를 용인하지 않는 파드는 즉시 축출된다.
- `tolerations`명세에 `tolerationSeconds`를 지정하지 않고 `Taints`를 용인하는 파드는 계속 실행된다.
- `tolerationSeconds`가 지정된 `Taints`를 용인하는 파드는 지정된 시간 동안 실행된다.

## 기본 내장 Taints

다음은 Kubernetes(Kubelet)가 기본으로 설정 및 관리하는 Taints이다. 해당 Taints가 설정된 노드는 문제가 발생한 노드 이므로 상황에 맞게 대응해야 한다.

- `node.kubernetes.io/not-ready` : 노드가 준비되지 않았다. 이는 NodeCondition Ready 가 "False"로 됨에 해당한다.
- `node.kubernetes.io/unreachable` : 노드가 노드 컨트롤러에서 도달할 수 없다. 이는 NodeCondition Ready 가 "Unknown"로 됨에 해당한다.
- `node.kubernetes.io/memory-pressure` : 노드에 메모리 할당 압박이 있다.
- `node.kubernetes.io/disk-pressure` : 노드에 디스크 할당 압박이 있다.
- `node.kubernetes.io/pid-pressure` : 노드에 PID 할당 압박이 있다.
- `node.kubernetes.io/network-unavailable` : 노드의 네트워크를 사용할 수 없다.
- `node.kubernetes.io/unschedulable` : 노드를 스케줄할 수 없다.
- `node.cloudprovider.kubernetes.io/uninitialized` : "외부" 클라우드 공급자로 kubelet을 시작하면, 이 테인트가 노드에서 사용 불가능으로 표시되도록 설정된다. 클라우드-컨트롤러-관리자의 컨트롤러가 이 노드를 초기화하면, kubelet이 이 테인트를 제거한다.
