# Deploying the DNS Cluster Add-on

이번 실습에서는 Kubernetes cluster안에서 실행되는 어플리케이션들에 대해서, [CoreDNS](https://coredns.io/)를 이용한 DNS 기반 service discovery를 하는 [DNS add-on](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)을 배포하겠습니다.

## The DNS Cluster Add-on


`coredns` add-on 배포:

```
ssh k8s-client
```

```
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns.yaml
```

> output

```
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.extensions/coredns created
service/kube-dns created
```

`kube-dns` deployment로 생성된 pods 리스트 확인:

```
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> output

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-699f8ddd77-94qv9   1/1     Running   0          20s
coredns-699f8ddd77-gtcgb   1/1     Running   0          20s
```

## Verification

`busybox` deployment 생성:

```
kubectl run --generator=run-pod/v1 busybox --image=busybox:1.28 --command -- sleep 3600
```

`busybox` deployment로 생성된 pod 리스트 확인:

```
kubectl get pods -l run=busybox
```

> output

```
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          3s
```

`busybox` pod의 full name 확인:

```
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

`kubernetes` service에서 `busybox` pod의 DNS lookup 실행:

```
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> output

```
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

Next: [Smoke Test](13-smoke-test.md)
