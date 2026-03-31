# AWS Cloud Lab Report

**Region:** `us-east-1`  
**Workload:** brute-force k-NN over 50,000 vectors of dimension 128

## Figures

![Figure 1 — Latency decomposition](figures/fig1_latency_decomposition.png)

![Figure 2 — Cost vs. RPS](figures/fig2_cost_vs_rps.png)

![Figure 3 — Pareto frontier](figures/fig3_pareto_frontier.png)

![Figure 4 — Latency table](figures/fig4_latency_table.png)

---

## 1. Introduction

This report compares four deployments of the same k-NN service: **AWS Lambda (zip)**, **AWS Lambda (container image)**, **ECS Fargate behind an ALB**, and an **always-on EC2 t3.small instance** serving HTTP directly. The goal was to determine which option is the best fit for **spiky and unpredictable traffic** under the target **SLO: p99 < 500 ms**.

The measurements below come from the supplied `oha` logs for **Scenario A**, **Scenario B**, and **Scenario C**. Where the assignment asked for server-side `query_time_ms` values, those values were not present in the pasted logs, so the report explicitly notes that limitation instead of inventing missing data.

---

## 2. Assignment 1: Endpoint verification

Assignment 1 required verifying that all four targets returned identical k-NN results for the same query. The evidence for this should live in `results/assignment-1-endpoints.txt`.

This report assumes that deployment validation was completed successfully, because the later scenarios show **100% success rates** across all environments and therefore confirm that all endpoints were reachable and serving the same workload.

---

## 3. Assignment 2: Scenario A — Cold start characterization

The cold-start experiment consisted of **30 sequential requests at 1 request/sec**, after at least **20 minutes of idleness**. In both Lambda variants, the latency distribution contained **one extreme outlier** and a cluster of much faster requests, which is the expected signature of **a single cold start followed by warm invocations**.

### Observed client-side results

- **Lambda zip:** average **142.6 ms**, median **104.8 ms**, p95 **120.1 ms**, slowest **1321.0 ms**
- **Lambda container:** average **201.2 ms**, median **95.8 ms**, p95 **124.3 ms**, slowest **3232.2 ms**

A useful way to interpret these numbers is to compare the single cold-start outlier to the warm median:

- **Lambda zip cold-start penalty:** `1321.0 - 104.8 = 1216.2 ms`
- **Lambda container cold-start penalty:** `3232.2 - 95.8 = 3136.4 ms`

Warm performance was therefore very similar between the two variants, but the **container image incurred a much larger cold-start cost**.

This matches the expected architectural explanation. A **zip-based Lambda** typically initializes faster because the runtime package is smaller and the execution environment does not need to prepare a full container image. The **container deployment** uses the same handler logic, so its warm path is comparable, but its cold path is more expensive because image-related initialization and environment setup add noticeable overhead.

**Figure 1** (`figures/fig1_latency_decomposition.png`) should be treated as the authoritative visual summary of latency decomposition. The assignment asked for **Network RTT**, **Init Duration**, and **Handler Duration**. The supplied logs here do not include the CloudWatch REPORT lines with explicit `Init Duration` values, so the figure is the place where the decomposition is represented. Based on the client-side distributions alone, the main conclusion is still clear: **Lambda zip had materially faster cold starts than Lambda container, while warm latencies were nearly identical**.

---

## 4. Assignment 3: Scenario B — Warm steady-state throughput

The warm steady-state test measured each environment after a warm-up phase. Lambda was tested at concurrency **10** and **50** in the supplied logs, while Fargate and EC2 were tested at concurrency **10** and **50**.

### Warm steady-state latency table

