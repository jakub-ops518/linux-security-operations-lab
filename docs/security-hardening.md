# Security Hardening

## UFW Firewall

UFW is enabled with a default-deny inbound policy.

Configuration:

- Incoming traffic: deny by default
- Outgoing traffic: allow by default
- SSH: allowed on port 22

This provides a basic host-level firewall baseline for the Ubuntu Server VM.

## SSH Brute-force Protection

Fail2ban is enabled for the SSH service as a baseline intrusion prevention mechanism.

Configuration:

- Jail: sshd
- Backend: systemd
- Max retries: 5
- Find time: 10 minutes
- Ban time: 10 minutes
- Ban action: UFW

Fail2ban monitors failed SSH authentication attempts and applies temporary firewall bans through UFW.

This is treated as a baseline control. A custom IDS component can later be integrated to analyze flow logs and live traffic for richer detection logic.