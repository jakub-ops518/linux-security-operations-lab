# Linux Security Operations Lab

A production-style Linux security operations lab built on Ubuntu Server virtual machines and managed with Ansible.

The goal of this project is to demonstrate practical Linux administration, infrastructure automation, security hardening, service deployment, monitoring, alerting, centralized logging, multi-site web hosting, ingress routing, and validation through documented tests.

## Current State

The lab currently runs as a multi-node Linux security operations environment.

- `vm-ingress-01` runs the dedicated ingress layer.
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
- Dedicated ingress node deployed as the external HTTP entry point
- Nginx ingress proxy deployed on `vm-ingress-01`
- Ingress access and error logs collected by Filebeat
- Baseline Nginx container deployed on the application node
- Multi-site PHP Apache hosting deployed on the application node
- Nginx reverse proxy for PHP Apache backend containers
- Two separated PHP sites: `site-alpha` and `site-beta`
- Two PHP Apache backend containers per site
- Per-site Linux users and groups for ownership separation
- Per-site document roots under `/srv/www`
- Request ID and visitor ID correlation for web requests
- Per-site access and error log separation
- Application backend port restricted to the ingress node
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
- Separate Elasticsearch index patterns for auth, Fail2ban, Nginx, Docker, site, and ingress logs
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
    ├── vm-ingress-01
    │   ├── Ubuntu Server
    │   ├── UFW
    │   ├── Fail2ban
    │   ├── Docker
    │   ├── Nginx ingress proxy
    │   │   └── lab-ingress-nginx
    │   └── Filebeat
    │       └── lab-ingress-filebeat
    │
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
Nginx ingress proxy on vm-ingress-01
    ↓
Multi-site PHP hosting on vm-app-01
    ↓
Site access and error logs
    ↓
Filebeat on vm-app-01
    ↓
Elasticsearch on vm-observability-01
    ↓
Kibana Data Views
```

```txt
Ingress logs
    ↓
Filebeat on vm-ingress-01
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
| `ingress_proxy` | Deploys the dedicated Nginx ingress proxy |
| `ingress_filebeat_agent` | Deploys Filebeat on the ingress node |
| `nginx_container` | Deploys the baseline Nginx container |
| `multi_site_php_hosting` | Deploys multi-site PHP Apache hosting behind an internal Nginx reverse proxy |
| `node_exporter_agent` | Deploys node_exporter on the application node |
| `filebeat_agent` | Deploys Filebeat on the application node |
| `monitoring_stack` | Deploys Prometheus, Grafana, Grafana provisioning, and Prometheus alert rules |
| `logging_stack` | Deploys Elasticsearch and Kibana |
| `app_observability_cleanup` | Removes old backend containers from the application node |
| `observability_cleanup` | Removes application-side containers from the observability node |

## Inventory Layout

The inventory is split into host groups:

```ini
[ingress]
vm-ingress-01 ansible_host=<INGRESS_VM_IP_ADDRESS> ansible_user=ansible ansible_ssh_private_key_file=<PATH_TO_PRIVATE_KEY>

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
| Ingress proxy | `vm-ingress-01` | 80 | `http://<INGRESS_VM_IP_ADDRESS>` |
| Baseline Nginx | `vm-app-01` | 80 | `http://<APP_VM_IP_ADDRESS>` |
| Multi-site PHP hosting backend | `vm-app-01` | 8080 | restricted backend service |
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

### Dedicated ingress layer

Ingress containers are validated with:

```bash
ansible ingress -i ansible/inventory.ini -m command -a "docker ps" -b
```

Expected ingress containers:

```txt
lab-ingress-nginx
lab-ingress-filebeat
```

Ingress health is validated with:

```bash
curl http://<INGRESS_VM_IP_ADDRESS>/health
```

Expected response:

```txt
ingress ok
```

Site Alpha and Site Beta are reached through the ingress node:

```bash
curl -i http://<INGRESS_VM_IP_ADDRESS>/site-alpha/
curl -i http://<INGRESS_VM_IP_ADDRESS>/site-beta/
```

Expected response headers include:

```txt
X-Ingress-Node: vm-ingress-01
X-Request-ID: <generated_request_id>
```

Application backend port access is restricted to the ingress node:

```bash
ansible app -i ansible/inventory.ini -m command -a "ufw status numbered" -b
```

Expected rule model:

```txt
8080/tcp ALLOW FROM <INGRESS_VM_IP_ADDRESS>
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
| Ingress Access Logs | `logs-ingress-access-*` |
| Ingress Error Logs | `logs-ingress-error-*` |
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
- [Dedicated ingress layer](docs/validation/dedicated-ingress-layer.md)

## License

This repository is published for portfolio and educational review purposes only.

All rights reserved. See [LICENSE](LICENSE).

## Project Status

Current milestone:

- Security baseline automated with Ansible
- Dedicated ingress node deployed as the external HTTP entry point
- Nginx ingress proxy deployed on `vm-ingress-01`
- Ingress access and error logs collected by Filebeat
- Application backend port restricted to the ingress node
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
- Centralized logging configured for auth, Fail2ban, Nginx, Docker, site, and ingress logs
- Kibana Data Views created for separate log categories
- Monitoring, application, and ingress workloads split across dedicated nodes
- Validation documented with screenshots and test notes
- Idempotence verified with repeated Ansible runs

Next planned milestone:

- Add screenshots for dedicated ingress validation
- Add Kibana Discover screenshots for ingress access logs
- Add screenshots for multi-site PHP hosting validation
- Add Kibana Discover screenshots for site access and error logs
- TLS preparation for the ingress layer
- Alertmanager integration
- Kibana dashboard provisioning for ingress and site traffic
- Structured parsing for Nginx, ingress, and site access logs
- Elasticsearch ingest pipelines
- Custom IDS integration for flow logs and live traffic analysis
