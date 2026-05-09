# vm/

Canonical libvirt domain XML for the project VM lives here, per the
"Domain XML is canonical" rule in `../CLAUDE.md`.

The first XML — `pincherx100-guest.xml` or similar — is committed
when Phase 2 (`../runbook/02-guest-provisioning.md`) is verified
working. Each meaningful change to the VM definition (added
controller, added passthrough device, memory bump, etc.) is a new
commit so the history shows how the definition evolved.

To recreate the VM on a fresh host:

```sh
$ virsh define vm/pincherx100-guest.xml
$ virsh start pincherx100-guest
```

Empty for now.
