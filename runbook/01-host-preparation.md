# 01 — Host preparation

Get the Ubuntu host ready to run KVM/libvirt VMs. Nothing about the
guest, the robot, or the network is touched yet — this is just the
hypervisor stack and a sanity check that virtualization works.

## Goal

After this phase:

- The x86 hypervisor stack is installed: `qemu-system-x86` plus its
  display/SPICE/OpenGL module siblings, `qemu-utils`,
  `libvirt-daemon-system`, `virt-manager`, `virtinst`, `cpu-checker`,
  and `ovmf`. (The Ubuntu libvirt how-to's `qemu-kvm` recommendation
  is broken on Noble — see Step 2 for why.)
- Your user is in the `libvirt` and `kvm` groups, with the new
  groups effective (after logout/login).
- `kvm-ok` reports KVM is usable.
- `virt-host-validate qemu` reports all checks PASS (or only
  benign WARNs).
- libvirt's default NAT network `virbr0` is active and autostarts
  on boot.
- `qemu-system-x86_64 --display help` shows `gtk` available; QEMU
  accepts `gl=on` (so the guest's virtio-gpu can do 3D in Phase 2).

## Why libvirt (and not Multipass)?

Ubuntu's [official QEMU page][ubuntu-qemu] editorially prefers
Multipass and UVtool over manual QEMU/libvirt for everyday VM use.
This project deliberately uses **libvirt** because:

- **USB pass-through control.** The U2D2 (and later the RealSense)
  need precise libvirt `<hostdev>` XML. Multipass doesn't expose
  this in a first-class way.
- **Canonical domain XML.** Per `../CLAUDE.md`, the VM definition
  is checked into `vm/` so anyone can recreate the VM with
  `virsh define`. Multipass abstracts the underlying libvirt XML
  away.
- **Cross-host shared qcow2 + Phase 6 (`virt-clone`).** These
  workflows are libvirt-native; Multipass doesn't support them in
  the same form.
- **Classic ISO install.** Trossen's `xsarm_amd64_install.sh -d
  humble` was tested against conventional Ubuntu installs, not
  cloud-init images.

If you're forking this runbook for a project that doesn't have
those requirements, Multipass really is simpler — see [Ubuntu's
Multipass doc][ubuntu-multipass].

## Sources

The package selections, group memberships, and verification commands
below come primarily from these Ubuntu pages, with small additions
specific to our project (called out inline):

- [Ubuntu QEMU how-to][ubuntu-qemu]
- [Ubuntu libvirt how-to][ubuntu-libvirt]
- [Ubuntu virt-manager how-to][ubuntu-virtmgr]

The follow-up [GPU virtualisation with QEMU/KVM][ubuntu-gpu] page
covers several approaches for in-guest graphics. We use one of them:
**virtio-gpu with virgl3D** for in-guest 3D acceleration (rviz and
similar ROS 2 GUI tooling). Quoting the page:

> "Use `-vga virtio` with a local display having a GL context
> `-display gtk,gl=on`. This will use virgil3d on the host, and
> guest drivers are needed (which are common in Linux since Kernels
> >= 4.4...)."

The Ubuntu 22.04 guest's kernel (5.15) and our Ubuntu 24.04 host's
kernel both satisfy that requirement. The host-side library is
`libvirglrenderer1` (added to the install list in Step 2).

**Advanced graphics extensions (deferred — to revisit later):** the
Ubuntu page also covers higher-performance approaches we are *not*
using in the initial setup but that you may want to explore later if
rviz performance is inadequate, if you add CUDA / Isaac / heavy 3D
workloads, or if you simply want native-grade graphics in the guest:

- **GPU passthrough via VFIO** — assigns a dedicated host GPU to the
  guest. Native performance. Requires IOMMU (VT-d/AMD-Vi) enabled in
  firmware, a discrete GPU, kernel parameter changes
  (`intel_iommu=on` or `amd_iommu=on`), and host driver blacklisting.
- **Intel GVT-g (mediated devices)** — splits an Intel iGPU into
  multiple virtual GPUs. Only supported on certain Intel generations
  (Broadwell through some Coffee Lake; Intel has since deprecated it
  in favor of SR-IOV on newer chips).
- **NVIDIA vGPU** — vendor-licensed vGPU on supported professional
  cards (Quadro, Tesla, certain RTX). Requires NVIDIA's commercial
  vGPU manager package.
- **Looking-Glass** — low-latency framebuffer share between guest and
  host display, typically paired with VFIO passthrough for GPU + LG
  for display so you keep one physical screen.

These would each be its own phase to add (host hardware audit, BIOS
changes, kernel parameters, libvirt XML changes, separate verification).
For the initial PincherX-100 control path, virtio-gpu + virgl3D is
sufficient; we lock that in now and treat the above as a future
exploration track.

[ubuntu-qemu]: https://ubuntu.com/server/docs/how-to/virtualisation/qemu/
[ubuntu-libvirt]: https://ubuntu.com/server/docs/how-to/virtualisation/libvirt/
[ubuntu-virtmgr]: https://ubuntu.com/server/docs/how-to/virtualisation/virtual-machine-manager/
[ubuntu-multipass]: https://ubuntu.com/server/docs/how-to/virtualisation/multipass/
[ubuntu-gpu]: https://ubuntu.com/server/docs/how-to/graphics/gpu-virtualization-with-qemu-kvm/

## Prerequisites

- Ubuntu 24.04 LTS host install completed and updated.
- Sudo access on the host.
- Internet access on the host (Wi-Fi is fine — the host's network
  is not touched in this phase).

## Adapt

| Aspect | This runbook | If yours differs |
|---|---|---|
| Host distro | Ubuntu 24.04 LTS | Fedora: `dnf install qemu-kvm libvirt virt-manager virt-install edk2-ovmf`. Arch: `pacman -S qemu-base libvirt virt-manager dnsmasq edk2-ovmf`. Concepts and verification commands are the same. |
| CPU vendor | Intel or AMD, supporting VT-x / AMD-V | Older CPU lacking VT-x/AMD-V: stop here, KVM will not work. TCG software emulation is too slow for ROS 2 control loops. |
| Dual-boot 26.04 | This runbook will be re-run on the 26.04 host | The package names are the same. The qcow2 cross-host hazard described in `../reference/cross-host-shared-qcow2.md` applies — read it before running both. |
| Headless host (no GUI) | Optional | Skip `virt-manager`. Use `virt-install` (from `virtinst`) for guest creation and `virsh console` to interact. |

---

## Step 1 — Confirm the CPU supports hardware virtualization

**Why:** KVM is a thin shim on top of CPU virtualization extensions
(Intel VT-x, AMD-V). If the CPU doesn't expose them — or the
firmware/BIOS has them disabled — KVM cannot run, and any apt
install you do is wasted effort. This is a 1-second pre-check
before touching the package manager.

```sh
$ egrep -c '(vmx|svm)' /proc/cpuinfo
```

**Verify:** the number printed should be **greater than 0** —
typically equal to the number of logical CPU cores. `vmx` is
Intel; `svm` is AMD.

```sh
$ lscpu | grep -i virtual
```

Should also show a `Virtualization:` line with `VT-x` or `AMD-V`.

**Watch out:**

- If `egrep` returns `0` but `lscpu` says your CPU model supports
  VT-x/AMD-V: the extension is disabled in BIOS/UEFI. Reboot,
  enter firmware setup (typically `F2` / `F10` / `Del`), find the
  setting (often "Intel Virtualization Technology", "AMD-V", "SVM
  Mode"), enable it, save and reboot, and re-run the check.
- If `lscpu` shows no virtualization support at all, the CPU is
  too old. Stop — KVM won't work on this hardware.

---

## Step 2 — Inventory and install the hypervisor stack

### Step 2a — Pre-flight inventory (do this first)

**Why:** many Ubuntu installs already have parts of the QEMU/libvirt
stack — Kubuntu desktop preinstalls a chunk of it, a prior libvirt
session leaves the rest, and Ubuntu Server has its own variants. A
30-second check tells you exactly what's already there so you and
any reader (human or AI) can install only what's missing. The full
install command in Step 2b is idempotent (apt no-ops on packages
already at the latest version), but knowing your starting state is
worth doing for everyone who follows this runbook.

> **Note on `qemu-kvm`:** Ubuntu's libvirt how-to recommends `apt
> install qemu-kvm libvirt-daemon-system`. **That command fails on
> Ubuntu 24.04 (Noble)** — the `qemu-kvm` binary package was removed.
> The current correct binary is `qemu-system-x86`, which is what
> we use below. KVM acceleration is *compiled into* that binary;
> there is no separate "kvm" binary anymore.

```sh
$ apt list --installed 2>/dev/null | grep -E \
    '^(qemu-system-x86|qemu-system-gui|qemu-system-modules-opengl|qemu-system-modules-spice|qemu-utils|libvirt-daemon-system|libvirt-clients|libvirglrenderer1|cpu-checker|ovmf|virt-manager|virtinst|virt-viewer|swtpm)/' \
    | sort
$ for cmd in qemu-system-x86_64 virsh kvm-ok virt-host-validate qemu-img \
             virt-manager virt-install virt-clone virt-viewer; do
    printf "%-22s " "$cmd:"; command -v "$cmd" 2>/dev/null || echo "NOT FOUND"
  done
$ ls -d /usr/share/OVMF 2>/dev/null && ls /usr/share/OVMF/ 2>/dev/null
$ lsmod | grep -E '^kvm'
$ ls -l /dev/kvm 2>/dev/null
$ groups
```

**Verify** (interpret as you go):

- Each `qemu-*`, `libvirt-*`, `cpu-checker`, `ovmf` line in the apt
  list output → that package is already installed; **skip it** in
  Step 2b's install command.
- Each "NOT FOUND" in the binary-presence loop → the providing
  package is missing; **include it** in Step 2b.
- `/dev/kvm` should exist with group `kvm`. `lsmod` should show
  `kvm_intel` (Intel hosts) or `kvm_amd` (AMD hosts).
- `groups` should — eventually — list both `libvirt` and `kvm`.
  If they're missing here, Step 3 fixes that.

For a fresh-from-ISO Ubuntu 24.04 install, expect *almost everything*
to be missing and Step 2b's install command to do the bulk of the
work. For Kubuntu desktop or any host that's run libvirt before,
expect substantial overlap and a much shorter list of new packages.

### Step 2b — Install (or top up) the stack

**Why:** each package serves a specific purpose. The full list is
spelled out below so the install is reproducible across hosts even
if their apt config has `APT::Install-Recommends "false"`.

- `qemu-system-x86` *(the actual x86 emulator binary; replaces the
  removed `qemu-kvm`)* — provides `qemu-system-x86_64` with KVM
  acceleration compiled in. Per [packages.ubuntu.com][pkg-qsx], its
  Recommends are exactly the next four packages in this list, so on a
  default-apt host you could omit them, but spelling them out makes
  this command portable to apt configs that disable Recommends.
- `qemu-system-gui` *(per Ubuntu GPU-virtualisation page)* — provides
  the GTK display backend so we can use `-display gtk,gl=on`.
- `qemu-system-modules-opengl` *(per Ubuntu GPU-virtualisation page)*
  — provides virgl3D OpenGL passthrough modules; depends on
  `libvirglrenderer1`. This is what makes rviz in the guest work
  without GPU passthrough.
- `qemu-system-modules-spice` *(per Ubuntu GPU-virtualisation page)*
  — SPICE display modules; libvirt's default display protocol is
  SPICE.
- `qemu-utils` — `qemu-img` (image manipulation, used in Phase 2 + 6)
  and `qemu-nbd` (mount qcow2 images as block devices for inspection).
- `cpu-checker` *(implied by Ubuntu QEMU how-to)* — provides the
  `kvm-ok` verification command used in Step 4.
- `ovmf` *(project-specific)* — UEFI firmware (EDK II) for q35
  guests; Phase 2 will boot a UEFI guest.
- `libvirt-daemon-system` *(from Ubuntu libvirt how-to)* — `libvirtd`
  as a systemd service. Pulls in `libvirt-clients` (which provides
  `virsh` and `virt-host-validate`) transitively.
- `virt-manager` *(from Ubuntu virt-manager how-to)* — the GTK GUI
  for managing VMs interactively.
- `virtinst` *(from Ubuntu virt-manager how-to)* — provides
  `virt-install`, `virt-clone` (used in Phase 6), `virt-xml`,
  `virt-sparsify`.

[pkg-qsx]: https://packages.ubuntu.com/noble/qemu-system-x86

```sh
$ sudo apt update
$ sudo apt install -y \
    qemu-system-x86 \
    qemu-system-gui \
    qemu-system-modules-opengl \
    qemu-system-modules-spice \
    qemu-utils \
    cpu-checker \
    ovmf \
    libvirt-daemon-system \
    virt-manager \
    virtinst
```

**If your inventory in Step 2a showed packages already installed:**
running this command is still safe — apt prints "X is already the
newest version (...)" for each one. You can also pare the list down
to just what was NOT FOUND. Both produce the same final state.

**Verify:** the install completes without errors. Then:

```sh
$ systemctl is-active libvirtd
$ systemctl is-enabled libvirtd
```

Both should print `active` and `enabled` respectively. The
`libvirt-daemon-system` package starts and enables `libvirtd`
automatically.

**Watch out:**

- On a fresh Ubuntu 24.04 install, this download is ~250–400 MB.
- If `apt` complains about held packages or unmet dependencies,
  run `sudo apt full-upgrade` first to reconcile, then retry.
- AppArmor profiles for libvirt are installed and enforced by
  default on Ubuntu. If you hit AppArmor denials later (Phase 2
  or 4), check `sudo journalctl -k | grep -i apparmor` — but you
  shouldn't see any in this phase.

---

## Step 3 — Add your user to the `libvirt` and `kvm` groups

**Why:** by default, only root can talk to `libvirtd` over its
Unix socket and only root can open `/dev/kvm`. Adding your user
to these groups lets you run `virsh` / `virt-manager` and
hypothetically QEMU directly without `sudo`.

The Ubuntu libvirt how-to is explicit on this:

> "Add the user who will manage virtual machines to the `libvirt`
> group. Members of the sudo group are added automatically, but
> you must manually add other users who need access to system-wide
> libvirt resources." […] "This grants the user access to advanced
> networking options."

The Ubuntu QEMU how-to additionally notes: *"invoking QEMU manually
may sometimes require your user to be part of the `kvm` group"* —
relevant if you ever want to run `qemu-system-x86_64` directly
outside libvirt.

```sh
$ sudo adduser $USER libvirt
$ sudo adduser $USER kvm
```

(The single-command equivalent `sudo usermod -aG libvirt,kvm $USER`
is portable to non-Debian distros and works identically.)

**Verify:**

```sh
$ groups $USER
```

Should list both `libvirt` and `kvm` among your groups — but
**only after you log out and log back in.** A new login session
re-reads group memberships; existing shells, terminals, and the
desktop session still see the old set. The simplest reliable
approach is to log out of your desktop entirely and log back in.

**Watch out:**

- `newgrp libvirt` works for a single shell but won't fix
  graphical apps like virt-manager. Just log out / log in.
- If you skip this step, every `virsh` invocation needs `sudo`,
  and you'll keep wondering why virt-manager prompts for
  authentication. Doing it now saves friction throughout.

---

## Step 4 — Verify KVM is usable

**Why:** confirm end-to-end that the kernel module is loaded,
`/dev/kvm` exists with sane permissions, and your user can use it.

```sh
$ kvm-ok
```

**Verify:** output should be exactly:

```
INFO: /dev/kvm exists
KVM acceleration can be used
```

Also confirm the kernel module is actually loaded:

```sh
$ lsmod | grep kvm
```

You should see `kvm_intel` (Intel hosts) **or** `kvm_amd` (AMD
hosts), plus the generic `kvm` module.

**Adapt:** if `lsmod` shows neither, the kernel module didn't
auto-load. `sudo modprobe kvm_intel` (or `kvm_amd`) should fix it
and you can confirm with `lsmod | grep kvm` again. If `modprobe`
errors with "operation not supported", it's almost always BIOS —
re-check Step 1.

**Watch out:**

- `kvm-ok` uses heuristics; on rare hardware (e.g., very recent
  Intel chips before `cpu-checker` knows about them) it may say
  "KVM acceleration can NOT be used" while KVM actually works.
  In that case the truth is `lsmod | grep kvm_intel` showing the
  module loaded *and* `ls -l /dev/kvm` showing the device exists.

---

## Step 5 — Run the libvirt host validator

**Why:** `virt-host-validate` runs a battery of checks libvirt
itself uses to determine whether a host is suitable for
virtualization. It catches issues `kvm-ok` doesn't, like missing
IOMMU support (relevant later if we ever want PCI passthrough),
cgroup mountpoints, and AppArmor configuration.

```sh
$ virt-host-validate qemu
```

**Verify:** every line should end in `PASS`. A few `WARN` lines
are acceptable for our use case:

- `Checking for cgroup 'devices' controller support : WARN` — benign
  on Ubuntu 24.04. Noble uses systemd's cgroup **v2 unified
  hierarchy**, where libvirt does device access control via BPF
  (`devices.allow`/`devices.deny` files don't exist in v2). The
  legacy v1 `devices` controller this check looks for is therefore
  *expected* to be absent on Noble. Not actionable.
- `Checking for secure guest support : WARN (Unknown if this platform
  has Secure Guest support)` — benign. This checks for AMD SEV /
  Intel TDX confidential-computing extensions, which the validator
  can't always detect from userspace. We don't use SEV/TDX.
- `Checking for device assignment IOMMU support : WARN (Unknown if
  this platform has IOMMU support)` — only seen on hosts with IOMMU
  disabled or absent. Benign for this project (we don't do PCI
  passthrough; USB pass-through doesn't need IOMMU). On a modern
  Intel laptop with VT-d enabled in firmware, you'll see PASS here
  instead, which is also fine.

Any `FAIL` line must be resolved before continuing. The validator
output usually tells you the fix.

**Watch out:**

- `Checking for cgroup 'cpu' controller support: FAIL` (or
  similar) on systems where systemd's cgroup v2 hierarchy is set
  up unusually — Ubuntu 24.04 default install handles this; if
  you see it, your host has been customized.

---

## Step 6 — Verify (or load) the `vhost_net` kernel module

**Why:** `vhost_net` is the in-kernel data path for virtio-net.
Without it, every guest network packet traps to QEMU userspace —
slow. With it, packets flow directly between the guest's virtio
ring and the host's tap device in the kernel. We don't have
to do anything if the module is loaded; just confirm.

```sh
$ lsmod | grep vhost_net
```

**Verify:** at least one line printed (the module is loaded).

**Adapt / fix:** if nothing is printed:

```sh
$ sudo modprobe vhost_net
$ lsmod | grep vhost_net
```

To make it auto-load on boot:

```sh
$ echo 'vhost_net' | sudo tee /etc/modules-load.d/vhost_net.conf
```

**Watch out:** Ubuntu 24.04 normally auto-loads `vhost_net` once
libvirtd creates a tap interface (Phase 2). If it's not loaded
yet, that's fine — it'll come up automatically. The `modprobe`
above is a belt-and-suspenders move.

---

## Step 7 — Verify libvirt's default NAT network is active

**Why:** libvirt ships with a virtual network named `default`
that gives any guest attached to it NAT'd internet access via
`virbr0` — a host-side bridge that *is not* connected to your
physical NIC. Guests get IPs from libvirt's built-in dnsmasq
(typically `192.168.122.0/24`), and outbound traffic is NAT'd
through whichever physical interface has the host's default
route (your Wi-Fi).

This is exactly what we want: zero host-network configuration,
guests have internet access, multicast within the guest works on
its loopback, and we can add a single host→guest port-forward
later for the Flutter client without touching anything else. (See
`../CLAUDE.md` for the architectural reasoning.)

> **Background — `qemu:///system` vs `qemu:///session`.** libvirt
> has two connection URIs. The default is `qemu:///system` (the
> system-wide libvirtd, which owns `virbr0`, `/var/lib/libvirt/`,
> and persistent VM definitions; you need `libvirt` group
> membership to talk to it without sudo). The alternative
> `qemu:///session` runs per-user with reduced privileges, has its
> own separate network and VM list, and is **not** what this
> runbook uses. Every `virsh` / `virt-manager` command below
> implicitly targets `qemu:///system`. If you ever see "no networks"
> or "no VMs" unexpectedly, check you haven't accidentally pointed
> at the session URI.

```sh
$ virsh net-list --all
```

**Verify:** you should see exactly one network listed:

```
 Name      State    Autostart   Persistent
----------------------------------------------
 default   active   yes         yes
```

If `Autostart` is `no`, fix it:

```sh
$ sudo virsh net-autostart default
```

If `State` is `inactive`, start it:

```sh
$ sudo virsh net-start default
```

Inspect the network's IP range and DHCP scope (handy reference
for later phases):

