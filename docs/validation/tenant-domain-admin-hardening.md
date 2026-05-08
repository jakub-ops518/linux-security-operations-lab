# Tenant Domain Admin Hardening

## Goal

Harden the tenant domain administration portal after the initial MVP deployment.

This milestone improves operational safety, auditability, and request tracing for tenant-managed domain assignments.

## Scope

Implemented hardening changes:

- Tenant admin audit logging
- Duplicate domain assignment protection
- Canonical `X-Request-ID` propagation through the ingress layer
- Removal of duplicate upstream `X-Request-ID` response headers
- Audit log path exposed through the tenant admin container environment
- Generated tenant domain Nginx config updated to match request ID behavior

## Environment

- Ingress node: `vm-ingress-01`
- Application node: `vm-app-01`
- Observability node: `vm-observability-01`
- Tenant admin portal: `http://<INGRESS_VM_IP_ADDRESS>/admin/`
- Tenant registry: `/opt/lab-tenant-admin/data/tenant-domains.json`
- Tenant audit log: `/opt/lab-tenant-admin/data/audit.log`
- Generated ingress config: `/opt/lab-ingress/dynamic/tenant-domains.conf`

## Audit Logging

The tenant admin portal now writes JSON audit events to:

```txt
/opt/lab-tenant-admin/data/audit.log
```

Audit events include:

- successful login
- failed login
- logout
- domain update
- domain removal
- failed duplicate domain assignment

Example audit events:

```json
{"domain": "", "event_type": "login", "reason": "", "remote_addr": "<HOST_PRIVATE_IP>", "status": "ok", "timestamp": 1778252778, "username": "site-alpha"}
{"domain": "alpha.lab.local", "event_type": "domain_remove", "reason": "", "remote_addr": "<HOST_PRIVATE_IP>", "status": "ok", "timestamp": 1778252780, "username": "site-alpha"}
{"domain": "random.test", "event_type": "domain_update", "reason": "previous=", "remote_addr": "<HOST_PRIVATE_IP>", "status": "ok", "timestamp": 1778252784, "username": "site-alpha"}
{"domain": "random.test", "event_type": "domain_update", "reason": "Domain already assigned to site-alpha", "remote_addr": "<HOST_PRIVATE_IP>", "status": "failed", "timestamp": 1778252923, "username": "site-beta"}
```

## Duplicate Domain Protection

The tenant admin portal prevents two tenants from assigning the same domain.

Validation scenario:

1. `site-alpha` owns `alpha.lab.local`.
2. `site-beta` attempts to assign `alpha.lab.local`.
3. The portal rejects the update.
4. The rejection is written to the audit log.

Expected audit log entry:

```txt
"status": "failed"
"reason": "Domain already assigned to site-alpha"
"username": "site-beta"
```

## Request ID Normalization

Before hardening, responses could include duplicate `X-Request-ID` headers:

```txt
X-Request-ID: <application_request_id>
X-Request-ID: <ingress_request_id>
```

The ingress layer now hides the upstream response `X-Request-ID` and returns a single canonical ingress request ID.

Implemented in:

```txt
ansible/roles/ingress_proxy/templates/default.conf.j2
ansible/roles/tenant_admin_portal/templates/render_tenant_domains.py.j2
```

Expected behavior:

```txt
X-Request-ID: <single_ingress_request_id>
```

## Validation Commands

Validate admin portal availability:

```bash
curl -i http://<INGRESS_VM_IP_ADDRESS>/admin/
```

Expected result:

```txt
Tenant Domain Administration
```

Validate Site Alpha routing and request ID behavior:

```bash
curl -i -H "Host: alpha.lab.local" http://<INGRESS_VM_IP_ADDRESS>/ | grep -E "X-Lab-Tenant|X-Request-ID"
```

Expected result:

```txt
X-Lab-Tenant: site-alpha
X-Request-ID: <single_request_id>
```

Validate Site Beta routing and request ID behavior:

```bash
curl -i -H "Host: beta.lab.local" http://<INGRESS_VM_IP_ADDRESS>/ | grep -E "X-Lab-Tenant|X-Request-ID"
```

Expected result:

```txt
X-Lab-Tenant: site-beta
X-Request-ID: <single_request_id>
```

Validate audit log:

```bash
ansible ingress -i ansible/inventory.ini -m command -a "tail -n 20 /opt/lab-tenant-admin/data/audit.log" -b
```

Expected event types:

```txt
login
logout
domain_update
domain_remove
```

Validate duplicate domain rejection:

```txt
1. Log in as site-beta.
2. Attempt to assign a domain already used by site-alpha.
3. Confirm the portal rejects the change.
4. Confirm audit.log records a failed domain_update event.
```

Validate ingress access logs in Elasticsearch/Kibana:

```txt
log_type: ingress_access
lab_host: vm-ingress-01
service_name: ingress_proxy
host: alpha.lab.local
request_id: <single_request_id>
```

Example ingress log message:

```txt
remote_addr=<HOST_PRIVATE_IP> time="08/May/2026:15:01:31 +0000" host="alpha.lab.local" method=GET uri="/" status=200 bytes=736 request_time=0.003 upstream_addr="<HOST_PRIVATE_IP>29:8080" upstream_status="200" request_id="306cf5063753be57f8ea758317ef222e" user_agent="curl/8.18.0"
```

## Confirmed Results

Confirmed working behavior:

```txt
/admin/          -> tenant admin portal
alpha.lab.local  -> site-alpha
beta.lab.local   -> site-beta
```

Confirmed hardening behavior:

```txt
audit log: OK
duplicate domain protection: OK
single canonical X-Request-ID: OK
ingress access log correlation: OK
```

## Files Changed

Relevant files:

```txt
ansible/roles/tenant_admin_portal/templates/app.py.j2
ansible/roles/tenant_admin_portal/templates/render_tenant_domains.py.j2
ansible/roles/ingress_proxy/templates/default.conf.j2
ansible/roles/ingress_proxy/templates/docker-compose.yml.j2
```

## Follow-up Improvements

Possible next improvements:

- Ship tenant admin audit logs to Elasticsearch as a dedicated index
- Add Kibana Data View for tenant admin audit events
- Add rate limiting for admin login attempts
- Add account lockout after repeated failed login attempts
- Add CSRF protection for admin form submissions
- Replace local JSON registry with persistent database storage
- Add role-based admin and tenant permissions
- Add TLS termination on the ingress layer
