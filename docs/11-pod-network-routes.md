# Provisioning Pod Network Routes

이번 실습에서는 Kubernetes CNI로 calico를 설치하도록 하겠습니다. 또한. External Loadbalancer를 구성하기 위해여 metalLB도 설치하도록 하겠습니다.

## Installing kubernetes CNI calico

이번 작업은 `k8s-client`에서 진행합니다. 

```
ssh k8s-client
```

Download Calico manifests

```
curl https://docs.projectcalico.org/manifests/calico.yaml -o calico.yaml
```

Calico Version check

```
 less calico.yaml | grep image
```

> output

```
          image: docker.io/calico/cni:v3.19.1
          image: docker.io/calico/cni:v3.19.1
          image: docker.io/calico/pod2daemon-flexvol:v3.19.1
          image: docker.io/calico/node:v3.19.1
          image: docker.io/calico/kube-controllers:v3.19.1
```


Calico의 기본 cluster-cidr은 192.168.0.0/16입니다. 이번 실습에서는 10.200.0.0/16이므로 이 부분은 수정을 하도록 하겠습니다.

```
sed -i 's|# - name: CALICO_IPV4POOL_CIDR|- name: CALICO_IPV4POOL_CIDR|g' calico.yaml
sed -i 's|#   value: "192.168.0.0/16"|  value: "10.200.0.0/16"|g' calico.yaml

kubectl apply -f calico.yaml
```

## Verification Calico

Calico가 정상 동작하기까지는 약 2~3분 정도 소요됩니다.
무선 환경에서 배포할 경우 인터넷 환경에 따라 1시간 이상이 소요될 수도 있습니다. -0-

```
kubectl get pods -n kube-system
```

> output

```
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-785fc6b5f6-bgkdk   1/1     Running   0          10m
calico-node-sw6q8                          1/1     Running   0          10m
calico-node-txwlt                          1/1     Running   0          10m
calico-node-x8hk6                          1/1     Running   0          10m
```

## Install MetalLB

```
ssh k8s-client
```

```
curl https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml -o metallb_namespace.yaml
curl https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml -o metallb.yaml
```

```
kubectl apply -f metallb_namespace.yaml
```

```
kubectl apply -f metallb.yaml
```

다음 명령어는 최초 설치시에만 실행합니다.

```
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

MetalLB ConfigMap 생성:

```
cat << EOF | kubectl create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    peers:
    - peer-address: 10.240.0.100
      peer-asn: 64501
      my-asn: 64500
    address-pools:
    - name: default
      protocol: bgp
      addresses:
      - 192.168.72.0/24
EOF
```

## Verification MetalLB

MetalLB 설치가 잘 동작 되는 지 확인 합니다.

```
kubectl get pods -n metallb-system
```

> output

```
NAME                          READY   STATUS    RESTARTS   AGE
controller-6568695fd8-mlj6t   1/1     Running   0          20m
speaker-5lh6r                 1/1     Running   0          20m
speaker-65cz9                 1/1     Running   0          20m
speaker-gfcpj                 1/1     Running   0          20m
```



## FRRouting 설치

MetalLB를 BGP모드로 동작시키고 있으며, BGP로 전파가 제대로 되는지 확인 하기 위해 FRRouting을 `k8s-client`에 설치해서 BGP를 연결해 봅니다.

```
FRRVER="frr-stable"
curl -O https://rpm.frrouting.org/repo/$FRRVER-repo-1-0.el8.noarch.rpm
sudo dnf -y install ./$FRRVER*
sudo dnf -y install frr frr-pythontools
```

BGP Daemon 사용
```
sudo sed -i 's/^bgpd=no/bgpd=yes/' /etc/frr/daemons
sudo sed -i 's/^vtysh_enable=no/vtysh_enable=yes/' /etc/frr/daemons
sudo sed -i 's/^#MAX_FDS=1024/MAX_FDS=1024/' /etc/frr/daemons
sudo sed -i 's/^zebra_options="  -A 127.0.0.1 -s 90000000"/zebra_options=" --daemon -A 10.240.0.100 -s 90000000"/' /etc/frr/daemons
sudo sed -i 's/^bgpd_options="   -A 127.0.0.1"/bgpd_options=" --daemon -A 10.240.0.100"/' /etc/frr/daemons
```

BGP 설정
```
cat << EOF | sudo tee /etc/frr/frr.conf
frr version 8.0
frr defaults traditional
hostname localhost.localdomain
log syslog informational
no ipv6 forwarding
!
router bgp 64501
 no bgp ebgp-requires-policy
 neighbor k8s peer-group
 neighbor k8s remote-as 64500
 bgp listen range 10.240.0.0/24 peer-group k8s
 !
 address-family ipv4 unicast
  neighbor k8s soft-reconfiguration inbound
 exit-address-family
!
line vty
!
EOF
```

FRR 재시작
```
sudo systemctl restart frr
```



Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
