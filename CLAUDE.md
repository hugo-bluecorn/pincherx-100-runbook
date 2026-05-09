# PincherX-100 ROS 2 Humble Setup

## Project goal

Stand up a development environment to operate a single Trossen Interbotix
PincherX-100 robotic arm from a laptop, using ROS 2 Humble. The setup must
support a downstream Flutter + Zenoh client (Bluecorn) consuming the arm's
ROS topics over the network without disrupting the Trossen-supported control
path.

## Architecture (decided)

- **Host**: dual-boot Ubuntu 24.04 LTS and Ubuntu 26.04 LTS on the same drive,
  sharing a common partition for VM storage so the same qcow2 boots from
  either host without copying.
- **Guest**: Ubuntu 22.04 LTS in QEMU/KVM, managed via libvirt and
  virt-manager. ROS 2 Humble's required Tier-1 platform.
- **Robot**: Trossen Interbotix PincherX-100, single arm.
- **Robot interface**: U2D2 USB-serial adapter (FTDI FT232H, vendor 0x0403,
  product 0x6014) passed through to the guest via libvirt `<hostdev>`. Host
  also gets the udev rules so `/dev/ttyDXL` enumerates correctly before
  pass-through.
- **Camera**: Intel RealSense D415 via USB 3.0 pass-through. Deferred until
  the arm control path is verified end-to-end.
- **Network**: libvirt default NAT network (`virbr0`). All DDS
  participants — the ROS 2 nodes and `zenoh-bridge-ros2dds` — live
  inside the guest, so DDS multicast discovery is bounded to the
  guest's loopback and never has to traverse the host boundary. The
  only data flow that exits the guest is Zenoh (TCP/QUIC unicast),
  which the Flutter client reaches via a host→guest port-forward
  configured when the Flutter integration is wired up. This removes
  the need for a host bridge and is compatible with a Wi-Fi-only host.
  RMW: default FastDDS unless it proves unreliable in the
  virtualized network path, in which case fall back to CycloneDDS
  (`ros-humble-rmw-cyclonedds-cpp`, `RMW_IMPLEMENTATION=rmw_cyclonedds_cpp`).
- **External middleware**: zenoh-bridge-ros2dds standalone binary inside the
  guest, exposing ROS topics/services/actions over Zenoh keyexpressions to
  the Flutter client. Match Zenoh versions across bridge, zenohd (if used),
  and zenoh-dart on the Flutter side.
- **Install method**: Trossen's `xsarm_amd64_install.sh -d humble`,
  unmodified. The `-n` flag is not needed since the install runs interactively
  in the guest.

## Implementation phases (in order)

1. **Host preparation** — install qemu-kvm, libvirt-daemon-system,
   libvirt-clients, virt-manager. Add user to libvirt and kvm groups.
   Verify KVM with kvm-ok. Verify libvirt's default NAT network
   (`virbr0`) is autostarted; do not create a host bridge.
2. **Guest provisioning** — create Ubuntu 22.04 LTS guest with 4 vCPU, 8 GB
   RAM, 60 GB virtio disk, virtio NIC on libvirt's default NAT network.
   Install from ISO. Take a libvirt snapshot of the clean install.
3. **Trossen software install** — run `xsarm_amd64_install.sh -d humble`
   inside the guest. Reboot the guest. Verify the 27 expected interbotix_*
   packages appear in `ros2 pkg list`.
4. **Hardware integration** — extract the udev rules from the guest install
   and apply them on the host so the U2D2 enumerates as `/dev/ttyDXL` on the
   host. Attach the U2D2 to the guest via virt-manager USB Host Device or
   `<hostdev>` XML. Confirm `ls /dev | grep ttyDXL` inside the guest.
5. **Functional verification** — power on the arm. Launch
   `interbotix_xsarm_control` for the px100 model. Run torque on/off services
   and the bartender demo. Confirm joint state telemetry, motion commands,
   clean shutdown.
6. **Golden image** — `virt-clone` the working VM. Preserve the original
   untouched. Future experimentation occurs on clones.
7. **Zenoh bridge integration** — install zenoh-bridge-ros2dds standalone
   binary inside the guest. Verify the Flutter app subscribes to arm topics
   over Zenoh keyexpressions.

## Out of scope

- ROS distribution upgrades beyond Humble (Lyrical port evaluated separately,
  not pursued)
- Docker for the runtime control path (evaluated, rejected; acceptable later
  for CI builds of the workspace)
- Multi-arm or ALOHA-style configurations
- PREEMPT_RT / Ubuntu Pro real-time kernel (deferrable; not needed at
  PincherX-100 control rates)
- Gazebo simulation parity
- Perception integration before arm control is verified

## Key constraints

- **Trossen does not officially support VM-based installs.** We proceed
  knowing this. Hardware-side issues (motor calibration, U2D2 firmware)
  escalate to ROBOTIS, not Trossen, and are unaffected by the VM boundary.
- **Snapshots and USB pass-through interact badly.** Detach the U2D2 before
  taking live snapshots; reattach after. Disk-only snapshots when the arm is
  connected are safe.
- **Cross-boot shared qcow2** requires shutting the VM down cleanly before
  rebooting between host OSes. Never `virsh save` or suspend across host
  boundaries — saved state files don't transfer.
- **Both host OSes must define the VM** in their respective libvirt
  instances, but only one runs it at a time. Domain XML lives in version
  control; each host runs `virsh define` on the canonical XML.

## Working conventions

- **RTFM before instructing.** Fetch and read actual documentation, READMEs,
  or source before recommending commands. Do not guess from shallow web
  search snippets.
- **Be precise, not lazy.** Verify line numbers, file paths, command flags,
  package versions, and URL schemes. Use exact matching.
- **Lead with the primary action.** Optional alternatives, tangents, and
  edge cases are separated under an "Extras" subheading or equivalent, not
  mixed into the main response.
- **Domain XML is canonical.** All libvirt VM definitions live in version
  control as XML. Recreate the VM on any host with `virsh define`.
- **Snapshot before destructive operations.** Internal snapshots
  (`virsh snapshot-create-as`) live inside the qcow2 and survive cross-host
  boot; use them liberally before risky changes.

## References

- Trossen X-series docs (ROS 2):
  https://docs.trossenrobotics.com/interbotix_xsarms_docs/
- Install script:
  https://raw.githubusercontent.com/Interbotix/interbotix_ros_manipulators/main/interbotix_ros_xsarms/install/amd64/xsarm_amd64_install.sh
- Perception package (deferred phase):
  https://docs.trossenrobotics.com/interbotix_xsarms_docs/ros2_packages/perception_pipeline_configuration.html
- Eclipse Zenoh ROS 2 DDS bridge:
  https://github.com/eclipse-zenoh/zenoh-plugin-ros2dds
- ROS 2 Humble installation:
  https://docs.ros.org/en/humble/Installation.html
