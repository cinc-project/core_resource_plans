# 0028. Built-in `filesystem_format` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

The shared codebase ships `mount` for configuring mounted filesystems
(`lib/chef/resource/mount.rb:27`). Partition and LVM proposals supply block
devices. Filesystem creation remains a separate operation with different
safety and idempotence requirements.

The shared Chef source has no built-in `filesystem_format` resource.

On Ubuntu 26.04 LTS arm64, e2fsprogs 1.47.2 and xfsprogs 6.18.0 formatted
separate sparse files without force flags. `blkid -p` identified each
filesystem. The live-device guards remain guest tests.

## Decision

Add `filesystem_format` to Cinc Client in the shared Chef codebase. Create ext4 or XFS on an explicitly selected blank local block device. Existing filesystems of the requested type converge without writes.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Create the filesystem if the target is blank, or verify the existing type. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `device` | String | Required | Stable local block-device path under `/dev`. Require unique device identity. | Partition or logical volume. |
| `fstype` | Symbol | Required | `:ext4` or `:xfs`. | On-disk filesystem format. |
| `label` | String | Unset | Optional ASCII label within the selected format's byte limit. Reject controls and whitespace. | Label at creation and a requested postcondition. |

### Current state and action steps

- Resolve the device to its major and minor number. Read type, label and
  signatures with `blkid -p` and `wipefs --no-act`. A matching type and
  optional label is a no-op. A conflicting type, label or ambiguous probe
  fails before calling a formatter.
- For a blank target, require a writable, inactive partition or a logical
  volume without mounts, swap, dependent devices or competing signatures.
  Recheck device identity and usage under a local lock.
- Invoke the format-specific tool with explicit arguments and no force
  option. Verify the filesystem signature and requested label afterwards.
  Treat tool failures as incomplete until a fresh probe shows actual state.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Existing signature or ambiguous read | Inspect low-level signature output and exit status. | Fail before writing. |
| Mounted, held or unavailable device | Check current block-device usage. | Fail with the observed dependency. |
| Formatter failed or read-back differs | Command status and fresh probe. | Report actual state and let the next converge re-probe. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS, Debian 13 and RHEL 10 family | Proposed with packaged e2fsprogs or xfsprogs and util-linux. |
| Ubuntu 26.10 and Debian 12 LTS | Utility smoke and compatibility fixtures. |

See the [platform baseline](../../README.md#platform-baseline) for images and client availability.

### Target mode

Defer until remote device identity, usage checks and filesystem creation have parity tests.

## Usage examples

### :create

```ruby
filesystem_format 'worker-filesystem' do
  device '/dev/worker_vg/worker_data'
  fstype :xfs
  action :create
end
```

## End-to-end testing

Check creation and verification for each format, idempotence, mismatched
labels and rejection of active or formatted disks. Assert the filesystem
UUID stays stable on a second converge and mount it through `mount`.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for formatter and signature fixtures on disposable image files. |
| kitchen-exec | Possible with a private loop device and isolated mount namespace. |
| Vagrant | Preferred for real block devices and refusal cases. |

Run image-file fixtures in the baseline containers. Gate block-device safety and mount assertions in guests.

## Consequences

### Positive

Cookbooks can compose partitioning, LVM, formatting and mounting while keeping each storage operation explicit.

### Negative

Filesystem type and label changes require a separate maintenance workflow.

### Follow-up

Verify signature detection, minimum filesystem sizes and formatter output across the baseline.

## Alternatives considered

A combined format-and-mount operation couples two failure modes. Separate resources keep UUID verification and mount management individually observable.

## References

- [e2fsprogs mke2fs manual](https://man7.org/linux/man-pages/man8/mke2fs.8.html)
- [xfsprogs mkfs.xfs manual](https://man7.org/linux/man-pages/man8/mkfs.xfs.8.html)
- [Logical volume proposal](0019-lvm-logical-volume-resource.md)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
