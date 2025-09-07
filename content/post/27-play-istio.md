---
title: "(1)Istio 맛보기"
author: Bumgu
draft: false
date: 2025-09-07T11:17:37+09:00
categories: 
    - DevOps
tags: 
    - kubernetes
    - istio
slug: "play-with-istio"
---

# 1. Istio?
많은 서비스들이 마이크로서비스아키텍처(MSA)로 구성 되어있습니다.  
MSA는 각 서비스 간의 의존도를 낮추어 장애 전파를 낮추는데 특히 효과적 입니다.  
하지만 MSA에서 서비스가 많아지면 많아질수록 복잡도가 증가하고, 어떤 서비스가 어떤 서비스를 호출하는지, 통신은 안전한지, 트러블 슈팅이 힘들어지는 문제 등이 있습니다.  
이러한 복잡한 문제를 해결하는데 도움을 주는 도구가 Istio입니다.

Istio는 Service Mesh의 대표적인 오픈소스 소프트웨어 입니다.  
마이크로 서비스 간의 통신을 제어하고, 가시성을 제공하며 보안을 강화합니다.  
마치 교통경찰이 수신호를 하여 교통흐름을 통제하고 있다고 생각하면 됩니다.  

## 1-1. Istio의 주요 기능
- 트래픽 관리
    - 트래픽 분배, A/B 테스트, 카나리 배포 등 복잡한 라우팅 규칙을 코드 수정 없이 설정 가능
    - ex) 신규버전을 10%의 사용자에게만 먼저 보여주고 문제가 없으면 점진적으로 늘려가는 것
- 보안
    - 서비스 간의 통신을 자동으로 암호화(mTLS)하고, 서비스의 신원을 확인하여 보안을 강화
- 관찰 가능성(Observability)
    - 모든 서비스 간의 통신을 상세하게 기록 및 모니터링 하여 병목현상이나 오류를 쉽게 찾아낼 수 있도록 도움
-  정책 적용
    - 특정 서비스는 1초에 몇 번만 호출되도록 제한하거나, 특정 사용자만 접근 가능하도록 하는 등의 정책을 쉽게 적용 가능


# 2. Istio 설치
Helm chart로 Istio를 설치하겠습니다. 먼저 repo를 등록하고 업데이트 합니다.  

`helm repo add istio https://istio-release.storage.googleapis.com/charts`
`helm repo update`


## 2-1. istio base 설치
istio base는 클러스터 전역에 CRD를 배포합니다.  
`helm install isitio-base istio/base -n istio-system --set defaultRevision=default --create-namespace`

```
$ kubectl get crd | grep istio
authorizationpolicies.security.istio.io      2025-09-07T02:32:00Z
destinationrules.networking.istio.io         2025-09-07T02:32:00Z
envoyfilters.networking.istio.io             2025-09-07T02:32:00Z
gateways.networking.istio.io                 2025-09-07T02:32:00Z
peerauthentications.security.istio.io        2025-09-07T02:32:00Z
proxyconfigs.networking.istio.io             2025-09-07T02:32:00Z
requestauthentications.security.istio.io     2025-09-07T02:32:00Z
serviceentries.networking.istio.io           2025-09-07T02:32:00Z
sidecars.networking.istio.io                 2025-09-07T02:32:00Z
telemetries.telemetry.istio.io               2025-09-07T02:32:00Z
virtualservices.networking.istio.io          2025-09-07T02:32:00Z
wasmplugins.extensions.istio.io              2025-09-07T02:32:00Z
workloadentries.networking.istio.io          2025-09-07T02:32:00Z
workloadgroups.networking.istio.io           2025-09-07T02:32:00Z
```

## 2-2 istiod 설치
istiod는 istio discovery를 배포합니다.  
`helm install istiod istio/istiod -n istio-system --wait`


## 2-3 istio-ingress 설치
`helm install istio-ingressgateway istio/gateway -n istio-ingress --create-namespace`  
`nginx ingress controller` 처럼 ingress controller의 역할이지만, 서비스 메쉬와 통합하여 더 고급 기능을 활용할 수 있습니다.


# 3. 실습
## 3-1. 샘플 앱 배포
Istio의 공식 샘플 앱을 배포합니다.  
`curl -L https://istio.io/downloadIstio | sh -`  
`kubectl create ns bookinfo`  
`kubectl apply -f istio-1.27.1/samples/bookinfo/platform/kube/bookinfo.yaml`  
배포가 되었다면 port-forwarding을 해 브라우저 접속이 가능하도록 합니다.  
`kubectl port-forwarding <productpage-pod> 9080:9080`  
브라우저에 접속해 `localhost:9080/productpage` 로 접근합니다.  