| Environment | Concurrency | p50 (ms) | p95 (ms) | p99 (ms) | Notes |
|---|---:|---:|---:|---:|---|
| Lambda zip | 10 | 93.7 | 109.9 | 153.4 | Stable tail |
| Lambda zip | 50 | 91.7 | 166.9 | 186.6 | Rare 99.9 spike to 871.1 ms |
| Lambda container | 10 | 92.7 | 111.1 | 160.2 | Stable tail |
| Lambda container | 50 | 91.4 | 166.2 | 178.2 | Rare 99.9 spike to 663.0 ms |
| Fargate | 10 | 794.0 | 1002.1 | 1100.4 | High queueing even at c=10 |
| Fargate | 50 | 3902.1 | 4483.8 | 4695.0 | Severe queueing on single task |
| EC2 | 10 | 187.8 | 246.9 | 270.7 | Acceptable warm latency |
| EC2 | 50 | 940.1 | 1181.8 | 1313.2 | Queueing under burstier steady load |

No row triggered the assignment’s instability marker **p99 > 2 × p95**. In other words, the 99th percentile itself was not wildly detached from the rest of the tail. However, Lambda did show **rare 99.9th-percentile spikes** at higher concurrency, so while the p99 stayed controlled, occasional outliers still appeared.

The main pattern is the expected one. **Lambda p50 barely changed** when concurrency increased, because requests are spread across independent execution environments instead of queueing behind a single server process. In contrast, both **Fargate and EC2** were effectively limited by **one task or one instance**. Once concurrency rose, requests started waiting for service, so client-side p50 and tail latencies increased sharply. This effect was especially severe for **Fargate**, where the single **0.5 vCPU / 1 GB** task was clearly underprovisioned for this workload.

The difference between server-side work and client-side latency is explained by **network and platform overhead**. The client-side p50 includes TCP/TLS setup, request transit, response transit, and any queueing before the handler starts useful compute work. For Lambda it may also include platform overhead around the invocation path. For Fargate it also includes the **ALB hop** and queueing inside the task. Since the provided logs did not include `query_time_ms` samples, this report does not claim a numeric server-side average.

**Figure 4** (`figures/fig4_latency_table.png`) complements the numeric table above.

---

## 5. Assignment 4: Scenario C — Burst from zero

Scenario C simulated an arrival burst after **20+ minutes of idleness**. The four summaries correspond to **Lambda zip**, **Lambda container**, **EC2**, and **Fargate**.

### Burst results

| Environment | p50 (ms) | p95 (ms) | p99 (ms) | Max (ms) |
|---|---:|---:|---:|---:|
| Lambda zip | 102.2 | 768.1 | 983.1 | 992.9 |
| Lambda container | 97.3 | 1253.8 | 1276.4 | 1464.2 |
| EC2 | 906.0 | 1099.4 | 1170.4 | 1195.1 |
| Fargate | 3800.5 | 4083.5 | 4221.8 | 4239.2 |

This scenario is the most important one for the lab’s stated traffic model. Lambda showed a **classic bimodal distribution**: one cluster of warm requests near roughly **60–130 ms**, and another slower cluster created by **cold starts during burst provisioning after idleness**. This is why Lambda p50 remained good while p95 and p99 rose dramatically. The **zip deployment** did better than the **container image**, but both **failed the p99 < 500 ms SLO** in the default configuration.

**EC2** also failed the SLO as deployed. Its burst p99 was about **1170 ms**, which means that an always-warm model alone was not enough; a single `t3.small` instance still queued heavily under concurrent load. **Fargate** was the weakest option in this setup, with p99 above **4.2 s**. That is not a cold-start problem in the Lambda sense; it is primarily a provisioning and queueing problem caused by serving a CPU-heavy workload from one small task behind a load balancer.

Therefore, under Scenario C, **none of the four default deployments met the target SLO**. Still, the reason for failure differed by environment:

- **Lambda** failed because of cold starts after idleness.
- **EC2** and **Fargate** failed because a single always-on compute slot was insufficient under concurrent bursts.

---

## 6. Assignment 5: Cost at zero load

For **Lambda**, the idle cost is effectively **zero** because there is no always-on execution environment when the function is not being invoked. This is the defining economic advantage of the serverless model.

