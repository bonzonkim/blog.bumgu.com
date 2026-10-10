---
title: "Vector vs OpenTelemetry Collector: 로그 수집기 비교"
author: Bumgu
draft: false
date: 2026-10-10T00:00:00+09:00
categories:
    - DevOps
    - SRE
tags:
    - Vector
    - OpenTelemetry
    - Kubernetes
    - Logging
slug: "vector-otel-log-pipeline-benchmark"
---

EKS에서 노드 로그를 많이 수집할 때 Vector를 쓸지 OpenTelemetry Collector를 쓸지 제품 소개나 기능 목록만으로 결정하기는 어렵습니다. 노드에서 파일을 읽고, 중앙으로 보내고, 파싱하고, 라우팅하는 실제 경로에서 CPU와 메모리가 어떻게 달라지는지가 궁금했습니다.

그래서 Kubernetes에 두 파이프라인을 올려 같은 형태의 1 KiB 로그를 흘려보냈습니다. 결론부터 말하면 이번 단일 노드 실행에서는 **10k logs/s에서 OTel 쪽 평균 CPU가 낮았고, Vector 쪽 메모리 사용량이 적었습니다.** 다만 이 문장은 운영 승자를 뜻하지 않습니다. 마지막 출력이 실제 저장소가 아닌 폐기 sink였고, 25k부터는 두 수집기에 들어간 부하 자체가 달랐습니다.

측정 조건과 주요 관측치를 남기되, 유효하지 않은 비교는 유효하지 않다고 표시하는 것이 이 글의 목적입니다.

## 무엇을 비교했나

두 구성 모두 생성기 Pod의 stdout을 노드 로그 파일을 통해 읽고, Agent에서 중앙 수집기로 전달한 뒤 최종 sink에서 폐기합니다.

```text
Vector
┌──────────────────┐
│ Generator Pod    │  1 KiB JSON / stdout
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Vector Agent     │  DaemonSet / kubernetes_logs
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Vector Aggregator│  Deployment / parse, filter, route
└────────┬─────────┘
         ▼
    blackhole sink
```

```text
OpenTelemetry Collector
┌──────────────────┐
│ Generator Pod    │  1 KiB JSON / stdout
└────────┬─────────┘
         ▼
┌──────────────────┐
│ OTel Agent       │  DaemonSet / filelog receiver
└────────┬─────────┘
         │ OTLP
         ▼
┌──────────────────┐
│ OTel Gateway     │  Deployment / processors, routing
└────────┬─────────┘
         ▼
      nop exporter
```

여기서 blackhole과 nop은 의도적으로 실제 저장소를 대신하지 않습니다. 매번 같은 로그를 저장소에 쓰는 비용을 빼고 수집·처리 경로를 관찰하기 위한 종단입니다. 반대로 말하면, 이 실험만으로 Loki나 Datadog까지 연결된 전체 운영 비용을 말할 수는 없습니다. 특히 OTel nop exporter에는 의미 있는 출력 레코드 수가 없어서 Gateway의 accepted 수를 최종 출력 수로 읽으면 안 됩니다.

## 실험 환경과 조건

| 항목 | 설정 |
|---|---|
| Kubernetes | OrbStack, 단일 노드 |
| 노드 / 컨테이너 런타임 | arm64 / Docker |
| Vector | `0.58.0-debian` |
| OpenTelemetry Collector | contrib `0.158.0` |
| 생성기 | Pod 1개, 1 KiB JSON, stdout |
| 생성 로그 비율 | INFO 90%, WARN 8%, ERROR 2% |
| Collector 요청 | CPU 100m, 메모리 128Mi |
| Collector 제한 | CPU 2, 메모리 1Gi |
| 측정 부하 | 1k, 5k, 10k, 25k logs/s |
| 관측 구간 | 30초 warmup 후 60초 측정 |
| 메트릭 scrape | 15초 간격 |

각 부하 단계에서 Vector와 OTel을 동시에 실행하지 않았습니다. 한 쪽을 실행하고 결과를 수집한 뒤 다른 쪽을 실행하는 방식으로 진행했습니다. 요청·제한 리소스 값은 맞췄지만, 버전이나 내부 큐의 구현까지 동일하게 만든 것은 아닙니다.

각 구현에서 같은 다섯 가지 모드를 순서대로 실행했습니다.

