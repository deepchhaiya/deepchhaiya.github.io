---
layout: post
title: "Building a Power-Efficient Homelab, Part 1: Overview"
date: 2026-09-01 22:44:00 -0500
categories: homelab
tags: [homelab, proxmox, talos, kubernetes, power-efficiency, networking]
excerpt: >-
  How I turned a pile of mismatched machines I already owned into a
  power-efficient Proxmox + Talos cluster — the hardware, the OS stack, a DAS
  that kept disconnecting, and the rack that finally cooled it all.
---

I wanted to build a homelab after getting inspired by a handful of prominent YouTube channels — [Mischa van den Burg](https://www.youtube.com/@mischavandenburg), [Hardware Haven](https://www.youtube.com/@HardwareHaven), and a few others. But inspiration is cheap; the real project was figuring out what to do with the pile of machines I already had lying around.

That constraint ended up shaping everything. Instead of buying one big server, I set out to build a **power-efficient homelab** that could run a mix of Kubernetes and Docker (more on that decision later), support a **hybrid AI setup** — self-hosted models alongside the well-known hosted ones — and do it all with minimal overhead. This post covers how I got from "one loud desktop" to a small cluster, and the decisions along the way.

## Where I Started

The first version was about as simple as it gets: a raw Ubuntu install on an aging i7 (2nd gen) with 24 GB of DDR3 RAM and roughly 10 TB of storage behind an LSI 9205-8i SAS HBA (H220, 6 Gb/s, dual SFF-8087 connectors) and a GTX 1650 Ti for anything that needed a GPU.

On top of that I ran CasaOS with a handful of Docker containers — NextCloud, Gotify, Jellyfin, Immich, Vaultwarden, Beszel and few others. It worked fine for a while, but the cracks showed as soon as I wanted to run n8n and Ollama on the same box. Additionally the idea of learning and exploring Kubernetes and improving power efficiency led me to start thinking about newer architecture.

So I upgraded: an i9-14900KF, 32 GB of DDR5, and an RTX 5070, still running Ubuntu with Docker, now pushing about 13 TB across SSDs, HDDs, and external drives. It was faster, but it introduced two new problems. First, the power draw jumped noticeably for a single always-on machine. The idle power draw was about 110 W and jumped up to 500 W when GPU was heavily utilized. Second, that one CPU was now a single point of failure for everything I self-hosted — and its memory ceiling became obvious the moment I tried loading models north of 24B parameters.

That was the turning point. I had several other PCs sitting idle, so instead of buying yet another bigger box, I started thinking about turning what I already owned into a cluster — spreading the load across small, efficient machines that could run 24×7 without spiking my power bill.

The chart below is the lab's power draw, tracked in Home Assistant from a smart plug and from the UPS — the UPS is monitored by a [NUT](https://networkupstools.org/) service running on the cluster.

![Home Assistant chart of the homelab's power consumption over time, pulled from a smart plug and a NUT-monitored UPS.](/assets/images/power-usage-home-assistant.jpg)

The honest accounting: the old single node idled at about 110 W. The new always-on core — three machines, a proper 3-2-1 backup, better support for larger AI models, and 96 GB of unified memory — idles at about 120 W. So in raw watts I didn't really come out ahead. What I got instead is a dynamic, decentralized lab that does far more per watt: the heavy compute nodes stay powered off until I need them, and nothing is a single point of failure anymore.

## Decision 1: Cluster What I Already Own, and Keep It Power-Efficient

Here's the inventory I was working with — a genuinely mismatched set of mini PCs, tower CPUs, and one enterprise leftover:

- **Peladn mini PC** — AMD Ryzen 7 7840HS w/ Radeon 780M Graphics (8c/16t), 32 GB RAM, 1 TB SSD. Running Proxmox with an Ubuntu Server VM, plus a 26 TB external HDD attached for weekly backups from the main server.
- **Main server** — i9-14900KF, 32 GB DDR5, Nvidia RTX 5070, ~13 TB across SSDs, HDDs, and external storage, running Ubuntu with Docker.
- **Intel NUC** — i3-4010, 8 GB RAM, 256 GB SSD.
- **Windows PC** — 12th Gen Intel Core i7-12700K (12c/20t), 32 GB RAM, Nvidia RTX 3070, 1 TB SSD.
- **Dell R610** — dual Intel Xeon E5645 @ 2.40 GHz (12c/24t combined), ~80 GB DDR3, ~1.2 TB SAS HDDs, running Proxmox.
- **Raspberry Pi 4** — 4 GB RAM, tower cooler, 32 GB microSD, with 128 GB external SSD.

The plan: make the Pi 4 and the Peladn my 24×7 machines, and use everything else as worker nodes that could be powered on when I actually needed the extra compute. That solved the power problem, but it exposed a new gap — none of these machines had enough memory to comfortably run larger models at a decent token rate.

After some research and budget math, I picked up a **GMKtek EVO-X2** with 96 GB of unified memory to anchor the cluster's AI workloads.

With that decided, I hit the next problem: my main server was no longer a 24×7 machine, which meant I needed to rethink storage entirely — and put a real **3-2-1 backup strategy** in place instead of hoping nothing failed. The Peladn had no clean way to attach multiple HDDs over PCIe, but it did have a USB-C Gen 3 Thunderbolt port, so I added a DAS with 11 TB of drives as the primary array, plus a 26 TB external Seagate HDD for periodic backups connected to Evo-x2 which later I decided to include as my alternate control node and 24 x 7 machine.

![Hardware inventory: six reused machines plus the purchased EVO-X2, grouped into always-on 24×7 nodes and on-demand worker nodes, with an 11 TB DAS on the Peladn and a 26 TB drive on the EVO-X2 for backups.](/assets/images/hardware-inventory.svg)
*The fleet at a glance — what stays on, what gets woken up, and where the storage hangs.*

## Decision 2: Picking the OS Stack

I wanted an OS setup that made backups straightforward and kept Docker overhead to a minimum. After comparing options, I landed on:

- **Proxmox** as the host OS across the fleet except Raspberry Pi 4
- **Talos** for Kubernetes — an immutable, API-managed OS built specifically to run Kubernetes with no SSH access and no package manager to babysit
- **Debian LXCs** for anything better suited to plain Docker

This combination let me run microservices on Kubernetes while keeping a few essential containers close to the metal in LXCs, minimizing network hops for the things that didn't need to be orchestrated.

I later added a Proxmox Backup Server VM to handle incremental backups onto the 26 TB HDD, retaining 30 daily, 12 monthly, and 1 yearly snapshot.

## Decision 3: Storage and Chasing Down a DAS That Kept Disconnecting

Once the storage was in place, I ran into the most frustrating bug of the whole build: the Wavlink 2-Bay HDD DAS with 2 12TB Toshiba N300 pro HDDs, connected through an ASMedia USB RAID controller, kept intermittently disconnecting from Proxmox. No obvious cause, no consistent trigger — just random drops.

After digging into it and checking the disconnect pattern, the culprit turned out to be heat: the controller chip had no heatsink. I added one, and the disconnects vanished — transfer speeds settled in around 300 MB/s. A DIY solution using five-dollar heatsink fixed a problem that could have easily passed for a driver or cabling issue.

Separately, I had a spare 128 GB M.2 SSD and an M.2-to-USB enclosure sitting around, so I put that to use as the Pi 4's main drive instead of relying solely on microSD.

With that sorted, the cluster came together: every machine runs a Talos VM on top of Proxmox, except the Pi 4, which runs Talos directly on bare metal.

## Decision 4: Building a Rack That Could Actually Cool Everything

Six machines crammed into a closet need airflow, not just a shelf. I 3D-printed a rack based on models from [kellerlabs/homeracker](https://github.com/kellerlabs/homeracker), along with a custom Pi 4 case: a front intake fan pulls air in and pushes it out the back, with an exposed side window for an ice tower fan to pull heat off the CPU — enough headroom that I could bump CPU and GPU frequencies in the config for sustained 24×7 operation.

![3D-printed homelab rack with six machines and a custom Raspberry Pi 4 case](/assets/images/homelab-rack.jpg)

I also printed a set of ducts to route airflow for the mini PCs, and repurposed two 90 mm fans salvaged from a dead HP Z420 (failed motherboard) alongside an ESP32 controller to build a venting system: it ramps up to full speed when the HVAC kicks in to pull in cold air, then drops to a lower speed to push warm air out toward the pantry door the rest of the time.

![ESP32-controlled venting system built from salvaged 90 mm fans](/assets/images/homelab-venting.jpg)

![Circuit diagram for the ESP32 fan controller](/assets/images/fan-controller-circuit.png)
*Circuit diagram for the ESP32-based venting controller.*

## Decision 5: Network and Security

None of this matters if the network underneath it is an afterthought. I picked up a **Lanner SED7551** off eBay and installed **OPNsense** on it as the main router for the whole house. That gave me proper firewall rules plus **Suricata** for IDS/IPS, DNS-level ad and threat blocking through **Unbound**, and five bridged ports feeding an 8-port switch in the rack. **Deco** units handle Wi-Fi as access points, with 2.5 Gbps LAN throughout. More details in later dedicated section.

## Where Things Stand Now

That's the full parts-and-decisions story: six mismatched machines turned into a Proxmox + Talos cluster, a DAS-backed 3-2-1 backup setup, a 3D-printed rack that actually stays cool, and an OPNsense-based network with proper IDS/IPS in front of it all — built almost entirely from hardware I already owned, plus one deliberate purchase (the EVO-X2) to close the memory gap for bigger models.

The parts list and the plumbing were the easy part in hindsight. The next post in this series will get into the actual Kubernetes and hybrid-AI workload setup — how workloads get scheduled across such an uneven fleet, and what running self-hosted models alongside hosted ones looks like in practice. If you're working through a similar "I have too many old machines and one good idea" problem, I'd genuinely like to hear how you approached it.
