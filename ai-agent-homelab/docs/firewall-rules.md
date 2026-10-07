# Firewall Rules

OPNsense evaluates interface rules top to bottom; the first match wins (all rules use *quick*).
All rules: direction **in**, **IPv4**. Anything not explicitly passed is dropped by the default deny.

## AGENTS (VLAN 20)

| # | Action | Proto | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | AGENTS network | AGENTS address | 53 | – | Allow DNS to firewall |
| 2 | Pass | ICMP | AGENTS network | AGENTS address | – | – | Allow ping to gateway |
| 3 | Block | TCP/UDP | AGENTS network | any | 53 | ✅ | Block external DNS |
| 4 | Block | any | AGENTS network | This Firewall | any | ✅ | Block access to firewall |
| 5 | Block | any | AGENTS network | `PRIVATE_NETS` | any | ✅ | Block internal networks |
| 6 | Pass | any | AGENTS network | any | any | – | Allow internet |

**Why the order matters**

- Rules 1–2 sit above rule 4, otherwise blocking the firewall would also block DNS and gateway ping.
- Rule 3 sits above rule 6, otherwise DNS to public resolvers would be allowed as "internet".
- Rule 5 sits above rule 6. Because every private destination is already blocked, rule 6's "any" effectively means "public internet only".
- Rule 4 overlaps with rule 5 (all firewall addresses are currently private). It is kept for a clear log label and to remain correct if WAN ever receives a public address (defence in depth).

**Name reference**

| Name | Meaning |
|---|---|
| AGENTS network | The whole subnet `<VLAN_AGENTS_CIDR>` |
| AGENTS address | Only the firewall's own address on that VLAN, `<VLAN_AGENTS_GATEWAY_IP>` |
| This Firewall | All addresses configured on OPNsense interfaces (not the Tailscale address) |

DHCP works without explicit rules because OPNsense's automatically generated DHCP rules are evaluated before interface rules.

## SERVICES (VLAN 30)

Same six rules as AGENTS, with `SERVICES network` as source and `SERVICES address` as destination in rules 1–2.
Specific AGENTS → SERVICES rules will be added once the services are defined.

## QUARANTINE (VLAN 99)

| # | Action | Proto | Source | Destination | Log | Description |
|---|---|---|---|---|---|---|
| 1 | Block | any | QUARANTINE network | any | ✅ | Quarantine: block all |

The default deny would block this traffic anyway; the explicit rule gives a clear log label, which is useful evidence during an investigation.

## TAILSCALE (`tailscale0`)

| # | Action | Proto | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP | `<MACBOOK_TAILSCALE_IP>` (admin laptop) | This Firewall | 443 | ✅ | Admin GUI from laptop via Tailscale |
| 2 | Pass | TCP | `<MACBOOK_TAILSCALE_IP>` (admin laptop) | AGENTS network | 22 | ✅ | SSH from laptop to agents via Tailscale |

Rule 2 is only effective because **SNAT is disabled** in the Tailscale plugin. With SNAT enabled, Tailscale re-originates
subnet traffic from the firewall itself and this rule is bypassed. See [troubleshooting.md](troubleshooting.md#case-2-tailscale-subnet-traffic-bypassed-the-firewall-rule).

## WAN

No pass rules and no port forwards. Bogon blocking on; RFC 1918 blocking off (WAN itself is on a private network behind the home router).

## LAN (management)

Default OPNsense rules (anti-lockout, LAN to any). No DHCP on this network, so a VM attached without a VLAN tag receives no address.
