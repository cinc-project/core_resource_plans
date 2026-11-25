# 0019. Built-in `lvm_logical_volume` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

Logical volumes allocate extents from an LVM2 volume group and present
block devices to filesystems and applications. ADR-0018 proposes management
of the group. Cinc Client's `mount` manages the mounted filesystem
(`lib/chef/resource/mount.rb:27`).

The shared Chef source has no built-in `lvm_logical_volume` resource.

On Ubuntu 24.04 with LVM2 2.03.16, `lvs --reportformat json` accepted the
name, size, attributes and segment type fields and returned an empty
report. Extent growth and device identity remain guest tests.

## Decision

Add `lvm_logical_volume` to Cinc Client in the shared Chef codebase. Create or grow a conventional linear logical volume in an existing local VG. Use an explicit extent count. Filesystem formatting and growth remain separate operations.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Create a missing linear LV or verify the existing LV size. |
| `:extend` |  | Grow the existing LV to the requested total extent count. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | Literal LVM name starting with an alphanumeric character. Reject path components and reserved suffixes. | Logical volume name within the VG. |
| `volume_group` | String | Required | Existing unambiguous local VG name. | Parent volume group. |
| `extents` | Integer | Required | Positive extent count, within the VG's representable range. | Absolute size in VG extents. |

### Current state and action steps

- Read `lvs --reportformat json` and `vgs --reportformat json`. Resolve the
  VG and LV name and compare byte size using the VG extent size. Require
  a healthy VG and a conventional linear LV when one exists.
- `:create` runs `lvcreate` with the requested extent count if the LV is
  missing. An existing LV at the desired size is a no-op. A size mismatch
  fails with the observed size and identifies `:extend` for growth.
- `:extend` is a no-op at the desired size. Require enough free extents,
  reject a smaller requested size, and pass an absolute total to
  `lvextend`. Re-read the size and confirm LV identity after the operation.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Wrong or missing device type | LV report shows a snapshot, thin, RAID or other unsupported LV, or extend cannot find the LV. | Fail without mutation. |
| Shrink, incompatible extent size or insufficient capacity | Compare size and free extents before calling LVM. | Report actual and desired sizes. |
| Metadata write fails | Command status or unexpected postcondition. | Report the observed LV so retry starts with current state. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 24.04, Debian 12 and RHEL 9 | Proposed for local LVM2 linear volumes. |
| Thin, snapshot, RAID and clustered volumes | Deferred. |

### Target mode

Defer until target privileges and read-after-write reporting have SSH coverage.

## Usage examples

### :create

```ruby
lvm_logical_volume 'worker_data' do
  volume_group 'worker_vg'
  extents 256
  action :create
end
```

### :extend

```ruby
lvm_logical_volume 'worker_data' do
  volume_group 'worker_vg'
  extents 512
  action :extend
end
```

## End-to-end testing

Create a new LV, verify it has the expected block size, grow it, and check
that a sentinel filesystem and data survive. Repeat each desired state
and check no changes. Reject shrink, insufficient space and non-linear LVs.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for report and size-conversion fixtures. |
| kitchen-exec | Possible on isolated runners with dedicated guest disks. |
| Vagrant | Preferred for LV identity, block size and filesystem preservation tests. |

Run reporting fixtures in Docker and a disposable LVM guest suite for real creation and growth.

## Consequences

### Positive

The actual allocation is deterministic and cookbook reports distinguish LV creation from growth.

### Negative

Cookbooks must handle filesystems explicitly. Changing VG extent size would change the meaning of the numeric size.

### Follow-up

Check size and segment reporting across distros, and document supported LV name rules against the installed LVM2 tools.

## Alternatives considered

Percentage sizes are convenient for provisioning but depend on mutable capacity. Resizing a filesystem in the same action couples two failure modes, so the initial resource manages only the block volume.

## References

- [ADR-0018](0018-lvm-volume-group-resource.md)
- [LVM2 lvcreate manual](https://man7.org/linux/man-pages/man8/lvcreate.8.html)
- [LVM2 lvextend manual](https://man7.org/linux/man-pages/man8/lvextend.8.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
