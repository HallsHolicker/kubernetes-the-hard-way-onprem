# Smoke Test

이 실습은 kubernetes cluster가 제대로 구성되어 있는지 테스트를 진행하겠습니다.

## Data Encryption

이번 섹션에서는 [encrypt secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted)의 기능에 대해서 검증하겠습니다.

generic secret 생성:

```
ssh k8s-client
```

```
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Etcd에 저장된 `kubernetes-the-hard-way` secret의 hexdump 출력:

```
ssh k8s-controller-1
ETCDCTL_API=3
sudo /usr/local/bin/etcdctl get \
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
00000040  3a 76 31 3a 6b 65 79 31  3a 0f ff 37 1f e0 87 50  |:v1:key1:..7...P|
00000050  47 18 89 e5 ec 3e 77 be  39 d9 69 79 48 d2 df c1  |G....>w.9.iyH...|
00000060  80 04 0b 24 64 fc f7 a4  09 a8 8b c2 c3 e1 10 95  |...$d...........|
00000070  a1 35 08 ce f7 f3 ad 04  5f 67 81 29 6a 8e 4e 65  |.5......_g.)j.Ne|
00000080  61 8d 45 6f 3b bc 04 12  82 0e 17 cf a5 bd 65 be  |a.Eo;.........e.|
00000090  e5 5c 36 43 96 43 d2 24  ad f2 fc c6 e2 90 80 be  |.\6C.C.$........|
000000a0  d3 15 eb 22 07 68 0f 2d  18 74 8d f5 ec a1 b3 bb  |...".h.-.t......|
000000b0  24 8d 0f 8f e5 fb 39 82  ac 70 e4 49 b6 1f 33 b3  |$.....9..p.I..3.|
000000c0  77 60 f6 1b f4 48 74 54  49 48 35 1f cf dd 41 c0  |w`...HtTIH5...A.|
000000d0  7f 4c 9b c9 80 5f 9b 04  1a 03 18 b9 9f c2 c5 fb  |.L..._..........|
000000e0  e0 e5 ab 55 48 e9 8a 26  bf 50 af 16 67 f3 8e f1  |...UH..&.P..g...|
000000f0  ab 7f c3 90 7a 3e 88 91  73 27 65 3f ef 40 4a 92  |....z>..s'e?.@J.|
00000100  54 a6 10 25 22 2c b9 65  96 72 94 f9 33 ed 82 c4  |T..%",.e.r..3...|
00000110  35 ae 2d 9a 06 ca 2a 2c  dc ed 5d ab a0 f8 27 21  |5.-...*,..]...'!|
00000120  27 8c c3 d6 0c a8 15 7d  2b 12 71 64 17 7b 30 26  |'......}+.qd.{0&|
00000130  c1 4e 14 90 ec 24 cb 3d  a8 d6 55 48 4c f5 59 11  |.N...$.=..UHL.Y.|
00000140  2d f5 4b 3e 60 a4 c5 4a  c2 0a                    |-.K>`..J..|
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

Controller에서 forwarding 주소를 사용해서 HTTP 요청을 합니다.


```
curl --head http://127.0.0.1:8080
```

> output

```
HTTP/1.1 200 OK
Cache-Control: no-cache, private
Content-Type: application/json
Date: Tue, 16 Feb 2021 04:18:54 GMT
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
nginx version: nginx/1.19.6
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
WORKER_IP=$(grep "k8s-worker-1" /etc/hosts | awk '{print $1}')
```

worker IP 및 `nginx` node port를 사용하여 HTTP 요청:

```
curl -I http://${WORKER_IP}:${NODE_PORT}
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.19.6
Date: Tue, 16 Feb 2021 04:26:43 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 15 Dec 2020 13:59:38 GMT
Connection: keep-alive
ETag: "5fd8c14a-264"
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
EXTERNAL_IP=$(kubectl get svc nginx --output=jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

BGP로 전파 받는지 확인

```
ip route
```

>output
> 아래의 값은 테스트 때마다 다를 수 있으며, EXTERNAL_IP가 있는지 확인만 되면 됩니다.

```
default via 10.0.2.2 dev enp0s3 proto dhcp metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
10.240.0.0/24 dev enp0s8 proto kernel scope link src 10.240.0.100 metric 101
192.168.72.0 proto bgp metric 20
	nexthop via 10.240.0.21 dev enp0s8 weight 1
	nexthop via 10.240.0.23 dev enp0s8 weight 1
```

External IP를 이용하여 HTTP 요청:

```
curl -I http://${EXTERNAL_IP}
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.19.6
Date: Tue, 16 Feb 2021 05:16:56 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 15 Dec 2020 13:59:38 GMT
Connection: keep-alive
ETag: "5fd8c14a-264"
Accept-Ranges: bytes
```

Next: [Cleaning Up](14-cleanup.md)
