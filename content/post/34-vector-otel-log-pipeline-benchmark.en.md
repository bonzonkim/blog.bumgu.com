---
title: "Vector vs OpenTelemetry Collector: Log Collector Comparison"
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

When collecting a large volume of node logs on EKS, it is difficult to decide whether to use Vector or the OpenTelemetry Collector based only on product descriptions or feature lists. I wanted to know how CPU and memory usage differ along the actual path: reading files on the node, sending them centrally, parsing them, and routing them.

So I deployed both pipelines on Kubernetes and sent them the same kind of 1 KiB logs. To give the conclusion first: **in this single-node run, OTel had lower average CPU usage at 10k logs/s, while Vector used less memory.** However, that does not mean either is the operational winner. The final output was a discard sink rather than a real storage backend, and from 25k onward the load entering the two collectors was itself different.

The purpose of this post is to record the measurement conditions and key observations, while labeling invalid comparisons as invalid.

## What was compared

Both configurations read the Generator Pod's stdout through the node log files, forward it from an Agent to a central collector, and discard it at the final sink.

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

The blackhole and nop are intentionally not substitutes for an actual storage backend. They are end points for observing the collection and processing path without the cost of writing the same logs to storage each time. Conversely, this experiment cannot tell us the total operational cost of a full path connected to Loki or Datadog. In particular, the OTel nop exporter has no meaningful output record count, so the Gateway's accepted count must not be read as the final output count.

## Test environment and conditions

| Item | Configuration |
|---|---|
| Kubernetes | OrbStack, single node |
| Node / container runtime | arm64 / Docker |
| Vector | `0.58.0-debian` |
| OpenTelemetry Collector | contrib `0.158.0` |
| Generator | 1 Pod, 1 KiB JSON, stdout |
| Generated log levels | INFO 90%, WARN 8%, ERROR 2% |
| Collector requests | CPU 100m, memory 128Mi |
| Collector limits | CPU 2, memory 1Gi |
| Measured load | 1k, 5k, 10k, 25k logs/s |
| Observation window | 60 seconds measured after 30 seconds of warmup |
| Metrics scrape interval | 15 seconds |

Vector and OTel were not run simultaneously at each load level. I ran one, collected the results, and then ran the other. Request and limit resource values were matched, but versions and internal queue implementations were not made identical.

For each implementation, the same five modes were run in sequence.

| Mode | Processing |
|---|---|
| raw | Forward logs only |
| json | Parse JSON |
| transform | Remove `user_id`, rename `status` to `http_status`, add `environment=benchmark` and a `pipeline` value naming each collector |
| filter | Remove DEBUG; also send logs with `status >= 500` to an error-only path |
| multiroute | Route to all / error (`ERROR` or status 500 and above) / slow request (`duration_ms >= 1000`) paths |

The actual generator produced no DEBUG logs. Therefore, the filter stage should not be interpreted as representing the cost of selecting data using the DEBUG condition. The test conditions for the error and slow-request paths were matched as closely as possible in both configurations, but settings from different products cannot guarantee byte-for-byte identical behavior. Since the `pipeline` field has different values in Vector and OTel, it should be excluded when comparing result contents for equality.

## `Valid` means only that the comparison conditions were met

`Valid` in the result files means **the minimum conditions for comparing CPU and memory at the same load were met**. We checked that the actual input rate was 80–120% of the configured logs per second and that CPU and memory metrics were available. `Invalid` means the result did not meet these conditions and is excluded from an equal-load comparison; it does not mean the test program failed or that logs were confirmed lost. For example, with a 25k/s target, at least 20k/s must be observed as input, but Vector reached only about 12k/s and was excluded from the comparison.

This assessment does not guarantee that:

- The actual number of logs generated in both runs was exactly the same.
- There was no data loss or backlog.
- A difference was statistically significant.
- The same number of records reached the sink.

These results are single 60-second measurements for each mode and load. The Prometheus result's `sample_count` of 61 does not mean 61 independent repetitions; it is the number of samples evaluated by the range query during the measurement window. I report the averages and observed peaks as they are, without describing them as distributions from repeated experiments.

## Results at 10k logs/s

The CPU and memory values below combine the Agent and Aggregator/Gateway roles. CPU is in cores; memory is the average of the container memory usage metric (`container_memory_working_set_bytes`), converted to MiB. The table values are rounded from the `totals` in the actual JSON stored for each mode.

| Mode | Vector CPU cores | OTel CPU cores | Vector memory MiB | OTel memory MiB |
|---|---:|---:|---:|---:|
| raw | 0.463 | 0.384 | 415.7 | 700.9 |
| json | 0.528 | 0.451 | 414.1 | 822.4 |
| transform | 0.556 | 0.488 | 415.1 | 820.9 |
| filter | 0.573 | 0.484 | 423.6 | 776.3 |
| multiroute | 0.613 | 0.525 | 452.7 | 740.1 |