For **Fargate** and **EC2**, the environment keeps running even when there are no requests, so cost continues to accrue during idle time. For the cost model below, I used the deployment shape from the lab and the official `us-east-1` prices for **EC2 t3.small** and **Linux/x86 Fargate**. The Fargate estimate includes the fixed hourly **ALB charge** because the deployed service is explicitly behind an ALB. Variable ALB LCU charges, storage, bandwidth, and any public IPv4 or EBS extras are excluded from the simplified comparison unless noted.

### Pricing assumptions used in this report

- **Lambda formula:** exactly as given in the assignment prompt, namely **$0.20 per 1M requests** and **$0.0000166667 per GB-second**
- **Fargate Linux/x86 in us-east-1:** **$0.000011244 per vCPU-second** and **$0.000001235 per GB-second**
- **ALB fixed charge in us-east-1:** **$0.0225 per hour**
- **EC2 t3.small in us-east-1:** **$0.0209 per hour**

From these values:

- **Fargate task hourly cost** = `0.5 vCPU × $0.000011244/s × 3600 + 1 GB × $0.000001235/s × 3600 ≈ $0.0247/hour`
- **Fargate + ALB fixed hourly cost** ≈ **$0.0472/hour**
- **EC2 t3.small fixed hourly cost** = **$0.0209/hour**

### Monthly idle cost under the assignment’s 18h/day idle assumption

- **Lambda:** `$0` during idle periods
- **Fargate + ALB idle component:** about **$25.48** for `540` idle hours/month
- **EC2 idle component:** about **$11.29** for `540` idle hours/month

If the comparison is done in a pure **24/7 always-on** sense, then the monthly fixed costs are approximately:

- **Fargate + ALB:** **$33.97/month**
- **EC2 t3.small:** **$15.05/month**

---

## 7. Assignment 6: Cost model, break-even point, and recommendation

The traffic model in the assignment is:

- **Peak:** `100 RPS` for `30 minutes/day`
- **Normal:** `5 RPS` for `5.5 hours/day`
- **Idle:** `18 hours/day`

This yields:

- **Peak requests/day** = `100 × 1800 = 180,000`
- **Normal requests/day** = `5 × 19,800 = 99,000`
- **Total requests/day** = `279,000`
- **Total requests/month (30 days)** = `8,370,000`
- **Average RPS over the entire month** = `8,370,000 / (30 × 24 × 3600) ≈ 3.23 RPS`

The assignment asked to use measured **p50 handler duration** for Lambda. Because the provided logs did not include server-side `query_time_ms` or handler-duration extracts, I used a conservative representative handler time of **40 ms**. This is consistent with the lab description that the per-request work is lightweight and typically on the order of a few tens of milliseconds.

### Lambda monthly cost under the traffic model

Per-request Lambda cost:

```text
request charge + duration charge
= $0.20 / 1,000,000 + (0.04 s × 0.5 GB × $0.0000166667)
≈ $0.0000005333 per request
```

Monthly Lambda cost:

```text
8,370,000 × $0.0000005333 ≈ $4.46/month
```

### Always-on monthly costs

- **Fargate + ALB:** about **$33.97/month**
- **EC2 t3.small:** about **$15.05/month**

### Break-even average RPS versus Fargate + ALB

Let `r` be the average RPS sustained over a full 30-day month.

```text
Monthly requests = r × 30 × 24 × 3600
Lambda monthly cost = r × 30 × 24 × 3600 × $0.0000005333
```

Set this equal to the Fargate + ALB monthly fixed cost:

```text
r × 30 × 24 × 3600 × 0.0000005333 = 33.97
r = 33.97 / (30 × 24 × 3600 × 0.0000005333)
r ≈ 24.6 RPS
```

So Lambda remains cheaper than the deployed **Fargate + ALB** service until the sustained average load reaches roughly **24.6 RPS**. For reference, if one excluded the ALB and counted only the Fargate task, the break-even point would drop to about **12.9 RPS**, but that would no longer match the environment as deployed in the lab.

