---
title: "Nginx와 Envoy Gateway의 라우팅"
author: Bumgu
draft: true
date: 2026-08-02T16:07:35+09:00
categories:
    - DevOps
    - SRE
tags:
    - Envoy
    - Kubernetes
slug: "envoy-gateway-proxy"
---

# nginx 정규식 라우팅을 Envoy Gateway로 옮길 수 있을까

엣지 게이트웨이를 OpenResty(nginx)에서 Envoy Gateway로 전환하려면 먼저 답해야 하는
질문이 하나 있었습니다. **nginx conf에 있는 정규식 location 설정을 Gateway API로
그대로 옮길 수 있는가.**

문서를 읽는 것으로는 답이 나오지 않았습니다. Gateway API 스펙이 정규식 경로 매칭을
Extended 등급으로 두고 **우선순위를 아예 규정하지 않기** 때문입니다. 구현체 재량이라고만
적혀 있습니다.

그래서 kind에 Envoy Gateway를 올려 직접 측정했습니다. 이 글은 그 기록입니다.
모든 매니페스트를 포함해 그대로 재현할 수 있게 썼습니다.

측정 환경은 아래와 같습니다.

| 항목 | 버전 |
|---|---|
| Envoy Gateway | v1.8.3 |
| Gateway API | v1 (`gateway.networking.k8s.io/v1`) |
| kind | colima 위 |
| MetalLB | v0.14.8 |

**스펙 미규정 영역을 측정한 것이므로 이 결과는 EG v1.8.3 한정입니다.** 버전을 올릴 때는
다시 재보셔야 합니다.

---

## 무엇을 측정했나

네 가지입니다.

1. **정규식 매칭 범위** : nginx는 부분매칭입니다. Envoy도 그런가
2. **우선순위** : 정규식과 PathPrefix가 겹치면 누가 이기는가. 순서에 의존하는가
3. **그룹 캡처 rewrite** : `(\d+)` 로 잡은 값을 경로 재작성에 쓸 수 있는가
4. **메서드 매칭** : 같은 경로를 GET/POST로 분기할 수 있는가

---

## 환경 구성

### kind 클러스터

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```bash
kind create cluster --name routing-lab --config kind-config.yaml
```

### MetalLB

kind에는 LoadBalancer 서비스에 IP를 줄 주체가 없습니다. Envoy Gateway가 만드는
서비스가 `<pending>` 상태로 멈추는 것을 막기 위해 MetalLB를 넣습니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
kubectl wait -n metallb-system --for=condition=ready pod --selector=app=metallb --timeout=90s

# kind 도커 네트워크 서브넷을 먼저 확인한다
docker network inspect -f '{{.IPAM.Config}}' kind
```

확인한 서브넷 안에서 풀 범위를 잡습니다.

```yaml
# metallb-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: kind-pool
  namespace: metallb-system
spec:
  addresses:
    - 172.18.255.200-172.18.255.250   # docker network inspect 결과에 맞춰 조정
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: kind-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - kind-pool
```

```bash
kubectl apply -f metallb-pool.yaml
```

### Envoy Gateway

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.8.3 \
  -n envoy-gateway-system --create-namespace

kubectl wait -n envoy-gateway-system deployment/envoy-gateway \
  --for=condition=Available --timeout=5m
```

### GatewayClass와 Gateway

```yaml
# gatewayclass.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eg
  namespace: default
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All        # 다른 네임스페이스의 HTTPRoute 도 붙을 수 있게
```

```bash
kubectl apply -f gatewayclass.yaml -f gateway.yaml
kubectl get gateway -n default eg
```

`PROGRAMMED` 컬럼이 `True`가 되어야 합니다. 여기가 `False`면 이후 모든 curl이
연결 실패로 나오므로 먼저 해결해야 합니다.

`allowedRoutes.namespaces.from`을 `All`로 둔 것은 실습 편의를 위해서입니다.
운영에서는 `Selector`로 좁히는 편이 낫습니다.

---

## 검증용 백엔드

라우팅 검증에서 가장 중요한 것은 **"어느 백엔드로 갔는지"와 "백엔드가 어떤 경로를
받았는지"를 응답만 보고 알 수 있어야 한다**는 점입니다.

처음에 `nginx:alpine`을 썼는데 두 백엔드가 똑같은 기본 페이지를 반환해서 구분이
안 됐습니다. `ealen/echo-server`로 바꿨습니다. 요청 경로와 헤더를 JSON으로
되돌려주고, 환경변수에 파드 이름이 들어 있어 백엔드 식별까지 됩니다.

