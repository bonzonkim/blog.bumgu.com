---
title: "Routing with Nginx and Envoy Gateway"
author: Bumgu
draft: false
date: 2026-08-02T16:07:35+09:00
categories:
    - DevOps
    - SRE
tags:
    - Envoy
    - Kubernetes
slug: "envoy-gateway-proxy"
---

# Can Nginx Regex Routing Be Moved to Envoy Gateway?

There was one question we had to answer first in order to migrate the edge gateway from OpenResty (nginx) to Envoy Gateway: **Can the regex `location` settings in nginx conf be moved to the Gateway API as-is?**

Reading the documentation did not provide an answer. The Gateway API specification classifies regex path matching as Extended and **does not specify its priority at all**. It only says that it is left to the discretion of the implementation.

So I set up Envoy Gateway on kind and measured it directly. This post is a record of that process.
It is written so that you can reproduce it exactly, including all the manifests.

The measurement environment was as follows.

| Item | Version |
|---|---|
| Envoy Gateway | v1.8.3 |
| Gateway API | v1 (`gateway.networking.k8s.io/v1`) |
| kind | on colima |
| MetalLB | v0.14.8 |

**Since this measures an area left unspecified by the spec, these results apply only to EG v1.8.3.** You should measure again when upgrading versions.

---

## What Did We Measure?

Four things:

1. **Regex matching scope**: nginx uses partial matching. Does Envoy do the same?
2. **Priority**: If a regex and PathPrefix overlap, which one wins? Does it depend on the order?
3. **Group capture rewrite**: Can a value captured with `(\d+)` be used in a path rewrite?
4. **Method matching**: Can the same path be split between GET and POST?

---

## Environment Setup

### kind Cluster

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

kind has no entity to assign IPs to LoadBalancer services. We add MetalLB to prevent the service created by Envoy Gateway from getting stuck in the `<pending>` state.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
kubectl wait -n metallb-system --for=condition=ready pod --selector=app=metallb --timeout=90s

# First check the kind Docker network subnet
docker network inspect -f '{{.IPAM.Config}}' kind
```

Set the pool range within the subnet you found.

```yaml
# metallb-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: kind-pool
  namespace: metallb-system
spec:
  addresses:
    - 172.18.255.200-172.18.255.250   # Adjust to match the docker network inspect output
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

### GatewayClass and Gateway

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
          from: All        # Allow HTTPRoutes from other namespaces to attach as well
```

```bash
kubectl apply -f gatewayclass.yaml -f gateway.yaml
kubectl get gateway -n default eg
```

The `PROGRAMMED` column should become `True`. If it is `False`, all subsequent curl requests will fail to connect, so resolve that first.

`allowedRoutes.namespaces.from` is set to `All` for convenience in this exercise.
In production, it is better to narrow this down with a `Selector`.

---

## Test Backend

The most important thing in validating routing is that **you must be able to tell from the response alone which backend received the request and what path the backend received**.

I initially used `nginx:alpine`, but both backends returned the same default page, so there was no way to distinguish them. I switched to `ealen/echo-server`. It returns the request path and headers as JSON, and the pod name is included in an environment variable, allowing the backend to be identified as well.

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

`ealen/echo-server` listens on **port 80** by default. You can change it with the `PORT` environment variable, but then you must change `targetPort` as well. If you change only one of them, you get a 503 (see the troubleshooting notes below).

### Helper for Checking Responses

We use this function throughout the measurements. It shows the status code and pod name on one line, and it does not break on non-JSON responses (404 or 503 bodies).

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

# To inspect only the path (for rewrite verification)
rewritten() {
  curl -s -H "$H" "$U$1" | jq -r '.http.originalUrl'
}
```

The pod name is `.environment.HOSTNAME`. `.host.hostname` contains the request's Host header, so it cannot be used to identify the backend. I went down a dead end here once.

### Connecting

```bash
kubectl get svc -n envoy-gateway-system
kubectl port-forward -n envoy-gateway-system svc/envoy-default-eg-<hash> 8080:80
```