Among runs that met the comparison conditions at 10k/s, OTel had lower total average CPU than Vector in all five modes. Conversely, Vector used less memory in every mode. In this single run, the observed directions for CPU and memory diverged.

For example, in raw mode, the Vector Agent used about 0.354 cores and the Aggregator 0.110 cores. The OTel Agent used about 0.315 cores and the Gateway 0.069 cores. The respective overall averages were about 0.463 and 0.384 cores. Comparing just one component without adding the role-level costs is not a pipeline comparison.

Vector's raw result includes a value actually received by the blackhole sink, but OTel nop has no final output count. OTel Gateway accepted is the Gateway ingress. Therefore, these measurements observe input rate, CPU, and memory; they do not compare end-to-end delivery success rates for the two products.

Also, the 1m CPU peak for each measurement is the maximum among values evaluated by a one-minute rate query. It can miss instantaneous peaks between scrapes. RSS, CPU throttling, and OOM information were not available in the metrics, so I have not made up numbers for them in this post.

## Why 25k/s could not be compared at the same load

At a configured rate of 25k/s, Vector Agent input stayed at about 12.2k–12.6k logs/s, depending on the mode. OTel observed about 25k logs/s. Vector was outside the 80–120% range of the configured value, so CPU and memory could not be compared as if both products had processed the same volume of logs.

| Observation | Vector | OTel |
|---|---:|---:|
| Agent source input at 25k target | about 12.2k–12.6k/s | about 25k/s |
| Result file status | Invalid (excluded from comparison) | Valid (conditions met) |
| Equal-load CPU performance comparison | Not possible | Not possible |

This difference should not be used as evidence that Vector saturated at 12k. Docker log file rotation and file discovery/scanning timing in OrbStack may also limit the input rate. Because the conditions for an equal-load comparison at 25k/s were not met, the normal-load stages at 50k and 100k were skipped. This benchmark did not establish Vector's maximum throughput or data loss rate.

## Variables introduced by file discovery and the Docker runtime

On the OrbStack node, `/var/log/pods` was a symlink structure leading to the actual Docker container log location under `/var/lib/docker/containers`. The read-only mounts had to be matched so both Agents could read this path. Without following the file paths directly, it would have been easy to misconfigure Vector and OTel as if their input locations were different.

Another variable was the default file discovery interval for Vector `kubernetes_logs`. In combination with the default 60-second discovery interval, observed input increased in chunks of about 17,733 events and then stopped for periods. After adjusting it to `glob_minimum_cooldown_ms=1000`, it kept up with 1k/s input, but still fell short of the target at 25k/s. At this point, the effects of log rotation and the collector itself could not be separated.

The node's Docker configuration rotates `json-file` logs at 20 MiB per file, with a maximum of 5 files, for a configured retention ceiling of about 100 MiB. At higher generation rates, older files may be removed before the collector scans them. Therefore, input counters alone cannot establish that the application's generation rate was reliably delivered to the Collector.

## Burst: configured load was 100k, not the measured-window average

A separate burst experiment was configured to change the rate from 10k → 100k → 10k logs/s. However, changing the rate replaced the Generator Deployment using the `Recreate` strategy, leaving a gap while the Pod restarted. That gap during generator restarts distorts per-phase delivery-rate calculations, so the overall instrumented-window average in the result JSON cannot be called the processing rate of the 100k burst.

The recorded **average input rates for the entire instrumented window** were about 25.9k/s for OTel Agent and 10.1k/s for Vector. These are not rates isolated to the burst phase configured at 100k. The peak queue values shown in the results were 9,972 events for Vector and 300 items for OTel. The queue units and configurations differ, and the Generator restarted, so I did not declare a queue-efficiency winner or burst winner based on these numbers alone.

## Downstream outage and queue observations

In a separate backpressure scenario, the downstream Aggregator/Gateway was made unavailable for about 60 seconds. The Agent queue peaks observed during this experiment window were:

| Collector | Agent queue peak | Unit |
|---|---:|---|
| Vector | 10,000 | events |
| OTel | 9,968 | items |

This is an observation that the queues grew, not a loss measurement. Metrics such as refused or send-failed cannot by themselves confirm that individual logs were lost either. The earlier backpressure run did not collect the time to return to normal throughput or sequence validation, so loss and recovery were left as N/A in the results report.

In a separate follow-up, I injected a failure while maintaining 1k/s input and added observations of normal receive rate and Agent queue recovery to pre-failure levels. **The reference point for time is the recovery request time.** I assessed whether the estimated rate over 8–15-second windows from the generated sequence and Gateway receive counter was each within ±10% of 1k/s, and whether the Agent queue remained within 200 of its pre-failure average for at least 30 seconds. The first qualifying range in the table and the time at which 30 seconds of stability was confirmed are different. This is not the time at which end-to-end storage or complete draining of queued logs was verified.

The profile measurements above scraped Prometheus every 15 seconds. For this retest, collector metrics and the generator sequence were observed directly at roughly 1.75-second intervals; the actual observation interval could vary with call latency.