```yaml
# echo.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: routing-lab
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-a
  namespace: routing-lab
  labels:
    app.kubernetes.io/name: echo-a
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: echo-a
  template:
    metadata:
      labels:
        app.kubernetes.io/name: echo-a
    spec:
      containers:
        - name: echo
          image: ealen/echo-server:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 10m
              memory: 48Mi
            limits:
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: echo-a
  namespace: routing-lab
spec:
  selector:
    app.kubernetes.io/name: echo-a
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-b
  namespace: routing-lab
  labels:
    app.kubernetes.io/name: echo-b
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: echo-b
  template:
    metadata:
      labels:
        app.kubernetes.io/name: echo-b
    spec:
      containers:
        - name: echo
          image: ealen/echo-server:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 10m
              memory: 48Mi
            limits:
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: echo-b
  namespace: routing-lab
spec:
  selector:
    app.kubernetes.io/name: echo-b
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f echo.yaml
kubectl wait -n routing-lab --for=condition=Available deploy/echo-a deploy/echo-b --timeout=90s
```

`ealen/echo-server`는 기본으로 **80번 포트**를 리슨합니다. `PORT` 환경변수로 바꿀 수
있지만, 그러면 `targetPort`도 같이 바꿔야 합니다. 둘 중 하나만 바꾸면 503이 납니다
(뒤의 삽질 기록 참조).

### 응답 확인용 헬퍼

측정 내내 이 함수를 씁니다. 상태 코드와 파드 이름을 한 줄로 보여주고, JSON이 아닌
응답(404나 503 본문)에도 깨지지 않습니다.

```bash
H='Host: routing.test'; U=127.0.0.1:8080

probe() {
  local method=${1} path=${2}
  local r code pod
  r=$(curl -s -X "$method" -w '\n%{http_code}' -H "$H" "$U$path")
  code=$(tail -1 <<<"$r")
  pod=$(sed '$d' <<<"$r" | jq -r '.environment.HOSTNAME' 2>/dev/null)
  printf '%-8s %-40s %s %s\n' "$method" "$path" "$code" "${pod:--}"
}

# 경로만 보고 싶을 때 (rewrite 검증용)
rewritten() {
  curl -s -H "$H" "$U$1" | jq -r '.http.originalUrl'
}
```

파드 이름은 `.environment.HOSTNAME`입니다. `.host.hostname`은 요청의 Host 헤더가
들어오므로 백엔드 식별에 쓸 수 없습니다. 여기서 한 번 헛걸음했습니다.

### 접속

```bash
kubectl get svc -n envoy-gateway-system
kubectl port-forward -n envoy-gateway-system svc/envoy-default-eg-<해시> 8080:80
```

**반드시 `svc/`로 해야 합니다.** Envoy proxy 파드는 non-root로 돌아서 컨테이너 내부에서
80이 아닌 상단 포트를 리슨합니다. 파드에 직접 port-forward하면 connection refused가
납니다.

호스트명으로 `.local`을 쓰지 마십시오. macOS mDNS 때문에 요청마다 5초씩 지연됩니다.
`.test`를 씁니다.

---

## 검증 1 : 정규식 매칭 범위

nginx의 `location ~`은 **부분매칭**입니다. `^/foo`라고 쓰면 `/foo/bar/baz`에도
매칭됩니다. Envoy도 그런지 봅니다.

```yaml
# route-regex.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: regex-only
  namespace: routing-lab
spec:
  parentRefs:
    - name: eg
      namespace: default        # Gateway 와 ns 가 다르면 명시 필수
  hostnames:
    - "routing.test"
  rules:
    - matches:
        - path:
            type: RegularExpression
            value: '^/v3/customer/bookings/\d+/coupons'
      backendRefs:
        - name: echo-b
          port: 80
```

`parentRefs.namespace`를 빠뜨리면 HTTPRoute 자신의 네임스페이스에서 Gateway를 찾습니다.
`routing-lab`에는 Gateway가 없으니 못 찾는데, **에러 메시지 없이 `status`가 통째로
비어버립니다.** 원인 추적이 꽤 괴로운 종류의 실수입니다.

정규식은 **작은따옴표**로 감쌉니다. YAML 큰따옴표 안에서는 `\d`가 이스케이프로
해석되려 해서 `\\d`로 써야 합니다.

```bash
kubectl apply -f route-regex.yaml

# 적용 확인 : RE2 가 거부하는 문법이면 여기서 Accepted=False 로 뜬다
kubectl get httproute -n routing-lab regex-only \
  -o jsonpath='{.status.parents[*].conditions[*]}' | jq .
```

### 측정

```bash
for p in \
  '/v3/customer/bookings/123/coupons' \
  '/v3/customer/bookings/123/coupons/456' \
  '/v3/customer/bookings/123/coupons?x=1' \
  '/api/v3/customer/bookings/123/coupons' \
  '/v3/customer/bookings/abc/coupons' ; do
  probe GET "$p"
done
```

