---
layout: post
title: "Building a Power-Efficient Homelab, Part 2: Setting Up the OS and GitHub Repo"
date: 2026-09-04 20:00:00 -0500
categories: homelab
tags: [homelab, proxmox, talos, github, gitops, sops, vaultwarden]
excerpt: >-
  Installing Proxmox across the fleet, standing up a GitOps-managed GitHub
  repo, and building a zero-trust workstation with SOPS, Age, and Vaultwarden
  so no secret key ever touches disk.
---

This part mainly focusses on setting up different machines and GitHub repo. I started exploring the different options and settled on Talos for Kubernetes not only because it was immutable and secure but also because I found the no-SSH and API-only-driven approach fascinating to learn and test.

#### Step - 1: Setting up Proxmox Virtual Environment in my Peladn Mini PC and other PCs

I created a bootable USB with [Proxmox Virtual Environment 9.2](https://proxmox.com/en/downloads/proxmox-virtual-environment) using [Rufus](https://rufus.ie/en/) and replaced the default vendor-provided Windows 11. Replicated the same for my Intel NUC, GMKtek Evo X2, and my main i9 server, which was still running Ubuntu with Docker up to this point. The Dell R610 was already on Proxmox 8 from earlier, so instead of a fresh install I just upgraded it in place to Proxmox 9 with `apt`. This was though a bit time-consuming but was a straightforward setup.

**Note**: *Proxmox VE doesn't like Wi-Fi, and anyways the best practice for a 24x7 lab environment is to connect an Ethernet cable. Initially during my setup I still didn't have the Lanner SED firewall and just had a simple home router connected to an 8-Gigabit-Ethernet-port unmanaged switch.*

#### Step - 2: Creating a GitHub Repo

One of the objectives I had while doing this setup was to create a Public GitHub Repo for maintaining my setup and use GitOps via Flux or Argo CD to manage the services in Kubernetes.

So the Git repo was created via the UI: [Homelab-ops GitHub](https://github.com/deepchhaiya/Homelab-ops) and then cloned to my Windows laptop that I was using to access and set up the lab.

```bash
git clone https://github.com/deepchhaiya/Homelab-ops.git

cd Homelab-ops
```

**Note:** I used VSCode to create a workspace where I created directory structure for the setup and files that I would need for the setup. I started taking notes by dividing them into phases. For each phase doc, I did set up a goal and noted implementation, code, challenges and decisions. I am using those docs and aligning each part in this blog post series, starting with [Part 1]({% post_url 2026-09-01-building-a-power-efficient-homelab-part-1 %}).

#### Step - 3: Setting up my local machine/laptop with SOPS, Talosctl and Kubectl

Before anything from that repo can go into Flux or Argo CD, it needs a way to hold secrets safely — that's what this step is for. I implemented a **Zero-Trust** workstation model. My infrastructure secrets are encrypted using SOPS and Age. The private keys are never stored on disk. I use the Bitwarden CLI to fetch the keys from my self-hosted Vaultwarden directly into the environment memory only during the encryption process.

![Zero-trust secrets flow: age-keygen generates a keypair once, the private key is stored in Vaultwarden, the Bitwarden CLI pulls it into session memory only, SOPS uses that in-memory key to encrypt or decrypt files, and only the encrypted files are committed to the GitHub repo.](/assets/images/sops-secrets-flow.svg)

So below are the steps to install the needed tools:

1. Installed using **[winget](https://learn.microsoft.com/en-us/windows/package-manager/winget/)** commands: [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/) and [talosctl](https://winget.run/pkg/Sidero/talosctl)

    Install talosctl:

    ```powershell
    winget install -e --id Sidero.talosctl
    ```

    Install kubectl:

    ```powershell
    winget install -e --id Kubernetes.kubectl
    ```

2. Installed `age-keygen` and the `bw` (Bitwarden) CLI into my laptop (management machine) so I can bridge them to Vaultwarden (Self-hosted).

    **Windows (using PowerShell/Winget):**

    ```powershell
    winget install FiloSottile.age
    winget install Bitwarden.CLI
    ```

    **If you are on macOS (using Homebrew):**

    ```bash
    brew install age
    brew install bitwarden-cli
    ```

3. Generated and saved the key in Vaultwarden

Once the tools are installed on my laptop, I followed these steps to link them to my **Vaultwarden**:

1. **Generate the key pair:**

   ```bash
   age-keygen -o dev-key.txt
   ```

2. **Open the file** (`dev-key.txt`). It will look like this:
   - `# public key: age1...` (This is safe to share/put in GitHub).
   - `AGE-SECRET-KEY-1...` (This is your **Master Secret**).

3. **Upload to Vaultwarden:**
   - Log into your Vaultwarden web UI or Bitwarden.
   - Create a **New Item** > **Secure Note**.
   - Name it `HOMELAB_SOPS_KEY`.
   - Paste the **entire content** of `dev-key.txt` into the Note field.

4. **Secure Delete:** Delete `dev-key.txt` from the laptop.

Now, whenever I need to encrypt file content or create a secret for a GitHub repo, I don't need the key stored on my disk. I fetch it into my session memory using the Bitwarden CLI.

**The command to encrypt a file without saving the key to disk:**

I used both bash ([Git CLI](https://git-scm.com/)) and Windows PowerShell, so I covered both approaches.

```bash
# 1. Log into your Vaultwarden (replace with your URL)
bw config server https://your-vaultwarden-url.com
bw login

# 2. Get the key and encrypt your secret
export SOPS_AGE_KEY=$(bw get notes "HOMELAB_SOPS_KEY")
sops --encrypt --age $(age-keygen -y <<< "$SOPS_AGE_KEY") secret-file.yaml > secret-file.enc.yaml
```

For Windows (PowerShell):

```powershell
# 1. Log into your Vaultwarden (replace with your URL)
bw config server https://your-vaultwarden-url.com
bw login

# 2. Unlock the vault and set the session token
$env:BW_SESSION = $(bw unlock --raw)

# 3. List items to find your note's exact name
bw sync
bw list items --search "HOMELAB" | ConvertFrom-Json | Select-Object name, id

# 4. Once you confirm the name, fetch the age key from Vaultwarden into an environment variable
$env:SOPS_AGE_KEY = $(bw get notes "HOMELAB_SOPS_KEY")
```

**Quick test — encrypt `test-secrets.yaml`:**

```powershell
# After steps 1-3 above, run:
sops --encrypt --in-place test-secrets.yaml

# Verify it worked — you should see "sops" metadata and encrypted values:
cat test-secrets.yaml

# To decrypt and view:
sops --decrypt test-secrets.yaml
```

## Where Things Stand After This Part

Proxmox is on the Peladn, the Intel NUC, the GMKtek EVO-X2, and the main i9 server, plus the Dell R610 upgraded in place to version 9. The [Homelab-ops](https://github.com/deepchhaiya/Homelab-ops) repo exists and is where GitOps will eventually manage everything. The SOPS key lives in Vaultwarden and retrieved when needed to encrypt the YAMLs or its content later.

What's left to do: Flux or Argo CD actually reading and decrypting these secrets from the repo. That's for a later part, once the cluster itself exists.

I'll cover Talos VM setup and Raspberry Pi setup in the upcoming part.