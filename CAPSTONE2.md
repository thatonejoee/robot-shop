# Capstone 2: Keep It Running (Stan's Robot Shop)

Forked from `instana/robot-shop` and run on the local `kind` cluster from Capstone 1 (my AWS account blocks EKS, so the cluster is local).

## Status

| Step | Status |
|---|---|
| 1. Deploy and watch (Robot Shop + load generator) | In progress |
| 2. Metrics: kube-prometheus-stack + one dashboard | Not started |
| 3. Logs: Fluent Bit to Loki | Not started |
| 4. One trace in Jaeger | Not started |
| 5. SLO, error budget policy, Slack alert | Not started |
| 6. Incident drill and postmortem | Not started |

## Step 1: Deploy Robot Shop

```bash
kubectl create namespace robot-shop
helm install robot-shop K8s/helm --namespace robot-shop
kubectl -n robot-shop apply -f K8s/load-deployment.yaml
```

### Changes I made to the chart

- **Removed the Instana agent settings** (`INSTANA_AGENT_HOST`) from the cart, catalogue, dispatch, payment and user deployments. The web deployment's Instana settings only render when an EUM key is set, and I left it unset.
- **Raised memory limits** for MySQL (1 GiB to 2 GiB) and RabbitMQ (512 MiB to 1 GiB).

### Problems I hit and how I fixed them

- **MySQL and RabbitMQ were killed within seconds of starting (`OOMKilled`, exit code 137)** even after raising their limits. The node's container runtime was giving every container an open-files limit of about 2 billion, which makes these programs allocate far more memory than their limits allow. Fix: I set `LimitNOFILE=1048576` for `containerd` inside the kind node and recreated the pods. This lives in the node, not in the chart, so it has to be repeated if the cluster is recreated.
- **Images stuck in `ContainerCreating` / `ImagePullBackOff`** because the node could not download them on my connection. Fix: I pulled each image on my machine and copied it into the cluster with `kind load`.
- **The cluster hung** (API timeouts) while many images unpacked at once. Fix: restarting WSL and the node container.
