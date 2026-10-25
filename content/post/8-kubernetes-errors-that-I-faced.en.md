---
title: "Kubernetes Errors I Encountered"
author: Bumgu
date: 2024-04-25
categories: 
    - DevOps
tags: 
    - Kubernetes
    - 오류
slug: "kuberntes_errors_that_I_faced"
---
When setting up a cluster with `kubeadm` in an on-premises environment to build a monitoring server (prometheus, zabbix), I ran into several errors. Here I've collected them.

# 1.
```
Unable to connect to the server: x509:
certificate signed by unknown authority
(possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")
```
This error can occur when running `kubeadm init` or `kubeadm join`. It usually happens when using an old or incorrectly configured config.

`rm -rf ~/.kube`  
`mkdir ~/.kube`  
`cp -i /etc/kubernetes/admin.conf ~/.kube/config`  
`chown $(id -u):$(id -g) ~/.kube/config`  


# 2.
```
When kube-flannel and kube-proxy are in CrashLoopBackOff
```
This error may occur when installing `kube-flannel` for communication within the cluster.

Use the `kubectl describe po -n kube-flannel ${kube-flannel pod-name}` command to check which node it is running on, then connect to that node server and run  
`containerd config default | /etc/containerd/config.toml`  
Then, in the generated `/etc/containerd/config.toml`, change  
`[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]`  
`SystemdCgroup = false`  
to  
`SystemdCgroup = true`  
Then restart `containerd` with
`systemctl restart containerd`
and check whether the status has changed.


# 3.
```
Taint error
```
I encountered this error when I created only a master node and tried to launch a Pod as a test.
Pod creation is not allowed on the Control Plane, but I removed the Taint temporarily to test it.
Taint is configured on nodes where you don't want Pods to run.
Therefore, when a Taint is set, Pods are not scheduled on the node with that Taint.

- Check Taint
`kubectl describe node ${node-name} | grep -i "taint"`
- Remove Taint
`kubectl taint nodes -all node-role.kubernetes.io.master-`



---
Thank you.