| Collector | Failure | First observation of normal receive rate | 30-second stability confirmed |
|---|---|---:|---:|
| Vector | Gateway Pod stopped for 60 seconds | 43–46 seconds | 78 seconds |
| OTel | Gateway Pod stopped for 60 seconds | 23–25 seconds | 55 seconds |
| Vector | Agent restart, about 50 MB generation target | 18–21 seconds | 53 seconds |
| OTel | Agent restart, about 50 MB generation target | 0–13 seconds | 44 seconds |

The Agent restart test did not measure the actual retained file bytes; it was an experiment with a generation target of about 50 MB. Although 50 MB is below the nominal retention ceiling of about 100 MiB, it is generated across multiple 20 MiB files. The actual remaining amount and whether the restarted collector discovered and replayed all rotated files were not verified. The 0-second lower bound for OTel does not mean it recovered immediately. It means the failure state before the first observation could not be confirmed, so the lower bound could not be narrowed. Total backlog drain time and whether individual logs were lossless remain N/A.

One initial OTel Gateway outage test was excluded because the node log timestamps were about 3.9 seconds ahead of local observation time and the run stopped during baseline validation. The retest used an approximate method that aligned log timestamps with local observation time. Since the time offset also changed during observation, these values should not be used to claim absolute latency or log freshness.

## Backlog replay did not reach the target size

The planned backlog test target was 500 MB. However, Docker `json-file` rotation is limited to about 100 MiB, so we could not create a condition in which 500 MB of logs remained in files. Actual retained bytes were also not measured and are N/A. The 1 GB and 5 GB stages were not run.

Therefore, this experiment does not answer how quickly a collector can read through several gigabytes of log files. The `Agent restart` recovery table should not be interpreted as a multi-gigabyte backlog drain result either. A backlog recovery comparison requires measuring persistent queues and node log retention separately, then validating the replayed sequence numbers at the end point.

## What this experiment did and did not establish

### What was established

- On a single OrbStack arm64/Docker node, source input rates and Collector resource metrics were observed in the 1k–10k target range.
- At 10k, OTel had lower total average CPU in each mode, while Vector used less memory.
- Vector input fell short of the target at 25k, making an equal-load performance comparison with OTel impossible.
- Docker log file discovery intervals and retention rotation settings can affect the measurable input rate.
- A run was recorded in which the queues grew on both Agents during a 60-second downstream outage.

### What was not established

- Confirmed data loss rate: impossible to measure without a sequence sink.
- The final output record count that passed through OTel nop.
- Vector's maximum sustained throughput or saturation point.
- 50k and 100k steady-state performance.
- Replay time for an actually retained backlog of 500 MB or more.
- End-to-end CPU, memory, retries, and durability with an operational backend.
- Statistical performance differences from multiple repetitions.

The source results and detailed per-metric values are recorded in the benchmark repository's `results/report.md`, `results/suite-progress.json`, and individual JSON files. The figures in this post cite only observations stored in those materials.

## Applying this to an EKS production environment

In this test, the two implementations were run alternately on a single node. EKS's containerd, multiple nodes, actual node log directories, CSI or hostPath usage, network latency, and real backend queue settings differ from this environment. The input limitation at 25k/s could not separate the effects of Docker log rotation and file discovery intervals from the collector itself, so it cannot be used as an EKS performance ceiling.

To choose a production candidate, the minimum requirement is to connect the real backend and run repeated tests with the same source workload. A validation sink that preserves generator sequences through to storage is needed, and Agent/Gateway queue size and capacity, retries, throughput, and actual file retention should be compared on the same timeline. If using a disk buffer, it should also be verified separately that data survives a restart.

A configuration I would like to test next is **Vector Agent → OTel Gateway → Loki or Datadog**. This would use Vector 0.58's native OTLP/HTTP capability, but requires explicitly mapping Vector events to an OTLP log envelope. This hybrid configuration has not yet been tested. After defining the meaning of the output sent by the Agent and its transformations, I plan to validate connections and queue behavior separately for each backend.

A hybrid that adds a Vector Aggregator centrally is not an automatic default either. A separate central layer should be added for comparison only when there is a hypothesis that a Collector role is actually needed. Adding another collector just because the two products are mixed only makes the operational setup more complex.

## Conclusion

In the 10k/s results, measured once for 60 seconds per mode, a CPU/memory trade-off was visible. OTel had lower average CPU in each mode, while Vector used less memory. However, input rates differed in the 25k range, and the end-point observability of nop and blackhole was not the same.

Therefore, the current data cannot support the conclusion that “Vector is faster” or that “OTel should be used on EKS.” Rather than replacing an operational decision, these measurements are closer to a preliminary comparison that shows what needs to be controlled in the next experiment.

At least from this experience, when comparing log collection performance, the runtime file paths, discovery intervals, container log rotation policy, and actual output boundary matter as much as the Collector version. Throughput figures that do not control for these factors make it difficult to distinguish limits of the collector itself from limits of the surrounding log pipeline.
