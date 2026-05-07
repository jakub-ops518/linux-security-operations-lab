# Multi-site PHP Apache Hosting

## Goal

Deploy a multi-site PHP Apache hosting layer on the application node using Docker containers, Nginx reverse proxying, separated site ownership, request tracking, and centralized per-site logging.

## Environment

- Host: Windows machine running VMware Workstation Pro
- Control node: WSL Ubuntu with Ansible
- Application node: `vm-app-01`
- Observability node: `vm-observability-01`
- Deployment method: Ansible-managed Docker containers

## Architecture

```txt
vm-app-01
├── Existing baseline web service
│   └── lab-nginx
│       └── port 80
│
├── Multi-site PHP hosting stack
│   ├── lab-multisite-nginx
│   │   └── port 8080
│   ├── lab-site-alpha-php-01
│   ├── lab-site-alpha-php-02
│   ├── lab-site-beta-php-01
│   └── lab-site-beta-php-02
│
├── lab-node-exporter
└── lab-filebeat
```

The existing Nginx service remains available on port `80`.

The new multi-site PHP hosting stack is exposed on port `8080`.

## Site Layout

The hosting stack uses separated site directories under `/srv/www`.

```txt
/srv/www
├── _proxy
│   └── default.conf
├── site-alpha
│   ├── public
│   │   └── index.php
│   └── logs
│       ├── access.log
│       └── error.log
└── site-beta
    ├── public
    │   └── index.php
    └── logs
        ├── access.log
        └── error.log
```

## Ownership and Permission Model

Each site has its own Linux group and deployment user.

| Site | Deployment user | Group | Document root |
|---|---|---|---|
| `site-alpha` | `site_alpha_deploy` | `site_alpha` | `/srv/www/site-alpha/public` |
| `site-beta` | `site_beta_deploy` | `site_beta` | `/srv/www/site-beta/public` |

This provides a foundation for future multi-site hosting isolation where each hosted application can have separated ownership, deployment paths, and logs.

## Request and Visitor Correlation

The PHP application and Nginx reverse proxy provide basic request correlation.

Each request includes:

- site identifier
- backend instance name
- container hostname
- generated request ID
- generated visitor ID cookie
- timestamp

Nginx access logs include:

- request method
- URI
- status code
- request time
- upstream backend address
- upstream status
- request ID
- visitor ID
- user agent

This allows requests to be traced through:

```txt
Browser / curl
    ↓
Nginx reverse proxy
    ↓
PHP Apache backend container
    ↓
Per-site Nginx access log
    ↓
Filebeat
    ↓
Elasticsearch
    ↓
Kibana Data View
```

## Log Separation

Filebeat collects site logs separately and sends them to Elasticsearch on the observability node.

| Log source | Path | Elasticsearch index pattern |
|---|---|---|
| Site Alpha access logs | `/srv/www/site-alpha/logs/access.log` | `logs-site-alpha-access-*` |
| Site Alpha error logs | `/srv/www/site-alpha/logs/error.log` | `logs-site-alpha-error-*` |
| Site Beta access logs | `/srv/www/site-beta/logs/access.log` | `logs-site-beta-access-*` |
| Site Beta error logs | `/srv/www/site-beta/logs/error.log` | `logs-site-beta-error-*` |

## Kibana Data Views

The following Kibana Data Views can be created through the Kibana API:

| Data View | Index pattern |
|---|---|
| Site Alpha Access Logs | `logs-site-alpha-access-*` |
| Site Alpha Error Logs | `logs-site-alpha-error-*` |
| Site Beta Access Logs | `logs-site-beta-access-*` |
| Site Beta Error Logs | `logs-site-beta-error-*` |

Example API request:

```bash
curl -X POST "http://<OBSERVABILITY_VM_IP_ADDRESS>:5601/api/data_views/data_view" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "data_view": {
      "title": "logs-site-alpha-access-*",
      "name": "Site Alpha Access Logs",
      "timeFieldName": "@timestamp"
    }
  }'
```

## Validation Commands

Verify application node containers:

```bash
ansible app -i ansible/inventory.ini -m command -a "docker ps" -b
```

Expected containers:

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

Validate Site Alpha:

```bash
curl -i http://<APP_VM_IP_ADDRESS>:8080/site-alpha/
```

Validate Site Beta:

```bash
curl -i http://<APP_VM_IP_ADDRESS>:8080/site-beta/
```

Validate reverse proxy health endpoint:

```bash
curl http://<APP_VM_IP_ADDRESS>:8080/health
```

Validate Site Alpha load balancing:

```bash
for i in {1..20}; do
  curl -s http://<APP_VM_IP_ADDRESS>:8080/site-alpha/ | grep -o "site-alpha-php-[0-9][0-9]"
done
```

Validate Site Beta load balancing:

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

Validate ownership and permissions:

```bash
ansible app -i ansible/inventory.ini -m command -a "stat -c '%U:%G %a %n' /srv/www/site-alpha /srv/www/site-alpha/public /srv/www/site-beta /srv/www/site-beta/public" -b
```

Expected ownership model:

```txt
root:site_alpha /srv/www/site-alpha
site_alpha_deploy:site_alpha /srv/www/site-alpha/public
root:site_beta /srv/www/site-beta
site_beta_deploy:site_beta /srv/www/site-beta/public
```

Validate log files:

```bash
ansible app -i ansible/inventory.ini -m command -a "ls -la /srv/www/site-alpha/logs /srv/www/site-beta/logs" -b
```

Validate Elasticsearch indices:

```bash
curl "http://<OBSERVABILITY_VM_IP_ADDRESS>:9200/_cat/indices/logs-site-*?v"
```

Expected index patterns:

```txt
logs-site-alpha-access-*
logs-site-alpha-error-*
logs-site-beta-access-*
logs-site-beta-error-*
```

## Result

The multi-site PHP Apache hosting stack was deployed successfully.

The application node now hosts two separated PHP sites behind an Nginx reverse proxy, with per-site ownership, request correlation, load balancing across backend containers, and centralized access/error logging.

## Evidence

Screenshots will be added in a follow-up commit.

Planned screenshots:

- Site Alpha page showing backend instance, visitor ID, and request ID
- Site Beta page showing backend instance, visitor ID, and request ID
- Kibana Data Views for site access and error logs
- Kibana Discover showing request ID and visitor ID in site access logs

## Notes

This milestone adds a foundation for a future multi-site hosting model with stronger permission boundaries, per-site deployment workflows, and more advanced log parsing.