### 결과

| 요청 | 결과 | nginx `location ~` |
|---|---|---|
| `.../123/coupons` | 200 | 같음 |
| `.../123/coupons/456` | **404** | **다름** : nginx는 매칭 |
| `.../123/coupons?x=1` | 200 | 같음 (쿼리스트링은 경로 매칭에서 제외) |
| `/api/v3/...` | 404 | 같음 (`^` 앵커) |
| `.../abc/coupons` | 404 | 같음 (`\d+`) |

**Envoy는 정규식을 경로 전체에 대해 매칭합니다.** `safe_regex` 경로 매처의 동작이고,
엔진은 RE2입니다.

nginx conf를 그대로 옮기면 하위 경로가 전부 404로 죽습니다. 변환 규칙이 필요합니다.

```
nginx  location ~ ^/foo        →  Gateway  ^/foo.*
nginx  location ~ ^/foo$       →  Gateway  ^/foo    (또는 Exact)
nginx  location ~ ^/foo/[^/]+  →  Gateway  ^/foo/[^/]+.*
```

`.*`를 붙이면 `/foobar`도 매칭되는데, **nginx도 원래 그렇게 매칭했습니다.** 즉 `.*`
추가가 동작 보존이고 빼는 것이 동작 변경입니다. 끝에 `$`가 있는 것만 예외로 두고
기계적으로 적용할 수 있습니다.

### RE2 제약

Envoy는 RE2를 씁니다. PCRE와 다른 점이 있습니다.

| 문법 | 지원 |
|---|---|
| `\d`, `[^/]+`, `{n,m}`, `(?i)` | ✅ |
| lookahead `(?=)`, lookbehind `(?<=)` | ❌ |
| 패턴 내 역참조 `\1` | ❌ |

미지원 문법을 쓰면 HTTPRoute가 `Accepted: False`로 거부됩니다. 조용히 틀리는 것보다
낫지만, 옮기기 전에 conf에서 미리 색출해두는 편이 좋습니다.

nginx의 대소문자 무시(`~*`)는 `(?i)` 접두사로 옮깁니다.

---

## 검증 2 : 우선순위

여기가 이번 측정의 본편입니다. `.*`를 붙이면 정규식이 넓어지면서 다른 규칙과 겹치는
범위가 늘어나는데, **정규식 우선순위는 스펙 미규정**입니다.

스펙이 규정한 순위는 여기까지입니다.

1. `Exact` 경로 매칭
2. `PathPrefix` 중 문자 수가 가장 긴 것
3. 메서드 매칭이 있는 것
4. 헤더 매칭 개수가 많은 것
5. 쿼리 파라미터 매칭 개수가 많은 것

`RegularExpression`은 이 목록에 없습니다. 그래서 네 가지 조합을 측정했습니다.

### A : 정규식 vs PathPrefix

```yaml
# route-priority.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: prio-regex-prefix
  namespace: routing-lab
spec:
  parentRefs:
    - name: eg
      namespace: default
  hostnames:
    - "routing.test"
  rules:
    # (1) 넓은 prefix → echo-a
    - matches:
        - path:
            type: PathPrefix
            value: /v3/customer
      backendRefs:
        - name: echo-a
          port: 80
    # (2) 좁은 정규식 → echo-b
    - matches:
        - path:
            type: RegularExpression
            value: '^/v3/customer/bookings/\d+/coupons.*'
      backendRefs:
        - name: echo-b
          port: 80
```

측정 전에 검증 1의 라우트를 지워야 합니다. 같은 호스트에 여러 HTTPRoute가 붙으면
리소스 간 tiebreak(생성시각 → 이름순)이 섞여서 판정이 흐려집니다.

```bash
kubectl delete -f route-regex.yaml
kubectl apply -f route-priority.yaml

probe GET /v3/customer/bookings/123/coupons
probe GET /v3/customer/bookings/123/coupons/456
probe GET /v3/customer/profile
```

```
GET      /v3/customer/bookings/123/coupons        200 echo-b-548f5495f-c7wv8
GET      /v3/customer/bookings/123/coupons/456    200 echo-b-548f5495f-c7wv8
GET      /v3/customer/profile                     200 echo-a-868fdcc6bd-xl679
```

**정규식이 이겼습니다.** `/profile`은 정규식에 매칭되지 않아 prefix로 떨어졌으니
prefix 규칙도 정상 동작합니다.

nginx도 매칭 순위가 `=` exact → `^~` prefix → **정규식** → 일반 prefix 최장이라
정규식이 일반 prefix를 이깁니다. 이 부분은 동작이 보존됩니다.

### B : 순서를 뒤집으면

