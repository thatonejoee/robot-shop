# SLO and Error Budget Policy: Payment Service

**Service:** `payment` (namespace `robot-shop`)
**Owner:** Joseph Fatoke
**Window:** rolling 30 days

## SLI (what we measure)

The fraction of time at least one `payment` pod is Ready, taken from the Kubernetes metric `kube_pod_status_ready` that Prometheus scrapes.

    max(kube_pod_status_ready{namespace="robot-shop",condition="true",pod=~"payment-.*"}) or vector(0)

**Limitation:** this measures whether the service is available, not whether each request succeeds or how fast it answers. Robot Shop does not export request-level metrics, so a pod that is Ready but returning errors would not be detected. Prometheus here keeps only 6 hours of data, so the 30-day figure is tracked from recorded downtime, not computed from the query.

## SLO (the target)

99.5% availability over 30 days.

## Error budget

| Item | Value |
|---|---|
| Window | 30 days = 43,200 minutes |
| Allowed failure | 0.5% = **216 minutes** |
| Budget used in drill 1 | 7 min 23 s (about 3.4%) |
| Budget remaining after drill 1 | about 208 minutes |

Drill 2 (21:38, testing the Slack message format) was a short second outage and is not counted here.

## Alerting

`RobotShopPaymentAvailabilitySLOBreached` fires when no payment pod is Ready for 30 seconds. It is sent to Slack `#alerts` with a link to `RUNBOOK.md`.

## Error budget policy

| Budget remaining | What we do |
|---|---|
| More than 50% | Normal releases |
| 50% down to 10% | Only low-risk changes. Review each incident before the next release |
| Under 10% or exhausted | Freeze feature changes. Work only on reliability until the window recovers |

After any outage that uses more than 10% of the budget, write a blameless postmortem.
