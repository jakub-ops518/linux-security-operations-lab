# Monitoring Stack Deployment Test

## Goal

Verify that the monitoring stack is deployed with Ansible and that Linux host metrics are collected successfully.

## Environment

- Host: Windows machine running VMware Workstation Pro
- Control node: WSL Ubuntu with Ansible
- Managed node: Ubuntu Server `vm-app-01`
- Monitoring stack: Prometheus, Grafana, node_exporter
- Deployment method: Docker Compose managed by Ansible

## Components

- Prometheus: metrics collection and query engine
- Grafana: dashboard and visualization interface
- node_exporter: Linux host metrics exporter

## Test Procedure

1. Deployed the monitoring stack using the Ansible `monitoring_stack` role.
2. Started Prometheus, Grafana, and node_exporter containers using Docker Compose.
3. Allowed TCP ports `9090` and `3000` through UFW.
4. Verified that the containers were running with:

```bash
docker ps
```

5. Verified Prometheus readiness with:

```bash
curl http://<VM_IP_ADDRESS>:9090/-/ready
```

6. Checked Prometheus target health in the web UI:

```txt
http://<VM_IP_ADDRESS>:9090
Status -> Target health
```

7. Checked Grafana availability in the browser:

```txt
http://<VM_IP_ADDRESS>:3000
```

## Result

The monitoring stack deployed successfully.

Prometheus reported the configured targets as healthy, including the `node_exporter` target for Linux host metrics.

Expected Prometheus readiness response:

```txt
Prometheus Server is Ready.
```

Expected Prometheus targets:

```txt
prometheus      UP
node_exporter   UP
```

## Evidence

Screenshots:

![Prometheus targets](doscreenshots/prometheus-targets.png)

![Grafana dashboard](docs/screenshots/grafana-node-dashboard.png)

## Notes

This validates that the lab can collect host-level metrics and expose a dashboard-ready monitoring stack through the same Ansible workflow used for baseline security and service deployment.

At this stage, Grafana is deployed and reachable. A dedicated Node Exporter dashboard can be added as a follow-up improvement.