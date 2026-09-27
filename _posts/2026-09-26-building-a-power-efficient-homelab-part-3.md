---
layout: post
title: "Building a Power-Efficient Homelab, Part 3: Creating LXCs and Setting Up the Talos Cluster"
date: 2026-09-26 20:30:00 -0500
categories: homelab
tags: [homelab, proxmox, lxc, talos, kubernetes, flux, gitops, sops]
excerpt: >-
  The always-on and on-demand tiers, the two Docker LXCs, the Talos control
  plane and Raspberry Pi worker, Flux CD, and what I got wrong along the way.
---

My main goal was using Peladn as my main 24/7 "Base Station", Raspberry Pi 4 as main monitoring server and later Evo-X2 as my main AI machine with the Intel NUC, Dell R610 and i9 Server as "On-Demand Reinforcements" via Wake-on-LAN (WOL) to mimic the production type power efficient environment.

Below is the blueprint where eventually I divided 30+ containers across different hardwares to maximize the usage of Radeon 780M (Peladn), Radeon 8060S (Evo-X2) and the NVIDIA GPU (i9 Server).

```text
Tier 1 — Always on (24x7)
  Peladn (192.168.4.150)  Ryzen 7 7840HS / 32 GB / Radeon 780M
    VM 201   Talos control plane          192.168.4.172
    CT 202   media-ai-ops (Docker)        192.168.4.12   -> Nextcloud, Immich, Jellyfin, Frigate
    CT 203   home-ops (Docker)            192.168.4.13   -> Home Assistant, NPM, Vaultwarden, Gotify
    11 TB Wavlink DAS mounted on the host, bind-mounted into both LXCs

  Evo-X2 (192.168.4.84)   Ryzen AI Max+ 395 / 96 GB / Radeon 8060S
    Ollama on the host (ROCm)             qwen3.6:35b-a3b
    CT 200   Proxmox Backup Server        192.168.4.27
    CT 405   observability                192.168.4.66   -> VictoriaMetrics, Loki, Grafana
    VM 402   Talos worker (tier=ai-worker) 192.168.4.71

  Raspberry Pi 4 (192.168.4.141)  bare-metal Talos worker (tier=always-on)

Tier 2 — On demand, woken by WOL (tier=on-demand)
  Intel NUC (192.168.4.151) | Dell R610 (192.168.4.70) | i9 + RTX 5070 (192.168.4.170)
```

A fair warning before the steps: this blueprint is *not* what I planned in March. The original plan had a dedicated storage LXC, the Intel NUC as the always-on worker, and the i9 as the AI machine. All three of those turned out to be wrong, and I have written at the end of this post what replaced them and why. I would rather show the working version along with the detours than pretend it came out clean on the first attempt.

