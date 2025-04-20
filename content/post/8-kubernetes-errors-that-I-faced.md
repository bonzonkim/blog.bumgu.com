---
title: "내가 겪은 Kubernetes 오류들"
author: Bumgu
date: 2024-04-25
categories: 
    - DevOps
tags: 
    - Kubernetes
    - 오류
slug: "kuberntes_errors_that_I_faced"
---
모니터링 서버(prometheus, zabbix)를 구축하기 위해 온프레미스 환경에서 `kubeadm`을 사용해 클러스터를 구성 할 때 제가 겪은 오류들을 정리합니다.

# 1.
```
Unable to connect to the server: x509:
certificate signed by unknown authority
(possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")
```

`kubeadm init` 혹은 `kubeadm join`을 할때 생기는 오류 입니다. 주로 이전에 사용하거나 잘못된 config를 사용할때 발생합니다.

`rm -rf ~/.kube`  
`mkdir ~/.kube`  
`cp -i /etc/kubernetes/admin.conf ~/.kube/config`  
`chown $(id -u):$(id -g) ~/.kube/config`  


# 2.
```
kube-flannel, kube-proxy가 CrashLoopBackOff 상태일 경우
```

클러스터내의 통신을 위한 `kube-flannel`을 설치했을때 발생할 수 있는 오류입니다.

`kubectl describe po -n kube-flannel ${kube-flannel pod명}` 명령어로 어떤 node에 떠있는지 확인을 하고 그 node서버에 접속해서  
`containerd config default | /etc/containerd/config.toml`  
그리고 생성된 `/etc/containerd/config.toml`에서  
`[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]`  
`SystemdCgroup = false`  를  
`SystemdCgroup = true`  로 바꾼후  
`systemctl restart containerd`
로 `containerd`를 재시작 하고 상태가 변경되었는지 확인합니다.


# 3.
```
Taint 오류
```

마스터 노드만 생성하고 테스트로 Pod을 띄워볼때 만난 오류입니다.
Controle Pane에서는 Pod생성이 안되지만 임의로 Taint를 해제해 테스트 해봤습니다.
Taint는 Pod를 띄우고 싶지 않은 노드에 Taint 설정을 합니다.
따라서 Taint를 지정하면 지정된 노드에는 Pod가 스케줄링 되지 않습니다.

- Taint 확인
`kubectl describe node ${node명} | grep -i "taint"`
- Taint 해제
`kubectl taint nodes -all node-role.kubernetes.io.master-`



---
감사합니다.
