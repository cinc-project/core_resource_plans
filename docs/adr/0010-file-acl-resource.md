# 0010. Built-in `file_acl` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @tas50
- **Consulted / Informed:** Cinc maintainers

## Context

Linux access ACLs grant permissions to specific users or groups beyond
traditional mode bits. Named entries share an ACL mask that limits their
effective permissions. Cinc Client's `file` resource provides ownership
and mode (see `lib/chef/resource/file.rb:32` and its properties), and recipes
currently need to arrange Linux ACL reads and updates separately.

The shared Chef source has no built-in `file_acl` resource.

On Ubuntu 24.04 with acl 2.3.2, `setfacl --no-mask` set a named user
entry and the requested mask on a test file. Removing the named entry
made it disappear from `getfacl` output.

## Decision

Add `file_acl` to Cinc Client in the shared Chef codebase. Manage a single named access ACL entry on one existing local regular file or directory. Preserve the ACL mask so unrelated entries retain their effective permissions.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:set` | Yes | Set the specified named entry. |
| `:remove` |  | Remove the specified named entry. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `path` | String | Required | Absolute existing path. Reject a symlink or non-file/non-directory. | ACL target. |
| `principal_type` | Symbol | Required | `:user` or `:group`. | Entry namespace. |
| `principal` | Integer | Required | Positive numeric UID or GID accepted by the platform. | Stable entry identity. |
| `permissions` | String | Required for set | Three characters in `rwx` positions with `-` for absent bits. | Requested effective access. |

### Current state and action steps

- Read the numeric access ACL using `getfacl`. Compare the named entry and
  effective bits. Use the existing mask, or the current group mode bits if
  this is the first extended ACL. Reject a requested permission outside it.
- `:set` invokes `setfacl --no-mask --modify` with the desired entry. Supply
  an explicit mask equal to the prior group bits when creating an extended
  ACL. Re-read the ACL and confirm the entry and unrelated effective access.
- `:remove` is a no-op for a missing entry. Use `--no-mask --remove` for
  the single entry and verify the resulting ACL.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Permission constrained by mask | Compare requested bits with the effective mask. | Fail and identify the missing bits. |
| Unsupported filesystem or tools | ACL read or change fails. | Report the target and tool error. |
| Concurrent changes or unexpected ACL | Fresh postcondition or file identity differs. | Fail and require a new read before retrying. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 24.04, Debian 12 and RHEL 9 | Proposed for local Linux filesystems supporting access ACLs. |
| Network filesystems and default ACLs | Deferred. |

### Target mode

Defer until remote identity and ACL checks can be executed together with a remote-safe update.

## Usage examples

### :set

```ruby
file_acl 'worker-log-reader' do
  path '/var/log/worker.log'
  principal_type :user
  principal 1001
  permissions 'r--'
  action :set
end
```

### :remove

```ruby
file_acl 'worker-log-reader' do
  path '/var/log/worker.log'
  principal_type :user
  principal 1001
  action :remove
end
```

## End-to-end testing

Prove allowed and denied accesses under real numeric identities. Include a
file with no extended ACL, a masked entry, unaffected named entries and
mode bits, removal and repeated converge.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good on Linux with ACL-capable private volumes. |
| kitchen-exec | Good on disposable Linux runner storage. |
| Vagrant | Useful for real ext4 and XFS coverage. |

Run ACL fixture tests in an Ubuntu and EL matrix. Gate filesystem-specific support on guests with ext4 and XFS.

## Consequences

### Positive

Recipes can model access by per-principal ACL entry and converge it idempotently.

### Negative

Changing the shared mask needs a separate operation. A later chmod can affect ACL access, so recipes must order mode changes before this resource.

### Follow-up

Check first-entry mask creation and last-entry removal on the packaged ACL tools and test interaction with chmod.

## Alternatives considered

Running `setfacl` with its default mask recalculation can affect other named entries. Managing the entire ACL from one recipe works when that recipe owns all access entries.

## References

- [Linux ACL format and access checks](https://man7.org/linux/man-pages/man5/acl.5.html)
- [setfacl manual](https://man7.org/linux/man-pages/man1/setfacl.1.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
