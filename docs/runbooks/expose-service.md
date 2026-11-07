# Runbook: Expose a service behind the reverse proxy

Goal: A service on a container host is only reachable through the reverse proxy. Each exposure is limited to one source and one port, on three layers.

## 1. Podman: bind the port to the host IP

In the compose file, publish the port with the host's own IP in front of it, not on all interfaces. The reverse proxy, firewalld and OPNsense only ever see this host-side port. The port inside the container stays internal.

## 2. firewalld: allow only the reverse proxy

Add a permanent rich rule that accepts the port only from the reverse proxy's IP, then reload firewalld:

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<proxy-ip>" port port="<port>" protocol="tcp" accept'
sudo firewall-cmd --reload
```

## 3. OPNsense: allow the connection between networks

On the reverse proxy's network interface, add a pass rule from the reverse proxy to the container host on that port, above the interface's block rule. Several ports of one service go into one port alias.

## 4. Reverse proxy: add the proxy host

Create a proxy host pointing to the host IP and port. Internal services get the access list for LAN and VPN.

Services with their own location blocks, such as gRPC or WebSockets for NetBird, need the target and port changed there as well.

## 5. Test from the inside out

Request the service in three places, in this order:

1. on the container host itself
2. from the reverse proxy
3. in the browser

The first place that fails shows where the problem is:

| Fails on | Cause |
|---|---|
| the container host | the service itself, check its container and logs |
| the reverse proxy | firewalld or the OPNsense rule |
| only in the browser | the proxy host configuration |
