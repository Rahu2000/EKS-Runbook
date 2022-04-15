# Pods Spread

효율적인 리소스 사용 및 Pod 고가용성을 향상시키기 위해서 Affinity 등을 적절하게 설정한다.

## Affinity and anti-affinity

`Affinity`와 `AntiAffinity`는 `nodeSelector`의 확장 기능으로 볼 수 있다. 주요 특징은 다음과 같다.

- 어피니티/안티-어피니티 언어가 더 표현적이다. 언어는 논리 연산자인 AND 연산으로 작성된 정확한 매칭 항목 이외에 더 많은 매칭 규칙을 제공한다.
- `nodeSelector`와 달리 엄격한 요구 사항이 아닌 "유연한(soft)"/"선호(preference)" 규칙을 의미하므로 스케줄러가 규칙을 만족할 수 없더라도, Pod가 계속 스케줄되도록 한다.
- 노드 자체에 레이블을 붙이기보다는 노드(또는 다른 토폴로지 도메인)에서 실행 중인 다른 파드의 레이블을 제한할 수 있다. 이를 통해 어떤 파드가 함께 위치할 수 있는지와 없는지에 대한 규칙을 적용할 수 있다.

Affinity는 Node Affinity와 Pod Affinity로 구분된다.

## Node Affinity

`nodeAffinity`는 `nodeSelector`와 유사하다. (조금 더 확장된 기능을 제공한다.) 레이블 기반으로 Pod 스케줄을 제한한다.

- `requiredDuringSchedulingIgnoredDuringExecution` : `nodeSelector`와 비슷하며 엄격한 위치 제약 조건
- `preferredDuringSchedulingIgnoredDuringExecution` : `nodeSelector`보다 유연한 제약 조건
- `operator`로는 `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`을 사용할 수 있다.
- `NotIn`과 `DoesNotExist`는 antiAffinity에 사용할 수 있다.
- 또한 `Exists`, `DoesNotExist`는 `values`를 설정할 수 없다.
- `nodeAffinity` 유형과 연관된 `nodeSelectorTerms` 를 지정하면, `nodeSelectorTerms` 중 **하나라도 만족**시키는 노드에 파드가 스케줄된다.
- `nodeSelectorTerms` 와 연관된 여러 `matchExpressions` 를 지정하면, 파드는 **`matchExpressions` 를 모두 만족**하는 노드에만 스케줄된다.
- 파드가 스케줄된 노드의 레이블을 지우거나 변경해도 파드는 제거되지 않는다.
- `weight` 필드의 범위는 1-100이며 전체 점수가 가장 높은 노드를 가장 선호한다.

구성 예시

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```

위 예제에서 반드시 노드의 레이블 `kubernetes.io/e2e-az-name`의 값이 `e2e-az1`, `e2e-az2`에만 Pod가 배치된다.(만일 해당 레이블 값을 가진 노드가 하나도 없다면 Pod는 배포되지 못한다.) 그리고, 그 노드 중에서 레이블 `another-node-label-key`의 값이 `another-node-label-value`인 노드를 더욱 선호한다. (만일 해당 레이블 값을 가진 노드가 없어도 Pod는 배포 된다.)

## Pod Affinity

파드 어피니티는 노드에서 이미 실행 중인 파드 레이블을 기반으로 파드가 스케줄되도록 제약한다. 이를 통해 특정 파드들의 분산 또는 군집을 유도할 수 있다.

- 소규모 클러스터의 경우 Pod Affinity 설정을 통해 고가용성을 향상 시킬 수 있다.
- Pod Affinity는 상당한 양의 계산이 필요하기에 대규모 클러스터에서는 스케줄링 속도가 크게 느려질 수 있다. 따라서 **수백 개의 노드를 넘어가는 클러스터에서 이를 사용하지 말것**
- 클러스터의 모든 노드는 `topologyKey`와 매칭되는 적절한 레이블을 가지고 있어야 한다. `topologyKey`가 없는 노드가 존재할 경우, 의도치 않은 동작이 발생 할 수 있다.
- `requiredDuringSchedulingIgnoredDuringExecution` : 엄격한 규칙
- `preferredDuringSchedulingIgnoredDuringExecution` : 유연한 규칙
- 상황에 맞게 규칙을 지정한다. 일반적으로 Pod는 엄격한 규칙보다 유연한 규칙을 적용할 것을 추천한다.
- `requiredDuringSchedulingIgnoredDuringExecution`와 `preferredDuringSchedulingIgnoredDuringExecution`는 `topologyKey`의 빈 값을 허용하지 않는다.
- 위의 경우를 제외하고, `topologyKey`는 적법한 어느 레이블-키도 가능하다.

구성 예시

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: topology.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: topology.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: k8s.gcr.io/pause:2.0
```

## Pod Topology Spread Constraints

사용자는 토폴로지 분배 제약 조건을 사용해서 `region`, `zone`, `node` 그리고 기타 사용자-정의 토폴로지 도메인과 같이 장애-도메인으로 설정된 클러스터에 걸쳐 파드가 분산되는 방식을 제어할 수 있다. **Pod Affinity를 대체**하여 설정할 수 있다.

이 기능은 Kubernetes v1.19부터 활성화되었다.

### 필수 요소

- node label

### 설정

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  topologySpreadConstraints:
    - maxSkew: <integer>
      topologyKey: <string>
      whenUnsatisfiable: <string>
      labelSelector: <object>
```

- `maxSkew` : 토폴로지 조건에서 파드가 균등하게 배치되지 않아도 허용되는 최대치. 0보다 큰 수를 설정한다.
- `topologyKey` : 노드 레이블의 키
- `whenUnsatisfiable` : `DoNotSchedule`, `ScheduleAnyway` 설정할 수 있다.
- `labelSelector` : 파드 레이블 키
- 복수의 `topologySpreadConstraints`는 `AND` 조건으로 연결된다.

예를 들어 maxSkew에 1을 설정하고 topologyKey에 `zone`을 설정하면, 가령 노드가 a zone, b zone에 구성되었다면 a zone과 b zone에 동일한 파드를 배포될 때 영역간 최대 1의 불균형 배포를 허용한다. (a zone: 2, b zone: 1)

### 내부 기본 제약

Kubernetes v1.20부터 활성화되었다.

기본 설정 값은 다음과 같다.

```yaml
defaultConstraints:
  - maxSkew: 3
    topologyKey: "kubernetes.io/hostname"
    whenUnsatisfiable: ScheduleAnyway
  - maxSkew: 5
    topologyKey: "topology.kubernetes.io/zone"
    whenUnsatisfiable: ScheduleAnyway
```

호스트 당 최대 3, 영역별 5의 Pod 불균형 배치을 허용한다. (가능한 경우 더 큰 불균형도 허용된다.)
