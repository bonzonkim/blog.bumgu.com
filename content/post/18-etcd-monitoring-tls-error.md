---
title: "Etcd 모니터링 tls 에러 해결"
author: Bumgu
date: 2025-01-18
categories: 
    - DevOps
    - SRE
tags: 
    - Grafana
    - TLS
    - etcd
    - Kubernetes
slug: "etcd_monitoring_tls_error"
---
Server OS : Ubuntu 22.04
Kubernetes 구축환경 : Kubespray  
Prometheus : [kube-prometheus-stack](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack/65.1.1) v65.1.1


# 1. 개요
Grafana etcd 대쉬보드가 Pod로 띄워지지 않고 쿠버네티스 서버에 직접 설치되어서 모니터링이 안되고 있던 상황
타겟에 `<etcd-server>:2379/metrics`를 추가했으나 tls문제로 etcd에 접근하지 못해 수집이 안되고있었다. 

### 2. etcd 실행 플래그 확인
`etcd`가 실행되고있는 쿠버네티스 노드에 접근해서 `ps aux | grep etcd` 로 명령을 확인한다.

```sh
$ ps aux | grep etcd
root        9330  2.9  1.1 11494180 180124 ?     Ssl   2024 3132:04 /usr/local/bin/etcd
root       10329  4.7  6.3 2480192 1041808 ?     Ssl   2024 5166:20 kube-apiserver --advertise-address=<server ip> --allow-privileged=true --anonymous-auth=True --apiserver-count=3 --authorization-mode=Node,RBAC --bind-address=0.0.0.0 --client-ca-file=/etc/kubernetes/ssl/ca.crt --default-not-ready-toleration-seconds=300 --default-unreachable-toleration-seconds=300 --enable-admission-plugins=NodeRestriction --enable-aggregator-routing=False --enable-bootstrap-token-auth=true --endpoint-reconciler-type=lease --etcd-cafile=/etc/ssl/etcd/ssl/ca.pem --etcd-certfile=/etc/ssl/etcd/ssl/node-controlplane.pem --etcd-compaction-interval=5m0s --etcd-keyfile=/etc/ssl/etcd/ssl/node-controlplane-key.pem --etcd-servers=https://<etcd cluster ip1>:2379,https://<etcd cluster ip2>:2379,https://<etcd cluster ip3>:2379 --event-ttl=1h0m0s --kubelet-client-certificate=/etc/kubernetes/ssl/apiserver-kubelet-client.crt --kubelet-client-key=/etc/kubernetes/ssl/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalDNS,InternalIP,Hostname,ExternalDNS,ExternalIP --profiling=False --proxy-client-cert-file=/etc/kubernetes/ssl/front-proxy-client.crt --proxy-client-key-file=/etc/kubernetes/ssl/front-proxy-client.key --request-timeout=1m0s --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/etc/kubernetes/ssl/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=6443 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/etc/kubernetes/ssl/sa.pub --service-account-lookup=True --service-account-signing-key-file=/etc/kubernetes/ssl/sa.key --service-cluster-ip-range=10.233.0.0/18 --service-node-port-range=30000-32767 --storage-backend=etcd3 --tls-cert-file=/etc/kubernetes/ssl/apiserver.crt --tls-private-key-file=/etc/kubernetes/ssl/apiserver.key
```

```sh
--etcd-cafile=/etc/ssl/etcd/ssl/ca.pem --etcd-certfile=/etc/ssl/etcd/ssl/node-controlplane.pem --etcd-keyfile=/etc/ssl/etcd/ssl/node-controlplane-key.pem
```
이부분이 `etcd`가 요구하는 인증서 정보이다.

`etcd`는 exporter가 이미 구현되어있기 때문에 별도의 exporter설치가 필요없다.

```sh
curl --cacert /etc/ssl/etcd/ssl/ca.pem \
     --cert /etc/ssl/etcd/ssl/node-controlplane.pem \
     --key /etc/ssl/etcd/ssl/node-controlplane-key.pem \
     https://<etcd ip>:2379/metrics
```
위 명령을 실행하면 `metrics`이 반환된다.


### 3. Prometheus에 인증서 마운트
#### 3-1. target 작성
기존 prometheus의 target설정을 `prometheus.prometheusSpec.additionalScrapeConfigsSecret` 으로 관리했다. (타겟파일을 작성하고 그 파일을 기반으로 시크릿을 생성하고 생성한 시크릿을 참조하는 방식) [참고](https://blog.bumgu.com/post/2024/12/22/kube_prometheus_stack_scrape_target_with_secret/)

우선 target을 작성한다.
```yaml
- job_name: kube-etcd
  scheme: https
  tls_config:
    ca_file: '/etc/prometheus/etcd-tls/ca.pem'
    cert_file: '/etc/prometheus/etcd-tls/cert.pem'
    key_file: '/etc/prometheus/etcd-tls/key.pem'
  static_configs:
    - targets:
      - <etcd ip>:2379  
      - <etcd ip>:2379  
      - <etcd ip>:2379
```

#### 3-2. 인증서를 담은 시크릿 생성
```sh
kubectl create secret generic etcd-tls-secret \
--from-file=ca.pem=/etc/ssl/etcd/ssl/ca.pem \
--from-file=cert.pem=/etc/ssl/etcd/ssl/node-controlplane.pem \
--from-file=key.pem=/etc/ssl/etcd/ssl/node-controlplane-key.pem \
``` 
파일을 참조해서 시크릿을 생성한다

#### 3-3. Prometheus Volume 마운트
`kube-prometheus-stack` helm chart의 values에서 볼륨을 마운트한다.

```yaml
prometheus:
  prometheusSpec:
    volumes:
      - name: etcd-tls
        secret:
          secretName: etcd-tls-secret
    volumeMounts:
      - name: etcd-tls
        mountPath: /etc/prometheus/etcd-tls
        readOnly: true
```

작성한뒤 `helm upgrade` 한다.


### 4. 확인
기존 Grafana의 etcd 대쉬보드에서 모두 No data로 표시되었지만

![](/images/post/18-etcd-grafana/etcd-grafana.png)


잘 뜨는것을 확인할 수 있다.
