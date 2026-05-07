# Fail2ban SSH Ban Test

## Goal

Verify that fail2ban detects repeated failed SSH login attempts and bans the source IP address.

## Environment

- Host: Windows machine running VMware Workstation Pro
- VM: Ubuntu Server `vm-app-01`
- Protection: UFW + fail2ban
- Jail tested: `sshd`

## Test Procedure

1. Enabled UFW with a default-deny inbound policy.
2. Enabled fail2ban with the `sshd` jail.
3. Temporarily configured fail2ban with a lower retry threshold for testing.
4. Attempted multiple failed SSH logins from the host machine.
5. Checked fail2ban status using:

```bash
sudo fail2ban-client status sshd
Result

Fail2ban successfully detected repeated failed SSH authentication attempts and banned the source IP.

Observed output:

Currently banned: 1
Total banned: 1
Banned IP list: RESTRICTED
Evidence

Screenshot:

Recovery

The source IP was manually unbanned using:

sudo fail2ban-client set sshd unbanip RESTRICTED
Notes

This test confirms that the SSH brute-force protection baseline is functional.

For the public GitHub version, private lab IP addresses may be left as RFC1918/private addresses or partially redacted if needed.