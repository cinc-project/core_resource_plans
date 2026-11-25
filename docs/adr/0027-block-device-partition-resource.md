# 0027. Built-in `block_device_partition` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

A partition table divides a disk into separately addressable block devices.
Cinc Client's `mount` resource expects the device or filesystem to exist
(`lib/chef/resource/mount.rb:27`). ADR-0017 starts with an existing block
device. A safe way to create the partition is a separate storage step.

The shared Chef source has no built-in `block_device_partition` resource.

On Ubuntu 26.04 LTS arm64, sfdisk 2.41.3 created a GPT partition in a
sparse file. Its JSON report returned the expected start, size, partition
type and UUID. Block-device rediscovery remains a guest test.

## Decision

Add `block_device_partition` to Cinc Client in the shared Chef codebase. Create and verify a GPT Linux data partition at a specified sector range on an explicitly selected local disk. Existing partition geometry and identities stay in place.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Create the missing partition or verify that its existing geometry matches. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `disk` | String | Required | Stable absolute disk path under `/dev/disk/by-id`. Resolve to a local whole-disk block device. | Disk to inspect and update. |
| `partition_number` | Integer | Required | Positive GPT partition slot supported by the existing table or tool default. | Partition identity within the disk. |
| `start_sector`, `sector_count` | Integer | Required | Positive logical sectors. Require an aligned start and free range within usable GPT boundaries. | Exact position and length. |

### Current state and action steps

- Read logical sector size, device identity, partition table and signatures.
  Use `sfdisk --json` to compare number, range and the Linux data type.
  A matching partition is a no-op. Refuse a conflicting slot or layout.
- If the disk has no table, initialise GPT only after verifying the disk
  is blank. On existing GPT, verify the desired extent is free and refuse
  active mounts, swap, holders and ambiguous signatures. Check the
  complete proposed geometry with the tool's read-only path.
- Under a local lock, recheck disk identity and geometry. Write the table
  using `sfdisk` with signature wiping disabled. Preserve existing disk
  and partition IDs. Re-read the table and wait for the matching kernel
  device. A retry distinguishes a recorded partition from a delayed device.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Occupied range or unsupported disk layout | Read geometry, signatures and holders. | Fail before writing. |
| Conflicting existing slot or signature | Compare the exact requested extent and type. | Fail with the observed state. |
| Table written but device unavailable | Disk read-back matches and kernel state is missing. | Report partial completion and let the next run verify discovery. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS, Debian 13 and RHEL 10 family | Proposed for GPT and util-linux sfdisk on local disks. |
| Ubuntu 26.10 and Debian 12 LTS | Smoke and compatibility coverage respectively where the required report and check options exist. |

See the [platform baseline](../../README.md#platform-baseline) for images and client availability.

### Target mode

Defer until remote device identity and kernel rediscovery are verified together.

## Usage examples

### :create

```ruby
block_device_partition 'worker-data-partition' do
  disk '/dev/disk/by-id/wwn-0x5000000000000027'
  partition_number 1
  start_sector 2048
  sector_count 2_097_152
  action :create
end
```

## End-to-end testing

Use sparse files for parser fixtures and disposable guest disks for writes.
Verify a blank disk, an unused region on an existing GPT disk, second-run
idempotence, overlap refusal and preservation of all existing identifiers.
Check that the partition node appears without rewriting the table on retry.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for table fixtures stored in regular files. |
| kitchen-exec | Possible on a disposable runner with a private attached block device. |
| Vagrant | Preferred for real device discovery and reboot assertions. |

Use the baseline container images for tool fixtures and guest disks for destructive-case and device rediscovery tests.

## Consequences

### Positive

Recipes can declare the exact storage geometry before provisioning a filesystem or physical volume.

### Negative

Initial support requires explicit sector geometry. Resizing and repartitioning remain operator-controlled storage changes.

### Follow-up

Verify sfdisk preservation flags, locking and kernel rediscovery behaviour on each distro. Confirm device alignment for 512-byte and 4 KiB logical sectors.

## Alternatives considered

A shell command can create a partition, with repeat detection and disk identity checks left to the cookbook. A separate resource makes both part of the convergence contract.

## References

- [util-linux sfdisk manual](https://man7.org/linux/man-pages/man8/sfdisk.8.html)
- [LVM physical volume proposal](0017-lvm-physical-volume-resource.md)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