| 모드 | 처리 내용 |
|---|---|
| raw | 로그 전달만 수행 |
| json | JSON 파싱 |
| transform | `user_id` 제거, `status`를 `http_status`로 변경, `environment=benchmark` 및 각 수집기 이름의 `pipeline` 추가 |
| filter | DEBUG 제거, `status >= 500`인 로그를 오류 전용 경로에도 전달 |
| multiroute | 전체 / 오류(`ERROR` 또는 status 500 이상) / 느린 요청(`duration_ms >= 1000`) 경로로 라우팅 |

실제 생성기에 DEBUG 로그는 없었습니다. 따라서 filter 단계가 DEBUG 조건을 이용한 데이터 선별 비용을 대표한다고 해석하면 안 됩니다. 오류와 느린 요청 경로의 테스트 조건은 양쪽 설정에서 가능한 한 맞췄지만, 서로 다른 제품의 설정이 바이트 단위로 같은 동작을 보장하는 것은 아닙니다. `pipeline` 필드는 Vector와 OTel에서 값이 다르므로 결과 내용을 동일성 비교할 때는 제외해야 합니다.

## `Valid`는 "비교 조건 통과" 표시일 뿐이다

결과 파일의 `Valid`는 **같은 부하에서 CPU·메모리를 비교하기 위한 최소 조건을 통과했다**는 표시입니다. 실제 수집 입력률이 설정한 로그/초의 80~120%이고 CPU·메모리 지표가 있는지 확인했습니다. `Invalid`는 이 조건을 충족하지 못해 해당 수치를 동등 부하 비교에서 제외한다는 뜻이지, 테스트 프로그램이 실패했거나 로그가 확정 유실됐다는 뜻은 아닙니다. 예를 들어 25k/s로 설정했다면 최소 20k/s는 입력으로 관측돼야 하지만, Vector는 약 12k/s여서 비교에서 제외했습니다.

이 판정은 다음을 보증하지 않습니다.

- 두 실행에서 실제 생성된 로그 수가 완전히 같다.
- 데이터 손실이나 backlog가 없었다.
- 통계적으로 유의한 차이다.
- sink까지 같은 수의 레코드가 전달됐다.

이번 결과는 모드·부하별 단일 60초 측정입니다. Prometheus 결과의 `sample_count` 61도 독립된 61회 반복이라는 뜻이 아니라 측정 구간 중 range query가 평가한 sample 개수입니다. 평균과 관측 peak는 그대로 보고하되 반복 실험의 분포인 것처럼 설명하지 않겠습니다.

## 10k logs/s 결과

아래 CPU와 메모리는 Agent와 Aggregator/Gateway 역할을 합한 값입니다. CPU는 cores, 메모리는 컨테이너 메모리 사용 지표(`container_memory_working_set_bytes`)의 평균을 MiB로 환산한 값입니다. 표 수치는 각 모드의 실제 저장 JSON에 있는 `totals`를 반올림했습니다.

| 모드 | Vector CPU cores | OTel CPU cores | Vector 메모리 MiB | OTel 메모리 MiB |
|---|---:|---:|---:|---:|
| raw | 0.463 | 0.384 | 415.7 | 700.9 |
| json | 0.528 | 0.451 | 414.1 | 822.4 |
| transform | 0.556 | 0.488 | 415.1 | 820.9 |
| filter | 0.573 | 0.484 | 423.6 | 776.3 |
| multiroute | 0.613 | 0.525 | 452.7 | 740.1 |

이번 10k/s 비교 조건을 통과한 실행에서는 다섯 모드 모두 Vector보다 OTel의 총 평균 CPU가 낮게 관측됐습니다. 반대로 메모리 사용량은 모든 모드에서 Vector가 낮았습니다. 이 단일 실행에서는 CPU와 메모리의 관측 방향이 엇갈렸습니다.

예를 들어 raw에서 Vector Agent는 약 0.354 cores, Aggregator는 0.110 cores를 썼습니다. OTel Agent는 약 0.315 cores, Gateway는 0.069 cores였습니다. 전체 평균은 각각 약 0.463과 0.384 cores입니다. 역할별 비용을 더하지 않고 한쪽 컴포넌트만 비교하면 파이프라인 비교가 되지 않습니다.

여기서 Vector의 raw 결과는 blackhole sink가 실제로 받은 수치가 있지만, OTel nop은 최종 출력 count가 없습니다. OTel Gateway accepted는 Gateway ingress입니다. 따라서 수신률·CPU·메모리를 관측한 것이지, 두 제품의 end-to-end 전달 성공률을 비교한 것은 아닙니다.

