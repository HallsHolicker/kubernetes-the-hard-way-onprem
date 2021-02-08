# Installing the Client Tools

이번 실습에서는 Kubernetes 구성에 필요한 유틸리티를 설치하겠습니다. 
* [cfssl](https://github.com/cloudflare/cfssl)
* [cfssljson](https://github.com/cloudflare/cfssl)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl)


## Install CFSSL

`cfssl`와 `cfssljson`은 [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure)와 TLS 인증서를 만들기 위한 유틸리티입니다.


`cfssl`와 `cfssljson`를 `k8s-client`에 설치하겠습니다:


```
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/linux/cfssl \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/linux/cfssljson
  
```

```
chmod +x cfssl cfssljson

```

```
sudo mv cfssl cfssljson /usr/local/bin/

```

### Verification


`cfssl`와 `cfssljson`의 설치 버전이 1.3.4 이상인지 확인합니다.

```
cfssl version

```

> output

```
Version: 1.3.4
Revision: dev
Runtime: go1.13
```

```
cfssljson --version

```
```
Version: 1.3.4
Revision: dev
Runtime: go1.13
```

## Install kubectl

`kubectl`은 Kubernetes API Server와 통신을 하기 위한 유틸리티입니다. 공식 릴리즈된 바이너리 버전을 설치하겠습니다.

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubectl

```

```
chmod +x kubectl

```

```
sudo mv kubectl /usr/local/bin/

```

### Verification

`kubectl`의 설치 버전이 1.15.3 이상인지 확인합니다.

```
kubectl version --client

```

> output

```
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.3", GitCommit:"2d3c76f9091b6bec110a5e63777c332469e0cba2", GitTreeState:"clean", BuildDate:"2019-08-19T11:13:54Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
