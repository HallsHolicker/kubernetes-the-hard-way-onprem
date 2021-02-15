# Provisioning Pod Network Routes

이번 실습에서는 Kubernetes CNI로 calico를 설치하도록 하겠습니다. 또한. External Loadbalancer를 구성하기 위해여 metalLB도 설치하도록 하겠습니다.

## Installing kubernetes CNI calico

이번 작업은 `k8s-client`에서 진행합니다. 

```
ssh k8s-client
```

Calico의 기본 cluster-cidr은 192.168.0.0/16입니다. 이번 실습에서는 10.200.0.0/16이므로 이 부분은 수정을 하도록 하겠습니다.

```
curl https://docs.projectcalico.org/manifests/calico.yaml -o calico.yaml

sed -i 's|# - name: CALICO_IPV4POOL_CIDR|- name: CALICO_IPV4POOL_CIDR|g' calico.yaml
sed -i 's|#   value: "192.168.0.0/16"|  value: "10.200.0.0/16"|g' calico.yaml

kubectl apply -f calico.yaml
```

## Verification Calico

Calico가 잘 동작 되는지 확인합니다.

```
ssh k8s-client
```

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
curl https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/namespace.yaml -o metallb_namespace.yaml
curl https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml -o metallb.yaml
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
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.72.102-192.168.72.103
EOF
```

## Verification MetalLB

MetalLB 설치가 잘 동작 되는 지 확인 합니다.

```
ssh k8s-client
```

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



Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
