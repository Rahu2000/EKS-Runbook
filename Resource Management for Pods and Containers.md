# Resource Management for Pods and Containers

파드를 지정할 때, 컨테이너에 필요한 각 리소스의 양을 선택적으로 지정할 수 있다.

## Requests and Limits

- `requests` : 스케줄러에서 Pod를 어떤 노드에 위치 시킬 판단 근거가 되며 또한 hpa에서 Pod를 확장 또는 축소시킬 기준 값이 된다.
- `limits` : `limits`를 설정하면 해당 Container 또는 Pod는 limits을 넘어선 자원을 사용할 수 없다.

Pod의 각 Container는 다음을 설정할 수 있다.

- `spec.containers[].resources.limits.cpu`
- `spec.containers[].resources.limits.memory`
- `spec.containers[].resources.limits.hugepages-<size>`
- `spec.containers[].resources.requests.cpu`
- `spec.containers[].resources.requests.memory`
- `spec.containers[].resources.requests.hugepages-<size>`

안정적인 운영을 위해서 다음을 고려한다.

- 적절한 `spec.containers[].resources.requests.cpu`, `spec.containers[].resources.requests.memory`를 설정한다.
- 적절한 cpu와 memory는 성능테스트를 통해 도출한다.
- `spec.containers[].resources.requests.memory`와 동일한 `spec.containers[].resources.limits.memory`를 설정한다.
- memory의 경우 limits 제한이 없을 경우 `OOM`이 발생하기 쉽다.

설정 예제

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: images.my-company.example/app:v4
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: log-aggregator
    image: images.my-company.example/log-aggregator:v6
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```