rule (1)과 (2)의 위치만 바꿔 재적용하고 같은 요청을 보냅니다.

```
GET      /v3/customer/bookings/123/coupons        200 echo-b-548f5495f-c7wv8
GET      /v3/customer/bookings/123/coupons/456    200 echo-b-548f5495f-c7wv8
GET      /v3/customer/profile                     200 echo-a-868fdcc6bd-xl679
```

**동일합니다.** YAML 배치 순서와 무관하게 특성 기반으로 정렬합니다.

이것은 nginx와 다른 점입니다. nginx는 정규식 location을 **파일에 쓴 순서대로
첫 매칭이 이기는** 방식입니다. 반면 EG는 순서를 보지 않습니다.

운영 관점에서는 EG 쪽이 낫습니다. 매니페스트를 여러 파일로 쪼개도 라우팅이
결정적이고, 파일 병합 순서 같은 것을 걱정하지 않아도 됩니다.

### C : prefix가 더 길 때

스펙의 "PathPrefix 최장 우선"이 정규식과의 비교에도 끼어드는지 봅니다.
prefix를 정규식보다 구체적으로 만들었습니다.

```yaml
# route-priority-c.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: prio-long-prefix
  namespace: routing-lab
spec:
  parentRefs:
    - name: eg
      namespace: default
  hostnames:
    - "routing.test"
  rules:
    # 긴 prefix (25자) → echo-a
    - matches:
        - path:
            type: PathPrefix
            value: /v3/customer/bookings/123
      backendRefs:
        - name: echo-a
          port: 80
    # 정규식 → echo-b
    - matches:
        - path:
            type: RegularExpression
            value: '^/v3/customer/bookings/\d+/coupons.*'
      backendRefs:
        - name: echo-b
          port: 80
```

```
GET      /v3/customer/bookings/123/coupons        200 echo-b-548f5495f-c7wv8
GET      /v3/customer/bookings/123/coupons/456    200 echo-b-548f5495f-c7wv8
GET      /v3/customer/bookings/123/reviews        200 echo-a-868fdcc6bd-xl679
```

**정규식이 prefix 길이와 무관하게 이깁니다.** 가장 단순한 결과입니다. prefix와 정규식이
겹치는 경우를 전수 대조할 필요가 없어졌습니다.

### D : 정규식끼리 겹칠 때

실전 위험도가 가장 높은 조합입니다. conf에 정규식이 61개 있으면 서로 겹칩니다.

```yaml
# route-priority-d.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: prio-regex-regex
  namespace: routing-lab
spec:
  parentRefs:
    - name: eg
      namespace: default
  hostnames:
    - "routing.test"
  rules:
    # (1) 넓은 정규식 → echo-a
    - matches:
        - path:
            type: RegularExpression
            value: '^/v3/customer/bookings/.*'
      backendRefs:
        - name: echo-a
          port: 80
    # (2) 좁은 정규식 → echo-b
    - matches:
        - path:
            type: RegularExpression
            value: '^/v3/customer/bookings/\d+/coupons.*'
      backendRefs:
        - name: echo-b
          port: 80
```

```
GET      /v3/customer/bookings/123/coupons        200 echo-b-548f5495f-c7wv8
GET      /v3/customer/bookings/123/coupons/456    200 echo-b-548f5495f-c7wv8
GET      /v3/customer/bookings/123/reviews        200 echo-a-868fdcc6bd-xl679
```

넓은 정규식이 YAML 위쪽에 있는데도 **좁은 쪽이 이겼습니다.** 순서를 뒤집어도
결과가 같았습니다.

### 남은 미확인

**좁은 쪽이 이기는 판정 기준이 무엇인지는 확정하지 못했습니다.** "구체성"인지 단순
**패턴 문자열 길이**인지 구분이 안 됩니다. 측정 D에서 좁은 정규식이 문자열 길이도
더 길었기 때문입니다.

길이 기준이라면 `길지만 넓은` 정규식이 `짧지만 좁은` 정규식을 이겨 엉뚱한 백엔드로
갈 수 있습니다. 실제 conf에 그런 조합이 있으면 개별 확인이 필요합니다.

이런 종류의 불확실성이 남는다는 것이, 스펙 미규정 영역에 의존할 때 지불하는 값입니다.
전환 시 **경로 → 기대 백엔드 전수 대조표**를 회귀 테스트로 만들어야 하는 이유입니다.

---

## 검증 3 : 그룹 캡처 rewrite

`(\d+)`로 잡은 값을 경로 재작성에 쓸 수 있는지 봅니다.

표준 Gateway API의 `URLRewrite` 필터에는 `ReplacePrefixMatch`와 `ReplaceFullPath`만
있습니다. 정규식 치환이 없습니다. Envoy Gateway는 `HTTPRouteFilter`라는 별도 CRD로
이 기능을 제공합니다.

