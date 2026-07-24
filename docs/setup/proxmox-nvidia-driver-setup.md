# Proxmox Host — NVIDIA Driver Setup (GTX 1060)

> **Status:** Complete — host driver installed and verified
> **Last updated:** 2026-07-24

---

## Context / Goal

Install the NVIDIA proprietary driver on the Proxmox VE 9 (Debian trixie) host itself, as a prerequisite for LXC cgroup GPU passthrough into the local AI inference stack (see `local-ai-stack.md`). The host owns the driver; containers only need a matching CUDA userspace, not the kernel driver.

Hardware: NVIDIA GeForce GTX 1060 6GB. Host kernel at time of install: `7.0.6-2-pve`.

---

## Decisions Log

### Repo configuration for Debian trixie / PVE 9

PVE 9 uses DEB822-format `.sources` files (not the legacy `.list` format) under `/etc/apt/sources.list.d/`. Two files matter here:

- `debian.sources` — the Debian base repos
- `pve-no-subscription.list` — the Proxmox repo (legacy format, still in use)

**Learning:** `non-free non-free-firmware` components belong on the **Debian** repo lines, not the Proxmox repo line. Adding them to the Proxmox line produces `Warning: ... doesn't have the component 'non-free'` errors on `apt update`, since Proxmox's own repo doesn't carry those components. If `apt install nvidia-driver` reports no installation candidate, first check that `debian.sources` actually has `non-free` listed — a fresh PVE 9 install may already have `non-free-firmware` but not `non-free`.

**Learning:** In practice, `apt install nvidia-driver` was not the path used in the end — the Debian-packaged driver had compatibility gaps reported against recent PVE 9 kernels in community forum threads. The NVIDIA `.run` installer with `--dkms` was used instead (see below). DKMS ensures the kernel module rebuilds automatically on future kernel upgrades.

### Driver version and source

- **Driver used:** `580.173.02`, downloaded from `https://download.nvidia.com/XFree86/Linux-x86_64/580.173.02/NVIDIA-Linux-x86_64-580.173.02.run`
- **Released:** 2026-06-26 (legacy 580 branch for Maxwell/Pascal/Volta GPUs, including the GeForce 10 series; primarily bug fixes — notably a DKMS build-failure fix relevant to manual `nvidia-installer` use)
- **Stored at:** `/NVIDIA-Linux-x86_64-580.173.02.run` on the Proxmox host (root directory `/`)
- Always check `https://download.nvidia.com/XFree86/Linux-x86_64/latest.txt` for the current version rather than assuming 580.173.02 is still current when repeating this process.

### etckeeper

`etckeeper` was installed on the Proxmox host to git-track `/etc`. It auto-commits on package install/remove and can be committed to manually. **Reminder: run `git -C /etc status` / commit manually before and after any manual edits under `/etc`** (repo configs, modprobe configs, udev rules, etc.) whenever touching this host in the future.

---

## Step-by-Step: Host Driver Install

### 1. Fix APT repos

Edit `/etc/apt/sources.list.d/debian.sources` so **both** `Components:` lines read:

```
Components: main contrib non-free non-free-firmware
```

Ensure `/etc/apt/sources.list.d/pve-no-subscription.list` contains only:

```
deb http://download.proxmox.com/debian/pve trixie pve-no-subscription
```

(no `non-free`/`non-free-firmware` on this line)

```bash
apt update
```

### 2. Install prerequisites

```bash
apt install pve-headers-$(uname -r) gcc make build-essential
```

`pve-headers` matching the running kernel is required for DKMS to build the kernel module.

### 3. Blacklist nouveau (if not already)

Check first:

```bash
cat /etc/modprobe.d/blacklist.conf 2>/dev/null | grep nouveau
```

The NVIDIA installer will attempt to do this automatically in step 5 if not already done — this manual step is only needed if you want to pre-empt the reboot-and-retry cycle described below.

### 4. Download the installer

```bash
cd /
wget https://download.nvidia.com/XFree86/Linux-x86_64/580.173.02/NVIDIA-Linux-x86_64-580.173.02.run
chmod +x NVIDIA-Linux-x86_64-580.173.02.run
```

### 5. Run the installer — expect a two-pass process

```bash
./NVIDIA-Linux-x86_64-580.173.02.run --dkms
```