**You must use `svc/`.** Envoy proxy pods run as non-root and listen on a high-numbered port inside the container, not port 80. Port-forwarding directly to a pod results in connection refused.

Do not use `.local` for the hostname. macOS mDNS causes a 5-second delay on every request.
Use `.test` instead.

---
## Verification 1: Regular Expression Matching Scope

nginx's `location ~` performs **partial matching**. If you write `^/foo`, it also
matches `/foo/bar/baz`. Let's see whether Envoy behaves the same way.

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
      namespace: default        # Required when the Gateway is in a different namespace
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

If you omit `parentRefs.namespace`, the HTTPRoute looks for the Gateway in its own namespace.
There is no Gateway in `routing-lab`, so it cannot find one, but **the entire `status` ends up
empty without an error message.** This is the kind of mistake that is quite painful to track down.

Wrap regular expressions in **single quotes**. Inside YAML double quotes, `\d` is treated as an
escape, so you have to write `\\d`.

```bash
kubectl apply -f route-regex.yaml

# Verify application: syntax rejected by RE2 will show Accepted=False here
kubectl get httproute -n routing-lab regex-only \
  -o jsonpath='{.status.parents[*].conditions[*]}' | jq .
```

### Measurements

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

### Results

| Request | Result | nginx `location ~` |
|---|---|---|
| `.../123/coupons` | 200 | Same |
| `.../123/coupons/456` | **404** | **Different**: nginx matches |
| `.../123/coupons?x=1` | 200 | Same (query string is excluded from path matching) |
| `/api/v3/...` | 404 | Same (`^` anchor) |
| `.../abc/coupons` | 404 | Same (`\d+`) |

**Envoy matches the regular expression against the entire path.** This is the behavior of the
`safe_regex` path matcher, and its engine is RE2.

If you move an nginx conf over unchanged, all subpaths will fail with 404. You need a conversion rule.

```
nginx  location ~ ^/foo        →  Gateway  ^/foo.*
nginx  location ~ ^/foo$       →  Gateway  ^/foo    (or Exact)
nginx  location ~ ^/foo/[^/]+  →  Gateway  ^/foo/[^/]+.*
```

Appending `.*` also matches `/foobar`, but **nginx matched it that way originally, too.** In other
words, adding `.*` preserves behavior, while omitting it changes behavior. This can be applied
mechanically, with only patterns ending in `$` as exceptions.

### RE2 constraints

Envoy uses RE2. It differs from PCRE in the following ways.

| Syntax | Supported |
|---|---|
| `\d`, `[^/]+`, `{n,m}`, `(?i)` | ✅ |
| lookahead `(?=)`, lookbehind `(?<=)` | ❌ |
| backreference in pattern `\1` | ❌ |

If you use unsupported syntax, the HTTPRoute is rejected with `Accepted: False`. That's better than
failing silently, but it's best to find such syntax in the conf beforehand, before migrating.

Translate nginx's case-insensitive match (`~*`) using the `(?i)` prefix.

---

## Verification 2: Priority

This is the main part of these measurements. Appending `.*` broadens a regular expression and
increases the range in which it overlaps with other rules, but **regular expression priority is
unspecified by the spec.**

The spec defines the following order, and no further:

1. `Exact` path match
2. The `PathPrefix` with the greatest number of characters
3. A match that includes a method
4. A match with the greatest number of header matches
5. A match with the greatest number of query parameter matches

`RegularExpression` is not on this list. So I measured four combinations.

### A: RegularExpression vs. PathPrefix

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
    # (1) Broad prefix → echo-a
    - matches:
        - path:
            type: PathPrefix
            value: /v3/customer
      backendRefs:
        - name: echo-a
          port: 80
    # (2) Narrow regular expression → echo-b
    - matches:
        - path:
            type: RegularExpression
            value: '^/v3/customer/bookings/\d+/coupons.*'
      backendRefs:
        - name: echo-b
          port: 80
