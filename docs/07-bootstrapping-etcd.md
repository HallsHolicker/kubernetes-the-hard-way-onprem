# Bootstrapping the etcd Cluster

Kubernetes 구성요소는 stateless하고 [etcd](https://github.com/etcd-io/etcd)에 클러스터 상태를 저장합니다. 이번 실습에서는 고가용성을 위해 3개의 노드에 etcd 클러스터를 부트스트랩 및 구성하고 보안 원격 접속을 설정합니다.


## Prerequisites

이번 실습 명령어는 각 Controller node에서 진행해야 합니다. 그래서 각 node에 ssh로 접속해서 진행합니다.
Controller node: `k8s-controller-1`, `k8s-controller-2`, `k8s-controller-3`

```
ssh k8s-controller-1
```

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki)를 사용하여 동시에 여러 node에 같은 명령어르 실행할 수 있습니다.  이 실습을 진행하기 전에 [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux)를 보시면 좋습니다.

## Bootstrapping an etcd Cluster Member

### Download and Install the etcd Binaries

[etcd](https://github.com/etcd-io/etcd) 깃허브에서 공식 버전을 다운로드 합니다.

```
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.5/etcd-v3.4.5-linux-amd64.tar.gz"
```

압축을 풀고 `etcd` 서버와 `etcdctl` 명령어 유틸리티를 설치합니다.

```
{
  sudo tar -xvf etcd-v3.4.5-linux-amd64.tar.gz
  sudo mv etcd-v3.4.5-linux-amd64/etcd* /usr/local/bin/
}
```

### Configure the etcd Server

```
{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
}
```

클라이언트 요청을 처리하고 etcd 클러스터 피어와 통신하는데 사용하는 INTERNAL IP 정보를 설정합니다.

```
INTERNAL_IP=$(grep "$(hostname)" /etc/hosts | awk '{print $1}')
```

각 etcd 구성원은 etcd 클러스터 내에서 고유 한 이름을 가져야 합니다. 고유 한 이름은 현재 서버의 이름과 동일하게 설정합니다.

```
ETCD_NAME=$(hostname -s)
```

`etcd.service` systemd unit file 생성:

```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster k8s-controller-1=https://10.240.0.11:2380,k8s-controller-2=https://10.240.0.12:2380,k8s-controller-3=https://10.240.0.13:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the etcd Server

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
}
```

> 위의 명령어들은 각 controller node에서 실행해야 합니다. controller node : `k8s-controller-1`, `k8s-controller-2`, `k8s-controller-3`

## Verification

etcd 클러스터 멤버 정보:

```
ETCDCTL_API=3
sudo /usr/local/bin/etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

> output

```
2fe2f5d17fc97dab, started, k8s-controller-3, https://10.240.0.13:2380, https://10.240.0.13:2379, false
3a57933972cb5131, started, k8s-controller-2, https://10.240.0.12:2380, https://10.240.0.12:2379, false
ffed16798470cab5, started, k8s-controller-1, https://10.240.0.11:2380, https://10.240.0.11:2379, false
```

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
