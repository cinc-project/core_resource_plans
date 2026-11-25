# 0003. Built-in `systemd_sysusers` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

Packages use `sysusers.d` to declare local system accounts for provisioning
at installation or boot. Cinc Client ships `user` and `group` for immediate
management (see `lib/chef/resource/user.rb:28` and
`lib/chef/resource/group.rb:57`). The shared Chef resource inventory has no
resource that installs and applies a declarative sysusers fragment.

On Ubuntu 24.04 with systemd 255.4-1ubuntu8.16, a dry run with `--replace`
reported the planned account without changing the account databases.
Applying the fragment created a user and same-name primary group. The
second application made no changes.

## Decision

Add `systemd_sysusers` to Cinc Client in the shared Chef codebase. Each
declaration owns one `u` or `g` line in
`/etc/sysusers.d/90-chef-NAME.conf`. systemd allocates IDs if the recipe
leaves them unspecified. A `u` entry provisions a login-disabled system user
and its same-name primary group.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Install the boot-time account declaration. |
| `:apply` | | Provision the declared account on the live node. |
| `:delete` | | Remove the declaration. Keep existing accounts and their IDs. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | Lowercase POSIX account name, up to 31 characters. Reject `root`. | Account and fragment identifier. |
| `entry_type` | Symbol | `:user` | `:user` or `:group`. | Whether the fragment creates a user and primary group or only a group. |
| `numeric_id` | Integer | Unspecified | Positive ID in the target's supported system range. Reject conflicts with an existing account. | Explicit UID for a user or GID for a group. Allocated by sysusers when unspecified. |

### Current state and action steps

- `:create` renders the line with `-` as the default ID and the systemd
  defaults for user description, home and shell. Check the local databases
  for name and ID conflicts before publishing. Preflight candidate text
  with `systemd-sysusers --dry-run --replace=PATH -`. Compare file bytes and
  metadata. Atomically install the marked fragment as `root:root` and `0644`.
- `:apply` reads the installed fragment and inspects the local user and
  group databases. For an existing matching account, check an explicit ID
  and, for users, the same-name primary group. A match is a no-op. For a
  missing account, check for conflicting identities and run
  `systemd-sysusers` with the absolute fragment path. Verify the new local
  entries. sysusers owns account allocation and locking.
- `:delete` removes the marked fragment if present. Account changes such as
  renumbering or removing users remain the responsibility of `user` and
  `group`.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Conflicting declaration | Another effective fragment declares the same identity with conflicting settings. | Fail before publishing and name the other fragment. |
| Account conflict | Existing local account has the wrong explicit ID or primary group, or requested name or ID belongs to another identity. | Fail with the observed identity. Leave account records untouched. |
| Missing tool or dry run fails | Capability check or failed command. | Keep the original fragment. |
| Apply fails or account is inconsistent | Command status and a fresh local database read. | Fail with the remaining mismatch. Keep the declaration so the next run can inspect current state and retry. |
| Unmanaged path | Existing path is a symlink, special file or regular file without the marker. | Fail before mutation. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 24.04 and Debian 12 | Initial targets for the system sysusers tool and local account database. |
| RHEL 9 | Initial target after testing ID ranges and package integration. |
| Other platforms and external identity provisioning | Deferred. |

### Target mode

Defer support at launch. Tests must prove that account collision checks,
file inspection and sysusers invocations all use the remote node.

## Usage examples

### :create

```ruby
systemd_sysusers 'telemetry' do
  action :create
end
```

### :apply

```ruby
systemd_sysusers 'telemetry' do
  action :apply
end
```

### :delete

```ruby
systemd_sysusers 'telemetry' do
  action :delete
end
```

## End-to-end testing

Prove that boot creates an account from a valid fragment on a clean guest.
Cover group-only declarations, repeated converge, allocation without a
fixed ID, occupied IDs, a vendor declaration conflict, and deletion with
existing accounts on disk.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for local databases, both entry types and collision fixtures. |
| kitchen-exec | Good for a live smoke case on a disposable Ubuntu GitHub runner. |
| Vagrant | Good for proving that the distro's boot unit consumes the fragment. |

Run the account cases on Ubuntu, Debian and RHEL family Dokken images.
Use the Ubuntu runner for a native smoke check. Run the boot scenario on a
rebootable guest before release.

## Consequences

### Positive

Cookbooks can provision service accounts using the same native mechanism as
packages. Image builders can install the declaration for first boot.

### Negative

Existing accounts may require manual reconciliation before adopting the
declaration. The initial interface covers users and groups with a simple
primary-group relationship.

### Follow-up

Check local account range policy and conflict detection, including name
service lookups, on Debian and RHEL. Evaluate supplementary memberships
after validating their ordering and ownership model.

## Alternatives considered

Using `user` and `group` works when the cookbook owns the account lifecycle.
A native fragment integrates with first boot and sysusers' ID allocation.
A generic raw-content resource would expose more directives and make
collision checks and action semantics harder to define.

## References

- [systemd sysusers.d format](https://www.freedesktop.org/software/systemd/man/latest/sysusers.d.html)
- [systemd-sysusers command](https://www.freedesktop.org/software/systemd/man/latest/systemd-sysusers.html)
- [Shared Chef resource inventory](https://github.com/chef/chef/tree/819adc4979af2e92980ae856a48e67c3f380111f/lib/chef/resource)