**On a host where nouveau is still loaded (the normal first-run case), this will fail.** The installer detects nouveau, writes blacklist configs to disable it (`/usr/lib/modprobe.d/nvidia-installer-disable-nouveau.conf` and `/etc/modprobe.d/nvidia-installer-disable-nouveau.conf`), rebuilds the initramfs, and then aborts — because nouveau is still active in the *running* kernel and can't be unloaded live. This is expected, not a real error.

```bash
reboot
```

After reboot, confirm nouveau is gone:

```bash
lsmod | grep nouveau   # should print nothing
```

Re-run the installer — this time it should complete:

```bash
./NVIDIA-Linux-x86_64-580.173.02.run --dkms
```

### 6. Verify

```bash
nvidia-smi
```

Should show the GTX 1060, driver version 580.173.02.

### 7. Load NVIDIA modules at boot + udev rules

```bash
cat >> /etc/modules-load.d/modules.conf << 'EOF'
nvidia
nvidia_uvm
EOF

update-initramfs -u -k all
```

```bash
cat > /etc/udev/rules.d/70-nvidia.rules << 'EOF'
KERNEL=="nvidia", RUN+="/bin/bash -c '/usr/bin/nvidia-smi -L'"
KERNEL=="nvidia_uvm", RUN+="/bin/bash -c '/usr/bin/nvidia-modprobe -c0 -u'"
EOF
```

These ensure `/dev/nvidia*` device nodes reliably appear when the modules load — needed later for LXC bind-mounting.

### 8. Install the persistence daemon

Keeps the GPU initialized between uses; avoids re-init latency and state loss.

```bash
apt install nvidia-modprobe
mkdir -p /opt/nvidia && cd /opt/nvidia
git clone https://github.com/NVIDIA/nvidia-persistenced.git
cd nvidia-persistenced/init
./install.sh
reboot
```

Verify after reboot:

```bash
nvidia-smi
systemctl status nvidia-persistenced
```

`Persistence-M` should show `On` in the `nvidia-smi` output, and the service should be `active (running)`.

---

## Gotchas Encountered

- **`non-free` on the wrong repo line:** wastes an `apt update` cycle with confusing warnings. Fix is always: `non-free`/`non-free-firmware` → Debian repo lines only.
- **`.run` installer "failure" on first attempt is often not a failure:** if the log shows it wrote nouveau-blacklist configs and rebuilt initramfs before aborting, the fix is just `reboot` + re-run, not troubleshooting.
- **Debian's packaged `nvidia-driver` vs. the `.run` installer:** the `.run` installer with `--dkms` proved to be the more reliable, better-documented path for current PVE 9 kernels at the time of this install. Worth re-checking Debian package availability/compatibility if repeating this in the future, in case it has matured.

---

## Next Step: LXC GPU Wiring (prep notes for the next session)

Not yet executed, but the following is settled and should be treated as the starting point rather than re-derived from scratch:

**Container type:** Must be **privileged**. Unprivileged LXCs UID-shift device node ownership in ways that break CUDA access — this is not optional for this use case.

**Base template:** Not yet decided. Debian 12 (bookworm) is the safer choice — CUDA toolkit packages are best-tested against it. Debian 13 (trixie) is plausible but more bleeding-edge; matches the host OS but toolkit compatibility is less proven. Decide before creating the container.

**Config additions** (`/etc/pve/lxc/<CTID>.conf`) — find actual major numbers via `ls -la /dev/nvidia*` on the host first, these vary by driver version:

```ini
lxc.cgroup2.devices.allow: c <nvidia-major>:* rwm
lxc.cgroup2.devices.allow: c <nvidia-uvm-major>:* rwm

lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
```

**CUDA toolkit inside the container must match the host driver.** Host driver is 580.173.02 — check NVIDIA's CUDA compatibility table for the corresponding CUDA toolkit version before installing. Do **not** install `nvidia-driver` inside the container — the host owns the driver; the container only needs the toolkit/userspace libraries.

**Known gotcha — boot race:** If the LXC autostarts before `nvidia-uvm` is loaded on the host (e.g. right after a host reboot), the bind mount fails silently and CUDA errors out inside the container with no obvious cause. `nvidia_uvm` is already in `/etc/modules-load.d/modules.conf` (see above) which should make this reliable, but verify with `lsmod | grep nvidia_uvm` on the host before the LXC starts if this is ever revisited after a reboot.

**After wiring:** build llama-server with `-DGGML_CUDA=ON`, verify `nvidia-smi` inside the container shows VRAM climbing when a model loads.
