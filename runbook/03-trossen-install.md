# 03 — Trossen software install (ROS 2 Humble + Interbotix workspace)

Run Trossen's `xsarm_amd64_install.sh` script inside the guest to install
ROS 2 Humble and the Interbotix X-series ROS 2 workspace. The arm
hardware does *not* need to be connected — that comes in Phase 4. This
phase is entirely guest-side; the host is untouched.

## Goal

After this phase:

- ROS 2 Humble (the `ros-humble-desktop` apt metapackage and its
  transitive deps) is installed under `/opt/ros/humble/`.
- A colcon workspace at `~/interbotix_ws/` contains the three
  Interbotix repos (`interbotix_ros_core`, `interbotix_ros_manipulators`,
  `interbotix_ros_toolboxes`), built with `colcon build`.
- `ros2 pkg list | grep '^interbotix' | wc -l` returns **≥ 27**, per
  the verification target in `../CLAUDE.md`. The core packages
  (`interbotix_xs_sdk`, `interbotix_xs_msgs`, `interbotix_common_modules`,
  `interbotix_xs_modules`, `interbotix_xsarm_control`,
  `interbotix_xsarm_descriptions`, `interbotix_xsarm_moveit`,
  `interbotix_xsarm_ros_control`, `interbotix_xsarm_sim`) are among
  them.
- The udev rules file `/etc/udev/rules.d/99-interbotix-udev.rules` is
  installed — irrelevant until Phase 4 when the U2D2 is attached to
  the guest, but pre-staged here.
- `~/.bashrc` sources `/opt/ros/humble/setup.bash` and
  `~/interbotix_ws/install/setup.bash` so every new shell has the
  ROS env.
- A `trossen-installed` backup pair exists at
  `/var/lib/libvirt/images/pincherx-100.trossen-installed.qcow2` and
  `/var/lib/libvirt/qemu/nvram/pincherx-100-dev_VARS.trossen-installed.fd`,
  *distinct* from the `clean-install` pair from Phase 2 — so Phase 4's
  USB pass-through work has a rollback target that doesn't also
  unwind the ROS install.

## Why this shape

- **`-d humble` flag.** `../CLAUDE.md` pins ROS 2 Humble. The script
  supports `galactic` / `humble` / `rolling` for ROS 2; Galactic is
  EOL, Rolling lacks Trossen's testing matrix, Humble is the LTS we
  want. Pre-selecting the distro skips one interactive prompt.
- **Run interactively (no `-n`).** Trossen's docs show
  `./xsarm_amd64_install.sh -d <distro>` (no `-n`); we want the
  interactive prompts so we can decline the optional Perception and
  MATLAB-ROS sub-installs. Both are out of scope for the project right
  now (perception is deferred per `../CLAUDE.md`; MATLAB-ROS is not
  used).
- **Default workspace path (`~/interbotix_ws`).** Per
  `feedback_defaults_first.md` — no concrete reason to override.
  Trossen's docs, tutorials, and tooling assume this path.
- **Run as your regular user, not root.** The script needs sudo
  internally (apt, udevadm, etc.) and will prompt; it must NOT be
  invoked with `sudo` itself, because it sets `$HOME` /
  `$USER`-relative paths and ownerships. Running as root would land
  the workspace under `/root/` and the bashrc edits in the wrong
  shell config.
- **Reboot the guest after install.** The script's final message
  prints `NOTE: Remember to reboot the computer before using the
  robot!` — fresh udev rules need a `udevadm trigger` (the script
  does this) *plus* a clean shell session for the bashrc env to be
  active everywhere. A reboot covers both deterministically.
- **Phase backup with a distinct name.** Same `qemu-img convert` +
  NVRAM `cp` pattern as Phase 2 Step 7, but the filenames carry a
  `trossen-installed` suffix. Phase 4's USB pass-through work is
  risky; we want to revert to "Trossen installed, no U2D2 attached"
  without going all the way back to "blank Kubuntu."

## Adapt

