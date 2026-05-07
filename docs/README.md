# Linux Security Operations Lab

A production-style Linux security operations lab built on Ubuntu Server virtual machines and managed with Ansible.

The goal of this project is to demonstrate practical Linux administration, infrastructure automation, basic security hardening, service deployment, and validation through documented tests.

## Current Features

- Ubuntu Server VM deployed in VMware Workstation Pro
- Dedicated Ansible automation user
- SSH key-based Ansible access
- UFW firewall with default-deny inbound policy
- Fail2ban SSH brute-force protection
- Docker installed through Ansible
- Nginx container deployed through Ansible
- HTTP access allowed through UFW
- Manual validation documented with screenshots and test notes

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
        └── Nginx container
```

## Ansible Roles

| Role | Purpose |
|---|---|
| `common` | Installs baseline system packages |
| `firewall` | Configures UFW firewall rules |
| `fail2ban` | Configures SSH brute-force protection |
| `docker` | Installs and enables Docker |
| `nginx_container` | Deploys a containerized Nginx service |

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

## Documentation

- [Security hardening](docs/security-hardening.md)
- [Fail2ban SSH ban test](docs/fail2ban-ssh-ban-test.md)
- [Nginx container deployment test](docs/validation/nginx-container-deployment.md)

## Project Status

Current milestone:

- Security baseline automated with Ansible
- Dockerized Nginx service deployed
- Idempotence verified with repeated Ansible runs

Next planned milestone:

- Monitoring with Prometheus and Grafana
- Node exporter on the managed VM
- Dashboard and alerting validation