```sh
$ virsh net-dumpxml default
```

You should see something like:

```xml
<network>
  <name>default</name>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>
```

Note the IP range — Phase 2 guests will get an address in
`192.168.122.0/24`.

**Watch out:**

- If `virsh` errors with `failed to connect to the hypervisor`
  followed by a permission denial, your group changes from Step
  3 didn't take effect. Log out and log back in, or run with
  `sudo virsh ...` for now.
- If `default` doesn't exist at all (rare on Ubuntu, but possible
  on minimal installs), recreate it from the canonical XML:

  ```sh
  $ sudo virsh net-define /etc/libvirt/qemu/networks/default.xml
  $ sudo virsh net-start default
  $ sudo virsh net-autostart default
  ```

---

## Step 8 — Verify QEMU has the GTK/GL display backend

**Why:** rviz and other 3D-accelerated GUI ROS 2 tooling will run
inside the guest. For 3D acceleration to work without doing GPU
passthrough, the guest's `virtio-gpu` device needs the host's QEMU
to expose a display backend with an OpenGL context — Ubuntu's GPU
virtualisation page recommends `-display gtk,gl=on` (or
`-display spice-app,gl=on` when using SPICE, which is what libvirt
uses by default with virt-manager).

This step confirms the QEMU binary you just installed has GTK
display + GL support compiled in. On Ubuntu 24.04 it does, but if
you ever rebuild QEMU or use a non-Ubuntu binary, this is the check.

