# 0018. Built-in `lvm_volume_group` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

An LVM2 volume group (VG) pools physical volumes so logical volumes can
allocate extents from them. ADR-0017 proposes the physical volume
prerequisite. The shared client provides Linux MD management through
`mdadm` (`lib/chef/resource/mdadm.rb:26`), which manages a different storage
layer.

The shared Chef source has no built-in `lvm_volume_group` resource.

On Ubuntu 24.04 with LVM2 2.03.16, `vgs --reportformat json` accepted the
name, UUID, extent size and PV count fields on a node with no volume
groups. The creation and membership changes remain guest tests.

## Decision

Add `lvm_volume_group` to Cinc Client in the shared Chef codebase. Manage creation and additive membership of a local LVM2 volume group. Require already initialised physical volumes.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Create the group if absent. Verify the requested members on an existing group. |
| `:extend` |  | Add the requested unassigned members to an existing group. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | Literal LVM name starting with an alphanumeric character. Use ASCII letters, digits, underscores, periods and hyphens. | Group identifier. |
| `physical_volumes` | Array of String | Required | Nonempty, unique stable block-device paths. Reject multiple aliases for one PV UUID. | Members required by the selected action. |

### Current state and action steps

- Read `vgs` and `pvs` JSON reports and resolve requested devices to PV
  UUIDs. Require an unambiguous group name and a healthy local group. Reject
  missing PV labels or PVs assigned to another group.
- `:create` runs `vgcreate` only if the group is absent. For an existing
  group it checks that the requested UUIDs are already members, and reports
  missing members with the `:extend` action as the explicit remedy.
- `:extend` computes the missing UUIDs and runs `vgextend` on those devices
  only. An empty difference is a no-op. Neither action removes existing
  members. Re-read both reports after writes and verify membership.
- Serialize resource mutations locally, recheck the preconditions and let
  normal LVM locking protect metadata. Refresh state after interrupted
  operations before deciding what to retry.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Unknown or ineligible PV | No PV UUID or PV already assigned to a different VG. | Fail before commands can implicitly initialise a device. |
| Unexpected group state | Missing target for extend, duplicate name or partial group. | Fail with the group identity and diagnostic. |
| Write fails or membership differs | Command status and new report. | Report actual membership so the next run can reevaluate safely. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS, Debian 13 and RHEL 10 | Proposed for local LVM2 volume groups. |
| Shared, exported or partial VGs | Deferred. |

Use the [platform baseline](../../README.md#platform-baseline) for Ubuntu 26.10 smoke tests and
Debian 12 LTS compatibility checks when the required tools are available.

### Target mode

Defer until remote reads, local device identity and LVM command routing are proven together.

## Usage examples

### :create

```ruby
lvm_volume_group 'worker_vg' do
  physical_volumes ['/dev/disk/by-id/wwn-0x5000000000000017']
  action :create
end
```

### :extend

```ruby
lvm_volume_group 'worker_vg' do
  physical_volumes ['/dev/disk/by-id/wwn-0x5000000000000018']
  action :extend
end
```

## End-to-end testing

Create a VG on disposable PVs, extend it with another PV, then check a
second converge changes nothing. Verify foreign-PV, duplicate-device and
uninitialised-device refusal. Confirm existing member UUIDs survive.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for report parsing and command-selection fixtures. |
| kitchen-exec | Possible with isolated extra guest disks. |
| Vagrant | Preferred for metadata and repeat-converge tests. |

Run logic fixtures in Dokken. Use a rebootable guest with three disks for VG operations and foreign membership checks.

## Consequences

### Positive

Cookbooks can model additive capacity changes separately from disk preparation and filesystem management.

### Negative

Moving storage between groups and removing PVs require a separate maintenance workflow.

### Follow-up

Verify local LVM locking and devices-file integration, and test detection of incomplete groups on each supported distro.

## Alternatives considered

A combined PV and VG resource could initialise an unintended disk as part of grouping. Explicit PV preparation makes the disk-write decision visible and keeps the lifecycle reusable.

## References

- [ADR-0017](0017-lvm-physical-volume-resource.md)
- [LVM2 vgcreate manual](https://man7.org/linux/man-pages/man8/vgcreate.8.html)
- [LVM2 vgextend manual](https://man7.org/linux/man-pages/man8/vgextend.8.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
