# Multi-node Observability Split

## Goal

Refactor the lab from a single-node observability setup into a multi-node architecture where the application node runs application services and lightweight agents, while the observability node runs monitoring and logging backend services.

## Environment

- Host: Windows machine running VMware Workstation Pro
- Control node: WSL Ubuntu with Ansible
- Application node: `vm-app-01`
- Observability node: `vm-observability-01`
- Deployment method: Ansible-managed Docker containers

## Architecture

```txt
Windows Host
├── WSL Ubuntu
│   └── Ansible control node
└── VMware Workstation Pro
    ├── vm-app-01
    │   ├── Ubuntu Server
    │   ├── UFW
    │   ├── Fail2ban
    │   ├── Docker
    │   ├── Nginx
    │   ├── node_exporter
    │   └── Filebeat
    │
    └── vm-observability-01
        ├── Ubuntu Server
        ├── UFW
        ├── Fail2ban
        ├── Docker
        ├── Prometheus
        ├── Grafana
        ├── Elasticsearch
        └── Kibana
```

## Service Placement

After the split, services are distributed as follows:

| Node | Services |
|---|---|
| `vm-app-01` | Nginx, node_exporter, Filebeat |
| `vm-observability-01` | Prometheus, Grafana, Elasticsearch, Kibana |

## Ansible Role Layout

The playbook is split by host group:

| Host group | Roles |
|---|---|
| `all` | `common`, `firewall`, `fail2ban`, `docker` |
| `app` | `app_observability_cleanup`, `nginx_container`, `node_exporter_agent`, `filebeat_agent` |
| `observability` | `observability_cleanup`, `monitoring_stack`, `logging_stack` |

## Data Flow

```txt
Nginx traffic
    ↓
vm-app-01 / Nginx access logs
    ↓
Filebeat on vm-app-01
    ↓
Elasticsearch on vm-observability-01
    ↓
Kibana Data Views

Host metrics
    ↓
node_exporter on vm-app-01
    ↓
Prometheus on vm-observability-01
    ↓
Grafana dashboards and Prometheus alert rules
```

## Monitoring Changes

Prometheus now runs on `vm-observability-01` and scrapes `node_exporter` on the application node.

Expected Prometheus target:

```txt
<APP_VM_IP_ADDRESS>:9100
```

The old single-node target was removed:

```txt
node-exporter:9100
```

## Logging Changes

Filebeat now runs on `vm-app-01` and sends logs to Elasticsearch on `vm-observability-01`.

Expected Elasticsearch output:

```txt
http://<OBSERVABILITY_VM_IP_ADDRESS>:9200
```

Filebeat routes log sources into separate Elasticsearch index patterns:

| Log source | Index pattern |
|---|---|
| Linux authentication logs | `logs-linux-auth-*` |
| Fail2ban logs | `logs-fail2ban-*` |
| Nginx access logs | `logs-nginx-access-*` |
| Docker container logs | `logs-docker-*` |

Kibana Data Views were created through the Kibana API for the following views:

| Data View | Index pattern |
|---|---|
| Linux Auth Logs | `logs-linux-auth-*` |
| Fail2ban Events | `logs-fail2ban-*` |
| Nginx Access Logs | `logs-nginx-access-*` |
| Docker Container Logs | `logs-docker-*` |
| All Lab Logs | `logs-*` |

## Validation Commands

Verify application node containers:

```bash
ansible app -i ansible/inventory.ini -m command -a "docker ps" -b
```

Expected application node containers:

```txt
lab-nginx
lab-node-exporter
lab-filebeat
```

Verify observability node containers:

```bash
ansible observability -i ansible/inventory.ini -m command -a "docker ps" -b
```

Expected observability node containers:

```txt
lab-prometheus
lab-grafana
lab-elasticsearch
lab-kibana
```

Validate Nginx on the application node:

```bash
curl http://<APP_VM_IP_ADDRESS>
```

Validate Prometheus readiness on the observability node:

```bash
curl http://<OBSERVABILITY_VM_IP_ADDRESS>:9090/-/ready
```

Validate Elasticsearch on the observability node:

```bash
curl http://<OBSERVABILITY_VM_IP_ADDRESS>:9200
```

Validate Elasticsearch logging indices:

```bash
curl "http://<OBSERVABILITY_VM_IP_ADDRESS>:9200/_cat/indices/logs-*?v"
```

Validate Prometheus target health in the web UI:

```txt
http://<OBSERVABILITY_VM_IP_ADDRESS>:9090/targets
```

Expected target:

```txt
node_exporter UP
instance="<APP_VM_IP_ADDRESS>:9100"
```

## Kibana Data View API Example

The Kibana Data Views can be recreated through the Kibana API.

Example:

```bash
curl -X POST "http://<OBSERVABILITY_VM_IP_ADDRESS>:5601/api/data_views/data_view" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "data_view": {
      "title": "logs-nginx-access-*",
      "name": "Nginx Access Logs",
      "timeFieldName": "@timestamp"
    }
  }'
```

## Result

The multi-node observability split was completed successfully.

The application node now runs only the application service and lightweight agents, while the observability node runs the monitoring and logging backends.

## Notes

This milestone separates application workloads from observability workloads and better reflects a production-style architecture.

Future improvements could include Alertmanager integration, Kibana dashboard provisioning, Elasticsearch ingest pipelines, and structured parsing for Nginx access logs.
EOFcat > docs/validation/multi-node-observability-split.md <<'EOF'
# Multi-node Observability Split

