# Tenant Domain Admin Portal

## Goal

Add a tenant-facing administration portal that allows assigned site owners to log in and connect one domain to their hosted site.

The portal writes domain assignments into a registry file. A systemd-managed renderer then generates Nginx ingress configuration and reloads the ingress proxy.

## Environment

- Ingress node: `vm-ingress-01`
- Application node: `vm-app-01`
- Observability node: `vm-observability-01`
- Admin portal path: `/admin/`
- Ingress IP: `<INGRESS_VM_IP_ADDRESS>`
- Deployment method: Ansible-managed Docker containers and systemd units

## Architecture

```txt
vm-ingress-01
├── lab-ingress-nginx
│   ├── default ingress routing
│   ├── /admin/ reverse proxy
│   └── dynamic tenant domain server blocks
│
├── lab-tenant-admin
│   └── tenant domain administration portal
│
├── lab-ingress-filebeat
│   └── ships ingress logs to Elasticsearch
│
├── /opt/lab-tenant-admin
│   ├── app.py
│   ├── config/tenants.json
│   └── data/tenant-domains.json
│
└── /opt/lab-ingress/dynamic
    └── tenant-domains.conf
```

## Tenant Model

The current MVP supports two predefined tenants.

| Tenant | Assigned site path | Example domain |
|---|---|---|
| `site-alpha` | `/site-alpha/` | `alpha.lab.local` |
| `site-beta` | `/site-beta/` | `beta.lab.local` |

Tenant passwords and portal secret values are stored only in the local ignored `ansible/inventory.ini`.

The public example inventory contains only placeholders.

## Admin Portal

The admin portal is exposed through the ingress node:

```txt
http://<INGRESS_VM_IP_ADDRESS>/admin/
```

A tenant can:

- log in with assigned credentials
- view its assigned site path
- set one domain
- remove the configured domain

The admin portal writes domain assignments to:

```txt
/opt/lab-tenant-admin/data/tenant-domains.json
```

Example registry content:

```json
{
  "site-alpha": {
    "domain": "alpha.lab.local",
    "updated_at": 1778248521
  },
  "site-beta": {
    "domain": "beta.lab.local",
    "updated_at": 1778249417
  }
}
```

## Dynamic Ingress Rendering

A systemd service renders the tenant registry into Nginx configuration:

```txt
tenant-domain-render.service
```

A systemd timer periodically runs the renderer:

```txt
tenant-domain-render.timer
```

The generated config is written to:

```txt
/opt/lab-ingress/dynamic/tenant-domains.conf
```

The main ingress Nginx config includes the generated tenant domain config after the default server block.

This keeps direct IP access routed to the default ingress server while tenant domains route to their assigned sites.

## Routing Model

| Request | Expected route |
|---|---|
| `http://<INGRESS_VM_IP_ADDRESS>/` | Default ingress page |
| `http://<INGRESS_VM_IP_ADDRESS>/admin/` | Tenant admin portal |
| `Host: alpha.lab.local` | `site-alpha` |
| `Host: beta.lab.local` | `site-beta` |

## Validation

Validate admin portal:

```bash
curl -i http://<INGRESS_VM_IP_ADDRESS>/admin/
```

Expected result:

```txt
Tenant Domain Administration
```

Validate default ingress route:

```bash
curl -i http://<INGRESS_VM_IP_ADDRESS>/
```

Expected result:

```txt
Dedicated ingress layer. Available paths: /site-alpha/, /site-beta/, and /admin/
```

Validate generated tenant domain config:

```bash
ansible ingress -i ansible/inventory.ini -m command -a "cat /opt/lab-ingress/dynamic/tenant-domains.conf" -b
```

Expected generated server blocks:

```txt
server_name alpha.lab.local;
proxy_pass http://app_php_hosting/site-alpha/;

server_name beta.lab.local;
proxy_pass http://app_php_hosting/site-beta/;
```

Validate Site Alpha domain routing without DNS:

```bash
curl -i -H "Host: alpha.lab.local" http://<INGRESS_VM_IP_ADDRESS>/
```

Expected headers/content:

```txt
X-Lab-Tenant: site-alpha
Site: site-alpha
```

Validate Site Beta domain routing without DNS:

```bash
curl -i -H "Host: beta.lab.local" http://<INGRESS_VM_IP_ADDRESS>/
```

Expected headers/content:

```txt
X-Lab-Tenant: site-beta
Site: site-beta
```

Validate containers:

```bash
ansible ingress -i ansible/inventory.ini -m command -a "docker ps" -b
```

Expected containers:

```txt
lab-ingress-nginx
lab-ingress-filebeat
lab-tenant-admin
```

Validate renderer manually:

```bash
ansible ingress -i ansible/inventory.ini -m command -a "systemctl start tenant-domain-render.service" -b
```

Validate service logs:

```bash
ansible ingress -i ansible/inventory.ini -m command -a "journalctl -u tenant-domain-render.service -n 80 --no-pager" -b
```

## Result

The tenant domain admin portal was deployed successfully.

The lab now supports tenant-managed domain assignment for hosted sites through the dedicated ingress layer.

Confirmed routing:

```txt
/admin/          -> tenant admin portal
alpha.lab.local  -> site-alpha
beta.lab.local   -> site-beta
```

## Notes

DNS is not automated in this lab milestone. Domain routing is validated through the HTTP `Host` header.

For a live deployment, DNS records would need to point tenant domains to the ingress node public IP.

## Follow-up Improvements

- Normalize request ID propagation between ingress and application reverse proxy
- Add audit logging for domain changes
- Add validation to prevent duplicate domain assignments
- Add TLS support on the ingress layer
- Add persistent database storage for tenant records
- Add role-based admin and tenant permissions
