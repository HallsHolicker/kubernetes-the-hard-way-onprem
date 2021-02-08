# Bootstrapping the Kubernetes Worker Nodes

이번 실습은 Kubernete worker node들을 부트스트랩 하겠습니다. 각 노드에 다음 구성요소를 설치하겠습니다.
CNI의 경우는 calico를 설치할 것이며, 설치는 [Provisioning Pod Network Routes](11-pod-network-routes.md)에서 진행하도록 하겠습니다.
* [runc](https://github.com/opencontainers/runc)
* [container networking plugins](https://github.com/containernetworking/cni)
* [containerd](https://github.com/containerd/containerd)
* [kubelet](https://kubernetes.io/docs/admin/kubelet)
* [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies)

## Prerequisites

이번 실습 명령어는 각 Worker node에서 진행해야 합니다. 그래서 각 node에 ssh로 접속해서 진행합니다.
Worker node: `k8s-worker-1`, `k8s-worker-2`, `k8s-worker-3`

```
ssh k8s-worker-1
```

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki)를 사용하여 동시에 여러 node에 같은 명령어르 실행할 수 있습니다.  이 실습을 진행하기 전에 [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux)를 보시면 좋습니다.

## Provisioning a Kubernetes Worker Node

의존성 설치:

```
{
  sudo dnf -y install socat conntrack

}
```

> socat은 `kubectl port-forward` 명령어를 지원하기 위함입니다.

### Disable Swap

기본적으로 [swap](https://help.ubuntu.com/community/SwapFaq)이 켜져있으면, kubelet은 시작되지 않습니다. Kubernetes는 적절한 리소스 할당 및 서비스 품질을 제공할 수 있도록 swap를 비활성화 하는 것을 권고 합니다. [recommended](https://github.com/kubernetes/kubernetes/issues/7294)

swap이 활성화되어 있는지 확인:

```
sudo swapon --show
```

출력이 빈 화면일 경우 비활성화 상태입니다. 만약 swap이 활성화 상태라면 다음 명령어를 입력하여 swap을 즉시 비활성화 합니다.:

```
sudo swapoff -a
```

### Download and Install Worker Binaries

```
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.15.0/crictl-v1.15.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc8/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.8.2/cni-plugins-linux-amd64-v0.8.2.tgz \
  https://github.com/containerd/containerd/releases/download/v1.2.9/containerd-1.2.9.linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubelet

```

각 구성요소의 디렉토리 생성:

```
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes

```

각 구성요소 설치:

```
{
  mkdir containerd
  tar -xvf crictl-v1.15.0-linux-amd64.tar.gz
  tar -xvf containerd-1.2.9.linux-amd64.tar.gz -C containerd
  sudo tar -xvf cni-plugins-linux-amd64-v0.8.2.tgz -C /opt/cni/bin/
  sudo mv runc.amd64 runc
  chmod +x crictl kubectl kube-proxy kubelet runc 
  sudo mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
  sudo mv containerd/bin/* /bin/

}
```

### Configure containerd

`containerd` 구성파일 생성:

```
sudo mkdir -p /etc/containerd/

```

```
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
EOF

```

`containerd.service` systemd 파일 생성

```
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF

```

### Configure the Kubelet

```
{
  sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
  sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
  sudo mv ca.pem /var/lib/kubernetes/

}
```

`kubelet-config.yaml` 구성 파일 생성:

```
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF

```

> `resolvConf` 구성은 `systemd-resolved`가 동작하는 시스템에서 service discovery를 위해 CoreDNS를 사용할 때 루프가 되는 것을 방지하기 위해 사용합니다.

`kubelet.service` systemd 파일 생성:

```
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

```

### Configure the Kubernetes Proxy

```
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig

```

`kube-proxy-config.yaml` 구성 파일 생성:

```
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF

```

`kube-proxy.service` systemd 파일 생성

```
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

```

### Start the Worker Services

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable containerd kubelet kube-proxy
  sudo systemctl start containerd kubelet kube-proxy

}
```

> 각 worker node에서 위의 명령어들을 실행해야 합니다. Worker node: `k8s-worker-1`, `k8s-worker-2`, `k8s-worker-3`.

## Verification

Kubernetes nodes 등록 상태 확인:

```
  ssh k8s-client
  kubectl get nodes --kubeconfig admin.kubeconfig
```

> output

```
NAME           STATUS     ROLES    AGE    VERSION
k8s-worker-1   NotReady   <none>   109s   v1.15.3
k8s-worker-2   NotReady   <none>   109s   v1.15.3
k8s-worker-3   NotReady   <none>   109s   v1.15.3
```

아직 네트워크 CNI를 설정하지 않았기 때문에 NotReady 상태로 표시 됩니다.

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
