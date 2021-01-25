# Prerequisites

# VirtualBox

가상 머신은 [VirtualBox](https://www.virtualbox.org/wiki/Downloads)를 사용하였습니다.

# Vagrant

VirtualBox에 이미지 프로비저닝을 쉽게 하기 위하여 [Vagrant](https://www.vagrantup.com/)를 사용하였습니다.
이미지는 centos8-stream으로 별도로 만든 이미지를 사용하였습니다.

'''
{
cat > vagrantfile << EOF
Vagrant.configure("2") do |config|
  config.vm.box = "hallsholicker/centos8-stream-k8s"

  config.vm.define "k8s-master-1" do |master1|
    master1.vm.network "private_network", ip: "10.240.0.11"
  end

  config.vm.define "k8s-master-2" do |master2|
    master2.vm.network "private_network", ip: "10.240.0.12"
  end

  config.vm.define "k8s-master-3" do |master3|
    master3.vm.network "private_network", ip: "10.240.0.13"
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
}
'''

# Preset

K8S를 연습하기 전에 서버 접속 및 hostname등 기본적인 세팅을 진행 합니다.
## SSH Key setting
K8S 설정 작업은 k8s-client에서 진행을 할 예정이며, 원활한 접속을 위해 k8s-client의 ssh key를 전체 서버에 설정하겠습니다.
Expect의 EOF와 Cat의 EOF가 겹치게 되므로 Cat의 종료 단어를 ENDFILE로 변경하여 스크립트를 생성합니다.

### SSH Key Generater
'''
{
dnf -y install expect

cd ~/
mkdir .ssh
su -

ssh-keygen

Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): [엔터키]
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase): [엔터키]
Enter same passphrase again: [엔터키]
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:u80BrVp55p2RbzJhzaFplqcDsGCOBFZ0Ag4V62z0Zko root@localhost.localdomain
The key's randomart image is:
+---[RSA 3072]----+
|..+=+ .          |
| oo. o           |
| .+.             |
| + .. o ..    .  |
|  E.++ .So.  * . |
| o +. . .=. O.+  |
|  .     = +=o+   |
|       o B o=+.  |
|      . . + o=.  |
+----[SHA256]-----+
}
'''

### SSH Key Copy & hostname
'''
{
cat <<EOF | sudo tee /root/Preset.sh
#!/bin/bash

ROOTPW="vagrant"
declare -A Target
Target=( ["k8s-master-1"]="10.240.0.11" \
         ["k8s-master-2"]="10.240.0.12" \
         ["k8s-master-3"]="10.240.0.13" \
         ["k8s-worker-1"]="10.240.0.21" \
         ["k8s-worker-2"]="10.240.0.22" \
         ["k8s-worker-3"]="10.240.0.23" )

for key in "\${!Target[@]}"; do

  echo "\${Target[\$key]} \${key}" >> /etc/hosts

  echo "\${key} SSH Public Key Copy Start!"

  expect <<EOL
  spawn ssh-copy-id -i /root/.ssh/id_rsa.pub \${key}
  expect (yes/no/\[fingerprint\])?
  send "yes\n"
  expect -re "password:"
  send "\${ROOTPW}\n"
  expect eof
EOL

  echo "\${key} SSH Public Key Copy Complate!"

sleep 1
done

for key in "\${!Target[@]}"; do

  echo "\${key} append hostname to hosts file & Set hostname Start!"

  expect <<EOL
  spawn ssh \${Target[\$key]}
  expect -re "~]#"
  send "echo -e \"10.240.0.11 k8s-master-1\n10.240.0.12 k8s-master-2\n10.240.0.13 k8s-master-3\n10.240.0.21 k8s-worker-1\n10.240.0.22 k8s-worker-2\n10.240.0.23 k8s-worker-3\" >> /etc/hosts \r\n"
  send "hostnamectl set-hostname \${key} \r\n"
  expect eof
EOL

  echo "\${key} append hostname to hosts file & Set hostname Complate!"

sleep 1
done

EOF
}
'''
