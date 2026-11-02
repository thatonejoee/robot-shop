# Postmortem: Payment service outage (planned drill 1)

**Date:** 2026-11-02
**Author:** Joseph
**Status:** Complete
**Severity:** Drill (no real customers affected)

## Summary

The `payment` deployment was deliberately scaled to zero replicas to test detection, alerting and recovery. The alert fired, a Slack notification arrived with the alert name, and service was restored by scaling back up. This was a planned exercise, so the root cause below is the injected fault, and the useful findings are about detection, notification and the gaps in our monitoring.

## Impact

- Payment was unavailable for **7 minutes 23 seconds** (21:11:09 to 21:18:32 WAT), plus the few seconds the new pod needed to become Ready.
- Error budget used: 443 s of 12,960 s allowed per 30 days (99.5% SLO), about **3.4%**.
- No real users were affected.

## Timeline (WAT)

| Time | Event | How it was recorded |
|---|---|---|
| 21:11:09 | Drill start: `payment` scaled to 0 replicas | Command timestamp |
| 21:12:16 | Alert `RobotShopPaymentAvailabilitySLOBreached` became Pending | Prometheus `activeAt` (20:12:16Z) |
| about 21:12:46 | Alert Firing, 30 s after Pending per the rule's `for: 30s` | Calculated, not measured |
| about 21:13 | Slack `[FIRING:1]` message received | Read from Slack |
| 21:18:32 | Mitigation: `payment` scaled back to 1 replica | Command timestamp |
| about 21:22 | Slack `[RESOLVED]` message received | Read from Slack |

## Detection

- Prometheus noticed the outage **67 seconds** after it began, and Slack was notified about **2 minutes** after it began.
- The resolved notice arrived about 3.5 minutes after mitigation. Alertmanager is configured with a 5-minute `group_interval`, which delays follow-up notifications.

## Mitigation

Mitigation was scaling `payment` back to one replica, following the first step of the runbook. The 7 minutes between the alert firing and mitigation was not response time: it was spent deliberately capturing evidence (screenshots and Prometheus timings) before restoring service. A real incident would have been mitigated much sooner after the alert.

## Root cause

An injected fault: `kubectl scale deploy/payment --replicas=0`. The alert condition (no Ready `payment` pod for 30 seconds) is detected by Kubernetes readiness metrics from `kube-state-metrics`.

## What went well

- The alert fired quickly and reached Slack without manual steps.
- Only the payment alert fired. The web alert correctly stayed quiet.
- Mitigation was a single command, and recovery was confirmed by a resolved notice.

## What went poorly

- The first Slack message showed only the alert name and labels. It had no description and **no runbook link**, even though the alert rule defined one.
- The SLI measures pod readiness, not whether requests succeed or how long they take, so a Ready pod returning errors would not be detected.
- The dashboard has no request-rate, error-rate or latency panels, because Robot Shop does not export request metrics. Its "latency" panel shows CPU.
- Prometheus has no persistent storage and keeps only 6 hours, so history is lost when it restarts.

## Action items

| Action | Status |
|---|---|
| Add description and runbook link to the Slack message | Done (verified in drill 2, 21:38 WAT) |
| Add request-level metrics (for example a blackbox probe) so the SLI measures request success and latency | Open |
| Rename or replace the dashboard panel labelled "latency" | Open |
| Add persistent storage and longer retention for Prometheus | Open |
| Investigate why the `load` generator pod restarts repeatedly | Open |
