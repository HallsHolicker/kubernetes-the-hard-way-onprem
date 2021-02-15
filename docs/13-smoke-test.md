# Smoke Test

이 실습은 kubernetes cluster가 제대로 구성되어 있는지 테스트를 진행하겠습니다.

## Data Encryption

이번 섹션에서는 [encrypt secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted)의 기능에 대해서 검증하겠습니다.

generic secret 생성:

```
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Etcd에 저장된 `kubernetes-the-hard-way` secret의 hexdump 출력:

```
ssh k8s-controller-1
sudo ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C
```

> output

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 44 ac 6e ac 11 2f 28  |:v1:key1:D.n../(|
00000050  02 46 3d ad 9d cd 68 be  e4 cc 63 ae 13 e4 99 e8  |.F=...h...c.....|
00000060  6e 55 a0 fd 9d 33 7a b1  17 6b 20 19 23 dc 3e 67  |nU...3z..k .#.>g|
00000070  c9 6c 47 fa 78 8b 4d 28  cd d1 71 25 e9 29 ec 88  |.lG.x.M(..q%.)..|
00000080  7f c9 76 b6 31 63 6e ea  ac c5 e4 2f 32 d7 a6 94  |..v.1cn..../2...|
00000090  3c 3d 97 29 40 5a ee e1  ef d6 b2 17 01 75 a4 a3  |<=.)@Z.......u..|
000000a0  e2 c2 70 5b 77 1a 0b ec  71 c3 87 7a 1f 68 73 03  |..p[w...q..z.hs.|
000000b0  67 70 5e ba 5e 65 ff 6f  0c 40 5a f9 2a bd d6 0e  |gp^.^e.o.@Z.*...|
000000c0  44 8d 62 21 1a 30 4f 43  b8 03 69 52 c0 b7 2e 16  |D.b!.0OC..iR....|
000000d0  14 a5 91 21 29 fa 6e 03  47 e2 06 25 45 7c 4f 8f  |...!).n.G..%E|O.|
000000e0  6e bb 9d 3b e9 e5 2d 9e  3e 0a                    |n..;..-.>.|
```

etcd key는 `k8s:enc:aescbc:v1:key1`로 시작해야 하며, 이것은 `aescbc` provider가 `key1`암호화 키로 데이터를 암호화하는데 사용되었음을 보여줍니다.

## Deployments

이번 섹션은 [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)의 생성과 관리 기능에 대한 검증을 합니다.

[nginx](https://nginx.org/en/) 웹 서버의 deployment 생성:

```
kubectl create deployment nginx --image=nginx
```

`nginx` deployment로 생성된 pod 리스트 확인:

```
kubectl get pods -l app=nginx
```

> output

```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-554b9c67f9-vt5rn   1/1     Running   0          10s
```

### Port Forwarding

이번 섹션 [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)을 사용하여 외부에서 어플리케이션에 접근하는 기능을 검증합니다.

`nginx` pod의 full name 검색:

```
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
```

`nginx` pod의 `80`를 로컬의 `8080` port로 forward:

```
kubectl port-forward $POD_NAME 8080:80
```

> output

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

새로운 터미널에서 forwarding 주소를 사용해서 HTTP 요청을 합니다.

```
curl --head http://127.0.0.1:8080
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.17.3
Date: Sat, 14 Sep 2019 21:10:11 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 13 Aug 2019 08:50:00 GMT
Connection: keep-alive
ETag: "5d5279b8-264"
Accept-Ranges: bytes
```

이전에 사용하던 터미널로 돌아가서 `nginx` pod에 port forwarding을 중지:

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### Logs

이번 섹션은 [retrieve container logs](https://kubernetes.io/docs/concepts/cluster-administration/logging/)의 기능을 검증합니다.

`nginx` pod 로그 출력:

```
kubectl logs $POD_NAME
```

> output

```
127.0.0.1 - - [14/Sep/2019:21:10:11 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.52.1" "-"
```

### Exec

이번 섹션은 [execute commands in a container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container)의 기능을 검증합니다.

`nginx` 컨테이너 안에서 `nginx -v` 명령어를 실행하여 nginx version을 출력:

```
kubectl exec -ti $POD_NAME -- nginx -v
```

> output

```
nginx version: nginx/1.17.3
```

## Services

이번 섹션은 [Service](https://kubernetes.io/docs/concepts/services-networking/service/)을 사용하여 어플리케이션을 외부에 노출하는 기능을 검증합니다.

[NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) 서비스를 사용하여 `nginx` deployment 노출:

```
kubectl expose deployment nginx --port 80 --type NodePort
```

`nginx` service에 할당된 node port를 검색:

```
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

worker의 IP 확인:

```
WORKER_IP=$(gcloud compute instances describe worker-0 \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')
```

worker IP 및 `nginx` node port를 사용하여 HTTP 요청:

```
curl -I http://${WORKER_IP}:${NODE_PORT}
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.17.3
Date: Sat, 14 Sep 2019 21:12:35 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 13 Aug 2019 08:50:00 GMT
Connection: keep-alive
ETag: "5d5279b8-264"
Accept-Ranges: bytes
```

Nodeport Service 삭제

```
kubectl delete svc nginx
```


[LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) 서비스를 사용하여 `nginx` deployment 노출:

```
kubectl expose deployment nginx --port 80 --type LoadBalancer
```

External IP 확인:

```
```

External IP를 이용하여 HTTP 요청:

```
curl -I https://${EXTERNAL_IP}
```

> output

```
```

Next: [Cleaning Up](14-cleanup.md)