```

Before measuring, delete the route from Verification 1. When multiple HTTPRoutes are attached to
the same host, the inter-resource tiebreak (creation time → name order) gets mixed in, making the
result less clear.

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

**The regular expression won.** `/profile` did not match the regular expression, so it fell through
to the prefix, meaning the prefix rule also works as expected.

nginx also gives priority in the order `=` exact → `^~` prefix → **regular expression** → longest
ordinary prefix, so a regular expression beats an ordinary prefix. This behavior is preserved.

### B: Reverse the order

Swap only the positions of rules (1) and (2), reapply, and send the same requests.

```
GET      /v3/customer/bookings/123/coupons        200 echo-b-548f5495f-c7wv8
GET      /v3/customer/bookings/123/coupons/456    200 echo-b-548f5495f-c7wv8
GET      /v3/customer/profile                     200 echo-a-868fdcc6bd-xl679
```

**The result is identical.** Rules are sorted by their characteristics, regardless of their order in YAML.

This differs from nginx. nginx evaluates regular expression locations **in the order they are
written in the file, with the first match winning.** EG, on the other hand, does not consider order.

From an operational perspective, EG is better. Routing remains deterministic even if you split
manifests across multiple files, and you don't have to worry about things like file merge order.

### C: When the prefix is longer

Let's see whether the spec's "longest PathPrefix wins" rule also applies when comparing with a
regular expression. I made the prefix more specific than the regular expression.

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
    # Long prefix (25 characters) → echo-a
    - matches:
        - path:
            type: PathPrefix
            value: /v3/customer/bookings/123
      backendRefs:
        - name: echo-a
          port: 80
    # Regular expression → echo-b
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

**The regular expression wins regardless of prefix length.** This is the simplest possible result.
There's no need to exhaustively compare cases where prefixes and regular expressions overlap.

### D: Overlapping regular expressions

This is the combination with the highest risk in practice. If the conf contains 61 regular
expressions, some of them will overlap.

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
    # (1) Broad regular expression → echo-a
    - matches:
        - path:
            type: RegularExpression
            value: '^/v3/customer/bookings/.*'
      backendRefs:
        - name: echo-a
          port: 80
    # (2) Narrow regular expression → echo-b
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

Even though the broad regular expression is higher in the YAML, **the narrower one won.** The
result was the same when the order was reversed.

### What remains unverified

**I could not determine the criterion that makes the narrower expression win.** It is unclear
whether it is "specificity" or simply **pattern string length**. In measurement D, the narrower
regular expression also had a longer string.

If length is the criterion, a `long but broad` regular expression could beat a `short but narrow`
one and send requests to the wrong backend. If such combinations exist in your actual conf, each
case needs to be checked individually.

This kind of remaining uncertainty is the price you pay for relying on areas the spec does not
specify. It is why, during migration, you need to create a **complete path → expected backend
comparison table** as a regression test.

---
## Verification 3: Capture-group rewrite

We check whether a value captured by `(\d+)` can be used in a path rewrite.

The standard Gateway API's `URLRewrite` filter only has `ReplacePrefixMatch` and
`ReplaceFullPath`. It has no regex substitution. Envoy Gateway provides this
functionality through a separate CRD called `HTTPRouteFilter`.

The CRD is separate to avoid breaking the standard spec. It references an external
resource through the extension point called `ExtensionRef`, which the standard
provides. As a side effect, the same substitution rule can be reused across multiple
routes.

### What we wanted to know

The question was what happens to the **path remaining after** the regex match.

```
Request  /nt/bookings/123/coupons/456
pattern  ^/nt/bookings/(\d+)/coupons
         └── matched portion ──────┘└─┬─┘
                                       tail

