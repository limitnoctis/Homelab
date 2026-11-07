# Changelog

## 2026-11
- Checkmk moved to AlmaLinux 10, Podman monitoring on `ctsrv`
- `ctdmz` built: rootless NPMplus with CrowdSec agent, AppSec and Alloy; old Ubuntu VM deleted
- CrowdSec split: LAPI on `ctadm`, alerts sent directly to Loki
- Real client IPs in every service behind the reverse proxy

## 2026-10-28
- Services on `ctsrv` moved to one user each
- `ctadm` built; Loki/Grafana, Vaultwarden and NetBird moved there

## 2026-10
- Squid egress proxy for all server networks, update rule disabled
- `ctsrv` built, services moved from the Docker VM and the LXCs

## 2026-09
- NPM replaced by NPMplus, CrowdSec AppSec enabled
- CrowdSec switched to a central LAPI, firewall bouncer on OPNsense
- Wazuh agents on every host
- NetBird self-hosted, relay only