- **reviews-v1**
![](/images/post/27-play-istio/1.png)
- **reviews-v2**
![](/images/post/27-play-istio/2.png)
- **reviews-v3**
![](/images/post/27-play-istio/3.png)

페이지를 새로고침할 때마다 'Book Reviews' 섹션의 별 색깔(빨간색, 검은색)이 바뀌거나 아무것도 나타나지 않는 것을 볼 수 있습니다.  
이는 여러 버전의 reviews 서비스가 라운드 로빈 방식으로 호출되기 때문입니다.

## 3-2. DestinationRule, VirtualService 생성
- `VirtualService`: 어디로 보낼 것인가?(Routing Rules)
    - 어떤 조건의 트래픽(subset: v1)으로 보내 와 같은 라우팅 규칙을 정의
    - 트래픽 흐름을 제어
- `DestinationRule`: subset이 무엇인가?
    - pod들을 특정 그룹으로 묶고, 각 그룹에 v1, v2와 같은 이름을 붙임.
    - subset을 실제로 정의하는 역할

`kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml`  

## 3-2. 트래픽 제어
### A. reviews를 특정버전으로 라우팅
- Bookinfo앱의 `reviews`서비스는 v1, v2, v3 세 가지 버전이 있습니다.
v1은 별점이 없는 상태입니다.
`kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml`  
명령어로 모든서비스 v1을 적용합니다.  
이제는 페이지를 새로고침해도 `reviews v1`만 나타납니다.  
`virtual-service-all-v1.yaml` 파일을 보면,
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
```
`reviews`의 `subset`을 v1으로 고정하였기 때문에 모든 트래픽은 v1으로 라우팅 되기 때문입니다.

이제 새로운 VirtualService를 생성해 v2로만 라우팅 되게 해보겠습니다.  
우선 기존 `reviews VirtualService`를 삭제합니다.  
`kubectl delete virtualservices.networking.istio.io reviews`
```yaml
# reviews-v2.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
      - destination:
          host: reviews
          subset: v2
```
이제 reviews는 새로고침을 해도 v2(검정색 별)만 나타납니다.

### B. 특정 사용자만 특정버전으로 라우팅
우선 아까 만든 v2  VirtualService를 삭제하고 `virtual-service-all-v1.yaml`을 적용합니다.
이후 `virtual-service-reviews-jason-v2-v3.yaml` 을 적용합니다. 이후 jason 유저로 로그인합니다.(Username은 jason으로 하고 비밀번호는 아무 비밀번호)  
![](/images/post/27-play-istio/4.png) 로그인을 하고나면 v2(검정색 별)만 보입니다. 다른유저 혹은 로그아웃을 하면 v3(빨간색 별)만 보입니다.  
* jason 유저
![](/images/post/27-play-istio/5.png)
* john 유저
![](/images/post/27-play-istio/6.png)
`virtual-service-reviews-jason-v2-v3.yaml` 내용을 보면,
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
```
이렇게 되어있습니다. 이는 http 요청 header안에 `end-user`의 값이 `jason`일 경우에는 v2이고, 그 이외에는 v3로 라우팅 한다는 뜻입니다.  
여기에 end-user가 `john`이면 v1으로 라우팅 되는 룰만 추가해보겠습니다.
* jason 유저 (v2)
![](/images/post/27-play-istio/7.png)
* john 유저 (v1)
![](/images/post/27-play-istio/8.png)
* no유저 (나머지 모든 유저) (v3)
![](/images/post/27-play-istio/9.png)


# 4. 마무리
Istio를 직접 설치하고 샘플 애플리케이션에 적용하면서 서비스 메시가 실제로 어떻게 동작하는지 조금 더 피부로 느낄 수 있었습니다. 단순히 이론만 접했을 때는 막연했던 개념들이, 실습을 통해 트래픽 흐름이 어떻게 제어되고 각 버전으로 라우팅되는지 눈으로 확인하니 확실히 이해가 더 잘 되었던 것 같습니다.

아직은 기본적인 트래픽 관리와 라우팅 정도만 다뤘지만, 보안(mTLS), Policy, Observability 등 아직 공부할 부분이 많이 남아있습니다. 앞으로는 실제 운영 환경에서 고민해야 할 부분이나, Istio가 제공하는 다양한 고급 기능들도 차근차근 실습해 보면서 정리해 보고자 합니다.  
