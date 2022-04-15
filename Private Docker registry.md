# Private docker registry

## 1. Docker registry 도메인 등록

### Linux

### Docker registry DNS 서버가 없는 경우

먼저 hosts 파일에 docker registry address를 등록한다.

다음의 명령어를 root 권한을 가진 사용자로 실행한다.

```Shell
echo 'sample.docker.registry.io  10.100.10.100' >> /etc/hosts
```

### Docker registry DNS 서버가 있는 경우

#### `/etc/resolv.conf`가 자동 생성되는 경우

자동 생성 도구에 따라 다르므로 여기서는 AWS Linux2를 기반으로 설명한다.

AWS Linux2의 경우 dhclient에 의해 `/etc/resolv.conf`가 자동생성된다.

특정 DNS로 변경하기 위해서는 `/etc/sysconfig/network-script/ifcfg-XXX`를 수정한다.

보통 `XXX`는 `eth0`이나 ls 명령어로 사전에 확인한다.

다음의 명령어를 root 권한을 가진 사용자로 실행한다.

```Shell
echo 'DNS1=10.100.10.2' >> /etc/sysconfig/network-script/ifcfg-eth0  # DNS1
echo 'DNS2=10.100.10.2' >> /etc/sysconfig/network-script/ifcfg-eth0  # DNS2
```

서버가 재시작되면 `/etc/resolv.conf`에 적용된다. **이 경우 Kubernetes의 기본 DNS를 덮어쓴다.**

#### `/etc/resolv.conf`가 자동으로 생성되지 않는 경우

`/etc/resolv.conf`에 다음을 등록한다.

```text
nameserver 10.100.10.2 # DNS 서버 IP
```

Kubernetes의 기본 DNS이외에 프라이빗 DNS를 추가한다.

## 2. Runtime configure 수정

Runtime 환경에 따라서 private registry 설정이 필요하다.
Kubernetes의 경우, v1.22 이후 버전에서에서는 더 이상 docker runtime(cri-o를 지원)을 지원하지 않으므로 실제 구성 된 runtime에 맞는 설정을 해야한다.

예를 들면 EKS의 경우 v1.22에서 cri-containerd로 변경된다.

### Docker runtime

- insecure registry

  linux의 경우 `/etc/docker/daemon.json`의 파일을 다음과 같이 수정한다.

  ```json
  {
    "insecure-registries" : ["myregistrydomain.com:5000"]
  }
  ```

  수정 후, 다음을 실행한다.

  ```Shell
  systemctl restart docker
  ```

### Containerd runtime

- insecure registry

  linux의 경우 `/etc/containerd/config.toml`을 수정한다.

  예제는 다음을 참고한다.

  ```yaml
  # change <IP>:5000 to your registry url

  [plugins."io.containerd.grpc.v1.cri".registry]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."<IP>:5000"]
        endpoint = ["http://<IP>:5000"]
    [plugins."io.containerd.grpc.v1.cri".registry.configs]
      [plugins."io.containerd.grpc.v1.cri".registry.configs."<IP>:5000".tls]
        insecure_skip_verify = true
  ```

  수정 후, 다음을 실행한다.

  ```Shell
  systemctl restart containerd
  ```
