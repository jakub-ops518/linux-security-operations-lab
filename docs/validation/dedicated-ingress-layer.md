# Dedicated Ingress Layer

## Goal

Add a dedicated ingress node in front of the application node so public HTTP traffic enters the lab through a single edge layer instead of directly reaching backend services.

## Environment

- Host: Windows machine running VMware Workstation Pro
- Control node: WSL Ubuntu with Ansible
- Ingress node: `vm-ingress-01`
- Application node: `vm-app-01`
- Observability node: `vm-observability-01`
- Deployment method: Ansible-managed Docker containers

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
    │   └── Filebeat
    │
    ├── vm-app-01
    │   ├── Ubuntu Server
    │   ├── UFW
    │   ├── Fail2ban
    │   ├── Docker
    │   ├── Baseline Nginx
    │   ├── Multi-site PHP hosting
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

| Node | Services |
|---|---|
| `vm-ingress-01` | Nginx ingress proxy, ingress Filebeat |
| `vm-app-01` | Baseline Nginx, multi-site PHP hosting, node_exporter, application Filebeat |
| `vm-observability-01` | Prometheus, Grafana, Elasticsearch, Kibana |

## Ingress Routing

The ingress node exposes HTTP on port `80` and forwards selected paths to the internal application node.

| Public path | Backend target |
|---|---|
| `/site-alpha/` | `vm-app-01:8080/site-alpha/` |
| `/site-beta/` | `vm-app-01:8080/site-beta/` |
| `/health` | local ingress health endpoint |

The application node keeps the PHP hosting stack on port `8080`, but the intended entry point is now the ingress node.

## Backend Isolation

The application node firewall is configured so port `8080` is allowed only from the ingress node.

Expected model:

```txt
User / Browser / curl
    ↓
vm-ingress-01:80
    ↓
vm-app-01:8080
    ↓
PHP Apache backend containers
```

Backend traffic should not require users to access `vm-app-01:8080` directly.

## Logging

Ingress access and error logs are collected separately.

| Log source | Path | Elasticsearch index pattern |
|---|---|---|
| Ingress access logs | `/opt/lab-ingress/logs/access.log` | `logs-ingress-access-*` |
| Ingress error logs | `/opt/lab-ingress/logs/error.log` | `logs-ingress-error-*` |

Ingress logs include:

- remote address
- requested host
- HTTP method
- URI
- status code
- request time
- upstream backend address
- upstream status
- request ID
- user agent

This allows tracing requests through:

```txt
Client
    ↓
Ingress access log
    ↓
Application backend
    ↓
Site access log
    ↓
Filebeat
    ↓
Elasticsearch
    ↓
Kibana
```

## Kibana Data Views

The following Kibana Data Views were created through the Kibana API:

| Data View | Index pattern |
|---|---|
| Ingress Access Logs | `logs-ingress-access-*` |
| Ingress Error Logs | `logs-ingress-error-*` |

Example API request:

```bash
curl -X POST "http://<OBSERVABILITY_VM_IP_ADDRESS>:5601/api/data_views/data_view" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "data_view": {
      "title": "logs-ingress-access-*",
      "name": "Ingress Access Logs",
      "timeFieldName": "@timestamp"
    }
  }'
```

## Validation Commands

Verify ingress containers:

```bash
ansible ingress -i ansible/inventory.ini -m command -a "docker ps" -b
```

Expected ingress containers:

```txt
lab-ingress-nginx
lab-ingress-filebeat
```

Validate ingress health endpoint:

```bash
curl http://<INGRESS_VM_IP_ADDRESS>/health
```

Expected response:

```txt
ingress ok
```

Validate Site Alpha through ingress:

```bash
curl -i http://<INGRESS_VM_IP_ADDRESS>/site-alpha/
```

Validate Site Beta through ingress:

```bash
curl -i http://<INGRESS_VM_IP_ADDRESS>/site-beta/
```

Expected response headers include:

```txt
X-Ingress-Node: vm-ingress-01
X-Request-ID: <generated_request_id>
```

Validate that the ingress node can reach the application backend:

```bash
ansible ingress -i ansible/inventory.ini -m command -a "curl -s http://<APP_VM_IP_ADDRESS>:8080/health" -b
```

Expected response:

```txt
ok
```

Validate application node firewall rules:

```bash
ansible app -i ansible/inventory.ini -m command -a "ufw status numbered" -b
```

Expected rule model:

```txt
8080/tcp ALLOW FROM <INGRESS_VM_IP_ADDRESS>
```

Validate ingress logs:

```bash
ansible ingress -i ansible/inventory.ini -m command -a "tail -n 10 /opt/lab-ingress/logs/access.log" -b
```

Expected log fields include:

```txt
remote_addr=
uri=
status=
upstream_addr=
request_id=
```

Validate Elasticsearch ingress indices:

```bash
curl "http://<OBSERVABILITY_VM_IP_ADDRESS>:9200/_cat/indices/logs-ingress-*?v"
```

Expected index patterns:

```txt
logs-ingress-access-*
logs-ingress-error-*
```

## Result

The dedicated ingress layer was deployed successfully.

The lab now has a clear edge/backend/observability separation:

- ingress traffic enters through `vm-ingress-01`
- application hosting remains on `vm-app-01`
- metrics and logs are centralized on `vm-observability-01`

## Evidence

Screenshots will be added in a follow-up commit.

Planned screenshots:

- Ingress health endpoint
- Site Alpha and Site Beta accessed through ingress
- Kibana Discover showing ingress access logs
- UFW rule showing backend port restricted to the ingress node

## Notes

This milestone prepares the lab for future public-facing deployments where DNS and TLS can terminate at the ingress node while backend services remain isolated.
