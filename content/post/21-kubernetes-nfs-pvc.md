---
title: "온프레미스 Kubernetes 환경에서 nfs로 데이터 영속성(pv) 설정"
author: Bumgu
date: 2024-04-28
categories: 
    - DevOps
    - SRE
tags: 
    - Kubernetes
slug: "kubernetes_nfs_pv"
---
# 마스터 서버
## 1. 마스터 서버 NFS 서버 설치

- nfs (Network File System) 설치
`apt-get install nfs-kernel-server`
- nfs 서버 기동
`systemctl start nfs-server`

## 2. 공유 디렉토리 지정
- 지정할 공유 디렉토리 생성
`mkdir /data/nfsk8s`

- 공유 디렉토리 지정
`vim /etc/expoerts`
**맨 아래줄에 <디렉토리 경로> <\IP>(\<권한>) 항목 추가**
**권한 삽입 시 띄어쓰기 하지말것 (ro, rw) X → (ro,rw) O**
**모든 IP에 대해 권한 적용**
`/data/nfsk8s *(rw,sync,no_root_squash)`

이후 nfs 재시작
`systemctl restart nfs-server`

nfs 옵션
```
ro : 읽기 권한 부여 한다.  
  
rw : 읽기 쓰기 권한 부여 한다.  
  
root_squash : 클라이언트에서 root를 서버상의 nobody 계정으로 매핑한다.  
  
no_root_squash : 클라이언트 및 서버 모두 root 계정 사용한다.  
  
sync : 파일시스템이 변경되면 즉시 동기화한다.  
  
all_squash : root 계정이 아닌 다른 계정도 사용 할  수 있게 한다.
```


# 클라이언트(워커노드) 서버

## 1. nfs 클라이언트 설치
- nfs 클라이언트 설치
`apt-get install nfs-common`
- 공유된 디렉토리 확인
`showmount -e ${nfs 서버 IP}`


## 2. 마운트
- 마운트할 디렉토리 생성
`mkdir /data/nfsk8s`
- 마운트
`mount -t nfs ${nfs 서버 IP}:/${nfs 서버 공유 디렉토리} ${클라이언트 마운트 디렉토리}`
- 마운트 확인
`df | grep nfs`

- 재부팅 시 자동 nfs 설정
`vim /etc/fstab`
`${nfs 서버 IP}:/${nfs 서버 공유 디렉토리} nfs defaults 0 0`

## 3. 마운트 됐는지 확인

![](/images/post/21-nfsk8s/1.png)
`192.168.64.129`에서 `test.txt`파일을 만듭니다.
nfs-server인 131번에서 공유 nfs 디렉토리를 확인해보면
![](/images/post/21-nfsk8s/2.png)
공유가 된것을 확인 할 수 있습니다.


# 마스터 노드
## 1. StorageClass 설치

StorageClass는 클라우드 환경에서 pv가 요구하는 조건에 맞게 동적으로 볼륨을 생성해주는 프로비저닝 방식입니다. 즉 pv가 볼륨의 크기, ssd여부, filesystem type, zone등을 지정하면 그에 맞는 볼륨을 생성해줍니다.

설치는 [helm](https://helm.sh/docs/intro/install/) 을 통해 설치합니다 
```
# StorageClass repo 추가
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

# StorageClass repo 확인
helm repo list
```

```
# StorageClass 설치
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
 --set nfs.server=${nfs 서버 IP} \
 --set nfs.path=${nfs 서버 공유 디렉토리 } \
 --set storageClass.name=nfs \
 --set storageClass.defaultClass=true
```

```
# 설치 확인
kubectl get sc
```

## 2. storageClass 사용
이제 기존의 pvc에서 `StorageClassName`을 지정해줍니다. `kubectl get sc`로 `StorageClass`를 확인해보면 이름이 `nfs`이기에 `StorageClassName`에 `nfs`를 적어주고 `helm`을 사용해 설치를하면 공유 디렉토리에 데이터 폴더가 생깁니다.

![](/images/post/21-nfsk8s/3.png)


참고한 글:  
https://junwork123.tistory.com/33  
https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-20-04
