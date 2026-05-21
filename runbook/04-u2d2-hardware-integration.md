# 04 — U2D2 hardware integration (USB pass-through to the guest)

Attach the Trossen U2D2 USB-to-serial adapter (FTDI FT232H,
vendor `0x0403`, product `0x6014`) to the development guest via
libvirt `<hostdev>` pass-through, with the Trossen udev rules
installed *both* on the host and the guest so `/dev/ttyDXL`
enumerates cleanly on both sides.

Unlike Phase 3, this phase has **host-side changes** (new
`/etc/udev/rules.d/` entry + canonical VM XML diff). Both pieces
land in version control.

## Goal

After this phase:

- The Trossen udev rules file
  `/etc/udev/rules.d/99-interbotix-udev.rules` is installed on the
  host, sourced from the locally cloned `interbotix_ros_core` repo
  (the same content the Phase 3 install script wrote inside the
  guest).
- With the VM **shut off** and the U2D2 plugged into any host USB
  port, `ls -l /dev/ttyDXL` on the host shows a symlink to a
  `ttyUSB*` device, and `udevadm info /dev/ttyDXL` reports the
  FT232H attributes. This proves the U2D2 itself is healthy and the
  host rules apply before pass-through enters the picture.
- The canonical libvirt domain XML at `vm/pincherx-100-dev.xml`
  contains a `<hostdev mode='subsystem' type='usb' managed='yes'>`
  block matching `0x0403:0x6014`, defined into libvirt with
  `virsh define`.
