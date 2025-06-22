---
title: "Kubernetes Cluster 버전 업그레이드"
author: Bumgu
draft: false
date: 2025-06-22T13:25:06+09:00
categories: 
    - DevOps
    - SRE
tags: 
    - Kubernetes
slug: "kube-cluster-upgrade"
---

# Kubernetes Cluster Upgrade
**Server: Vagrant VM (Ubuntu 22.04)**  
**현재 Kubeamd, Kubectl, Kubelet: 1.32.6**  
**업그레이드 할 Kubeamd, Kubectl, Kubelet: 1.33**

현재 운영중이라는 가정하에 간단한 API 서버를 실행중 업그레이드를 수행한다.

Kubernetes yaml, 소스코드, Vagrantfile 등 [Github](https://github.com/bonzonkim/kube-cluster-upgrade)

# 현재
`GET :8080/hello`
```
method: GET  
Response:
{
Message: Hello world!
}
```
`GET :8080/bye`
```
method: GET  
Response:
{
Message: Bye world!
}
```

## 1. Control plane - Kubeadm
```
apt update
apt-cache madison kubeadm
```

위 명령어 실행 시 1.32.x 버전만 보인다면 apt 저장소 업데이트가 필요합니다.

`/etc/apt/sources.list.d/kubernetes.list`
```
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ / 
                                                                                               ^^ 이 부분을 1.33으로 수정
```

1. apt 저장소 업데이트  
`apt-get update`  
2. 업데이트할 버전 검색
`apt-cache madison kubeadm`
```bash
$ /etc/apt/sources.list.d# apt-cache madison kubeadm
   kubeadm | 1.33.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
   kubeadm | 1.33.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
   kubeadm | 1.33.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
```

```bash
 apt-mark unhold kubeadm && \
 apt-get update && apt-get install -y kubeadm=1.33.2-1.1 && \
 apt-mark hold kubeadm
```


업그레이드 플랜을 확인한다.  
`kubeadm upgrade plan`  
![](/images/post/24-kube-cluster-upgrade/1.png)
사진의 보이듯이 나오는 명령어를 실행하고 완료될때까지 기다립니다.
```bash
[upgrade] SUCCESS! A control plane node of your cluster was upgraded to "v1.33.2".

[upgrade] Now please proceed with upgrading the rest of the nodes by following the right order.
```
와 같은 내용이 나오면 정상적으로 업그레이드가 완료 된 것 입니다.  
저의 경우에는 `context deadline exceeded` 가 나왔지만 기다리니 정상적으로 완료 되었습니다.

## 2. Control plane - Kubectl, Kubelet

kubeadm과 같이 apt 저장소에서 버전을 검색하여 업그레이드 할 버전을 결정합니다.
```bash
$ apt-cache madison kubectl
   kubectl | 1.33.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
   kubectl | 1.33.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
   kubectl | 1.33.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
$ apt-cache madison kubelet
   kubelet | 1.33.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
   kubelet | 1.33.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
   kubelet | 1.33.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
```

```bash
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.33.2-1.1 kubectl=1.33.2-1.1 && \
apt-mark hold kubelet kubectl
```

명령어가 실행 완료되고 버전을 확인해보면
![](/images/post/24-kube-cluster-upgrade/2.png)
1.33.2 버전으로 완료가 되었습니다.

## 3. Worker - Drain
지금은 실습환경이라 컨트롤플레인 1대, 워커노드 2대의 구성이지만, 만약 실무에서 운영중이라면 많은 Pod가 실행중이고,  
여러대의 노드들이 있을 것 입니다. 그렇기 때문에 **현재 실행중인 Pod를 옮기고** 업그레이드를 수행합니다.
현재 hello app은 worker1에 실행되고 있습니다 worker1을 Drain해 실행중인 Pod를 worker1으로부터 evict 합니다.  
`kubectl drain worker1 --ignore-daemonsets`
![](/images/post/24-kube-cluster-upgrade/3.png)
노드를 cordon 시키고, 그 이후에 evict 합니다.  
* cordon : 노드를 **스케줄링 불가**한 상태로 만듬. 현재 실행 중인 Pod를 evict 시키진 않는다.
* drain : 노드에 **실행중인 Pod를 Evict** 한다.


![](/images/post/24-kube-cluster-upgrade/4.png)
Drain 전에는 worker1 (192.168.64.201)에서 실행중이었지만,
![](/images/post/24-kube-cluster-upgrade/5.png)
Drain 후에는 woker2 (192.168.64.202)노드로 옮겨간것을 확인 할 수 있습니다.

## 4. Worker - Kubeadm
이제 노드에 실행중인 Pod도 없으니 업그레이드를 수행합니다.  
컨트롤 플레인과 같이 apt 저장소를 업데이트 해주고 명령어를 실행합니다.
* kubeadm
```bash
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.33.x-00 kubectl=1.33.x-00 && \
apt-mark hold kubelet kubectl
```

* kubectl, kubelet
```bash
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.33.2-1.1 kubectl=1.33.2-1.1 && \
apt-mark hold kubelet kubectl
```

1번 노드 업그레이드가 끝났다면 `uncordon`을 통해 노드를 다시 스케줄링 가능한 상태로 만듭니다.
`kubectl uncordon worker1`

위 과정을 워커노드에게 반복합니다.

## 5. 완료
두번째 노드까지 uncordon 해주면 업그레이드 과정중 Pod의 정지 없이 클러스터를 업그레이드 하는 과정을 완료했습니다.  
버전을 확인해보면

![](/images/post/24-kube-cluster-upgrade/6.png)


이렇게 클러스터 업그레이드가 완료되었습니다.  
