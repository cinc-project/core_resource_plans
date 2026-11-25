# 0009. Built-in `logrotate_rule` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

Linux distributions commonly run logrotate on a timer or from cron. A
package or local administrator supplies fragments included by the service's
main configuration. Cinc Client can write a fragment with `template`
(`lib/chef/resource/template.rb:38`). The cookbook currently has to
construct the rule and arrange validation.

The shared Chef source has no built-in `logrotate_rule` resource.

On Ubuntu 24.04 with logrotate 3.21.0, debug validation accepted a test
fragment. A forced rotation using a private state file produced the
expected compressed log.

## Decision

Add `logrotate_rule` to Cinc Client in the shared Chef codebase. Manage a single literal log path in `/etc/logrotate.d/chef-NAME` with a typed retention policy. Leave rotation scheduling to the distribution.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Validate and install the rotation rule. |
| `:delete` |  | Remove the managed rule. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | ASCII letters, digits, underscores and hyphens. | Fragment ID. |
| `path` | String | Required for create | Literal absolute path below `/var/log`. Reject whitespace, quoting, glob and dot components. | One log file. |
| `frequency` | Symbol | `:daily` | `:daily`, `:weekly` or `:monthly`. | Rotation cadence. |
| `rotate` | Integer | `7` | Positive integer. | Number of rotations to retain. |
| `compress` | Boolean | `false` | Boolean. | Enable compression of rotated logs. |

### Current state and action steps

- `:create` renders a stanza with the frequency, count, `missingok`,
  `notifempty` and compression choice. Stage it outside the include path.
  Invoke `logrotate --debug --state /dev/null` on the candidate for isolated
  validation. Check the supported default main configuration includes the
  destination directory. Compare current file contents and metadata before
  atomically installing a marked `0644` file owned by root.
- `:delete` removes the marked rule if present. It leaves current and rotated
  logs and logrotate's state file in place.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Invalid log path or stanza | Field checks and debug parse. | Keep the existing rule. |
| Overlapping installed rule | Detect duplicate literal paths when checking installed rules. | Report the conflicting fragment. Glob overlaps require integration fixtures. |
| Missing tool or include | Tool and supported main configuration checks. | Fail before publishing. |
| Unmanaged path | Symlink, special file or missing marker. | Refuse to replace or delete it. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 24.04, Debian 12 and RHEL 9 | Proposed with packaged logrotate and its standard include path. |
| Applications that require postrotate scripts | Use the existing native configuration mechanism. |

### Target mode

Defer support until remote staging and validation use the target's own logrotate.

## Usage examples

### :create

```ruby
logrotate_rule 'worker' do
  path '/var/log/worker.log'
  frequency :daily
  rotate 7
  compress true
  action :create
end
```

### :delete

```ruby
logrotate_rule 'worker' do
  action :delete
end
```

## End-to-end testing

Use a disposable log and state file to force a rotation in the test
fixture. Check empty and absent logs, compression, retained files, a
competing rule, rule removal and two converges without false updates.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for validation and isolated rotation. |
| kitchen-exec | Good for packaged timer smoke tests on Ubuntu. |
| Vagrant | Useful for reboot and distro scheduler integration. |

Run fixture rotations on the initial distributions. Use an Ubuntu guest for normal timer activation.

## Consequences

### Positive

Recipes can declare simple rotation and retention without writing arbitrary logrotate syntax.

### Negative

The first interface omits process signalling and ownership-changing directives. Rotation compatibility depends on how a producer reopens its log.

### Follow-up

Test duplicate detection against existing glob rules, and document which applications can use this initial rotation policy.

## Alternatives considered

Rendering an arbitrary stanza from a template supports custom hooks. A typed resource can validate common retention rules while leaving advanced rules to that mechanism.

## References

- [logrotate upstream manual](https://github.com/logrotate/logrotate/blob/main/logrotate.8.in)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
