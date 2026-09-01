# EPP+KEDA+FMA Benchmark Report

Three actuation paths, measured under KEDA [saturation-based][guide] and
[queue-based][queue-guide] autoscaling. Both sections compare the same passes,
which differ only in **who owns the model-server lifecycle**:

| Pass | Model load |
|---|---|
| **Baseline** | Cold — a new decode pod loads Qwen3-32B on every scale-up |
| **Warm** | FMA launcher already running; scale-up creates a new vLLM instance |
| **Hot** | All replicas pre-loaded during standup, then scaled to 1 (vLLM sleeps); scale-up wakes a sleeping instance |

Run on an NVIDIA H100-80GB-HBM3 OpenShift cluster, one GPU per replica. Every
metric reported below is defined in [Metric Definitions](#metric-definitions).

> [!NOTE]
> Figures below are the **mean of three runs per pass**. Run-to-run variance is
> still substantial — particularly for the Baseline and Warm columns, where some
> runs shed requests under load — so small differences between columns should not
> be read as significant. Within each section all three passes used identical
> workload, model, and autoscaling configuration — only the model-server
> lifecycle owner differs.

## Saturation-Based Autoscaling

These three passes use the autoscaling setup from
[keda-epp-saturation][guide] — KEDA polling the EPP
`llm_d_epp_flow_control_pool_saturation` and `llm_d_epp_request_running` gauges,
EPP `flowControl` feature gate on, the optimized-baseline scheduler plugins, and
a KEDA-generated HPA on the scale target.

[guide]: ../../../workload-autoscaling/keda-epp-saturation/README.md

### Configuration

The [keda-epp-saturation][guide] configuration, with `maxReplicaCount` lowered
to 6 and HPA scale-up/scale-down policies added. Thresholds, polling interval,
trigger queries, metric source, and plugin set are the guide's defaults.

| Component | Parameter | Value |
|---|---|---|
| Model | Name / max length | `Qwen/Qwen3-32B` / 16000 |
| Model | GPU memory utilization | 0.95 |
| Model | Tensor parallelism | 1 (1 GPU per replica) |
| KEDA | Trigger 1 — pool saturation threshold | `0.7` |
| KEDA | Trigger 2 — running requests threshold | `16` |
| KEDA | Polling interval | 15 s |
| HPA | Min / Max replicas | 1 / 6 |
| HPA | Scale-up | 0 s stabilization, 1 Pod / 180 s |
| HPA | Scale-down | 300 s stabilization, 1 Pod / 300 s |
| Workload | Harness / profile | `inference-perf` / `shared_prefix_synthetic_heavy.yaml` |
| Workload | Load pattern | constant, 2 workers — 1 RPS aggregate for 120 s, then 4 RPS for 300 s |
| Workload | Prompt shape | 4000-token shared prefix + 256-token question, 256-token output |
| Idle replicas | Baseline / Warm / Hot | 1 decode pod / 1 requester / 6 requesters pre-loaded, then scaled to 1 |

### Results

Mean of three runs per pass.

| Metric | Baseline | Warm | Hot | Δ% Hot vs Baseline |
| :------------------------------------- | -------: | -------: | -------: | -----: |
| Duration (s)                           | 434.7    | 463.7    | 437.3    | +0.6%  |
| Total requests                         | 1,320    | 1,320    | 1,320    | —      |
| P99 TTFT (ms)                          | 101,871.1 | 101,434.8 | 22,840.4 | −77.6% |
| P99 ITL (ms/tok)                       | 433.5    | 447.6    | 435.0    | +0.3%  |
| Avg replicas                           | 2.00     | 3.09     | 2.46     | +22.8% |
| Max replicas                           | 6.0      | 6.0      | 3.3      | −44.4% |
| Avg KV cache utilization               | 53.8%    | 47.3%    | 52.7%    | −2.1%  |
| Avg queue depth (EPP)                  | 16.3     | 8.3      | 1.2      | −92.8% |
| Avg flow-control pool saturation (EPP) | 3.34     | 1.99     | 0.76     | −77.4% |
| Avg flow-control queue (EPP)           | 31.1     | 24.6     | 1.6      | −95.0% |
| Avg running requests (EPP)             | 38.2     | 38.3     | 17.9     | −53.2% |
| Avg pod startup (s)                    | 183.0    | 108.0    | 4.7      | −97.4% |
| Hot hit rate                           | _n/a_    | 0.0%     | 100.0%   | —      |
| Warm hit rate                          | _n/a_    | 100.0%   | 0.0%     | —      |
| Failures                               | 73.3     | 86.0     | 0.0      | −100.0% |

Pod startup is the mechanism behind the rest: 183 s (baseline, cold model load)
→ 108 s (warm, new vLLM on a live launcher) → 4.7 s (hot, wake a sleeping vLLM).
Because scale-up relief arrives ~39× faster in the hot path, queues never build
and **P99 TTFT falls 78%** — from 101.9 s to 22.8 s. The cost is ~23% more average
replicas: the hot path scales out sooner precisely because it can.

The hot path is also the only one that completed every request in all three
runs. Baseline and Warm each dropped requests in two of three runs (73 and 86
failures on average, 0 for hot) — when relief arrives too late, requests time
out rather than merely waiting longer.

P99 ITL is flat across all three (~434-448 ms/tok), which is expected: once a
request is being decoded it streams at the same rate regardless of how its
replica was actuated. The actuation path affects how long a request *waits*,
not how fast it generates.

The `n/a` hit rates for Baseline are expected: there is no FMA dual-pods
controller in that pass, so no actuations to classify. The 100% split between
Warm and Hot confirms each pass exercised its intended path — every warm
actuation was a `create_instance`, every hot actuation a `wake`.

## Queue-Based Autoscaling

The same three passes under the [keda-epp-queue][queue-guide] setup, which
scales on EPP queue depth (`llm_d_epp_flow_control_queue_size`) rather than pool
saturation, with running requests as the second trigger.

[queue-guide]: ../../../workload-autoscaling/keda-epp-queue/README.md

### Configuration

The [keda-epp-queue][queue-guide] configuration, with `maxReplicaCount` lowered
to 6. Thresholds, polling interval, trigger queries, metric source, and plugin
set are the guide's defaults. Model and workload are unchanged from the
saturation section.

| Component | Parameter | Value |
|---|---|---|
| KEDA | Trigger 1 — queue size threshold | `1` |
| KEDA | Trigger 2 — running requests threshold | `16` |
| KEDA | Polling interval | 15 s |
| HPA | Min / Max replicas | 1 / 6 |
| HPA | Scale-up | 0 s stabilization, 100% / 15 s |
| HPA | Scale-down | 300 s stabilization, 100% / 15 s |

### Results

Mean of three runs per pass.

| Metric | Baseline | Warm | Hot | Δ% Hot vs Baseline |
| :------------------------------------- | -------: | -------: | -------: | -----: |
| Duration (s)                           | 461.7    | 437.0    | 433.3    | −6.1%  |
| Total requests                         | 1,320    | 1,320    | 1,320    | —      |
| P99 TTFT (ms)                          | 119,550.8 | 147,005.3 | 19,453.5 | −83.7% |
| P99 ITL (ms/tok)                       | 455.7    | 468.1    | 423.3    | −7.1%  |
| Avg replicas                           | 2.13     | 2.78     | 3.69     | +73.1% |
| Max replicas                           | 6.0      | 6.0      | 5.3      | −11.1% |
| Avg KV cache utilization               | 54.0%    | 51.6%    | 34.7%    | −35.7% |
| Avg queue depth (EPP)                  | 19.4     | 15.1     | 0.2      | −99.0% |
| Avg flow-control pool saturation (EPP) | 3.86     | 3.23     | 0.47     | −87.8% |
| Avg flow-control queue (EPP)           | 33.1     | 34.5     | 0.3      | −99.1% |
| Avg running requests (EPP)             | 42.4     | 42.0     | 13.3     | −68.6% |
| Avg pod startup (s)                    | 174.7    | 118.3    | 6.7      | −96.2% |
| Hot hit rate                           | _n/a_    | 0.0%     | 100.0%   | —      |
| Warm hit rate                          | _n/a_    | 100.0%   | 0.0%     | —      |
| Failures                               | 124.7    | 162.0    | 0.0      | −100.0% |

The same mechanism holds, more sharply: pod startup 174.7 s → 118.3 s → 6.7 s,
and **P99 TTFT falls 84%** (119.6 s → 19.5 s). Because a queue-size threshold of
`1` scales out on the first sign of backlog, the hot path drives the flow-control
queue almost to zero (33.1 → 0.3) and holds average queue depth at 0.2.

That responsiveness costs more capacity than the saturation triggers did: **~73%
more average replicas** (2.13 → 3.69, versus ~23% under saturation), and KV cache
utilization drops to 34.7% as load spreads across more replicas. Queue-based
scaling buys lower latency by running warmer.

Failures are also higher here than under saturation triggers: Baseline averaged
124.7 and Warm 162.0 dropped requests, against 73.3 and 86.0 respectively. The
Hot path completed all 1,320 requests in every run under both trigger modes.
Note that Warm's mean P99 TTFT (147.0 s) exceeds Baseline's (119.6 s) — driven by
runs 2 and 3, where Warm shed the most requests; with three runs and this much
spread, the Baseline-vs-Warm ordering is not meaningful.

## Metric Definitions

Shared by both sections above. Which EPP gauge acts as the KEDA scale trigger
differs — `llm_d_epp_flow_control_pool_saturation` for saturation-based,
`llm_d_epp_flow_control_queue_size` for queue-based, with
`llm_d_epp_request_running` as the second trigger in both — but all gauges are
reported throughout so the two modes can be compared on the same axes.

| Metric | Definition |
|---|---|
| Duration | Wall-clock length of the benchmark window (s) |
| Total requests | Requests issued by the harness over the run |
| P99 TTFT | 99th-percentile time-to-first-token (ms) — lower is better |
| P99 ITL | 99th-percentile inter-token latency (ms/token) — lower is better |
| Avg replicas | Mean ready pod count during the test window |
| Max replicas | Peak ready pod count; 6 means the `maxReplicaCount` ceiling was reached |
| Avg KV cache utilization | Mean GPU KV cache utilization across serving replicas |
| Avg queue depth (EPP) | Mean pending-request queue depth at the endpoint proxy |
| Avg flow-control pool saturation (EPP) | Mean `llm_d_epp_flow_control_pool_saturation`; >1.0 means the pool is overloaded and throttling |
| Avg flow-control queue (EPP) | Mean EPP flow-control queue size |
| Avg running requests (EPP) | Mean `llm_d_epp_request_running` — in-flight requests across the pool |
| Avg pod startup | Mean time for a new replica to become ready and serve (s) — lower is better |
| Hot hit rate | Share of run-phase FMA actuations that woke a sleeping vLLM (`wake`) |
| Warm hit rate | Share of run-phase FMA actuations that created a new vLLM on an existing launcher (`create_instance`) |
| Failures | Requests that did not complete successfully |
| Δ% Hot vs Baseline | Relative change from the Baseline column to the Hot column; sign follows the raw value, so a negative delta is an improvement for the latency, queue, and startup metrics |
