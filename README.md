# Linux Security Operations Lab

A production-style Linux security operations lab built on Ubuntu Server virtual machines and managed with Ansible.

The goal of this project is to demonstrate practical Linux administration, infrastructure automation, security hardening, service deployment, monitoring, alerting, centralized logging, and validation through documented tests.

## Current Features

- Ubuntu Server VM deployed in VMware Workstation Pro
- Dedicated Ansible automation user
- SSH key-based Ansible access
- UFW firewall with default-deny inbound policy
- Fail2ban SSH brute-force protection
- Docker installed through Ansible
- Nginx container deployed through Ansible
- HTTP access allowed through UFW
- Prometheus deployed through Docker Compose
- Grafana deployed through Docker Compose
- node_exporter deployed for Linux host metrics
- Prometheus datasource provisioned automatically in Grafana
- Linux Node Overview dashboard provisioned automatically in Grafana
- Prometheus alert rules deployed automatically
- NodeExporterDown alert validated through simulated outage
- Elasticsearch deployed through Docker Compose
- Kibana deployed through Docker Compose
- Filebeat deployed for centralized log collection
- Separate Elasticsearch index patterns for auth, Fail2ban, Nginx, and Docker logs
- Kibana Data Views created for switching between log categories
- Monitoring and logging ports allowed through UFW
- Manual validation documented with screenshots and test notes
- Ansible idempotence verified with repeated playbook runs

## Architecture

```txt
Windows Host
├── WSL Ubuntu
│   └── Ansible control node
└── VMware Workstation Pro
    └── vm-app-01
        ├── Ubuntu Server
        ├── UFW
        ├── Fail2ban
        ├── Docker
        ├── Nginx container
        ├── Monitoring stack
        │   ├── Prometheus
        │   │   └── Alert rules
        │   ├── Grafana
        │   │   ├── Prometheus datasource
        │   │   └── Linux Node Overview dashboard
        │   └── node_exporter
        └── Logging stack
            ├── Elasticsearch
            ├── Kibana
            └── Filebeat
```

## Ansible Roles

| Role | Purpose |
|---|---|
| `common` | Installs baseline system packages |
| `firewall` | Configures UFW firewall rules |
| `fail2ban` | Configures SSH brute-force protection |
| `docker` | Installs and enables Docker |
| `nginx_container` | Deploys a containerized Nginx service |
| `monitoring_stack` | Deploys Prometheus, Grafana, node_exporter, Grafana provisioning, and Prometheus alert rules |
| `logging_stack` | Deploys Elasticsearch, Kibana, and Filebeat for centralized logging |

## Usage

Copy the example inventory:

```bash
cp ansible/inventory.example.ini ansible/inventory.ini
```

Edit `ansible/inventory.ini` and set your VM IP address, SSH user, and private key path.

Run the playbook:

```bash
ansible-playbook -i ansible/inventory.ini ansible/site.yml
```

Run the playbook again to verify idempotence:

```bash
ansible-playbook -i ansible/inventory.ini ansible/site.yml
```

A clean second run should report no unnecessary changes.

## Exposed Services

| Service | Port | URL |
|---|---:|---|
| Nginx | 80 | `http://<VM_IP_ADDRESS>` |
| Prometheus | 9090 | `http://<VM_IP_ADDRESS>:9090` |
| Grafana | 3000 | `http://<VM_IP_ADDRESS>:3000` |
| Elasticsearch | 9200 | `http://<VM_IP_ADDRESS>:9200` |
| Kibana | 5601 | `http://<VM_IP_ADDRESS>:5601` |

## Validation

### Fail2ban SSH ban test

Fail2ban was tested against repeated failed SSH login attempts.

```bash
sudo fail2ban-client status sshd
```

Expected result:

```txt
Currently banned: 1
Banned IP list: <HOST_PRIVATE_IP>
```

### Nginx container test

The Nginx container deployment was validated with:

```bash
curl http://<VM_IP_ADDRESS>
```

Expected response:

```html
<h1>Linux Security Operations Lab</h1>
<p>Nginx container deployed with Ansible.</p>
<p>Baseline security: UFW + fail2ban.</p>
```

### Prometheus readiness test

Prometheus readiness was validated with:

```bash
curl http://<VM_IP_ADDRESS>:9090/-/ready
```

Expected response:

```txt
Prometheus Server is Ready.
```

### Prometheus target health

Prometheus target health was checked in the web UI:

```txt
http://<VM_IP_ADDRESS>:9090
Status -> Target health
```

Expected targets:

```txt
prometheus      UP
node_exporter   UP
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
http://<VM_IP_ADDRESS>:9090/alerts
```

### NodeExporterDown alert test

The `NodeExporterDown` alert was validated by intentionally stopping the `lab-node-exporter` container.

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

Elasticsearch was validated with:

```bash
curl http://<VM_IP_ADDRESS>:9200
```

Expected result:

```txt
cluster_name: docker-cluster
tagline: You Know, for Search
```

### Kibana Data Views

Kibana Data Views were created for switching between centralized log categories:

| Data View | Index pattern |
|---|---|
| Linux Auth Logs | `logs-linux-auth-*` |
| Fail2ban Events | `logs-fail2ban-*` |
| Nginx Access Logs | `logs-nginx-access-*` |
| Docker Container Logs | `logs-docker-*` |
| All Lab Logs | `logs-*` |

Logging indices can be checked with:

```bash
curl "http://<VM_IP_ADDRESS>:9200/_cat/indices/logs-*?v"
```

## Documentation

- [Security hardening](docs/security-hardening.md)
- [Fail2ban SSH ban test](docs/fail2ban-ssh-ban-test.md)
- [Nginx container deployment test](docs/validation/nginx-container-deployment.md)
- [Monitoring stack deployment test](docs/monitoring-stack-deployment.md)
- [Prometheus alert rules validation](docs/validation/prometheus-alert-rules.md)
- [NodeExporterDown alert test](docs/validation/node-exporter-down-alert-test.md)
- [Elastic Stack logging Data Views](docs/validation/elastic-logging-data-views.md)

## Project Status

Current milestone:

- Security baseline automated with Ansible
- Dockerized Nginx service deployed
- Prometheus, Grafana, and node_exporter deployed
- Grafana Prometheus datasource provisioned automatically
- Grafana Linux Node Overview dashboard provisioned automatically
- Prometheus alert rules deployed automatically
- NodeExporterDown alert validated through simulated outage
- Elasticsearch, Kibana, and Filebeat deployed
- Centralized logging configured for auth, Fail2ban, Nginx, and Docker logs
- Kibana Data Views created for separate log categories
- UFW rules managed for SSH, HTTP, Prometheus, Grafana, Elasticsearch, and Kibana
- Validation documented with screenshots
- Idempotence verified with repeated Ansible runs

Next planned milestone:

- Kibana dashboard provisioning
- Structured parsing for Nginx access logs
- Elasticsearch ingest pipelines
- Alertmanager integration
- Custom IDS integration for flow logs and live traffic analysis