CRD가 분리된 이유는 표준 스펙을 깨지 않기 위해서입니다. 표준이 정해둔 `ExtensionRef`
라는 확장 구멍을 통해 외부 리소스를 참조합니다. 부수 효과로 같은 치환 규칙을 여러
라우트에서 재사용할 수 있습니다.

### 무엇이 궁금했나

정규식이 매칭한 범위 **뒤에 남는 경로**가 어떻게 되는지가 쟁점이었습니다.

```
요청     /nt/bookings/123/coupons/456
pattern  ^/nt/bookings/(\d+)/coupons
         └── 매칭 범위 ──────────┘└─┬─┘
                                  꼬리

결과가 /coupons/123 인가 (꼬리 버림)  /coupons/123/456 인가 (꼬리 보존)
```

꼬리가 버려진다면 61개 패턴 전부에 `(.*)`를 추가해 명시적으로 되붙여야 합니다.
보존된다면 손댈 필요가 없습니다.

두 패턴을 나란히 두고 비교했습니다. `nt` = 꼬리 미캡처, `wt` = 꼬리 캡처.

```yaml
# route-rewrite.yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: HTTPRouteFilter
metadata:
  name: rw-no-tail
  namespace: routing-lab
spec:
  urlRewrite:
    path:
      type: ReplaceRegexMatch
      replaceRegexMatch:
        pattern: '^/nt/bookings/(\d+)/coupons'
        substitution: '/coupons/\1'
---
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: HTTPRouteFilter
metadata:
  name: rw-with-tail
  namespace: routing-lab
spec:
  urlRewrite:
    path:
      type: ReplaceRegexMatch
      replaceRegexMatch:
        pattern: '^/wt/bookings/(\d+)/coupons(.*)'
        substitution: '/coupons/\1\2'
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: rewrite-capture
  namespace: routing-lab
spec:
  parentRefs:
    - name: eg
      namespace: default
  hostnames:
    - "routing.test"
  rules:
    # 꼬리 미캡처
    - matches:
        - path:
            type: RegularExpression
            value: '^/nt/bookings/\d+/coupons.*'
      filters:
        - type: ExtensionRef
          extensionRef:
            group: gateway.envoyproxy.io
            kind: HTTPRouteFilter
            name: rw-no-tail
      backendRefs:
        - name: echo-b
          port: 80
    # 꼬리 캡처
    - matches:
        - path:
            type: RegularExpression
            value: '^/wt/bookings/\d+/coupons.*'
      filters:
        - type: ExtensionRef
          extensionRef:
            group: gateway.envoyproxy.io
            kind: HTTPRouteFilter
            name: rw-with-tail
      backendRefs:
        - name: echo-b
          port: 80
```

`extensionRef`에는 `namespace` 필드가 없습니다. **HTTPRoute와 같은 네임스페이스**에
있는 리소스만 참조할 수 있습니다.

### 정규식이 두 번 나오는 이유

같은 정규식이 `matches`와 필터 양쪽에 나타나서 중복처럼 보이지만 역할이 다릅니다.

| | `matches` | 필터 `pattern` |
|---|---|---|
| 목적 | 이 rule을 적용할지 참/거짓 | 경로를 어떻게 바꿀지 |
| 매칭 방식 | **전체 경로 매칭** | **부분 매칭 후 치환** |
| 캡처 그룹 | 써도 의미 없음 | 여기서만 의미 있음 |

`matches`가 통과하지 못하면 필터는 실행되지 않습니다. 판정이 먼저, 치환이 나중입니다.
그래서 `matches`에는 `.*`를 붙여 하위 경로까지 rule에 들어오게 하고, 필터에서 실제
변환을 합니다.

### 측정

```bash
kubectl apply -f route-rewrite.yaml

for p in \
  '/nt/bookings/123/coupons' \
  '/nt/bookings/123/coupons/456' \
  '/wt/bookings/123/coupons/456' ; do
  printf '%-34s -> %s\n' "$p" "$(rewritten "$p")"
done
```

### 결과

```
/nt/bookings/123/coupons           -> /coupons/123
/nt/bookings/123/coupons/456       -> /coupons/123/456
/wt/bookings/123/coupons/456       -> /coupons/123/456
```

**꼬리가 보존됩니다.** Envoy의 `regex_rewrite`는 RE2 GlobalReplace 계열이라
매칭된 부분만 치환하고 나머지는 유지합니다.

경로 매칭(전체매칭)과 정반대라는 점이 헷갈리는 지점입니다. 정리하면 이렇습니다.

