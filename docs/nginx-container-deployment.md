# Nginx Container Deployment Test

## Goal

Verify that Ansible can install Docker, deploy an Nginx container, expose HTTP traffic, and serve a custom static page.

## Environment

- Host: Windows machine running VMware Workstation Pro
- Control node: WSL Ubuntu with Ansible
- Managed node: Ubuntu Server `vm-app-01`
- Service: Dockerized Nginx
- Port: TCP/80

## Test Procedure

1. Installed Docker using the Ansible `docker` role.
2. Deployed a custom `index.html` file to `/opt/lab-nginx/html/index.html`.
3. Started an Nginx container using the Ansible `nginx_container` role.
4. Allowed TCP port 80 through UFW.
5. Tested HTTP response from the control node using:

```bash
curl http://<VM_IP_ADDRESS>

Result

The Nginx container successfully served the custom lab page.

Observed output:

<h1>Linux Security Operations Lab</h1>
<p>Nginx container deployed with Ansible.</p>
<p>Baseline security: UFW + fail2ban.</p>
Notes

This validates that the server baseline can be extended with application deployment through the same Ansible workflow.