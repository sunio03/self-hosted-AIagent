# Network Design

## Addressing plan

| Network | Subnet | Gateway / DNS | DHCP | Purpose |
|---|---|---|---|---|
| Home (WAN side) | `<HOME_LAN_CIDR>` | `<HOME_GATEWAY_IP>` (home router) | Home router | Uplink; Proxmox host at `.100` |
| LAN (management) | `<OPNSENSE_LAN_CIDR>` | `<OPNSENSE_LAN_IP>` | **None, on purpose** | Admin network, fail-closed |
| VLAN 20 AGENTS | `<VLAN_AGENTS_CIDR>` | `<VLAN_AGENTS_GATEWAY_IP>` | `.100 – .200` | AI agents |
| VLAN 30 SERVICES | `<VLAN_SERVICES_CIDR>` | `<VLAN_SERVICES_GATEWAY_IP>` | `.100 – .200` | Supporting services |
| VLAN 99 QUARANTINE | `<VLAN_QUARANTINE_CIDR>` | `<VLAN_QUARANTINE_GATEWAY_IP>` | `.100 – .200` | Containment |
| Tailnet | `100.64.0.0/10` | – | Tailscale | Remote administration |

Addresses `.2 – .99` in each VLAN are reserved for static assignments.

## Interfaces

Every OPNsense interface has a FreeBSD device name, an internal identifier (used in `config.xml`) and a description.

| Description | Identifier | Device | Address |
|---|---|---|---|
| WAN | `wan` | `vtnet0` | `<HOME_LAN_WAN_IP>` (DHCP), IPv6 none |
| LAN | `lan` | `vtnet1` | `<OPNSENSE_LAN_IP>/24` |
| TAILSCALE | `opt1` | `tailscale0` | `<OPNSENSE_TAILSCALE_IP>` (assigned by Tailscale) |
| AGENTS | `opt2` | `vlan0.20` | `<VLAN_AGENTS_GATEWAY_IP>/24` |
| SERVICES | `opt3` | `vlan0.30` | `<VLAN_SERVICES_GATEWAY_IP>/24` |
| QUARANTINE | `opt4` | `vlan0.99` | `<VLAN_QUARANTINE_GATEWAY_IP>/24` |

`tailscale0` is a Layer 3 tunnel interface, which is why it shows MAC `00:00:00:00:00:00`.

## Proxmox side

| Bridge | Physical NIC | Purpose |
|---|---|---|
| `vmbr0` | USB Ethernet adapter | Home network / WAN uplink; Proxmox management |
| `vmbr1` | **None** | VLAN-aware internal bridge for all inside zones |

Because `vmbr1` has no physical NIC, nothing on the inside can reach the outside world except through OPNsense.

### VM network chain

```
Inside the guest:  vtnet0   (FreeBSD VirtIO driver name; the IP lives here)
                     │  same virtual NIC
Proxmox config:    net0     (model virtio, MAC, bridge=vmbr0)
                     │  realised on the host as
Proxmox host:      tap100i0 ──plugged into──> vmbr0
```

### VLAN membership (`bridge vlan show`)

| Port | VLANs | Role |
|---|---|---|
| `tap100i1` (OPNsense LAN NIC) | 1 PVID untagged, 2–4094 tagged | **Trunk**: native VLAN = management LAN |
| `tap101i0` (agent VM) | 20 PVID untagged | **Access port** on VLAN 20 |
| `vmbr1` (host port) | 1 only | Host is not present in any agent VLAN |

VLAN tags are applied by Proxmox (`tag=20` on the VM NIC), not inside guests, so a guest cannot hop VLANs.
The Proxmox host has **no IP address on `vmbr1`**: it forwards agent traffic at Layer 2 but cannot be reached from it.

## DNS and DHCP

| Service | Port | Scope |
|---|---|---|
| Unbound (DNS) | 53 | Recursive resolution from the root servers, DNSSEC validation, caching |
| Dnsmasq (DHCP) | DNS part on 53053 | DHCP on AGENTS, SERVICES, QUARANTINE only |

- Dnsmasq hands out the VLAN's interface address as both gateway and DNS server; Unbound answers on port 53 at that address.
- No DHCP range exists for the management LAN.
- Unbound does not use upstream forwarders and does not accept DNS from the WAN DHCP lease.

## Firewall aliases

| Alias | Type | Content |
|---|---|---|
| `PRIVATE_NETS` | Network(s) | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `100.64.0.0/10` |

`100.64.0.0/10` is included so that agents cannot reach tailnet devices (including the Proxmox host's Tailscale address).