```sh
$ qemu-system-x86_64 --display help
```

**Verify:** the output should list `gtk` (and typically also
`spice-app`, `vnc`, `none`, `egl-headless`, `dbus`).

```sh
$ qemu-system-x86_64 --display gtk,gl=on --version 2>&1 | head -1
```

**Verify:** prints the QEMU version with no parser error. (`--version`
makes QEMU exit immediately after parsing args; if `gl=on` were
unsupported you'd see "GL is not supported with this display.")

Also confirm the virgl renderer library is present:

```sh
$ apt list --installed 2>/dev/null | grep -i virglrenderer
```

**Verify:** at least `libvirglrenderer1/...` is listed as installed.

**Watch out:**

- If `gtk` is missing from the display list, your QEMU was built
  without GTK support — uncommon on Ubuntu but possible on minimal
  / server-without-recommends installs. `apt install qemu-system-gui`
  adds the GTK frontend.
- The `spice-app` backend requires `spice-client-glib-2.0`; on a
  desktop Ubuntu install with virt-manager already installed, this
  is satisfied.

---

## Step 9 — Grant `libvirt-qemu` access to the host DRM render node

**Why:** Phase 2 will configure a virtio-gpu device with virgl3D
acceleration. When QEMU starts the guest, it opens a host DRM
render node (`/dev/dri/renderD128` on a single-iGPU host) to set
up the EGL/GBM context that virglrenderer uses for offscreen GL
rendering. The render node is mode `0660 root:render` — only
members of the `render` group can open it.

QEMU under libvirt runs as the `libvirt-qemu` system user. By
default `libvirt-qemu` is in `kvm` but **not** `render`. Without
the `render` group, VM start fails with:

```
qemu-system-x86_64: egl: eglInitialize failed: EGL_NOT_INITIALIZED
qemu-system-x86_64: egl: render node init failed
```

This isn't documented on Ubuntu's GPU virtualisation page; it
surfaced empirically in Phase 2's first VM-start attempt. Add
`libvirt-qemu` to the `render` group and restart libvirtd so any
newly-spawned QEMU child inherits the updated supplementary group
set:

```sh
$ sudo usermod -aG render libvirt-qemu
$ sudo systemctl restart libvirtd
$ id libvirt-qemu
```

**Verify:** `id libvirt-qemu` output should include `render` (the
GID varies per host; `getent group render` shows yours) among
the supplementary groups, in addition to `kvm` and the user's
own primary group.

**Adapt:** on a hybrid-graphics host (laptop with Intel iGPU +
NVIDIA / AMD dGPU), there will be multiple render nodes
(`renderD128`, `renderD129`, …). The `render` group covers all of
them; which one virgl actually uses is selected per-VM via the
`<gl rendernode='...'/>` attribute on the `<graphics>` element —
see [Phase 2's pre-flight inventory](02-guest-provisioning.md)
and the canonical XML at `vm/pincherx-100-dev.xml` for the
hybrid-graphics handling.

**Watch out:**

- libvirtd must be restarted after the group change. QEMU child
  processes inherit supplementary groups via `initgroups()` at
  fork time; restarting libvirtd guarantees the next VM start
  picks up the new group set.
- This grants `libvirt-qemu` read+write on the render node, which
  is necessary for any 3D-accelerated SPICE display. It does *not*
  grant access to the card node (`/dev/dri/cardN`), so it does
  not enable display-driver-level operations like modesetting —
  exactly the surface we want.

---

## Step 10 — Smoke-test virt-manager (optional but recommended)

**Why:** confirms the GUI works and your group memberships are
effective in the desktop session.

```sh
$ virt-manager &
```

**Verify:** virt-manager opens, with a single connection labelled
`QEMU/KVM` and status `Connected`. No "user not authorized"
prompts. The connection inspector should show 0 VMs (we
haven't created one yet) and the `default` network in the
"Virtual Networks" tab under the connection's properties.

**Watch out:**

- A "user is not member of group libvirt" dialog means Step 3
  hasn't taken effect — see the logout/login note there.
- If virt-manager prompts for the root password every time, you
  haven't been added to `libvirt` (or your session is stale).

---

## Done — Phase 1 exit criteria

Tick all of these before moving on:

- [ ] `egrep -c '(vmx|svm)' /proc/cpuinfo` returns > 0
- [ ] `kvm-ok` says "KVM acceleration can be used"
- [ ] `lsmod | grep kvm_intel` (or `kvm_amd`) shows the module loaded
- [ ] `groups $USER` shows `libvirt` and `kvm`
- [ ] `systemctl is-active libvirtd` returns `active`
- [ ] `virt-host-validate qemu` shows no FAILs
- [ ] `virsh net-list --all` shows `default` as `active` + `autostart yes`
- [ ] `qemu-system-x86_64 --display help` lists `gtk`; the binary accepts `gl=on`
- [ ] `apt list --installed | grep virglrenderer` shows `libvirglrenderer1`
- [ ] `id libvirt-qemu` lists `render` among supplementary groups
- [ ] `virt-manager` opens and connects to QEMU/KVM cleanly

If you're following this on the second host (Ubuntu 26.04), run
the entire phase identically — no per-host substitutions in this
phase.

## Next

[`02-guest-provisioning.md`](02-guest-provisioning.md) — create
the Ubuntu 22.04 LTS guest with virtio storage, virtio NIC on the
default NAT, a `qemu-xhci` controller for USB pass-through, and a
`virtio-gpu` video device with virgl3D acceleration for in-guest
rviz.
