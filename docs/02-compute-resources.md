# Provisioning Compute Resources

Kubernetes는 control plane과 Container가 실행되는 Worker node를 호스팅하기 위한 장비가 필요합니다.
이번 실습은 안전하고 가용성이 높은 Kubernetes Cluster를 실행하는데 필요한 컴퓨팅 리소스를 프로비저닝합니다.

## Compute Instances

이미지는 `centos8-stream`으로 별도로 만든 이미지를 사용하였습니다.

3개의 Control Plane Node와 3개의 Worker Node를 생성하겠습니다.

```
cat << EOF > vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "hallsholicker/centos8-stream-k8s"

  config.vm.define "k8s-controller-1" do |controller1|
    controller1.vm.network "private_network", ip: "10.240.0.11"
  end

  config.vm.define "k8s-controller-2" do |controller2|
    controller2.vm.network "private_network", ip: "10.240.0.12"
  end

  config.vm.define "k8s-controller-3" do |controller3|
    controller3.vm.network "private_network", ip: "10.240.0.13"
  end

  config.vm.define "k8s-worker-1" do |worker1|
    worker1.vm.network "private_network", ip: "10.240.0.21"
  end
Prerequier
  config.vm.define "k8s-worker-2" do |worker2|
    worker2.vm.network "private_network", ip: "10.240.0.22"
  end

  config.vm.define "k8s-worker-3" do |worker3|
    worker3.vm.network "private_network", ip: "10.240.0.23"
  end
  
  config.vm.define "k8s-client" do |client|
    client.vm.network "private_netwrok", ip: "10.240.0.100"
  end
end
EOF

vagrant up
```

## Networking

