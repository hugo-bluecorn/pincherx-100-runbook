# 02 — Guest provisioning

Stand up the Kubuntu 22.04 LTS guest on top of the host KVM/libvirt
stack you set up in [Phase 1](01-host-preparation.md). End state is
a freshly installed, network-attached, 3D-accelerated VM with a
clean-install snapshot you can revert to.

## Goal

After this phase:

- A Kubuntu 22.04 LTS desktop VM named `pincherx-100-dev` is defined
  in libvirt, persistent (`virsh dumpxml pincherx-100-dev` returns
  the domain).
- The VM has 4 vCPU, 8 GiB RAM, a 60 GiB virtio-blk qcow2 disk, a
  virtio-net NIC on libvirt's default NAT network, a `qemu-xhci` USB
  controller, a virtio-gpu video device with virgl3D, and SPICE
  graphics.
- Kubuntu 22.04 LTS is installed inside the VM via Ubiquity's
  **Minimal installation** option, with the install ISO detached,
  boot order set to disk-only.
- spice-vdagent is installed inside the guest; clipboard sharing and
  resolution sync between host (via virt-manager) and guest work.
- 3D acceleration is verified inside the guest (`glxinfo` reports
  virgl as the renderer).
- A file-based clean-install backup exists (qcow2 + NVRAM) so any
  future destructive action can revert. Internal libvirt snapshots
  cannot be used: QEMU/libvirt does not support internal snapshots
  on VMs with pflash-based firmware (i.e. UEFI/OVMF VMs like ours),
  so we drop down a layer to `qemu-img convert` + `cp` of the NVRAM
  VARS file. See Step 7.

## Why this shape

- **Kubuntu 22.04 LTS desktop, minimal install** — chosen in
  `../CLAUDE.md` to match the Bluecorn end-user environment (KDE
  Plasma) while keeping the qcow2 small and the apt update churn
  low. ROS 2 Humble's Tier-1 platform is the Jammy *base*; KDE
  vs GNOME doesn't affect ROS 2 itself.
- **4 vCPU / 8 GiB / 60 GiB** — `interbotix_xsarm_control` is light;
  the guest needs headroom for `colcon build` (parallel C++ compiles
  for `interbotix_*` packages, MoveIt, ros2_control), rviz, and the
  Trossen install script's transient artifacts. 60 GiB is plenty
  for ROS 2 Humble + a workspace + future Zenoh bridge; under 30 GiB
  starts to feel cramped after the first few `apt upgrade` cycles.
- **virtio everywhere, q35 + UEFI** — minimum-overhead transports
  and the modern PCIe-based machine type. UEFI (OVMF) is what
  Kubuntu 22.04 expects when installed onto a modern target.
- **`qemu-xhci`** — required by Phase 4: the U2D2 (FT232H, USB 2.0)
  and RealSense D415 (USB 3.0, deferred) both physically attach via
  the host's USB 3.0 ports, so the guest needs a USB 3.0–capable
  controller. Project memory `project_host_usb_topology.md` covers
  the hardware reasoning.
- **virtio-gpu + virgl3D + SPICE** — the Phase 1 doc's graphics
  decision lands here as the actual `<video>` and `<graphics>` XML
  elements. No GPU passthrough; rviz acceleration via virgl is
  sufficient for a small URDF.
- **libvirt's default NAT network** — `../CLAUDE.md` ratifies this
  as the network architecture: all DDS multicast stays bounded to
  the guest, only Zenoh unicast (configured in Phase 7) leaves.

## Adapt

