---
layout: post
title: "Building a Power-Efficient Homelab, Part 6: Networking with OPNsense"
date: 2026-10-28 21:00:00 -0500
categories: homelab
tags: [homelab, networking, opnsense, dns, wireguard, suricata]
excerpt: >-
  How the lab is wired: one OPNsense router, DHCP reservations as the IP
  contract, split DNS for Nginx Proxy Manager, WireGuard for remote access,
  closing port 80, and the Thunderbolt link that never came up.
---

In [Part 1]({% post_url 2026-09-01-building-a-power-efficient-homelab-part-1 %}) I only gave the network one paragraph: an OPNsense box, a couple of switches and some Deco access points. Since then, every part of this series has quietly depended on it. The Talos control plane needs the same IP after a restore, Nginx Proxy Manager needs DNS pointing at it, and I need a VPN to get into the LAN from outside. This post covers the internal network and the few decisions that made the rest of the lab easier.

This one is short on purpose. Most of the work lives in the OPNsense GUI rather than in Git.

### The layout

![Network diagram: the ISP ONT feeds igb1 on the Lanner FW-7551 running OPNsense. igb4 goes straight to the Peladn, igb2 to the rack switch (Dell R610, Intel NUC, i9 workstation, Raspberry Pi 4, EVO-X2), igb3 to the house switch (second Deco, other LAN devices) and igb5 to the main Deco.](/assets/images/cluster-network.jpg)

*How everything connects to the OPNsense router.*

Everything goes through a **Lanner FW-7551** running **OPNsense**. It has 6 Gigabit Ethernet ports and currently I am using one as WAN port and four other as LAN ports:

| OPNsense port | Connected to          | What sits behind it                                                             |
| ------------- | --------------------- | ------------------------------------------------------------------------------- |
| igb1 (WAN)    | ISP ONT               | The internet                                                                    |
| igb2          | 8-port switch (rack)  | Dell PowerEdge R610, Intel NUC, i9 workstation, Raspberry Pi 4 and GMKtec EVO-X2 |
| igb3          | 8-port switch (house) | Second Deco unit, other wired devices                                           |
| igb4          | Peladn                | Directly on the router, no switch in between                                    |
| igb5          | Deco (main unit)      | Wi-Fi for the house                                                             |

The four LAN ports are bridged, so the whole house is one flat `192.168.4.0/24` network. I know VLANs are the "proper" answer, and that is something I will plan for future. But with one person running the lab, a flat network meant one place to look when something could not reach something else, and that mattered more while I was still moving services around every weekend.

The Peladn gets its own port on the router for a simple reason: it runs the Talos control plane, Home Assistant, NPM and Vaultwarden. If a cheap switch dies, I would rather lose the workers than the Peladn.

---

#### Step - 1: Make DHCP reservations the IP contract

I don't set static IPs inside the guests. Every important machine gets its address from a **DHCP reservation on OPNsense**, tied to its MAC address. I have listed some of them in below table.

| Guest | Address | Why it must not change |
|---|---|---|
| Talos control plane (VM 201) | 192.168.4.172 | The kubeconfig and every node trust this address |
| PBS (CT 200, Evo-X2) | 192.168.4.27 | Peladn's backup jobs point at it |
| `hl-media-ai-ops-lxc` (CT 202) | 192.168.4.12 | Port forwards and NPM upstreams |
| `hl-home-ops-lxc` (CT 203) | 192.168.4.13 | NPM lives here |

Reservations helped me save time. When I wrote the failover runbook for restoring Peladn's guests onto the Evo-X2, Proxmox kept the same MAC address on restore. OPNsense handed out the same IP, and the cluster came back without touching a single config file. Because of this, I never had to hard-code an IP in the Talos machine config.

The rule I follow now: **if anything else points at a machine, it gets a reservation the day I create it**. I missed this for PBS, which ran on a plain DHCP lease for months after I moved it to the Evo-X2, and it was only luck that the address never changed.

#### Step - 2: Let OPNsense do DNS

DNS used to be AdGuard Home in a container. In August I moved it to **Unbound** on OPNsense and retired AdGuard. Unbound handles the blocklists, every device already uses the router for DNS, and there is one less container to keep alive. That is also why OPNsense runs on its own box rather than as a VM: if Proxmox is down, the house still has internet and DNS.

The more important job is **split DNS**. My services are reachable from outside as `service.yourdomain.duckdns.org`. DuckDNS points that name at my public IP, and OPNsense forwards port 443 to NPM on `192.168.4.13`.

