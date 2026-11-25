# 0008. Built-in `systemd_journald` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

journald configuration drop-ins let operators set journal retention without
editing packaged defaults. `systemd_unit` manages units
(`lib/chef/resource/systemd_unit.rb:27`), while journald settings belong in
`journald.conf.d`.

The shared Chef source has no built-in `systemd_journald` resource.

On Ubuntu 24.04 with systemd 255.4-1ubuntu8.19,
`systemd-analyze cat-config systemd/journald.conf` included the test
retention fragment. Live retention was outside the container check.

## Decision

Add `systemd_journald` to Cinc Client in the shared Chef codebase. Own one retention fragment under `/etc/systemd/journald.conf.d` for the default journal namespace. Stage changes for the next service restart.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Install the retention fragment. |
| `:delete` |  | Remove the managed fragment. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | ASCII letters, digits, underscores and hyphens. | Basename in `90-chef-NAME.conf`. |
| `system_max_use`, `runtime_max_use` | String | Unset | Positive integer with an optional `K`, `M`, `G` or `T` suffix. | Limits for persistent and runtime journals. |
| `max_retention_sec` | String | Unset | Positive integer with `s`, `min`, `h` or `day` suffix. | Maximum retention interval. |

### Current state and action steps

- `:create` requires at least one setting. Render a deterministic `[Journal]`
  fragment and compare bytes, owner, group and mode. Atomically publish a
  marked `root:root` file with mode `0644`. Unset properties omit their
  directives from this fragment.
- `:delete` removes the marked fragment if present and keeps journal data.
  Recipes that need immediate activation can notify a journald service restart.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Empty or invalid configuration | Typed input validation. | Fail before writing. |
| Path collision | Nonregular file, symlink or missing managed marker. | Keep the existing path. |
| A later fragment sets the same key | Effective configuration inspection. | Report the conflicting file when diagnosed. Test precedence independently of the resource's file state. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS, Debian 13 and RHEL 10 | Proposed for the default journald namespace. |
| Other log daemons | Outside this resource's scope. |

Use the [platform baseline](../../README.md#platform-baseline) for Ubuntu 26.10 smoke tests and
Debian 12 LTS compatibility checks when the required tools are available.

### Target mode

Defer until remote file and precedence checks are covered. A future port needs no remote service restart as part of the file actions.

## Usage examples

### :create

```ruby
systemd_journald 'site-retention' do
  system_max_use '500M'
  max_retention_sec '14day'
  action :create
end
```

### :delete

```ruby
systemd_journald 'site-retention' do
  action :delete
end
```

## End-to-end testing

Check bytes, permissions, effective precedence and a no-op second converge.
In a guest, restart journald, generate test entries and confirm retention
uses the selected runtime or persistent backing store. Check deletion and
restart with a vendor value available.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for fragment and precedence checks. |
| kitchen-exec | Good for isolated runner smoke tests. |
| Vagrant | Preferred for persistent journal and reboot tests. |

Run Dokken on the initial platforms. Verify effective configuration and retention in a disposable systemd guest.

## Consequences

### Positive

Journal retention can be reviewed as structured recipe data.

### Negative

Activation follows the service lifecycle. Older journal files may remain until journald's own cleanup conditions apply.

### Follow-up

Confirm live behaviour of size and age limits, including active journal files, before documenting retention guarantees.

## Alternatives considered

A whole-file template gives useful control when a cookbook owns all journal settings. A typed drop-in leaves storage, forwarding and unrelated settings with their existing owners.

## References

- [journald configuration and drop-in precedence](https://www.freedesktop.org/software/systemd/man/latest/journald.conf.html)
- [journalctl and journal file cleanup](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
