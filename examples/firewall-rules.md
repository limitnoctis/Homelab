# Firewall Rules

All OPNsense rules per interface, as pseudocode. Rules are evaluated top to bottom, the first match wins.

Ports are replaced by service names, addresses by role names. Rules for individual personal apps (messengers, push notifications) are left out. The principles behind the rules are in the [firewall policy](../docs/firewall-policy.md).

## WAN

```text
block   ipv6 any
pass    in   wireguard                      -> firewall
block   in   from bruteforce-lockout        -> firewall   [ssh, https]
block   in   from connection-flood-list     -> any
block   in   from crowdsec-banlist          -> any
block   in   from bogons                    -> any
reject  out  firewall                       -> crowdsec-banlist, connection-flood-list
pass         dhcp                           <-> isp-router
pass    out  firewall                       -> internet
nat     in   https                          -> reverse-proxy (ctdmz)
block   any                                                   log
```

## LAN

```text
block   ipv6 any
reject  from lan                     -> connection-flood-list
pass    from admin-devices           -> firewall-gui, proxmox-gui, router-gui
pass    from admin-devices           -> all-servers        [ssh]
pass    from admin-devices           -> reverse-proxy      [ssh, proxy-admin]
pass    from admin-devices           -> ctsrv              [git-ssh]
pass    from lan                     -> egress-proxy
pass    from lan-servers             -> wazuh              [agent]
pass    from lan-servers             -> checkmk            [agent-registration]
pass    from pentest-vm              -> any                # temporary, to be restricted
pass    from browsing-clients        -> internet           [https]
pass    from lan                     -> firewall           [dns]
pass    from lan                     -> any                [icmp]
block   any                                                   log
```

## WireGuard

```text
block   ipv6 any
reject  from wireguard               -> crowdsec-banlist, connection-flood-list
pass    from trusted-wg-peers        -> firewall-gui, proxmox-gui
pass    from trusted-wg-peers        -> firewall           [ssh]
pass    from trusted-wg-peers        -> all-servers        [ssh]
pass    from trusted-wg-peers        -> reverse-proxy      [proxy-admin]
pass    from wireguard               -> firewall           [dns]
pass    from wireguard               -> any                [ntp]
pass    from browsing-wg-peers       -> internet           [https]
pass    from wireguard               -> any                [icmp]
block   any                                                   log
```

## NetBird

```text
pass    from netbird-peers           -> firewall           [dns]
pass    from netbird-peers           -> firewall-gui
pass    from netbird-peers           -> reverse-proxy      [proxy-admin]
pass    from netbird-peers           -> internet           [https]
pass    from school-peer             -> all-servers        [ssh]
pass    from school-peer             -> reverse-proxy      [https]
pass    from school-peer             -> proxmox-gui
pass    from school-peer             -> ctsrv              [git-ssh]
block   any                                                   log
```

## DMZ

```text
pass    from reverse-proxy           -> firewall           [dns, ntp]
pass    from reverse-proxy           -> egress-proxy
pass    from reverse-proxy           -> ctsrv              [gitea, paperless, couchdb, immich, searxng]
pass    from reverse-proxy           -> ctadm              [vaultwarden, grafana, netbird]
pass    from reverse-proxy           -> ctadm              [loki, crowdsec-lapi]
pass    from reverse-proxy           -> proxmox-gui, checkmk-gui, wazuh-gui
pass    from reverse-proxy           -> wazuh              [agent]
pass    from reverse-proxy           -> checkmk            [agent-registration]
block   any                                                   log
```

## Admin network (ADM)

```text
pass    from adm                     -> firewall           [dns, ntp]
pass    from adm-servers             -> egress-proxy
pass    from checkmk                 -> all-servers        [checkmk-agent]
block   any                                                   log
```

## Server network (SRV)

```text
pass    from srv                     -> firewall           [dns, ntp]
pass    from ctsrv                   -> egress-proxy
pass    from srv-servers             -> wazuh              [agent]
pass    from srv-servers             -> checkmk            [agent-registration]
pass    from srv                     -> any                [icmp]
block   any                                                   log
```

## Proxy network (PRX)

```text
pass    from egress-proxy            -> firewall           [dns]
pass    from egress-proxy            -> not private-nets   [http, https]
block   any                                                   log
```

## Floating

```text
pass    from any                     -> firewall           [ntp]
```