Does the result become /coupons/123 (tail discarded) or /coupons/123/456 (tail preserved)?
```

If the tail is discarded, we need to add `(.*)` to all 61 patterns and explicitly
append it again. If it is preserved, no changes are needed.

We compared two patterns side by side. `nt` = tail not captured, `wt` = tail captured.

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
    # Tail not captured
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
    # Tail captured
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

`extensionRef` has no `namespace` field. It can only reference resources in the
**same namespace as the HTTPRoute**.

### Why does the regex appear twice?

The same regex appears in both `matches` and the filter, which looks redundant, but
they serve different purposes.

| | `matches` | Filter `pattern` |
|---|---|---|
| Purpose | Whether to apply this rule: true or false | How to change the path |
| Matching behavior | **Full-path match** | **Partial match followed by substitution** |
| Capture groups | Have no effect even if used | Only meaningful here |

If `matches` does not pass, the filter does not run. The check comes first, then the
substitution. So we add `.*` to `matches` to allow subpaths into the rule, and perform
the actual transformation in the filter.

### Measurement

```bash
kubectl apply -f route-rewrite.yaml

for p in \
  '/nt/bookings/123/coupons' \
  '/nt/bookings/123/coupons/456' \
  '/wt/bookings/123/coupons/456' ; do
  printf '%-34s -> %s\n' "$p" "$(rewritten "$p")"
done
```

### Results

```
/nt/bookings/123/coupons           -> /coupons/123
/nt/bookings/123/coupons/456       -> /coupons/123/456
/wt/bookings/123/coupons/456       -> /coupons/123/456
```

**The tail is preserved.** Envoy's `regex_rewrite` is in the RE2 GlobalReplace
family, so it replaces only the matched part and keeps the rest.

The confusing point is that this is the exact opposite of path matching (full match).
In summary:

| Use | Transformation | Reason |
|---|---|---|
| `matches` regex | **Add `.*` at the end** | Full match. Without it, subpaths return 404 |
| rewrite `pattern` | **Leave it as is** | Partial substitution. The rest is preserved |

Since `/nt` and `/wt` produced the same result, `(.*)`+`\2` is unnecessary work.

### Caution: GlobalReplace performs multiple substitutions

It substitutes **every matching occurrence**. With the `^` anchor, there is only one
position, so it is safe; patterns without an anchor may be substituted multiple times.

```
pattern '/coupons'  →  both occurrences replaced in /a/coupons/b/coupons
```

Since nginx's `location ~` works without anchors, the conf may contain regexes with no
anchor. It is a good idea to check whether rewrite target patterns start with `^`.

### It cannot be used for redirects

`ReplaceRegexMatch` is **rewrite-only**. It changes the internal path, so the client
does not know it changed.

The `RequestRedirect` filter has no regex substitution option, so captured values
cannot be placed in the `Location` header. Istio is the same: `HTTPRedirect` in
`VirtualService` uses a fixed string for `uri`, while `uriRegexRewrite` is only
available for rewrites.

Envoy core's `RedirectAction` has `regex_rewrite`, but neither implementation exposes
it through its higher-level API. It can only be accessed through low-level patches like
`EnvoyPatchPolicy` (EG) or `EnvoyFilter` (Istio), which can silently break during
version upgrades.

If you need regex capture redirects, **setting up a small redirect service separately**
is the practical choice.

---

## Verification 4: Method matching

We check whether the same path can be routed by method. A common pattern in
authorization systems is to allow `GET` while requiring higher privileges for `POST`.

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
    # E-1: Regular expression + GET → echo-a
    - matches:
        - path:
            type: RegularExpression
            value: '^/v3/backoffice/wages.*'
          method: GET
      backendRefs:
        - name: echo-a
          port: 80
    # E-2: Same regular expression + POST → echo-b
    - matches:
        - path:
            type: RegularExpression
            value: '^/v3/backoffice/wages.*'
          method: POST
      backendRefs:
        - name: echo-b
          port: 80
    # F-1: No method condition → echo-a
    - matches:
        - path:
            type: PathPrefix
            value: /w2
      backendRefs:
        - name: echo-a
          port: 80
    # F-2: Same path + POST → echo-b
    - matches:
        - path:
            type: PathPrefix
            value: /w2
          method: POST
      backendRefs:
        - name: echo-b
          port: 80
```

`method` is a sibling field of `path` within the same match entry. It is important to
understand the logical relationships precisely here.

- **Items in the `matches` list are ORed**: only one of the conditions needs to match
- **`path`/`method`/`headers`/`queryParams` within one match entry are ANDed**: all must match

