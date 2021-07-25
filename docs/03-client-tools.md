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
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssl \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssljson
```

```
chmod +x cfssl cfssljson
```

```
sudo mv cfssl cfssljson /usr/local/bin/
```

### Verification


`cfssl`와 `cfssljson`의 설치 버전이 1.4.1 이상인지 확인합니다.

```
cfssl version
```

> output

```
Version: 1.4.4
Runtime: go1.12.12
```

```
cfssljson --version

```
```
Version: 1.4.4
Runtime: go1.12.12
```

## Install kubectl

`kubectl`은 Kubernetes API Server와 통신을 하기 위한 유틸리티입니다. 공식 릴리즈된 바이너리 버전을 설치하겠습니다.

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/local/bin/
```

### Verification

`kubectl`의 설치 버전이 1.21.0 이상인지 확인합니다.

```
kubectl version --client
```

> output

```
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