기존 [Kelsey Hightower](https://github.com/kelseyhightower/kubernetes-the-hard-way)는 기본 CNI를 사용하여 각 Worker Node는 Cluster CIDR 범위에서 pod-cidr를 중복되지 않게 설정을 해야 했습니다. 그러나 이번 구성에는 CNI로 Calico를 사용하므로 별도로 pod-cidr을 설정하지 않아도 됩니다.

> Kubernetes cluser CIDR 범위는 Controller Manager의 `--cluster-cidr` 옵션에 설정 됩니다. 이번 튜토리얼에서는 cluster CIDR 범위로 `10.200.0.0/16`을 설정하였습니다.

## Configuring SSH Access & Hostname & /etc/hosts

K8S를 연습하기 전에 서버 접속 및 hostname등 기본적인 세팅을 진행 합니다.
K8S 설정 작업은 `k8s-client`에서 진행을 할 예정이며, 원활한 접속을 위해 k8s-client의 ssh key 및 hostname을 전체 서버에 설정하겠습니다.

`k8s-client`에 ssh 접속:

```
vagrant ssh k8s-client
sudo dnf -y install expect
```

SSH를 접속할 때 패스워드를 입력하지 않고 접속하기 위해 SSH Key를 만듭니다.

```
ssh-keygen
```

> output

```
Generating public/private rsa key pair.
Enter file in which to save the key (/home/vagrant/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/vagrant/.ssh/id_rsa.
Your public key has been saved in /home/vagrant/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:oY3xIsdalof03dcfZv7S7OKNVdGxaH1Uhoisz7ozR0A vagrant@localhost.localdomain
The key's randomart image is:
+---[RSA 3072]----+
|         . . . o=|
|        E o . +o+|
|      o...   o +o|
|     o Xoo ..  .o|
|    . @ S+. . .+o|
|     * o  +  .+ +|
|    .    o     +o|
|        + .   o++|
|        .=   .o++|
+----[SHA256]-----+
```

우선 SSH Key를 복사하기 전에 서버들에 접속을 원활하게 위해서 /etc/hosts에 hostname 및 IP를 설정하겠습니다.


```
cat <<EOF | sudo tee ./Setting_hosts.sh
#!/bin/bash

declare -A Target
Target=( ["k8s-controller-1"]="10.240.0.11" \\
         ["k8s-controller-2"]="10.240.0.12" \\
         ["k8s-controller-3"]="10.240.0.13" \\
         ["k8s-worker-1"]="10.240.0.21" \\
         ["k8s-worker-2"]="10.240.0.22" \\
         ["k8s-worker-3"]="10.240.0.23" \\
         ["k8s-client"]="10.240.0.100" \\
         ["k8s-controller"]="10.240.0.10" )

echo -e "Regist hostname to /etc/hosts start!\n"

for key in "\${!Target[@]}"; do

  if [[ -z \`grep "\${key}" /etc/hosts\` ]]; then
    echo "\${Target[\$key]} \${key}" >> /etc/hosts
    echo -e "\033[40;32m\${key} : \${Target[\$key]} regist\033[0m"
  else
    ip_check=\`grep "\${key}" /etc/hosts | awk '{print \$1}'\`
    if [[ ! "\${Target[\$key]}" == "\${ip_check}" ]]; then
      sed -i "s/\${ip_check} \${key}/\${Target[\$key]} \${key}/g" /etc/hosts
      echo -e "\033[40;31m\${key} : \${ip_check} -> \${Target[\$key]} change\033[0m"
    else
      echo "\${key} : \${Target[\$key]} Already regist!"
    fi
  fi

done

echo ""
echo "Regist hostname to /etc/hosts end!"

EOF
```

```
sudo sh Setting_hosts.sh
```

이제 만들어진 SSH Key 중 Public Key를 k8s-controller 3대, k8s-worker 3대에 복사 및 hostname, /etc/hosts를 설정하겠습니다.
expert 실행 시에 딜레이가 발생하므로 실행시키고 조금 기다려야 합니다.

```
cat <<EOF | sudo tee ./Setting_hostname.sh
#!/bin/bash

declare -A Target
Target=( ["k8s-controller-1"]="10.240.0.11" \\
         ["k8s-controller-2"]="10.240.0.12" \\
         ["k8s-controller-3"]="10.240.0.13" \\
         ["k8s-worker-1"]="10.240.0.21" \\
         ["k8s-worker-2"]="10.240.0.22" \\
         ["k8s-worker-3"]="10.240.0.23" )

echo "Set hostname start!"
echo ""

for key in "\${!Target[@]}"; do

  ip_check=\`ip route | grep -o -e " \${Target[\$key]} " | tr -d " "\`

  if [[ ! "\${ip_check}" == "" ]]; then
    hostnamectl set-hostname \${key}
    echo -e "Set hostname \033[40;32m\${key}\033[0m"
  fi

done

echo ""
echo "Set hostname end!"

EOF
```

```
cat <<EOF | sudo tee ./SSH-key-copy.sh
#!/bin/bash

passwd="vagrant"
declare -A Target
Target=( ["k8s-controller-1"]="10.240.0.11" \\
         ["k8s-controller-2"]="10.240.0.12" \\
         ["k8s-controller-3"]="10.240.0.13" \\
         ["k8s-worker-1"]="10.240.0.21" \\
         ["k8s-worker-2"]="10.240.0.22" \\
         ["k8s-worker-3"]="10.240.0.23" )

echo "Script Start"

for key in "\${!Target[@]}"; do

  echo "\${key} SSH Public Key Copy Start!"

  expect <<EOL
  spawn ssh-copy-id -i /home/vagrant/.ssh/id_rsa.pub \${key}
  expect (yes/no/\[fingerprint\])?
  send "yes\n"
  expect -re "password:"
  send "\${passwd}\n"
  expect eof
EOL

  echo "\${key} SSH Public Key Copy end!"
  echo ""
  echo "\${key} Setting hostname & hosts Start!"

  scp /home/vagrant/Setting_hosts.sh \${key}:/home/vagrant
  scp /home/vagrant/Setting_hostname.sh \${key}:/home/vagrant
  expect <<EOL
  spawn ssh \${key}
  expect -re "~]\$"
  send "sudo sh Setting_hosts.sh\n"
  send "sudo sh Setting_hostname.sh\n"
  expect eof
EOL

  echo "\${key} Setting hostname & hosts end!"
sleep 1
done

echo ""
echo ""
echo "Script END!"

EOF
```

```
sh SSH-key-copy.sh
```

Next: [Installing the Client Tools](03-client-tools.md)