### Measurement

```bash
kubectl apply -f route-method.yaml

for m in GET POST PATCH DELETE OPTIONS; do probe $m /v3/backoffice/wages/123; done
for m in GET POST; do probe $m /w2/anything; done
```

### Results

```
GET      /v3/backoffice/wages/123      200 echo-a-868fdcc6bd-xl679
POST     /v3/backoffice/wages/123      200 echo-b-548f5495f-c7wv8
PATCH    /v3/backoffice/wages/123      404 -
DELETE   /v3/backoffice/wages/123      404 -
OPTIONS  /v3/backoffice/wages/123      404 -

GET      /w2/anything                  200 echo-a-868fdcc6bd-xl679
POST     /w2/anything                  200 echo-b-548f5495f-c7wv8
```

Method-based routing works as expected. The fact that a rule with a method condition
wins over one without it is also specified as priority number 3 in the spec.

### We found a pitfall here

**A method mismatch returns 404, not 405.** Since no rule matches, this is natural
from the routing perspective, but if the path used to return 405 through nginx's
`limit_except`, the response code changes.

More importantly, **OPTIONS also returns 404**.

Envoy's CORS filter is **route-level**. It runs after a route has matched. Therefore,
if OPTIONS fails to match a route and ends in a 404, the **preflight breaks** even if
CORS is configured with `SecurityPolicy`.

This is hard to notice with curl. It only fails in the browser, and because the
response code is 404, it is not caught by 5xx-based alerts either. It fails silently.

### Mitigation pattern

For paths with method-based routing, also add a **rule for the same path without a
method condition**. As confirmed in Measurement F, the rule with a method condition
wins, so the routing split remains intact, while OPTIONS falls through to the
unconditional rule and gets a route.

```yaml
rules:
  # Write operation requiring elevated privileges (POST)
  - matches:
      - path: { type: PathPrefix, value: /v3/backoffice/wages }
        method: POST
    backendRefs:
      - name: echo-b
        port: 80
  # Everything else: OPTIONS is handled here
  - matches:
      - path: { type: PathPrefix, value: /v3/backoffice/wages }
    backendRefs:
      - name: echo-a
        port: 80
```

It is also possible to explicitly add `OPTIONS` to the method list, but as the number
of paths with splits grows, it is easy to miss one. The pattern above is safer.

---
## Summary of results

Here are the behavioral differences from nginx in one table.

| Item | Envoy Gateway v1.8.3 | nginx | Response |
|---|---|---|---|
| Regex matching scope | **Full-path matching** | Partial matching | Add `.*` at the end |
| Query string | Excluded from path matching | Same | : |
| `^` anchor / `\d+` | Works as expected | Same | : |
| Case-insensitive matching | `(?i)` prefix | `~*` | Convert syntax |
| Regex engine | RE2 | PCRE | Pre-screen for lookahead and backreferences |
| Regex vs. prefix | **Regex wins** (regardless of length) | Regex wins | : |
| Order of rules | **Irrelevant** (sorted by characteristics) | First match in file order wins | No need to reproduce order |
| Regex vs. regex | Narrower one wins | Earlier one wins | Compare overlapping ranges |
| Regex replacement in rewrite | Partial replacement, preserves the tail | Same | Keep pattern as-is |
| Method not matched | **404** | 405 (`limit_except`) | Check client impact |
| OPTIONS | **404** when there is a method condition | Handled by Lua/conf | Place a rule without a condition alongside it |

### Conclusion

**Regex routing can be migrated.** However, this depends on three things:

1. Do not forget the mechanical conversion of **adding `.*` to matching regexes**
2. **Do not change rewrite patterns** : matching and rewrite rules are opposites
3. **Place a rule without a condition alongside paths with a method condition** (CORS)

The most welcome result was that **rule order does not matter**. In nginx, the file order of regex locations is their priority, so preserving the order is a major burden when migrating at scale. EG has no such burden, so files can be split up freely.

