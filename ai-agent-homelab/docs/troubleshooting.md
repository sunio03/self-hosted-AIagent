# Troubleshooting Case Studies

## Case 1: Tailscale GUI access hit "Default deny" despite a correct rule

**Symptom:** `https://<OPNSENSE_TAILSCALE_IP>` never loaded from the laptop.

| Evidence | Observation | Meaning |
|---|---|---|
| `tailscale status` | `opnsense ... active; direct`, tx 6348 / rx 284 bytes | Tunnel up; packets reach the firewall, almost nothing returns |
| Live firewall log | `TAILSCALE in <MACBOOK_TAILSCALE_IP> → <OPNSENSE_TAILSCALE_IP> block "Default deny"` | The pass rule is not matching |
| Repeated entries | Three blocks in the same second | TCP SYN retransmits: the first packet is dropped |
| `curl -vk https://<OPNSENSE_TAILSCALE_IP>` | Timeout after 75 s | Silent drop (firewall), not a refused connection |
| Rule list | Rule present, enabled, correct fields | The rule definition is not the problem |

**Root cause:** the `tailscale0` interface was assigned but **not enabled**. OPNsense does not load rules for disabled interfaces,
so the rule was visible in the GUI but absent from the packet filter.

**Fix:** enable the interface, save, apply, re-apply rules.

**Lesson:** a rule shown in the GUI is not proof that pf has it. `pfctl -sr` shows the rules actually loaded.

---

## Case 2: Tailscale subnet traffic bypassed the firewall rule

**Context:** OPNsense advertises `<VLAN_AGENTS_CIDR>` to the tailnet so the laptop can SSH to agents. An OPNsense rule on
TAILSCALE allowed only the laptop on port 22.

**Observation:** the log entry for the SSH connection was on `vlan0.20` (AGENTS), direction **out**, with
**source `<VLAN_AGENTS_GATEWAY_IP>`** (the firewall itself) and the automatic label *"let out anything from firewall host itself"*.

**Decisive test:** disabling the OPNsense SSH rule — SSH **still worked**.

**Root cause:** with SNAT enabled (the plugin default), Tailscale re-originated the forwarded connection from the firewall's
own address. pf saw firewall-originated traffic, which the automatic outbound rule allows to any port. The interface rule
on TAILSCALE was never consulted.

**Impact before the fix:** combined with Tailscale's default allow-all policy, any tailnet device could reach any port on
the agent VLAN, and agent logs showed every login as `<VLAN_AGENTS_GATEWAY_IP>` (no attribution).

**Fix:**
1. Replaced the tailnet policy with default deny (laptop only; agents on TCP 22 only).
2. Enabled **Disable SNAT** in the Tailscale plugin (advanced mode).
3. Re-tested: rule disabled → SSH blocked by default deny; rule enabled → SSH passes, logged with source `<MACBOOK_TAILSCALE_IP>`.

**Lessons:**
- Test enforcement by removing the allow rule, not only by confirming that access works.
- Know which component actually makes the forwarding decision; a firewall rule only matters if traffic passes through it as routed traffic.
- Address translation can silently change which rules apply and destroy attribution.

---

## Case 3: Plugin install refused — "Installation out of date"

OPNsense only installs plugins on a fully current system. Updating from 26.7.4 to 26.7.5 resolved it.

## Case 4: Changelog download timeout during update check

`fetch: transfer timed out ... changelog.txz appears to be truncated` only affected the descriptive changelog file.
The package catalogue update succeeded and the upgrade proceeded normally.
