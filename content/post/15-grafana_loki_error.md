---
title: "Grafana, Loki ERROR - Failed to load the data source configuration for loki: Unable to fetch alert rules. Is the loki data source properly configured?"
author: Bumgu
date: 2024-12-08
categories: 
    - DevOps
    - SRE
tags: 
    - Grafana
    - Loki
    - Helm Chart
    - Kubernetes
slug: "grafana_loki_error"
---
Loki를 Kubernetes에 배포하고 Grafana에 Data Source로 연결해서 잘 사용하고 있었으나, Alerting rulers 페이지에서 `Failed to load the data source configuration for loki: Unable to fetch alert rules. Is the loki data source properly configured?` 와 같은 에러 메세지를 볼 수 있었습니다.

사용하고 있는 Loki Helm Chart는 [여기](https://artifacthub.io/packages/helm/grafana/loki) 의 6.22 버전을 사용하고 있습니다.

---
# 1. 기존 values
```yaml
rulerConfig:
    storage:
      type: local
      local:
        directory: /etc/loki/rules
    rule_path: /rules
    enable_api: true
    alertmanager_url: "<alertmanager_url>"
```

이는 `/etc/loki/rules`는 Alerts 룰을 영구적으로 저장하는 디렉토리이기 때문에 읽기전용이고 클라이언트(Grafana 등)에서 생성한 룰은 `rule_path`에 지정한 디렉토리에 임시로 저장되기 때문에 쓰기가 가능합니다. 그렇기 때문에 `rule_path`를 마운트 해줘야 합니다.


# 2. rule path 볼륨 마운트
```yaml
ruler:
  replicas: 2
  maxUnavailable: 1
  persistence:
    enabled: true
    size: 10Gi
    storageClassName: ""
  ring:
    kvstore:
      store: memberlist
  extraVolumes:
    - name: loki-rules-generated
      emptyDir: {}
  extraVolumeMounts:
    - name: loki-rules-generated
      mountPath: /rules
```
`ruler.extraVolumeMounts`에 `rulerConfig.rule_path` 볼륨을 마운트 합니다.

```sh
➜ k -n test exec -it curl-test-5bd8ffb644-vclbr -- sh
~ $ curl http://<Loki ruler endpoint>/loki/api/v1/rules
no alert groups found
```
이후 pod내부에서 `curl`을 통해 요청을 보내면 정의된 알림 그룹이 없다고 나옵니다.
(curl 요청을 보내는건 Loki ruler 컴포넌트의 endpoint와 통신할 수 있는 파드면 상관없습니다.)

Loki Helm chart의 `template/ruler/statefulset-ruler.yaml`을 보면

```yaml
volumeMounts:
	- name: config
	  mountPath: /etc/loki/config
	- name: runtime-config
	  mountPath: /etc/loki/runtime-config
	- name: data
	  mountPath: /var/loki
	- name: tmp
	  mountPath: /tmp/loki
	{{- if .Values.enterprise.enabled }}
	- name: license
	  mountPath: /etc/loki/license
	{{- end }}
	{{- range $dir, $_ := .Values.ruler.directories }}
	- name: {{ include "loki.rulerRulesDirName" $dir }}
	  mountPath: /etc/loki/rules/{{ $dir }}
```
`	{{- range $dir, $_ := .Values.ruler.directories }}`

즉 values.yaml에서 `ruler.directories`에 적는건 `/etc/loki/rules/$dir`로 마운트 됩니다. 그렇기에 위의 `rulerConfig`에서 `/etc/loki/rules`에 적용한것입니다.이제 여기에 예제 알림을 작성합니다.


# 3. Values 수정
```yaml
ruler:
  replicas: 2
  maxUnavailable: 1
  persistence:
    enabled: true
    size: 10Gi
    storageClassName: "nfs"
  ring:
    kvstore:
      store: memberlist
  extraVolumes:
    - name: loki-rules-generated
      emptyDir: {}
  extraVolumeMounts:
    - name: loki-rules-generated
      mountPath: /rules
  directories:
    fake: 
      rules1.txt: |
        groups:
          - name: test_alert
            rules:
              - alert: AlertManagerHealthCheck
                expr: 1 + 1
                for: 1m
                labels:
                    severity: critical
                annotations:
                    summary: Not an alert! Just for checking AlertManager Pipeline
              - alert: TestAlert
                expr: sum(rate({app="loki"} | logfmt | level="info"[1m])) by (container) > 0
                for: 1m
                labels:
                    severity: warning
                annotations:
                    summary: Loki info warning per minute rate > 0
                    message: 'Loki warning per minute rate > 0 container:"{{`{{`}} $labels.container {{`}}`}}"'
```
이후에 다시 Pod내부에서 curl요청을 날려보면
```sh
~ $ curl http://loki-ruler.lgtm.svc.cluster.local:3100/loki/api/v1/rules
rules1.txt:
    - name: test_alert
      rules:
        - alert: AlertManagerHealthCheck
          expr: 1 + 1
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: Not an alert! Just for checking AlertManager Pipeline
        - alert: TestAlert
          expr: sum(rate({app="loki"} | logfmt | level="info"[1m])) by (container) > 0
          for: 1m
          labels:
            severity: warning
          annotations:
            message: Loki warning per minute rate > 0 container:"{{`{{`}} $labels.container {{`}}`}}"
            summary: Loki info warning per minute rate > 0
```
정상적으로 나오는 것을 확인할 수 있습니다.

[참고1](https://honglab.tistory.com/256)
[참고2](https://github.com/canonical/loki-k8s-operator/issues/396)

---
# 마무리


그동안 HelmChart를 사용하면서 어지러운 GoTemplate 문법에 Template파일 분석을 미뤘습니다..만 이번기회에 막히는 부분이 있으면 Template을 먼저 확인해보는 습관이 생겼습니다.. 막힌다면 Template을 읽고 values가 어떻게 배포되는지를 따라가보면 보다 쉽게 해결 할 수 있을거라 생각합니다.
