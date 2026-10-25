---
title: "kube-prometheus-stack: Managing Scrape Targets with an External Secret"
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
As you add targets directly to `additionalScrapeConfigs`, the `values.yaml` file can grow to 1,000 or 2,000 lines and become difficult to manage.

So I changed the setup to keep Scrape Targets in an external Secret and manage them by referencing that Secret.

# 1. Create the existing ScrapeTarget in a file

Write the configuration that was previously in `prometheus.prometheusSpec.additionalScrapeConfigs` in `values.yaml` to a separate file.
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
# 2. Generate a Secret manifest from the file
`kubectl create secret generic --from-file=prometheus-scrape-target.yaml --dry-run=client -o yaml > additional-scrape-target.yaml`
The command above generates a Secret manifest YAML file that can create a Kubernetes Secret based on the file we just wrote.

Create the Secret using the generated `additional-scrape-target.yaml` file.
`kubectl apply -f additional-scrape-target.yaml`

# 3. Apply it to the existing values
In the existing `values.yaml`,
remove `prometheus.prometheusSpec.additionalScrapeConfigs` and
configure `prometheus.prometheusSpec.additionalScrapeConfigsSecret` to reference the Secret created in step 2.
```yaml
additionalScrapeConfigsSecret:
  enabled: true
  name: additional-scrape-configs
  key: prometheus-scrape-target.yaml
```
Create a Secret manifest based on prometheus-scrape-target.yaml => create a Secret from the generated manifest => reference that Secret from `values.yaml`.

---
Personally, as the `kube-prometheus-stack` values file became longer and harder to manage over time in operation, I found this approach.
Note that **`additionalScrapeConfigs` and `additionalScrapeConfigsSecret` cannot be used together**.

[Post I referred to](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/additional-scrape-config.md)
