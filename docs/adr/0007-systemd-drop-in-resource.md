# 0007. Built-in `systemd_drop_in` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

Linux packages ship systemd units and can replace those unit files on upgrade.
Drop-ins let an administrator override individual settings under
`/etc/systemd/system/UNIT.d`. Cinc Client's `systemd_unit` manages a unit
(`lib/chef/resource/systemd_unit.rb:27`). A `file` and a separate reload
notification currently express a drop-in.

The shared Chef source has no built-in `systemd_drop_in` resource.

On Ubuntu 24.04 with systemd 255.4-1ubuntu8.19, `systemd-analyze verify`
accepted a test service with a `TimeoutStopSec` drop-in in the normal
unit search path. The staging overlay still needs verification.

## Decision

Add `systemd_drop_in` to Cinc Client in the shared Chef codebase. Manage one drop-in for a system service, socket or timer. A manager reload is an explicit action. Existing processes continue running until a recipe or operator restarts them.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Validate and install the drop-in. |
| `:delete` |  | Remove this drop-in. |
| `:reload` |  | Ask systemd to reload the unit graph. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | ASCII letters, digits, underscores and hyphens. | Fragment basename in `90-chef-NAME.conf`. |
| `unit` | String | Required for file actions | One literal `.service`, `.socket` or `.timer` name. Reject path separators, globs and template names. | Existing system unit. |
| `content` | String | Required for `:create` | Nonempty UTF-8 INI text without NUL. | Complete fragment text. |

### Current state and action steps

- `:create` compares bytes and metadata. For a candidate change, verify the
  effective unit with `systemd-analyze verify` using a private unit-search
  overlay of the target's unit and drop-in paths. Preserve basename precedence
  and fail if the overlay cannot represent the installed layout. Publish the
  marked fragment atomically as `root:root` and `0644` after verification.
- `:delete` unlinks the marked fragment if present. Leave the directory and
  other drop-ins in place.
- `:reload` runs `systemctl daemon-reload` with a timeout on each invocation.
  A single delayed notification can reload multiple changes.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Missing or masked unit | Effective unit discovery and staged verification. | Fail before publishing. |
| Unexpected destination | Symlink, special file or missing managed marker. | Refuse file mutation. |
| Invalid or unverifiable candidate | Nonzero verifier exit, rejected directives reported for the candidate, or unsupported overlay layout. | Keep the installed file and report the diagnostic. |
| Manager unavailable | Reload returns failure or times out. | Keep the on-disk edit and allow a separate reload retry. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS, Debian 13 and RHEL 10 | Proposed after overlay parity and manager checks. |
| Containers without systemd PID 1 | File operation candidates where the unit and verifier exist. Reload requires the manager. |

Use the [platform baseline](../../README.md#platform-baseline) for Ubuntu 26.10 smoke tests and
Debian 12 LTS compatibility checks when the required tools are available.

### Target mode

Defer target mode until staging, validation and manager calls can use the same remote unit search paths.

## Usage examples

### :create

```ruby
systemd_drop_in 'worker-timeout' do
  unit 'worker.service'
  content "[Service]\nTimeoutStopSec=20s\n"
  action :create
end
```

### :delete

```ruby
systemd_drop_in 'worker-timeout' do
  unit 'worker.service'
  action :delete
end
```

### :reload

```ruby
systemd_drop_in 'worker-timeout' do
  action :reload
end
```

## End-to-end testing

Verify an override in `systemctl show` after reload, a no-op second converge,
an invalid candidate, and deletion that restores the vendor setting after
reload. Cover namespace precedence and confirm the test service's PID stays
unchanged through daemon reload.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for staged verification and file cases. |
| kitchen-exec | Good for a live manager on an Ubuntu runner. |
| Vagrant | Useful for package-upgrade and reboot cases. |

Run Dokken validation on all proposed platforms. Verify the live manager on Ubuntu and an EL guest before release.

## Consequences

### Positive

Cookbooks can override a vendor setting without copying the whole unit.

### Negative

An effective-unit verification overlay adds work on changed converges and needs parity tests across systemd versions.

### Follow-up

Prove that staged verification handles dash-prefix and top-level drop-ins and detects the intended invalid cases on each supported systemd version. Test package-upgrade behaviour.

## Alternatives considered

Copying the complete vendor unit into `systemd_unit` makes the cookbook responsible for incorporating later package changes. The drop-in keeps that responsibility with the package.

## References

- [systemd unit loading and drop-ins](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html)
- [systemd-analyze verify](https://www.freedesktop.org/software/systemd/man/latest/systemd-analyze.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