- With the VM **running**, `lsusb` inside the guest shows the FT232H
  and `ls /dev | grep ttyDXL` returns `ttyDXL` — the verbatim exit
  criterion from Trossen's
  [ROS 2 Standard Software Setup, Installation Checks](https://docs.trossenrobotics.com/interbotix_xsarms_docs/ros_interface/ros2/software_setup.html)
  page. (Same as `../CLAUDE.md` phase 4 requirement.)
- The Phase 4 commit lands both files in one go:
  `runbook/04-u2d2-hardware-integration.md` and the updated
  `vm/pincherx-100-dev.xml`.

## Why this shape

- **`managed='yes'` on the `<hostdev>`.** libvirt auto-detaches the
  device from the host's `ftdi_sio` driver on VM start and reattaches
  on VM stop. Without it, the host driver keeps the device claimed
  and the VM start fails with "device or resource busy." The
  alternative — `managed='no'` plus manual `virsh nodedev-detach`
  before each start — adds friction without buying anything. Per
  [libvirt USB host device docs](https://libvirt.org/formatdomain.html#usb-host-device).
- **Match by vendor:product, not bus:device.** Two equivalent forms
  exist. `0x0403:0x6014` is stable across USB port swaps, cable
  replugs, and reboots. Bus:device pinning breaks the moment the
  device is replugged (kernel reassigns device numbers). Per
  `feedback_defaults_first.md`, prefer the form that needs no
  bookkeeping when the user moves the cable.
- **Install udev rules on the host even though the VM owns the
  device at runtime.** Three reasons:
    1. `../CLAUDE.md` "Robot interface" requires it explicitly.
    2. It's a pre-attach sanity check — if `/dev/ttyDXL` doesn't
       appear on the host with the rules in place, the U2D2 is bad
       or the cable is bad, and pass-through can't fix that.
    3. It enables host-side serial smoke testing (`screen
       /dev/ttyDXL 1000000`, `python -m serial.tools.list_ports`)
       without booting the VM, which is useful when isolating
       hardware faults from VM faults.
- **Source the rules from the local `interbotix_ros_core` clone,
  not by extracting from the guest.** The file at
  `git/interbotix_ros_core/interbotix_ros_xseries/interbotix_xs_sdk/99-interbotix-udev.rules`
  is upstream — same content the install script copies into the
  guest. Skips an unnecessary host↔guest copy. The guest copy is
  unchanged and continues to work.
- **Phase 4 is host-and-guest, but the runbook orders host-side
  steps first.** The host udev install can happen with the VM shut
  off; the verification step needs the VM running. Doing host first
  lets us catch hardware issues without booting the VM at all.
- **No `<address>` element under `<hostdev>`.** libvirt auto-assigns
  the USB address inside the guest's xhci controller (index 0,
  added in Phase 2). Manual `<address>` blocks are needed only when
  pinning to a specific guest USB port for software that
  hard-codes it — Trossen's stack doesn't.
- **No `<vmctl>` USB hot-plug.** We add the `<hostdev>` to the
  *persistent* domain XML (`virsh define`), not via
  `virsh attach-device --live`. Hot-plug works but doesn't survive
  VM restart, which would force re-attaching every Phase 5 launch.
  Persistent is the right default for a daily-driver VM.

## Adapt

| Aspect | This runbook | If yours differs |
|---|---|---|
| Robot interface | U2D2 (FTDI FT232H, `0x0403:0x6014`) | Older OpenCM9.04C (`0xfff1:0xff48`): same udev rules file already has a `SYMLINK+="ttyDXL"` line for it; substitute the vendor/product IDs in the `<hostdev>` block. ROBOTIS U2D2 firmware revisions don't change the USB IDs. |
| Host distro | Ubuntu 24.04 Noble | Ubuntu 22.04 Jammy / 26.04 Resolute: identical commands. `udevadm` and `libvirt` semantics haven't shifted across these. |
| Host USB port | Any USB 3.0 port (all this laptop's ports are USB 3.0 per `project_host_usb_topology`) | USB 2.0-only host port: works (xHCI handles USB 2.0 devices on USB 3.0 controllers). U2D2 is a USB 2.0 device. |
| libvirt `<hostdev>` form | vendor:product (`0x0403`/`0x6014`), `managed='yes'` | bus:device (`<address bus='3' device='8'/>`): only when you must disambiguate two FT232H devices on the same host. Not your case — single arm. |
| Snapshot policy with U2D2 attached | Detach before live snapshot; disk-only snapshots are fine | Some setups never snapshot live — then no policy needed. We need at least disk-only snapshots for Phase 5 rollback safety. |
| Other USB devices passed through | None yet | RealSense D415 (Phase deferred): adds a second `<hostdev>` block with Intel's RealSense vendor:product (`0x8086:0x0ad3` for D415); same shape. |

## Sources

- Trossen ROS 2 Standard Software Setup (verification of `/dev/ttyDXL`):
  https://docs.trossenrobotics.com/interbotix_xsarms_docs/ros_interface/ros2/software_setup.html
- Interbotix udev rules file (upstream):
  https://github.com/Interbotix/interbotix_ros_core/blob/main/interbotix_ros_xseries/interbotix_xs_sdk/99-interbotix-udev.rules
- libvirt USB host device XML schema:
  https://libvirt.org/formatdomain.html#usb-host-device
- libvirt domain XML format (top-level reference):
  https://libvirt.org/formatdomain.html
- `udev(7)` (Ubuntu Noble manpage):
  https://manpages.ubuntu.com/manpages/noble/en/man7/udev.7.html

## Prerequisites

- [Phase 3](03-trossen-install.md) closed: ROS 2 Humble + Interbotix
  workspace installed inside the guest, `ros2 pkg list | grep
  '^interbotix' | wc -l` returned ≥ 27.
- The PincherX-100 arm is **powered on** and the U2D2 is connected
  via USB to the host. (Power on first; then plug USB. Hot-plugging
  USB after the arm is powered is fine but the reverse — powering
  the arm after USB is connected — can cause a momentary
  enumeration glitch on some hosts.)
- The locally cloned Trossen `interbotix_ros_core` repo exists at
  `/home/hugo-bluecorn/robots/pincherx_100/git/interbotix_ros_core/`.
  (If it doesn't, see `project_trossen_repos_local.md` or
  `git clone https://github.com/Interbotix/interbotix_ros_core.git`
  into that path.)
- The VM `pincherx-100-dev` is defined in libvirt and reachable via
  `virsh list --all`. Whether it's running or shut off doesn't
  matter for Step 1; Step 5 needs it shut off first.

---

## Step 1 — Pre-flight inventory on the host

**Why:** Same pattern as prior phases — confirm the host's view of
the U2D2 before changing anything. Catches "wrong device plugged
in," "ftdi_sio module not loaded," and "device already claimed by
something else" before they bite mid-runbook.

Run **on the host**:

```sh
$ lsusb | grep '0403:6014'
$ lsmod | grep ftdi_sio
$ ls /dev/ttyUSB* 2>/dev/null
$ ls /dev/ttyDXL 2>/dev/null || echo 'no ttyDXL yet (expected pre-rules)'
$ ls /etc/udev/rules.d/ | grep -i interbotix || echo 'no Interbotix udev rules yet (expected)'
$ virsh list --all
```

**Verify:**

- `lsusb` shows exactly one line with `ID 0403:6014 Future
  Technology Devices International, Ltd FT232H Single HS
  USB-UART/FIFO IC`. The bus/device numbers (e.g., `Bus 003
  Device 008`) will differ across replugs — only the vendor:product
  pair matters.
- `ftdi_sio` is loaded as a kernel module (modern Ubuntu kernels
  auto-load it on FT232H enumeration). If `lsmod` shows nothing,
  the device hasn't enumerated as a serial port yet — replug, then
  re-check.
- `/dev/ttyUSB*` shows at least one device (typically `ttyUSB0`).
  This is the raw FTDI character device the kernel created; the
  Trossen rules will add the `ttyDXL` symlink to it.
- `/dev/ttyDXL` does **not** exist yet (this step's whole point is
  to add it). If it already exists, you've installed the rules in
  a prior attempt — skip Step 2.
- `99-interbotix-udev.rules` is **not** in `/etc/udev/rules.d/` yet.
- `virsh list --all` shows `pincherx-100-dev` exists (state `shut
  off` or `running` is fine here).

**Watch out:**

- If `lsusb` shows the FT232H entry but no `/dev/ttyUSB*` appears,
  ModemManager may have grabbed the port and rejected it. Check
  `journalctl -u ModemManager.service --since '5 minutes ago' |
  grep -i ftdi`. The Trossen udev rule installed in Step 2 sets
  `ENV{ID_MM_DEVICE_IGNORE}="1"` to keep ModemManager off this
  device — but until the rule lands, MM may interfere. Worst case:
  `sudo systemctl stop ModemManager.service`, replug the U2D2,
  then `systemctl start` after Step 3 verifies `/dev/ttyDXL`.
- If `lsusb` shows two FT232H entries, you have another FTDI-based
  device plugged in (some debug probes, some Arduino clones). The
  Trossen udev rule will create `ttyDXL` symlinks for *both*; the
  rule matches by vendor:product, not serial number. Unplug the
  unrelated device for Phase 4 if you can; otherwise pin via
  `ATTRS{serial}==` in a custom udev rule (out of scope here).
- If `virsh list --all` shows the VM as `running`, leave it
  running — Step 5 will shut it down then. Don't shut it down now
  unnecessarily; Phase 3's reboot already happened.

---

## Step 2 — Install the Trossen udev rules on the host

**Why:** The udev rules file ships in `interbotix_ros_core`. Copying
it to `/etc/udev/rules.d/` and reloading udevadm makes the host
kernel apply the rules to the (already-enumerated) FT232H device,
which creates the `/dev/ttyDXL` symlink and sets the latency timer
to 1 ms (needed for fast Dynamixel polling — the FTDI default of
16 ms makes the bus unusably slow for real-time control). Same
content the Phase 3 install script wrote into the guest; sourcing
from the local clone avoids an unnecessary host↔guest copy.

Run **on the host**:

```sh
$ RULES=/home/$USER/robots/pincherx_100/git/interbotix_ros_core/interbotix_ros_xseries/interbotix_xs_sdk/99-interbotix-udev.rules
$ ls -l "$RULES"
$ cat "$RULES" | grep -E '0403|6014|ttyDXL'
$ sudo cp "$RULES" /etc/udev/rules.d/99-interbotix-udev.rules
$ ls -l /etc/udev/rules.d/99-interbotix-udev.rules
$ sudo udevadm control --reload-rules
$ sudo udevadm trigger
```

**Verify:**

- `ls -l "$RULES"` shows the source file (~1 KiB).
- The `grep` confirms the file contains a `SYMLINK+="ttyDXL"` rule
  matching FTDI vendor `0403` and product `6014`. (Should print one
  line, the U2D2 rule.)
- After `sudo cp`, `ls -l /etc/udev/rules.d/99-interbotix-udev.rules`
  shows the file at the destination, mode `-rw-r--r--`, owner
  `root:root`.
- `udevadm control --reload-rules` returns silently.
- `udevadm trigger` returns silently and re-triggers all device
  uevents — which means the host kernel re-applies all udev rules,
  including the just-installed Trossen rules, to currently
  enumerated devices.

**Watch out:**

- If the cp fails with "Permission denied," you didn't get sudo
  (e.g., cached creds expired). Re-run; `sudo -v` first if it keeps
  prompting.
- If `udevadm control --reload-rules` errors with "Failed to connect
  to udev control socket," systemd-udevd isn't running. That would
  be unusual on a desktop install. `sudo systemctl status
  systemd-udevd` to investigate.
- The Trossen rules also set `MODE:="0666"` for `ttyUSB*`, `ttyACM*`,
  `js*`, and `video*` — that's a *broad* permission change that
  affects all serial / joystick / video devices on the host. For a
  developer laptop this is acceptable (these devices are
  user-controlled), but for a multi-user host you may want to
  tighten by adding `GROUP:="dialout"` and removing the world
  permission. Out of scope for this project; called out for
  awareness.

---

## Step 3 — Verify the host now sees `/dev/ttyDXL`

**Why:** The udev rules created the symlink. Confirm it.

Run **on the host**:

```sh
$ ls -l /dev/ttyDXL
$ udevadm info /dev/ttyDXL | grep -E 'ID_VENDOR_ID|ID_MODEL_ID|ID_SERIAL|DEVNAME|DEVLINKS'
$ cat /sys/bus/usb-serial/devices/ttyUSB0/latency_timer 2>/dev/null
$ lsusb -t | grep -A2 -B2 'ftdi_sio'
```

**Verify:**

- `ls -l /dev/ttyDXL` returns a symlink like
  `/dev/ttyDXL -> ttyUSB0`. The exact `ttyUSB` index varies by
  enumeration order; the symlink target is what matters.
- `udevadm info` confirms `ID_VENDOR_ID=0403`, `ID_MODEL_ID=6014`,
  and `DEVLINKS` contains `/dev/ttyDXL`.
- `latency_timer` reads `1` (in ms). If it reads `16` (the FTDI
  default), the udev `ATTR{device/latency_timer}="1"` line in the
  rule didn't apply — replug the U2D2 to re-trigger the rules and
  re-check.
- `lsusb -t` shows the FT232H bound to the `ftdi_sio` driver,
  parented to an xHCI bus. If it's bound to `usbfs` or unbound,
  the kernel module isn't loading correctly — `sudo modprobe
  ftdi_sio` to force-load, then replug.

**Watch out:**

- If `/dev/ttyDXL` doesn't appear, replug the U2D2 USB cable. udev
  rules apply on uevent (device add/remove). `udevadm trigger`
  fires synthetic events but a physical replug is the cleanest
  way to confirm the rule path works for a real attach.
- If after replug the symlink still doesn't appear, check that the
  Trossen rules file is syntactically valid:
  `sudo udevadm test $(udevadm info -q path /dev/ttyUSB0) 2>&1 |
  grep -E 'INTERBOTIX|99-interbotix'`. If you see "could not parse
  rule," there's a typo in the file you copied — re-copy from the
  source.
- Once `/dev/ttyDXL` shows on the host, the U2D2 is health-checked.
  We don't read or write the serial port from the host — that
  would conflict with the upcoming pass-through. Step 5 hands the
  device to the guest.

---

## Step 4 — Add the `<hostdev>` block to the canonical VM XML

**Why:** This is the persistent libvirt configuration change. By
adding `<hostdev>` to `vm/pincherx-100-dev.xml` rather than
hot-plugging via `virsh attach-device --live`, the U2D2 binds to
the guest on every VM start until you remove the block — no
re-attach ritual at the start of every Phase 5 session.

**This step is already done in the repo.** The XML block landed in
the same commit as this runbook. Inspect the diff against the prior
canonical XML, then validate the XML before defining it:

Run **on the host**:

```sh
$ cd /home/$USER/robots/pincherx_100/git/pincherx-100-runbook
$ git diff HEAD -- vm/pincherx-100-dev.xml
$ xmllint --noout vm/pincherx-100-dev.xml && echo 'XML well-formed'
```

**Verify:**

The diff should add a `<hostdev>` block right after the
`<controller type='usb' .../>` line, with shape:

```xml
<hostdev mode='subsystem' type='usb' managed='yes'>
  <source>
    <vendor id='0x0403'/>
    <product id='0x6014'/>
  </source>
</hostdev>
```

— and a comment block above it documenting `managed='yes'`
semantics and the snapshot-detach constraint. No other XML changes.

`xmllint --noout` should print `XML well-formed` with no error
output. If it complains about `Double hyphen within comment`,
that's the XML 1.0 §2.5 rule: the literal string `--` cannot
appear inside `<!-- ... -->`. Rewrite any prose mentioning CLI
flags like `--live` or `--disk-only` to avoid the literal
double-dash (use phrases like "the live flag" instead).
`virsh define` will hit the same parse error if you skip this
sanity check.

**Watch out:**

- If you're porting this runbook to a different host (e.g., later
  to a Resolute host for the Lyrical fork) with different USB
  hardware, the vendor:product values stay the same — the U2D2 is
  the same physical board. Only the host kernel and libvirt
  versions differ; both honor the same XML schema.
- Don't add `<address type='usb' bus='0' port='1'/>` under the
  `<hostdev>` source — that pins the *guest-side* USB topology and
  is over-specification for our use. libvirt auto-assigns guest USB
  addresses on the xhci controller from Phase 2. On
  `virsh dumpxml` you *will* see libvirt's auto-assigned
  `<address type='usb' bus='0' port='N'/>` echoed back; that's
  libvirt's bookkeeping, not user-supplied config — leave it out
  of the canonical XML.
- libvirt 10.x on Noble also auto-injects a
  `<watchdog model='itco' action='reset'/>` device on q35 domains
  when missing. It's harmless and unrelated to USB pass-through;
  don't try to "fix" the diff by removing it after define.

---

## Step 5 — Apply the XML and start the guest

**Why:** `virsh define` reads the XML and persists it as the
domain's canonical configuration. Subsequent `virsh start` invocations
use that config. The canonical copy lives in the repo — that's the
source of truth.

There's a wrinkle the first time you update an *existing* domain
from the canonical XML: the canonical XML intentionally omits
`<uuid>` (see the comment block at the top of
`vm/pincherx-100-dev.xml` — "Intentionally omitted so libvirt
regenerates them per host"). This is the right design for
cross-host portability: the same XML defines fresh domains on
Noble and Resolute without UUID collisions. But it means
`virsh define` on an existing-by-name domain fails with
`error: operation failed: domain 'pincherx-100-dev' already
exists with uuid ...`, because libvirt sees a UUID-less XML and
treats the define as a *new* domain rather than an *update* to
the existing one.

The fix is `virsh undefine --keep-nvram` first, then `virsh
define`. `--keep-nvram` preserves the OVMF VARS file at
`/var/lib/libvirt/qemu/nvram/pincherx-100-dev_VARS.fd`, so the
guest's UEFI boot order is unchanged. The qcow2 disk path is in
the XML and is referenced again post-define, so the guest's
filesystem is untouched. A fresh UUID gets auto-assigned —
that's fine, UUIDs are local libvirt state, not part of the
canonical XML.

Run **on the host**:

```sh
$ cd /home/$USER/robots/pincherx_100/git/pincherx-100-runbook
$ virsh list --all
$ virsh shutdown pincherx-100-dev 2>/dev/null || true       # no-op if already shut off
$ # wait until 'shut off'
$ virsh list --all
$ virsh undefine pincherx-100-dev --keep-nvram
$ virsh list --all                                          # briefly empty
$ virsh define vm/pincherx-100-dev.xml
$ virsh dumpxml pincherx-100-dev | grep -B1 -A5 hostdev
$ virsh start pincherx-100-dev
$ virsh list --all
$ ls /dev/ttyDXL 2>/dev/null || echo 'host ttyDXL gone (expected: managed=yes detached it)'
```

**Verify:**

- The pre-undefine `virsh list --all` shows `pincherx-100-dev` in
  `shut off` state. (Define-while-running works but silently
  defers changes to the next start, which is confusing — easier to
  define cold.)
- `virsh undefine ... --keep-nvram` prints
  `Domain 'pincherx-100-dev' has been undefined`. Between undefine
  and define, `virsh list --all` shows an empty table — the
  domain is briefly gone from libvirt's persistent registry. The
  disk and NVRAM files are still on disk.
- `virsh define vm/pincherx-100-dev.xml` prints
  `Domain 'pincherx-100-dev' defined from vm/pincherx-100-dev.xml`.
- `virsh dumpxml ... | grep -B1 -A5 hostdev` shows the just-defined
  `<hostdev>` block — with `<address type='usb' bus='0'
  port='N'/>` auto-added by libvirt (expected, see Step 4
  "Watch out").
- `virsh start` returns `Domain 'pincherx-100-dev' started`
  without errors.
- Post-start `virsh list --all` shows the VM in `running` state.
- On the host, `ls /dev/ttyDXL 2>/dev/null` now returns **nothing**.
  This is correct: `managed='yes'` told libvirt to detach the
  device from the host, so the host's `/dev/ttyDXL` symlink
  disappears for the duration of VM runtime. It re-appears when
  the VM stops.

**Watch out:**

- If `virsh define` fails with `error: operation failed: domain
  'pincherx-100-dev' already exists with uuid ...`, you skipped
  the `virsh undefine --keep-nvram` step above. The canonical XML
  is UUID-less by design (see the "Why" preamble of this Step);
  every XML update needs undefine-then-define, not a plain
  re-define. This is the pattern for *all* future XML changes
  to the canonical domain — bookmark it for Phase 7 onward.
- If `virsh undefine` errors with `Refusing to undefine while
  domain managed save image exists`, you have a previous
  managed-save state (`virsh save`). Wake it (`virsh start
  pincherx-100-dev`), shut it down cleanly, then undefine. Per
  `../CLAUDE.md` "Key constraints," we don't use `virsh save`
  across host boundaries — but the file can still exist from a
  same-host save.
- If `virsh start` fails with `Failed to open file '/dev/bus/usb/...':
  Permission denied`, libvirt's AppArmor profile or cgroup config
  isn't allowing access to the device node. On Ubuntu Noble's
  libvirt 10.x this is rare but possible — `sudo aa-status | grep
  libvirt` to confirm the profile is loaded; `journalctl -u
  libvirtd.service --since '1 minute ago'` will show the actual
  permission error.
- If `virsh start` fails with `unsupported configuration: USB is
  disabled for this domain`, you somehow removed the
  `<controller type='usb' .../>` line in Phase 2 — re-add it from
  the canonical XML.
- If `virsh start` fails with `internal error: process exited
  while connecting to monitor: ... -device usb-host: 'usbredir' has
  exclusive use`, you also have a `<redirdev bus='usb'>` SPICE USB
  redirection block somewhere claiming the same device. The
  canonical XML in this repo doesn't include one; check
  `virsh dumpxml pincherx-100-dev | grep -E 'hostdev|redirdev'`
  for stray entries.
- If `virsh start` succeeds but the device disappears from the
  guest a few seconds later, you may be hitting a USB autosuspend
  race — uncommon for FT232H but possible. Set
  `/sys/bus/usb/devices/<addr>/power/control` to `on` on the host
  (or `usbcore.autosuspend=-1` kernel parameter) and replug. Out
  of scope unless you see this; called out preemptively.

---

## Step 6 — Verify the device inside the guest (exit criterion)

**Why:** This is the canonical Trossen verification from
[ROS 2 Standard Software Setup → Installation Checks](https://docs.trossenrobotics.com/interbotix_xsarms_docs/ros_interface/ros2/software_setup.html).
It proves that the device crossed the host→guest boundary, that
the guest's `ftdi_sio` module bound to it, and that the guest's
Trossen udev rules (installed in Phase 3) created `/dev/ttyDXL`.

Open a guest session (virt-manager, SPICE viewer, or `ssh` into
the guest). Run **inside the guest**:

```sh
$ lsusb | grep '0403:6014'
$ ls /dev | grep ttyDXL
$ ls -l /dev/ttyDXL
$ udevadm info /dev/ttyDXL | grep -E 'ID_VENDOR_ID|ID_MODEL_ID|DEVLINKS'
$ cat /sys/bus/usb-serial/devices/ttyUSB0/latency_timer 2>/dev/null
$ dmesg | grep -iE 'ftdi|usb 1' | tail -20
```

**Verify (exit criterion for Phase 4):**

- `lsusb` inside the guest shows the FT232H — same vendor:product
  pair as on the host, on the guest's xhci bus.
- `ls /dev | grep ttyDXL` prints `ttyDXL` — matches Trossen's
  verbatim verification command from their docs.
- `ls -l /dev/ttyDXL` shows the symlink to a `ttyUSB*` device.
- `udevadm info` confirms `ID_VENDOR_ID=0403`, `ID_MODEL_ID=6014`,
  `DEVLINKS` contains `/dev/ttyDXL` — same rule fired in the guest
  that fired on the host in Step 3.
- `latency_timer` is `1` in the guest too. (The udev rule sets it
  on whichever side the rule fires — and the guest's Trossen
  install put the rule there in Phase 3.)
- `dmesg` shows the FT232H enumeration messages from the guest
  kernel, e.g., `usb 1-1: new high-speed USB device number ...`
  followed by `ftdi_sio 1-1:1.0: FTDI USB Serial Device converter
  detected` and `usb 1-1: FTDI USB Serial Device converter now
  attached to ttyUSB0`.

**Watch out:**

- If `lsusb` inside the guest doesn't show the FT232H, the
  pass-through didn't reach the guest. Most common causes:
  the `<hostdev>` block isn't in the active config (re-run `virsh
  dumpxml pincherx-100-dev | grep -A4 hostdev`), or the U2D2 was
  unplugged from the host between `virsh define` and `virsh start`.
  Replug, `virsh shutdown && virsh start`.
- If `lsusb` shows the device but `/dev/ttyDXL` doesn't exist in
  the guest, the guest's udev rules didn't fire. Confirm the rules
  file is in the guest at
  `/etc/udev/rules.d/99-interbotix-udev.rules` (Phase 3 put it
  there). If missing, copy it from the guest's source workspace:
  `sudo cp ~/interbotix_ws/src/interbotix_ros_core/interbotix_ros_xseries/interbotix_xs_sdk/99-interbotix-udev.rules
  /etc/udev/rules.d/` then `sudo udevadm control --reload-rules &&
  sudo udevadm trigger`.
- If `/dev/ttyDXL` appears but `latency_timer` is `16`, the udev
  rule fired the SYMLINK action but the ATTR write was racy. This
  doesn't fail Phase 4 but will slow Phase 5. Workaround:
  `echo 1 | sudo tee /sys/bus/usb-serial/devices/ttyUSB0/latency_timer`
  and document this in your Phase 5 launch checklist.
- The arm's servos do *not* need to be commanded yet. Phase 4 only
  proves the serial transport exists. Phase 5's
  `interbotix_xsarm_control` launch is what actually talks to the
  Dynamixels and torques the joints.

---

## Done — Phase 4 exit criteria

Tick all of these before moving on to Phase 5:

- [ ] On the host (VM shut off), `ls -l /dev/ttyDXL` shows a
      symlink to `/dev/ttyUSB*`, and `udevadm info /dev/ttyDXL`
      reports `ID_VENDOR_ID=0403`, `ID_MODEL_ID=6014`.
- [ ] On the host, `/etc/udev/rules.d/99-interbotix-udev.rules`
      exists, owner `root:root`.
- [ ] `vm/pincherx-100-dev.xml` contains a `<hostdev mode='subsystem'
      type='usb' managed='yes'>` block matching `0x0403:0x6014`,
      and `virsh dumpxml pincherx-100-dev` shows the same block in
      the live domain config.
- [ ] With the VM running, `lsusb` inside the guest shows the
      FT232H.
- [ ] With the VM running, `ls /dev | grep ttyDXL` inside the
      guest returns `ttyDXL` (Trossen's canonical check).
- [ ] On the host while the VM is running, `ls /dev/ttyDXL`
      returns nothing (proves `managed='yes'` detach worked).

When all are green, commit and push the Phase 4 deliverable as a
single commit:

```sh
$ cd /home/$USER/robots/pincherx_100/git/pincherx-100-runbook
$ git status                              # runbook/04-u2d2-hardware-integration.md + vm/pincherx-100-dev.xml
$ git add runbook/04-u2d2-hardware-integration.md vm/pincherx-100-dev.xml
$ git commit -m "Add Phase 4 (U2D2 hardware integration) runbook"
$ git push origin main
```

## Next

[`05-functional-verification.md`](05-functional-verification.md) —
power the arm, launch `interbotix_xsarm_control` for the `px100`
model, run torque on/off services, run the bartender demo,
visualize the URDF in rviz to confirm virgl3D is rendering the
3D model correctly. This is where the end-to-end control path
(Flutter excluded) gets exercised for the first time.

**Before any live snapshot from this phase forward:** detach the
U2D2 via `virsh detach-device pincherx-100-dev <(cat <<EOF
<hostdev mode='subsystem' type='usb' managed='yes'>
  <source><vendor id='0x0403'/><product id='0x6014'/></source>
</hostdev>
EOF
)` (or virt-manager → Details → USB Host Device → Remove), take
the snapshot, then re-attach with `virsh attach-device`. Disk-only
snapshots (`virsh snapshot-create --disk-only`) are safe with the
U2D2 attached and may be used freely. Per `../CLAUDE.md` "Key
constraints."
