# Provisioning a CA and Generating TLS Certificates

이번 실습은 CloudFlare의 PKI toolkit인 [cfssl](https://github.com/cloudflare/cfssl)를 사용하여 [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure)를 만들도록 하겠습니다. 만들어지는 파일들은 인증기관(Certificate Authority) 부트스트랩 및 다음 구성 요소의 TLS 인증에 사용 됩니다.
구성요소 :
* etcd
* kube-apiserver
* kube-controller-manager
* kube-scheduler
* kubelet
* kube-proxy

## Certificate Authority

이번 섹션에서는 추가적인 TLS 인증서를 생성하는데 필요한 인증기관(Certificate Authority)을 구성합니다.
CA 설정 파일, 인증서, 개인키를 생성합니다.
```
{

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
```

Results:

```
ca-key.pem
ca.pem
```

## Client and Server Certificates

이번 섹션에서는 각각의 kubernetes 구성요소들의 client 및 서버 인증서와 kubernetes `admin` 유저에 대한 client의 인증서를 만듭니다.

### The Admin Client Certificate

`admin`유저의 client 인증서와 개인키를 생성합니다.

```
{

cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

}
```

Results:

```
admin-key.pem
admin.pem
```

### The Kubelet Client Certificates

Kubernetes는 Node Authorizer라고 불리는 [special-purpose authorization mode](https://kubernetes.io/docs/admin/authorization/node/)를 사용하여, [Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet)에서 만든 API 요청을 승인합니다. Node Authorizer의 승인을 받으려면 kubelet은 `system:node:<nodeName>` 이름으로 `system:nodes` 그룹에 속해 있다는 것을 식별할 수 있는 자격증명을 사용해야 합니다. 
이번 섹션에서는 Node Authorizer의 요구 사항을 충족하는 각 Kubernetes worker node에 대한 인증서를 작성합니다.

각 Kubernetes worker node의 Private key와 certificate를 생성:

```
for hostname in k8s-worker-1 k8s-worker-2 k8s-worker-3; do
cat > ${hostname}-csr.json <<EOF
{
  "CN": "system:node:${hostname}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

INTERNAL_IP=$(grep "${hostname}" /etc/hosts | awk '{print $1}')
EXTERNAL_IP=$(grep "k8s-controller$" /etc/hosts | awk '{print $1}')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${hostname},${INTERNAL_IP},${EXTERNAL_IP} \
  -profile=kubernetes \
  ${hostname}-csr.json | cfssljson -bare ${hostname}
done

```

Results:

```
k8s-worker-1-key.pem
k8s-worker-1.pem
k8s-worker-2-key.pem
k8s-worker-2.pem
k8s-worker-3-key.pem
k8s-worker-3.pem
```

### The Controller Manager Client Certificate

Generate the `kube-controller-manager` client certificate and private key:

```
{
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
```

Results:

```
kube-controller-manager-key.pem
kube-controller-manager.pem
```


### The Kube Proxy Client Certificate

Generate the `kube-proxy` client certificate and private key:

```
{

cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
```

Results:

```
kube-proxy-key.pem
kube-proxy.pem
```

### The Scheduler Client Certificate

Generate the `kube-scheduler` client certificate and private key:

```
{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
```

Results:

```
kube-scheduler-key.pem
kube-scheduler.pem
```


### The Kubernetes API Server Certificate

원격 클라이언트의 유효성을 인증하기 위해 Kubernetes API 서버 인증서에 Client IP, kubernetes API VIP 정보를 포함합니다.

Generate the Kubernetes API Server certificate and private key:

```
{

KUBERNETES_PUBLIC_ADDRESS=$(grep "k8s-controller$" /etc/hosts | awk '{print $1}')

KUBERNETES_CLIENT_ADDRESS=$(grep "k8s-client" /etc/hosts | awk '{print $1}')

KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,10.240.0.13,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES},${KUBERNETES_CLIENT_ADDRESS} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
```

> Kubernetes API Server는 [control plane bootstrapping](08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-api-server) 실습에서 내부 cluster services를 위해 할당하는 (`10.32.0.0/24`) 대역 범위에서 가장 첫번째 IP address인 (`10.32.0.1`)을 `kubernetes 내부 DNS에 자동으로 할당을 합니다.

Results:

```
kubernetes-key.pem
kubernetes.pem
```

## The Service Account Key Pair

Kubernetes Controller Manager는 [managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/) 문서에서 설명하듯이 키쌍을 사용하여, 서비스 계정 토큰을 생성하고 서명합니다.

Generate the `service-account` certificate and private key:

```
{

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}
```

Results:

```
service-account-key.pem
service-account.pem
```


## Distribute the Client and Server Certificates

각 Worker node에 certificate와 Private key를 복사합니다:

```
for hostname in k8s-worker-1 k8s-worker-2 k8s-worker-3; do
  scp ca.pem ${hostname}-key.pem ${hostname}.pem ${hostname}:~/
done

```

각 Controller node에 certificate와 Private key를 복사합니다:

```
for hostname in k8s-controller-1 k8s-controller-2 k8s-controller-3; do
  scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ${hostname}:~/
done

```

> `kube-proxy`, `kube-controller-manager`, `kube-scheduler`, `kubelet` client certificates는 다음 실습에서 클라이언트 인증서 구성 파일을 생성하겠습니다.

Next: [Generating Kubernetes Configuration Files for Authentication](05-kubernetes-configuration-files.md)