Every config file I mention is in the public repo — [Homelab-ops](https://github.com/deepchhaiya/Homelab-ops). If you are doing something similar, the full sequence with verification steps at each stage is in [docs/homelab-build-guide.md](https://github.com/deepchhaiya/Homelab-ops/blob/main/docs/homelab-build-guide.md).

#### Step - 1: Deciding what runs in Docker and what runs in Kubernetes

This was the first real decision and it saved me a lot of pain later. My rule ended up being simple:

- If the thing needs hardware attached to a specific machine — the Zigbee USB stick, the iGPU for transcoding, the UPS cable — it runs in a **Docker LXC** on that machine.
- If the thing is a stateless or semi-stateless service that I want scheduled, restarted and upgraded by the platform, it runs in **Kubernetes**.

I had originally planned three separate GPU LXCs and consolidated them into one. Three containers fighting over the same `/dev/dri` device with three different compose files is a maintenance tax for no benefit. So, two LXCs:

| LXC                          | What it carries                                                             | Why here and not in K8s                                                                                                  |
| ---------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| CT 202 `hl-media-ai-ops-lxc` | Nextcloud, Immich, Jellyfin, Frigate                                        | needs `/dev/dri` (780M) for transcoding and a direct path to the DAS                                                     |
| CT 203 `hl-home-ops-lxc`     | Home Assistant, Nginx Proxy Manager, Vaultwarden, Mosquitto, Gotify, UpSnap | Zigbee USB passthrough, and NPM acts as proxy to route traffic from internet for all services and needs port 80 and 443. |

Both compose files are in the repo — [media-ops-lxc](https://github.com/deepchhaiya/Homelab-ops/tree/main/infrastructure/media-ops-lxc) and [home-ops-lxc](https://github.com/deepchhaiya/Homelab-ops/tree/main/infrastructure/home-ops-lxc).

#### Step - 2: Creating the two Debian LXCs

I used the `debian-13-standard` template from the Proxmox UI (**CT Templates > Templates**), and created both containers **unprivileged** with nesting enabled. Unprivileged is worth insisting on — a privileged container shares root with the host, and there was no reason to hand that over for a Docker host.

You can use the Templates button under CT Templates to explore available templates or can upload your own.

![Proxmox VE 9.2 web UI on the Peladn node showing the CT Templates page, with the Templates button and the debian-13-standard template listed.](/assets/images/proxmox-ct-templates.png)
*Proxmox CT Templates, with the debian-13-standard template.*

Using Shell:

```bash
# On the Peladn Proxmox host
pveam update
pveam download local debian-13-standard_13.1-2_amd64.tar.zst
```

CT 202 (media and AI) and CT 203 (home ops), as they actually run today:

| | CT 202 | CT 203 |
|---|---|---|
| Hostname | `hl-media-ai-ops-lxc` | `hl-home-ops-lxc` |
| Cores / Memory | 6 / 16 GB | 3 / 4 GB |
| Rootfs (local NVMe) | 250 GB | 50 GB |
| IP | 192.168.4.12/24 | 192.168.4.13/24 |
| Unprivileged | Yes | Yes |
| GPU | `/dev/dri` passed through | not needed |

You can use the configuration wizard to set up the LXCs.

![Proxmox Create: LXC Container wizard on the General tab, with Unprivileged container and Nesting both ticked and the Create CT button highlighted.](/assets/images/proxmox-create-lxc-wizard.png)
*LXC creation wizard: unprivileged, with nesting enabled.*

The part that is not in the creation wizard is the config file. After creating them I stopped both and edited `/etc/pve/lxc/202.conf`:

```bash
features: nesting=1
mp0: /mnt/pvedas,mp=/mnt/das
mp1: /mnt/sg-ext-hdd,mp=/mnt/sg-ext-hdd
mp2: /mnt/wd-ext-hdd,mp=/mnt/wd-ext-hdd
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
```

`nesting=1` is what allows Docker to run inside. `c 226:*` is the DRM/render device class, which is what Jellyfin and Frigate need for hardware acceleration. The `mp0` to `mp2` lines are the storage bind mounts — the DAS and two external drives are mounted on the **Proxmox host** and handed into the container, not mounted inside it. How the DAS itself is set up on the host is the whole of [Part 4]({% post_url 2026-09-26-building-a-power-efficient-homelab-part-4 %}), so I will not repeat it here. The live configs are committed as [`202 (hl=media-ai-ops).conf`](https://github.com/deepchhaiya/Homelab-ops/tree/main/infrastructure/media-ops-lxc) and [`203 (hl-home-ops-lxc).conf`](https://github.com/deepchhaiya/Homelab-ops/tree/main/infrastructure/home-ops-lxc).

Then the usual inside each container:

```bash
pct start 202
pct enter 202

apt update && apt install -y docker.io docker-compose curl git
systemctl enable --now docker

ls -la /dev/dri/          # card0 and renderD128 should be there
ls /mnt/das               # the DAS, via bind mount
```

**Note:** *one thing that cost me an evening later is that in an unprivileged LXC, container UID 0 maps to host UID 100000. So a `chown 1000:1000` that you run inside the container is not the ownership the host sees. Every permission fix for bind-mounted paths has to be run on the Proxmox host with the mapped UID, meaning `101000:101000` for container user 1000. Doing it from inside looks like it worked and then the container still cannot write.*

#### Step - 3: The Talos control plane VM on Peladn

Talos is distributed as an image you boot, not an installer you click through. I built mine at [factory.talos.dev](https://factory.talos.dev) — platform **nocloud** for Proxmox, amd64, and the `qemu-guest-agent` extension so Proxmox can see the guest properly. Then uploaded the ISO in **Datacenter > Storage > local > ISO Images**.

VM 201 settings that mattered:

| Setting | Value |
|---|---|
| Machine / BIOS | q35 / OVMF (UEFI), no Secure Boot |
| Disk | 40 GB VirtIO Block, discard on |
| CPU / RAM | 2 cores (type: host) / 4 GB |
| Network | VirtIO on vmbr0, **firewall off** at VM level |
| QEMU agent | enabled |

Firewall off at the VM level is deliberate — OPNsense is my perimeter, and a second firewall layer here only gives me two places to debug the same packet.

With the VM booted, everything else happens from my laptop. No SSH into the node, ever — this is the part of Talos I genuinely enjoyed:

```powershell
mkdir ~\talos-config
cd ~\talos-config

talosctl gen config Ira-cluster https://192.168.4.172:6443
```

That produces `controlplane.yaml`, `worker.yaml` and `talosconfig`. I patched three things into the control plane config — a static IP, the guest agent, and permission for pods to schedule on the control plane (it is a single CP node and I wanted the Flux controllers on an always-on machine):

```yaml
machine:
  network:
    hostname: talos-cp
    interfaces:
      - interface: eth0
        addresses:
          - 192.168.4.172/24
        routes:
          - network: 0.0.0.0/0
            gateway: 192.168.4.1
    nameservers:
      - 192.168.4.1
  install:
    extensions:
      - image: ghcr.io/siderolabs/qemu-guest-agent:10.2.0
cluster:
  allowSchedulingOnControlPlanes: true
```

Apply, then bootstrap — and bootstrap is a **once ever** command, not once per node:

```powershell
talosctl apply-config --insecure --nodes 192.168.4.172 --file controlplane.yaml

talosctl config merge ./talosconfig
talosctl config endpoint 192.168.4.172
talosctl config node 192.168.4.172
talosctl health --wait-timeout 5m

talosctl bootstrap --nodes 192.168.4.172
talosctl kubeconfig ./kubeconfig --nodes 192.168.4.172

$env:KUBECONFIG = "$PWD\kubeconfig"
kubectl get nodes
```

If `talosctl health` hangs, the node is usually still booting rather than broken — `talosctl dmesg --nodes 192.168.4.172 | tail -20` told me more than anything else during this stage.

#### Step - 4: Joining the Raspberry Pi 4 as the always-on worker

The Pi is the only machine in the lab running Talos on bare metal, using the ARM64 image and booting from a 128 GB USB SSD rather than the microSD card. Same `worker.yaml`, patched with its own address and **importantly labels**:

```yaml
machine:
  network:
    hostname: talos-rpi4
    interfaces:
      - interface: eth0
        addresses:
          - 192.168.4.141/24
        routes:
          - network: 0.0.0.0/0
            gateway: 192.168.4.1
    nameservers:
      - 192.168.4.1
  nodeLabels:
    node-role: worker
    tier: always-on
    arch: arm64
```

```powershell
talosctl apply-config --insecure --nodes 192.168.4.141 --file worker.yaml
kubectl get nodes
```

Putting the labels in the **machine config** rather than running `kubectl label node` is a small thing that is useful to save later headaches. The label survives a node rebuild, which means my scheduling rules are part of the infrastructure definition and not something I have to remember to reapply at 11 pm after reflashing a Pi.

![Output of kubectl get nodes --show-labels showing the tier labels: tier=always-on on the Raspberry Pi 4, tier=ai-worker on the Evo-X2, and tier=on-demand on the Dell R610 and Intel NUC.](/assets/images/kubectl-get-nodes-tier-labels.png)
*Nodes and their tier labels, set in the Talos machine configs.*

`tier=always-on` and `arch=arm64` are the two labels that do the actual work later. Anything pinned to `tier=always-on` will keep running when the WOL machines are asleep, and `arch=arm64` is a reminder to myself that only multi-arch images land there.

#### Step - 5: Encrypting the machine configs before they go anywhere near Git

The generated Talos configs contain the cluster CA, the machine join token and the bootstrap token. A leaked join token lets somebody add a node to your cluster, so these are exactly the files that must never be pushed in plaintext to a public repo.

```powershell
$env:SOPS_AGE_KEY = $(bw get notes "HOMELAB_SOPS_KEY")

sops --encrypt controlplane.yaml > kubernetes/talos/controlplane.enc.yaml
sops --encrypt worker.yaml       > kubernetes/talos/worker.enc.yaml

Remove-Item controlplane.yaml, worker.yaml
```

And the `.gitignore` side of the same coin — [the repo's `.gitignore`](https://github.com/deepchhaiya/Homelab-ops/blob/main/.gitignore) excludes `talos-config`, `controlplane.yaml`, `worker.yaml`, `kubeconfig` and every `.env`, so only `*.enc.yaml` can be committed. I would verify this before your first push rather than after.

#### Step - 6: Flux CD, so the cluster pulls from Git instead of me pushing to it

```powershell
flux bootstrap github `
  --owner=<your-github-user> `
  --repository=Homelab-ops `
  --branch=main `
  --path=kubernetes/clusters/homelab `
  --personal
```

Then the Age private key goes into the cluster once, as a secret Flux uses to decrypt at apply time. Same rule as before — the key comes straight from Vaultwarden into memory and never lands in a file on the laptop:

```powershell
$env:SOPS_AGE_KEY = (bw get notes "HOMELAB_SOPS_KEY") -join "`n"

kubectl create secret generic sops-age `
  --namespace=flux-system `
  --from-literal=age.agekey="$env:SOPS_AGE_KEY"

Remove-Item Env:SOPS_AGE_KEY
```

Which is what makes the `decryption` block in [`clusters/homelab/apps.yaml`](https://github.com/deepchhaiya/Homelab-ops/tree/main/kubernetes/clusters/homelab) work:

```yaml
spec:
  interval: 10m
  path: ./kubernetes/apps
  prune: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

From this point adding an application is a commit, not a `kubectl apply`. I write the manifests under `kubernetes/apps/<name>/`, add the folder to [`kubernetes/apps/kustomization.yaml`](https://github.com/deepchhaiya/Homelab-ops/blob/main/kubernetes/apps/kustomization.yaml), push, and Flux picks it up within ten minutes. `prune: true` also means deleting a manifest from Git deletes it from the cluster — correct behaviour for GitOps, mildly alarming the first time you see it happen. Why I chose Flux over ArgoCD, and Talos over K3s, is written up as [ADR-001](https://github.com/deepchhaiya/Homelab-ops/blob/main/ADR/ADR-001-talos-over-k3s.md), [ADR-002](https://github.com/deepchhaiya/Homelab-ops/blob/main/ADR/ADR-002-flux-over-argocd.md) and in more depth in the [GitOps case study](https://github.com/deepchhaiya/Homelab-ops/blob/main/docs/case-study-gitops-talos-flux.md).

#### Step - 7: Labels as the scheduling contract

This is the piece that made the uneven fleet manageable. Pods do not select machines, they select **tiers**:

| Label | Meaning | Carried by today |
|---|---|---|
| `tier=always-on` | 24x7, low power | Raspberry Pi 4 |
| `tier=ai-worker` | interactive AI, anything a person waits on | Evo-X2 worker VM 402 |
| `tier=on-demand` | may be asleep, batch only | NUC, R610, i9 |

So a deployment says this, and never names a machine:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        tier: ai-worker
```

The value of that showed up when my AI workloads moved from the i9 to the Evo-X2. Not a single manifest changed — I only changed which machine carried the `tier=ai-worker` label. If I had pinned pods to hostnames, that migration would have been a pull request touching every AI service.

The one exception I allow myself is a hostname pin, and only where a `local-path` PVC ties a pod to a particular disk — n8n's Postgres, NUT talking to the UPS over USB, the metrics agents.

### What I got wrong here, and changed later

The blueprint at the top is the fourth version of itself. In the order the changes happened:

**The storage LXC went away.** I first built a privileged CT 200 that owned the DAS and re-exported it over NFS to everything else. It lasted about three weeks. `nfsd` lives in the host kernel, so when it is asked to export a bind-mounted path from inside a container it tries to resolve the path on the host, fails to `stat()` it, and exports nothing — the error was `exportfs: Failed to stat /mnt/pvedas/n8n`. The fix was to stop pretending a container should sit between a disk and other containers on the same machine. The DAS now mounts on the host and bind-mounts into the LXCs, which is the shape shown in Step 2. The full story is in [Part 4]({% post_url 2026-09-26-building-a-power-efficient-homelab-part-4 %}).

**The Intel NUC lost its job to the Raspberry Pi.** The plan had the NUC as the primary always-on worker and the Pi as a "witness" node for monitoring only. In practice the NUC draws about 11 W at idle, to carry work the Pi handles comfortably at about 6.5 W, so the Pi became the always-on worker and now carries n8n, Miniflux with its Postgres, Beszel, NUT and the metrics agents. The NUC dropped down to the WOL tier where it belongs.

**The Evo-X2 was not in the plan at all.** It came in around the middle of May and it rewrote more of the topology than any faster machine would have. The i9 with an RTX 5070 was supposed to be my AI tier, woken on demand. But 96 GB of unified memory runs `qwen3.6:35b-a3b` at roughly 44 tokens/sec on an **iGPU**, a 12 GB discrete GPU cannot hold that model at all, and waking a machine before every prompt is not something you do twice. So the i9 retired from AI duty. Having a *second* always-on host then fixed two more things almost for free: observability moved off Peladn (monitoring the control-plane host from the control-plane host is a bad idea — if Peladn dies you lose the cluster and the tool you would diagnose it with), and Proxmox Backup Server with the 26 TB drive moved across too, because a machine's backups should not live on that machine. My failover target is now a local restore on the Evo-X2 with no sync jobs at all.

**I planned to retire the 780M Ollama on Peladn and kept it.** Once inference moved to the Evo-X2 the small model was supposed to go. It still serves `qwen3:4b`, because a couple of services need cheap always-available tagging and embedding that should not queue behind a 35B model. Keeping a small model next to its small consumers was the reason for continuing, even though it contradicted my own plan.

The honest summary is that the *shape* of the plan survived i.e. an always-on core with on-demand reinforcements, and almost every specific assignment inside it changed at least once. If you are planning something similar, I would spend less time getting the assignment right on paper and more time making it cheap to change, which for me meant using labels instead of hostnames and configs in Git instead of on my laptop that needs manual management.

The next part covers the storage layer underneath all of this — the DAS, the heatsink that fixed it, and the backup strategy that sits on top.
