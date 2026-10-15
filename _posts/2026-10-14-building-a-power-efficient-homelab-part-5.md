---
layout: post
title: "Building a Power-Efficient Homelab, Part 5: Deploying Docker Compose and Kubernetes Apps"
date: 2026-10-14 22:15:00 -0500
categories: homelab
tags: [homelab, docker, kubernetes, flux, sops, lxc, gitops]
excerpt: >-
  Docker Compose stacks in the two LXCs, the first Kubernetes apps deployed
  through Flux, and the secrets pattern — and the UID-mapping, GPU and
  storage mistakes — that ties both halves together.
---

In [Part 3]({% post_url 2026-09-26-building-a-power-efficient-homelab-part-3 %}) I created two Docker LXCs and a Talos cluster with Flux, and in [Part 4]({% post_url 2026-09-26-building-a-power-efficient-homelab-part-4 %}) I got the DAS stable and decided which data lives where. Now it is time to put applications on top of both.

I have split this post into two halves. The first half covers the Docker Compose stacks running in the two LXCs on Peladn. The second half covers the Kubernetes apps deployed through Flux on the Raspberry Pi worker. Both follow the same idea: **the config lives in Git, the secrets are encrypted, and the machine only pulls and runs.**

Every compose file and manifest I mention is in the public repo, [Homelab-ops](https://github.com/deepchhaiya/Homelab-ops). Docker Compose for the LXCs is under the [infrastructure](https://github.com/deepchhaiya/Homelab-ops/tree/main/infrastructure) folder, while Kubernetes app manifests are under the [kubernetes/apps](https://github.com/deepchhaiya/Homelab-ops/tree/main/kubernetes/apps) folder. You can copy them, change the IPs and paths to match your setup, and run them.

### A quick reminder of where things run

From Part 3, my rule was simple. If a service needs hardware that is attached to one specific machine (GPU or USB stick), it runs in Docker inside an LXC on that machine. Everything else goes to Kubernetes. The exception is NUT, which runs directly on the Talos worker on the Raspberry Pi 4. It needs to connect to the UPS via USB.

| Where | What runs there |
|---|---|
| CT 202 `hl-media-ai-ops-lxc` (192.168.4.12) | Immich, Nextcloud + MariaDB, Jellyfin, Webtop |
| CT 203 `hl-home-ops-lxc` (192.168.4.13) | Home Assistant, Nginx Proxy Manager, Vaultwarden, Gotify, UpSnap, Obsidian LiveSync |
| Talos cluster, Raspberry Pi worker | NUT (UPS monitor), Beszel, Miniflux, and later n8n |

Both LXCs also run a small What's Up Docker (WUD) container, a node-exporter and a Grafana Alloy log shipper. I will come back to WUD near the end, and the monitoring pieces will get their own post later.

---

## Part A — Docker Compose in the LXCs

#### Step - 1: Lay out the repo so each LXC has its own folder

Each LXC gets one folder in the repo with exactly three things in it: the compose file, the encrypted `.env`, and a copy of the Proxmox LXC config for reference.

```text
Homelab-ops/
└── infrastructure/
    ├── .sops.yaml
    ├── encrypt-secrets.ps1          # run on my laptop
    ├── decrypt-secrets.sh           # run inside the LXC
    ├── media-ops-lxc/
    │   ├── docker-compose.yml
    │   ├── .env.enc.yaml            # committed (encrypted)
    │   └── 202 (hl=media-ai-ops).conf
    └── home-ops-lxc/
        ├── docker-compose.yml
        ├── .env.enc.yaml            # committed (encrypted)
        └── 203 (hl-home-ops-lxc).conf
```

The [`.gitignore`](https://github.com/deepchhaiya/Homelab-ops/blob/main/.gitignore) blocks the `.env` file, only the `.env.enc.yaml` version gets committed.

The two compose files are here:

- [media-ops-lxc/docker-compose.yml](https://github.com/deepchhaiya/Homelab-ops/blob/main/infrastructure/media-ops-lxc/docker-compose.yml)
- [home-ops-lxc/docker-compose.yml](https://github.com/deepchhaiya/Homelab-ops/blob/main/infrastructure/home-ops-lxc/docker-compose.yml)

#### Step - 2: Keep passwords out of the compose file

The compose files only refer to variables like `${NC_DB_PASSWORD}` or `${VW_ADMIN_TOKEN}`. The real values sit in a `.env` file next to the compose file, which Docker Compose reads automatically.

A trimmed example of what the media `.env` looks like (with fake values):

```bash
NC_DB_USER=nextcloud
NC_DB_PASSWORD=change-me
MARIADB_ROOT_PASSWORD=change-me-too
NC_TRUSTED_DOMAINS=nextcloud.yourdomain.duckdns.org 192.168.4.12
DB_PASSWORD=immich-db-password
WEBTOP_USER=me
WEBTOP_PASSWORD=change-me
```

On my laptop I encrypt it with the same Age key from [Part 2]({% post_url 2026-09-04-building-a-power-efficient-homelab-part-2 %}), pulled from Vaultwarden into memory only. The helper script is [encrypt-secrets.ps1](https://github.com/deepchhaiya/Homelab-ops/blob/main/infrastructure/encrypt-secrets.ps1), and the important line inside it is:

```powershell
$env:BW_SESSION   = $(bw unlock --raw)
$env:SOPS_AGE_KEY = $(bw get notes "HOMELAB_SOPS_KEY")

sops --encrypt --input-type dotenv --output-type yaml media-ops-lxc/.env > media-ops-lxc/.env.enc.yaml
```

**Note:** *The file is named `.env.enc.yaml`, so SOPS can mistake it for a Kubernetes YAML file and apply the Kubernetes rule, which only encrypts the `data` and `stringData` fields. A `.env` has neither, so the values end up in plain text inside a file that looks encrypted. Always pass `--input-type dotenv` and then open the output before committing. Every value should start with `ENC[`. If you see a readable password, do not commit it.*

#### Step - 3: Get the repo and secrets onto the LXC

Inside each LXC (Docker was already installed in Part 3) I cloned the repo once and installed SOPS:

```bash
pct enter 202

apt install -y git age
curl -LO https://github.com/getsops/sops/releases/download/v3.9.4/sops-v3.9.4.linux.amd64
install -m 0755 sops-v3.9.4.linux.amd64 /usr/local/bin/sops

cd /root
git clone https://github.com/deepchhaiya/Homelab-ops.git
```

Unlike Kubernetes, Docker Compose cannot read a SOPS file by itself. It needs a plain `.env` sitting next to the compose file, otherwise every `${...}` variable comes through blank and services like Immich refuse to start. So the first time only, [decrypt-secrets.sh](https://github.com/deepchhaiya/Homelab-ops/blob/main/infrastructure/decrypt-secrets.sh) turns the encrypted file back into a normal `.env`. It asks for the Age private key at the prompt, so the key is only in memory for the few seconds it runs and is never saved in the container:

```bash
cd /root/Homelab-ops/infrastructure
./decrypt-secrets.sh
# Enter your Age private key (starts with AGE-SECRET-KEY-):
```

I paste the key from Vaultwarden, and it writes `media-ops-lxc/.env` and `home-ops-lxc/.env`, and unsets the variable.

**This is a one-time step, not part of every deploy.** The `.env` is gitignored, so `git pull` never touches it and it simply stays on the LXC. Honestly, I had forgotten I ever ran this script until I went back through my notes for this post. Day to day I only run `git pull` and `docker compose up -d`. The encrypted copy in Git is really my backup: if an LXC dies and I rebuild it, this is how its passwords come back. I only need to run the script again if I add a new secret or rebuild a container.

**Note:** *Because the live `.env` on the LXC is what actually runs, it is easy for the encrypted copy in Git to fall behind without you noticing. Mine did. When I add or change a secret, I change it in both places on the same day: edit the `.env`, re-encrypt it on my laptop, and commit. Every so often, check that the copy in Git still decrypts. Later, pull and replace it in the LXCs. You don't want to find out it doesn't on the day you need it.*

#### Step - 4: Prepare the folders, and fix ownership on the host

Before the first `docker compose up`, the folders that the containers mount need to exist with the right owner. This is where the unprivileged LXC UID mapping from Part 3 comes back.

Quick recap: inside an unprivileged LXC, user `0` is really user `100000` on the Proxmox host, and user `1000` is really `101000`. So for anything on the DAS (bind-mounted from the host), the `chown` has to happen **on the Proxmox host**, using the mapped number.

The formula is simply: **host UID = container UID + 100000.**

| Container runs as | Inside UID | Run `chown` on host with |
|---|---|---|
| root (Gotify, NPM, Vaultwarden) | 0 | `100000:100000` |
| www-data (Nextcloud) | 33 | `100033:100033` |
| user 1000 (Jellyfin, Immich) | 1000 | `101000:101000` |
| couchdb (Obsidian LiveSync) | 5984 | `105984:105984` |

```bash
# On the Peladn Proxmox host
mkdir -p /mnt/pvedas/{immich/upload,immich/model-cache,jellyfin/config,jellyfin/cache,nextcloud/data,gotify/data}

chown -R 101000:101000 /mnt/pvedas/jellyfin /mnt/pvedas/immich/upload
chown -R 100033:100033 /mnt/pvedas/nextcloud
chown -R 100000:100000 /mnt/pvedas/gotify
```

Folders on the LXC's own disk (like `/var/lib/vaultwarden`) are different. Those you can create and `chown` from inside the container like normal, because they are not shared with the host.

**Note:** *If you run `chown` inside the container on a bind-mounted path, it either fails with "Operation not permitted" or looks like it worked and the app still cannot write. If an app keeps saying "permission denied" on a DAS path, this is almost always the reason.*

#### Step - 5: Give the containers the GPU

Jellyfin, Immich and Webtop use the Radeon 780M through `/dev/dri` for hardware transcoding. The LXC config already passes `/dev/dri` in (Part 3). In the compose file, each service also gets the device:

```yaml
  jellyfin:
    image: jellyfin/jellyfin:12.2
    user: "1000:1000"
    group_add:
      - "44"   # video
      - "992"  # render
    devices:
      - /dev/dri:/dev/dri
```

Two things I learned here:

1. **Use numbers, not names, in `group_add`.** The `video` and `render` group names may not exist inside the image, and then the container fails with "group not found". Check the real numbers in your LXC with `ls -ln /dev/dri`.
2. **In an unprivileged LXC, the group numbers still may not line up.** The device can show up as `nobody:nogroup` inside the container. What finally fixed it for me was a udev rule on the Proxmox host that makes the render devices readable by everyone:

```bash
# On the Proxmox host
echo 'SUBSYSTEM=="drm", MODE="0666"' > /etc/udev/rules.d/99-dri.rules
udevadm control --reload-rules && udevadm trigger
```

This is fine for a home lab where only my own containers run on that host. Then check that it actually works:

```bash
docker exec jellyfin ls -la /dev/dri/          # should show crw-rw-rw-
docker exec jellyfin vainfo --display drm --device /dev/dri/renderD128
```

If `vainfo` lists H.264 and HEVC profiles, go to Jellyfin **Dashboard > Playback**, choose **VAAPI** and `/dev/dri/renderD128`.

#### Step - 6: Decide where each container keeps its data

This is the rule from Part 4, now applied to real services:

- **Big files go on the DAS:** photos, Nextcloud files, Jellyfin media and config.
- **Databases and anything that writes often go on the LXC's local NVMe:** MariaDB, Immich Postgres, Vaultwarden, Home Assistant, NPM, CouchDB.

You can see it directly in the compose file. Nextcloud's files are on the DAS, but its database is local:

```yaml
  nextcloud:
    volumes:
      - /var/www/nextcloud/:/var/www/html           # app code, local NVMe
      - /mnt/das/nextcloud/data:/var/www/html/data  # user files, DAS

  mariadb:
    volumes:
      - /var/lib/nextcloud-db:/config               # database, local NVMe
```

I did not start this way. In the first version, every service in the home-ops LXC, including Home Assistant and Vaultwarden, kept its data on the DAS. Then the DAS had its disconnect problems (Part 4), and I came very close to losing the password vault. I moved all of the home-ops data to the local disk with a simple tar pipe from the Proxmox host:

```bash
# On the Proxmox host, with the containers stopped
(cd /mnt/pvedas/vaultwarden/data && tar cf - .) | \
  pct exec 203 -- tar xf - --no-same-owner -C /var/lib/vaultwarden/data
```

After that, I only changed the path on the left side of each volume line in the compose file. The containers did not even notice.

A good side effect showed up later: since the important data now lives inside the LXC's own disk, a normal Proxmox backup of the LXC captures it. More on that in the next part.

#### Step - 7: Bring the stacks up

With the `.env` in place, folders ready and the GPU visible:

```bash
cd /root/Homelab-ops/infrastructure/media-ops-lxc
docker compose pull
docker compose up -d
docker compose ps
```

Same for `home-ops-lxc` in CT 203. If your Docker install only has the older `docker-compose` command, the steps are the same with a hyphen.

Then I opened each service in the browser by IP and port first, before adding any domain name. If it doesn't work on `http://<LXC_IP>:<PORT>`, it won't work through a reverse proxy either, and you are debugging two things at once.

**Updating later** is the same loop every time: change the compose file on my laptop, commit, push, then on the LXC run `git pull` and `docker compose up -d`. Compose only recreates the containers that changed. No decrypting here, because the `.env` from Step 3 is still sitting there.

#### Step - 8: Move the reverse proxy last

Nginx Proxy Manager (NPM) is what makes `nextcloud.yourdomain` work from outside. It owns ports 80 and 443, and it was still running on my old CasaOS server at this point. I moved it last, after everything else was already working on the new IPs.

1. Stop NPM on the old server first. Only one machine can own ports 80 and 443.
2. Copy NPM's `data` and `letsencrypt` folders across with `rsync`, so all proxy hosts and SSL certificates come along.
3. Start NPM in CT 203 and open the admin page on port 81.
4. Edit each proxy host to point at the new IP and port (for example Jellyfin moved from `old-server:8097` to `<MEDIA_LXC_IP>:8096`).
5. Point the router's port forward for 80 and 443 at `<HOME_OPS_LXC_IP>`.

I kept the old NPM data for a couple of weeks. If something goes wrong, I can stop the new one, start the old one and switch the port forward back in five minutes.

Two problems came up after the move:

- **NPM admin page refused to connect.** The certificate files were in a different folder on CasaOS than I expected. `docker inspect` on the old container shows the real mount paths. Check them before copying, not after.
- **Home Assistant returned "400 Bad Request" through the proxy.** Home Assistant only accepts proxied requests from addresses it trusts, and NPM's traffic was coming from a Docker network address it did not know. The fix in [`configuration.yaml`](https://github.com/deepchhaiya/Homelab-ops/blob/main/infrastructure/home-ops-lxc/homeassistant/configuration.yaml) was to trust the full Docker range, not just one subnet:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 127.0.0.1
    - 192.168.4.0/24
    - 172.16.0.0/12   # covers every Docker bridge network
```

#### Mistakes worth avoiding in the Docker half

- **Home Assistant: mount the `config` folder, not its parent.** I mounted `/var/lib/homeassistant` as `/config`. That folder also had a subfolder called `homeassistant`, and Python loaded it instead of the real Home Assistant package, which crashed it with a strange `get_default_config_dir` error. Mounting `/var/lib/homeassistant/config:/config` fixed it. I changed this path back and forth three times before I understood why.
- **Frigate on an AMD iGPU.** Frigate's OpenVINO GPU detector only works on Intel GPUs, so on the 780M it crashed and I had to run detection on the CPU. My cloud-only Arlo cameras made it worse with broken streams. I later replaced the cameras with Reolink and removed Frigate completely, using Home Assistant's built-in Reolink integration instead. The old Frigate block is still commented out in the compose file if you want to see it.
- **Pinning image versions.** I started with `:latest` and `:stable` everywhere. Then Nextcloud's `:stable` tag jumped two major versions while my container was still on the old one, and Nextcloud cannot skip a major version during an upgrade. Now the important images are pinned (`nextcloud:35-apache`, `vaultwarden/server:1.37.4`, `jc21/nginx-proxy-manager:2.16.0`, and so on). I upgrade them on purpose, one at a time.
- **Keeping track of available updates.** Pinning means you stop getting updates automatically, so you need a way to know when one is out. [What's Up Docker](https://github.com/getwud/wud) watches all containers in the LXC and shows which images have newer versions. It does not update anything by itself, which is exactly what I want. Note that WUD 9 and newer need an admin login to start, and the username and password come from the same encrypted `.env`.

---

## Part B — First Kubernetes apps with Flux

The Docker side is "pull and run by myself". The Kubernetes side is fully automatic: I push to GitHub, and Flux applies it to the cluster within ten minutes. Flux was set up in Part 3, Step 6. Here I add the pieces an app needs before it can run.

#### Step - 9: Give the cluster somewhere to store data

A fresh Talos cluster has no storage class, so any app that asks for a volume will wait forever in `Pending`. I used Rancher's [local-path-provisioner](https://github.com/rancher/local-path-provisioner). It is a small controller that creates volumes as plain folders on the node's own disk. No NFS, no extra storage system.

```powershell
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.30/deploy/local-path-storage.yaml

kubectl get storageclass
# local-path   rancher.io/local-path   Delete   WaitForFirstConsumer
```

If it is not marked as `(default)`, set it (in Git Bash, since PowerShell breaks the JSON quotes):

```bash
kubectl patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

On the Pi, these volumes end up on the 128 GB USB 3.0 SSD, which is plenty for small apps and their databases. This is also how [ADR-004](https://github.com/deepchhaiya/Homelab-ops/blob/main/ADR/ADR-004-local-path-over-nfs-for-databases.md) from Part 4 works in practice: databases in Kubernetes always use `local-path`, never NFS.

**Note:** *`local-path` ties the volume to the node where the pod first started. If that pod moves to another node, its data does not follow. That is fine for the Pi, because the Pi is always on. Just keep it in mind when you choose which `tier` an app runs on. Also, this local data is backed up and can be recovered if needed.*

#### Step - 10: The pattern every app follows

Every app in [`kubernetes/apps/`](https://github.com/deepchhaiya/Homelab-ops/tree/main/kubernetes/apps) has the same small set of files:

```text
kubernetes/apps/miniflux/
├── namespace.yaml
├── pvc.yaml             # storage request (local-path)
├── secret.enc.yaml      # SOPS-encrypted passwords
├── postgres.yaml        # the database
├── deployment.yaml      # the app itself
├── service.yaml         # how to reach it
└── kustomization.yaml   # lists the files above
```

And to turn an app on, I add its folder name to [`kubernetes/apps/kustomization.yaml`](https://github.com/deepchhaiya/Homelab-ops/blob/main/kubernetes/apps/kustomization.yaml):

```yaml
resources:
  - beszel
  - miniflux
  - nut
  # ...
```

That is the whole deploy process: create the folder, add one line, commit, push.

#### Step - 11: Secrets for Kubernetes apps

Kubernetes secrets go through SOPS too, but this time Flux decrypts them inside the cluster, using the `sops-age` secret from Part 3. Unlike the Docker side, there is no decrypt step for you at all. You commit the encrypted file, and Flux handles the rest on every sync. I write the plain secret file locally (gitignored):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: miniflux-secrets
  namespace: miniflux
type: Opaque
stringData:
  POSTGRES_PASSWORD: "<strong-password>"
  ADMIN_USERNAME: "admin"
  ADMIN_PASSWORD: "<strong-password>"
  DATABASE_URL: "postgres://miniflux:<same-postgres-password>@miniflux-db:5432/miniflux?sslmode=disable"
```

Encrypt it and delete the plain copy:

```powershell
$env:SOPS_AGE_KEY = $(bw get notes "HOMELAB_SOPS_KEY")
sops --encrypt kubernetes/apps/miniflux/secret.yaml > kubernetes/apps/miniflux/secret.enc.yaml
Remove-Item kubernetes/apps/miniflux/secret.yaml
```

The [`kubernetes/.sops.yaml`](https://github.com/deepchhaiya/Homelab-ops/blob/main/kubernetes/.sops.yaml) rule only encrypts `data` and `stringData`. The rest of the file stays readable, so you can still see in GitHub which secret it is and which namespace it belongs to, but not the values.

#### Step - 12: Miniflux, a full example with a database

[Miniflux](https://miniflux.app/) is a simple RSS reader that needs PostgreSQL. It is a good first app because it has every piece: a volume, a secret, a database and a web UI.

The storage request ([pvc.yaml](https://github.com/deepchhaiya/Homelab-ops/blob/main/kubernetes/apps/miniflux/pvc.yaml)):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: miniflux-db
  namespace: miniflux
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-path
  resources:
    requests:
      storage: 2Gi
```

The database ([postgres.yaml](https://github.com/deepchhaiya/Homelab-ops/blob/main/kubernetes/apps/miniflux/postgres.yaml)), shortened. Notice that the password comes from the secret, and the pod picks a **tier**, not a machine name, just like in Part 3, Step 7:

```yaml
    spec:
      nodeSelector:
        tier: always-on
      containers:
        - name: postgres
          image: postgres:16-alpine
          env:
            - name: POSTGRES_USER
              value: miniflux
            - name: POSTGRES_DB
              value: miniflux
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: miniflux-secrets
                  key: POSTGRES_PASSWORD
          volumeMounts:
            - name: db-data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: db-data
          persistentVolumeClaim:
            claimName: miniflux-db
```

The app itself ([deployment.yaml](https://github.com/deepchhaiya/Homelab-ops/blob/main/kubernetes/apps/miniflux/deployment.yaml)) reads `DATABASE_URL` and the admin login from the same secret, and sets `RUN_MIGRATIONS=1` and `CREATE_ADMIN=1` so the first start prepares the database for you.

To reach it from my network I used a `NodePort` service, which opens a fixed port on the node:

```yaml
spec:
  type: NodePort
  selector:
    app: miniflux
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080
```

Commit, push, and after a few minutes:

```powershell
flux get kustomizations          # apps should say "Applied revision: main@..."
kubectl get pods -n miniflux     # miniflux and miniflux-db both Running
kubectl get pvc -n miniflux      # Bound
```

Miniflux opens at `http://<LOCAL_IP>:30080`. Later I added a proxy host for it in NPM, exactly like the Docker services.

If you are in a hurry and do not want to wait for Flux, `flux reconcile kustomization apps --with-source` makes it check GitHub right away.

#### Step - 13: Beszel, the small and simple one

[Beszel](https://github.com/henrygd/beszel) is a light server monitor that works on ARM. Its folder is the same pattern without a database: namespace, a 1 GB PVC, a deployment with `tier: always-on`, and a NodePort service on `30090`. The full set is in [kubernetes/apps/beszel](https://github.com/deepchhaiya/Homelab-ops/tree/main/kubernetes/apps/beszel). I would start with this one if you want to test that Flux, storage and the node labels all work before trying anything with a secret.

**Note:** *The Pi is ARM64. Before you deploy anything there, check that the image has an `arm64` build on its Docker Hub or GitHub page. Otherwise the pod will crash with `exec format error`, which does not clearly say "wrong CPU type".*

#### Step - 14: NUT, the one that needs hardware

My UPS is plugged into the Raspberry Pi over USB. [NUT](https://networkupstools.org/) reads its battery status and shares it on port 3493, so Home Assistant can show the battery level and runtime. This app breaks the usual rules, so it is a good example of how to handle the exceptions:

- It is pinned to the Pi by **hostname**, not by tier, because the USB cable is physically there. This is one of the hostname-pin exceptions I mentioned in Part 3.
- It uses `hostNetwork: true`, so Home Assistant can reach it at `<LOCAL_IP>:3493`.
- It runs as `privileged` to access `/dev/bus/usb`. Kubernetes blocks that by default, so the namespace needs a label to allow it:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nut
  labels:
    pod-security.kubernetes.io/enforce: privileged
```

The NUT config files need passwords, but a ConfigMap is not encrypted. So the [ConfigMap](https://github.com/deepchhaiya/Homelab-ops/blob/main/kubernetes/apps/nut/configmap.yaml) keeps placeholders like `__MONITOR_PASSWORD__`, and a small init container replaces them with the real values from the encrypted secret before NUT starts. This pattern works for any app whose config file has a password in it.

A few NUT-specific problems I hit, in case you try the same thing:

| Error | What fixed it |
|---|---|
| `Can't claim USB device` | The image's start script started the driver twice. I replaced the command with my own start steps. |
| `insufficient permissions` | Talos owns USB devices as root, so I added `user = root` in `ups.conf`. |
| New pod stuck in `Pending` after an update | The old pod still held port 3493. Delete the old pod first. |

The working [deployment.yaml](https://github.com/deepchhaiya/Homelab-ops/blob/main/kubernetes/apps/nut/deployment.yaml) has all three fixes.

---

### Docker or Kubernetes: how it feels after a few months

Both halves have been running for months now, and they feel different to manage.

The **Docker LXCs** are easy to understand. When something breaks, `docker logs` on one machine usually tells me why. The downside is that updates are manual (`git pull` and `docker compose up -d`), and the LXC can drift from Git if I make a quick change on the box and forget to commit it.

The **Kubernetes side** takes more files to set up, but after that I never touch the machines. Git is always the truth, because Flux puts things back the way Git says. Moving an app to another node is just a label change.

If you are building your own lab, I would start with Docker Compose for everything. Then move a service to Kubernetes only when you actually want scheduling, automatic restarts or GitOps for it. Services tied to a USB stick or a GPU on one box get very little from Kubernetes, and that is why mine are still in Docker today.

If you want the whole build in order with a check at the end of each stage, it is in [docs/homelab-build-guide.md](https://github.com/deepchhaiya/Homelab-ops/blob/main/docs/homelab-build-guide.md).

With the services running, the next question is how to get them back if a disk or a machine dies. The next part covers backups: Proxmox Backup Server on the Evo-X2, the 26 TB drive, n8n running the backup jobs, and actually testing a restore.