또, 각 측정의 1m CPU peak는 1분 rate query가 평가한 값 중 최대입니다. scrape 사이에 발생한 순간 peak는 놓칠 수 있습니다. RSS, CPU throttling, OOM 정보도 메트릭에서 확인되지 않아 이 글에서 숫자를 만들어 채우지 않았습니다.

## 25k/s에서 같은 부하로 비교할 수 없었던 이유

25k/s로 설정했을 때 Vector Agent 입력은 모드에 따라 약 12.2k\~12.6k logs/s에 머물렀습니다. OTel은 약 25k logs/s를 관측했습니다. Vector는 설정값의 80~120% 범위에 들어오지 않아, 두 제품이 같은 양의 로그를 처리했다고 보고 CPU·메모리를 비교할 수 없었습니다.

| 관측 | Vector | OTel |
|---|---:|---:|
| 25k target에서 Agent source 입력 | 약 12.2k~12.6k/s | 약 25k/s |
| 결과 파일 표시 | Invalid (비교 제외) | Valid (조건 충족) |
| CPU 성능의 동등 부하 비교 | 불가 | 불가 |

이 차이를 Vector가 12k에서 포화됐다는 증거로 쓰면 안 됩니다. OrbStack의 Docker 로그 파일 회전과 파일 발견/스캔 타이밍도 입력률을 제한할 수 있기 때문입니다. 25k/s에서 같은 부하로 비교할 조건을 충족하지 못해 50k와 100k의 정상 부하 단계는 건너뛰었습니다. 이 벤치마크로 확인된 Vector 최대 처리량이나 데이터 손실률은 없습니다.

## 파일 발견과 Docker 런타임이 만든 변수

OrbStack 노드에서 `/var/log/pods`는 실제 Docker 컨테이너 로그 위치인 `/var/lib/docker/containers` 쪽으로 연결되는 symlink 구조였습니다. 두 Agent가 이 경로를 읽을 수 있도록 읽기 전용 마운트를 맞춰야 했습니다. 파일 경로를 직접 따라가 보지 않았다면 Vector와 OTel의 입력 위치가 서로 다른 것처럼 잘못 구성할 수 있었습니다.

또 하나의 변수는 Vector `kubernetes_logs`의 기본 파일 발견 주기였습니다. 기본 탐색 간격 60초와 맞물려 관측 입력이 약 17,733개 이벤트 단위로 증가하고 중간에 멈추는 구간이 있었습니다. `glob_minimum_cooldown_ms=1000`으로 조정하자 1k/s 입력은 따라왔지만, 25k/s에서는 여전히 목표에 미달했습니다. 이 지점에서 로그 회전과 수집기 자체의 영향을 분리하지 못했습니다.

노드 Docker 설정은 `json-file` 로그를 파일당 20 MiB, 최대 5개 파일로 회전시켜 설정상 보존 상한이 대략 100 MiB입니다. 생성률이 높을수록 오래된 파일이 수집기 파일 스캔보다 먼저 밀려날 수 있습니다. 그러므로 입력 카운터만 보고 애플리케이션 생성률이 안정적으로 Collector에 전달됐다고 단정할 수 없습니다.

## Burst: 설정 부하는 100k, 측정 구간 평균은 아니다

별도의 burst 실험은 10k → 100k → 10k logs/s로 rate를 바꾸도록 설정했습니다. 그러나 rate를 바꿀 때 Generator Deployment를 `Recreate` 방식으로 교체하면서 Pod가 다시 시작되는 공백이 있었습니다. 생성기 재기동 동안의 공백은 phase별 전달률 계산을 흐리므로, 결과 JSON에 남은 전체 계측 창 평균을 100k burst 처리율이라고 부를 수 없습니다.

기록된 **전체 계측 창의 평균 입력률**은 OTel Agent 약 25.9k/s, Vector 약 10.1k/s였습니다. 이는 100k로 설정한 burst 구간만 분리한 처리율이 아닙니다. 결과에 나타난 queue 최고값은 Vector 9,972 events, OTel 300 items였습니다. 서로 queue 단위와 구성이 다르고 Generator 재시작도 있었으므로 숫자만으로 큐 효율이나 burst 승자를 선언하지 않았습니다.

## Downstream 중단과 queue 관측

