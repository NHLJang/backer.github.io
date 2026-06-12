# Fix Alert for "No Running Consumer for Queue"

**Jira:** DEVOPS-3683 | **Status:** Done | **Assignee:** Hau Ha | **Reporter:** Dan Steen

---

## Problem Statement

We have an issue where if there are no **idle** consumers for a specific AmazonMQ queue, we get an alert for "no running consumers". We want to fix this to only alert when there are really no consumers — not just no idle consumers.

---

## Root Cause Analysis

### 2025-06-23 — Investigation

By inspecting the Evaluation Graph of the Advantage Prod Analytic queue on Jun 23, the monitoring formula was found to be incorrect. Key findings are listed below.

#### 1. Alerts Were ~15 Minutes Late

Our alerts likely/always fired roughly **15 minutes later** than they should — even if the system had returned to normal status for the past few minutes.

**Examples:**

| Queue | Alert fired | `consumer_count` reported 0 |
|---|---|---|
| Advantage - analytics | 18:19:51 UTC+7 | 18:04:00 UTC+7 |
| Mchire - analytics | 15:39:30 UTC+7 | 15:25:00 UTC+7 |
| Test - send_report | 16:22:31 UTC+7 | 16:08:00 UTC+7 |
| Prod - media-priority | 10:27:45 UTC+7 | 10:13:00 UTC+7 |

When `consumer_count` metrics are reported as `0` while `message_count > 0`, we expect to get notified **immediately**, not after 15 minutes. This leads to false impressions, potentially late reactions, and makes it hard to track down real problems due to the elasticity of our system.

---

#### 2. Unstable Evaluation Graph — Formula Issue

Looking at the evaluation graphs, we rarely see a flat and stable graph even when `consumer_count` and `message_count` are stable.

The existing formula was:

```
sum(last_15m):(1 - max(consumer_count).as_count()) * max(message_count.as_rate())
```

While it looks correct, according to the Datadog documentation on `as_count()` in monitor evaluations [[1]](#references):

> ```
> Monitors involving arithmetic and at least 1 as_count() modifier use a separate
> evaluation path that changes the order in which arithmetic and time aggregation
> are performed.
> ```

> ```
> Aggregation methods other than sum do not make sense to use with (and cannot be
> used with) .as_count().
> ```

The critical behavior when using `sum()` with `as_count()` is:

> ```
> Aggregation function applied before division
> ```

This means the formula is actually evaluated as:

```
(1 - sum(consumer_count.as_count(), last_15mins)) * sum(message_count.as_rate(), last_15mins)
```

---

#### 3. Math Verification

Taking a 10-minute sliding window as an example, where `consumer_count.as_count()` is always `1` and `message_count.as_rate()` is always `0.83` for the last 15 minutes:

| Data-point | Calculation | Result |
|---|---|---|
| 1st | `(1 - 15) × 0.83 × 15` | ≈ **-174.3** |
| 2nd | `(1 - 14) × 0.83 × 14` | ≈ **-151.06** |
| 3rd | `(1 - 13) × 0.83 × 13` | ≈ **-129.48** |
| … | … | … |
| 10th | `(1 - 6) × 0.83 × 6` | ≈ **24.9** |

This confirms the Datadog claim: aggregation is applied **before** arithmetic.

---

#### 4. Why the 10-Minute Repeating Pattern?

According to the Datadog AWS Integration documentation [[2]](#references):

> ```
> Metric polling: API polling comes out of the box with the AWS integration.
> A metric-by-metric crawl of the CloudWatch API pulls data and sends it to Datadog.
> New metrics are pulled every ten minutes, on average.
> ```

And confirmed in the Datadog AWS Integration and CloudWatch FAQ [[3]](#references):

> ```
> By default, Datadog collects AWS metrics every 10 minutes.
> ```

Given polls at e.g. 12:10 and 12:20, only the 12:20 evaluation will have a full set of 15 data-points. Every minute after, the number of available data-points drops by 1, until at 12:29 we have 9 empty data-points. At 12:30 the full set is available again — explaining the repeating 10-minute graph shape.

---

#### 5. Why Exactly 14–15 Minutes Late?

For example: if `consumer_count` drops to `0` at 12:14:

- Only at **12:29** will `(1 - sum(consumer_count.as_count(), last_15mins)) < 0` be true
- This is because it's computed from 6 zero-value data-points (12:15–12:20) plus 9 missing/unreported data-points (12:21–12:29)

This also means the alert **will not fire at all** if:

```
(time_out_in_minutes + 9) < 15
```

---

## Fix & Timeline

### Proposed Fix (2025-06-23)

Change the formula to use `as_rate()`, multiply by `60` to recover the original metric value, and watch the `max()` over the last 10 minutes — aligning with the AWS CloudWatch polling interval. This allows immediate detection when a worker pod truly loses its connection with the broker unexpectedly.

---

### Update (2025-07-04)

The new formula produced a significantly higher number of alerts, likely due to pod rebalancing and spot instance turnover.

**Remediation:** Alert only when the maximum of `aws.consumer_count.minimum` over the last **3 minutes / samples is `0`**, meaning no consumer count for over 2 minutes (3 samples = 120 seconds). We assume new workers spin up within this timeframe.

Additionally, `aws.consumer_count` was found to be an **average value per one-minute window** from CloudWatch. It hints at multiples-of-3 sub-samples per minute interval, as the metric occasionally reports an intriguing value of `0.33`.

---

### Update (2025-07-07)

Datadog's `rollup()` function uses **look-forward aggregation**, bucketing samples from a fixed Unix epoch start time. Since we need **look-backward sampling**, the formula was switched to `moving_rollup()`.

---

## References

1. [as_count() in Monitor Evaluations — Datadog Docs](https://docs.datadoghq.com/monitors/guide/as-count-in-monitor-evaluations/)
2. [Amazon Web Services Integration — Datadog Docs](https://docs.datadoghq.com/integrations/amazon_web_services/)
3. [AWS Integration and CloudWatch FAQ — Datadog Docs](https://docs.datadoghq.com/integrations/guide/aws-integration-and-cloudwatch-faq/#how-can-i-reduce-the-delay-of-receiving-my-cloudwatch-metrics-to-datadog)