## Goal

Refactor the lab from a single-node observability setup into a multi-node architecture where the application node runs application services and lightweight agents, while the observability node runs monitoring and logging backend services.

## Environment

- Host: Windows machine running VMware Workstation Pro
- Control node: WSL Ubuntu with Ansible
- Application node: `vm-app-01`
- Observability node: `vm-observability-01`
- Deployment method: Ansible-managed Docker containers

## Architecture

```txt
Windows Host
├── WSL Ubuntu
│   └── Ansible control node
└── VMware Workstation Pro
    ├── vm-app-01
    │   ├── Ubuntu Server
    │   ├── UFW
    │   ├── Fail2ban
    │   ├── Docker
    │   ├── Nginx
    │   ├── node_exporter
    │   └── Filebeat
    │
    └── vm-observability-01
        ├── Ubuntu Server
        ├── UFW
        ├── Fail2ban
        ├── Docker
        ├── Prometheus
        ├── Grafana
        ├── Elasticsearch
        └── Kibana
```

## Service Placement

After the split, services are distributed as follows:

| Node | Services |
|---|---|
| `vm-app-01` | Nginx, node_exporter, Filebeat |
| `vm-observability-01` | Prometheus, Grafana, Elasticsearch, Kibana |

## Ansible Role Layout

The playbook is split by host group:

| Host group | Roles |
|---|---|
| `all` | `common`, `firewall`, `fail2ban`, `docker` |
| `app` | `app_observability_cleanup`, `nginx_container`, `node_exporter_agent`, `filebeat_agent` |
| `observability` | `observability_cleanup`, `monitoring_stack`, `logging_stack` |

## Data Flow

```txt
Nginx traffic
    ↓
vm-app-01 / Nginx access logs
    ↓
Filebeat on vm-app-01
    ↓
Elasticsearch on vm-observability-01
    ↓
Kibana Data Views

Host metrics
    ↓
node_exporter on vm-app-01
    ↓
Prometheus on vm-observability-01
    ↓
Grafana dashboards and Prometheus alert rules
```

## Monitoring Changes

Prometheus now runs on `vm-observability-01` and scrapes `node_exporter` on the application node.

Expected Prometheus target:

```txt
<APP_VM_IP_ADDRESS>:9100
```

The old single-node target was removed:

```txt
node-exporter:9100
```

## Logging Changes

Filebeat now runs on `vm-app-01` and sends logs to Elasticsearch on `vm-observability-01`.

Expected Elasticsearch output:

```txt
http://<OBSERVABILITY_VM_IP_ADDRESS>:9200
```

Filebeat routes log sources into separate Elasticsearch index patterns:

| Log source | Index pattern |
|---|---|
| Linux authentication logs | `logs-linux-auth-*` |
| Fail2ban logs | `logs-fail2ban-*` |
| Nginx access logs | `logs-nginx-access-*` |
| Docker container logs | `logs-docker-*` |

Kibana Data Views were created through the Kibana API for the following views:

| Data View | Index pattern |
|---|---|
| Linux Auth Logs | `logs-linux-auth-*` |
| Fail2ban Events | `logs-fail2ban-*` |
| Nginx Access Logs | `logs-nginx-access-*` |
| Docker Container Logs | `logs-docker-*` |
| All Lab Logs | `logs-*` |

## Validation Commands

Verify application node containers:

```bash
ansible app -i ansible/inventory.ini -m command -a "docker ps" -b
```

Expected application node containers:

```txt
lab-nginx
lab-node-exporter
lab-filebeat
```

Verify observability node containers:

```bash
ansible observability -i ansible/inventory.ini -m command -a "docker ps" -b
```

Expected observability node containers:

```txt
lab-prometheus
lab-grafana
lab-elasticsearch
lab-kibana
```

Validate Nginx on the application node:

```bash
curl http://<APP_VM_IP_ADDRESS>
```

Validate Prometheus readiness on the observability node:

```bash
curl http://<OBSERVABILITY_VM_IP_ADDRESS>:9090/-/ready
```

Validate Elasticsearch on the observability node:

```bash
curl http://<OBSERVABILITY_VM_IP_ADDRESS>:9200
```

Validate Elasticsearch logging indices:

```bash
curl "http://<OBSERVABILITY_VM_IP_ADDRESS>:9200/_cat/indices/logs-*?v"
```

Validate Prometheus target health in the web UI:

```txt
http://<OBSERVABILITY_VM_IP_ADDRESS>:9090/targets
```

Expected target:

```txt
node_exporter UP
instance="<APP_VM_IP_ADDRESS>:9100"
```

## Kibana Data View API Example

The Kibana Data Views can be recreated through the Kibana API.

Example:

```bash
curl -X POST "http://<OBSERVABILITY_VM_IP_ADDRESS>:5601/api/data_views/data_view" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "data_view": {
      "title": "logs-nginx-access-*",
      "name": "Nginx Access Logs",
      "timeFieldName": "@timestamp"
    }
  }'
```

## Result

The multi-node observability split was completed successfully.

The application node now runs only the application service and lightweight agents, while the observability node runs the monitoring and logging backends.

## Notes

This milestone separates application workloads from observability workloads and better reflects a production-style architecture.

Future improvements could include Alertmanager integration, Kibana dashboard provisioning, Elasticsearch ingest pipelines, and structured parsing for Nginx access logs.
