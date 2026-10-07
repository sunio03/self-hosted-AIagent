# Design Decisions

| # | Topic | Decision | Rationale |
|---|---|---|---|
| D1 | Hosting | Self-host on Proxmox instead of a VPS | Full control of the network path; learning value |
| D2 | Firewall | OPNsense (over pfSense) as a VM | Modern GUI/API, active plugin ecosystem |
| D3 | Policy engine | OPNsense is the only policy engine; Proxmox firewall off on firewall-facing and agent NICs | One place to read rules and logs; simpler troubleshooting |
| D4 | Inside bridge | `vmbr1` VLAN-aware with **no physical NIC** | Inside zones can only reach the outside through OPNsense |
| D5 | VLAN tagging | Tags applied by Proxmox, not inside guests | Guests cannot hop VLANs |
| D6 | Hypervisor placement | Proxmox host never has an address in an agent VLAN | The host controls every VM; a compromised agent must not reach it |
| D7 | Initial GUI access | Temporary runtime-only `<OPNSENSE_TEMP_GUI_IP>` on `vmbr1` plus an SSH tunnel; removed afterwards | WAN never exposed the GUI |
| D8 | Domain | `internal` | Reserved for private use; `.local` conflicts with mDNS |
| D9 | WAN | IPv4 DHCP; IPv6 none; bogon blocking on; RFC 1918 blocking off | WAN sits on a private home network (double NAT) |
| D10 | IPv4-only | IPv6 disabled | An unmanaged IPv6 path could bypass IPv4 rules |
| D11 | Management LAN | Static `<OPNSENSE_LAN_IP>/24`, **no DHCP** | Fail closed: an untagged VM gets no address |
| D12 | Remote access | Tailscale on OPNsense via the plugin | Zero inbound ports; config lives in `config.xml` |
| D13 | Tailscale scope on OPNsense | No tailnet DNS, no accepted routes, no exit node | Firewall keeps control of its own DNS and routing |
| D14 | GUI access rule | TCP 443 from the admin laptop only | Least privilege inside the tailnet |
| D15 | Out-of-band path | Proxmox keeps its own Tailscale | The firewall can always be repaired from the console |
| D16 | Node keys | Expiry disabled for remotely managed servers; kept on for the laptop | Avoid remote lockout; a lost laptop's access still expires |
| D17 | DNS | Unbound recursive resolver with DNSSEC, no upstream forwarders | Privacy and validation; no third-party resolver |
| D18 | DHCP | Dnsmasq for VLANs; its DNS moved to port 53053 | Simple fit; avoids port conflict with Unbound |
| D19 | Gateway and DNS | Each VLAN's interface address serves both | Single control point for routing and name resolution |
| D20 | Forced DNS | Outbound DNS (53) to other resolvers blocked and logged | Preserves logging and filtering; hinders DNS tunnelling. Gap: DoH |
| D21 | Agent isolation | Block firewall and `PRIVATE_NETS` (RFC 1918 + `100.64.0.0/10`), then allow the rest | Agents get internet only |
| D22 | Quarantine | Block all, logged; containment by retagging to VLAN 99 | Contain first, analyse second |
| D23 | Agent hosts | Full VMs (not containers), one per agent where possible | Separate kernel; limited blast radius |
| D24 | Agent OS | Ubuntu Server 24.04 LTS (regular) | Matches most install guides; stable LTS |
| D25 | SSH | Keys only (ed25519 with passphrase), root login off, config in `00-hardening.conf` | Brute-force resistant; avoids Ubuntu config-order trap |
| D26 | Tailnet policy | Default deny; only the admin laptop initiates; policy tests on every save | Prevents lateral movement (e.g. Kali → Proxmox) |
| D27 | Agent SSH path | OPNsense as Tailscale subnet router with **SNAT disabled** | Traffic keeps its real source, passes OPNsense rules, attributable in logs |
| D28 | Agent runtime | Dedicated system user without sudo or login shell (planned) | A compromised agent does not inherit admin privileges |
