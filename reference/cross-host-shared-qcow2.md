# Cross-host shared qcow2 — the basic concern

Plain-language note for the dual-boot (Ubuntu 24.04 ↔ Ubuntu 26.04)
shared-qcow2 setup described in `CLAUDE.md`. Written 2026-05-09.

## The basic concern in plain terms

Imagine the VM's disk image (`vm.qcow2`) as a single notebook on a
shared bookshelf. Two roommates (Ubuntu 24.04 and Ubuntu 26.04) both
have library cards for it, but only one is ever home (booted) at a
time.

While a roommate is writing in the notebook, they keep some pages
stacked on their desk in RAM, planning to slot them back in later.
The notebook has a master index up front that says where every page
lives. The index gets updated *only after* all the desk pages are
slotted back in. That moment — "everything flushed back, index
consistent" — only happens at a clean VM shutdown.

There's a "do not disturb" sticky note (the file lock) that says
"I'm writing in this!" — but the sticky note is only enforced while
a roommate is actually home. Reboot, and there's no enforcement at
all from the previous session.

## The dangerous scenario (the realistic one)

1. **Monday, booted in Ubuntu 24.04.** You start the VM, do some ROS 2
   work. The VM has dirty pages on the desk (in QEMU's RAM cache).
   The notebook's index is currently slightly out of date.
2. **You get distracted, close the laptop lid.** The host suspends.
   The VM is "paused" but not shut down. Pages are still on the desk,
   not in the notebook.
3. **Tuesday, you boot Ubuntu 26.04 instead of resuming.** The 24.04
   session never came back to flush its desk.
4. **You start the VM in 26.04.** QEMU opens `vm.qcow2`. The qcow2
   *does* have a dirty bit it sets while open and clears on clean
   close, so QEMU sees "this was not closed cleanly." On a good day,
   QEMU refuses or runs a refcount rebuild. On a bad day, the
   inconsistency is in a place the rebuild doesn't catch — pages on
   the desk that were already promised to a snapshot, or refcount
   drift on cluster numbers — and now writes happen on top of
   inconsistent metadata. Result: filesystem corruption inside the
   guest, or worse, the qcow2 itself becomes unopenable.

The mistake wasn't the dual-boot. The mistake was step 2: closing
the lid instead of `virsh shutdown` first.

## The safe scenario

1. **Monday, Ubuntu 24.04.** Work in the VM.
2. **`virsh shutdown ubuntu22-guest`** — guest OS shuts down
   gracefully, QEMU flushes the desk back into the notebook, updates
   the index, releases the lock, exits. Dirty bit cleared.
3. **`sudo reboot` into Ubuntu 26.04.**
4. **`virsh start ubuntu22-guest`.** QEMU sees a clean qcow2, opens
   it, places its own (new, irrelevant) lock, off you go.

## Why `virsh save` is a separate trap

`virsh save` writes a *second* file: a snapshot of the VM's RAM,
CPU registers, and emulated device state — but **not** the disk.
It's a live freeze.

That second file embeds host-specific stuff: which IRQ vector the
xHCI controller landed on, which memory regions the kernel mapped
where, which exact QEMU build produced the state. Restore it on a
*different* host OS and the guest finds itself with devices wired
to nothing, pages mapped where the new QEMU didn't expect, and
either silently breaks (devices stop responding) or panics. So
`virsh save` is fine for "shut down quickly, resume later on the
same host," but a trap for our dual-boot model. Use `virsh
shutdown` instead — slower, but portable.

## The simple rule

Before rebooting between Ubuntu 24.04 and Ubuntu 26.04: **the VM
must be off, not paused, not suspended, not saved**. `virsh list`
should report no running domains. Then it doesn't matter which host
opens the qcow2 next — it's a clean notebook on a shelf.
