# Project 02 – Migration to AlmaLinux and Rootless Podman

**Status:** Done. Podman monitoring in Checkmk runs on `ctsrv`; `ctadm` and `ctdmz` are pending.
**Period:** October to November 2026

## Starting point

A review of [Project 01](01-ubuntu-docker.md#weaknesses-that-led-to-project-02) showed that network segmentation alone wasn't enough: rootful Docker, all services under one user, keys in the DMZ, no mandatory access control, egress by switch, and false positives in Wazuh that buried real findings.

## Result

| | Project 01 | Project 02 |
|---|---|---|
| Container host | Ubuntu + rootful Docker | AlmaLinux 10 + rootless Podman |
| Mandatory access control | none | SELinux, every volume labeled |
| Isolation between services | one user for all containers | one Linux user per service |
| Host firewall | OPNsense only | plus firewalld, each port restricted to one source IP |
| Egress | update rule toggled by hand | Squid with domain allowlist |
| NetBird | in the DMZ | in the admin network |
| CrowdSec LAPI | in the DMZ | in the admin network; DMZ runs agent + AppSec only |
| Container monitoring | Portainer | Podman monitoring in Checkmk |

## Approach

Each step fixes a specific weakness. The reasoning behind each choice is in the [design decisions](../docs/design-decisions.md).

1. **Egress through Squid.** *Fixes: egress by switch.* Servers no longer get a temporary open door to the whole internet, but permanent access to exactly the domains they need, and nothing else.
2. **New container host `ctsrv`.** *Fixes: rootful Docker, no mandatory access control, Wazuh false positives.* AlmaLinux with rootless Podman and SELinux. A container escape no longer means root on the VM, and Wazuh findings are usable again.
3. **One Linux user per service.** *Fixes: all services under one user.* A compromised container now only reaches the files of its own service.
4. **NetBird into the admin network.** *Fixes: keys in the DMZ.* The VPN control plane, with peers, keys and routes into every network, no longer sits next to the reverse proxy.
5. **CrowdSec split in two.** *Fixes: keys in the DMZ.* The decision database and the firewall's bouncer keys moved to the admin network. Only the agent remains in the DMZ, with an accepted residual risk.
6. **New reverse proxy host `ctdmz`.** *Fixes: rootful Docker in the DMZ.* NPMplus rootless with AppSec. Cut over by taking the old VM's IP, so every firewall rule stayed valid.
7. **Checkmk on AlmaLinux with Podman monitoring.** *Fixes: Wazuh false positives on the last Ubuntu VM, and the loss of Portainer.* Portainer doesn't work with rootless Podman, so container monitoring moved into Checkmk. Its Podman plugin queries the Podman API of every service user through that user's own socket, so Checkmk sees the containers of all services on a host while each of them keeps running rootless under its own user. Container status shows up next to the host's own checks.

One step at a time, each tested before the next. Old VMs kept running until the test passed and were deleted only afterwards.

**The tricky part:** each service user has its own UID range, so data can't simply be copied from one user to another. I packed and unpacked it from the container's point of view (`podman unshare tar`, `podman volume export/import`), so ownership stayed correct without reassigning permissions. Details in the [runbooks](../docs/runbooks/).

## What if someone gets in?

A walk through an attack on the setup, from the outside in. At each step: what stops or slows the attacker, and where the limit is.

**1. Reaching the homelab**
- Only HTTPS to the reverse proxy and the WireGuard port are open from the internet.
- IPs on the CrowdSec blocklist, bogons and hosts flooding connections are dropped at the firewall before they reach anything.
- Almost every service behind the proxy is only reachable from LAN and VPN. From the internet, NetBird is the only service exposed.

**2. Attacking the reverse proxy**
- CrowdSec AppSec inspects every request as a WAF. Scans and repeated WAF hits lead to bans that the firewall enforces on all ports.
- The proxy's admin interface is not reachable from the internet, only from admin devices and trusted VPN peers.
- *Limit:* an unknown vulnerability in NPMplus or a proxied service gets through the WAF.

**3. Code execution in a container**
- The container runs rootless. Root inside it is the unprivileged service user on the host.
- SELinux confines the container process and its labeled volumes.
- *Limit:* on `ctdmz`, NPMplus, the CrowdSec agent and Alloy run under the same user. Whoever controls NPMplus also controls the agent and could use its credentials to lift bans.

**4. Escaping to the host**
- The attacker lands as an unprivileged user without sudo, owning no file of any other service.
- No keys of value lie on `ctdmz`: no CrowdSec database, no VPN keys, no password vault.
- The Wazuh agent on the host watches files, processes and rootkits.
- *Limit:* a kernel exploit that defeats both user namespaces and SELinux gives root on the VM. It's still only the DMZ.

**5. Moving sideways**
- From the DMZ, only specific ports on specific hosts are open, and firewalld on each container host checks a second time.
- No SSH from the DMZ to anywhere. SSH comes only from admin devices and trusted VPN peers.
- Every further service has to be exploited separately and again only gives one service user.
- Blocked attempts are logged by the DMZ's block rule.

**6. Getting data out or calling home**
- No server has a direct route to the internet. A reverse shell to the attacker's server fails.
- The only way out is Squid, and Squid only allows the domains on the allowlist.

**7. Covering tracks**
- Logs leave the host right away: Alloy ships proxy logs to Loki in the admin network, CrowdSec alerts go there directly, Wazuh collects host events centrally.
- An attacker on one host can't delete what's already stored elsewhere.

**8. Recovering**
- The VMs are backed up with PBS, so a compromised host can be restored to a known state.

## Open points

- **Pentest VM.** A pentest VM in the LAN still has an allow-any rule. It will be restricted to the networks and ports actually needed.
- **CrowdSec agent credentials.** The agent's machine credentials in the DMZ can still delete bans through the API, see [CrowdSec LAPI into the admin network](../docs/design-decisions.md#crowdsec-lapi-into-the-admin-network).
- **Proxy admin interface.** The NPMplus admin interface with Node.js still runs in the DMZ. It's only reachable from admin devices and trusted VPN peers.
