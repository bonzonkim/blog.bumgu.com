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
I deployed Loki on Kubernetes and connected it to Grafana as a Data Source, and it had been working well. However, on the Alerting rulers page, I saw an error message like `Failed to load the data source configuration for loki: Unable to fetch alert rules. Is the loki data source properly configured?`.

The Loki Helm Chart I use is version 6.22 from [here](https://artifacthub.io/packages/helm/grafana/loki).

---
# 1. Existing values
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

`/etc/loki/rules` is read-only because it is the directory where alert rules are stored persistently. Rules created by clients such as Grafana are temporarily stored in the directory specified by `rule_path`, so that directory needs to be writable. Therefore, we need to mount `rule_path` as a volume.

# 2. Mount a volume at the rule path
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
Mount a volume for `rulerConfig.rule_path` under `ruler.extraVolumeMounts`.

```sh
➜ k -n test exec -it curl-test-5bd8ffb644-vclbr -- sh
~ $ curl http://<Loki ruler endpoint>/loki/api/v1/rules
no alert groups found
```
After this, sending a request with `curl` from inside a Pod returns that no alert groups are defined.
(It doesn't matter which Pod sends the curl request, as long as it can communicate with the Loki ruler component's endpoint.)

Looking at `template/ruler/statefulset-ruler.yaml` in the Loki Helm chart:

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
`{{- range $dir, $_ := .Values.ruler.directories }}`

In other words, entries in `ruler.directories` in `values.yaml` are mounted at `/etc/loki/rules/$dir`. That's why `/etc/loki/rules` was set in the `rulerConfig` above. Now let's add an example alert here.

# 3. Modify values
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
Now, if we send a curl request from inside the Pod again:
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
We can confirm it is returned correctly.

[Reference 1](https://honglab.tistory.com/256)
[Reference 2](https://github.com/canonical/loki-k8s-operator/issues/396)

---
# Conclusion

While using Helm Charts, I kept putting off analyzing template files because the GoTemplate syntax was confusing. But this time, whenever I got stuck, I developed the habit of checking the template first. If you get stuck, I think you can solve the issue more easily by reading the template and following how the values are deployed.
