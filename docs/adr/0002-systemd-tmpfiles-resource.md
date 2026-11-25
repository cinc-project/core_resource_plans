# 0002. Built-in `systemd_tmpfiles` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

Cinc Client can create a directory during a converge. A directory under `/run`
must be recreated at boot, and a cache may need age-based cleanup between runs.
systemd reads `tmpfiles.d` fragments to do both. Its system rules take local
files from `/etc/tmpfiles.d` and order the effective fragments by basename.

The shared Chef source ships `file` and `systemd_unit`, which manage files
and units respectively (see `lib/chef/resource/file.rb:32` and
`lib/chef/resource/systemd_unit.rb:27`). Its resource inventory has no
`tmpfiles.d` resource.

On Ubuntu 24.04 with systemd 255.4-1ubuntu8.16, applying one fragment with
`systemd-tmpfiles --create` created a root-owned directory with mode `0750`.

## Decision

Add `systemd_tmpfiles` to Cinc Client in the shared Chef codebase. Each
declaration owns one directory rule in `/etc/tmpfiles.d/90-chef-NAME.conf`.
Render the systemd `d` directive. The optional age enables periodic cleanup
of eligible contents. The target directory itself remains in place.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Install the persistent directory rule. |
| `:apply` | | Apply that rule to the live filesystem. |
| `:delete` | | Remove the fragment. Existing files and directories remain. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | Lowercase letters, digits, underscores and hyphens. Starts with a letter or digit. | Fragment identifier. |
| `path` | String | Required for `:create` | Absolute path below `/run` or `/var/cache`. Reject empty, dot and parent components, whitespace, glob and specifier characters. Reject the base directories themselves. | Directory under management. |
| `mode` | String | `'0755'` | Four octal digits with leading zero. | Directory permissions. |
| `owner`, `group` | String | `'root'` | Simple local account names. Resolve before `:apply`. | Directory ownership. |
| `age` | String | `'-'` | `-` or a positive integer followed by `m`, `h` or `d`. | Age threshold for cleanup. |

### Current state and action steps

- `:create` validates the fields and renders a single deterministic line plus
  a managed-file marker. Compare bytes, owner, group and mode. Atomically
  publish changes as `root:root` and `0644`.
- `:apply` reads the installed managed fragment. Inspect the directory with
  no symlink following and resolve the requested owner and group. If absent
  or its mode or ownership differs, run `systemd-tmpfiles --create` with
  the fragment's absolute path and check the postcondition. Apply the
  installed rule so notifications can use a name-only declaration.
- `:delete` removes the marked regular file if present.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Fragment collision | Path is a symlink, special file or lacks the marker. | Fail before writing or removing it. |
| Conflicting directory rule | Effective config contains another literal or glob rule affecting the same directory. | Fail on a detected conflict. Include boot-precedence fixtures in testing. |
| Unsafe live path or missing prerequisite | Symlink component, wrong object type, missing account or missing tool. | Fail before running the tool. |
| Apply failure | Nonzero exit or wrong postcondition. | Report the error. Keep the persistent rule for a subsequent run. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS, Debian 13, RHEL 10 | Initial test targets with the system tmpfiles utility and a compatible boot setup. |
| Other platforms | Require a separate support decision after fixture and boot tests. |

Use the [platform baseline](../../README.md#platform-baseline) for Ubuntu 26.10 smoke tests and
Debian 12 LTS compatibility checks when the required tools are available.

### Target mode

Defer support at launch. Path checks, identity lookup and filesystem reads
must share a target with the command. Plan SSH coverage before enabling it.

## Usage examples

### :create

```ruby
systemd_tmpfiles 'worker-cache' do
  path '/var/cache/worker'
  mode '0750'
  age '7d'
  action :create
end
```

### :apply

```ruby
systemd_tmpfiles 'worker-cache' do
  action :apply
end
```

### :delete

```ruby
systemd_tmpfiles 'worker-cache' do
  action :delete
end
```

## End-to-end testing

Check that a fresh boot recreates a test directory under `/run`, the cleanup
service removes aged test files and keeps recent files, and removing the rule
stops future recreation. Check a repeated converge, mode drift, a symlink
target and competing vendor fragments.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for file convergence, drift and isolated utility runs. Recreating a container is insufficient evidence for boot behaviour. |
| kitchen-exec | Good for Ubuntu smoke tests on disposable GitHub runners. |
| Vagrant | Required for reboot and cleanup-timer assertions on a guest. |

Run Dokken on the three initial platforms. Add an Ubuntu runner smoke suite
and gate boot semantics on one rebootable Ubuntu guest.

## Consequences

### Positive

Runtime directories and cache cleanup use the same native policy at boot and
during a converge.

### Negative

The first release covers a limited set of directory rules. Other tmpfiles
directives still use native fragments managed by the cookbook.

### Follow-up

Confirm conflict detection against the supported versions' precedence and glob
rules. Verify age-based cleanup and boot behaviour on all initial platforms.

## Alternatives considered

Using `directory` plus `cron` creates separate boot and cleanup policies.
Using `RuntimeDirectory=` fits a directory owned by one service. A tmpfiles
fragment also covers shared paths and caches that outlive a service.

## References

- [systemd tmpfiles.d format](https://www.freedesktop.org/software/systemd/man/latest/tmpfiles.d.html)
- [systemd-tmpfiles command](https://www.freedesktop.org/software/systemd/man/latest/systemd-tmpfiles.html)
- [Shared Chef resource inventory](https://github.com/chef/chef/tree/819adc4979af2e92980ae856a48e67c3f380111f/lib/chef/resource)
