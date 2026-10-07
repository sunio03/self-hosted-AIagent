# Agent VM

## Specification (VM 101)

| Setting | Value | Reason |
|---|---|---|
| OS | Ubuntu Server 24.04 LTS (regular, not minimized) | Most agent install guides target Ubuntu; interactive tools available |
| vCPU | 2 | Agents mostly wait on cloud API responses |
| RAM | 4 GB | Runtime plus headroom for a headless browser |
| Disk | 32–40 GB qcow2 on `local` | Snapshot support on directory storage |
| NIC | VirtIO, `bridge=vmbr1`, `tag=20`, **firewall=0** | AGENTS zone; OPNsense is the only policy engine |
| GPU | None | Models run via cloud APIs, not locally |

A full VM (not an LXC container) is used for agents because a separate kernel is a stronger isolation boundary.
Each agent is intended to get its own VM to limit blast radius (one compromised agent cannot read another's files or keys).

## Installation notes

- ISO downloaded directly to Proxmox from `releases.ubuntu.com`, verified against `SHA256SUMS`.
- Network left on DHCP: the installer received `<AGENT_VM_IP>`, the first proof that tag → trunk → OPNsense → Dnsmasq works.
- OpenSSH server installed; featured snaps skipped (Docker etc. will be installed deliberately from official repositories when needed).
- Personal admin account (not `admin`/`ubuntu`) for accountability in `auth.log`.

## Hardening baseline

| Measure | Status |
|---|---|
| Full upgrade after install | ✅ |
| Automatic security updates (`unattended-upgrades`) | ✅ (Ubuntu default, verified active) |
| QEMU guest agent | ✅ |
| SSH: keys only (ed25519, passphrase-protected), password and keyboard-interactive auth off | ✅ |
| SSH: root login disabled | ✅ |
| Effective sshd settings verified with `sshd -T` | ✅ |
| Network isolation (VLAN 20 rules) | ✅ verified |
| Dedicated `agents` service account (system user, no sudo, `nologin` shell) | ⏳ planned |
| Listening-services review (`ss -tulpn`) | ⏳ planned |
| Host firewall (`ufw`, allow 22 only) | Optional extra layer |

### SSH hardening file

```text
# /etc/ssh/sshd_config.d/00-hardening.conf
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
```

The `00-` prefix matters: sshd uses the **first** value it reads, and Ubuntu's `50-cloud-init.conf` can contain
`PasswordAuthentication yes`. A file sorted after it would be silently ignored. Always confirm with:

```bash
sudo sshd -t
sudo sshd -T | grep -E 'passwordauthentication|kbdinteractiveauthentication|permitrootlogin'
```

## Planned service account

```bash
sudo adduser --system --group --home /opt/agents --shell /usr/sbin/nologin agents
```

Agents will run as systemd services under this account, so a compromised agent does not inherit the admin account's `sudo`.
