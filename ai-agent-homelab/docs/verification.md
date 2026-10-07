# Verification

Configuration is not proof. Every security property below was tested from the relevant side and checked against the
OPNsense live firewall log (Firewall → Log Files → Live View).

## 1. Agent zone isolation

Run from the agent VM `<AGENT_VM_IP>` (VLAN 20).

| # | Command | Expected | Rule exercised | Result |
|---|---|---|---|---|
| 1 | `ip -4 addr show` | Address in `<VLAN_AGENTS_DHCP_RANGE>` | DHCP (automatic rules) | ✅ `.185` |
| 2 | `ip route` | Default via `<VLAN_AGENTS_GATEWAY_IP>` | – | ✅ |
| 3 | `ping -c 3 <VLAN_AGENTS_GATEWAY_IP>` | Reply | AGENTS #2 | ✅ |
| 4 | `nslookup github.com` | Resolves via `<VLAN_AGENTS_GATEWAY_IP>` | AGENTS #1 | ✅ |
| 5 | `curl -I https://example.com` | HTTP response | AGENTS #6 | ✅ |
| 6 | `nslookup github.com 8.8.8.8` | Timeout, logged | AGENTS #3 | ✅ blocked |
| 7 | `curl -k -m 5 https://<OPNSENSE_LAN_IP>` | Timeout, logged | AGENTS #4 | ✅ blocked |
| 8 | `ping -c 3 -W 2 <HOME_GATEWAY_IP>` | Timeout, logged | AGENTS #5 | ✅ blocked |
| 9 | `ping -c 3 -W 2 <PROXMOX_TAILSCALE_IP>` | Timeout, logged | AGENTS #5 | ✅ blocked |

Evidence to capture: terminal output next to the matching Live View lines (filter `src = <AGENT_VM_IP>`).
Pass rules 1–2 do not log by default; logging was enabled temporarily on rule 2 to observe the gateway ping.

## 2. Remote SSH enforcement by OPNsense

| Step | Condition | Expected | Result |
|---|---|---|---|
| 1 | Tailscale SNAT **enabled**, OPNsense SSH rule **disabled** | – | ❌ SSH still worked (rule bypassed) |
| 2 | SNAT **disabled**, SSH rule **disabled** | SSH blocked by default deny on TAILSCALE | ✅ |
| 3 | SNAT **disabled**, SSH rule **enabled** | SSH passes, logged with source `<MACBOOK_TAILSCALE_IP>` | ✅ |

## 3. SSH authentication

```bash
ssh asuni@<AGENT_VM_IP>                                  # key login: succeeds
ssh -o PubkeyAuthentication=no asuni@<AGENT_VM_IP>       # password login: "Permission denied (publickey)"
```

## 4. Tailnet policy

Tailscale evaluates the `tests` block in [`configs/tailscale-policy.hujson`](../configs/tailscale-policy.hujson) on every save:
the laptop may reach `<AGENT_VM_IP>:22` but not port 80; a lab VM (`<ADMIN_DEVICE_TAILSCALE_IP>`) may reach neither the agent nor the Proxmox host.

## 5. WAN exposure

From the Proxmox host (which sits on the WAN side):

```bash
curl -vk --max-time 10 https://<OPNsense WAN IP>     # expected: timeout
```

## Re-running

Repeat section 1 after any firewall change, OPNsense upgrade, or new VLAN.
