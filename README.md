# Arc Node
Arc is an open Layer-1 blockchain purpose-built for stablecoin finance that delivers the performance, reliability, and liquidity needed to meet global financial demands.



## What it monitors

| Check | What it detects |
|---|---|
| **Process health** | `arc-node-execution` and `arc-node-consensus` are running |
| **IPC sockets** | `/run/arc/reth.ipc` and `/run/arc/auth.ipc` exist |
| **RPC reachability** | EL JSON-RPC responds on port 8545 |
| **Block progression** | Block height advances between checks (detects stuck nodes) |
| **Prometheus metrics** | EL (:9001) and CL (:29000) metrics endpoints are scrapeable |
| **Disk space** | Data dir usage warning at 80%, critical at 90% |

## Alert channels

- **Slack** — via incoming webhook
- **PagerDuty** — via Events API v2
- **Log file** — always written to `/var/log/arc-monitor/monitor.log`

Alerts include **cooldown** (default 5 min) to avoid spam, and auto-send **recovery** notifications.

---

## Quick start

### 1. Local (bare metal / VM)

```bash
cp config/config.env.template /etc/arc-monitor/config.env
# Edit /etc/arc-monitor/config.env with your endpoints and webhook URLs

chmod +x scripts/monitor.sh
./scripts/monitor.sh
```

### 2. Docker

```bash
# Copy and edit config
cp config/config.env.template .env

# Build
docker build -t arc-node-monitor .

# Run (mounts host network to reach node on localhost)
docker run -d \
  --name arc-node-monitor \
  --env-file .env \
  --network host \
  -v arc-monitor-logs:/var/log/arc-monitor \
  -v arc-monitor-state:/var/lib/arc-monitor \
  arc-node-monitor
```

### 3. Docker Compose

```bash
cp config/config.env.template .env
# Set SLACK_WEBHOOK_URL and PAGERDUTY_ROUTING_KEY in .env

docker compose up -d
docker compose logs -f
```

### 4. Kubernetes

```bash
# 1. Create secrets
kubectl create secret generic arc-monitor-secrets \
  --from-literal=SLACK_WEBHOOK_URL=https://hooks.slack.com/services/... \
  --from-literal=PAGERDUTY_ROUTING_KEY=your-key \
  -n arc-ops

# 2. Update image name in k8s/manifests.yaml
# 3. Apply
kubectl apply -f k8s/manifests.yaml

# 4. Watch logs
kubectl logs -n arc-ops deploy/arc-node-monitor -f
```

Use the **Deployment** for continuous monitoring (60s loop).
Use the **CronJob** if you prefer Kubernetes-native scheduling (every 2 min).

---

## Configuration reference

| Variable | Default | Description |
|---|---|---|
| `ARC_RPC_URL` | `http://localhost:8545` | EL JSON-RPC endpoint |
| `ARC_EL_METRICS_URL` | `http://localhost:9001/metrics` | EL Prometheus metrics |
| `ARC_CL_METRICS_URL` | `http://localhost:29000/metrics` | CL Prometheus metrics |
| `SLACK_WEBHOOK_URL` | _(empty)_ | Slack incoming webhook |
| `PAGERDUTY_ROUTING_KEY` | _(empty)_ | PagerDuty routing key |
| `MONITOR_INTERVAL` | `60` | Seconds between checks (Docker mode) |
| `BLOCK_STUCK_THRESHOLD` | `10` | Consecutive stuck checks before critical alert |
| `ALERT_COOLDOWN_SECONDS` | `300` | Seconds between repeat alerts for same issue |
| `DISK_WARN_PCT` | `80` | Disk usage % for warning alert |
| `DISK_CRIT_PCT` | `90` | Disk usage % for critical alert |
| `DEBUG` | `0` | Set to `1` for verbose logging |

---

## Project structure

```
arc-node-monitor/
├── scripts/
│   ├── monitor.sh       # Main monitoring logic (all checks + alerting)
│   └── entrypoint.sh    # Docker loop entrypoint
├── k8s/
│   └── manifests.yaml   # Namespace, ConfigMap, Secret, PVC, Deployment, CronJob
├── config/
│   └── config.env.template
├── docker-compose.yml
├── Dockerfile
└── README.md
```

## Arc node endpoints (reference)

From the [Arc node docs](https://docs.arc.network/arc/tutorials/run-an-arc-node):

| Endpoint | Port | Purpose |
|---|---|---|
| EL JSON-RPC | 8545 | `eth_blockNumber`, `eth_syncing` |
| EL Prometheus | 9001 | Execution layer metrics |
| CL RPC | 31000 | Consensus layer status |
| CL Prometheus | 29000 | Consensus layer metrics |
| EL IPC | `/run/arc/reth.ipc` | Local IPC socket |
| Auth IPC | `/run/arc/auth.ipc` | EL↔CL auth socket |