| 용도 | 변환 | 이유 |
|---|---|---|
| `matches` 정규식 | 끝에 **`.*` 추가** | 전체매칭. 없으면 하위 경로 404 |
| rewrite `pattern` | **손대지 않음** | 부분 치환. 나머지가 보존됨 |

`/nt`와 `/wt`가 같은 결과를 냈으니 `(.*)`+`\2`는 불필요한 작업입니다.

### 주의 : GlobalReplace는 여러 번 치환합니다

매칭되는 **모든 위치**를 치환합니다. `^` 앵커가 있으면 위치가 하나뿐이라 안전하지만,
앵커 없는 패턴은 여러 번 치환될 수 있습니다.

```
pattern '/coupons'  →  /a/coupons/b/coupons  에서 두 곳 모두 치환
```

nginx의 `location ~`은 앵커 없이도 동작하므로 conf에 앵커 없는 정규식이 있을 수
있습니다. rewrite 대상 패턴이 `^`로 시작하는지 확인해두는 편이 좋습니다.

### 리다이렉트에는 쓸 수 없습니다

`ReplaceRegexMatch`는 **rewrite 전용**입니다. 내부 경로 변경이라 클라이언트는 바뀐
것을 모릅니다.

`RequestRedirect` 필터에는 정규식 치환 옵션이 없어서 캡처한 값을 `Location` 헤더에
넣을 수 없습니다. Istio도 마찬가지입니다 : `VirtualService`의 `HTTPRedirect`는
`uri`가 고정 문자열이고, rewrite 쪽에만 `uriRegexRewrite`가 있습니다.

Envoy 코어의 `RedirectAction`에는 `regex_rewrite`가 있지만 어느 구현체도 상위 API로
노출하지 않습니다. `EnvoyPatchPolicy`(EG)나 `EnvoyFilter`(Istio) 같은 저수준 패치로만
닿을 수 있고, 이런 것들은 버전 업그레이드 때 조용히 깨집니다.

정규식 캡처 리다이렉트가 필요하면 **작은 리다이렉트 서비스를 따로 두는 편**이
현실적입니다.

---

## 검증 4 : 메서드 매칭

같은 경로를 메서드로 분기할 수 있는지 봅니다. 권한 체계에서 `GET`은 열고 `POST`는
더 높은 권한을 요구하는 패턴이 흔합니다.

```yaml
# route-method.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: method-match
  namespace: routing-lab
spec:
  parentRefs:
    - name: eg
      namespace: default
  hostnames:
    - "routing.test"
  rules:
    # E-1: 정규식 + GET → echo-a
    - matches:
        - path:
            type: RegularExpression
            value: '^/v3/backoffice/wages.*'
          method: GET
      backendRefs:
        - name: echo-a
          port: 80
    # E-2: 같은 정규식 + POST → echo-b
    - matches:
        - path:
            type: RegularExpression
            value: '^/v3/backoffice/wages.*'
          method: POST
      backendRefs:
        - name: echo-b
          port: 80
    # F-1: 메서드 조건 없음 → echo-a
    - matches:
        - path:
            type: PathPrefix
            value: /w2
      backendRefs:
        - name: echo-a
          port: 80
    # F-2: 같은 경로 + POST → echo-b
    - matches:
        - path:
            type: PathPrefix
            value: /w2
          method: POST
      backendRefs:
        - name: echo-b
          port: 80
```

`method`는 `path`와 같은 match 항목 안의 형제 필드입니다. 여기서 논리 관계를 정확히
알아둘 필요가 있습니다.

- **`matches` 리스트의 항목 간에는 OR** : 여러 조건 중 하나만 맞으면 됨
- **한 match 항목 안의 `path`/`method`/`headers`/`queryParams` 간에는 AND** : 전부 맞아야 함

### 측정

```bash
kubectl apply -f route-method.yaml

for m in GET POST PATCH DELETE OPTIONS; do probe $m /v3/backoffice/wages/123; done
for m in GET POST; do probe $m /w2/anything; done
```

### 결과

```
GET      /v3/backoffice/wages/123      200 echo-a-868fdcc6bd-xl679
POST     /v3/backoffice/wages/123      200 echo-b-548f5495f-c7wv8
PATCH    /v3/backoffice/wages/123      404 -
DELETE   /v3/backoffice/wages/123      404 -
OPTIONS  /v3/backoffice/wages/123      404 -

GET      /w2/anything                  200 echo-a-868fdcc6bd-xl679
POST     /w2/anything                  200 echo-b-548f5495f-c7wv8
```

메서드 분기는 정상 동작합니다. 메서드 조건이 있는 rule이 없는 rule을 이기는 것도
스펙에 규정된 순위 3번대로입니다.

### 여기서 함정을 하나 찾았습니다

