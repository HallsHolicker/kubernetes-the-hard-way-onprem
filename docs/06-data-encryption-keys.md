# Generating the Data Encryption Config and Key

Kubernetes는 클러스터 상태, 어플리케이션 구성, Secret등의 데이터를 저장합니다. Kubernetes는 클러스터 데이터를 암호화[encrypt](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) 하는 기능을 지원합니다.

이번 실습에서는 kubernetes 암호화에 적합한 암호화 키와 [encryption config](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration)을 생성합니다. 

## The Encryption Key

암호화 키 생성:

```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## The Encryption Config File

암호화 config `encryption-config.yaml` 생성:

```
cat <<EOF | sudo tee encryption-config.yaml
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

각 Controller node에 암호화 config인 `encryption-config.yaml`을 복사 합니다.

```
for hostname in k8s-controller-1 k8s-controller-2 k8s-controller-3; do
  scp encryption-config.yaml ${hostname}:~/
done
```

Next: [Bootstrapping the etcd Cluster](07-bootstrapping-etcd.md)