Inside the house, I don't want traffic going out to the public IP and coming back in. So Unbound has a host override that answers `*.yourdomain.duckdns.org` with `192.168.4.13` directly:

```text
Services → Unbound DNS → Overrides
  Host: *   Domain: yourdomain.duckdns.org   IP: 192.168.4.13
```

Now the same URL works at home and away, with the same Let's Encrypt certificate, and local traffic never leaves the LAN. When I moved NPM from the old main server to the home-ops LXC, this override and the port forward were the only two network changes needed. Both pointed at the old main server (`.55`) before and `.13` after.

Until recently I also forwarded port 80 to NPM, but I closed it in October. All it did was answer with a 301 redirect to HTTPS, and certificates did not need it. NPM renews my Let's Encrypt certificates, a wildcard for the subdomains plus the base domain, through the **DuckDNS DNS-01 challenge**. That proves I own the domain by setting a TXT record, not by answering an HTTP request on port 80. So the open port was just one more thing for scanners to hit. After closing it, port 80 times out from outside, HTTPS works as before, and nothing changed at home because split DNS sends LAN traffic straight to NPM. The one thing to remember: if any certificate uses the HTTP-01 challenge, it would need port 80 reopened, or should be switched to the DNS challenge instead.

One gotcha came with this. Home Assistant behind NPM started returning `400: Bad Request` at random. NPM runs in Docker, and Docker can give each Compose network a different subnet. Home Assistant's `trusted_proxies` only listed one narrow Docker subnet, so it rejected the proxy whenever the IP changed. The fix was to trust the whole Docker range:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 127.0.0.1
    - 192.168.4.0/24
    - 172.16.0.0/12
```

The full file is [configuration.yaml](https://github.com/deepchhaiya/Homelab-ops/blob/main/infrastructure/home-ops-lxc/homeassistant/configuration.yaml).

#### Step - 3: WireGuard for getting in from outside

Not everything should be on the internet. Things like the NPM admin page, the browser-in-a-container (Webtop) and the Proxmox UIs are private. For those I use **WireGuard on OPNsense**, with VPN clients on their own `192.168.100.0/24` subnet.

Two things made this work properly:

1. **The VPN clients use OPNsense as their DNS server.** I learned this the hard way. My laptop's WireGuard profile was using `8.8.8.8`, so on the VPN my own domains resolved to the public IP. Traffic left the tunnel, went out to the internet and came back in through the port forward. Public services still worked, but NPM saw my public IP instead of the VPN subnet, so the access list on the private services blocked me. With DNS pointed at OPNsense, the split DNS override from Step 2 applies on the VPN too.
2. **Private services have an access list in NPM.** It allows only the LAN and the VPN subnet. Anyone on the public internet gets a 403 even though the hostname resolves.

#### Step - 4: Suricata as the tripwire

OPNsense also runs **Suricata**. I turned it on with a basic ruleset and kept it conservative. A homelab IPS that blocks too much is worse than none, because the first time it breaks something I would just turn it off. For now it mostly tells me what is happening rather than stopping it, and its alerts are shipped off the router to Grafana so I get alerted and can look at them. More on that in the next part.

---

### What didn't work: the Thunderbolt link

I wanted a direct Thunderbolt/USB4 cable between the Peladn and the Evo-X2 as a private 10.10.10.x link, so backups to PBS would skip the LAN. On paper both machines support it. In practice the link never came up. Across several BIOS settings and port combinations, neither host saw a peer device, and `dmesg` showed no `thunderbolt-net` events on either side. I concluded it was a firmware limitation and stopped.

It turned out not to matter. The backups and the failover restores run over the normal **2.5 GbE LAN**, and that has been plenty. The lesson I took: try the boring option first, and only chase the fast path once the boring one is actually too slow.

### What I would change next

- **VLANs.** IoT devices, cameras and the lab still share one network with my laptop. Splitting cameras and IoT onto their own VLANs is the biggest gap in this setup, and the Lanner can do it. I just haven't done it yet.
- **Config in Git.** Everything else in this lab is in [Homelab-ops](https://github.com/deepchhaiya/Homelab-ops), but OPNsense lives in its own XML backup. Exporting that config on a schedule, encrypted with SOPS like the other secrets, would close the gap.

### What's next

The router sees everything: every DNS query, every firewall decision and every Suricata alert. In the next part I will show where all of that goes. OPNsense logs, the LXCs and the Kubernetes cluster all feed into Grafana, Loki and VictoriaMetrics running on the Evo-X2, and I will explain why that stack deliberately lives off the machine it is watching.