별도 backpressure 시나리오에서는 downstream Aggregator/Gateway를 약 60초 사용할 수 없도록 했습니다. 이 실험 창에서 관측한 Agent queue peak는 다음과 같습니다.

| Collector | Agent queue peak | 단위 |
|---|---:|---|
| Vector | 10,000 | events |
| OTel | 9,968 | items |

이는 queue가 커졌다는 관측이지 손실 수치가 아닙니다. refused 또는 send-failed 같은 지표만으로 개별 로그가 유실됐다고 확정할 수도 없습니다. 기존 backpressure 실행은 정상 처리율 복귀 시점과 sequence 검증을 수집하지 않아, 결과 보고서에서는 손실 및 recovery를 N/A로 남겼습니다.

이후 별도로 1k/s 입력을 유지하면서 장애를 주입하고, 정상 수신률 및 Agent queue가 장애 전 수준으로 돌아오는 관측을 추가했습니다. **시간의 기준점은 복구 요청 시각**입니다. 생성 sequence와 Gateway 수신 카운터의 8~15초 구간 추정률이 각각 1k/s의 ±10%이고, Agent 큐가 장애 전 평균+200 이내인 관측이 30초 이상 이어지는지를 판정했습니다. 표의 첫 충족 범위와 30초 유지 확인 시각은 다릅니다. 종단 저장이나 밀린 로그의 완전 배출을 확인한 시간은 아닙니다.

앞의 프로파일 측정은 Prometheus를 15초마다 scrape했습니다. 이 재시험은 별도로 수집기 메트릭과 생성기 sequence를 약 1.75초 간격으로 직접 관측하도록 설정했고, 실제 관측 간격은 호출 지연에 따라 달라질 수 있습니다.

| Collector | 장애 | 정상 수신률 최초 관측 | 30초 유지 확인 |
|---|---|---:|---:|
| Vector | Gateway Pod 60초 중단 | 43–46초 | 78초 |
| OTel | Gateway Pod 60초 중단 | 23–25초 | 55초 |
| Vector | Agent 재시작, 약 50 MB 생성 목표 | 18–21초 | 53초 |
| OTel | Agent 재시작, 약 50 MB 생성 목표 | 0–13초 | 44초 |

Agent 재시작은 실제 보존된 파일 바이트를 측정한 것이 아니라 약 50 MB의 생성 목표를 둔 실험입니다. 50 MB는 명목상 약 100 MiB의 보존 상한보다 작지만 여러 20 MiB 파일에 걸쳐 생성됩니다. 실제 잔존량과 재시작한 수집기가 회전 파일을 모두 발견해 재생했는지는 확인하지 못했습니다. OTel의 0초 하한은 즉시 복구됐다는 뜻이 아닙니다. 첫 관측 전의 실패 상태를 확인할 수 없어 하한을 좁히지 못했다는 의미입니다. 전체 backlog 배출 시간과 개별 로그 무손실 여부는 여전히 N/A입니다.

처음 시도한 OTel Gateway 장애 실험 하나는 노드 로그 시각이 로컬 관측 시간보다 약 3.9초 앞서 baseline 검증에서 중단되어 제외했습니다. 재시험은 로그 시각과 로컬 관측 시점을 맞춘 근사 방법을 사용했습니다. 관측 도중 시간 offset도 달라졌기 때문에 이 값으로 절대 지연이나 로그 신선도를 주장하지 않습니다.

## Backlog 재생은 목표 크기에 도달하지 못했다

계획했던 backlog 테스트 목표는 500 MB였습니다. 하지만 Docker `json-file` 회전이 최대 약 100 MiB라서 500 MB의 로그가 파일로 유지되는 조건을 만들지 못했습니다. 실제 retained bytes도 측정되지 않아 N/A입니다. 1 GB와 5 GB 단계는 실행하지 않았습니다.

따라서 이 실험은 수 GB 로그 파일을 수집기가 얼마나 빨리 읽어내는지 답하지 못합니다. `Agent 재시작` 복귀 표를 수 GB backlog drain 결과로 해석해서도 안 됩니다. persistent queue와 노드 로그 파일 보존량을 각각 측정하고, 재생된 sequence 번호를 종단에서 검증해야 backlog recovery 비교가 됩니다.

## 이번 실험이 확인한 것과 확인하지 못한 것

### 확인된 범위

