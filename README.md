# Linux Security Operations Lab

A production-style Linux security operations lab built on Ubuntu Server virtual machines and managed with Ansible.

The goal of this project is to demonstrate practical Linux administration, infrastructure automation, security hardening, service deployment, monitoring, alerting, centralized logging, and validation through documented tests.

## Current State

The lab currently runs as a multi-node Linux security operations environment.

- `vm-app-01` runs the application service and lightweight observability agents.
- `vm-observability-01` runs monitoring and logging backend services.
- WSL Ubuntu is used as the Ansible control node.

## Current Features

- Ubuntu Server virtual machines deployed in VMware Workstation Pro
- WSL Ubuntu used as the Ansible control node
- Dedicated `ansible` automation user on managed nodes
- SSH key-based Ansible access
- UFW firewall with default-deny inbound policy
- Fail2ban SSH brute-force protection
- Docker installed through Ansible
- Nginx container deployed on the application node
- node_exporter deployed on the application node
- Filebeat deployed on the application node
- Prometheus deployed on the observability node
- Grafana deployed on the observability node
- Elasticsearch deployed on the observability node
- Kibana deployed on the observability node
- Prometheus datasource provisioned automatically in Grafana
- Linux Node Overview dashboard provisioned automatically in Grafana
- Prometheus alert rules deployed automatically
- NodeExporterDown alert validated through simulated outage
- Separate Elasticsearch index patterns for auth, Fail2ban, Nginx, and Docker logs
- Kibana Data Views created for switching between log categories
- Monitoring and logging workloads split across dedicated nodes
- Manual validation documented with screenshots and test notes
- Ansible idempotence verified with repeated playbook runs

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

## Data Flow

```txt
HTTP traffic
    ↓
Nginx on vm-app-01
    ↓
Nginx access logs
    ↓
Filebeat on vm-app-01
    ↓
Elasticsearch on vm-observability-01
    ↓
Kibana Data Views
```

```txt
Host metrics
    ↓
node_exporter on vm-app-01
    ↓
Prometheus on vm-observability-01
    ↓
Grafana dashboards
    ↓
Prometheus alert rules
```

## Ansible Roles

| Role | Purpose |
|---|---|
| `common` | Installs baseline system packages |
| `firewall` | Configures UFW firewall rules |
| `fail2ban` | Configures SSH brute-force protection |
| `docker` | Installs and enables Docker |
| `nginx_container` | Deploys a containerized Nginx service |
| `node_exporter_agent` | Deploys node_exporter on the application node |
| `filebeat_agent` | Deploys Filebeat on the application node |
| `monitoring_stack` | Deploys Prometheus, Grafana, Grafana provisioning, and Prometheus alert rules |
| `logging_stack` | Deploys Elasticsearch and Kibana |
| `app_observability_cleanup` | Removes old backend containers from the application node |
| `observability_cleanup` | Removes application-side containers from the observability node |

## Inventory Layout

The inventory is split into host groups:

```ini
[app]
vm-app-01 ansible_host=<APP_VM_IP_ADDRESS> ansible_user=ansible ansible_ssh_private_key_file=<PATH_TO_PRIVATE_KEY>

[observability]
vm-observability-01 ansible_host=<OBSERVABILITY_VM_IP_ADDRESS> ansible_user=ansible ansible_ssh_private_key_file=<PATH_TO_PRIVATE_KEY>

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

The local `ansible/inventory.ini` file is ignored by Git. A safe template is provided as:

```txt
ansible/inventory.example.ini
```

## Usage

Copy the example inventory:

```bash
cp ansible/inventory.example.ini ansible/inventory.ini
```

Edit `ansible/inventory.ini` and set your VM IP addresses, SSH user, and private key path.

Run the playbook:

```bash
ansible-playbook -i ansible/inventory.ini ansible/site.yml
```

Run the playbook again to verify idempotence:

```bash
ansible-playbook -i ansible/inventory.ini ansible/site.yml
```

A clean second run should report no unnecessary changes.

## Service Placement

| Service | Host | Port | URL |
|---|---|---:|---|
| Nginx | `vm-app-01` | 80 | `http://<APP_VM_IP_ADDRESS>` |
| node_exporter | `vm-app-01` | 9100 | `http://<APP_VM_IP_ADDRESS>:9100/metrics` |
| Prometheus | `vm-observability-01` | 9090 | `http://<OBSERVABILITY_VM_IP_ADDRESS>:9090` |
| Grafana | `vm-observability-01` | 3000 | `http://<OBSERVABILITY_VM_IP_ADDRESS>:3000` |
| Elasticsearch | `vm-observability-01` | 9200 | `http://<OBSERVABILITY_VM_IP_ADDRESS>:9200` |
| Kibana | `vm-observability-01` | 5601 | `http://<OBSERVABILITY_VM_IP_ADDRESS>:5601` |

## Validation

### Ansible connectivity

Both nodes are managed through Ansible.

```bash
ansible all -i ansible/inventory.ini -m ping
```

Expected result:

```txt
vm-app-01 | SUCCESS
vm-observability-01 | SUCCESS
```

