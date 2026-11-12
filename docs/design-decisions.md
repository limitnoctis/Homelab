# Design Decisions

Architecture decisions and their trade-offs, measured against the [principles](principles.md). Each entry notes the project in which the decision was made.

---

## Pi-hole replaced by Unbound blocklists and DNS over TLS
*Between Project 01 and 02*

**Before:** Clients asked Unbound on OPNsense, Unbound forwarded to a Pi-hole for ad blocking, and Pi-hole asked public resolvers over plain DNS. My internet provider could read every query, and the chain had an extra host that could fail.

**Decision:** Pi-hole removed. Unbound on OPNsense blocks ads, trackers and malicious domains itself with blocklists, validates answers with DNSSEC and forwards to Quad9 over DNS over TLS.

- **DNS over TLS** encrypts the queries on their way out, so the provider no longer sees which domains are looked up.
- **DNSSEC** doesn't hide anything, but proves that an answer really comes from the domain's owner and wasn't altered on the way.
- **Quad9** additionally refuses to resolve known malicious domains.

**Trade-off:** The queries are now visible to Quad9 instead of my provider. Quad9 states that it doesn't store client IP addresses, but some trust in the upstream resolver remains.

---

## Egress through Squid instead of an update rule
*Project 02*

**Before:** For updates, I briefly enabled a rule for outbound TCP 80 and 443. While it was active, a server could reach any domain. Services with a permanent need, such as CrowdSec hub updates behind a CDN with changing IPs, couldn't be handled this way.

**Decision:** Squid with a domain allowlist in its own network. All server networks may only connect there.

**Rejected:** FQDN aliases in OPNsense (filter by IP only, never worked reliably for me), ASN aliases or all CDN ranges (too broad).

**Cost:** Every new dependency needs an allowlist entry. In return, all egress is visible and justified.

---

## AlmaLinux with rootless Podman
*Project 02*

**Before:** Ubuntu with rootful Docker. A container escape meant root on the VM, and there was no mandatory access control.

**Considered:**

| | Debian | RHEL derivative | Fedora CoreOS / Flatcar |
|---|---|---|---|
| rootless Docker | considered | considered | considered |
| rootless Podman | considered | **chosen** | considered |

**Decision:** AlmaLinux 10 with rootless Podman, SELinux enforcing.
- **AlmaLinux** for SELinux and for Wazuh: on Ubuntu 26.04, Wazuh's vulnerability detection produced many false positives, mostly for kernel CVEs. AlmaLinux is binary compatible with RHEL 10 and has its own vulnerability feed in Wazuh.
- **Podman** because it runs without a daemon and was designed rootless from the start, and because the Wazuh and Checkmk agents keep running on the host.
- `podman-compose` instead of Quadlet, so the existing compose files keep working with few changes.

**Rejected:**
- **Fedora CoreOS:** the Wazuh agent would have to be layered with `rpm-ostree`, every update needs a reboot, and vulnerability detection only sees an ostree system partially. Provisioning via Ignition only pays off with many identical hosts.
- **Flatcar:** no package manager; the Wazuh agent would only run as a container and wouldn't see the host's packages.
- **Debian with rootless Docker:** well supported by Wazuh, but no SELinux isolation for containers.

**Cost:** Wazuh's Docker module and Portainer no longer work. Container monitoring is handled by Checkmk's Podman monitoring. RHEL derivatives don't ship the latest Podman, so I set network isolation myself with `isolate`.

---

## One Linux user per service
*Project 02*

**Before:** After switching to Podman, all containers initially ran under my own account. A compromised container would have reached every other service's data.

**Decision:** Every service gets its own user with a home under `/home/<service>` and its own subuid range. Whoever escapes a container owns no file of any other service. My own account no longer runs containers.

**Rejected:** Homes outside `/home`. Possible, but every service would need its own SELinux rules, which apply automatically under `/home`.

**Cost:** More effort for migration and administration, secrets and images per user. Administration via `machinectl shell <service>@`.

---

## NetBird into the admin network
*Project 02*

**Before:** The NetBird server ran on the same VM as the reverse proxy, in the same Docker network. It manages peers, setup keys and the routes into the internal network. A compromised reverse proxy could have created its own peer with access to every network.

**Decision:** NetBird on `ctadm` under its own user. The reverse proxy only forwards traffic. The old data was deleted from the DMZ right away.

**Cost:** One DMZ → ADM rule for two ports, restricted to the reverse proxy.

---

## CrowdSec LAPI into the admin network
*Project 02*

**Before:** The LAPI, with the decision database, the allowlist and the credentials of every bouncer, ran on the reverse proxy in the DMZ. The argument for that location was connection direction: Grafana queried from the admin network into the DMZ.

**Decision:** CrowdSec split in two. The LAPI runs on `ctadm` under its own user; agent and AppSec stay in the DMZ. Firewall bouncer and agent connect into the admin network, alerts go directly to Loki.

**Trade-off:** The connection direction is reversed. According to the principles, that weighs less than keys in the DMZ. The connection is restricted to one port and the reverse proxy's IP.

**Residual risk:** The agent's machine credentials are still in the DMZ, and machine credentials can also delete decisions via the API. An attacker there could still lift bans. What's no longer in the DMZ: the database itself, the firewall's bouncer keys, allowlist management and the console enrollment.

---

## Checkmk on AlmaLinux
*Project 02*

**Before:** Checkmk ran on Ubuntu 26.04, with the same Wazuh false positives as the container hosts.

**Decision:** A dedicated AlmaLinux 10 VM in the admin network, not on `ctadm`, so monitoring stays independent of the hosts it monitors. Migrated with `omd backup` and `omd restore` on an identical version.
