# 0036. Built-in `fapolicyd_rule` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

fapolicyd evaluates ordered rules for application execution. On supported
EL systems, `fagenrules` compiles ordered rule fragments and the daemon
can reload the result. Cinc Client offers `file` and `execute`
(`lib/chef/resource/file.rb:32` and `lib/chef/resource/execute.rb:27`).
They leave rule validation and activation to the cookbook.

The shared Chef source has no built-in `fapolicyd_rule` resource.

On AlmaLinux 10 arm64, fapolicyd 2.0.1 accepted the staged example with
`fapolicyd-cli --check-rules` and rejected an invalid decision. The
check exercised syntax without loading rules.

## Decision

Add `fapolicyd_rule` to Cinc Client in the shared Chef codebase. Manage one exact executable-path denial under `/etc/fapolicyd/rules.d`. Require a backend with a standalone rule syntax checker. Activate the site's compiled policy through a separate action.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Validate and stage the persistent rule. |
| `:delete` |  | Remove the owned rule fragment. |
| `:reload` |  | Validate, compile and request activation of the effective ruleset. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | ASCII letters, digits, underscores and hyphens. | Fragment basename. |
| `path` | String | Required for create | Absolute literal path with simple components. Reject whitespace, shell characters, escapes, globs and directories. | Exact executable path denied. |
| `order` | Integer | `40` | Two-digit natural-sort prefix supported by site policy. | Rule ordering in `NN-chef-NAME.rules`. |

### Current state and action steps

- `:create` renders a single `deny_audit perm=execute all : path=...`
  rule. Require the packaged checker to validate the staging file with
  `fapolicyd-cli --check-rules PATH`. Inspect known first-match conflicts
  against the effective fragment order. Atomically publish only the
  resource-owned rule, with a marker and root-only writes.
- `:delete` removes only the marked fragment at the requested order. It
  leaves live policy to the next explicit reload.
- `:reload` stages and validates the effective compiled rules in the site's
  packaged order. Under a local resource lock, verify the inputs still
  match the validated candidate and invoke the packaged compile-and-reload
  workflow. Run activation as one delayed
  notification after changes. Confirm the expected rule appears in the
  daemon's effective list, or report the activation failure.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Missing syntax-check capability | Test installed package, CLI feature and rule compilation prerequisites. | Fail before publishing or reloading. |
| Rule syntax, ordering or ownership conflict | Check staged content, effective order and destination marker. | Report the diagnostic and preserve the existing rule. |
| Compile or daemon reload fails | Command status and effective-rule check. | Keep the staged fragments for inspection and a fresh retry. |

### Platform support

| Platform | Support |
|---|---|
| RHEL 10 family | Proposed only where fapolicyd-cli exposes offline rule validation. AlmaLinux 10 is the userspace fixture. |
| Ubuntu and Debian | Outside this initial EL policy backend. |

See the [platform baseline](../../README.md#platform-baseline) for images and client availability.

### Target mode

Defer until remote rule validation and activation have test coverage on a disposable target.

## Usage examples

### :create

```ruby
fapolicyd_rule 'worker-execution-policy' do
  path '/opt/worker/legacy-worker'
  action :create
end
```

### :delete

```ruby
fapolicyd_rule 'worker-execution-policy' do
  action :delete
end
```

### :reload

```ruby
fapolicyd_rule 'worker-execution-policy' do
  action :reload
end
```

## End-to-end testing

Check rule validation and order in fixtures. Use an EL10 guest with a
private test executable to prove explicit activation, denial, removal and
repeat convergence. Ensure another permitted test executable still runs.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for syntax and ordering fixtures. |
| kitchen-exec | Limited by availability of an isolated EL10 guest. |
| Vagrant | Preferred for live fanotify policy and service checks. |

Run syntax fixtures in almalinux:10. Gate live change and recovery on a dedicated EL10 guest.

## Consequences

### Positive

A recipe can manage one explicit execution restriction with validation before site policy activation.

### Negative

Rules are evaluated in order, and reload affects the full site policy. Live activation requires a separate test of the package's policy and daemon behaviour.

### Follow-up

Verify package version and offline checker availability across the EL10 family. Confirm compiled rule identity and persistence after reboot.

## Alternatives considered

Managing a whole ruleset gives one recipe control of every decision. An owned fragment preserves ordering and separates proposed changes from activation.

## References

- [Red Hat Enterprise Linux 10 fapolicyd guidance](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/security_hardening/assembly_blocking-and-allowing-applications-using-fapolicyd_security-hardening)
- [fapolicyd upstream](https://github.com/linux-application-whitelisting/fapolicyd)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
