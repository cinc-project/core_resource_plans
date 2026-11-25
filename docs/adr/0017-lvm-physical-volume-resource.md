# 0017. Built-in `lvm_physical_volume` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

LVM2 stores volume metadata on physical volumes (PVs). Cinc Client ships
`mdadm` for Linux MD arrays and `mount` for filesystems
(`lib/chef/resource/mdadm.rb:26` and `lib/chef/resource/mount.rb:27`).
Neither manages an LVM label or its physical volume identity.

The shared Chef source has no built-in `lvm_physical_volume` resource.

On Ubuntu 24.04 with LVM2 2.03.16, `pvs --reportformat json` accepted the
proposed output fields and returned an empty PV report in a device-free
container. Initialisation remains a guest test.

## Decision

Add `lvm_physical_volume` to Cinc Client in the shared Chef codebase. Initialise one explicitly selected, unused block device as a physical volume. The design provides no action to erase or repurpose a PV.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Initialise an otherwise unused block device, or keep its existing PV identity. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `device` | String | Required | Stable absolute path under `/dev/disk/by-id` or `/dev/disk/by-partuuid`. Must resolve to one accessible block device. | The device Chef may initialise. |

### Current state and action steps

1. Resolve the device and identify it by major and minor number. Read
   `pvs --reportformat json` and compare the reported device identity. An
   existing unambiguous PV is a no-op, including one already in a group.
2. For a new PV, require a writable device. Check signatures without
   clearing them. Check mounts, swap, child partitions, holders and
   relevant LVM filters. Recheck the identity immediately before the
   write. Fail if the scan is incomplete or anything owns the device.
3. Run `pvcreate` without force or automatic confirmation and with closed
   stdin. Re-read the report to confirm exactly one PV for the device and
   record its UUID. Use a local resource lock during the preflight and
   mutation. LVM owns its own metadata locking.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Existing data signature or device in use | Non-destructive signatures, usage or holder checks report activity. | Fail before writing. |
| Ambiguous identity | Missing path, changed device identity, duplicate PV or filtered device. | Fail and identify the device and diagnostic. |
| Initialisation fails | Nonzero status or missing postcondition. | Report the result and leave the next run to discover actual state. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 24.04, Debian 12 and RHEL 9 | Proposed for local LVM2 devices with tested discovery and signatures tooling. |
| Multipath, shared LVM and network block devices | Deferred. |

### Target mode

Defer support until remote device scans and mutations can be verified on the same host.

## Usage examples

### :create

```ruby
lvm_physical_volume 'worker-pv' do
  device '/dev/disk/by-id/wwn-0x5000000000000017'
  action :create
end
```

## End-to-end testing

Use disposable guest disks. Verify the first converge, stable PV UUID on
repeat, alias resolution and refusal of formatted, mounted and partitioned
whole devices. Include an LVM devices-file exclusion and a read-only disk.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for report fixtures, without real device mutation. |
| kitchen-exec | Possible on an isolated VM with an explicitly attached test disk. |
| Vagrant | Preferred for real guest disks and destructive-case boundaries. |

Run report parsing in Dokken. Gate mutations and refusal cases on a guest with disposable disks.

## Consequences

### Positive

A cookbook can prepare LVM storage without embedding its own device discovery and initialisation commands.

### Negative

Hosts with pre-existing signatures need a separate operator-owned migration. An interrupted write may require diagnosis before retry.

### Follow-up

Verify filter and devices-file behaviour on all target distros. Confirm the device guards against actual Linux block-device layouts.

## Alternatives considered

Running `pvcreate` directly leaves repeat detection and device safety to each cookbook. A separate removal action would introduce a data-loss surface outside this proposal.

## References

- [LVM2 pvcreate manual](https://man7.org/linux/man-pages/man8/pvcreate.8.html)
- [LVM2 pvs reporting](https://man7.org/linux/man-pages/man8/pvs.8.html)
- [util-linux wipefs signature discovery](https://man7.org/linux/man-pages/man8/wipefs.8.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
