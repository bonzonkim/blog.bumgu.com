---
title: "Minikube를 이용하여 쿠버네티스 실습해보기"
author: Bumgu
date: 2024-03-10
categories: 
    - DevOps
    - Kubernetes
tags: 
    - DevOps
    - Kubernetes
    - minikube
slug: "kubernetes practice"
---

[subicura](https://subicura.com/k8s/guide/#%E1%84%8B%E1%85%AF%E1%84%83%E1%85%B3%E1%84%91%E1%85%B3%E1%84%85%E1%85%A6%E1%84%89%E1%85%B3-%E1%84%87%E1%85%A2%E1%84%91%E1%85%A9) 님의 글을 보고 실습을 진행 했습니다.

`minukube`와 `kubectl`을 설치 했다면 바로 실습을 진행 할 수 있습니다.

## Minikube 란
>miniKube는 macOs, Linux 및 Windows에서 로컬 Kubernetes 클러스터를 빠르게 설정해주는 도구이다. 즉, miniKube를 이용하면 손쉽게 로컬에서 쿠버네티스 클러스터를 만들 수 있다. 심지어 여러 클러스터를 관리하는 것도 가능하다.
쿠버네티스 클러스터를 실행하려면 최소한 sceduler, controller, api-server, etcd, kubelet, kube-proxy를 설치해야 하고 필요에 따라 dns, ingress controller, storage class등을 설치해야 한다. 쿠버네티스는 설치 또한 중요한 과정이지만 처음 공부할땐 설치보단 실질적인 사용법을 익히는 게 중요하다.
이러한 설치를 쉽고 빠르게 하기 위한 도구가 minikube이다. minikube는 windows, macOs, linux에서 사용할 수 있고 다양한 가상 환경을 지원하여 대부분의 환경에서 문제없이 동작한다.
[출처](https://velog.io/@yoojinjangjang/Minikube)


# 1. Minikube 실행 
`minikube start`명령어를 통해 `minikube`를 실행해줍니다.
![](/images/post/6-kubernetes-practice/1.png)

# 2. YAML 파일 작성
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
        - image: mariadb:10.7
          name: mysql
          env:
            - name: MYSQL_DATABASE
              value: wordpress
            - name: MYSQL_ROOT_PASSWORD
              value: password
          ports:
            - containerPort: 3306
              name: mysql

---
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
        - image: wordpress:5.9.1-php8.1-apache
          name: wordpress
          env:
            - name: WORDPRESS_DB_HOST
              value: wordpress-mysql
            - name: WORDPRESS_DB_NAME
              value: wordpress
            - name: WORDPRESS_DB_USER
              value: root
            - name: WORDPRESS_DB_PASSWORD
              value: password
          ports:
            - containerPort: 80
              name: wordpress

---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  type: NodePort
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
```
정확한 뜻은 아직 모르기 때문에 따라 적었습니다.


# 3. kubectl apply
`kubectl`의 `apply`명령어를 사용해 원하는 상태를 적용합니다.
![](/images/post/6-kubernetes-practice/2.png)

적용이 되었다면 `kubectl get all`명령어를 사용해 확인합니다.

![](/images/post/6-kubernetes-practice/3.png)

이제 배포한 `wordpress`를 실행해 봅시다.

# 4. 실행
`minikube service ${서비스명}`을 실행합니다. `${서비스명}`에는
`kubectl get all`명령어에서 `service/wordpress`이기 때문에 `wordpress`를 입력하면 됩니다.
![](/images/post/6-kubernetes-practice/4.png)
이렇게 반환하며 브라우저 창이 열립니다.
![](/images/post/6-kubernetes-practice/5.png)


# 5. 2개로 복제
현재는 `kubectl get all`명령어를 확인할시 `pod/wordpress`는 하나 밖에 없습니다. 만약 `kubectl delete ${pod명}`를 통해 pod를 삭제한다면 
![](/images/post/6-kubernetes-practice/6.png)
이렇게 접속이 끊깁니다. 하지만 곧바로 쿠버네티스가 다시 살리기 때문에 곧 다시 접속이 가능해집니다.

이제 2개로 복제해서 하나를 삭제해도 접속이 끊기지 않도록 하겠습니다.

`wordpress-k8s.yml`파일을 열어서
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
        - image: wordpress:5.9.1-php8.1-apache
          name: wordpress
          env:
            - name: WORDPRESS_DB_HOST
              value: wordpress-mysql
            - name: WORDPRESS_DB_NAME
              value: wordpress
            - name: WORDPRESS_DB_USER
              value: root
            - name: WORDPRESS_DB_PASSWORD
              value: password
          ports:
            - containerPort: 80
              name: wordpress
```
`wordpress`의 Deployment부분에 `spec`아래에 `replicas: 2`를 적어주고
다시 `kubectl apply -f wordpress-k8s.yml`을 통해 적용을 합니다.
그리고 `kubectl get all`을 통해 확인해보면

![](/images/post/6-kubernetes-practice/7.png)

 아까와 달리 pod/wordpress가 두개가 된 것을 볼 수 있습니다. 이제 하나의 pod/wordpress가 삭제되어도 접속이 끊기지 않습니다.


![](/images/post/6-kubernetes-practice/8.png)