Privilege escalation is validated with:

```bash
ansible all -i ansible/inventory.ini -m command -a "whoami" -b
```

Expected result:

```txt
root
```

### Service placement

Application node validation:

```bash
ansible app -i ansible/inventory.ini -m command -a "docker ps" -b
```

Expected application node containers:

```txt
lab-nginx
lab-node-exporter
lab-filebeat
```

Observability node validation:

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

### Nginx container test

The Nginx container deployment is validated with:

```bash
curl http://<APP_VM_IP_ADDRESS>
```

Expected response:

```html
<h1>Linux Security Operations Lab</h1>
<p>Nginx container deployed with Ansible.</p>
<p>Baseline security: UFW + fail2ban.</p>
```

### Prometheus readiness test

Prometheus readiness is validated on the observability node with:

```bash
curl http://<OBSERVABILITY_VM_IP_ADDRESS>:9090/-/ready
```

Expected response:

```txt
Prometheus Server is Ready.
```

### Prometheus target health

Prometheus target health is checked in the web UI:

```txt
http://<OBSERVABILITY_VM_IP_ADDRESS>:9090/targets
```

Expected target:

```txt
node_exporter UP
instance="<APP_VM_IP_ADDRESS>:9100"
```

### Grafana dashboard provisioning

Grafana is provisioned automatically with a Prometheus datasource and a Linux Node Overview dashboard.

Expected dashboard:

```txt
Dashboards -> Linux Security Operations Lab -> Linux Node Overview
```

The dashboard includes:

- CPU Busy
- Memory Used
- Root Disk Used
- System Uptime
- CPU Usage Over Time
- Memory Usage Over Time

### Prometheus alert rules

Prometheus alert rules are deployed automatically through Ansible.

Configured alert rules:

```txt
NodeExporterDown
HighCpuUsage
HighMemoryUsage
HighRootDiskUsage
```

Alert rules can be checked in the Prometheus web UI:

```txt
http://<OBSERVABILITY_VM_IP_ADDRESS>:9090/alerts
```

### NodeExporterDown alert test

The `NodeExporterDown` alert was validated by intentionally stopping the `lab-node-exporter` container on the application node.

```bash
docker stop lab-node-exporter
```

Expected alert:

```txt
Alert: NodeExporterDown
Expression: up{job="node_exporter"} == 0
Severity: critical
State: PENDING or FIRING
Value: 0
```

The service was restored after the test:

```bash
docker start lab-node-exporter
```

### Elasticsearch readiness test

Elasticsearch is validated on the observability node with:

```bash
curl http://<OBSERVABILITY_VM_IP_ADDRESS>:9200
```

Expected result:

```txt
cluster_name: docker-cluster
tagline: You Know, for Search
```

### Kibana Data Views

Kibana Data Views were created for switching between centralized log categories.

| Data View | Index pattern |
|---|---|
| Linux Auth Logs | `logs-linux-auth-*` |
| Fail2ban Events | `logs-fail2ban-*` |
| Nginx Access Logs | `logs-nginx-access-*` |
| Docker Container Logs | `logs-docker-*` |
| All Lab Logs | `logs-*` |

Logging indices can be checked with:

```bash
curl "http://<OBSERVABILITY_VM_IP_ADDRESS>:9200/_cat/indices/logs-*?v"
```

## Documentation

- [Security hardening](docs/security-hardening.md)
- [Fail2ban SSH ban test](docs/fail2ban-ssh-ban-test.md)
- [Nginx container deployment test](docs/validation/nginx-container-deployment.md)
- [Monitoring stack deployment test](docs/monitoring-stack-deployment.md)
- [Prometheus alert rules validation](docs/validation/prometheus-alert-rules.md)
- [NodeExporterDown alert test](docs/validation/node-exporter-down-alert-test.md)
- [Elastic Stack logging Data Views](docs/validation/elastic-logging-data-views.md)
- [Multi-node Ansible onboarding](docs/validation/multi-node-ansible-onboarding.md)
- [Multi-node observability split](docs/validation/multi-node-observability-split.md)

## License

This repository is published for portfolio and educational review purposes only.

All rights reserved. See [LICENSE](LICENSE).

## Project Status

Current milestone:

- Security baseline automated with Ansible
- Dockerized Nginx service deployed on the application node
- node_exporter deployed on the application node
- Filebeat deployed on the application node
- Prometheus and Grafana deployed on the observability node
- Elasticsearch and Kibana deployed on the observability node
- Grafana Prometheus datasource provisioned automatically
- Grafana Linux Node Overview dashboard provisioned automatically
- Prometheus alert rules deployed automatically
- Centralized logging configured for auth, Fail2ban, Nginx, and Docker logs
- Kibana Data Views created for separate log categories
- Monitoring and logging workloads split across dedicated nodes
- Validation documented with screenshots and test notes
- Idempotence verified with repeated Ansible runs

Next planned milestone:

- Alertmanager integration
- Kibana dashboard provisioning
- Structured parsing for Nginx access logs
- Elasticsearch ingest pipelines
- Custom IDS integration for flow logs and live traffic analysis