| Aspect | This runbook | If yours differs |
|---|---|---|
| Guest distro | Kubuntu 22.04 LTS desktop, minimal install | Stock Ubuntu 22.04 desktop: identical. Ubuntu 22.04 *server*: identical script, but `ros-humble-desktop` will pull in more packages than headless setups strictly need; tolerate or use `-d` plus manual `apt install ros-humble-ros-base` and run only the post-ROS portion of the Trossen script. |
| Guest CPU arch | amd64 (x86_64) | Raspberry Pi 4B: substitute `xsarm_rpi4_install.sh` per [Trossen's Raspberry Pi 4B setup page](https://docs.trossenrobotics.com/interbotix_xsarms_docs/ros_interface/ros2/raspberry_pi_setup.html). Different script, different prerequisites. |
| ROS 2 distro | Humble (`-d humble`) | Galactic: EOL since Dec 2022, Trossen retains the branch but you won't get security updates. Rolling: nightly, breaks frequently — only useful if you know why. Humble is the only LTS-grade choice. |
| Workspace path | `~/interbotix_ws` (default) | Override with `-p /abs/path` only if storage layout demands it. The bashrc edits and Trossen examples assume the default. |
| Arm model | PincherX-100 (`px100`) | The script installs the workspace for *all* X-series arms (px100, rx150, wx250s, vx300s, etc.); model selection happens in Phase 5 at launch time (`robot_model:=px100`). No flag change here. |
| MATLAB API | Decline | Only check yes if you have a MATLAB workflow tied to ROS — not applicable to this project. |
| Perception (RealSense + AprilTag) | Decline at install time | Deferred per `../CLAUDE.md` until arm control is verified. Re-runnable later via the perception-specific repos. |

## Sources

- Trossen ROS 2 Standard Software Setup page:
  https://docs.trossenrobotics.com/interbotix_xsarms_docs/ros_interface/ros2/software_setup.html
- Trossen ROS 2 Quickstart Guide:
  https://docs.trossenrobotics.com/interbotix_xsarms_docs/ros_interface/ros2/quickstart.html
- `xsarm_amd64_install.sh` source (the script we run):
  https://github.com/Interbotix/interbotix_ros_manipulators/blob/main/interbotix_ros_xsarms/install/amd64/xsarm_amd64_install.sh
- ROS 2 Humble installation docs (the script's underlying dependency):
  https://docs.ros.org/en/humble/Installation.html
- Interbotix ROS 2 package index:
  https://docs.trossenrobotics.com/interbotix_xsarms_docs/ros2_packages.html

The install script lives on the `main` branch of
`Interbotix/interbotix_ros_manipulators` and is not version-pinned.
Trossen treats this as a rolling install path. If you need byte-for-byte
reproducibility across hosts, commit the SHA you ran or vendor a copy
of the script alongside this runbook — but for the project's stated
goals, "whatever main produces at install time" is acceptable.

## Prerequisites

- [Phase 2](02-guest-provisioning.md) closed: VM `pincherx-100-dev`
  hardware-verified, running on the canonical XML, clean-install
  backup files present at
  `/var/lib/libvirt/images/pincherx-100.clean-install.qcow2` +
  `/var/lib/libvirt/qemu/nvram/pincherx-100-dev_VARS.clean-install.fd`.
- Guest network is up. The install pulls ~3–5 GiB of apt + git
  content from `packages.ros.org`, `archive.ubuntu.com`, and GitHub.
- Roughly 8–10 GiB of host disk headroom under
  `/var/lib/libvirt/images/`: the live qcow2 will grow ~3–5 GiB
  during install, and the Phase 3 backup file is another ~12–15 GiB
  compacted.
- The arm hardware does *not* need to be connected for Phase 3.

---

## Step 1 — Pre-flight inventory inside the guest

**Why:** Same pattern as Phase 1's Step 2a and Phase 2's Step 1 —
confirm the guest is in the expected pre-install state before
fetching a large install script and running it with sudo. Catches
"ROS already installed (interrupted prior attempt)," "workspace dir
already exists," and "no internet" before they bite mid-install.

Run **inside the guest**:

```sh
$ uname -a                                 # confirm we're in the guest, not the host
$ cat /etc/issue                           # Kubuntu 22.04 LTS
$ lsb_release -d                           # Ubuntu 22.04.x
$ df -h ~                                  # ≥ 6 GiB free in $HOME's filesystem
$ ls -la ~ | grep interbotix               # expect no interbotix_ws yet
$ dpkg -l | grep -E '^ii\s+ros-' | head    # expect no ros-* packages yet
$ ls /opt/ros 2>/dev/null                  # expect "No such file or directory"
$ ping -c 3 packages.ros.org               # ROS apt repo reachable
$ ping -c 3 raw.githubusercontent.com      # install script source reachable
$ command -v curl || sudo apt install -y curl
```

**Verify:**

- Hostname is the one you picked in Phase 2 (e.g., `trossen`), not
  the host. (If you accidentally ran this on the host, stop —
  Trossen install on the host is out of scope.)
- `lsb_release -d` reports Ubuntu 22.04. (Kubuntu says "Ubuntu" via
  lsb_release; that's expected — see Phase 2 Step 5b.)
- `~` has ≥ 6 GiB free.
- No existing `~/interbotix_ws/` directory. If one exists from a
  prior attempt, decide: resume vs. clean restart. To clean restart,
  `rm -rf ~/interbotix_ws/` and remove the Trossen-related lines
  from `~/.bashrc` (search for `interbotix_ws` and `ros/humble` —
  remove only the lines the script added).
- No `ros-*` packages installed yet. If there are some, the install
  script is idempotent for apt (it'll print "already the newest
  version"), but the prompts will fire as if from scratch.
- Both `ping` checks succeed.

**Watch out:**

- The guest's host has an Android Studio emulator that also uses
  `/dev/kvm`. As long as Phase 2's VM is running, that's fine. If
  you also boot the AS emulator and the host swaps heavily, the ROS
  install will be slower but won't fail.
- If you previously ran the install script and aborted partway, the
  Trossen workspace may have partial git clones. `ls ~/interbotix_ws/src`
  shows what's there. Either complete the install (re-run the
  script — it's mostly idempotent) or clean restart (above).

---

## Step 2 — Download the install script and verify what it is

**Why:** Per `feedback_primary_sources_only.md`, we install from
upstream Trossen, not a mirror. Per general security hygiene, we
also eyeball the script before `chmod +x` — it's going to run with
your sudo password, install apt packages, edit `~/.bashrc`, and copy
udev rules. Reading the first 100 lines confirms it's the script we
expect, not a redirect/tampered copy.

Run **inside the guest**:

Single-line `curl` invocation deliberately — multi-line bash
continuations (`\` then newline) lose the newline through some
copy-paste paths and concatenate flags onto the URL, producing
spurious 404s. Keep it on one line:

```sh
$ cd ~
$ curl -fL -o xsarm_amd64_install.sh 'https://raw.githubusercontent.com/Interbotix/interbotix_ros_manipulators/main/interbotix_ros_xsarms/install/amd64/xsarm_amd64_install.sh'
$ ls -lh xsarm_amd64_install.sh
$ head -80 xsarm_amd64_install.sh
$ grep -nE '^[A-Z_]+_ROS_DISTRO_ARRAY=|^ROS_NAME=|^INSTALL_PATH=' xsarm_amd64_install.sh
```

**Verify:**

- `curl` completes without `404`, `403`, or TLS warnings.
- The file is ~30–60 KiB, plain bash text.
- `head -80` shows: shebang `#!/usr/bin/env bash`, a comment block
  documenting the script's purpose, and the `getopts` block parsing
  `-d`, `-p`, `-n`, `-h`. No obvious obfuscation (eval of base64
  blobs, etc.).
- The grep for `ROS_DISTRO_ARRAY` confirms `humble` is in the
  supported list.

**Watch out:**

- If `curl` fails with a TLS error, the guest's CA store may be
  stale — `sudo apt install -y ca-certificates && sudo
  update-ca-certificates` and retry.
- The `main` branch on Trossen's repo can change. If you need a
  pinned version (for byte-for-byte reproducibility), check the
  upstream tags at
  https://github.com/Interbotix/interbotix_ros_manipulators/tags
  and substitute `main` with a tag in the URL. The runbook doesn't
  pin here because Trossen treats `main` as the stable install
  channel; cite-the-sha in your own commit notes if you care.

Make it executable:

```sh
$ chmod +x xsarm_amd64_install.sh
```

---

## Step 3 — Run the install script with `-d humble`

**Why:** This is the actual install. It runs for ~15–30 minutes
depending on apt mirror speed and CPU. During the run it will:

1. `sudo apt update && sudo apt install` ROS 2 apt prerequisites
   (`curl`, `gnupg`, `software-properties-common`, etc.).
2. Add the ROS 2 apt repo + key to the guest.
3. `sudo apt install ros-humble-desktop` plus rosdep / colcon tooling.
4. `pip install modern_robotics transforms3d`.
5. `git clone` the three Interbotix repos into `~/interbotix_ws/src/`.
6. `rosdep install` for the workspace.
7. `colcon build` the workspace.
8. Copy `99-interbotix-udev.rules` to `/etc/udev/rules.d/` and
   reload udevadm.
9. Append the ROS + workspace sourcing lines to `~/.bashrc`.

Run it **as your regular user**, not under sudo. The script will
prompt for your sudo password when it needs apt etc.:

```sh
$ ./xsarm_amd64_install.sh -d humble
```

**Two interactive prompts you'll see** — answer both with `n`
(decline) per the Adapt table above:

> ```
> Install the Interbotix Perception packages? This will include the
> RealSense and AprilTag packages as dependencies.
> ```
>
> Answer: **n** — perception is deferred per `../CLAUDE.md`. Can be
> added later via the perception-specific repos without re-running
> this script.

> ```
> Install the MATLAB-ROS API?
> ```
>
> Answer: **n** — not used by this project.

The script also asks for a final confirmation summary before
proceeding with the install. Confirm `y` to proceed.

**Verify (during the run):**

- The early apt steps should print `Get:` lines for `packages.ros.org`
  — that means the ROS apt repo is properly added.
- The git-clone phase should print three clone progress bars in
  succession (for `interbotix_ros_core`, `interbotix_ros_manipulators`,
  `interbotix_ros_toolboxes`).
- The colcon-build phase is the longest single step (~5–15 min).
  It should end with `Summary: NN packages finished` and no
  `[err]` lines.

**Verify (at the end):**

The final two lines printed should be (verbatim from the script
source):

```
Installation complete, took N seconds in total.
NOTE: Remember to reboot the computer before using the robot!
```

**Watch out:**

- If `colcon build` errors with `error: 'modern_robotics' module not
  found` mid-build, the pip step earlier in the script ran into a
  Python 2 vs 3 split. Verify `python3 -c 'import modern_robotics'`
  works post-install; if not, `pip3 install modern_robotics`
  manually and re-run `colcon build` from `~/interbotix_ws/`.
- If `rosdep install` fails with "Cannot resolve" for a specific
  package, that package isn't in `rosdep`'s default views — usually
  benign for our subset. Re-run the script (idempotent on apt) or
  `rosdep update && rosdep install --from-paths src --ignore-src
  -r -y` from `~/interbotix_ws/`.
- AppArmor on the guest doesn't sandbox the install. udev rule
  installation works inside the VM because the guest kernel has its
  own udev. Phase 4 will deal with whether the U2D2 actually
  enumerates through libvirt's USB pass-through.
- If you accidentally answered `y` to the perception or MATLAB
  prompts, let the install finish (it won't fail, just installs
  extra packages); you can ignore the extra packages or
  `apt remove` them after. Don't kill the script mid-install —
  partial state is harder to clean up than over-install.

---

## Step 4 — Reboot the guest

**Why:** the script told us to, and there's a real reason: the
bashrc env edits are picked up by *new* login shells, the udev
rules need a fresh trigger, and any apt configuration changes
during the install reach a quiescent state on reboot. A reboot
is one deterministic action instead of three separate
"source ~/.bashrc / udevadm trigger / verify daemon state" steps.

**Inside the guest:**

```
> Application launcher → Leave → Restart
```

Or from the host:

```sh
$ virsh reboot pincherx-100-dev
```

Wait for the VM to come back to the login screen / desktop.

**Verify (inside the guest, after reboot):**

```sh
$ echo $ROS_DISTRO                         # humble
$ which ros2                               # /opt/ros/humble/bin/ros2
$ env | grep -E '^(ROS|AMENT)'             # several ROS_*/AMENT_* vars set
```

If `$ROS_DISTRO` is empty, the bashrc edits didn't fire — check
`~/.bashrc` ends with two `source` lines (one for `/opt/ros/humble`,
one for `~/interbotix_ws/install`), and open a fresh terminal.

---

## Step 5 — Verify the install

**Why:** Confirm the package count target from `../CLAUDE.md` is
hit, and that the key packages we'll need in Phase 5 are present
and importable.

Run **inside the guest**:

```sh
$ ros2 pkg list | grep '^interbotix' | wc -l
$ ros2 pkg list | grep '^interbotix' | sort
$ ros2 pkg prefix interbotix_xs_sdk
$ ros2 pkg prefix interbotix_xsarm_control
$ ros2 pkg prefix interbotix_xsarm_descriptions
$ ros2 pkg prefix interbotix_xsarm_moveit
$ ls /etc/udev/rules.d/99-interbotix-udev.rules
$ cat /etc/udev/rules.d/99-interbotix-udev.rules | grep -i 'FT232\|6014\|0403'
$ ros2 doctor --report 2>&1 | head -40
```

**Verify:**

- `wc -l` reports **≥ 27** (per `../CLAUDE.md`). The full list
  spans `interbotix_common_*`, `interbotix_xs_*`, `interbotix_xsarm_*`,
  and `interbotix_perception_*` (built but inert without the
  optional perception deps).
- All four `ros2 pkg prefix` queries print a path like
  `/home/<user>/interbotix_ws/install/<pkg>`. None error with
  "Package not found."
- The udev rules file exists and contains a rule referencing
  FTDI's FT232H (vendor 0x0403, product 0x6014) — that's what
  Phase 4 will rely on for the U2D2 to enumerate as `/dev/ttyDXL`.
- `ros2 doctor` reports no critical errors. Warnings about
  "QoS overrides" or "no network interface for ROS_DOMAIN_ID"
  are benign — we're not on a multi-machine ROS network.

**Watch out:**

- If the package count is below 27, the colcon build dropped some
  packages. Inspect `~/interbotix_ws/log/latest_build/` for the
  per-package log of any package that failed.
- If `ros2 pkg prefix interbotix_xsarm_control` errors with
  "Package not found" but it's in the source tree
  (`ls ~/interbotix_ws/src/interbotix_ros_manipulators/`), the
  workspace's `install/setup.bash` isn't being sourced. Either
  the bashrc edits got dropped or the source command in the
  bashrc is pointing at the wrong path. Re-source manually:
  `source ~/interbotix_ws/install/setup.bash`.
- `ros2 doctor` may complain about "package 'rmw_implementation'"
  on a fresh install. As long as `RMW_IMPLEMENTATION` resolves
  via env (`echo $RMW_IMPLEMENTATION` or empty for default
  `rmw_fastrtps_cpp`), it's fine.

---

## Step 6 — Back up the trossen-installed state (optional, recommended before Phase 4)

**Why:** Phase 4 attaches the U2D2 via libvirt `<hostdev>` pass-through
and may surface guest-side udev / kernel-module issues that require
iteration. A backup at *this* point — Trossen software installed but
no hardware-touching changes yet — gives Phase 4 a rollback target
that doesn't unwind the ROS install. The Phase 2 `clean-install`
backup is preserved for the more disruptive "throw out everything
and restart" scenario.

**Optional?** Yes — strictly speaking, you can skip this and rely
on the Phase 2 `clean-install.qcow2` backup as a fallback. If Phase
4 goes badly, reverting to `clean-install` + re-running Step 3
(`./xsarm_amd64_install.sh -d humble`) recovers in ~7–8 minutes (the
script is mostly idempotent on apt). The Phase 3 backup just saves
that 7–8 min of rework and preserves any interactive answers you
gave to the Trossen script. Recommended before Phase 4, but not a
Phase 3 hard exit criterion. The Phase 3 commit can land before
this step is done.

Same pattern as [Phase 2 Step 7](02-guest-provisioning.md#step-7--back-up-the-clean-install-qcow2--nvram),
different filenames. Shut the VM down cleanly first — backups
taken on a running guest capture in-flight page-cache state:

```sh
$ virsh shutdown pincherx-100-dev
$ virsh list --all                          # wait until state is 'shut off'
```

Then back up both files **from the host**:

```sh
$ sudo qemu-img convert -O qcow2 \
       /var/lib/libvirt/images/pincherx-100.qcow2 \
       /var/lib/libvirt/images/pincherx-100.trossen-installed.qcow2
$ sudo cp \
       /var/lib/libvirt/qemu/nvram/pincherx-100-dev_VARS.fd \
       /var/lib/libvirt/qemu/nvram/pincherx-100-dev_VARS.trossen-installed.fd
```

**Verify** (mind the mode-0711 sudo+glob gotcha — see Phase 2 Step 7):

```sh
$ sudo ls -lh /var/lib/libvirt/images/
$ sudo ls -lh /var/lib/libvirt/qemu/nvram/
```

You should now see, under `/var/lib/libvirt/images/`:

- `pincherx-100.qcow2` (live, larger after the ROS install — expect ~25–30 GiB)
- `pincherx-100.clean-install.qcow2` (Phase 2 backup, ~9–10 GiB)
- `pincherx-100.trossen-installed.qcow2` (Phase 3 backup, ~12–15 GiB compacted)

And under `/var/lib/libvirt/qemu/nvram/`:

- `pincherx-100-dev_VARS.fd` (live)
- `pincherx-100-dev_VARS.clean-install.fd` (Phase 2 backup)
- `pincherx-100-dev_VARS.trossen-installed.fd` (Phase 3 backup)

Restart the VM:

```sh
$ virsh start pincherx-100-dev
```

**Reverting later** (don't run now; reference for Phase 4+):

```sh
$ virsh shutdown pincherx-100-dev
$ sudo cp /var/lib/libvirt/images/pincherx-100.trossen-installed.qcow2 \
          /var/lib/libvirt/images/pincherx-100.qcow2
$ sudo cp /var/lib/libvirt/qemu/nvram/pincherx-100-dev_VARS.trossen-installed.fd \
          /var/lib/libvirt/qemu/nvram/pincherx-100-dev_VARS.fd
$ virsh start pincherx-100-dev
```

(Cold revert, same shape as Phase 2's revert path.)

**Watch out:**

- Don't take the backup while the VM is running and a `colcon build`
  or apt operation is in progress. The page cache hasn't flushed and
  the qcow2 captures inconsistent state. Always `virsh shutdown` and
  wait for `shut off` first.
- The `trossen-installed.qcow2` backup is the rollback target Phase 4
  will use if/when USB pass-through goes sideways. Don't accidentally
  delete it during disk-pressure cleanups — it's the one file in
  `/var/lib/libvirt/images/` you'd want to preserve under all
  circumstances at this point.

---

## Done — Phase 3 exit criteria

Tick all of these before moving on to [Phase 4](04-u2d2-hardware-integration.md):

- [ ] Inside the guest, `echo $ROS_DISTRO` returns `humble`.
- [ ] Inside the guest, `which ros2` returns `/opt/ros/humble/bin/ros2`.
- [ ] Inside the guest, `ros2 pkg list | grep '^interbotix' | wc -l`
      returns **≥ 27**.
- [ ] Inside the guest, `ros2 pkg prefix interbotix_xsarm_control`
      returns a path under `~/interbotix_ws/install/`.
- [ ] Inside the guest, `ls /etc/udev/rules.d/99-interbotix-udev.rules`
      lists the file (size ~1–2 KiB).

**Recommended (not strict exit criteria) — see Step 6:**

- [ ] On the host, `sudo ls -lh /var/lib/libvirt/images/` shows the
      `trossen-installed.qcow2` backup alongside the live and
      `clean-install.qcow2`.
- [ ] On the host, `sudo ls -lh /var/lib/libvirt/qemu/nvram/` shows
      the `_VARS.trossen-installed.fd` backup.

When all are green, commit and push the Phase 3 runbook:

```sh
$ cd /home/$USER/robots/pincherx_100/git/pincherx-100-runbook
$ git status                              # runbook/03-trossen-install.md
$ git add runbook/03-trossen-install.md
$ git commit -m "Add Phase 3 (Trossen software install) runbook"
$ git push origin main
```

## Next

[`04-u2d2-hardware-integration.md`](04-u2d2-hardware-integration.md) —
attach the U2D2 (FTDI FT232H, vendor `0x0403`, product `0x6014`) to
the guest via libvirt `<hostdev>` USB pass-through. Apply the host-side
udev rules so `/dev/ttyDXL` enumerates correctly *before* pass-through.
Confirm the device shows up as `/dev/ttyDXL` inside the guest.
