# Services

| Service | Purpose | Host / user | Network | Reachable from |
|---|---|---|---|---|
| OPNsense | Firewall, routing, DNS, NTP, DHCP, CrowdSec firewall bouncer | own VM | all | internal |
| Squid | Egress proxy with domain allowlist | own LXC | PRX | server networks |
| NPMplus | Reverse proxy, certificates, access lists | `ctdmz` / `npmplus` | DMZ | public |
| CrowdSec agent + AppSec | Log analysis, WAF | `ctdmz` / `npmplus` | DMZ | local only |
| Alloy | Ships proxy logs to Loki | `ctdmz` / `npmplus` | DMZ | – |
| CrowdSec LAPI | Decision database, bans | `ctadm` / `crowdsec` | ADM | reverse proxy, OPNsense |
| Loki + Grafana | Security dashboard | `ctadm` / `secmon` | ADM | internal |
| Vaultwarden | Password vault | `ctadm` / `vaultwarden` | ADM | internal |
| NetBird | Mesh VPN, management + relay | `ctadm` / `netbird` | ADM | public via proxy |
| Wazuh | SIEM, agents on every host | own VM | ADM | internal |
| Checkmk | Availability, resources, Podman | own VM (AlmaLinux) | ADM | internal |
| SearXNG | Metasearch | `ctsrv` / `searxng` | SRV | internal |
| CouchDB | Obsidian sync | `ctsrv` / `couchdb` | SRV | internal |
| Paperless-ngx | Document management | `ctsrv` / `paperless` | SRV | internal |
| Gitea | Git server | `ctsrv` / `gitea` | SRV | internal |
| Immich | Photo management | `ctsrv` / `immich` | SRV | internal |
| PBS | Backups | own host | LAN | internal |

"Internal" means: from LAN and VPN only, enforced by access lists in the reverse proxy.
