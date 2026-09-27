---
layout: post
title: "Building a Power-Efficient Homelab, Part 4: Setting Up Storage in Peladn"
date: 2026-09-26 20:40:00 -0500
categories: homelab
tags: [homelab, storage, proxmox, das, raid, nfs, xfs, backups]
excerpt: >-
  The DAS that kept disconnecting, the NFS server that was in the wrong place,
  and the rule that decides what is allowed to live on the DAS.
---

This part covers setting up the Wavlink DAS as the primary storage of the cluster. The Peladn Mini PC is the control node for the Talos cluster and the primary machine of my homelab, and the Wavlink DAS in RAID 1 mode is attached to it.

It is also the part of the build I got most wrong on the first attempt, so I have written both versions — what I built first, and what is running today. All the configs referenced here live in the public repo, [Homelab-ops](https://github.com/deepchhaiya/Homelab-ops).

### Why RAID 1?

I did later implement proper 3-2-1 backup architecture, but I needed some safety in the meantime and I only had a 2-bay DAS. I did not want a single disk failure to take my data with it, so mirroring two 12 TB Toshiba N300 Pro drives was the straightforward choice for my primary machine's storage. It costs me half the capacity i.e. about 10.9 TiB usable but I am fine with that trade off for the data that everything else in the lab depends on.

### The first attempt, and the two problems it surfaced

I started by building a dedicated **privileged** storage LXC (CT 200, `hl-storage-lxc`) that owned the Wavlink DAS and re-exported it over NFS. The implementation was to pass the DAS through as a raw block device (`lxc.cgroup2.devices.allow: b 8:16 rwm`), format it XFS inside the container, and export `/mnt/das` over NFS to the other LXCs and to Kubernetes.

Soon after the setup, while testing the NFS share from a client on my Intel NUC, two issues showed up.

1. Intermittent disconnections during heavy read and write activity on the DAS.
2. The NFS server stopped with an error whenever the disk became unavailable.

I worked through both with a mix of AI help and DIY, and these are the fixes that actually held.

**Problem 1 — the disconnects were thermal.** I researched the failure pattern and the most likely cause was heat on the ASMedia ASM1352R controller chip, which ships with no heatsink at all. I had a few small heatsinks lying over from a previous build.

![Wavlink DAS manufacturer graphic showing the four RAID modes and the DIP switch positions on the back of the enclosure.](/assets/images/wavlink-das-raid-modes.png)

*Image credit: Wavlink.*

So I removed the back cover, found two chips on the backplane behind the fan, and stuck the heatsinks on. That solved it — the drops stopped and transfer speeds settled around 300 MB/s. After that I switched the enclosure into RAID 1 mode for redundancy across the two drives.

A five-dollar heatsink fixed something that could very easily have passed for a driver bug or a bad cable, and I would have spent weeks chasing it in the wrong place without AI's research.

**Problem 2 — the NFS server was in the wrong place entirely.** This one was not a hardware fault. `nfsd` is a kernel service and LXCs share the host kernel, so when it is asked to export a path that reached the container as a bind mount, it resolves that path back to the source on the host (`/mnt/pvedas/...`) which does not exist inside the container's mount namespace. `exportfs` then fails to `stat()` it, exports nothing, and every client gets "No such file or directory" *from the server side*. The actual line I saw was:

```text
exportfs: Failed to stat /mnt/pvedas/n8n: No such file or directory
```

So instead of using CT 200, I passed the Wavlink DAS through directly and added it as a disk on the Proxmox host on the Peladn machine.

![Proxmox Disks view on the Peladn host showing the 12 TB Toshiba drive from the DAS formatted as XFS.](/assets/images/proxmox-disks-das.png)

Once the disk lived on the host, the storage LXC had no job left. Its only responsibility had been serving NFS, and the DAS was physically attached to the very machine whose containers were mounting it over the network. I shut it down, set `onboot 0`, and destroyed it a couple of weeks later.

**Learning:** if you are trying something similar, I would recommend confirming the reliability of the DAS or the storage solution first, and then keeping it on the host OS rather than inside a VM or an LXC. A container between a disk and other containers on the same machine achieves nothing and adds a failure domain.

The rest of this post is the setup for the architecture that is live today.

### The rule

> **The DAS is mounted once, on the Proxmox host. Everything else gets it from there — by bind mount if it is a guest on that host, and by NFS only if it genuinely cannot be.**

No storage VM, no storage container, no NFS hop between a disk and a container sitting on the same machine.

```text
Peladn (192.168.4.150, Proxmox host) — the DAS is physically attached here
│
├── /mnt/pvedas        ← 10.9 TiB DAS, XFS, mounted by UUID in /etc/fstab
│   ├── bind mount ──► CT 202  hl-media-ai-ops-lxc   as /mnt/das
│   ├── bind mount ──► CT 203  hl-home-ops-lxc       as /mnt/das
│   └── nfsd (on the HOST) ──► K8s: /mnt/pvedas/karakeep/assets   ← the only NFS export
│
├── /mnt/sg-ext-hdd    ← 2 TB Seagate external, host mount, bind-mounted the same way
├── /mnt/wd-ext-hdd    ← 2 TB WD external, host mount, bind-mounted the same way
│
└── local NVMe         ← every database and anything transactional
                          (Home Assistant, Vaultwarden, NPM, Gotify, CouchDB,
                           Immich/Nextcloud DBs)
```

#### Step - 1: Mount the DAS on the host, by UUID and not by `/dev/sdX`

The USB device letters on this host shuffle across reboots — the DAS has come up as `sdb`, `sdc` and `sdd` at different times depending on kernel probe order. So everything gets mounted by filesystem UUID and referenced by mountpoint afterwards.

```bash
# Find the DAS and its UUID
lsblk -o NAME,SIZE,FSTYPE,UUID,MOUNTPOINT,SERIAL,MODEL

# Format XFS — first time only, this erases the disk
mkfs.xfs /dev/sdX

mkdir -p /mnt/pvedas
```

Then append to `/etc/fstab` on the Proxmox host. The real file is committed as [`infrastructure/peladn-host/fstab-external-drives.conf`](https://github.com/deepchhaiya/Homelab-ops/blob/main/infrastructure/peladn-host/fstab-external-drives.conf):

```bash
# 10.9 TiB DAS — XFS. PLAIN mount: do NOT add x-systemd.automount (see the note below)
UUID=cb8ce272-49dc-4cdb-9632-xxxxxxxxxxxx  /mnt/pvedas  xfs  defaults,nofail,nouuid,x-systemd.device-timeout=15  0 2
```

| Option                        | Why                                                                                                                                                                                                                           |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `nofail`                      | a missing or slow USB drive must never block boot                                                                                                                                                                             |
| `x-systemd.device-timeout=15` | caps how long boot waits for the device                                                                                                                                                                                       |
| `nouuid`                      | required for XFS when the filesystem UUID also appears elsewhere                                                                                                                                                              |
| **no** `x-systemd.automount`  | I had this at first and removed it. The automount registers `/mnt/pvedas` twice in `/proc/mounts` — an `autofs` entry plus the real device — which made Beszel key on `systemd-1` and report capacity but no disk I/O at all. |

```bash
systemctl daemon-reload && mount -a
df -h /mnt/pvedas
```

#### Step - 2: Map the DAS with LXCs via bind mounts

```bash
# On the Proxmox host
pct stop 202 && pct stop 203

pct set 202 -mp0 /mnt/pvedas,mp=/mnt/das
pct set 203 -mp0 /mnt/pvedas,mp=/mnt/das

# same pattern for the two external drives
pct set 202 -mp1 /mnt/sg-ext-hdd,mp=/mnt/sg-ext-hdd
pct set 202 -mp2 /mnt/wd-ext-hdd,mp=/mnt/wd-ext-hdd

pct start 202 && pct start 203
pct exec 202 -- ls /mnt/das
```

The containers still see `/mnt/das`, exactly as they did when it arrived over NFS. So every Docker Compose volume path stayed as it was and the migration was invisible above the mount. This is the main reason this was a cheap change to make rather than a rewrite of both compose files.

**Note:** *CT 202 and CT 203 are unprivileged, which means container UID 0 maps to host UID 100000. All ownership fixes for bind-mounted paths have to be run on the Proxmox host with the mapped UID — container `1000:1000` is `101000:101000` on the host. Running `chown` from inside the container appears to succeed and then the application still cannot write, which is a confusing half hour if you have not seen it before.*

#### Step - 3: NFS, only where a workload genuinely cannot bind-mount

Exactly one workload qualifies. Karakeep runs in Kubernetes on a *different* physical node and needs shared read-write access to its asset directory on the DAS. It could have been pointed at local storage but I kept this one service on NFS deliberately, partly because it genuinely benefits from RWX and partly because I wanted one live NFS path in the lab to keep working with. `nfsd` runs on the Peladn host, where the disk actually is, and never inside a container.

```bash
# On the Peladn host
apt install -y nfs-kernel-server
echo "/mnt/pvedas/karakeep 192.168.4.0/24(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
systemctl enable --now nfs-server
exportfs -ra
exportfs -v
```

And the Kubernetes side — [`kubernetes/apps/karakeep/karakeep-assets-pv.yaml`](https://github.com/deepchhaiya/Homelab-ops/blob/main/kubernetes/apps/karakeep/karakeep-assets-pv.yaml):

```yaml
spec:
  capacity:
    storage: 200Gi
  accessModes: [ReadWriteMany]
  storageClassName: nfs-pvedas-karakeep
  mountOptions: [nfsvers=4.1, hard, timeo=600, retrans=2]
  nfs:
    server: 192.168.4.150
    path: /mnt/pvedas/karakeep/assets
```

One thing worth knowing if you keep an export on a USB-attached disk: **a flap kills the export and it does not come back on its own.** `nfs-server` depends on the `/mnt/pvedas` mount, so if the DAS drops off the USB bus systemd stops the service and then does not restart it when the mount returns. I ended up writing a watchdog script for it i.e. a systemd timer on roughly a two minute tick that re-mounts the DAS, restarts `nfs-server`, re-runs `exportfs -ra` and pushes a Gotify notification so I know it happened. It is in the repo at [`infrastructure/peladn-host/`](https://github.com/deepchhaiya/Homelab-ops/tree/main/infrastructure/peladn-host) along with the udev rule that disables USB autosuspend for the ASMedia controller.

#### Step - 4: Deciding what is allowed to live on the DAS

| Data                                                                        | Lives on                          | Reached via                                 |
| --------------------------------------------------------------------------- | --------------------------------- | ------------------------------------------- |
| Media, photos, Nextcloud files, Frigate clips                               | DAS `/mnt/pvedas`                 | bind mount to `/mnt/das` in CT 202 / CT 203 |
| Karakeep assets (K8s, on another node)                                      | DAS `/mnt/pvedas/karakeep/assets` | NFS from the host — the only export         |
| Every database — Home Assistant, Vaultwarden, NPM, Gotify, CouchDB, app DBs | **local NVMe** in the LXC         | local path, never NFS and never the DAS     |
| K8s stateful apps — n8n with Postgres, Miniflux, Hindsight                  | node-local disk                   | `local-path` storage class                  |
| Backups and archives                                                        | 26 TB HDD on the Evo-X2           | Proxmox Backup Server over the network      |

That third row is the one that cost me something to learn. n8n was originally running its default SQLite database on a DAS-backed NFS PVC. On the 19th of June the DAS flapped mid-write, the share dropped, and n8n came back with `SQLITE_CORRUPT: database disk image is malformed`. SQLite over NFS is unsafe in general but NFS file locking is advisory across the network and does not give SQLite the guarantees it assumes. So putting it on a USB enclosure that occasionally disappears made it a question of when rather than whether.

So n8n moved to PostgreSQL on a `local-path` PVC, and I wrote the rule down as an ADR so I would not quietly undo it later: [ADR-004 — local-path over NFS for databases](https://github.com/deepchhaiya/Homelab-ops/blob/main/ADR/ADR-004-local-path-over-nfs-for-databases.md). Around the same time I moved Home Assistant, Vaultwarden, NPM, Mosquitto and the Obsidian LiveSync CouchDB off the DAS onto the home-ops LXC's local NVMe, which is why that container's rootfs is 50 GB rather than the 16 GB I first gave it.

The short version: **bulk data on the DAS, databases on local disk, NFS only for blobs that need sharing.** Everything in my storage layout now follows from that one sentence, and I would start from it if I were building this again.

### Where this leaves the 3-2-1 story

With the DAS stable and the data split sensibly, the backup side became simple to reason about. Proxmox Backup Server runs on the Evo-X2 with the 26 TB Seagate drive attached to it, and n8n drives `vzdump` of the Peladn guests across to it on a schedule. The important property is that the backups do not live on the machine they are backing up. So if Peladn dies, the restore happens on a host that is already holding the data, with no sync jobs and no waking a sleeping machine first.

The databases being in the container rootfs rather than on bind mounts turned out to matter here too, in a good way: a vzdump snapshot captures Vaultwarden, Home Assistant's config and the app databases, so a restore brings back working services rather than empty shells. Bulk media sits outside the snapshot and is backed up on a slower weekly cadence, which is the correct trade i.e. media is restore-when-needed, a password vault is not.

I have written the whole storage decision up as a case study in the repo, with the numbers and the failure timeline: [Storage architecture case study](https://github.com/deepchhaiya/Homelab-ops/blob/main/docs/case-study-storage-architecture.md). And if you want the end-to-end build sequence rather than just this layer, it is in [docs/homelab-build-guide.md](https://github.com/deepchhaiya/Homelab-ops/blob/main/docs/homelab-build-guide.md).

The next part gets into the backup implementation properly — PBS, the retention policy, and the n8n workflows that drive it.
