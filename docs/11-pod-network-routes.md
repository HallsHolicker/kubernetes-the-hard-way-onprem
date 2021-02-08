# Provisioning Pod Network Routes

이번 실습에서는 Kubernetes CNI로 calico를 설치하도록 하겠습니다. 또한. External Loadbalancer를 구성하기 위해여 metalLB도 설치하도록 하겠습니다.

## Installing kubernetes CNI calico

이번 작업은 `k8s-client`에서 진행합니다. 

```
ssh k8s-client
```


```

curl https://docs.projectcalico.org/manifests/calico.yaml -o calico.yaml

sed -i 's|# - name: CALICO_IPV4POOL_CIDR|- name: CALICO_IPV4POOL_CIDR|g' calico.yaml
sed -i 's|#   value: "192.168.0.0/16"|  value: "10.200.0.0/16"|g' calico.yaml

kubectl apply -f calico.yaml
```

## Verification

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

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
