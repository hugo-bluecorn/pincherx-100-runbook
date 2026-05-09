# pincherx-100-runbook

A reproducible, step-by-step runbook for standing up a development
environment to operate a single Trossen Interbotix **PincherX-100**
robotic arm from a laptop, using **ROS 2 Humble** inside an Ubuntu
22.04 KVM/libvirt guest, with the U2D2 USB-serial adapter passed
through to the guest from a USB 3.0 host port.

This repository is the artifact. Follow it on a similar machine and
you should get to the same working setup. Adapt where your hardware,
distro, or ROS 2 version differs (see [adaptation
notes](runbook/00-overview.md#adapting-this-runbook)).

You can also hand the entire directory to an AI coding assistant
(another Claude session, etc.) with: *"Perform the steps in
`runbook/`, adapting for the hardware/distro/ROS-2-version differences
listed in `00-overview.md`."*

## Project goals + architecture

See [`CLAUDE.md`](CLAUDE.md) for the full project rationale,
architecture decisions, phases, and constraints. That file is the
canonical source for *why* we do things this way; the runbook is the
canonical source for *how*.

## Repository layout

```
pincherx-100-runbook/
├── README.md            you are here
├── CLAUDE.md            project goals, architecture, constraints
├── runbook/             phase-by-phase setup instructions
│   ├── 00-overview.md   start here — phase index + adaptation guide
│   └── NN-*.md          one file per implementation phase, written as we go
├── reference/           explanatory notes that aren't sequential steps
└── vm/                  canonical libvirt domain XML (committed when produced)
```

## Status

Active build-out. Each phase is added to `runbook/` as it is
completed and verified. Phases not yet committed are not yet
verified.

## Conventions used in the runbook

- **Commands** are shown in fenced code blocks, copy-pasteable. Lines
  prefixed with `$` are run as your normal user; lines prefixed with
  `#` are run as root (or with `sudo`).
- **Why:** boxes explain the reason for each command, not just what
  it does.
- **Verify:** boxes give the output you should see after each
  meaningful step, so you know you can move on.
- **Adapt:** boxes flag values you may need to change for your
  hardware (USB vendor/product IDs, bridge name, etc.).
- **Watch out:** boxes call out known failure modes and how to
  recover.
