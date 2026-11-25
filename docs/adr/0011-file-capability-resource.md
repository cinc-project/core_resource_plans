# 0011. Built-in `file_capability` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @tas50
- **Consulted / Informed:** Cinc maintainers

## Context

Linux file capabilities grant selected capabilities to a process when a
supported executable is started. Cinc Client manages file bytes, ownership
and mode through `file` (`lib/chef/resource/file.rb:32`). There is no Cinc
file resource dedicated to the capability xattr.

The shared Chef source has no built-in `file_capability` resource.

On Ubuntu 24.04 with libcap 2.66, `setcap` assigned a capability to a
private test executable. `getcap` reported the assignment and returned
no assignment after removal.

## Decision

Add `file_capability` to Cinc Client in the shared Chef codebase. Manage the whole capability assignment for one existing ELF executable using the Linux libcap tools. The first version supports permitted capabilities with the effective flag and no inheritable set.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:set` | Yes | Set the executable's declared capability set. |
| `:remove` |  | Remove the executable capability assignment. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `path` | String | Required | Absolute path to an existing regular ELF executable. Reject symlink components and set-ID files. | Managed executable. |
| `capabilities` | Array of String | Required for set | Nonempty, unique lowercase names known to target libcap. Reject flags and assignment operators. | Complete permitted and effective capability set. |

### Current state and action steps

- Inspect the stable target and use `getcap` to read the capability value.
  Compare parsed sets and flags rather than raw output order.
- `:set` runs `setcap` with the normalised list and `=ep` if state differs.
  Order this action after package, owner and content updates. Repeat
  `getcap` to verify the xattr and confirm the file identity.
- `:remove` returns when no xattr exists. Otherwise call `setcap -r` and
  verify absence. An existing empty capability xattr counts as present.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Unsafe executable or unknown capability | Type, ownership and input checks. | Fail before changing the file. Require a root-owned file without group or other write permission. |
| Unsupported existing namespaced layout | Inspect the reported root ID and flags. | Fail and identify required manual reconciliation. |
| Insufficient privileges or missing xattr support | Tool exit status and postcondition. | Fail without reporting the desired state as applied. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 24.04, Debian 12 and RHEL 9 | Proposed on filesystems and mount settings supporting file capabilities. |
| Rootless containers or namespaced capability assignments | Deferred. |

### Target mode

Defer support until the target executable and privilege checks can be made over the remote transport.

## Usage examples

### :set

```ruby
file_capability 'worker-port-binding' do
  path '/usr/local/bin/worker'
  capabilities ['cap_net_bind_service']
  action :set
end
```

### :remove

```ruby
file_capability 'worker-port-binding' do
  path '/usr/local/bin/worker'
  action :remove
end
```

## End-to-end testing

Verify the xattr, repeated converge, replacement by a package-style atomic
rename and removal. On a guest, test a small executable with one narrowly
scoped capability under an ordinary account and verify capability status
from the launched process.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for xattrs if the container grants CAP_SETFCAP. |
| kitchen-exec | Good for isolated runner execution checks. |
| Vagrant | Preferred for mount flags and runtime privilege tests. |

Exercise xattrs on Ubuntu and EL containers. Use an isolated guest for the executable test.

## Consequences

### Positive

Recipes can reapply a small explicit capability set after replacing a binary.

### Negative

The resource owns the full xattr. Runtime capabilities also depend on mount, namespace and process policy.

### Follow-up

Test named and empty xattr representations, package upgrades and effective capability behaviour on the proposed filesystems.

## Alternatives considered

A set-ID executable runs with broader credentials. A systemd service capability policy is appropriate when one unit owns execution. File capabilities fit an executable launched outside a specific unit.

## References

- [Linux capability semantics](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [libcap setcap manual](https://man7.org/linux/man-pages/man8/setcap.8.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
