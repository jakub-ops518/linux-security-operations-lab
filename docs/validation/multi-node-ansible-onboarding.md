# Multi-node Ansible Onboarding

## Goal

Onboard a second Ubuntu Server virtual machine into the lab and prepare the project for a future multi-node observability split.

## Environment

- Host: Windows machine running VMware Workstation Pro
- Control node: WSL Ubuntu with Ansible
- Application node: `vm-app-01`
- Observability node: `vm-observability-01`
- Automation user: `ansible`
- Access method: SSH key authentication

## Current Node Roles

| Node | Current purpose |
|---|---|
| `vm-app-01` | Application server and current single-node monitoring/logging stack |
| `vm-observability-01` | Newly onboarded observability node with baseline Linux configuration |

At this stage, the observability node is onboarded and hardened, but the monitoring and logging backend services have not yet been migrated to it.

## Baseline Configuration

Both nodes are managed by Ansible and receive the common Linux baseline:

- Common system packages
- UFW firewall
- Fail2ban SSH protection
- Docker
- Dedicated `ansible` automation user
- SSH key-based access
- Passwordless sudo for automation

## Inventory Layout

The Ansible inventory is split into groups:

```ini
[app]
vm-app-01 ansible_host=<APP_VM_IP_ADDRESS> ansible_user=ansible ansible_ssh_private_key_file=<PATH_TO_PRIVATE_KEY>

[observability]
vm-observability-01 ansible_host=<OBSERVABILITY_VM_IP_ADDRESS> ansible_user=ansible ansible_ssh_private_key_file=<PATH_TO_PRIVATE_KEY>

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

## Current Playbook Layout

The current playbook keeps the existing single-node stack running on `vm-app-01`, while applying the common baseline to both nodes.

```yaml
---
- name: Configure common Linux baseline
  hosts: all
  become: true

  roles:
    - common
    - firewall
    - fail2ban
    - docker

- name: Configure current application and observability services
  hosts: app
  become: true

  roles:
    - nginx_container
    - monitoring_stack
    - logging_stack
```

## Validation Commands

Test Ansible connectivity:

```bash
ansible all -i ansible/inventory.ini -m ping
```

Verify privilege escalation:

```bash
ansible all -i ansible/inventory.ini -m command -a "whoami" -b
```

Apply the baseline and current service layout:

```bash
ansible-playbook -i ansible/inventory.ini ansible/site.yml
```

Run the playbook again to verify idempotence:

```bash
ansible-playbook -i ansible/inventory.ini ansible/site.yml
```

## Expected Result

Both nodes should be reachable through Ansible.

Expected connectivity result:

```txt
vm-app-01 | SUCCESS
vm-observability-01 | SUCCESS
```

Expected privilege escalation result:

```txt
root
```

The playbook should complete with:

```txt
failed=0
unreachable=0
```

## Result

The second VM was successfully onboarded into Ansible.

The lab is now prepared for a future multi-node split where:

- `vm-app-01` will run the application service, exporters, and log shippers
- `vm-observability-01` will run Prometheus, Grafana, Elasticsearch, and Kibana

## Notes

This milestone keeps the existing single-node service deployment functional while preparing the infrastructure for a cleaner multi-node architecture.

The next step is to split the current monitoring and logging roles into backend services and lightweight agents.
