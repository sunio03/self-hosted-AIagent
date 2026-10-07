# Runbook

## Snapshots

```bash
qm snapshot <vmid> <name> --description "<what changed>"
qm listsnapshot <vmid>
qm rollback <vmid> <name>
```

Take a snapshot of OPNsense (VM 100) before any firmware update or large rule change, and of agent VMs before installing software.

## OPNsense configuration backup

System → Configuration → Backups → **Download**. `config.xml` contains the entire firewall configuration
(and secrets), so store it privately — **never commit it to this repository**.

## Firmware updates

1. Snapshot VM 100.
2. System → Firmware → Status → Check for updates → Update.
3. Plugins only install on a fully current system ("Installation out of date" means update first).
4. Re-run the isolation tests in [verification.md](verification.md).

## Emergency access

If the OPNsense GUI or its Tailscale is unreachable:

1. Laptop → Tailscale → Proxmox (`<PROXMOX_TAILSCALE_IP>`).
2. Proxmox web UI → VM 100 → Console (or `qm terminal 100` if a serial port is configured).
3. Fix the rule or interface; the console menu also offers *Reset root password*.

If an agent VM's SSH is broken: `qm terminal 101` (serial console, no network needed), then fix
`/etc/ssh/sshd_config.d/00-hardening.conf` and `sudo systemctl reload ssh`.

## Quarantine an agent

1. Proxmox → VM → Hardware → Network Device → change **VLAN Tag 20 → 99**.
2. The VM keeps running but can reach nothing; every attempt is logged as *Quarantine: block all*.
3. Investigate (processes, memory, logs) via the Proxmox console. Snapshot first to preserve evidence.

## Add a new agent VM

1. Create the VM with NIC `bridge=vmbr1, tag=20, firewall=0`.
2. Install Ubuntu Server LTS; confirm a DHCP address in `<VLAN_AGENTS_DHCP_RANGE>`.
3. Apply the hardening baseline from [agent-vm.md](agent-vm.md).
4. Run the isolation tests.

## Checking what is actually loaded

```sh
# OPNsense shell (console option 8)
pfctl -sr | grep -i tailscale     # active rules, not just what the GUI shows
sockstat -4l | grep -E 'unbound|dnsmasq'
```

```bash
# Proxmox host
bridge vlan show                  # VLAN membership per port
bridge fdb show br vmbr1          # MAC table of the internal bridge
```
