---
title: "Loki-VictoriaLogs Proxy 사용해보기"
author: Bumgu
draft: false
date: 2026-04-19T18:50:12+09:00
categories: 
    - DevOps
    - SRE
tags: 
    - VictoriaLogs
    - Loki
    - Grafana
slug: "loki-vl-proxy"
---

개인 쿠버네티스 클러스터에서는 로그 저장소로 [VictoriaLogs](https://docs.victoriametrics.com/victorialogs/)를 사용하고 있습니다. Loki에 비해 리소스 사용량이 적고 설치/운영이 단순해 소규모 환경에 적합하기 때문입니다.  

다만 Grafana에서 VictoriaLogs 데이터소스를 공식 지원하긴 하나, 자동완성이나 Explore UX가 아직 Loki만큼 매끄럽진 않아서 결국 [vmui](https://docs.victoriametrics.com/victorialogs/querying/#web-ui)로 돌아가곤 했습니다.  
그러던 중 [loki-vl-proxy](https://github.com/ReliablyObserve/Loki-VL-proxy)라는 프로젝트를 발견했습니다. **VictoriaLogs를 백엔드로 두고 Loki API와 호환되는 엔드포인트를 제공하는 Proxy** 여서, Grafana 입장에선 그냥 Loki를 쓰는 것처럼 동작합니다. 저장은 VL로 하고, 조회는 Loki UX로 하겠다는 발상이라 바로 배포해보기로 했습니다.

## 아키텍처

loki-vl-proxy는 이름 그대로 **읽기 전용 프록시**입니다. `push`(수집) 경로는 막혀있고, Loki의 query API만 VictoriaLogs로 중계합니다. 즉 로그 수집은 기존처럼 `vlagent`/OTLP/Loki push → VL 로 그대로 두고, **조회 계층에만 Loki 호환 레이어**를 끼워 넣는 구조입니다.

```
┌─────────────────┐    LogQL     ┌──────────────────┐    LogsQL    ┌──────────────┐
│ Grafana /       │  /loki/api/* │  loki-vl-proxy   │              │ VictoriaLogs │
│ Drilldown / CLI │ ───────────▶ │  (translator +   │ ───────────▶ │   :9428      │
│                 │              │   cache + guard) │              │              │
└─────────────────┘              └──────────────────┘              └──────────────┘
```

프록시 내부는 대략 이렇게 구성되어 있습니다.

1. **API Layer** — `/loki/api/v1/query`, `query_range`, `labels`, `series`, `tail`, `patterns`, `rules`, `alerts` 등 Loki HTTP/WS API를 그대로 노출합니다. Grafana는 "Loki 데이터소스"로 붙기만 하면 되므로 별도의 플러그인이 필요 없습니다.
2. **Guardrails** — 테넌트 격리(`X-Scope-OrgID`), rate limit, circuit breaker, fanout 제한 등 운영 가드레일.
3. **Translator** — 핵심 부분. **LogQL → LogsQL** 변환기입니다. stream selector, 파이프라인 파서, `rate()`/`count_over_time()` 같은 range 함수, 라벨 필터까지 VL 쿼리 문법으로 바꿔줍니다. Loki에는 있지만 VL 네이티브엔 없는 의미(semantic)는 프록시단에서 보정합니다.
4. **Cache 계층** — 4단 구조로 되어 있습니다.
   - `Tier0` compatibility-edge: 호환성 변환 결과를 캐시 (반복 변환 비용 제거)
   - `L1` memory: 핫패스 in-memory 캐시
   - `L2` disk: 과거 window 재사용 (선택)
   - `L3` peer: fleet 내 동료 파드에서 공유 (선택)
   - query_range는 긴 구간을 작은 윈도우로 쪼개 캐싱하고, near-now만 짧은 TTL로 유지하는 식으로 장기 구간 조회 비용을 낮춥니다.
5. **Rules/Alerts Bridge** — `vmalert`의 rules/alerts 상태를 Loki ruler API 스키마로 변환해 Grafana Alerting 탭과 호환시킵니다.

결과적으로 Grafana는 "Loki에 붙어있다"고 믿고, VictoriaLogs는 "평소처럼 LogsQL로 질의가 들어온다"고 믿습니다. 그 사이의 번역/캐시/가드를 proxy가 전담하는 셈입니다.

## ArgoCD Application

Helm 차트가 OCI 레지스트리로 배포되어 있어서 ArgoCD에서 바로 붙일 수 있습니다. values는 별도 gitops 레포에서 관리하도록 multi-source로 구성했습니다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: loki-vl-proxy
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: ghcr.io/reliablyobserve/charts
      chart: loki-vl-proxy
      targetRevision: 1.8.1
      helm:
        releaseName: loki-vl-proxy
        valueFiles:
          - $values/envs/prod/loki-vl-proxy/values.yaml
    - repoURL: git@github.com:yourorg/gitops.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: observability
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true, ServerSideApply=true]
```

## values
차트의 default values는 [github](https://github.com/ReliablyObserve/Loki-VL-proxy/blob/main/charts/loki-vl-proxy/values.yaml)에서 확인할 수 있습니다.  
최소한 바꿔줘야 하는 것은 `.extraArgs.backend` 입니다. 여기에 같은 클러스터 내 VictoriaLogs 서비스 주소를 넣어주면 됩니다.

```yaml
extraArgs:
  listen: ":3100"
  backend: "http://victoria-logs-victoria-logs-single-server:9428"
```

`listen`을 `:3100`으로 둔 이유는 Loki 기본 포트와 맞춰 Grafana 데이터소스 설정을 그대로 재사용하기 위함입니다. Drilldown을 쓸 계획이면 `extraArgs.patterns-enabled=true`(기본값)도 그대로 두면 됩니다.


## 배포완료
![image](/images/post/31-loki-vl-proxy/1.png)

Pod이 올라왔으니 이제 Grafana 쪽에서 붙여보겠습니다.  
Grafana datasource에서 **Loki** 타입을 선택하고, URL만 loki-vl-proxy의 서비스 주소로 지정하면 됩니다. 플러그인 설치나 별도 설정은 필요 없습니다.

![image](/images/post/31-loki-vl-proxy/2.png)


## 결과
![image](/images/post/31-loki-vl-proxy/3.png)

로그 저장소는 VictoriaLogs지만, 질의는 Loki 문법(LogQL)으로 작성합니다. loki-vl-proxy가 LogQL을 LogsQL로 변환해주기 때문에 Grafana 입장에서는 평범한 Loki 데이터소스로 보입니다.  
덕분에 Explore의 자동완성, Drilldown, 기존 LogQL 기반 대시보드/알럿 자산을 그대로 쓰면서도 저장소는 가벼운 VictoriaLogs를 유지할 수 있게 되었습니다.  

Loki의 UX에 익숙하지만 Loki 운영이 부담스러운 환경, 혹은 이미 VictoriaMetrics 생태계로 묶여있어 VL을 쓰고 싶은 환경에서 꽤 좋은 선택지인 것 같습니다.
