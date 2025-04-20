---
title: "Kubespray 사용 시 Metallb 설치"
author: Bumgu
date: 2024-09-22
categories: 
    - DevOps
tags: 
    - Kubernetes
    - Kubespray
    - Metallb
    - Ansible
slug: "kubespray_metallb_install"
---
 [지난번 ](https://blog.bumgu.com/post/2024/09/15/kubernetes_clustering_with_kubespray/) `kubespray` 와 `ansible`을 활용해서 쿠버네티스 클러스터링을 할 때, 마지막 metallb 설치에 실패했는데, 오늘 설치해보겠습니다.
 


### 1. MetalLB 설치
[공식문서](https://metallb.universe.tf/installation/)를 보고 설치했습니다.


```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
```


으로 설치합니다.

#### 1-1. kube-proxy configMap 수정
`MetalLB`는 IPVS 모드에서 ARP 응답을 엄격하게 제어하여 로드밸런싱을 수행합니다. 이 모드에서 strictARP를 활성화하면, kube-proxy는 노드가 할당된 IP에 대한 ARP 응답을 명확하게 지정된 노드에서만 처리하도록 합니다.

`kubectl edit cm -n kube-system kube-proxy`로 편집을 열고
`strictARP: false` 부분을 `strictARP: true`로 수정해줍니다.

저장하고 빠져나온뒤엔
`kubectl rollout restart daemonset -n kube-system kube-proxy`로 `kube-proxy`를 재시작 해줍니다.


#### 1-2. IpAddressPool 설정
지난번 이 `IpAddressPool`리소스를 생성하면 kubectl이 고장났는데, 
그 이유는

`controlplane` : 192.168.64.176
`node1` : 192.168.64.177
`node2` : 192.168.64.178
이었는데 

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: cluster-ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.64.176-192.168.64.178
```
이렇게 작성한후 apply를 했습니다.
바보같이 노드들의 아이피와 같은 아이피 대역을 설정해서 ip충돌이 일어나서 해당 ip로 노드에 접근하려는 시도가 서비스 ip로 처리되어 `kubectl`을 통한 APIserver 접근이 차단되어 `kubectl`명령어가 고장난것 이었습니다.

그렇기에, 충돌이 되지않는 ip로설정합니다.

* ipAddressPool.yaml
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: cluster-ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.64.180-192.168.64.189
```
이제 생성하면 정상적으로 생성이 되고, `kubectl`명령어도 여전히 작동합니다.


#### 1-3. L2Advertisement 설정
`IpAddressPool`리소스를 생성하였으니, 그 Pool을 사용합니다.

* l2Advertisement.yaml
```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: cluster-l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - cluster-ip-pool
```
`spec.ipAddressPools[0]`은 `IPAddressPool` manifest의 `metadata.name`으로 작성합니다.

#### 1-1~1-3 을 스크립트로 한번에
기왕 자동화를 하다보니 더 편하게 더 한번에 하고싶은 마음이 생겨 스크립트를 작성했습니다.


* install-metllb.sh
```bash
#!/bin/bash
# strictARP: true 로 수정
kubeproxytmp=$(mktemp)
kubectl get cm -n kube-system kube-proxy -o yaml > $kubeproxytmp
sed -i '' 's/strictARP: false/strictARP: true/' $kubeproxytmp
kubectl replace -f $kubeproxytmp -n kube-system
rm $kubeproxytmp


# Metallb 설치
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
echo -e "\n MetalLB Successfully Installed.\n"

echo "\nWating to ensure CRDs are installed... \n"

sleep 5
# Metallb 설치 되었는지 확인
kubectl get all -n metallb-system

# IpAddressPool 생성
pooltmp=$(mktemp)

cat<<EOF > $pooltmp
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: cluster-ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.64.180-192.168.64.189
EOF

kubectl apply -f $pooltmp
echo -e "\nSuccessfully created IPAddressPool \n"
rm $pooltmp
# L2Advertisement 생성
l2advertisementtmp=$(mktemp)

cat<<EOF > $l2advertisementtmp
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: cluster-l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
    - cluster-ip-pool
EOF

kubectl apply -f $l2advertisementtmp
echo -e "\nSuccefully created L2Advertisement"
rm $l2advertisementtmp
```



#### 1-4. ingress controller external-ip 부여 확인

이제 필요한 리소스를 모두 생성했습니다.

`ingress-nginx`를 조회해보면
![](/images/post/14-kubespray2/1.png)


`external-ip`가 제가 설정한 `IpAddressPool`에서 부여된것을 확인 할 수 있습니다.

### 2. 테스트!
모든 설정을 했으니, `LoadBalancer`타입의 서비스를 생성해서 정상적으로 작동하는지 확인해보겠습니다.
2개의 nginx deployment와 service를 생성하고, configmap에 서로 다른 페이지를 html을 저장해 보여줍니다.
이후 ingress를 생성해 정상작동하는지 확인하는 흐름입니다.
간단히 그림으로 보자면,
![](/images/post/14-kubespray2/2.png)
라고 보면 되겠습니다.

즉, 요청은 `Ingress-nginx`로 요청을 보내고 (부여된 External-IP)받은 요청을 `Ingress`리소스에 보내고, 거기서 정해진대로 `Service`로 보내집니다.

#### 2-0. Ingress-nginx 의 Selector
테스트하기전에, `Ingress-nginx`의 endpoints를 조회해보면, `none`으로 나오는것을 확인 할 수 있습니다. 
![](/images/post/14-kubespray2/8.png)
이유는, `Ingress-nginx`의 Pod의 label과 Service에서 Selector의 label값이 다르기때문에, 올바른 Pod를 선택하지 못해서 endpoints가 없어서(선택하지 못해서)`none`이 나오는겁니다.

`kubectl edit svc -n ingress-nginx ingress-nginx`
로 편집을 열어주고
`spec.selector.app.kubernetes.io/port-of`를`spec.selector.app.kubernetes.io/part-of`로 바꿔주고 저장하고 빠져나옵니다. 
빠져나온 후 다시 endpoints를 조회해보면 값이 나오는것을 확인할 수 있습니다.





#### 2-1. Deplyment, Service 생성
* nginx-deployments.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app1
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-app1
  template:
    metadata:
      labels:
        app: nginx-app1
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: app1-vol
          mountPath: /usr/share/nginx/html
      volumes:
      - name: app1-vol
        configMap:
          name: app1-html

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc1
  namespace: test
spec:
  selector:
    app: nginx-app1
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app2
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-app2
  template:
    metadata:
      labels:
        app: nginx-app2
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: app2-vol
          mountPath: /usr/share/nginx/html
      volumes:
      - name: app2-vol
        configMap:
          name: app2-html

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc2
  namespace: test
spec:
  selector:
    app: nginx-app2
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
  ```
 2개의 nginx deployment와 service를 생성합니다.
 
 
 
 #### 2-2. ConfigMap 생성
 * nginx-configmap.yaml
 ```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app1-html
  namespace: test
data:
  index.html: |
    <html>
    <head><title>App 1</title></head>
    <body>
    <h1>This is App 1</h1>
    </body>
    </html>
 ---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app2-html
  namespace: test
data:
  index.html: |
    <html>
    <head><title>App 2</title></head>
    <body>
    <h1>This is App 2</h1>
    </body>
    </html>
 ```
 
 
#### 2-3. Ingress 생성
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: test
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: nginx-svc1
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: nginx-svc2
            port:
              number: 80
```
 현재 domain은 없으니, host는 지정하지않고 생성합니다.
 
 `kubectl get ingress`로 확인해보면
![](/images/post/14-kubespray2/9.png)
Address가 워커노드의 IP입니다. 하지만 이 주소가 아닌, `Ingress-nginx` Service의 부여된 External-IP로 접속하면 됩니다.

`http://192.168.64.180/app1`으로 접속해봅니다.

![](/images/post/14-kubespray2/10.png)

이제

`http://192.168.64.180/app2`으로 접속해봅니다.

![](/images/post/14-kubespray2/11.png)




정상적으로 ingress를 타고 정해진 서비스로 접속합니다.

---


### 마무리

Kubespray를 활용하면서, 환경자동화에 대한 관심이 더 커졌습니다.
ArgoCD를 이용해 GitOps전략과 App of Apps 패턴을 적용한 프로젝트도 소개하겠습니다.

포스팅은 여기서 마치도록 하겠습니다. 감사합니다.