The biggest concern is that **we could not establish the criteria for deciding between regexes**. Since this area is unspecified by the spec, it cannot be guaranteed in the documentation and may change in a future version. There is no alternative to creating exhaustive comparison regression tests.

---

## Troubleshooting notes

I spent more time here than on the measurements.

### Connection refused when port-forwarding to a pod

Envoy proxy pods run as non-root, so they listen on a high port inside the container rather than on 80. You **must** port-forward to `svc/`.

```bash
# ✅
kubectl port-forward -n envoy-gateway-system svc/envoy-default-eg-<hash> 8080:80

# ❌ connection refused
kubectl port-forward -n envoy-gateway-system pod/envoy-default-eg-<hash>-xxx 8080:80
```

### `.local` hostnames take 5 seconds per request

This is due to macOS mDNS. Use `.test` instead. This does not matter when using only `-H 'Host: ...'` with curl, since there is no DNS lookup, but it does matter when registering the name in `/etc/hosts` and using it.

### Two kinds of 503

The cause differs depending on the error message.

| Message | Cause |
|---|---|
| `no healthy upstream` | No endpoints at all : no pods or a selector mismatch |
| `upstream connect error ... remote connection failure` | Endpoints exist, but the connection is refused : **port mismatch** |

The second occurs if you set the `PORT=8080` environment variable on `ealen/echo-server` but set `targetPort` to 80, or vice versa. The two must be made to match.

### A 503 is also evidence of a successful match

A 503 during path-matching verification can still be used to determine the result.

- **404** : No matching rule. Envoy responds directly
- **503** : The rule matched, but the upstream connection failed

In other words, when checking only whether a match occurred, distinguish 404 from anything else. You can verify matching even before fixing the backend.

### Chose the wrong backend identification field

In `ealen/echo-server`, the pod name is `.environment.HOSTNAME`.
`.host.hostname` contains the **request's Host header**. I spent a long time trying to distinguish pods with this, only to see `routing.test` for every request.

```bash
# ✅ Pod name
jq -r '.environment.HOSTNAME'

# ❌ Request Host header
jq -r '.host.hostname'
```

### An `image` typo results in `InvalidImageName`

If you use a colon instead of a slash, as in `ealen:echo-server:latest`, kubelet cannot parse the image reference. The status is `InvalidImageName`, not `ImagePullBackOff`.

```bash
kubectl get pods -n routing-lab \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.containers[0].image}{"\n"}{end}'
```

### Omitting `parentRefs.namespace` leaves status completely blank

If an HTTPRoute is in a different namespace from the Gateway and you omit `parentRefs.namespace`, it fails while looking for the Gateway in its own namespace. **There is no error message, and `status` is empty.** This was the hardest mistake to track down.

When `status` is empty, check these two things:

1. Is `parentRefs.namespace` specified?
2. Does `allowedRoutes.namespaces.from` on the Gateway listener allow this namespace?

### YAML quotation marks

Since regexes contain backslashes, choosing the quotation marks matters.

| Notation | Result |
|---|---|
| `'^/foo/(\d+)'` | ✅ Passed through unchanged |
| `"^/foo/(\d+)"` | ❌ Attempts to interpret escapes |
| `"^/foo/(\\d+)"` | ⚠️ Works, but requires two backslashes |

**Use single quotes.**

---

## Wrap-up

It took half a day to measure how these features actually behave before deciding to rely on Gateway API features at the Extended level. There were four questions the documentation could not answer, and all four got answers.

Actual measurements are the only evidence, especially in **areas the spec leaves unspecified**. And since that evidence is tied to a version, be sure to **record the version tested** along with the measurements.

All the practice resources are grouped in one namespace, so cleanup is a single line.

```bash
kubectl delete ns routing-lab
```

---

## References

- [Gateway API : HTTPRoute path matching precedence](https://gateway-api.sigs.k8s.io/api-types/httproute/)
- [Envoy Gateway : HTTPRouteFilter](https://gateway.envoyproxy.io/docs/api/extension_types/)
- [Envoy : RouteMatch.safe_regex](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto)
- [RE2 syntax](https://github.com/google/re2/wiki/Syntax)


