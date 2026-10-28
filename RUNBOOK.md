# Robot Shop Runbook

Namespace for the shop is `robot-shop`; monitoring runs in `monitoring`.

## RobotShopPaymentAvailabilitySLOBreached

**What it means:** no `payment` pod has been Ready for 30 seconds. Customers cannot pay.

**Check:**

    kubectl -n robot-shop get pods -l service=payment
    kubectl -n robot-shop describe deploy/payment
    kubectl -n robot-shop logs deploy/payment --tail=50

**Mitigate first:**

    kubectl -n robot-shop scale deploy/payment --replicas=1
    kubectl -n robot-shop rollout restart deploy/payment

**Then find the cause:** look for `OOMKilled`, image pull errors, or a dependency (RabbitMQ, MySQL) being down.

## RobotShopWebAvailabilitySLOBreached

**What it means:** no `web` pod has been Ready for 30 seconds. The shop front page is down.

**Check:**

    kubectl -n robot-shop get pods -l service=web
    kubectl -n robot-shop logs deploy/web --tail=50

**Mitigate first:**

    kubectl -n robot-shop scale deploy/web --replicas=1
    kubectl -n robot-shop rollout restart deploy/web

## RobotShopHighCrashLoopRate

**What it means:** a pod restarted more than once per minute over 5 minutes.

**Check:**

    kubectl -n robot-shop get pods
    kubectl -n robot-shop describe pod <pod-name>
    kubectl -n robot-shop logs <pod-name> --previous --tail=50

**Look for:** `OOMKilled` (raise the memory limit in the chart), a failing dependency, or a bad image tag. If a recent change caused it, roll it back with `helm rollback robot-shop <revision> -n robot-shop`.
