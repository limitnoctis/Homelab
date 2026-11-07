# Principles

These goals and principles have applied since the beginning. Every [design decision](design-decisions.md) is measured against them.

## Goals

- **Security as far as possible:** Every layer is built as if the one in front of it had already failed.
- **Learn enterprise tooling:** Not just install SIEM, IPS, monitoring, firewall segmentation and container isolation, but understand them and run them day to day.
- **Self-host instead of outsource:** Data and control stay with me. No third party has access to devices, keys or content.

## Guideline: security before effort

Least privilege and zero trust are the goals. I weigh every measure against the resources and time it costs, but when in doubt, security wins. Where I do something only for consistency and the security gain is small, I say so.

## 1. Zones based on privilege

A system goes into the zone that matches its rights over other systems, not the value of its data.

- **ADM:** holds keys to other systems. Monitoring, SIEM, password vault, VPN management, the central CrowdSec database.
- **SRV:** applications without rights over other systems.
- **DMZ:** only what talks to the internet itself. Publicly reachable services stay in their own network and are passed through by the reverse proxy.

**Connection direction:** Connections from a more trusted zone into a less trusted one are less critical than the other way round. But direction matters less than where keys are stored. A narrowly scoped connection from the DMZ into the admin network is better than keys in the DMZ.

## 2. One container VM per network

Containers from different zones never share a kernel. A container escape stays inside its zone.

## 3. Explicit allow only

No connection between networks without its own rule naming source IP, destination IP and port. On the container hosts, firewalld enforces the same restriction a second time.

## 4. No unrestricted egress

Servers reach the internet only through a proxy with a domain allowlist, and only the domains they actually need.

## Working method

I use AI assistants for research and to discuss technical alternatives. Decisions, implementation and troubleshooting are my own: I check suggestions against the documentation, test every step individually and reject anything that doesn't fit these principles.
