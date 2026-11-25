# 0029. Built-in `xfs_project_quota` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

XFS project quotas account for directory trees independently of file
ownership. A mounted XFS filesystem must already enable project accounting
and enforcement. Cinc Client's `mount` configures mount options
(`lib/chef/resource/mount.rb:27`). Recipes currently have to initialise
project accounting and apply limits with separate commands.

The shared Chef source has no built-in `xfs_project_quota` resource.

The Ubuntu 26.04 LTS arm64 image provided `xfs_quota` from xfsprogs 6.18.0.
Quota enforcement and numeric report behaviour on a mounted filesystem
remain guest tests.

## Decision

Add `xfs_project_quota` to Cinc Client in the shared Chef codebase. Manage one directory tree's numeric project ID and hard limits on a local XFS mount. Require an existing tree and enabled project quotas.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:set` | Yes | Associate the directory tree and converge its project hard limits. |
| `:clear_limits` |  | Clear the hard limits of the verified project while keeping its accounting identity. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `path` | String | Required | Existing real directory on an identifiable local XFS mount. Refuse symlinks, nested mounts and overlapping managed roots. | Project directory tree. |
| `project_id` | Integer | Required | Positive XFS project ID. Reject reuse by another tree on the same filesystem. | Numeric accounting identity. |
| `block_hard_bytes`, `inode_hard_limit` | Integer | Unset | Positive integers. Require at least one for set. Byte limit must be exactly representable in the target tool's quota unit. | Desired hard limits. Omitted limits keep their current value. |

### Current state and action steps

- Resolve the XFS mount and verify quota accounting and enforcement.
  Read directory project IDs, inheritance flags and numeric quota data.
  Require an exclusive project mapping on the filesystem before tagging a
  tree. Check for pre-existing conflicting tags and resource mappings.
- `:set` uses `xfs_quota` with an explicit project path and ID for the
  initial tree assignment. Check inheritance and project state before
  applying only the requested limits. Re-read the numeric limits and
  project state. Track a partial tagging failure for a fresh retry.
- `:clear_limits` sets the verified project's hard limits to zero and leaves
  existing accounting and tags available for subsequent limit changes.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Project quotas disabled | Check filesystem type, mount and quota state. | Fail before mutation. |
| Conflicting tag, scope or unknown ownership | Inspect project mapping, mounts and tree. | Fail and identify the conflict. |
| Partial tagging or limit failure | Read-back reports changed identity or incorrect limits. | Report observed state before retry. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS, Debian 13 and RHEL 10 family | Proposed with xfsprogs and an XFS mount configured for project quotas. |
| Ubuntu 26.10 and Debian 12 LTS | Utility smoke and compatibility checks. |

See the [platform baseline](../../README.md#platform-baseline) for images and client availability.

### Target mode

Defer until remote mount recognition, state reads and quota mutations are verified together.

## Usage examples

### :set

```ruby
xfs_project_quota 'worker-cache' do
  path '/var/cache/worker'
  project_id 442
  block_hard_bytes 1_073_741_824
  inode_hard_limit 16_384
  action :set
end
```

### :clear_limits

```ruby
xfs_project_quota 'worker-cache' do
  path '/var/cache/worker'
  project_id 442
  action :clear_limits
end
```

## End-to-end testing

Create a private XFS volume in a guest. Verify project inheritance, hard
block and inode limits, an attempted write beyond the threshold, a second
converge and clearing limits. Cover ID conflicts and nested mounts.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for command fixtures. A private XFS mount and quota-capable kernel are required for enforcement. |
| kitchen-exec | Limited to runners with a private loop device and appropriate kernel support. |
| Vagrant | Preferred for XFS enforcement and filesystem isolation. |

Run userspace checks on the baseline images. Verify actual enforcement in a guest on a disposable XFS volume.

## Consequences

### Positive

Directory-tree limits apply across service accounts without changing ordinary ownership.

### Negative

Initial tagging walks the tree. A project ID must remain unique across the filesystem and its files.

### Follow-up

Prove exclusive mapping discovery and exact limit-unit conversions on supported xfsprogs versions. Test interruption during first tagging.

## Alternatives considered

User and group quotas are useful for account-based limits. XFS project quotas fit data directories that contain files owned by several users.

## References

- [XFS project quota administration](https://man7.org/linux/man-pages/man8/xfs_quota.8.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
