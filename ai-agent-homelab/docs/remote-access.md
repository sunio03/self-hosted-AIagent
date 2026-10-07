# Remote Access

The lab runs at home while I administer it from another country. Requirements: no open inbound ports, an independent
emergency path, least-privilege access, and the firewall as the enforcement point.

## Access paths

| Path | Route | Control |
|---|---|---|
| OPNsense GUI | Laptop → Tailscale → `<OPNSENSE_TAILSCALE_IP>:443` | OPNsense TAILSCALE rule (laptop only) + tailnet policy |
| SSH to agents | Laptop → Tailscale → OPNsense (subnet router) → `<VLAN_AGENTS_CIDR>:22` | Tailnet policy **and** OPNsense TAILSCALE rule |
| Proxmox host | Laptop → Tailscale → `<PROXMOX_TAILSCALE_IP>` | Tailnet policy; independent of OPNsense |
| Emergency console | Proxmox web UI → VM console, or `qm terminal <vmid>` | Works even if OPNsense or its Tailscale fails |

## Tailscale on OPNsense

Configured through the `os-tailscale` plugin so the settings live in `config.xml` (survive updates, included in backups).

| Setting | Value | Reason |
|---|---|---|
| Login server | `controlplane.tailscale.com` | Tailscale control plane |
| Listen port | 41641 | No inbound WAN rule needed; Tailscale uses outbound NAT traversal |
| Accept DNS | Off | The firewall keeps control of its own DNS |
| Exit node (advertise / use) | Off / None | Not an exit-node design |
| Accept subnet routes | Off | The firewall decides its own routing |
| Advertised routes | `<VLAN_AGENTS_CIDR>` | SSH to the agent zone |
| **Disable SNAT** | **On** | Keeps the laptop's real source address so OPNsense rules apply and logs are attributable |

The `tailscale0` interface is assigned (TAILSCALE / `opt1`), **enabled**, and locked. The advertised route was approved
in the Tailscale admin console. Node key expiry is disabled for remotely managed servers.

### Auth key practice

Single-use (not reusable), not ephemeral, 1-day expiry. Never pasted into chat, screenshots or the repository.
The auth key only admits a device; it is separate from the device's node key.

## Tailnet policy

Tailscale's default policy (`src *` → `dst *`, all ports) allowed every tailnet device to reach every other device, including
the Proxmox host. It was replaced with a default-deny policy: only the admin laptop may initiate connections.
See [`configs/tailscale-policy.hujson`](../configs/tailscale-policy.hujson).

| From → To | Before | After |
|---|---|---|
| Laptop → any tailnet device | ✅ | ✅ |
| Laptop → agents | ✅ any port | ✅ TCP 22 only |
| Lab machines (e.g. Kali, Rocky) → Proxmox | ✅ | ❌ |
| Lab machines ↔ each other | ✅ | ❌ |
| Any tailnet device → agents | ✅ any port | ❌ |

The policy's `tests` block is evaluated on every save; a failing test prevents the change from being applied.

## Defence in depth for agent SSH

```
Laptop ──> Tailscale policy (laptop → agents, tcp:22) ──> OPNsense rule (laptop → AGENTS net, 22, logged) ──> agent sshd (keys only)
```

All three layers must allow the connection.
