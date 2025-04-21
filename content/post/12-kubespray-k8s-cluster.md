---
title: "Kubespray로 k8s클러스터 구축"
author: Bumgu
date: 2024-09-15
categories: 
     - DevOps
tags: 
    - Kubernetes
    - Kubespray
    - Ansible
    - Vagrant
slug: "kubernetes_clustering_with_kubespray"
---
`Ansible`, `Kubespray`를 이용해 k8s클러스터 구축을 해보겠습니다.

---
### 1. Vagrant를 사용해 VM 생성


Vagrant 설정파일은 [kubernetes-certification-guide](https://github.com/techiescamp/kubernetes-certification-guide) 를 참고했습니다.

**setting.yaml**
```yaml
# settings.yaml
# box_name: "sloopstash/ubuntu-22-04"
box_name: "bento/ubuntu-22.04"
# box_version: "2.1.1"
vm:
- name: "controlplane"
  ip: "192.168.64.163"
  memory: "2048"
  cpus: "2"
- name: "node01"
  ip: "192.168.64.164"
  memory: "2048"
  cpus: "2"
- name: "node02"
  ip: "192.168.64.165"
  memory: "2048"
  cpus: "2"
```
**Vagrantfile**

```ruby
require 'yaml'
settings = YAML.load_file(File.join(File.dirname(__FILE__), 'settings.yaml'))

Vagrant.configure("2") do |config|
  config.vm.box = settings['box_name']
  # config.vm.box_version = settings['box_version']
  config.vm.box_check_update = false

  settings['vm'].each do |vm_config|
    config.vm.define vm_config['name'] do |vm|
      vm.vm.hostname = vm_config['name']
      vm.vm.network "public_network", ip: vm_config['ip']
      vm.vm.synced_folder ".", "/vagrant", disabled: false

      vm.vm.provider "vmware_fusion" do |vb|
        vb.gui = true
        vb.memory = vm_config['memory']
        vb.cpus = vm_config['cpus']
      end

      vm.vm.provision "shell", inline: <<-SHELL
        apt update
        apt upgrade -y
        apt install -y wget vim net-tools gcc make tar git unzip sysstat tree
        echo "192.168.201.10 controlplane" >> /etc/hosts
        echo "192.168.201.11 node01" >> /etc/hosts
        echo "192.168.201.12 node02" >> /etc/hosts
      SHELL
      # vm.vm.provision "shell", path: "scripts/common.sh"
    end
  end
end
```

`vagrant up`을 명령어를 통해 VM을 생성합니다.
생성이 된후엔 `vagrant ssh <vm name>`으로 접속해서 확인해봅니다.
`ssh vagrant@<ip>`로 접속해도 됩니다. `vagrant`유저의 초기 비밀번호는 `vagrant` 입니다.


### 1. ssh key 복사

ssh key를 생성합니다
`ssh-keygen -t rsa -b 4096 -f ~/.ssh/test_key`


`Ansible`은 ssh 키를 가지고 동작하므로 아까 생성한 `test_key`키를 복사 합니다.
```bash
#!/bin/bash

declare -a ips=(
   163
   164
   165
)

for ip in "${ips[@]}"; do
  sshpass -p vagrant ssh-copy-id -i ~/.ssh/id_ansible.pub vagrant@192.168.64."$ip"
done
```
스크립트로 한번에 키복사를 합니다.
`chmod +x <script filename>`
`./<script filename>`


### 2. Kubespray 클론
이제 `kubespray`를 클론받습니다.
`git clone https://github.com/kubernetes-sigs/kubespray.git`

받고나면 `git checkout release-<버전>` 으로 릴리즈 버전을 선택합니다.
이 버전은 각자 설치된 `Ansible`의 버전을 따르면 됩니다.
버전 확인은
`kubespray/playbooks/ansible_version.yaml`의
```yaml
vars:
    minimal_ansible_version: 2.15.5 
    maximal_ansible_version: 2.17.0
```
를 확인하면 됩니다. 저는 `Ansible`의 버전이 2.16.3이기 때문에 `release-2.24`를 사용했습니다.

#### 2-1. Inventory 작성
`kubespray/inventory/sample`을 복사합니다.

`cp -r inventory/sample inventory/mycluster`


이제 `mycluster`경로 안의 `inventory.ini`를 수정합니다.

**invetory/mycluster/inventory.ini**
```yaml
# ## Configure 'ip' variable to bind kubernetes services on a
# ## different ip than the default iface
# ## We should set etcd_member_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value.
[all]
controlplane ansible_host=192.168.64.163 ip=192.168.64.163 etcd_member_name=etcd1 ansible_ssh_private_key_file=~/.ssh/test_key  ansible_user=vagrant
node01 ansible_host=192.168.64.164 ip=192.168.64.164 etcd_member_name=etcd2 ansible_ssh_private_key_file=~/.ssh/test_key  ansible_user=vagrant
node02 ansible_host=192.168.64.165 ip=192.168.64.165 etcd_member_name=etcd3 ansible_ssh_private_key_file=~/.ssh/test_key  ansible_user=vagrant

# ## configure a bastion host if your nodes are not directly reachable
# [bastion]
# bastion ansible_host=x.x.x.x ansible_user=some_user

[kube_control_plane]
controlplane
node01
node02

[etcd]
controlplane
node01
node02

[kube_node]
node01
node02

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```
이 인벤토리가 제대로 작동하는지 확인해봅니다.

`ansible all -i <inventory.ini 경로> -m ping`

![](/images/post/12-kubespray/1.png)



#### 2-2. group_vars 작성

`inventory/sample`을 복사하면 `group_vars`폴더가 있습니다.
여기서 쿠버네티스의 `helm`, `nginx ingress controller`, `metallb`등의 애드온 설치를 지정할 수 있습니다.

**inventory/mycluster/group_vars/k8s_cluster/addons.yaml**
파일이 길기 때문에 `true`로 수정해준 부분만 적겠습니다.

`helm_enabled: true` : helm - 패키지매니저 쉽게 쿠버네티스 리소스를 배포할 수 있다
`ingress_nginx_enabled: true` : nginx ingress controller - L7에서 외부 트래픽 처리하는 컨트롤러
`local_path_provisioner_enabled: true` : 쿠버네티스 환경에서 로컬 스토리지를 동적으로 프로비저닝하는 Storage Provisioner

이외의 `ArgoCD`, `Cert-manager`등 있지만 차후에 설치하기로 합니다.

### 3. playbook 실행
이제 inventory와 addon설정까지 끝났으니 playbook을 실행해 클러스터를 잡도록 하겠습니다.

`ansble-playbook -i <invetory 파일경로> cluster.yaml`
클론받은 `kubespray`경로 루트에 있는 `cluster.yaml`을 실행하면 됩니다 혹은 `playbooks`디렉토리의 `cluster.yaml`을 지정해도 됩니다.

실행하면 약 15-20분정도 걸립니다.


![](/images/post/12-kubespray/2.png)


playbook실행이 완료되었습니다. 위의 에러는 Metallb 설치시 `kube-proxy`의 `configmap` 필드중 `strictARP`의 값을 수정해주지 않아서 그런것이니 수정후 다시 설치해주면 됩니다.

우선 클러스터링이 되었는지 확인부터 하겠습니다.

`kubectl get nodes` 명령어를 실행했으나,

![](/images/post/12-kubespray/3.png)



`The connection to the server localhost:8080 was refused - did you specify the right host or port?` 에러가 뜹니다.

이 에러는 `/etc/kubernetes/admin.conf`를 `~/.kube/config`로 복사해 실행할 수 소유권한을 주면 됩니다.

* `sudo mkdir ~/.kube/`: 디렉토리생성  
* `sudo touch ~/.kube/config`: config 파일생성  
* `sudo cp /etc/kubernetes/admin.conf ~/.kube/config`: 파일 복사  
* `sudo chown $(id -u):$(id -g) ~/.kube/config`: 소유권 부여  

이제 다시 `kubectl get nodes`명령어를 실행해봅니다.
![](/images/post/12-kubespray/4.png)






노드들이 정상적으로 클러스터링이 된것을 확인 할 수 있습니다.



### 4. 테스트 파드 띄우기

간단하게 nginx deployment, service 를 생성하고 마치도록하겠습니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
  namespace: test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
      - name: nginx
        image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: test-service
  namespace: test
spec:
  selector:
    app: nginx-test
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```
`kubectl create ns test`
`kubectl apply -f test.yaml`

이제 `kubectl get all -n test` 로 확인합니다.
![](/images/post/12-kubespray/9.png)


정상적으로 쿠버네티스 리소스가 생성된것을 확인할 수 있습니다.

---

### 마무리
`metallb`를 설치해서 ingress까지 사용해보는 것을 목표로 했으나, metallb의 `ipAddressPool`을 apply하면 `kubectl` 이 apiServer에 접근을 못하는 오류가 생겨서 차후 해결후 따로 작성하도록 하겠습니다. [여기서 확인](https://blog.bumgu.com/post/2024/09/22/kubespray_metallb_install/)

그동안은 `kubeadm`을 이용해서 노드마다 같은 설정을 반복해주는 번거로움이 있었는데, 처음으로 `kubespray`을 이용해 자동 클러스터링 작업을 했습니다. 

`Ansible`을 사용하고, 온프레미스 환경에서 쿠버네티스를 다루는 분들에겐 좋은 툴이라고 생각합니다. 클러스터링을 하기위해 필수 유틸리티 설치 및 설정을 해줘야하는데 휴먼에러도 방지하면서 필요한 유틸리티를 `group_vars`를 통해 설치할 수 있는것이 매력적이었습니다.
