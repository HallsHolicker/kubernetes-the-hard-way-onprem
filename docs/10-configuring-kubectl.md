# Configuring kubectl for Remote Access

이번 실습에서는 `admin` 자격 증명으로 `kubectl` 명령 유틸리티에 대한 kubeconfig 파일을 생성합니다.

> admin 클라이언트 인증서를 생성하는 디렉토리에서 명령을 실행합니다.

## The Admin Kubernetes Configuration File

각 kubeconfig는 Kubernetes API 서버에 연결하기 위해 필요합니다. 고가용성을 지원하기 위해 Kubernetes API 서버 접속을 Haproxy에 설정된 VIP를 사용합니다.


`admin` 사용자로 인증하기에 적합한 kubeconfig 파일 생성:

```
ssh k8s-client
```

```
{
  KUBERNETES_PUBLIC_ADDRESS=$(grep "k8s-controller$" /etc/hosts | awk '{print $1}')

  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
}
```

## Verification

Check the health of the remote Kubernetes cluster:

```
kubectl version
```

> output

```
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:25:06Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
```

```
kubectl get componentstatuses
```

> output

```
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-1               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
etcd-0               Healthy   {"health":"true"}
```

List the nodes in the remote Kubernetes cluster:

```
kubectl get nodes
```

> output

```
NAME           STATUS     ROLES    AGE   VERSION
k8s-worker-1   NotReady   <none>   2m9s  v1.21.0
k8s-worker-2   NotReady   <none>   2m9s  v1.21.0
k8s-worker-3   NotReady   <none>   2m9s  v1.21.0
```

Next: [Provisioning Pod Network Routes](11-pod-network-routes.md)