**메서드 미매칭은 405가 아니라 404입니다.** 매칭되는 rule이 없는 것이므로 라우팅
관점에서는 당연하지만, nginx에서 `limit_except`로 405를 주던 경로라면 응답 코드가
바뀝니다.

더 중요한 것은 **OPTIONS도 404가 된다**는 점입니다.

Envoy의 CORS 필터는 **route-level**입니다. 라우트가 매칭된 뒤에 동작합니다. 따라서
OPTIONS가 라우트 매칭에 실패해 404로 끝나면, `SecurityPolicy`로 CORS를 걸어두더라도
**preflight가 깨집니다.**

curl로는 잘 보이지 않습니다. 브라우저에서만 실패하고, 응답 코드는 404라서 5xx 기반
알람에도 잡히지 않습니다. 조용히 깨지는 종류입니다.

### 대응 패턴

메서드 분기 경로에는 **메서드 조건 없는 rule을 같은 경로에 함께** 둡니다.
측정 F에서 확인한 대로 메서드 조건이 있는 쪽이 이기므로 분기는 그대로 유지되고,
OPTIONS는 조건 없는 rule로 떨어져 라우트를 얻습니다.

```yaml
rules:
  # 승격 권한이 필요한 쓰기 (POST)
  - matches:
      - path: { type: PathPrefix, value: /v3/backoffice/wages }
        method: POST
    backendRefs:
      - name: echo-b
        port: 80
  # 나머지 전부 : OPTIONS 를 여기서 받는다
  - matches:
      - path: { type: PathPrefix, value: /v3/backoffice/wages }
    backendRefs:
      - name: echo-a
        port: 80
```

메서드 목록에 `OPTIONS`를 명시적으로 추가하는 방법도 있지만, 분기 경로가 많아지면
누락하기 쉽습니다. 위 패턴이 안전합니다.

---

## 결과 정리

nginx 대비 동작 차이를 한 표로 모으면 이렇습니다.

| 항목 | Envoy Gateway v1.8.3 | nginx | 대응 |
|---|---|---|---|
| 정규식 매칭 범위 | **전체 경로 매칭** | 부분 매칭 | 끝에 `.*` 추가 |
| 쿼리스트링 | 경로 매칭에서 제외 | 같음 | : |
| `^` 앵커 / `\d+` | 정상 | 같음 | : |
| 대소문자 무시 | `(?i)` 접두사 | `~*` | 문법 변환 |
| 정규식 엔진 | RE2 | PCRE | lookahead·역참조 사전 색출 |
| 정규식 vs prefix | **정규식 승** (길이 무관) | 정규식 승 | : |
| rules 배치 순서 | **무관** (특성 기반 정렬) | 파일 순서 첫 매칭 승 | 순서 재현 불필요 |
| 정규식 vs 정규식 | 좁은 쪽 승 | 먼저 쓴 쪽 승 | 겹침 구간 대조 |
| rewrite 정규식 치환 | 부분 치환, 꼬리 보존 | 같음 | 패턴 그대로 |
| 메서드 미매칭 | **404** | 405 (`limit_except`) | 클라이언트 영향 확인 |
| OPTIONS | 메서드 조건 있으면 **404** | Lua/conf가 처리 | 조건 없는 rule 병치 |

### 결론

**정규식 라우팅은 옮길 수 있습니다.** 다만 세 가지를 전제로 합니다.

1. **매칭용 정규식에 `.*`를 붙이는 기계적 변환**을 빠뜨리지 않는다
2. **rewrite 패턴은 손대지 않는다** : 매칭과 규칙이 반대다
3. **메서드 조건을 붙인 경로에는 조건 없는 rule을 병치**한다 (CORS)

가장 반가운 결과는 **배치 순서 무관**이었습니다. nginx는 정규식 location의 파일
순서가 곧 우선순위라서, 대량 이전 시 순서를 보존하는 것이 큰 부담입니다. EG는 그
부담이 없어 파일을 자유롭게 쪼갤 수 있습니다.

가장 신경 쓰이는 것은 **정규식끼리의 판정 기준을 확정하지 못한 것**입니다. 스펙
미규정 영역이므로 문서로 보증받을 수 없고, 버전이 올라가면 바뀔 수 있습니다.
전수 대조 회귀 테스트를 만들어두는 것 외에 대안이 없습니다.

---

## 삽질 기록

측정보다 여기서 시간을 더 썼습니다.

### 파드에 port-forward하면 connection refused

Envoy proxy 파드는 non-root로 실행돼서 컨테이너 내부에서 80이 아닌 상단 포트를
리슨합니다. **반드시 `svc/`로** port-forward해야 합니다.