- OrbStack 단일 arm64/Docker 노드에서 1k~10k target 구간의 source 입력률과 Collector resource metrics를 관측했습니다.
- 10k의 각 모드에서 OTel의 총 평균 CPU가 낮았고 Vector의 메모리 사용량은 적었습니다.
- 25k Vector 입력률은 목표에 미달해 OTel과의 동등 부하 성능 비교가 불가능했습니다.
- Docker 로그 파일 발견 주기와 보존 회전 설정이 측정 가능한 입력률에 영향을 줄 수 있음을 확인했습니다.
- 60초 downstream 중단 중 두 Agent 모두에서 queue가 증가한 실행을 기록했습니다.

### 확인되지 않은 범위

- 확정 데이터 손실률: sequence sink가 없어 측정 불가.
- OTel nop을 통과한 최종 출력 레코드 수.
- Vector 최대 지속 처리량 또는 포화 지점.
- 50k·100k steady-state 성능.
- 500 MB 이상 실제 보존 backlog의 재생 시간.
- 운영 backend가 있는 상태의 end-to-end CPU, 메모리, 재시도와 내구성.
- 여러 번 반복한 통계적 성능 차이.

결과 원본과 메트릭별 세부 값은 벤치마크 저장소의 `results/report.md`, `results/suite-progress.json`, 개별 JSON 파일에 기록했습니다. 이 글의 숫자는 그 자료에 저장된 관측치만 인용했습니다.

## EKS 운영 환경에 옮길 때

이번에는 단일 노드에서 두 구현을 번갈아 실행했습니다. EKS의 containerd, 여러 노드, 실제 노드 로그 디렉터리, CSI 또는 hostPath 사용 방식, 네트워크 지연, 실제 backend queue 설정은 이 환경과 다릅니다. 이번 25k/s 입력 제한은 Docker 로그 회전·파일 발견 주기의 영향과 수집기 자체의 영향을 분리하지 못했으므로 EKS 성능 상한으로 옮겨 쓸 수 없습니다.

운영 후보를 고르려면 최소한 실제 backend를 붙이고 같은 source workload에서 반복 실행해야 합니다. 생성기 sequence를 저장소까지 보존하는 검증 sink가 필요하고, Agent·Gateway의 queue size/capacity, 재시도, 처리량, 실제 파일 보존량을 같은 시간축으로 비교해야 합니다. disk buffer를 쓸 경우에는 재시작 이후 데이터가 유지되는지도 별도 확인해야 합니다.

다음에 시험해보고 싶은 구성은 **Vector Agent → OTel Gateway → Loki 또는 Datadog**입니다. Vector 0.58의 native OTLP/HTTP 기능을 이용하는 방향이지만, Vector 이벤트를 OTLP log envelope로 명시적으로 매핑하는 작업이 필요합니다. 이 hybrid 구성은 아직 시험하지 않았습니다. Agent에서 보내는 출력의 의미와 변환을 확정한 다음, backend별 연결과 queue 동작을 따로 검증할 계획입니다.

중앙에 Vector Aggregator를 추가하는 hybrid도 당연한 기본값은 아닙니다. Collector 역할이 실제로 필요하다는 가설이 있을 때만 별도 중앙 계층을 넣어 비교해야 합니다. 단순히 두 제품이 섞였다는 이유로 수집기 하나를 더 두면 운영 구성만 복잡해집니다.

## 결론

각 모드를 한 번씩 60초 측정한 10k/s 결과에서는 CPU와 메모리의 trade-off가 보였습니다. OTel은 각 모드에서 평균 CPU가 낮았고, Vector는 메모리 사용량이 적었습니다. 그러나 25k 구간에서는 입력률이 달라졌고, nop과 blackhole의 종단 관측 가능성도 같지 않았습니다.

따라서 현재 데이터로 “Vector가 더 빠르다” 또는 “OTel을 EKS에 써야 한다”고 결론 내릴 수 없습니다. 이 측정은 운영 결정을 대신하기보다, 다음 실험에서 무엇을 통제해야 하는지 보여주는 예비 비교에 가깝습니다.

적어도 이번 경험으로는 로그 수집 성능을 비교할 때 Collector 버전만큼 런타임 파일 경로, discovery 주기, 컨테이너 로그 회전 정책, 실제 출력 경계도 중요하다는 점을 확인했습니다. 이 항목을 통제하지 않은 throughput 숫자는 수집기 자체의 한계인지 주변 로그 파이프라인의 한계인지 구분하기 어렵습니다.
