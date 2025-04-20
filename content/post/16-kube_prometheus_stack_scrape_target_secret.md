---
title: "kube-prometheus-stack: scrape target을 외부 secret으로 관리"
author: Bumgu
date: 2024-12-22
categories: 
    - DevOps
    - SRE
tags: 
    - Kubernetes
    - Prometheus
slug: "kube_prometheus_stack_scrape_target_with_secret"
---
`additionalScrapeConfigs` 에 직접 target을 작성하다보면 1000줄, 2000줄이 넘어가 `values.yaml`파일이 길어져 관리가 힘들어집니다..

그렇기 때문에 Scrape Targets을 외부 Secret으로 빼고 Secret을 참조하는방식으로 관리하도록 변경했습니다.

# 1. 기존 ScrapeTarget을 파일로 생성


기존에 `values.yaml`의 `prometheus.prometheusSpec.additionalScrapeConfigs`에 적혀있던 설정을 파일로 따로 작성합니다.
* prometheus-scrape-target.yaml
```yaml
- job_name: server1
  static_configs:
    - targets: ["1.1.1.1:9100"]
      labels:
        alias: "server1"
        job: "node-exporter"
        component: "linux-server"
- job_name: server2
  static_configs:
    - targets: ["1.1.1.2:9100"]
      labels:
        alias: "server2"
        job: "node-exporter"
        component: "linux-server"
```
# 2. 파일 기준 Secret manifest 생성
`kubectl create secret generic --from-file=prometheus-scrape-target.yaml --dry-run=client -o yaml > additional-scrape-target.yaml`
위 명령어는 방금 작성한 파일을 기준으로 쿠버네티스 시크릿을 생성할 수 있는 secret manifest yaml파일을 생성합니다.

생성된 `additional-scrape-target.yaml` 파일을 기준으로 시크릿을 생성합니다.
`kubectl apply -f additional-scrape-target.yaml`


# 3. 기존 values에 적용
기존  `values.yaml`에 
`prometheus.prometheusSpec.additionalScrapeConfigs` 를 지우고
`prometheus.prometheusSpec.additionalScrapeConfigsSecret`에 2에서 생성한 시크릿을 참조하도록 작성합니다.
```yaml
additionalScrapeConfigsSecret:
  enabled: true
  name: additional-scrape-configs
  key: prometheus-scrape-target.yaml
```

prometheus-scrape-target.yaml을 기준으로 시크릿파일 생성 => 생성된 시크릿파일로 시크릿 생성 => `values.yaml`에서 그 시크릿을 참조

---
개인적으로 kube-prometheus-stack가 운영을 할 수록 values의 길이가 길어져 관리가 힘든차에 이러한 방법을 발견했습니다.
참고로 **additionalScrapeConfigs 와 additionalScrapeConfigsSecret은 같이 쓰일 수 없습니다**  

[참고한 글](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/additional-scrape-config.md)