```bash
# ✅
kubectl port-forward -n envoy-gateway-system svc/envoy-default-eg-<해시> 8080:80

# ❌ connection refused
kubectl port-forward -n envoy-gateway-system pod/envoy-default-eg-<해시>-xxx 8080:80
```

### `.local` 호스트명이 요청당 5초 걸림

macOS mDNS 때문입니다. `.test`를 쓰면 됩니다. curl에서 `-H 'Host: ...'`로만
쓸 때는 DNS 조회가 없으니 무관하지만, `/etc/hosts`에 등록해 쓰는 경우 걸립니다.

### 503이 두 종류

에러 메시지를 보면 원인이 갈립니다.

| 메시지 | 원인 |
|---|---|
| `no healthy upstream` | 엔드포인트가 아예 없음 : 파드가 없거나 selector 불일치 |
| `upstream connect error ... remote connection failure` | 엔드포인트는 있는데 연결 거부 : **포트 불일치** |

두 번째는 `ealen/echo-server`에 `PORT=8080` 환경변수를 주고 `targetPort`를 80으로
두거나, 그 반대로 했을 때 납니다. 둘을 반드시 같이 맞춰야 합니다.

### 503도 매칭 성공의 증거

경로 매칭 검증 중에 503이 나와도 판정에는 쓸 수 있습니다.

- **404** : 매칭되는 rule 없음. Envoy가 직접 응답
- **503** : rule은 매칭됐고 업스트림 연결에서 실패

즉 매칭 여부만 볼 때는 404와 그 외를 구분하면 됩니다. 백엔드를 고치기 전에도
매칭 검증은 진행할 수 있습니다.

### 백엔드 식별 필드를 잘못 골랐음

`ealen/echo-server`에서 파드 이름은 `.environment.HOSTNAME`입니다.
`.host.hostname`은 **요청의 Host 헤더**가 들어옵니다. 이걸로 파드를 구분하려다
모든 요청이 `routing.test`로 나와서 한참 헤맸습니다.

```bash
# ✅ 파드 이름
jq -r '.environment.HOSTNAME'

# ❌ 요청 Host 헤더
jq -r '.host.hostname'
```

### `image` 오타는 `InvalidImageName`

`ealen:echo-server:latest`처럼 슬래시를 콜론으로 쓰면 kubelet이 이미지 참조를 파싱하지
못합니다. `ImagePullBackOff`가 아니라 `InvalidImageName`으로 뜹니다.

```bash
kubectl get pods -n routing-lab \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.containers[0].image}{"\n"}{end}'
```

### `parentRefs.namespace` 누락 시 status가 통째로 빔

HTTPRoute가 Gateway와 다른 네임스페이스에 있는데 `parentRefs.namespace`를 빠뜨리면,
자신의 네임스페이스에서 Gateway를 찾다가 실패합니다. **에러 메시지가 없고 `status`가
비어 있습니다.** 이게 가장 추적하기 어려운 실수였습니다.

`status`가 비었을 때 볼 것 두 가지입니다.

1. `parentRefs.namespace`가 명시돼 있는가
2. Gateway 리스너의 `allowedRoutes.namespaces.from`이 이 네임스페이스를 허용하는가

### YAML 인용부호

정규식에는 백슬래시가 들어가므로 인용부호 선택이 중요합니다.

| 표기 | 결과 |
|---|---|
| `'^/foo/(\d+)'` | ✅ 그대로 전달 |
| `"^/foo/(\d+)"` | ❌ 이스케이프로 해석 시도 |
| `"^/foo/(\\d+)"` | ⚠️ 되지만 백슬래시 두 번 |

**작은따옴표를 씁니다.**

---

## 정리

Gateway API의 Extended 등급 기능에 의존하기로 결정하기 전에, 그 기능이 실제로 어떻게
동작하는지 재보는 데 반나절이면 충분했습니다. 문서로는 답이 없는 질문이 네 개
있었고, 네 개 다 답이 나왔습니다.

특히 **스펙이 미규정으로 남긴 영역**에서는 실측이 유일한 근거입니다. 그리고 그 근거는
버전에 묶여 있으니, 측정값과 함께 **측정한 버전을 반드시 기록**해두어야 합니다.

실습 리소스는 네임스페이스 하나에 몰아뒀으니 정리는 한 줄입니다.

```bash
kubectl delete ns routing-lab
```

---

## 참고

- [Gateway API : HTTPRoute path matching precedence](https://gateway-api.sigs.k8s.io/api-types/httproute/)
- [Envoy Gateway : HTTPRouteFilter](https://gateway.envoyproxy.io/docs/api/extension_types/)
- [Envoy : RouteMatch.safe_regex](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto)
- [RE2 syntax](https://github.com/google/re2/wiki/Syntax)

