# ansible-baseline

An Ansible collection (`mattiafait.baseline`) that gives **Debian/Ubuntu and RHEL-family** servers a production baseline in one run. Every role works standalone and is tested with Molecule on Debian 12 and Rocky Linux 9, including an idempotence pass.

| Role | What it does |
| --- | --- |
| `users` | Operators from a list (keys, groups, passwordless sudo for admins). Exclusive `authorized_keys`. **Revocation** locks accounts and deletes their keys while keeping them for audit trails |
| `hardening` | Key-only SSH (`sshd_config.d` drop-in, validated with `sshd -t`), no root login, optional `AllowGroups`, kernel sysctls, uncommon modules blacklisted, unattended security updates (`unattended-upgrades` / `dnf-automatic`) |
| `firewall` | One nftables ruleset for every distro: default-drop, per-port source CIDRs (IPv4 + IPv6), validated with `nft -c`, disables firewalld/ufw |
| `node_exporter` | Checksum-verified upstream release, sandboxed systemd unit, textfile collector directory |
| `restic` | Checksum-verified restic, password file + env (no secrets in unit files or logs), nightly timer, `forget --prune` retention, pre-backup hook for DB dumps |

## Quick start

```bash
ansible-galaxy collection install git+https://github.com/4137314/ansible-baseline.git
```

```yaml
# group_vars/all.yml
users_operators:
  - {name: alice, admin: true, keys: ["ssh-ed25519 AAAA... alice@laptop"]}
  - {name: ci, keys: ["ssh-ed25519 AAAA... deploy"], groups: [deploy]}
users_revoked: [bob]

firewall_rules:
  - {port: 22, from: [10.0.0.0/8]}
  - {port: 443}
  - {port: 9100, from: [10.0.10.5/32]}        # Prometheus only

restic_repository: s3:https://s3.eu-central-1.amazonaws.com/acme-backups/{{ inventory_hostname }}
restic_password: "{{ vault_restic_password }}"
restic_environment:
  AWS_ACCESS_KEY_ID: "{{ vault_aws_key }}"
  AWS_SECRET_ACCESS_KEY: "{{ vault_aws_secret }}"
```

```bash
ansible-playbook -i inventory mattiafait.baseline.site                 # everything
ansible-playbook -i inventory mattiafait.baseline.site --tags firewall # one role
```

Or use individual roles in your own playbooks: `- role: mattiafait.baseline.hardening`.

**Rollout tip:** apply `users` first and confirm that you can log in with your key before applying `hardening` (password logins are disabled) and `firewall` (default-drop).

## Works with

- [ops-toolkit](https://github.com/4137314/ops-toolkit): drop its `-f prom` output into `node_exporter_textfile_dir` from cron for certificate/backup/SSH-key metrics.
- [observability-stack](https://github.com/4137314/observability-stack): alert rules and dashboards for these hosts.
- [nixos-fleet](https://github.com/4137314/nixos-fleet): the same baseline, declaratively, for NixOS hosts.

## Development

```bash
pip install ansible-core ansible-lint molecule 'molecule-plugins[docker]'
ansible-galaxy collection build --force && ansible-galaxy collection install --force mattiafait-baseline-*.tar.gz
ansible-lint && molecule test
```

## License

MIT. Standardizing a mixed fleet or preparing for an audit? Open an issue or contact me via [GitHub](https://github.com/4137314).
