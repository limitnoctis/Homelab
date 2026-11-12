# Project 04 – Per-Host Egress Allowlists

**Status:** Done
**Period:** November 2026

## Starting point

Since [Project 02](02-migration-almalinux-podman.md), no server reaches the internet directly. All outbound traffic goes through Squid with a domain allowlist.

That allowlist was shared. Every server could reach every domain that any server needed: the reverse proxy could download Gitea releases, the photo server could reach the Wazuh package repository. Egress was controlled, but not per host.

## Goal

Least privilege for outbound traffic: each server may only reach the domains it actually needs for its own job.

## Approach

**Allowlist split by purpose.** The single list became several small lists, one per purpose, for example:

- package repositories per operating system
- container registries
- certificate issuance
- CrowdSec hub and API
- Wazuh feed and packages
- individual applications such as NetBird or Gitea

**Lists assigned per host.** Each server is identified by its IP and only gets the lists it needs. A new dependency on one host no longer opens that domain for every other host.

**Lists kept even without a current user.** A list that no host uses right now stays in place but isn't assigned to anyone. It costs nothing and is ready when a matching host appears.

## Decisions

**Squid stays in its unprivileged LXC.** Moving it to an AlmaLinux VM like the container hosts would have meant a full rebuild for a service that has no containers of its own. I hardened it in place instead.

**No TLS inspection.** Squid decides based on the hostname in the CONNECT request. I decided against inspecting the TLS handshake (SNI) on top of that.

**Monitored like every other host.** Squid has a Checkmk and a Wazuh agent.

## Result

| | Before | After |
|---|---|---|
| Allowlist | one list for all servers | one list per purpose |
| Scope | every server reaches every allowed domain | every server reaches only its own domains |
| New dependency | opens the domain for all servers | opens the domain for one server |
