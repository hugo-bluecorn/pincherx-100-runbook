# 00 — Overview, prerequisites, and adapting the runbook

This is the entry point. Read it once, then work through the numbered
phase files in order.

## What you end up with

A laptop running Ubuntu (24.04 or 26.04 LTS), booted into either of
those host OSes, that can launch an Ubuntu 22.04 KVM/libvirt guest;
inside the guest, ROS 2 Humble drives a Trossen Interbotix
PincherX-100 arm via the Trossen `xsarm_amd64` install. The U2D2
USB-serial adapter is passed through to the guest from one of the
host's USB 3.0 ports. Joint state telemetry, motion commands, and
the bartender demo work end-to-end.

A standalone `zenoh-bridge-ros2dds` inside the guest exposes ROS 2
topics over Zenoh keyexpressions, consumed by a Flutter client on
the same network.

## Phases

| #  | File                                | What it does                                                                |
|----|-------------------------------------|-----------------------------------------------------------------------------|
| 01 | `01-host-preparation.md`            | Install qemu-kvm + libvirt + bridge-utils + virt-manager; verify KVM; create bridged libvirt network |
| 02 | `02-guest-provisioning.md`          | Create Ubuntu 22.04 LTS guest with virtio disk, virtio NIC on bridge, qemu-xhci controller |
| 03 | `03-trossen-software-install.md`    | Run `xsarm_amd64_install.sh -d humble` inside the guest; verify packages    |
| 04 | `04-hardware-integration.md`        | Host udev rules for U2D2; pass U2D2 through to the guest                    |
| 05 | `05-functional-verification.md`     | Power on the arm; run torque + bartender demos; confirm telemetry           |
| 06 | `06-golden-image.md`                | `virt-clone` the working VM; preserve the original                          |
| 07 | `07-zenoh-bridge-integration.md`    | Install zenoh-bridge-ros2dds; verify Flutter client subscribes              |

A phase is committed only after it has been verified working on the
hardware. If a phase file isn't here yet, that phase hasn't been
done yet.

## Prerequisites

Hardware:

- A laptop with hardware virtualization (VT-x/AMD-V) enabled in
  firmware.
- At least 8 GB RAM and ~80 GB free disk on a partition reachable
  from your host OS install. (4 GB and 60 GB if you're tight; the
  guest is configured for 8 GB / 60 GB by default.)
- One free USB 3.0 port for the U2D2.
- A second free USB 3.0 port later for an Intel RealSense D415
  (deferred to a later phase, not blocking).
- A wired or wireless network on which the guest can be bridged to
  reach a Flutter client. Multicast must work end-to-end (see
  Phase 1).

Software:

- A fresh-ish Ubuntu install on the host. This runbook is written
  with **Ubuntu 24.04 LTS** as the primary host. Ubuntu 26.04 LTS
  is the stretch goal (dual-boot setup; see "Cross-host shared
  qcow2" in `reference/`); deltas from 24.04 are noted inline.
- `git` and a basic CLI familiarity.

Robot:

- A Trossen Interbotix PincherX-100 arm.
- The Trossen U2D2 USB-serial adapter (FTDI FT232H, vendor `0x0403`,
  product `0x6014`).

## Conventions

- Commands prefixed with `$` are run as your normal user.
- Commands prefixed with `#` are run as root (or with `sudo`).
- **Why:** explains the reason behind a step, so you know whether
  it applies to your setup.
- **Verify:** tells you what output indicates success.
- **Adapt:** flags values you'll likely change for different
  hardware. Substitute and continue.
- **Watch out:** known failure modes and how to recover.

## Adapting this runbook

This runbook is written for one specific setup, but the structure
is intentional so others can adapt it. The substitution points:

| Aspect                          | This setup                                | If yours differs…                                                                                  |
|---------------------------------|-------------------------------------------|----------------------------------------------------------------------------------------------------|
| Host OS                         | Ubuntu 24.04 LTS (with 26.04 as stretch)  | A different distro: package names diverge in Phase 1; libvirt + qemu work the same.                |
| Guest OS                        | Ubuntu 22.04 LTS                           | ROS 2 Humble's only Tier-1 platform is 22.04. Don't change unless you also change ROS 2 distro.    |
| ROS 2 distro                    | Humble                                     | Iron / Jazzy / etc.: change `-d humble` in Phase 3 and pick the right Tier-1 Ubuntu in Phase 2.    |
| Robot model                     | Trossen PincherX-100                       | Other Interbotix X-series: change the launch model arg in Phase 5 (e.g., `wx250s`, `vx300s`).      |
| USB-serial adapter              | U2D2 (FTDI FT232H, `0403:6014`)           | Different VID/PID: substitute everywhere it appears in Phases 4 + libvirt domain XML.              |
| Camera (deferred)               | Intel RealSense D415 (USB 3.0)             | Different camera: Phase 7+ needs revisiting. xHCI controller is still the right host-side choice.  |
| Host bridge name                | `br0`                                      | Anywhere `br0` appears, sub your bridge name. The libvirt domain XML in `vm/` references it too.   |
| Hypervisor                      | QEMU/KVM via libvirt                       | Other hypervisors are out of scope for this runbook.                                               |

If you fork this repo to capture a different setup, edit this
table and the affected phase files, and keep `CLAUDE.md` honest
about the new architecture.

## How to use this with another AI assistant

Hand it the repo and ask:

> Read `runbook/00-overview.md` and `CLAUDE.md`. Then work through
> `runbook/` in order. My setup differs from this runbook in the
> following ways: \[list\]. Adapt each step accordingly and explain
> the changes you make as you go. Pause and ask before any command
> that touches my system; I'll run them by hand.

The runbook is written so that the "why" of each step is explicit,
which is what makes adaptation safe.