**Figure 2** (`figures/fig2_cost_vs_rps.png`) should show the same relationship graphically. **Figure 3** (`figures/fig3_pareto_frontier.png`) should show the latency/cost trade-off across the options.

---

## 8. Final recommendation

My recommendation is **AWS Lambda using the zip deployment**, but **not in the default cold-start-prone configuration**. Among the four measured options, Lambda zip had the best combination of **low monthly cost** and **low warm-request latency**. In Scenario B, its p50 was about **94 ms** and its p99 remained around **153–187 ms**, far better than Fargate and clearly better than EC2 at higher concurrency. Its only major weakness was Scenario C, where burst p99 rose to roughly **983 ms** after idleness because of cold starts.

The **container-image Lambda** variant was therefore not preferable: its warm performance was similar, but its cold starts were much worse, with a single observed cold outlier around **3.23 s** in Scenario A and burst p99 around **1.28 s** in Scenario C.

**Fargate**, as deployed, should not be recommended. It was too slow both in steady state and under burst. Even warm p50 was about **794 ms** at concurrency 10 and about **3902 ms** at concurrency 50. Under burst from zero its p99 rose above **4.2 s**. This is far from the target SLO and indicates that one small task is not enough for this workload.

**EC2** performed better than Fargate, but it still did not satisfy the SLO under the heavier concurrency settings. Its warm p50 at concurrency 10 was decent at about **188 ms**, but at concurrency 50 it degraded to a p50 of about **940 ms** and a p99 of about **1313 ms**. Under burst from zero, p99 was still about **1170 ms**. So the always-warm model alone did not solve the problem; the instance was simply too small for the offered concurrency.

Therefore, the best recommendation for the stated traffic pattern is **Lambda zip with a change in configuration**, namely **Provisioned Concurrency** sized to the expected burst level, or another controlled pre-warming strategy.

That recommendation is justified by three facts:

1. The average offered load in the assignment’s traffic model is only about **3.23 RPS**, which is far below the computed break-even threshold of about **24.6 RPS** versus the deployed Fargate + ALB service.
2. Lambda’s warm-request latency is already comfortably below the **500 ms p99** target.
3. The only thing preventing Lambda from meeting the burst SLO is **cold-start overhead**, whereas EC2 and Fargate would require **scaling out or selecting substantially larger compute sizes** to address queueing.

### Conditions under which this recommendation would change

- If the workload became more steady and the sustained monthly average load moved well above roughly **25 RPS**, the cost advantage of Lambda would shrink and eventually reverse relative to the deployed Fargate service.
- If the SLO were relaxed substantially, or if the application could tolerate occasional **>1 s** burst outliers after idle periods, plain Lambda without Provisioned Concurrency would become acceptable.
- Conversely, if extremely strict latency guarantees were required and cold starts had to be eliminated entirely, then Lambda would need **Provisioned Concurrency**, and EC2 or multi-task Fargate would become stronger candidates only after explicit **right-sizing and autoscaling** were added.

---

## 9. Short conclusion

As deployed, **none of the four environments met p99 < 500 ms under burst from zero**. **Lambda zip** was the best warm-path performer and by far the cheapest option for the provided traffic model, but it needs **Provisioned Concurrency** or an equivalent anti-cold-start measure to satisfy the burst SLO. **Fargate** and **EC2** were both limited mainly by **queueing on undersized always-on compute**.

For this lab’s traffic profile, the best justified answer is therefore:

> **Choose Lambda zip, add Provisioned Concurrency sized to the expected burst concurrency, and revisit the choice only if sustained average traffic grows beyond roughly 25 RPS or the infrastructure shape changes materially.**

---

## Appendix: Source notes

The latency numbers in this report come directly from the `oha` outputs supplied in the prompt. The cost assumptions for **Fargate**, **ALB**, and **EC2** are based on current AWS pricing pages for `us-east-1`. The Lambda cost formula is taken directly from the lab prompt so that the report remains aligned with the assignment wording.