| Aspect | This runbook | If yours differs |
|---|---|---|
| Guest distro | Kubuntu 22.04 LTS desktop, **Minimal installation** | Stock Ubuntu/GNOME 22.04: same kernel + archive, same XML (just substitute the ISO filename and the user-creation screens). Ubuntu 22.04 *Server*: skip the desktop-only steps; you won't get rviz inside the guest unless you install `kubuntu-desktop` later. ROS 2 Humble works on all three. |
| RAM / vCPU | 8 GiB / 4 | If the host has ≥16 GiB RAM and is otherwise idle, 12 GiB / 6 vCPU is a comfortable bump. Below 6 GiB you'll start hitting OOM during `colcon build` of MoveIt. |
| Disk size | 60 GiB qcow2 | qcow2 is sparse — virtual size 60 GiB allocates only what the guest writes. Going smaller (40 GiB) is feasible but tight; larger (80–100 GiB) is fine and only pays the disk cost as you fill it. |
| Disk path | `/var/lib/libvirt/images/pincherx-100.qcow2` (libvirt's default location; AppArmor and ownership are pre-configured) | Custom path requires updating `/etc/apparmor.d/local/abstractions/libvirt-qemu` or adding the dir to `/etc/libvirt/qemu.conf` `cgroup_device_acl`. Avoid unless you have storage layout reasons. |
| Network | libvirt default NAT (`default`, `virbr0`) | A bridged setup would need a host bridge — see the Phase 1 commit history (we deliberately *removed* this requirement). Don't reintroduce it without revisiting `project_host_network_wifi.md`. |
| Host has Wi-Fi only | Fine | Bridged networking on Wi-Fi doesn't work (802.11 won't carry frames with the VM's source MAC). NAT is the only option — stay on this runbook's path. |
| ROS 2 distro | Humble (set in CLAUDE.md) | Jazzy/Lyrical would target Ubuntu 24.04, not 22.04 — different guest ISO and Trossen install flag. Out of scope for this project. |

## Sources

- libvirt domain XML format reference: https://libvirt.org/formatdomain.html
- libvirt firmware autoselection (`<os firmware='efi'>`): https://libvirt.org/formatdomain.html#bios-bootloader
- Ubuntu Server libvirt how-to: https://ubuntu.com/server/docs/how-to/virtualisation/libvirt/
- Ubuntu Server virt-manager how-to: https://ubuntu.com/server/docs/how-to/virtualisation/virtual-machine-manager/
- Ubuntu Server GPU virtualisation (virtio-gpu + virgl): https://ubuntu.com/server/docs/how-to/graphics/gpu-virtualization-with-qemu-kvm/
- Kubuntu 22.04 LTS download index (cdimage): https://cdimage.ubuntu.com/kubuntu/releases/22.04/release/
- Kubuntu 22.04 release notes (confirms Ubiquity, not Calamares — Kubuntu only switched to Calamares in 24.04): https://wiki.ubuntu.com/JammyJellyfish/ReleaseNotes/Kubuntu

The two canonical libvirt domain XMLs for this phase live in the repo:

- [`vm/pincherx-100-dev.xml`](../vm/pincherx-100-dev.xml) — post-install steady state (no install media, disk-only boot). **This is the source of truth for the VM definition.**
- [`vm/pincherx-100-dev-install.xml`](../vm/pincherx-100-dev-install.xml) — install-time variant (adds the Kubuntu ISO as a CDROM and boots from it first). Used once in Step 4, replaced by the canonical XML in Step 5a.

## Prerequisites

- [Phase 1](01-host-preparation.md) complete — `kvm-ok` green, you're
  in `libvirt` and `kvm` groups, `virsh net-list --all` shows
  `default` active and autostart.
- Internet to download the Kubuntu 22.04.5 LTS ISO (~3.9 GiB).
- ~70 GiB free under `/var/lib/libvirt/images/` (3.9 GiB ISO + the
  qcow2 grows toward 60 GiB virtual; allow ~30 GiB realistic
  near-term plus headroom for snapshots).

---

## Step 1 — Pre-flight inventory

**Why:** Same pattern as Phase 1's Step 2a — confirm the host is
ready before fetching anything large or making destructive changes.
Catches "default network not started," "image dir not writable,"
and "not enough disk" before they bite mid-install.

```sh
$ virsh net-list --all
$ ls -ld /var/lib/libvirt/images/
$ df -h /var/lib/libvirt/images/
$ id $USER | tr ',' '\n' | grep -E 'libvirt|kvm'
$ id libvirt-qemu | tr ',' '\n' | grep render
$ virsh list --all
$ sudo ls -la /var/lib/libvirt/images/   # sudo: dir is mode 0711 root:root by default
# Host DRM topology — hybrid-graphics detection:
$ ls /sys/class/drm/ | grep -E '^card[0-9]+($|-)' | sort
$ for f in /sys/class/drm/card*/device/uevent; do echo "=== $f ==="; cat "$f"; done
```

**Verify:**

- `virsh net-list --all` → `default` is `active` with `autostart yes`.
- `/var/lib/libvirt/images/` exists; ownership typically
  `root:root` (or `libvirt-qemu:kvm` depending on how libvirtd was
  started). The directory needs to be writable by `sudo` (we'll
  drop files there with `sudo` to keep ownership/AppArmor labels
  right).
- `df -h` shows ≥ 70 GiB free on the partition mounted at
  `/var/lib/libvirt/images/`.
- `id` output includes both `libvirt` and `kvm`.
- `id libvirt-qemu` output includes `render` (the system user that
  owns spawned QEMU processes needs read+write on the host's DRM
  render node for virgl3D EGL setup). If missing, see
  [Phase 1 Step 9](01-host-preparation.md#step-9--grant-libvirt-qemu-access-to-the-host-drm-render-node).
- `virsh list --all` is empty (no domain named `pincherx-100-dev`
  yet). If a stale `pincherx-100-dev` is listed from an aborted
  prior run, undefine it: `virsh destroy pincherx-100-dev 2>/dev/null
  ; virsh undefine --nvram pincherx-100-dev`.
- `/var/lib/libvirt/images/` doesn't already contain
  `pincherx-100.qcow2` or `kubuntu-22.04.5-desktop-amd64.iso`. If
  it does, you're either resuming this phase (skip ahead) or you
  have stale files from a prior attempt — verify with the user
  before deleting.
- **Single-GPU host** (one `cardN` entry, all `cardN-*` connectors
  on that card, one `renderD12N` render node): the canonical XML's
  `<gl enable='yes' rendernode='/dev/dri/renderD128'/>` works as-is.
- **Hybrid-graphics host** (two `cardN` entries, where one card
  has display connectors and the other has none): the connector-
  less card is a render-offload-only dGPU (Optimus/PRIME-style).
  libvirt's auto-pick of which render node to use for SPICE GL
  can land on the dGPU, whose EGL/GBM userspace may not be wired
  up for virgl — at VM start QEMU errors with
  `eglInitialize failed: EGL_NOT_INITIALIZED`. The canonical XML
  pins `rendernode='/dev/dri/renderD128'` explicitly (typically
  the iGPU). Confirm the iGPU's render-node number on your host
  with `ls -la /sys/class/drm/by-path/` — the
  `pci-<addr>-render` symlink that points at the card with
  connectors is the iGPU; substitute its `renderD12N` into the
  XML if it differs from `renderD128`.

**Watch out:**

- If `virsh` errors with `failed to connect to the hypervisor`,
  your group changes from Phase 1's Step 3 didn't take effect.
  Log out and back in (whole desktop session), retry.
- The default storage location is libvirt's convention; the
  AppArmor policy at `/etc/apparmor.d/abstractions/libvirt-qemu`
  whitelists exactly this path. Putting images elsewhere triggers
  AppArmor denials at VM start with cryptic "permission denied"
  errors in `/var/log/libvirt/qemu/<domain>.log`. Stay on the
  default path unless you have a specific reason to deviate.

---

## Step 2 — Download the Kubuntu 22.04 LTS desktop ISO and verify it

**Why:** ISOs from cdimage.ubuntu.com are signed. Verifying the
checksum (and the GPG signature on the checksum file) is the only
trustworthy way to confirm you're installing what you think you're
installing — particularly relevant given Kubuntu won't run an
unsigned EFI bootloader, and an attacker who tampered with the ISO
in transit could swap in a backdoored kernel before you ever boot
it.

The current Kubuntu 22.04 LTS point release at the time of writing
is **22.04.5**. Newer point releases (22.04.6+) will replace it on
the same URL when published; the SHA256SUMS verification confirms
whatever you actually downloaded.

### Step 2a — Download the ISO and the signed checksum

```sh
$ cd /tmp
$ wget https://cdimage.ubuntu.com/kubuntu/releases/22.04/release/kubuntu-22.04.5-desktop-amd64.iso
$ wget https://cdimage.ubuntu.com/kubuntu/releases/22.04/release/SHA256SUMS
$ wget https://cdimage.ubuntu.com/kubuntu/releases/22.04/release/SHA256SUMS.gpg
```

The ISO is ~3.9 GiB; on a typical home connection this takes 5–15
minutes. `wget`'s default progress bar is fine; if you'd prefer to
pipe it, `--progress=dot:giga` is more concise.

### Step 2b — Verify the GPG signature on SHA256SUMS

**Why:** SHA256SUMS by itself proves only that the file you have
matches a hash; SHA256SUMS.gpg + the Ubuntu CD signing key proves
*Canonical* published that hash.

```sh
$ gpg --keyid-format long --keyserver hkps://keyserver.ubuntu.com \
      --recv-keys 0xD94AA3F0EFE21092
$ gpg --verify SHA256SUMS.gpg SHA256SUMS
```

**Verify:** the `gpg --verify` output ends with:

```
gpg: Good signature from "Ubuntu CD Image Automatic Signing Key (2012) <cdimage@ubuntu.com>"
```

A `WARNING: This key is not certified with a trusted signature!` is
expected and *not* a verification failure — that's just gpg telling
you you haven't web-of-trust-signed Canonical's key (very few people
have). The "Good signature" line is what counts.

**Watch out:**

- If the keyserver fetch fails (`gpg: keyserver receive failed`),
  try `hkps://keys.openpgp.org` instead, or download the key from
  https://help.ubuntu.com/community/VerifyIsoHowto and import it
  manually with `gpg --import`.
- If `gpg --verify` reports `BAD signature`, **stop**. Don't use
  the ISO. Re-download from cdimage. If it fails twice, your
  network path is being tampered with — change networks and retry.

### Step 2c — Verify the ISO's SHA256

```sh
$ sha256sum -c SHA256SUMS --ignore-missing 2>&1 | grep desktop-amd64
```

**Verify:** prints exactly:

```
kubuntu-22.04.5-desktop-amd64.iso: OK
```

If you get `FAILED`, delete the ISO and re-download. Don't proceed.

### Step 2d — Move the ISO into libvirt's image directory

Use `install` rather than `mv` so the file lands as `root:root` in
one shot. (`sudo mv` only grants write access to the destination
directory; it doesn't chown the moved file — it inherits its
source ownership from `/tmp`, where you ran `wget` as your normal
user.)

```sh
$ sudo install -o root -g root -m 0644 \
       /tmp/kubuntu-22.04.5-desktop-amd64.iso \
       /var/lib/libvirt/images/
$ rm /tmp/kubuntu-22.04.5-desktop-amd64.iso     # install copies; remove the source
$ ls -lh /var/lib/libvirt/images/kubuntu-22.04.5-desktop-amd64.iso
```

`root:root 0644` matches what `qemu-img create` produces in the
next step, so all images in this directory have consistent
ownership. libvirt's default `dynamic_ownership` setting in
`/etc/libvirt/qemu.conf` will chown the file to its qemu user
at VM start anyway, so the at-rest ownership is mostly cosmetic
— but consistent is easier to reason about.

Confirm `dynamic_ownership` is at the default if you've ever
customized that file (it's `0600 root:root`, so use `sudo`):

```sh
$ sudo grep dynamic_ownership /etc/libvirt/qemu.conf
```

A default install has either no match (the line is commented out
in the shipped config, meaning the in-binary default of `1` is
used) or `dynamic_ownership = 1` if the line is uncommented.
Either is correct.

You can clean up the SHA256SUMS files now (`rm /tmp/SHA256SUMS*`)
or keep them for re-verification later.

---

## Step 3 — Pre-create the 60 GiB qcow2 disk

**Why:** the VM's primary disk is created on the host *before* libvirt
starts the guest. Pre-creating it (rather than letting virt-manager
or virt-install allocate it) lets us pin the cluster size and
options that matter for snapshot performance and host SSD wear.

```sh
$ sudo qemu-img create -f qcow2 \
       -o cluster_size=65536,lazy_refcounts=on \
       /var/lib/libvirt/images/pincherx-100.qcow2 60G
$ sudo qemu-img info /var/lib/libvirt/images/pincherx-100.qcow2
```

**Why each option:**

- `cluster_size=65536` (64 KiB) — qcow2 default; the right balance
  of metadata overhead vs. write amplification. Larger clusters
  worsen COW write amplification on snapshot-heavy workflows; we
  use snapshots heavily, so default it is. (Per project memory
  `qemu_block_qcow2_snapshots.md`, "Default 64 KiB is what we want;
  don't change.")
- `lazy_refcounts=on` — defers refcount writes to image-close time.
  Faster snapshots and faster shutdowns. Safe with clean shutdowns;
  on unclean shutdown, qcow2 marks the image dirty and rebuilds the
  refcount table on next open.

**Verify:** `qemu-img info` reports:

```
file format: qcow2
virtual size: 60 GiB (64424509440 bytes)
disk size: 196 KiB         (or similar — sparse, almost empty)
cluster_size: 65536
Format specific information:
    compat: 1.1
    compression type: zlib
    lazy refcounts: true
    refcount bits: 16
    corrupt: false
    extended l2: false
```

**Watch out:**

- If `qemu-img create` fails with `Permission denied`, you forgot
  the `sudo` — `/var/lib/libvirt/images/` is `0711` root-owned by
  default.
- The qcow2 is **sparse**: `disk size` is small now and grows as
  the guest writes. `du -h` will show real on-disk usage; `ls -l`
  shows the *virtual* size. Don't be alarmed by either number.

---

## Step 4 — Define the VM from the install-time XML and boot the installer

**Why:** the canonical XML at `vm/pincherx-100-dev.xml` defines the
*post-install* steady state — virtio disk, no CDROM, boot from disk.
For the first boot we need the install ISO attached and CDROM-first
boot order. The repo has a sibling XML at
`vm/pincherx-100-dev-install.xml` that contains exactly that
delta; we use it once, then replace with the canonical XML in
Step 7.

### Step 4a — Define the VM

From the repo root:

```sh
$ virsh define vm/pincherx-100-dev-install.xml
$ virsh list --all
$ virsh dumpxml pincherx-100-dev | grep -E '(<name|<memory|<vcpu|machine=|<source file|<model type)' | head -30
```

**Verify:**

- `virsh define` prints `Domain 'pincherx-100-dev' defined from
  vm/pincherx-100-dev-install.xml.`
- `virsh list --all` shows `pincherx-100-dev` in state `shut off`.
- `virsh dumpxml` reflects 8 GiB RAM, 4 vCPU, the qcow2 path,
  the ISO path, and `<model type='virtio'/>` for both disk and net.
  The machine type appears as `machine='pc-q35-X.Y'` inside the
  `<type arch='x86_64' ...>` element — libvirt writes it as an
  attribute, not its own element, which is why the grep above
  uses `machine=` rather than `<machine`.

### Step 4b — Start the VM and open the graphical console

The VM has SPICE graphics with `listen='none'` (Unix socket, local
only). Open the console with **virt-manager** — recommended — or
try `virt-viewer` with the caveat below:

```sh
$ virsh start pincherx-100-dev
$ virt-manager &        # GUI: double-click the running VM to open its console
# or:
$ virt-viewer --attach pincherx-100-dev &
```

The `--attach` flag tells virt-viewer to negotiate the SPICE socket
via libvirt's API rather than expecting a URI (which `<listen
type='none'/>` doesn't produce). Note: on at least Kubuntu 24.04
X11 + virt-viewer 11.0, the standalone viewer exhibits a SPICE-GL
rendering glitch — guest framebuffer renders as ghosted white with
edges visible only on window motion. `GDK_GL=disable virt-viewer
--attach …` does not fix it. **Falling back to virt-manager is
the documented workaround** until upstream virt-viewer's GTK-GL
widget initialization order is sorted out. virt-manager's embedded
SPICE display widget doesn't trip the same bug.

**Verify:** the VM window opens to the GRUB menu of the Kubuntu live
ISO, then auto-selects "Try or Install Kubuntu" and lands at the
Kubuntu live desktop.

**Watch out:**

- If the console window is black with `Guest has not initialized the
  display (yet)` for more than ~30 seconds, check
  `/var/log/libvirt/qemu/pincherx-100-dev.log` for QEMU errors.
  Most common cause is a typo in one of the XML paths (ISO or
  qcow2) — the file isn't openable, QEMU exited at start.
- If virt-manager complains about not finding the spice-vdagent
  channel target, that's expected — vdagent isn't installed yet
  inside the guest. Continue.

### Step 4c — Walk through the Ubiquity installer

Kubuntu 22.04's installer is **Ubiquity** with the KDE front-end
(Calamares became default only in Kubuntu 24.04, so don't expect
Calamares here). Click "Install Kubuntu" on the live desktop's
center to launch it.

Step-through (only screens with non-default choices listed; defaults
are fine for everything else):

1. **Welcome / Language** — your choice; defaults fine. → Continue.
2. **Keyboard layout** — defaults fine. → Continue.
3. **Updates and other software** — this is the screen with the
   minimal-install choice:
   - **Choose `Minimal installation`** (radio button). Drops
     LibreOffice, Krita, KMail, KOrganizer, games, multimedia
     extras. Keeps a full KDE Plasma desktop, the default browser,
     and core utilities. Saves ~1–2 GiB on the qcow2.
   - **Check** `Download updates while installing Kubuntu`.
   - **Check** `Install third-party software for graphics and Wi-Fi
     hardware…` if visible — irrelevant inside a VM (no Wi-Fi
     hardware to vendor-driver), but harmless.
   - → Continue.
4. **Installation type** — `Erase disk and install Kubuntu`.
   - We're inside a VM with a virtual 60 GiB blank disk; there is
     no other OS to coexist with.
   - **Do not enable** "Encrypt the new Kubuntu installation"
     (LUKS) — VM-level snapshots provide our recovery story, and
     in-guest LUKS would just add startup-prompt friction at every
     boot without security benefit (the qcow2 is at rest under the
     host's filesystem, which the user can encrypt at the host
     level if needed).
   - **Do not enable** ZFS-on-root — Kubuntu 22.04's KDE-front-end
     for Ubiquity didn't implement the ZFS option (per the Jammy
     release notes), and even if it had, it's deferrable.
   - → Install Now → Continue at the partition-write confirmation.
5. **Where are you?** — your time zone. → Continue.
6. **Who are you?** — set:
   - `Your name`: `developer` (or whatever you want).
   - `Your computer's name`: `pincherx100-dev`.
   - `Pick a username`: `developer`.
   - Password: pick one. **For a dev VM, "Log in automatically"
     is reasonable** — saves typing on every boot, the VM is
     gated by the host's login already. If the VM ever gets
     repurposed as a shared instance, change to "Require my
     password to log in" later.
   - → Continue.
7. **Install in progress** — ~10–25 minutes depending on disk and
   network. The slideshow plays; the install runs in the background.
8. **Installation complete** → click `Restart Now`. The installer
   then shows "Please remove the installation medium, then press
   ENTER:" — **do NOT press Enter.** The reboot will land back on
   the ISO live session because the install-time XML's CDROM has
   `<boot order='1'/>`; you would re-enter the Ubiquity welcome
   screen and be stuck in a loop. Instead, shut the VM down
   cleanly from the host:

   ```sh
   $ virsh shutdown pincherx-100-dev
   $ virsh list --all                       # wait for 'shut off'
   ```

   With the VM off, Step 5 swaps the install-time XML for the
   canonical XML (no CDROM, boot from disk) before re-starting.
   This is the order that actually works end-to-end on EFI VMs;
   the older "press Enter and let it reboot" advice assumed an
   `eject /dev/sr0`-only path that doesn't reset the libvirt boot
   order.

**Watch out:**

- If the installer can't reach the network for "Download updates
  while installing," the most common cause is libvirt's `default`
  network being in an odd state. From the host: `virsh net-list`
  → ensure it's `active`; restart with `sudo virsh net-destroy
  default && sudo virsh net-start default` if needed. Then
  reboot the VM.
- If you accidentally check `Encrypt the new Kubuntu installation`,
  power off the VM (`virsh shutdown pincherx-100-dev`), `virsh
  destroy pincherx-100-dev` if it doesn't shut down, and start
  Step 4b over — the qcow2 is freshly allocated and there's no
  state worth saving.
- The graphical console resolution may be small (1024×768) until
  spice-vdagent is installed inside the guest in Step 6 below.
  Don't try to fix it via Display Settings inside the guest at
  this stage.

---

## Step 5 — Adopt the canonical XML and verify the first boot from disk

**Why:** The VM is currently defined from
`vm/pincherx-100-dev-install.xml`, which has the ISO attached and
`<boot order='1'/>` on the CDROM. From here on, the canonical
`vm/pincherx-100-dev.xml` is the source of truth — it has no
CDROM and boots from the virtio disk. We swap the libvirt
definition over **before** the first post-install boot so the VM
comes up directly into installed Kubuntu, not back into the ISO
live session.

The VM should already be shut off from Step 4c step 8. Confirm:

```sh
$ virsh list --all                          # state must be 'shut off'
```

### Step 5a — Swap install-time XML for the canonical XML

Plain `virsh define vm/pincherx-100-dev.xml` fails because the
canonical XML deliberately omits `<uuid>` (libvirt auto-assigns
per host) — libvirt generates a fresh UUID, finds the name
`pincherx-100-dev` already taken by a different UUID, and refuses
with `domain already exists with uuid ...`. The correct pattern
is `undefine --keep-nvram` then `define`:

```sh
$ virsh undefine pincherx-100-dev --keep-nvram
$ virsh define vm/pincherx-100-dev.xml
$ virsh dumpxml pincherx-100-dev | grep -A1 -E '(<boot|<disk )'
$ virsh dumpxml pincherx-100-dev | grep cdrom
```

`--keep-nvram` preserves
`/var/lib/libvirt/qemu/nvram/pincherx-100-dev_VARS.fd` (the UEFI
variables, including the Kubuntu boot-entry registration written
during install). Required on EFI VMs — libvirt refuses a plain
`undefine` otherwise.

**Verify:**

- `virsh undefine` prints `Domain 'pincherx-100-dev' has been undefined`.
- `virsh define` prints `Domain 'pincherx-100-dev' defined from vm/pincherx-100-dev.xml.`
- The dumpxml grep shows `<boot dev='hd'/>` once, in `<os>`. No per-disk
  `<boot order='N'/>` tags.
- The `cdrom` grep is empty — the install media is gone from the
  definition.

> **Note — the `undefine --keep-nvram` + `define` pattern reappears.**
> This sequence is the canonical way to apply any structural change
> to the libvirt domain after the XML on disk is edited (since the
> XML is intentionally UUID-less for cross-host portability). You
> will use it again every time you edit `vm/pincherx-100-dev.xml`.
> File-based clean-install backups via `qemu-img convert` (Step 7)
> survive undefine — they live in `/var/lib/libvirt/images/`, not
> in libvirt's domain registry — so this pattern is safe.

### Step 5b — Start the VM and verify it boots from disk

```sh
$ virsh start pincherx-100-dev
$ virsh list                                # state must be 'running'
```

Open the console (virt-manager recommended; virt-viewer has the
SPICE-GL rendering glitch described in Step 4b).

**Verify (inside the guest):**

```sh
$ uname -a            # Linux ... 5.15.0-...-generic ... x86_64
$ lsb_release -d      # Description: Ubuntu 22.04.x LTS
$ cat /etc/issue      # contains "Kubuntu"
$ ip -br addr         # eth0 / enp1s0 with a 192.168.122.x address
$ ping -c 3 1.1.1.1   # reachable
$ ping -c 3 ubuntu.com
```

The guest gets its IP from libvirt's dnsmasq on `virbr0`. Expect a
192.168.122.0/24 address and clean ICMP both ways.

**Watch out:**

- `lsb_release -d` says `Ubuntu`, not `Kubuntu` — that's expected.
  Kubuntu and Ubuntu share an `os-release` of `Ubuntu`; the
  desktop is the only thing that differs. `cat /etc/issue` is the
  one that explicitly says "Kubuntu".
- If the IP is `169.254.x.x` (link-local APIPA), libvirt's dnsmasq
  didn't lease one. Check on the host: `sudo virsh net-dhcp-leases
  default` should show your VM. If not, `sudo virsh net-destroy
  default && sudo virsh net-start default` and reboot the VM.
- If `virsh start` errors with `eglInitialize failed`, the
  hybrid-graphics rendernode pin in `vm/pincherx-100-dev.xml`
  doesn't match your host's actual iGPU render node. Re-check
  the Step 1 pre-flight hybrid-graphics section and adjust the
  `<gl rendernode='/dev/dri/renderD12N'/>` attribute, then
  re-run the Step 5a undefine + define pattern.

---

## Step 6 — Install spice-vdagent and verify 3D acceleration

**Why:** two guest-side packages tighten the integration:

- `spice-vdagent` enables clipboard sharing, dynamic resolution
  resize, and host↔guest cursor mode in virt-manager. Without it,
  the SPICE channel we wired up in the XML is half-functional.
- `mesa-utils` provides `glxinfo`, which is how we *prove* virgl3D
  acceleration is actually working end-to-end (host
  libvirglrenderer ↔ guest virtio-gpu driver).

Inside the guest:

```sh
$ sudo apt update
$ sudo apt install -y spice-vdagent mesa-utils
$ sudo systemctl enable --now spice-vdagent
$ glxinfo -B | grep -E '(direct rendering|OpenGL renderer|OpenGL version)'
```

**Verify** the `glxinfo` output looks roughly like:

```
direct rendering: Yes
OpenGL renderer string: virgl
OpenGL version string: 4.x (Compatibility Profile) Mesa ...
```

The *renderer string* is the proof — it says `virgl` (or
`virgl-something`), confirming the guest's Mesa is rendering through
the virtio-gpu virgl path on the host. If it says `llvmpipe`, the
guest fell back to software rendering (something is wrong with
either the guest virtio-gpu driver or host virglrenderer); see
"Watch out" below.

After `spice-vdagent` is up, the VM window in virt-manager should
auto-resize when you drag its corners. If it doesn't, try
`Send Key → Ctrl+Alt+Delete` is *not* the way; instead, View →
Resize to VM in virt-manager.

**Watch out:**

- If `glxinfo` says `llvmpipe`: confirm on the host that
  `libvirglrenderer1` is installed (`apt list --installed | grep
  virgl`); confirm in `virsh dumpxml pincherx-100-dev` the
  `<video><model type='virtio'>` has `<acceleration accel3d='yes'/>`
  and `<graphics type='spice'>` has `<gl enable='yes'/>`. If both
  are right, restart the VM and recheck — virgl initialization
  happens at graphics-stack startup, which is too late if the
  guest already booted with software rendering.
- If `apt update` errors with `Could not resolve archive.ubuntu.com`,
  the network came up in step 5 but DNS is misbehaving. Restart
  the VM, retry. If still broken, check
  `/etc/resolv.conf` inside the guest — should point at 127.0.0.53
  (systemd-resolved) which forwards via libvirt's dnsmasq.

---

## Step 7 — Back up the clean install (qcow2 + NVRAM)

**Why:** Phase 3 (Trossen install) runs a long-lived install script
inside the guest that pulls in ~27 ROS 2 packages, creates a
catkin/colcon workspace, and edits files in `~/.bashrc`. If
anything in that phase goes wrong, reverting to a clean Kubuntu
install should be one copy command, not redoing Phase 2.

**Why not `virsh snapshot-create-as`?** libvirt's internal
snapshot machinery requires every block device the VM uses to be
qcow2. This VM uses UEFI firmware via OVMF, which libvirt models
as two `<pflash>` block devices (`OVMF_CODE_4M.ms.fd` and
`pincherx-100-dev_VARS.fd`). pflash files are *raw*, not qcow2,
so libvirt refuses with:

```
error: Operation not supported: internal snapshots of a VM with
pflash based firmware are not supported
```

This restriction applies regardless of whether you invoke the
snapshot via `virsh`, virt-manager's snapshot dialog, or the
libvirt API directly — they all hit the same `qemu-img`-level
constraint. External libvirt snapshots *do* work on EFI VMs, but
they scatter state across multiple files (original qcow2 + overlay
qcow2 + NVRAM), which complicates the cross-host dual-boot story
in `../CLAUDE.md`.

**Workaround:** drop down a layer and copy the qcow2 disk +
the NVRAM VARS file directly. The qcow2 stays a single
self-contained file (cross-host portable per
`qemu_block_qcow2_snapshots.md`), the NVRAM backup captures the
UEFI variable state including Kubuntu's boot-entry registration,
and reverting is symmetric `cp` commands.

Shut the VM down cleanly first — a backup taken while the guest
is running captures in-flight page-cache state and a partially
written disk:

```sh
$ virsh shutdown pincherx-100-dev
$ virsh list --all                          # wait until state is 'shut off'
```

Then back up both files:

```sh
$ sudo qemu-img convert -O qcow2 \
       /var/lib/libvirt/images/pincherx-100.qcow2 \
       /var/lib/libvirt/images/pincherx-100.clean-install.qcow2
$ sudo cp \
       /var/lib/libvirt/qemu/nvram/pincherx-100-dev_VARS.fd \
       /var/lib/libvirt/qemu/nvram/pincherx-100-dev_VARS.clean-install.fd
```

`qemu-img convert -O qcow2` produces a *compacted* copy: drops
freed clusters and rewrites the refcount table. A live ~21 GiB
qcow2 typically converts to a ~9–10 GiB backup after a fresh
Kubuntu minimal install. Faster than `cp` and the result is the
same shape (single qcow2 file).

**Verify** the backups landed:

```sh
$ sudo ls -lh /var/lib/libvirt/images/
$ sudo ls -lh /var/lib/libvirt/qemu/nvram/
```

> **Gotcha — sudo + globbing.** A natural-looking variant like
> `sudo ls -lh /var/lib/libvirt/images/pincherx-100*.qcow2` will
> fail with "No such file or directory" even when the file *is*
> there. The shell expands the glob as your normal user **before**
> sudo runs, but `/var/lib/libvirt/images/` is mode `0711` and
> `/var/lib/libvirt/qemu/nvram/` is mode `0700` — neither lets
> non-root read the directory contents, so the glob produces no
> matches and gets passed literally to ls. Either list the
> directory itself (above, no glob) or wrap the glob in a
> root shell:
>
> ```sh
> $ sudo bash -c 'ls -lh /var/lib/libvirt/images/pincherx-100*.qcow2'
> ```

Expect to see both `pincherx-100.qcow2` (live, may be ~20+ GiB after
the install) and `pincherx-100.clean-install.qcow2` (backup, ~9–10
GiB compacted) under `/var/lib/libvirt/images/`, and both
`_VARS.fd` (live) and `_VARS.clean-install.fd` (backup) under
`/var/lib/libvirt/qemu/nvram/`.

Restart the VM:

```sh
$ virsh start pincherx-100-dev
```

**Reverting later** (don't run now; reference for future phases):

```sh
$ virsh shutdown pincherx-100-dev          # or `virsh destroy` if hung
$ sudo cp /var/lib/libvirt/images/pincherx-100.clean-install.qcow2 \
          /var/lib/libvirt/images/pincherx-100.qcow2
$ sudo cp /var/lib/libvirt/qemu/nvram/pincherx-100-dev_VARS.clean-install.fd \
          /var/lib/libvirt/qemu/nvram/pincherx-100-dev_VARS.fd
$ virsh start pincherx-100-dev
```

This is a *cold revert* — no live RAM/CPU state is preserved. You
boot back into the installed Kubuntu desktop from a clean
power-on. Acceptable for a phase boundary; for runtime hot-swap
scenarios (which we don't have) something more elaborate would be
needed.

**Watch out:**

- Don't back up while USB pass-through devices are attached. Per
  project memory `qemu_usb_passthrough.md`, the guest's in-flight
  USB state goes into the qcow2 free space; a copy taken with an
  attached U2D2 may resume in a confused state. Phase 2 has no
  `<hostdev>` entries, so we're safe now — but the same backup
  pattern in Phase 4+ runbooks must detach USB first.
- Don't back up while `colcon build` or other heavy I/O is in
  progress — same reason: in-flight page-cache state. Always
  shut the VM down cleanly first.
- Both backup files stay `root:root` after the `sudo` commands
  above. libvirt's `dynamic_ownership` only relabels files
  referenced in a running domain; the backups aren't, so their
  ownership is stable.

---

## Done — Phase 2 exit criteria

Tick all of these before moving on to [Phase 3](03-trossen-install.md):

- [ ] `virsh list --all` shows `pincherx-100-dev` in state
      `running` or `shut off`.
- [ ] `virsh dumpxml pincherx-100-dev` matches `vm/pincherx-100-dev.xml`
      structurally (run `diff <(virsh dumpxml pincherx-100-dev | xmllint
      --format -) <(xmllint --format - < vm/pincherx-100-dev.xml)`
      to compare; expect only differences in auto-generated UUID,
      MAC address, PCI slot assignments, and SPICE socket path).
- [ ] Inside the guest, `lsb_release -d` reports Ubuntu 22.04.x
      (Kubuntu's `lsb_release` calls itself Ubuntu — confirm
      "Kubuntu" via `cat /etc/issue` instead).
- [ ] Inside the guest, `glxinfo -B | grep "OpenGL renderer"`
      contains `virgl`.
- [ ] Inside the guest, `pgrep -la spice-vdagent` shows both
      `spice-vdagent` and `spice-vdagentd`.
- [ ] Inside the guest, `ping -c 3 1.1.1.1` succeeds.
- [ ] `sudo ls -lh /var/lib/libvirt/images/` shows both
      `pincherx-100.qcow2` (live) and
      `pincherx-100.clean-install.qcow2` (backup).
- [ ] `sudo ls -lh /var/lib/libvirt/qemu/nvram/` shows both
      `pincherx-100-dev_VARS.fd` and
      `pincherx-100-dev_VARS.clean-install.fd`.

If you got here cleanly, this is the right time to commit and push
the runbook updates. Per project convention (`feedback_runbook_first.md`),
phase docs commit only after hardware verification:

```sh
$ cd /home/$USER/robots/pincherx_100/git/pincherx-100-runbook
$ git status                  # vm/*.xml + runbook/02-guest-provisioning.md
$ git add vm/ runbook/02-guest-provisioning.md
$ git commit -m "Add Phase 2 (guest provisioning) runbook + canonical libvirt XMLs"
$ git push origin main
```

## Next

[`03-trossen-install.md`](03-trossen-install.md) — run Trossen's
`xsarm_amd64_install.sh -d humble` inside the guest to install ROS 2
Humble, the `interbotix_*` workspace, and the udev rules. Take a
post-install snapshot before touching the U2D2 in Phase 4.
