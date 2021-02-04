# Bootstrapping the Kubernetes Control Plane

이번 실습은 고가용성을 위해 Kubernetes control plane을 3개의 node에 부트스트랩 합니다.

## Prerequisites

이번 실습 명령어는 각 Controller node에서 진행해야 합니다. 그래서 각 node에 ssh로 접속해서 진행합니다.
Controller node: `k8s-controller-1`, `k8s-controller-2`, `k8s-controller-3`

```
ssh k8s-controller-1
```

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki)를 사용하여 동시에 여러 node에 같은 명령어르 실행할 수 있습니다.  이 실습을 진행하기 전에 [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux)를 보시면 좋습니다.

## Provision the Kubernetes Control Plane

kubernetes 디렉토리 생성:

```
sudo mkdir -p /etc/kubernetes/config
```

### Download and Install the Kubernetes Controller Binaries

Kubernetes 공식 바이너리 파일 다운로드:

```
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kubectl"
```

Kubernetes 설치:

```
{
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}
```

### Configure the Kubernetes API Server

```
{
  sudo mkdir -p /var/lib/kubernetes/

  sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/
}
```

INTERNAL IP 주소는 API서버를 클러스터 멤버들에게 알리는데 사용됩니다. 

```
INTERNAL_IP=$(grep "$(hostname)" /etc/hosts | awk '{print $1}')
```

`kube-apiserver.service` systemd 파일 생성:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.240.0.11:2379,https://10.240.0.12:2379,https://10.240.0.13:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all=true \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Controller Manager

`kube-controller-manage`의 kubeconfig를 이동합니다.

```
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

`kube-controller-manager.service` systemd 파일 생성:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Scheduler

`kube-scheduler`의 kubeconfig를 이동합니다.

```
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

`kube-scheduler.yaml` 파일을 생성:

```
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

`kube-scheduler.service` systemd 파일 생성:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the Controller Services

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

> 약 10초 정도 기다리면 Kubernetes API Server가 동작합니다.

### Verification

```
kubectl get componentstatuses --kubeconfig admin.kubeconfig
```

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

> 각 Controller node에서 실행을 해야 합니다. Controller node: `k8s-controller-1`, `k8s-controller-2`, `k8s-controller-3`

## RBAC for Kubelet Authorization

이번 실습은 각 worker node의 Kubelet API가 kubernetes API 서버에 접속을 허용하기 위한 RBAC 구성을 진행합니다. Metrics, logs를 검색하고 포드에 명령을 실행하려면 kubelet API에 대한 access 권한이 필요합니다.


> 이 튜토리얼은 kubelet `--authorization-mode` 옵션을 `webhook`으로 설정합니다. Webhook 모드는 [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) API를 사용하여 인증을 결정합니다.

이번 섹션 명령어는 Controller node 중 한 곳에서 실행하여도 전체 클러스터에 영향을 미칩니다.

```
 ssh k8s-controller-1
```

Kubelet API에 access하고 포드 관리와 관련된 일반적인 대부분의 작업을 수행 할 권한이 있는 `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole)을 만듭니다.  

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

Kubernetes API Server는 `--kubelet-client-certificate`에 정의된 클라이언트 인증서를 사용하여 kubelet에 `kubernetes` 사용자 이름으로 인증합니다.

Bind the `system:kube-apiserver-to-kubelet` ClusterRole to the `kubernetes` user:

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

## The Kubernetes Frontend Load Balancer

이번 섹션은 Kubernetes API Server의 고가용성을 위해 `Keepalived` + `Haproxy`를 이용해서 endpoint에 사용되는 VIP 및 Load balancer를 설정하겠습니다.


### Install keepalived

Download and Install keepalived
Create the external load balancer network resources:

```
{

  sudo dnf -y install openssl-devel libnl3-devel

  wget -q --show-progress --https-only --timestamping \
   https://www.keepalived.org/software/keepalived-2.2.1.tar.gz
  
  tar -xvf keepalived-2.2.1.tar.gz

  cd keepalived-2.2.1
  ./configure --prefix=/usr/local/
  sudo make && sudo make install

  sudo cp ./keepalived/keepalived /usr/sbin/
}
```

/etc/sysconfig/keepalibved을 생성합니다.

```
cat <<EOF | sudo tee /etc/sysconfig/keepalived
# Options for keepalived. See 'keepalived --help' output and keepalived(8) and
# keepalived.conf(5) man pages for a list of all options. Here are the most
# common ones :
#
# --vrrp               -P    Only run with VRRP subsystem.
# --check              -C    Only run with Health-checker subsystem.
# --dont-release-vrrp  -V    Dont remove VRRP VIPs & VROUTEs on daemon stop.
# --dont-release-ipvs  -I    Dont remove IPVS topology on daemon stop.
# --dump-conf          -d    Dump the configuration data.
# --log-detail         -D    Detailed log messages.
# --log-facility       -S    0-7 Set local syslog facility (default=LOG_DAEMON)
#

KEEPALIVED_OPTIONS="-D -S 3 -f /etc/keepalived/keepalived.conf"
EOF
```

keepalived 디렉토리 복사:

```

sudo cp -R ./keepalived/etc/keepalived /etc/keepalived

```

keepalived config 생성:

```
cat <<EOF | sudo tee /etc/keepalived/keepalived.conf



https://github.com/acassen/keepalived/issues/1175

sudo systemctl enable keepalived
sudo systemctl start keepalived

```
### Verification

> The compute instances created in this tutorial will not have permission to complete this section. **Run the following commands from the same machine used to create the compute instances**.

Retrieve the `kubernetes-the-hard-way` static IP address:

```
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

Make a HTTP request for the Kubernetes version info:

```
curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version
```

> output

```
{
  "major": "1",
  "minor": "15",
  "gitVersion": "v1.15.3",
  "gitCommit": "2d3c76f9091b6bec110a5e63777c332469e0cba2",
  "gitTreeState": "clean",
  "buildDate": "2019-08-19T11:05:50Z",
  "goVersion": "go1.12.9",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Next: [Bootstrapping the Kubernetes Worker Nodes](09-bootstrapping-kubernetes-workers.md)
