# 0015. Built-in `sshd_config` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

OpenSSH server policy is commonly extended with `sshd_config.d` files.
Most directives use the first value obtained, so checking syntax alone
cannot prove an intended global setting takes effect. Cinc Client's
`service` manages the daemon lifecycle (`lib/chef/resource/service.rb:31`).

The shared Chef source has no built-in `sshd_config` resource.

On Ubuntu 24.04 with OpenSSH 9.6p1, `sshd -t` accepted a test drop-in
and `sshd -T` reported its global root-login policy.

## Decision

Add `sshd_config` to Cinc Client in the shared Chef codebase. Own one global OpenSSH hardening fragment for a supported distro Include layout. Manage only root-login and password-login restrictions at launch. Activate only through an explicit notification to the existing service resource.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Validate and persist a hardening fragment. |
| `:delete` |  | Remove the managed fragment after validating the remaining configuration. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | ASCII letters, digits, underscores and hyphens. | Basename in `00-chef-NAME.conf`. |
| `deny_root_login`, `deny_password_authentication` | Boolean | `false` | At least one must be true for create. | Emit the corresponding `PermitRootLogin no` or `PasswordAuthentication no` setting. |

### Current state and action steps

- Establish the installed include layout, service command and current
  syntax. Construct a staged copy of the supported configuration tree,
  keeping effective order, absolute references and the `Match` boundaries.
  Refuse layouts where exact staging cannot be demonstrated. Run `sshd -t`
  and compare `sshd -T` for the proposed global values against that tree.
- `:create` publishes the validated marked `0644` root-owned fragment with
  atomic replacement under a local lock, only when bytes or metadata differ.
- `:delete` validates a staged tree with the fragment absent and then unlinks
  the marked fragment. File actions leave the running daemon alone.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Conflicting higher-priority values | Staged `sshd -T` output does not match the requested global policy. | Fail before publication. |
| Unsupported Includes, service arguments or syntax | Discovery or candidate verification fails. | Keep installed configuration and report the reason. |
| Unmanaged path | Symlink, special file or missing marker. | Refuse file mutation. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS and Debian 13 | Proposed conditional on validating the packaged OpenSSH layout and standard service configuration. |
| RHEL custom policy and arbitrary Include trees | Defer until staging parity is demonstrated. |

Use the [platform baseline](../../README.md#platform-baseline) for Ubuntu 26.10 smoke tests and
Debian 12 LTS compatibility checks when the required tools are available.

### Target mode

Defer. Apply checks to staging alongside the live remote connection tests before offering remote support.

## Usage examples

### :create

```ruby
sshd_config 'site-hardening' do
  deny_root_login true
  action :create
end
```

### :delete

```ruby
sshd_config 'site-hardening' do
  action :delete
end
```

## End-to-end testing

Verify candidate and live configuration in a disposable guest. Start a
second test daemon on an isolated port. Prove that a valid key login still
works and that the selected restricted login fails. Test precedence,
conditional rules, a failing staged configuration and repeated converge.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for syntax and precedence fixtures. |
| kitchen-exec | Possible for a secondary isolated test daemon on a runner. |
| Vagrant | Preferred for packaged service reload and real remote-login tests. |

Run Dokken syntax fixtures on Ubuntu and Debian. Use an isolated guest and a secondary daemon for end-to-end login checks.

## Consequences

### Positive

Cookbooks can express common hardening defaults and verify that configured global values take effect before writing.

### Negative

Global checks cannot guarantee identical results for every connection covered by `Match`. Administrators still control when the running server reloads.

### Follow-up

Verify target include staging, the distro service's flags and effective output for representative `Match` contexts before broadening the interface.

## Alternatives considered

Replacing the whole server configuration would transfer distro defaults and upgrade reconciliation to the cookbook. A drop-in preserves the packaged base.

## References

- [OpenSSH server configuration](https://man.openbsd.org/sshd_config)
- [OpenSSH server test modes](https://man.openbsd.org/sshd)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
