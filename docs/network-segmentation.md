# Network Segmentation

Which networks exist and which connections between them are allowed. How the rules are built is described in the [firewall policy](firewall-policy.md), every single rule in [firewall rules](../examples/firewall-rules.md).

## Three layers

1. **OPNsense** between networks. Every rule names source, destination and port; everything else is blocked and logged.
2. **firewalld** on every container host. Every published port only accepts connections from the reverse proxy's IP.
3. **Podman.** Ports bound to the host IP instead of `0.0.0.0`, networks with `isolate`. Each service user already has its own network namespace. I still set `isolate` everywhere; the gain is small there, it's about a consistent setup.

## Allowed connections between networks

| From | To | Purpose |
|---|---|---|
| Internet | DMZ (`ctdmz`) | HTTPS to the reverse proxy |
| Internet | Firewall | WireGuard |
| DMZ (`ctdmz`) | SRV (`ctsrv`) | Reverse proxy to the services |
| DMZ (`ctdmz`) | ADM (`ctadm`) | Reverse proxy to Vaultwarden, Grafana, NetBird; log shipping to Loki; agent to CrowdSec LAPI |
| DMZ (`ctdmz`) | ADM (Wazuh, Checkmk) | Reverse proxy to their web interfaces; agent traffic |
| DMZ (`ctdmz`) | LAN (Proxmox) | Reverse proxy to the hypervisor web interface |
| ADM (Checkmk) | all servers | Agent polling |
| all servers | ADM (Wazuh, Checkmk) | Agent enrollment and events |
| Server networks, LAN servers | PRX (Squid) | Updates; the servers' only way to the internet |
| PRX (Squid) | Internet | Web ports on public addresses, filtered by the domain allowlist |
| LAN, VPN | DMZ (`ctdmz`) | Internal services through the reverse proxy |
| Admin devices, trusted VPN peers | Firewall, Proxmox, all servers | Web interfaces of firewall and hypervisor; SSH |

Because firewalld only opens the container hosts' published ports to the reverse proxy, their web interfaces are only reachable through it.

## Egress

- All server networks reach the internet only through Squid with a domain allowlist.
- The allowlist is assigned per host: each server only reaches the domains it needs itself, see [Project 04](../projects/04-squid-per-host-allowlists.md).
- Squid checks the requested hostname, for HTTPS via the CONNECT request. A firewall rule with an FQDN alias would only allow the IPs behind a name, and behind a CDN those are shared with many unrelated domains.
- Package repositories (dnf, EPEL) point to fixed servers so that a few domains are enough.
- Client devices browse directly, because an allowlist would make normal browsing impossible.
