# pfSense Segmented Firewall Lab

A home-lab project building a three-zone (WAN / LAN / DMZ) network using pfSense Community Edition in VirtualBox, with a firewall ruleset designed and tested to enforce zone isolation.

This project is part of a broader home-lab security portfolio, alongside a [honeypot project] and a [Wazuh SIEM home lab].

## Objective

Demonstrate core defensive networking concepts — network segmentation, default-deny access control, and DMZ architecture — by building an isolated lab environment and proving, with live traffic, that the rules behave as designed.

## Topology

```
                    Internet
                       │
                       ▼
                 ┌───────────┐
                 │  pfSense  │
                 │ Firewall  │
                 │  CE 2.8.1 │
                 └─────┬─────┘
          ┌────────────┼────────────┐
          │            │            │
        WAN          LAN          DMZ
      (NAT)     192.168.1.0/24  192.168.20.0/24
                       │            │
                  Kali Linux   Windows 11 + IIS
                (LAN client)   (DMZ web server)
```

**VirtualBox network adapters (pfSense VM):**
- Adapter 1 — NAT → WAN
- Adapter 2 — Internal Network (`cyber-lab`) → LAN
- Adapter 3 — Internal Network (`cyber-DMZ`) → DMZ

## Build Steps

1. Installed pfSense CE 2.8.1 (amd64) in VirtualBox, assigning WAN (em0) and LAN (em1) interfaces during initial setup.
2. Configured LAN as static `192.168.1.1/24` with DHCP enabled for client devices.
3. Added a third VirtualBox adapter and configured it in pfSense as a static DMZ interface (`192.168.20.1/24`), isolated on its own internal network.
4. Placed a Windows 11 VM (running IIS) on the DMZ to act as a test target representing an externally-facing service.
5. Kept Kali Linux on the LAN to act as a trusted-network test client.

## Firewall Ruleset

| Interface | Rule | Action | Purpose |
|---|---|---|---|
| LAN | Anti-Lockout Rule | Allow | Default pfSense rule preserving admin access to the web UI |
| LAN | BLOCK SSH – Security Test | Block (logged) | Deliberately denies inbound TCP/22 to prove custom block rules take effect and log correctly |
| LAN | Default allow LAN to any | Allow | Lets trusted LAN clients (Kali) reach other zones, including the DMZ |
| DMZ | Block DMZ → LAN | Block | Core DMZ principle — a compromised/exposed DMZ host must never reach the trusted LAN |
| DMZ | Allow DMZ → Internet | Allow | Lets the DMZ host reach the internet (e.g., for updates) without reaching internal resources |

## Testing & Validation

Rather than just configuring rules, each one was tested against live traffic to confirm actual behavior, not just intended behavior:

- **LAN → DMZ (allowed):** `curl http://192.168.20.100` from Kali successfully retrieved the IIS default page, confirming trusted-zone access to the DMZ service works as intended.
- **LAN → SSH (blocked):** `nc -vz -w 3 192.168.1.1 22` from Kali returned `Connection timed out`, confirming the custom SSH block rule is enforced.
- **Firewall logging:** The blocked SSH attempts appear in `Status → System Logs → Firewall`, tied to the correct rule (`BLOCK SSH - Security Test`), confirming logging is functioning and traceable to the rule that triggered it.
- **DMZ → Internet (allowed) / DMZ → LAN (blocked):** Rule hit counters on the DMZ tab confirm both rules are actively matching traffic in the expected direction.

## Key Findings

- pfSense's built-in default rules (e.g., "Default allow LAN to any") do **not** log traffic by default — logging must be explicitly enabled per rule. This was discovered while troubleshooting why known-successful traffic wasn't appearing in the firewall log, and is a good example of a default that shouldn't be assumed in a production environment.
- Verified end-to-end that a deny rule (SSH block) and an allow rule (DMZ web access) both produce the expected, logged result under real traffic — not just a passing configuration check.

## Future Work

- **SIEM integration:** Planned to forward pfSense firewall logs to an existing Wazuh SIEM instance via remote syslog (UDP/514). Confirmed the transport layer was working — packets were verified arriving at the Wazuh listener using `tcpdump` — but did not complete event parsing/indexing into searchable Wazuh alerts before time constraints. Next step would be installing/writing a pfSense-specific decoder and ruleset for Wazuh so forwarded logs are parsed into structured, alertable events rather than only being visible in the raw archive log.
- Add VLAN-based segmentation on a single interface as an alternative to the current physical/interface-based zone separation.
- Expand the DMZ ruleset to allow only specific ports/services rather than relying on broader default rules.
