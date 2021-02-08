# Generating Kubernetes Configuration Files for Authentication

이번 실습에서는 kubeconfigs로 불리는 [Kubernetes configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)를 생성하여, 클라이언트가 Kubernetes API Servers를 찾고 인증할 수 있도록 합니다.

## Client Authentication Configs

이번 섹션에서는 `controller manager`, `kubelet`, `kube-proxy`, `scheduler`와 `admin`을 위한 kubeconfig 파일을 생성합니다.

### Kubernetes Public IP Address

각 kubeconfig는 Kubernetes API Server와의 접속이 필요합니다. Kubernetes API Server를 사용함에 있어서 고가용성을 구성하기 위해 VIP (`Keepalived + Haproxy`)를 사용하였습니다.
Keepalived와 Haproxy는 [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)에서 다루겠습니다.

### The kubelet Kubernetes Configuration File

Kubelet을 위한 kubeconfig 파일을 생성 할 때 kubelet의 노드 이름과 일치하는 클라이언트 인증서를 사용해야 합니다. 이렇게 하면 kubelet이 kubernetes [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/)에 의해 인증이 됩니다.


> 다음 명령어들은 SSL 인증서를 만들었던 실습인 [Generating TLS Certificates](04-certificate-authority.md)와 디렉토리와 같은 곳에서 실행하셔야 합니다.

각 worker node의 kubeconfig file 생성:

```
for hostname in k8s-worker-1 k8s-worker-2 k8s-worker-3; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${hostname}.kubeconfig

  kubectl config set-credentials system:node:${hostname} \
    --client-certificate=${hostname}.pem \
    --client-key=${hostname}-key.pem \
    --embed-certs=true \
    --kubeconfig=${hostname}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${hostname} \
    --kubeconfig=${hostname}.kubeconfig

  kubectl config use-context default --kubeconfig=${hostname}.kubeconfig
done

```

Results:

```
k8s-worker-1.kubeconfig
k8s-worker-2.kubeconfig
k8s-worker-3.kubeconfig
```

### The kube-proxy Kubernetes Configuration File

`kube-proxy` 서비스를 위한 kubeconfig 파일 생성:

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

}
```

Results:

```
kube-proxy.kubeconfig
```

### The kube-controller-manager Kubernetes Configuration File

`kube-controller-manager` 서비스를 위한 kubeconfig 파일 생성:

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig

}
```

Results:

```
kube-controller-manager.kubeconfig
```


### The kube-scheduler Kubernetes Configuration File

`kube-scheduler` 서비스 kubeconfig 파일 생성:

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig

}
```

Results:

```
kube-scheduler.kubeconfig
```

### The admin Kubernetes Configuration File

`admin` 유저를 위한 kubeconfig 파일 생성:

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig

}
```

Results:

```
admin.kubeconfig
```


## 

## Distribute the Kubernetes Configuration Files

각 worker node에 알맞는 `kubelet`와 `kube-proxy`의 kubeconfig 파일을 복사 합니다.

```
for hostname in k8s-worker-1 k8s-worker-2 k8s-worker-3; do
  scp ${hostname}.kubeconfig kube-proxy.kubeconfig ${hostname}:~/
done

```
각 Controller node에 알맞는 `kube-controller-manager`와 `kube-scheduler`의 kubeconfig 파일을 복사 합니다.

```
for hostname in k8s-controller-1 k8s-controller-2 k8s-controller-3; do
  scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${hostname}:~/
done

```

Next: [Generating the Data Encryption Config and Key](06-data-encryption-keys.md)
