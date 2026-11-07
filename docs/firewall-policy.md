# Firewall Policy

How rules are built in OPNsense and on the hosts. Which connections between networks are allowed is listed in [network segmentation](network-segmentation.md), every single rule in [firewall rules](../examples/firewall-rules.md).

## Basic rules

- **Default deny.** Every interface ends with a block rule that logs. Only what has its own rule above it is allowed.
- **One rule per purpose.** Every rule names source, destination and port. Whole networks as source or destination only where every host is really meant, such as DNS and NTP to the firewall.
- **Rules on the ingress interface.** A rule sits on the interface of the network the connection comes from.
- **IPv6 blocked** on every interface, since none of the networks uses it.

## Administration

- **SSH** to the servers is only allowed from admin devices in the LAN and from trusted VPN peers.
- **Firewall and hypervisor web interfaces** are reached directly, from the same devices only.
- **All other web interfaces** go through the reverse proxy.

## Outbound to the internet

- Servers have no rule to the internet. Their only way out is the egress proxy in its own network.
- The egress proxy itself may only reach web ports on public addresses, never private networks. What a service may reach outside is controlled by the proxy's domain allowlist.
- Client devices in LAN and VPN that need to browse are grouped and may reach the internet directly. An allowlist would make normal browsing impossible there.
- Servers in the LAN, such as the NAS, use the egress proxy for updates.

## Inbound from the internet

- One forwarded port to the reverse proxy in the DMZ, and the WireGuard port on the firewall itself.
- The WAN drops traffic from:
  - IPs with an active CrowdSec ban, on all ports
  - bogon addresses
  - hosts locked out after too many login attempts
  - hosts opening too many connections at once
- The firewall itself and the WireGuard network reject outgoing connections to banned addresses.

## On the container hosts

In addition to OPNsense, firewalld filters on every container host:

- Every published port is a rich rule that only admits the reverse proxy's IP.
- Containers bind their ports to the host IP, not to all interfaces.

If one of the two firewalls is missing, the other still holds. See the runbook [Expose a service](runbooks/expose-service.md).

## Changes

- **Disable first, delete later.** After a rebuild, an old rule is disabled first and only deleted once nothing is missing.
- **No permanent temporary rules.** Rules needed only occasionally, like a former update rule for clients, stay disabled until used. Anything needed all the time gets its own narrowly scoped rule.
- **Keep the IP when replacing a host,** where possible. Rules then remain valid unchanged.
