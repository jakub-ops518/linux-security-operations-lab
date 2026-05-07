# Linux Security Operations Lab

A production-style Linux security operations lab built on Ubuntu Server virtual machines and managed with Ansible.

The goal of this project is to demonstrate practical Linux administration, infrastructure automation, security hardening, service deployment, monitoring, alerting, centralized logging, multi-site web hosting, and validation through documented tests.

## Current State

The lab currently runs as a multi-node Linux security operations environment.

- `vm-app-01` runs application services and lightweight observability agents.
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
- Baseline Nginx container deployed on the application node
- Multi-site PHP Apache hosting deployed on the application node
- Nginx reverse proxy for PHP Apache backend containers
- Two separated PHP sites: `site-alpha` and `site-beta`
- Two PHP Apache backend containers per site
- Per-site Linux users and groups for ownership separation
- Per-site document roots under `/srv/www`
- Request ID and visitor ID correlation for web requests
- Per-site access and error log separation
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
- Separate Elasticsearch index patterns for auth, Fail2ban, Nginx, Docker, and site logs
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
    │   ├── Baseline Nginx
    │   │   └── lab-nginx
    │   ├── Multi-site PHP hosting
    │   │   ├── lab-multisite-nginx
    │   │   ├── lab-site-alpha-php-01
    │   │   ├── lab-site-alpha-php-02
    │   │   ├── lab-site-beta-php-01
    │   │   └── lab-site-beta-php-02
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
Nginx / PHP hosting on vm-app-01
    ↓
Access and error logs
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
| `nginx_container` | Deploys the baseline Nginx container |
| `multi_site_php_hosting` | Deploys multi-site PHP Apache hosting behind an Nginx reverse proxy |
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
| Baseline Nginx | `vm-app-01` | 80 | `http://<APP_VM_IP_ADDRESS>` |
| Multi-site PHP hosting | `vm-app-01` | 8080 | `http://<APP_VM_IP_ADDRESS>:8080` |
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
lab-multisite-nginx
lab-site-alpha-php-01
lab-site-alpha-php-02
lab-site-beta-php-01
lab-site-beta-php-02
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

### Baseline Nginx test

The baseline Nginx container is validated with:

```bash
curl http://<APP_VM_IP_ADDRESS>
```

### Multi-site PHP hosting test

Site Alpha is validated with:

```bash
curl -i http://<APP_VM_IP_ADDRESS>:8080/site-alpha/
```

Site Beta is validated with:

```bash
curl -i http://<APP_VM_IP_ADDRESS>:8080/site-beta/
```

The reverse proxy health endpoint is validated with:

```bash
curl http://<APP_VM_IP_ADDRESS>:8080/health
```

### PHP backend load balancing

Site Alpha load balancing:

```bash
for i in {1..20}; do
  curl -s http://<APP_VM_IP_ADDRESS>:8080/site-alpha/ | grep -o "site-alpha-php-[0-9][0-9]"
done
```

Site Beta load balancing:

```bash
for i in {1..20}; do
  curl -s http://<APP_VM_IP_ADDRESS>:8080/site-beta/ | grep -o "site-beta-php-[0-9][0-9]"
done
```

Expected result:

```txt
site-alpha-php-01
site-alpha-php-02
site-beta-php-01
site-beta-php-02
```

### Site ownership and permissions

Ownership is validated with:

```bash
ansible app -i ansible/inventory.ini -m command -a "stat -c '%U:%G %a %n' /srv/www/site-alpha /srv/www/site-alpha/public /srv/www/site-beta /srv/www/site-beta/public" -b
```

Expected model:

```txt
root:site_alpha /srv/www/site-alpha
site_alpha_deploy:site_alpha /srv/www/site-alpha/public
root:site_beta /srv/www/site-beta
site_beta_deploy:site_beta /srv/www/site-beta/public
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
| Site Alpha Access Logs | `logs-site-alpha-access-*` |
| Site Alpha Error Logs | `logs-site-alpha-error-*` |
| Site Beta Access Logs | `logs-site-beta-access-*` |
| Site Beta Error Logs | `logs-site-beta-error-*` |
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
- [Multi-site PHP Apache hosting](docs/validation/multi-site-php-hosting.md)

## License

This repository is published for portfolio and educational review purposes only.

All rights reserved. See [LICENSE](LICENSE).

## Project Status

Current milestone:

- Security baseline automated with Ansible
- Baseline Nginx service deployed on the application node
- Multi-site PHP Apache hosting deployed on the application node
- Nginx reverse proxy deployed for multi-site PHP hosting
- Two separated PHP sites deployed with two backend containers per site
- Per-site Linux ownership and permission model implemented
- Request ID and visitor ID correlation added to PHP hosting flow
- Per-site access and error logs collected by Filebeat
- node_exporter deployed on the application node
- Filebeat deployed on the application node
- Prometheus and Grafana deployed on the observability node
- Elasticsearch and Kibana deployed on the observability node
- Grafana Prometheus datasource provisioned automatically
- Grafana Linux Node Overview dashboard provisioned automatically
- Prometheus alert rules deployed automatically
- Centralized logging configured for auth, Fail2ban, Nginx, Docker, and site logs
- Kibana Data Views created for separate log categories
- Monitoring and logging workloads split across dedicated nodes
- Validation documented with screenshots and test notes
- Idempotence verified with repeated Ansible runs

Next planned milestone:

- Add screenshots for multi-site PHP hosting validation
- Add Kibana Discover screenshots for site access and error logs
- Alertmanager integration
- Kibana dashboard provisioning for site traffic
- Structured parsing for Nginx and site access logs
- Elasticsearch ingest pipelines
- Custom IDS integration for flow logs and live traffic analysis
